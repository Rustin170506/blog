---
title: 'TiDB Statistics: Sync Load and Async Load'
date: 2025-02-05
description: 'Sync Load and Async Load are two ways to load statistics in TiDB. This article introduces the differences between them and explains how they work.'
tags: ["Golang", "TiDB", "SQL", "Statistics"]
categories: ["Database"]
---

# Sync Load and Async Load

In my previous article, I introduced the statistics in TiDB and how it initializes them. However, there is an issue: even with comprehensive initialization, statistics may still be missing for columns that are not indexed. Additionally, after initialization, statistics may be evicted from memory due to memory pressure. In this article, I will introduce two methods to load statistics in TiDB on the fly: Sync Load and Async Load.
