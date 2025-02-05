---
title: 'TiDB Statistics: Understanding the Initialization Process'
date: 2025-02-05
description: 'A deep dive into how TiDB initializes and manages its statistics system'
tags: ["Golang", "TiDB", "SQL", "Statistics"]
categories: ["Database"]
---

# Statistics

Statistics collection is a crucial process of modern database systems, forming the backbone of query optimization. In TiDB, statistics are indispensable, serving as the sole source of information for estimating query costs and selecting the most efficient execution plan.

TiDB collects several types of statistics for each table, including:
- TopN values (most frequent values to reflect data skewness)
- Histograms (data distribution)
- Number of Distinct Values (NDV)
- Other statistical metrics

These statistics will be stored in some system tables, such as `mysql.stats_meta`, `mysql.stats_top_n`, `mysql.stats_histograms`, and `mysql.stats_buckets`.

When TiDB starts, it must load these statistics from the system tables into memory, a process known as "statistics initialization." This step is crucial as it equips the query optimizer with the necessary statistical information to generate optimal execution plans.

In this blog post, we will discuss the initialization process of TiDB statistics.

## Initialization Process

### What Statistics Data Gets Loaded?

In TiDB, the statistics data can be divided into three parts:

> Note: If you are not familiar with these statistics fields yet, do not worry. We will explain them in the following sections.

1. Basic information: `modify_count`, `count`, `version` from `mysql.stats_meta` table.
2. TopN information: `value`, `count` from `mysql.stats_topn` table.
3. Histogram meta: `is_index`, `distinct_count`, `null_count`, `version`, `stats_ver` from `mysql.stats_histograms` table.
4. Histogram buckets: `count`, `repeats`, `lower_bound`, `upper_bound` from `mysql.stats_buckets` table.

When TiDB starts, it will load these statistics data from the system tables into memory.

### How TiDB Loads Statistics Data?

Since TiDB maintains extensive statistics for each column and index (including 100 TopN values and 256 histogram buckets), loading all statistics at once can be resource-intensive. To address this, TiDB provides two initialization approaches:

1. A lightweight mode that quickly loads only essential statistics
2. A comprehensive mode that loads all statistical data but takes longer to complete

This dual-mode approach allows TiDB to balance between startup speed and statistical completeness based on specific needs. But it also brings extra complexity and maintenance burden.

#### Lightweight Mode

In lightweight mode, TiDB loads only essential statistical information from two system tables:

From `mysql.stats_meta`:
- `modify_count`: Number of row modifications since the last statistics collection
- `count`: Total number of rows
- `version`: Last update time of the stats meta table (**This is the update time of the stats meta row, not the last analysis time**).

From `mysql.stats_histograms`:
- `is_index`: Whether the statistics are for an index.
- `distinct_count`: Number of distinct values.
- `null_count`: Number of NULL values.
- `version`: Last update time of the statistics.(**The real last analysis time**)
- `stats_ver`: Statistics format version(0, 1, 2). 0 means the statistics have not been analyzed yet.

The purpose of loading this basic information is to provide the `modify_count` and `count` metrics to other modules, allowing them to track both the real-time total number of rows in the table and how many rows have been modified since the last statistics collection.

#### Comprehensive Mode

In comprehensive mode, TiDB loads all statistical data from the system tables. This includes:

From `mysql.stats_meta`:
- `modify_count`: Number of row modifications since the last statistics collection
- `count`: Total number of rows
- `version`: Last update time of the stats meta table (**This is the update time of the stats meta row, not the last analysis time**).

From `mysql.stats_histograms`:
- `is_index`: Whether the statistics are for an index.
- `distinct_count`: Number of distinct values.
- `null_count`: Number of NULL values.
- `version`: Last update time of the statistics.(**The real last analysis time**)
- `stats_ver`: Statistics format version(0, 1, 2). 0 means the statistics have not been analyzed yet.

From `mysql.stats_topn`: (only for indexes)
- `value`: TopN value. (With data type)
- `count`: Count of the TopN value.

From `mysql.stats_buckets`: (only for indexes)
- `count`: Number of rows in the bucket.
- `repeats`: Number of times the upper-bound value of the bucket appears in the data.
- `lower_bound`: Lower bound of the bucket.
- `upper_bound`: Upper bound of the bucket.

**However, it is important to note that the comprehensive mode only loads statistics for indexes, not for columns.** For columns, TiDB will load the statistics on-demand when the column is accessed for the first time.

