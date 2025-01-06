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
- TopN values (most frequent values)
- Histograms (data distribution)
- Number of Distinct Values (NDV)
- Other statistical metrics

These statistics will be stored in some system tables, such as `mysql.stats_meta`, `mysql.stats_top_n`, `mysql.stats_histograms`, and `mysql.stats_buckets`.

When TiDB starts, it must load these statistics from the system tables into memory, a process known as "statistics initialization." This step is crucial as it equips the query optimizer with the necessary statistical information to generate optimal execution plans.

In this blog post, we will discuss the initialization process of TiDB statistics.

# Initialization Process

## What Statistics Data Gets Loaded?

In TiDB, the statistics data can be divided into three parts:

1. Basic information : `modify_count`, `row_count`, `version` from `mysql.stats_meta` table.
2. TopN information: `value`, `count` from `mysql.stats_topn` table.
3. Histogram information
   1. Histogram meta: `distinct_count`, `null_count`, `modify_count`, `version`, `stats_ver` from `mysql.stats_histograms` table.
   2. Histogram buckets: `count`, `repeats`, `lower_bound`, `upper_bound` from `mysql.stats_buckets` table.

## How TiDB Loads Statistics Data?

Since TiDB maintains extensive statistical data for each column (including 100 TopN values and 256 histogram buckets), loading all statistics at once can be resource-intensive. To address this, TiDB provides two initialization approaches:

1. A lightweight mode that quickly loads only essential statistics
2. A comprehensive mode that loads all statistical data but takes longer to complete

This dual-mode approach allows TiDB to balance between startup speed and statistical completeness based on specific needs. But it also brings extra complexity and maintenance burden.

### Lightweight Mode

In lightweight mode, TiDB loads only essential statistical information from two system tables:

From `mysql.stats_meta`:
- `modify_count`: Number of row modifications
- `row_count`: Total number of rows
- `version`: Statistics last update time

From `mysql.stats_histograms`:
- `stats_ver`: Statistics format version(0, 1, 2)
- `version`: Statistics last update time
- `distinct_count`: Number of distinct values
- `null_count`: Number of NULL values

The purpose of loading this basic information is to provide the `modify_count` and `row_count` metrics to other modules, allowing them to track both the total number of rows in the table and how many rows have been modified.

### Comprehensive Mode

In comprehensive mode, TiDB loads all statistical data from the system tables. This includes:

From `mysql.stats_meta`:
- `modify_count`: Number of row modifications
- `row_count`: Total number of rows
- `version`: Statistics last update time

From `mysql.stats_histograms`:
- `stats_ver`: Statistics format version(0, 1, 2)
- `version`: Statistics last update time
- `distinct_count`: Number of distinct values
- `null_count`: Number of NULL values

From `mysql.stats_topn`: (only for indexes)
- `value`: TopN value
- `count`: Count of the TopN value

From `mysql.stats_buckets`: (only for indexes)
- `count`: Number of rows in the bucket
- `repeats`: Number of repeated rows in the bucket
- `lower_bound`: Lower bound of the bucket
- `upper_bound`: Upper bound of the bucket

**However, it is important to note that the comprehensive mode only loads statistics for indexes, not for columns.** For columns, TiDB will load the statistics on-demand when the column is accessed for the first time.

# Maintain Statistics In Memory

It seems the initialization process is straightforward: load statistics data from system tables into memory. However, maintaining these statistics in memory is a complex task that involves several challenges:

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
	TblInfoUpdateTS uint64
	...
}
```

- `ColAndIdxExistenceMap`: A map that indicates which columns and indexes have real statistics data to avoid unnecessary statistics loading.
- `HistColl`: A collection of histograms for each column or index. (This is a big structure that contains all the statistical data.)
- `Version`: The version of the statistics data.
- `LastAnalyzeVersion`: The version of the last analyzed statistics data.
- `TblInfoUpdateTS`: The timestamp of the last table information update.

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

	/*
		Fields below are only used in a query, like for estimation, and they will be useless when stored in
		the stats cache. (See GenerateHistCollFromColumnInfo() for details)
	*/

	CanNotTriggerLoad bool
	// Idx2ColUniqueIDs maps the index id to its column UniqueIDs. It's used to calculate the selectivity in planner.
	Idx2ColUniqueIDs map[int64][]int64
	// ColUniqueID2IdxIDs maps the column UniqueID to a list index ids whose first column is it.
	// It's used to calculate the selectivity in planner.
	ColUniqueID2IdxIDs map[int64][]int64
	// UniqueID2colInfoID maps the column UniqueID to its ID in the metadata.
	UniqueID2colInfoID map[int64]int64
	// MVIdx2Columns maps the index id to its columns by expression.Column.
	// For normal index, the column id is enough, as we already have in Idx2ColUniqueIDs. But currently, mv index needs more
	// information to match the filter against the mv index columns, and we need this map to provide this information.
	MVIdx2Columns map[int64][]*expression.Column
}
```

For each column, it contains the following information:

```go
type Column struct {
	LastAnalyzePos types.Datum
	CMSketch       *CMSketch // Only for stats version 1.
	TopN           *TopN
	FMSketch       *FMSketch
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



