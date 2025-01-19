---
title: 'TiDB Statistics: Understanding the Initialization Process'
layout: post

categories: post
tags:
- Golang
- TiDB
- SQL
- Statistics
---

# Statistics

Statistics collection is a crucial component of modern database systems, forming the backbone of query optimization. In TiDB, statistics are indispensable, serving as the sole source of information for estimating query costs and selecting the most efficient execution plan.

TiDB collects several types of statistics for each table, including:
- TopN values (most frequent values to reflect data skewness)
- Histograms (data distribution)
- Number of Distinct Values (NDV)
- Other statistical metrics

These statistics will be stored in some system tables, such as `mysql.stats_meta`, `mysql.stats_top_n`, `mysql.stats_histograms`, and `mysql.stats_buckets`.

When TiDB starts, it must load these statistics from the system tables into memory, a process known as "statistics initialization." This step is crucial as it equips the query optimizer with the necessary statistical information to generate optimal execution plans.

In this blog post, we will discuss the initialization process of TiDB statistics.

# Initialization Process

## What Statistics Data Gets Loaded?

In TiDB, the statistics data can be divided into three parts:

1. Basic information: `modify_count`, `row_count`, `version` from `mysql.stats_meta` table.
2. TopN information: `value`, `count` from `mysql.stats_topn` table.
3. Histogram meta: `is_index`, `distinct_count`, `null_count`, `version`, `stats_ver` from `mysql.stats_histograms` table.
4. Histogram buckets: `count`, `repeats`, `lower_bound`, `upper_bound` from `mysql.stats_buckets` table.

When TiDB starts, it will load these statistics data from the system tables into memory.

## How TiDB Loads Statistics Data?

Since TiDB maintains extensive statistics for each column and index (including 100 TopN values and 256 histogram buckets), loading all statistics at once can be resource-intensive. To address this, TiDB provides two initialization approaches:

1. A lightweight mode that quickly loads only essential statistics
2. A comprehensive mode that loads all statistical data but takes longer to complete

This dual-mode approach allows TiDB to balance between startup speed and statistical completeness based on specific needs. But it also brings extra complexity and maintenance burden.

### Lightweight Mode

In lightweight mode, TiDB loads only essential statistical information from two system tables:

From `mysql.stats_meta`:
- `modify_count`: Number of row modifications
- `row_count`: Total number of rows
- `version`: Last update time of the statistics (Note: This is the update time of the stats meta row, not the last analysis time).

From `mysql.stats_histograms`:
- `is_index`: Whether the statistics are for an index.
- `distinct_count`: Number of distinct values.
- `null_count`: Number of NULL values.
- `version`: Last update time of the statistics.(The real last analysis time)
- `stats_ver`: Statistics format version(0, 1, 2).

The purpose of loading this basic information is to provide the `modify_count` and `row_count` metrics to other modules, allowing them to track both the real-time total number of rows in the table and how many rows have been modified since the last statistics collection.

### Comprehensive Mode

In comprehensive mode, TiDB loads all statistical data from the system tables. This includes:

From `mysql.stats_meta`:
- `modify_count`: Number of row modifications
- `row_count`: Total number of rows
- `version`: Last update time of the statistics (Note: This is the update time of the stats meta row, not the last analysis time).

From `mysql.stats_histograms`:
- `is_index`: Whether the statistics are for an index.
- `distinct_count`: Number of distinct values.
- `null_count`: Number of NULL values.
- `version`: Last update time of the statistics.(The real last analysis time)
- `stats_ver`: Statistics format version(0, 1, 2).

From `mysql.stats_topn`: (only for indexes)
- `value`: TopN value. (With data type)
- `count`: Count of the TopN value.

From `mysql.stats_buckets`: (only for indexes)
- `count`: Number of rows in the bucket.
- `repeats`: Number of times the upper-bound value of the bucket appears in the data.
- `lower_bound`: Lower bound of the bucket.
- `upper_bound`: Upper bound of the bucket.