#### Configuration

Because TiDB has two initialization modes, it provides these configuration options to specify which mode to use:

- `lite-init-stats`: Chooses the lightweight statistics initialization mode, defaulting to `true` after `v7.2.0`.
- `force-init-stats`: Forces TiDB to start before fully loading statistics, defaulting to `true` after `v7.2.0`. Enabling this with the comprehensive mode lengthens startup times. Disabling it may cause TiDB to serve queries with incomplete statistics.
- `concurrently-init-stats`: Enables concurrent statistics initialization, defaulting to `true` after `v8.1.0`. This option can speed up the initialization process by loading statistics concurrently. **Since the lightweight mode is already sufficiently fast, this setting is most useful for the comprehensive mode.**

Since these configurations are set in the configuration file and not as system variables, you need to restart the TiDB server to apply the changes. Therefore, ensure you adjust these settings according to your needs before starting the TiDB server.

## Maintain Statistics In Memory

The initialization process appears straightforward: load statistics data from system tables into memory. However, maintaining these statistics in memory is a complex task that involves several challenges:

1. Find a suitable data structure to store statistics data.
2. Manage the memory usage of statistics data to avoid excessive consumption.

### Data Structure

TiDB uses a hierarchical data structure to store statistics data in memory. The structure is as follows:

```go
type Table struct {
	...
	ColAndIdxExistenceMap *ColAndIdxExistenceMap
	HistColl
	Version uint64
	LastAnalyzeVersion uint64
	...
}
```

- `ColAndIdxExistenceMap`: A map that indicates which columns and indexes have real statistics data to avoid unnecessary statistics loading. We will discuss this in my next blog post.
- `HistColl`: A collection of histograms for each column or index. (This is the biggest structure that contains all the statistical data.)
- `version`: Last update time of the stats meta table (**This is the update time of the stats meta row, not the last analysis time**).
- `LastAnalyzeVersion`: The last analysis time of the statistics.

The `ColAndIdxExistenceMap` used to track which columns and indexes have real statistics data. It is a simple map that stores the column or index ID as the key and a boolean value as the value to indicate whether the column or index has been analyzed.

```go
type ColAndIdxExistenceMap struct {
	checked     bool
	colAnalyzed map[int64]bool
	idxAnalyzed map[int64]bool
}

func (m *ColAndIdxExistenceMap) HasAnalyzed(id int64, isIndex bool) bool {
	if isIndex {
		analyzed, ok := m.idxAnalyzed[id]
		return ok && analyzed
	}
	analyzed, ok := m.colAnalyzed[id]
	return ok && analyzed
}
```

Whenever you need to determine if an index or column has actual statistics data, you can use the `HasAnalyzed` method to check this map.

> Note: We could optimize memory usage by implementing a bitmap.

The `HistColl` structure serves as the central data structure for storing statistical information.

```go
type HistColl struct {
    ...
	columns    map[int64]*Column
	indices    map[int64]*Index
	PhysicalID int64
	RealtimeCount int64
	ModifyCount   int64
	StatsVer int
	Pseudo         bool
	...
}
```

It maintains two maps:
- `columns`: Maps column IDs to column statistics
- `indices`: Maps index IDs to index statistics

These maps enable quick lookups of statistics by ID. It also contains other table-level statistics, such as `RealtimeCount`, `ModifyCount`, `StatsVer`, and `Pseudo`.

For columns, each entry contains extensive statistical information:

```go
type Column struct {
	LastAnalyzePos types.Datum
	...
	TopN           *TopN
	...
	Info           *model.ColumnInfo
	Histogram

	StatsLoadedStatus
	PhysicalID int64
	StatsVer   int64

	IsHandle bool
}
```

For indexes, the structure is similar:

```go
type Index struct {
	LastAnalyzePos types.Datum
	...
	TopN           *TopN
	...
	Info           *model.IndexInfo
	Histogram

	StatsLoadedStatus
	StatsVer int64
	PhysicalID int64
}
```

As you can see, the main statistical data is stored in the `Histogram` structure, which contains multiple buckets to represent the data distribution.

```go
type Bucket struct {
	Count int64
	Repeat int64
	NDV int64
}

type Histogram struct {
	Tp *types.FieldType
	Bounds  *chunk.Chunk
	Buckets []Bucket

	Scalars   []scalar
	ID        int64
	NDV       int64
	NullCount int64
	LastUpdateVersion uint64

	Correlation float64
}
```

