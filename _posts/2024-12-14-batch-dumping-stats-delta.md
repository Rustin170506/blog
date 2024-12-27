---
title: 'Batch Dumping Statistics Delta'
layout: post

categories: post
tags:
- Golang
- TiDB
- SQL
- Statistics
---

# Background

Recently, we have been tackling the challenge of supporting 3 million tables within a single TiDB cluster. One of the most significant hurdles we've faced is optimizing the performance of statistics collection. In its current implementation, TiDB gathers basic table information from all servers and consolidates it into a single system table. While functional, this approach becomes highly inefficient when managing millions of tables, consuming excessive CPU and taking a considerable amount of time.

In this blog post, I’ll introduce a new batch processing approach for writing table information to the system table. This method not only improves efficiency but also significantly reduces resource consumption. Let's dive in!

# Statistics Delta

## What is Statistics Delta?

Statistics play a crucial role in optimizing query plans. In TiDB, the owner node is responsible for collecting statistics and storing them in system tables. To ensure that the statistics accurately reflect the current data distribution, they need to be updated periodically. This raises an important question: When is the optimal time to collect these statistics?

In TiDB, we primarily use Data Manipulation Language (DML) changes to determine when to update the statistics. This method is known as the statistics delta. The statistics delta consists of three fields: `modify_count`, `count`, and `version`. The `modify_count` field tracks the number of changes made to the table (DELETE/INSERT/UPDATE), the `count` field records the total number of rows in the table, and the `version` field identifies the version of the statistics.

TiDB stores the statistics delta in the `mysql.stats_meta` table. This table contains a row for each table in the database, with each row holding the statistics delta for that table.
For example, when you insert a row into a table, the `modify_count` field increases by one, and the `count` field also increases by one. Conversely, when you delete a row, the `modify_count` field increases by one, but the `count` field decreases by one. When you update a row, the `modify_count` field increases by one, while the `count` field remains unchanged.

After collecting the statistics delta, TiDB owner nodes use this information to initiate the statistics collection process. Therefore, it is crucial to ensure that the statistics delta is as accurate and up-to-date as possible.

## How to collect statistics delta?

Since every node in the TiDB cluster can modify the data, it is necessary to collect the statistics delta from all nodes. To accomplish this, we use a single system table to store the statistics delta. However, it is not feasible to keep the statistics delta in the system table updated in real-time. Therefore, we periodically update the statistics delta in the system table to ensure it remains as current as possible.

In the current implementation, whenever TiDB commits a transaction, it updates the statistics delta for the current session. Every 2 minutes, TiDB aggregates these deltas and writes them to the system table. Each TiDB server can have many sessions, so we need to collect statistics changes from all sessions. TiDB uses a linked list to store the statistics delta for each session. When it is time to dump the statistics delta to the system table, TiDB traverses the linked list to gather the deltas from each session.

```go
type TableDelta struct {
	Delta    int64
	Count    int64
	ColSize  map[int64]int64
	InitTime time.Time // InitTime is the time that this delta is generated.
	TableID  int64
}

type TableDelta struct {
	delta map[int64]variable.TableDelta // map[tableID]delta
	lock  sync.Mutex
}

type SessionStatsItem struct {
	mapper     *TableDelta
	next       *SessionStatsItem
	sync.Mutex

	// deleted is set to true when a session is closed.
	deleted bool
}
```

As illustrated in the code snippet above, the current implementation employs a linked list to store the statistics delta for each session, maintaining it in memory. Each session directly updates the table delta via the `SessionStatsItem`. When a session is closed, the `deleted` flag is set to true, and the session is subsequently removed from the linked list, ensuring it is no longer utilized.

To manage the `SessionStatsItem` linked list, TiDB utilizes the `SessionStatsList` structure. This structure includes a pointer to the head of the linked list. Additionally, the `SessionStatsList` structure offers a method to traverse the linked list, consolidating the statistics delta from each session into a unified delta.

```go
type SessionStatsList struct {
	// tableDelta contains all the delta map from collectors when we dump them to KV.
	tableDelta *TableDelta

	... // other fields

	// listHead contains all the stats collector required by session.
	listHead *SessionStatsItem
}

// SweepSessionStatsList will loop over the list, merge each session's local stats into handle
// and remove closed session's collector.
func (sl *SessionStatsList) SweepSessionStatsList() {
	deltaMap := NewTableDelta()
	...
	prev := sl.listHead
	prev.Lock()
	for curr := prev.next; curr != nil; curr = curr.next {
		curr.Lock()
		// Merge the session stats into deltaMap respectively.
		merge(curr, deltaMap, colMap)
		if curr.deleted {
			prev.next = curr.next
			curr.Unlock()
		} else {
			prev.Unlock()
			prev = curr
		}
	}
	prev.Unlock()
	sl.tableDelta.Merge(deltaMap.GetDeltaAndReset())
    ...
}
```

