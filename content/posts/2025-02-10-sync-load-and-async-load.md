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

In TiDB, the query execution process is divided into two stages: optimization and execution. The optimization stage generates the execution plan, while the execution stage executes the plan. During optimization, the optimizer checks if the necessary statistics are available. If not, it submits a load request to the sync load handler. This process occurs during the logical optimization phase, specifically in the `CollectPredicateColumnsPoint` and `SyncWaitStatsLoadPoint` steps.

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
	statsHandle statstypes.StatsHandle
	is          infoschema.InfoSchema
	StatsLoad   statstypes.StatsLoad
}

type StatsLoad struct {
	NeededItemsCh  chan *NeededItemTask
	TimeoutItemsCh chan *NeededItemTask
	sync.Mutex
}

type NeededItemTask struct {
	ToTimeout time.Time
	ResultCh  chan stmtctx.StatsLoadResult
	Item      model.StatsLoadItem
	Retry     int
}

type TableItemID struct {
	TableID          int64
	ID               int64
	IsIndex          bool
	IsSyncLoadFailed bool
}

type StatsLoadItem struct {
	TableItemID
	FullLoad bool
}

type StatsLoadResult struct {
	Item  model.TableItemID
	Error error
}
```