The data structure used to store statistics in memory is complex and hierarchical, which can make it difficult to understand and maintain.

To summarize, we can use this diagram to illustrate the hierarchical data structure used to store statistics in memory:

[![image](https://github.com/user-attachments/assets/8bdb5079-2623-4e2f-9810-76b0b92afa09)](https://www.plantuml.com/plantuml/umla/hLJ1Rjim3BtdAuWSFJJ0W67dTic66cYBeijwANfWBMOnGak69G79Wltx8bDHxLYxzUBGxp5Fxz6Ihgt3plc6PnMZjR36DoOupW00FYqDtsXLglttVMqTwOhkiOKY2yi_Ra_0YMOu5m8_KsThey7Nsdtz8jWTMdUZaGz_w8B-Eujcykj7SzMMgXqfU3E68s8u2Yfei7tfrLxV-Lhj_yUd9LE0OzBqZRQ3_cBPGr5IgxgY4LrgHNjX7xS7MrV8vGe6mPy8sTKDBOtNRaZS6rLl3XFufqDdJoCAMDIrv9M1iNEn1SV9T1-D1NTeoIvMw7mZ_Dgq3r24fxoNUcEWQ8mYNeXIGDu_gldTOGEf6ZYxCwX8XT9Rc22JGUI39QoqjwWLqqMuVgWVaQqN-h1eqn3vk2b7MkMSPTr28G5-rCHgVIg5-6QyLXQAQklrRh4CpqZuQaVEmikhLD56XOnTG6rV2JgUzyFgUVJgcIURBSpsLwlGKUxCBatN4QCB-8ODZhA9dNEmYV8JjOGkuqSqvEAPVvx3rLNuoH_-QLkwQCv58ejvF1DPIdRKJ3ek1MKZI4kMIjLGCdwFQzBAD_mF)

### LRU Stats Cache

Ideally, TiDB would load all statistics at startup, but this is impractical given the large volumes of data. To keep memory usage in check, TiDB relies on an LRU cache that loads and evicts statistics on demand.

The interface of the LRU cache is as follows:

```go

type StatsCacheInner interface {
	Get(tid int64) (*statistics.Table, bool)
	Put(tid int64, tbl *statistics.Table) bool
	Del(int64)
	Cost() int64
	Values() []*statistics.Table
	Len() int
	Copy() StatsCacheInner
	SetCapacity(int64)
	Close()
	TriggerEvict()
}
```

This interface implemented by the `LRU` structure:

```go
type LFU struct {
	cache *ristretto.Cache
	resultKeySet *keySetShard
	cost         atomic.Int64
	closed       atomic.Bool
	closeOnce    sync.Once
}
```

This `LFU` structure is a wrapper of the `ristretto.Cache` structure, which is [a high-performance cache library] in Go. It provides a simple API to manage the cache and handle the eviction of statistics data.

To initialize the `ristretto.Cache`, we need to specify the maximum cost to control memory usage. Additionally, we need to implement three hooks to handle the eviction of statistics data. Each time we put, get, or delete data from the cache, the corresponding hook will be triggered.

```go
// NewLFU creates a new LFU cache.
func NewLFU(totalMemCost int64) (*LFU, error) {
	cost, err := adjustMemCost(totalMemCost)
	if err != nil {
		return nil, err
	}
	...
	result := &LFU{}
	bufferItems := int64(64)
	cache, err := ristretto.NewCache(
		&ristretto.Config{
			NumCounters:        max(min(cost/128, 1_000_000), 10),
			MaxCost:            cost,
			BufferItems:        bufferItems,
			OnEvict:            result.onEvict,
			OnExit:             result.onExit,
			OnReject:           result.onReject,
			...
		},
	)
	if err != nil {
		return nil, err
	}
	result.cache = cache
	result.resultKeySet = newKeySetShard()
	return result, err
}
```

There are three hooks to handle the eviction of statistics data:
- `OnEvict`: Called for every eviction and passes the hashed key, value, and cost to the function.
- `OnExit`: Called whenever a value is removed from cache.
- `OnReject`: Called for every rejection done via the policy.

Whenever table statistics are evicted from the cache, TiDB prunes the data from its in-memory structures. The `OnEvict` hook triggers the `dropMemory` function, which finalizes the eviction and updates the memory usage to reflect the change.

```go
func (s *LFU) dropMemory(item *ristretto.Item) {
	...
	table := item.Value.(*statistics.Table).Copy()
	table.DropEvicted()
	s.resultKeySet.AddKeyValue(int64(item.Key), table)
	after := table.MemoryUsage().TotalTrackingMemUsage()
	s.addCost(after)
	s.triggerEvict()
}
```

In the `dropMemory` function, we first make a copy of the table statistics and then call the `DropEvicted` method to prune the data. This method will drop the TopN and Histogram data and mark the column or index as `AllEvicted`. Basically, it reinitializes the statistics data to free up memory.

```go
func (coll *HistColl) DropEvicted() {
	for _, col := range coll.columns {
		if !col.IsStatsInitialized() || col.GetEvictedStatus() == AllEvicted {
			continue
		}
		col.DropUnnecessaryData()
	}
	for _, idx := range coll.indices {
		if !idx.IsStatsInitialized() || idx.GetEvictedStatus() == AllEvicted {
			continue
		}
		idx.DropUnnecessaryData()
	}
}

func (c *Column) DropUnnecessaryData() {
	if c.StatsVer < Version2 {
		c.CMSketch = nil
	}
	c.TopN = nil
	c.Histogram.Bounds = chunk.NewChunkWithCapacity([]*types.FieldType{types.NewFieldType(mysql.TypeBlob)}, 0)
	c.Histogram.Buckets = make([]Bucket, 0)
	c.Histogram.Scalars = make([]scalar, 0)
	c.evictedStatus = AllEvicted
}
```
You might notice that the dropMemory method includes the newly updated table statistics cost in the cache. Consequently, when the memory usage is cleared, the `OnExit` hook is triggered to reflect this change in memory usage.

```go
func (s *LFU) onExit(val any) {
	...
	s.addCost(-val.(*statistics.Table).MemoryUsage().TotalTrackingMemUsage())
}
```

The `OnReject` hook behaves similarly to the `OnEvict` hook. When triggered, it calls the `dropMemory` function to finalize the eviction and adjust memory usage accordingly.

```go
func (s *LFU) onReject(item *ristretto.Item) {
	...
	s.dropMemory(item)
}
```

When the `dropMemory` function is called, it removes large data structures but also stores basic table statistics in the resultKeySet cache. This preserves essential details like `modify_count`, `count`, and `version` for quick retrieval. TiDB uses two caching layers for statistics: the `LRU` cache for frequently accessed tables with full stats and the `resultKeySet` cache, which is a simple map of table IDs to essential table information. The `Put`, `Get`, and `Del` methods in the LFU structure illustrate how these two layers interact.

```go
func (s *LFU) Put(tblID int64, tbl *statistics.Table) bool {
	cost := tbl.MemoryUsage().TotalTrackingMemUsage()
	s.resultKeySet.AddKeyValue(tblID, tbl)
	s.addCost(cost)
	return s.cache.Set(tblID, tbl, cost)
}
func (s *LFU) Get(tid int64) (*statistics.Table, bool) {
	result, ok := s.cache.Get(tid)
	if !ok {
		return s.resultKeySet.Get(tid)
	}
	return result.(*statistics.Table), ok
}
func (s *LFU) Del(tblID int64) {
	s.cache.Del(tblID)
	s.resultKeySet.Remove(tblID)
}
```

The `Put` method adds the table statistics to the LRU cache and the resultKeySet cache. The `Get` method retrieves the table statistics from the LRU cache and falls back to the resultKeySet cache if the data is not found. The `Del` method removes the table statistics from both caches.

#### Configuration

To configure the LRU cache, you can set the following system variable:

- `tidb_stats_cache_mem_quota`: Specifies the maximum memory usage for the statistics cache. The default value is half of the total memory of the TiDB instance. Recently, the default value has been changed to 20% of the total memory.

Determining the optimal value for this variable can be challenging as it depends on the workload and the size of the statistics data. I will revisit this topic in a future blog post after we have more insights into the on-fly statistics loading feature.

## Conclusion

In this blog post, we covered TiDB’s statistics initialization process and described how TiDB loads data from system tables. We also detailed TiDB’s dual-mode initialization approach and the data structures used to store statistics. Additionally, we examined how TiDB maintains these statistics in memory using an LRU cache to balance performance and resource usage. Future posts will address column-level statistics loading on demand and how TiDB updates its statistics.

[a high-performance cache library]: https://github.com/dgraph-io/ristretto