Here is an example of the in-memory data for each session:

[![SessionStatsList](https://img.plantuml.biz/plantuml/png/dPLDQy8m6CVl_HH_SbP8iZatJy8yJF1MTzajMpFe5bCn4SP6l_jCcInjNeg2K7pUmlFxaNPfh3ZO3zFeugS0G4ffJDteqWfhDhMnP84kuN9Ml2gvaieAB-eILHXpOKPP47JnymXsDmboZyrHkqFvGoodolfRkfd4JMQKJa2ugwQq3UlNkhRRUkSQ2AVyTihubC-tZ2we-xsGi6NhLbolkk6ibst___b74TMyVRe3DgUdh4WnA27g1F59Ycg0R2VsUta8cSLHPcZ20peB5eA7bD54-YAgk0eiicpHmui-ONYGdxNgOHwK4Ys_RCWqmHevtCWIXmVz9hOiFExpTC6bv967Fql2nnX_31KWi82yYB0XhWDP8nYHWZ4lyDJS9r30lnKyMtI58MGbiVGD-UiTyuI8AiHiOLHOjEsiJH-L2fCdET9Azpfx5yh8hFzaRU_Ingkw5TkYBPPILzq7wXS0)](https://editor.plantuml.com/uml/dPLDQy8m6CVl_HH_SbP8iZatJy8yJF1MTzajMpFe5bCn4SP6l_jCcInjNeg2K7pUmlFxaNPfh3ZO3zFeugS0G4ffJDteqWfhDhMnP84kuN9Ml2gvaieAB-eILHXpOKPP47JnymXsDmboZyrHkqFvGoodolfRkfd4JMQKJa2ugwQq3UlNkhRRUkSQ2AVyTihubC-tZ2we-xsGi6NhLbolkk6ibst___b74TMyVRe3DgUdh4WnA27g1F59Ycg0R2VsUta8cSLHPcZ20peB5eA7bD54-YAgk0eiicpHmui-ONYGdxNgOHwK4Ys_RCWqmHevtCWIXmVz9hOiFExpTC6bv967Fql2nnX_31KWi82yYB0XhWDP8nYHWZ4lyDJS9r30lnKyMtI58MGbiVGD-UiTyuI8AiHiOLHOjEsiJH-L2fCdET9Azpfx5yh8hFzaRU_Ingkw5TkYBPPILzq7wXS0)


After merging the statistics delta from each session, TiDB writes the aggregated delta to the system table.

```go
func (s *statsUsageImpl) DumpStatsDeltaToKV(dumpAll bool) error {
	...
	s.SweepSessionStatsList()
	deltaMap := s.SessionTableDelta().GetDeltaAndReset()
	defer func() {
		s.SessionTableDelta().Merge(deltaMap)
	}()

	return utilstats.CallWithSCtx(s.statsHandle.SPool(), func(sctx sessionctx.Context) error {
		is := sctx.GetDomainInfoSchema().(infoschema.InfoSchema)
		currentTime := time.Now()
		for id, item := range deltaMap {
			if !s.needDumpStatsDelta(is, dumpAll, id, item, currentTime) {
				continue
			}
			updated, err := s.dumpTableStatCountToKV(is, id, item)
			if err != nil {
				return errors.Trace(err)
			}
			if updated {
				UpdateTableDeltaMap(deltaMap, id, -item.Delta, -item.Count, nil)
			}
			if err = storage.DumpTableStatColSizeToKV(sctx, id, item); err != nil {
				delete(deltaMap, id)
				return errors.Trace(err)
			}
			if updated {
				delete(deltaMap, id)
			} else {
				m := deltaMap[id]
				m.ColSize = nil
				deltaMap[id] = m
			}
		}
		return nil
	})
}
```

However, this function is inefficient. It processes each item individually, which is both time-consuming and resource-intensive. Each item requires a separate transaction, and each transaction incurs overhead due to the need to lock the table and write data to storage.


## Performance Issues

After testing the current implementation with 3 million tables, we observed significant performance issues. In high-load clusters, the statistics delta is updated frequently, generating a large volume of transactions that degrade performance. The root cause lies in the `DumpStatsDeltaToKV` function, where each execution triggers query compilation over 100,000 times, resulting in substantial CPU overhead.

Examining the `Stats Meta Updating Duration` metric reveals that the time required to update the statistics delta is excessively high.

![Stats Meta Updating Duration](https://github.com/user-attachments/assets/706b208b-47e5-448b-8f14-cfdc5cdc798a)

Further analysis of the CPU profile reveals that the `DumpStatsDeltaToKV` function is a major contributor to CPU consumption. A significant portion of the CPU time is spent on the `compile` function, highlighting it as a critical bottleneck.

![CPU Profile](https://github.com/user-attachments/assets/608be425-4330-4892-a6f5-5dacc649bc82)

It executes the following function over 100,000 times:

```go
// UpdateStatsMeta update the stats meta stat for this Table.
func UpdateStatsMeta(
	ctx context.Context,
	sctx sessionctx.Context,
	startTS uint64,
	delta variable.TableDelta,
	id int64,
	isLocked bool,
) (err error) {
	...
	// use INSERT INTO ... ON DUPLICATE KEY UPDATE here to fill missing stats_meta.
	_, err = statsutil.ExecWithCtx(ctx, sctx, "insert into mysql.stats_meta (version, table_id, modify_count, count) values (%?, %?, %?, 0) on duplicate key "+
		"update version = values(version), modify_count = modify_count + values(modify_count), count = if(count > %?, count - %?, 0)",
		startTS, id, delta.Count, -delta.Delta, -delta.Delta)
	...
	return err
}
```

To address these performance issues, we need to find a more efficient way to update the statistics delta without processing each item individually. However, **we must ensure that this solution does not consume additional CPU and memory resources.** Therefore, simply adding more concurrent workers to process the items in parallel is not a viable option.

# Batch Dumping Statistics Delta

## Design

To enhance the performance of updating the statistics delta, we propose a new approach: batch dumping statistics delta. This method aggregates the statistics delta from all sessions and writes it to the system table in a single transaction/query. By batching the updates, we can significantly reduce the number of transactions required, thereby improving performance and reducing resource consumption.

We change above `UpdateStatsMeta` function to take a slice of `TableDelta` as input, and update the statistics delta in a single transaction.

```go
type DeltaUpdate struct {
	Delta    variable.TableDelta
	TableID  int64
	IsLocked bool
}

func UpdateStatsMeta(
	ctx context.Context,
	sctx sessionctx.Context,
	startTS uint64,
	updates ...*DeltaUpdate,
) (err error) {
	...
}
```

Within the `UpdateStatsMeta` function, we continue to use the `INSERT INTO ... ON DUPLICATE KEY UPDATE` statement to update the statistics delta. However, we now specify multiple values in the `INSERT INTO` statement, allowing us to update the statistics delta in a single transaction.

```go
sql := fmt.Sprintf("insert into mysql.stats_meta (version, table_id, modify_count, count) values %s "+
	"on duplicate key update version = values(version), modify_count = modify_count + values(modify_count), "+
	"count = count + values(count)", strings.Join(unlockedPosValues, ","))
if _, err = statsutil.ExecWithCtx(ctx, sctx, sql); err != nil {
	return err
}
```

## Test Results

I have submitted a PR to implement this new approach. After testing the batch dumping statistics delta method, we observed a significant improvement in performance. You can view the PR [here](https://github.com/pingcap/tidb/pull/58331).

The duration dropped from 40 minutes to 20 seconds, as shown in the following screenshot:

![Stats Meta Updating Duration](https://github.com/user-attachments/assets/cb8561a4-bda0-4c66-91a6-ea6777e7012a)

In the CPU profile, the `DumpStatsDeltaToKV` function no longer consumes a significant amount of CPU time. The `compile` function is no longer a bottleneck, as the number of executions has been reduced to a more manageable level.


## Tuning the Batch Size

To achieve optimal performance, it is essential to identify the ideal batch size for updating the statistics delta. Through testing various batch sizes, we determined that a batch size of 100,000 items delivers the best results. This configuration balances efficiency and resource utilization, ensuring the update process is both fast and resource-efficient.

| Batch Size | Duration               | Metric                                                                                                          |
| ---------- | ---------------------- | --------------------------------------------------------------------------------------------------------------- |
| 5000       | ~40 sec                | [Stats Meta Updating Duration](https://github.com/user-attachments/assets/fb4af677-da64-41eb-a475-8968a5d50c9b) |
| 10000      | 20-40 sec; Most 40 sec | [Stats Meta Updating Duration](https://github.com/user-attachments/assets/a16f826a-81e6-43ed-aa7e-ef8d9f982960) |
| 50000      | 20-40 sec; Most 20 sec | [Stats Meta Updating Duration](https://github.com/user-attachments/assets/7871f8ba-daf2-41af-b99b-36a5e6f27468) |
| 100000     | ~20 sec                | [Stats Meta Updating Duration](https://github.com/user-attachments/assets/cb8561a4-bda0-4c66-91a6-ea6777e7012a) |


# Conclusion

The new batch dumping approach for the statistics delta has significantly enhanced TiDB statistics update performance. This method aggregates deltas from all sessions into a single transaction, minimizing overhead and resource consumption while ensuring an efficient update process, even in large-scale deployments.