**However, it is important to note that the comprehensive mode only loads statistics for indexes, not for columns.** For columns, TiDB will load the statistics on-demand when the column is accessed for the first time.

# Maintain Statistics In Memory

The initialization process appears straightforward: load statistics data from system tables into memory. However, maintaining these statistics in memory is a complex task that involves several challenges:

1. Find a suitable data structure to store statistics data.
2. Manage the memory usage of statistics data.

## Data Structure

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

- `ColAndIdxExistenceMap`: A map that indicates which columns and indexes have real statistics data to avoid unnecessary statistics loading.
- `HistColl`: A collection of histograms for each column or index. (This is a big structure that contains all the statistical data.)
- `Version`: The update time of this stats meta row.
- `LastAnalyzeVersion`: The last analysis time of the statistics.

The `ColAndIdxExistenceMap` used to track which columns and indexes have real statistics data. It is a simple map that stores the column or index ID as the key and a boolean value as the value.

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

The `HistColl` structure is the core data structure that stores the statistics data. It contains the following information:

```go
type HistColl struct {
    ...
	columns    map[int64]*Column
	indices    map[int64]*Index
	PhysicalID int64
	RealtimeCount int64
	ModifyCount   int64
	StatsVer int
	Pseudo         bool // Pseudo is true means this HistColl is a pseudo statistics.
	...
}
```

It contains two maps: `columns` and `indices`.

Each column includes the following information:

```go
type Column struct {
	LastAnalyzePos types.Datum
	CMSketch       *CMSketch // Only for stats version 1.
	TopN           *TopN
	FMSketch       *FMSketch // Disabled by default.
	Info           *model.ColumnInfo
	Histogram

	StatsLoadedStatus
	PhysicalID int64
	StatsVer   int64

	IsHandle bool
}
```

For each index, it contains the following information:

```go
type Index struct {
	LastAnalyzePos types.Datum
	CMSketch       *CMSketch
	TopN           *TopN
	FMSketch       *FMSketch
	Info           *model.IndexInfo
	Histogram

	StatsLoadedStatus
	StatsVer int64
	PhysicalID int64
}
```

And the `Histogram` structure is as follows:

```go
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

It also contains a `Bucket` structure:

```go
type Bucket struct {
	Count int64
	Repeat int64
	NDV int64
}
```

Above is the data structure used to store statistics data in memory. It is complex and hierarchical. Sometimes it is difficult to understand and maintain.

## LRU Stats Cache

To manage the memory usage of statistics data, TiDB introduces an LRU cache to store the statistics data.

The interface of the LRU cache is as follows:

```go

type StatsCacheInner interface {
	// Get gets the cache.
	Get(tid int64) (*statistics.Table, bool)
	// Put puts a cache.
	Put(tid int64, tbl *statistics.Table) bool
	// Del deletes a cache.
	Del(int64)
	// Cost returns the memory usage of the cache.
	Cost() int64
	// Values returns the values of the cache.
	Values() []*statistics.Table
	// Len returns the length of the cache.
	Len() int
	// Copy returns a copy of the cache
	Copy() StatsCacheInner
	// SetCapacity sets the capacity of the cache
	SetCapacity(int64)
	// Close stops the cache
	Close()
	// TriggerEvict triggers the cache to evict some items
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

This `LFU` structure is a wrapper of the `ristretto.Cache` structure, which is a high-performance cache library in Go.

Every time the table statistics get ejected or evicted from the cache, we will evict some items from the table's statistics. The eviction policy is based on the LRU algorithm.

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
```

Basically, we just drop TopN, Histogram and mark the column or index as `AllEvicted`.

But you may also notice that the `LRU` cache contains another cache layer called `resultKeySet`. This is full cache of the table's statistics.
The reason we need this cache is that even we evict the table's statistics from the `ristretto.Cache`, we still need to know the basic information of the table, such as `modify_count`, `row_count`, `version`, etc.
These information are stored in the `resultKeySet` cache.
