---
title: 'TiDB Statistics: Sync Load and Async Load'
date: 2025-02-05
description: 'Sync Load and Async Load are two ways to load statistics in TiDB. This article introduces the differences between them and explains how they work.'
tags: ["Golang", "TiDB", "SQL", "Statistics"]
categories: ["Database"]
---

# Sync Load and Async Load

In my previous article, I introduced the statistics in TiDB and how it initializes them. However, there is an issue: even with comprehensive initialization, statistics may still be missing for columns that are not indexed. Additionally, after initialization, statistics may be evicted from memory due to memory pressure. In this article, I will introduce two methods to load statistics in TiDB on the fly: Sync Load and Async Load.

## Sync Load

Sync Load is a method to load statistics synchronously. When a query is executed, if the statistics of the table are missing, the query will wait for the statistics to be loaded before continuing.

For example, consider the following query:

```sql
SELECT * FROM t WHERE a = 1;
```

If the statistics of column `a` in table `t` are missing, the query will wait for the statistics to be loaded before continuing. This is called Sync Load.

### The basic idea of Sync Load

Although it might seem straightforward to implement Sync Load by loading the statistics before executing the query, the actual implementation is more complex. TiDB uses a sync load handler to manage this process. The sync load handler is a singleton in the TiDB server, responsible for loading statistics synchronously. When a query is executed, it submits a load request to the sync load handler. The handler then loads the statistics, and the optimizer checks if the statistics are available before proceeding with the query execution.

In TiDB, the query execution process is divided into two stages: optimization and execution. During optimization, the optimizer checks if the necessary statistics are available. If not, it submits a load request to the sync load handler. This process occurs during the logical optimization phase, specifically in the `CollectPredicateColumnsPoint` and `SyncWaitStatsLoadPoint` steps.

`CollectPredicateColumnsPoint` collects columns used in the query that require statistics and sends a request to the sync load handler. `SyncWaitStatsLoadPoint` waits for the statistics to be loaded, ensuring that all necessary statistics are available before the query execution proceeds.

### The handler implementation

To implement the sync load handler, the first challenge is ensuring that statistics are not loaded multiple times, as many queries might require the same statistics. To address this, TiDB introduces a `singleflight` mechanism. This mechanism ensures that only one loading operation occurs for a given statistic at any point in time, preventing multiple queries from triggering the same loading requests simultaneously.

TiDB uses an open-source library named `golang.org/x/sync/singleflight` to implement the `singleflight` mechanism. This mechanism allows multiple callers to share the result of a function call, ensuring that the function is only executed once for a given key. Here’s how it works:

```go
var g singleflight.Group

func loadStatistics(ctx context.Context, tableID int64) (*statistics.Table, error) {
    v, err, _ := g.Do(tableID, func() (interface{}, error) {
        return loadStatisticsFromStore(ctx, tableID)
    })
    if err != nil {
        return nil, err
    }
    return v.(*statistics.Table), nil
}
```

In this example, the `loadStatisticsFromStore` function is guaranteed to be called only once for each `tableID`, even if multiple goroutines invoke the `loadStatistics` function concurrently. This prevents redundant loading operations for the same statistics.

The handler is implemented as follows:

```go
type statsSyncLoad struct {
	statsHandle    statstypes.StatsHandle
	neededItemsCh  chan *statstypes.NeededItemTask
	timeoutItemsCh chan *statstypes.NeededItemTask
	...
}
```

The core of the sync load handler is the `statsSyncLoad` structure, which contains two channels: `NeededItemsCh` and `TimeoutItemsCh`. The `NeededItemsCh` channel is used to submit load requests, while the `TimeoutItemsCh` channel is used to handle timeout events.

The handler implementation can be divided into three parts:
1. Send requests to the handler
2. Load statistics concurrently
3. Wait for the statistics to be loaded

`statsSyncLoad` provides the `SendLoadRequests` method to allow the optimizer to submit load requests to the handler.

```go
func (s *statsSyncLoad) SendLoadRequests(sc *stmtctx.StatementContext, neededHistItems []model.StatsLoadItem, timeout time.Duration) error {
	remainedItems := s.removeHistLoadedColumns(neededHistItems)
	...
	sc.StatsLoad.Timeout = timeout
	sc.StatsLoad.NeededItems = remainedItems
	sc.StatsLoad.ResultCh = make([]<-chan singleflight.Result, 0, len(remainedItems))
		for _, item := range remainedItems {
		localItem := item
		resultCh := globalStatsSyncLoadSingleFlight.DoChan(localItem.Key(), func() (any, error) {
			timer := time.NewTimer(timeout)
			defer timer.Stop()
			task := &statstypes.NeededItemTask{
				Item:      localItem,
				ToTimeout: time.Now().Local().Add(timeout),
				ResultCh:  make(chan stmtctx.StatsLoadResult, 1),
			}
			select {
			case s.StatsLoad.NeededItemsCh <- task:
				metrics.SyncLoadDedupCounter.Inc()
				select {
				case <-timer.C:
					return nil, errors.New("sync load took too long to return")
				case result, ok := <-task.ResultCh:
					intest.Assert(ok, "task.ResultCh cannot be closed")
					return result, nil
				}
			case <-timer.C:
				return nil, errors.New("sync load stats channel is full and timeout sending task to channel")
			}
		})
		sc.StatsLoad.ResultCh = append(sc.StatsLoad.ResultCh, resultCh)
	}
	sc.StatsLoad.LoadStartTime = time.Now()
	return nil
}
```
The `SendLoadRequests` method first filters out columns that have already been loaded or are unnecessary. For the remaining columns, it:

1. Creates a `NeededItemTask` for each column/index requiring statistics
2. Sends these tasks to the `NeededItemsCh` channel
3. Includes in each task:
	- Column/index information
	- A `ResultCh` channel for receiving loading results
	- Timeout settings

This design ensures efficient handling of statistics loading requests while preventing duplicate loads for the different queries from different sessions.

**A thing to note is that we maintain the `ResultCh` and `NeededItems` in the `stmtctx.StatementContext` to keep track of the loading status for each statement from each session. This is a key point to track the loading status for each statement (query).**

After sending the load tasks, the optimizer waits for the statistics to be loaded. This process is implemented in the `WaitLoadFinished` method.

```go
func (*statsSyncLoad) SyncWaitStatsLoad(sc *stmtctx.StatementContext) error {
	...
	resultCheckMap := map[model.TableItemID]struct{}{}
	for _, col := range sc.StatsLoad.NeededItems {
		resultCheckMap[col.TableItemID] = struct{}{}
	}
	...
	for _, resultCh := range sc.StatsLoad.ResultCh {
		select {
		case result, ok := <-resultCh:
			...
			if result.Err != nil {
				errorMsgs = append(errorMsgs, result.Err.Error())
			} else {
				val := result.Val.(stmtctx.StatsLoadResult)
				if val.HasError() {
					errorMsgs = append(errorMsgs, val.ErrorMsg())
				}
				delete(resultCheckMap, val.Item)
			}
		case <-timer.C:
			metrics.SyncLoadCounter.Inc()
			metrics.SyncLoadTimeoutCounter.Inc()
			return errors.New("sync load stats timeout")
		}
	}
	if len(resultCheckMap) == 0 {
		metrics.SyncLoadHistogram.Observe(float64(time.Since(sc.StatsLoad.LoadStartTime).Milliseconds()))
		return nil
	}
	return nil
}
```

The `SyncWaitStatsLoad` method monitors the loading progress of statistics. It processes results from the `ResultCh` channels and handles any potential errors that occur during loading. The method operates under the fundamental premise that each result channel will receive exactly one result, and it will continue waiting until either:

1. All results are successfully received
2. A timeout occurs

To handle the tasks, `statsSyncLoad` utilizes multiple sub-workers to load statistics concurrently. This design ensures efficient processing without blocking query execution.

```go
func (s *statsSyncLoad) SubLoadWorker(sctx sessionctx.Context, exit chan struct{}, exitWg *util.WaitGroupEnhancedWrapper) {
	...
	var lastTask *statstypes.NeededItemTask
	for {
		task, err := s.HandleOneTask(sctx, lastTask, exit)
		lastTask = task
		if err != nil {
			switch err {
			case errExit:
				return
			default:
				...
				r := rand.Intn(500)
				time.Sleep(s.statsHandle.Lease()/10 + time.Duration(r)*time.Microsecond)
				continue
			}
		}
	}
}
```

From the code snippet above, we can see that the `SubLoadWorker` function is responsible for loading statistics concurrently. It processes tasks from the `NeededItemsCh` channel, loading statistics for each task. The worker continues processing until all statistics are loaded or the worker is terminated. Additionally, it incorporates a retry mechanism to handle potential errors during the loading process.

```go
func (s *statsSyncLoad) HandleOneTask(sctx sessionctx.Context, lastTask *statstypes.NeededItemTask, exit chan struct{}) (task *statstypes.NeededItemTask, err error) {
	...
	if lastTask == nil {
		task, err = s.drainColTask(sctx, exit)
		if err != nil {
			if err != errExit {
				logutil.BgLogger().Error("Fail to drain task for stats loading.", zap.Error(err))
			}
			return task, err
		}
	} else {
		task = lastTask
	}
	result := stmtctx.StatsLoadResult{Item: task.Item.TableItemID}
	err = s.handleOneItemTask(task)
	if err == nil {
		task.ResultCh <- result
		return nil, nil
	}
	if !isVaildForRetry(task) {
		result.Error = err
		task.ResultCh <- result
		return nil, nil
	}
	return task, err
}
```

The `HandleOneTask` function processes a single task from either the `NeededItemsCh` or `TimeoutItemsCh` channel. It loads statistics for the task and sends the result back to the ResultCh channel. If an error occurs during the loading process, the function checks if the task is eligible for a retry. If so, the task is returned for retry; otherwise, the error is sent back to the `ResultCh` channel. The default retry limit is set to 2 attempts.

