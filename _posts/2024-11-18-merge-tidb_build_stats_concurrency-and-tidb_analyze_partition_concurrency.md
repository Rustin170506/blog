---
title: 'Simplifying TiDB Statistics Collection: Unifying Concurrency Controls'
layout: post

categories: post
tags:
- Golang
- TiDB
- SQL
- Statistics
---

# Background

TiDB provides the `analyze table table_name` command to generate statistics for tables. When analyzing partitioned tables, TiDB processes each partition independently and in parallel. Once analysis completes, TiDB aggregates the individual partition statistics into a single global statistics object for the entire table.

The `analyze table` command has two concurrency-related parameters:

- `tidb_build_stats_concurrency`: Determines how many partitions can be processed simultaneously during statistics collection
- `tidb_analyze_partition_concurrency`: Controls parallel workers for saving partition statistics

However, these parameters have several drawbacks:

- The name `tidb_build_stats_concurrency` is imprecise - it actually governs parallel processing of partitions/tables rather than statistics building itself
- Having two separate concurrency controls is unnecessarily complex and makes it difficult for users to understand and predict the performance impact of their settings

In this blog post, we will discuss the issues with the current parameters and propose a solution to merge them into one.

# Test Result - Small Partition Table

To evaluate the performance impact of these parameters, I benchmarked an `analyze table` command on a partitioned table containing 100 partitions and 30 million rows. Here are the results:

> **Note:** Test environment: 3 TiKV nodes and 3 TiDB nodes, each with 16 cores and 32GB RAM.

| `tidb_build_stats_concurrency` | `tidb_analyze_partition_concurrency` | `analyze table` Time |
|--------------------------------|--------------------------------------|----------------------|
| 2                              | 2                                    | 6 min 8.61 sec       |
| 15                             | 2                                    | 2 min 55.71 sec      |
| 2                              | 15                                   | 6 min 4.41 sec       |
| 15                             | 15                                   | 2 min 26.63 sec      |

The benchmark results clearly show that `tidb_build_stats_concurrency` has a much larger impact on overall execution time than `tidb_analyze_partition_concurrency`.

This makes sense given the small partition sizes (300k rows per partition) - analyzing each partition completes quickly and generates minimal statistics data to save. As a result, the parallel statistics saving controlled by `tidb_analyze_partition_concurrency` isn't a performance bottleneck, explaining its minimal impact on total execution time.

The following sections will prove this point by showing the `analyze_jobs` table.

## Analysis

Let's take a closer look at the `analyze table` command execution details.

TiDB records the execution details of the `analyze table` command in the `mysql.analyze_jobs` table. The following is an example of the `analyze_jobs` table:

```sql
SELECT table_schema,
       table_name,
       partition_name,
       job_info,
       processed_rows,
       start_time,
       end_time,
       state
FROM mysql.analyze_jobs
LIMIT 3;

+------------+----------+--------------+-------------------------------------------------------------------------------------------------------------+--------------+-------------------+-------------------+--------+
|table_schema|table_name|partition_name|job_info                                                                                                     |processed_rows|start_time         |end_time           |state   |
+------------+----------+--------------+-------------------------------------------------------------------------------------------------------------+--------------+-------------------+-------------------+--------+
|test        |test_table|p18           |auto analyze table all indexes, columns id, part_id with 256 buckets, 100 topn, 1 samplerate                 |413000        |2024-11-18 15:53:04|2024-11-18 15:53:05|finished|
|test        |test_table|p6            |auto analyze table all indexes, columns id, part_id with 256 buckets, 100 topn, 0.2722772277227723 samplerate|425000        |2024-11-18 15:53:05|2024-11-18 15:53:06|finished|
|test        |test_table|p0            |auto analyze table all indexes, columns id, part_id with 256 buckets, 100 topn, 0.2736318407960199 samplerate|434000        |2024-11-18 15:53:05|2024-11-18 15:53:06|finished|
+------------+----------+--------------+-------------------------------------------------------------------------------------------------------------+--------------+-------------------+-------------------+--------+

```

Let's analyze the overall timeline from when statistics collection begins to when the data is persisted to storage. This will give us insight into both the collection and persistence phases of the operation.

| `tidb_build_stats_concurrency` | `tidb_analyze_partition_concurrency` | partition | Start Time          | End Time            |
|--------------------------------|--------------------------------------|-----------|---------------------|---------------------|
| 2                              | 2                                    | p18       | 2024-11-22 11:31:27 | 2024-11-22 11:31:36 |
| 15                             | 2                                    | p18       | 2024-11-22 12:37:35 | 2024-11-22 12:38:14 |



Examining the `start_time` and `end_time` columns reveals that individual partition processing times remain consistent across different parameter configurations, with all partitions completing within a 10-second window. This consistent timing indicates that the overall performance differences we observed cannot be explained by variations in how long it takes to process each partition.

However, examining the number of concurrent partition operations reveals that `tidb_build_stats_concurrency` directly controls the degree of parallelism in partition processing.

```sql
SELECT COUNT(*) AS record_count
FROM mysql.analyze_jobs
WHERE start_time BETWEEN '2024-11-22 11:31:00' AND '2024-11-22 11:31:59';
```

| `tidb_build_stats_concurrency` | `tidb_analyze_partition_concurrency` | record_count |
|--------------------------------|--------------------------------------|--------------|
| 2                              | 2                                    | 20           |
| 15                             | 2                                    | 29           |


The analysis reveals that increasing `tidb_build_stats_concurrency` enables more partitions to be processed concurrently. Since statistics persistence is not a performance bottleneck, this higher degree of partition-level parallelism directly translates to faster overall statistics collection.

This finding explains the significant performance impact of `tidb_build_stats_concurrency` compared to the relatively minor effect of `tidb_analyze_partition_concurrency`.

# Test Result - Large Partition Table

