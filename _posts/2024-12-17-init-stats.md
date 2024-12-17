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

# Background

Statistics collection is a critical component of any modern database system, serving as the foundation for query optimization. TiDB follows this industry standard practice.

In TiDB's query optimization process, statistics play a fundamental role - they are the sole source of information used to estimate query costs and determine the most efficient execution plan for each query.

TiDB collects several types of statistics for each table, including:
- TopN values (most frequent values)
- Histograms (data distribution)
- Number of Distinct Values (NDV)
- Other statistical metrics

These statistics will be stored in some system tables, such as `mysql.stats_top_n`, `mysql.stats_meta`, `mysql.stats_histograms`, and `mysql.stats_buckets`.

When TiDB starts up, it needs to load these statistics from the system tables into memory - a process known as "statistics initialization". This initialization step is crucial as it provides the query optimizer with the statistical information needed to generate optimal execution plans.

In this blog post, we will discuss the initialization process of TiDB statistics.

# Initialization Process

Before diving into the details, let's understand how TiDB initializes statistics. Since TiDB maintains extensive statistical data for each column (including 100 TopN values and 256 histogram buckets), loading all statistics at once can be resource-intensive. To address this, TiDB provides two initialization approaches:

1. A lightweight mode that quickly loads only essential statistics
2. A comprehensive mode that loads all statistical data but takes longer to complete

This dual-mode approach allows TiDB to balance between startup speed and statistical completeness based on specific needs. But it also brings extra complexity and maintenance burden.

## What Statistics Data Gets Loaded?

In TiDB, the statistics data can be divided into three parts:

1. Basic information : `modify_count`, `row_count`, `version` from `mysql.stats_meta` table.
2. TopN information: `value`, `count` from `mysql.stats_topn` table.
3. Histogram information
   1. Histogram meta: `distinct_count`, `null_count`, `modify_count`, `version` from `mysql.stats_histograms` table.
   2. Histogram buckets: `count`, `repeats`, `lower_bound`, `upper_bound` from `mysql.stats_buckets` table.



