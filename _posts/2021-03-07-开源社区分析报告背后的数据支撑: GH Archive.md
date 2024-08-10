---
title: '开源社区分析报告背后的数据支撑: GH Archive'
layout: post

categories: post
tags:
- X-lab
- GitHub
- pingcap
- tidb
- tikv
- GH Archive
---

最近应老板要求看了一个 X-lab 针对 GitHub 开源项目的[分析报告](http://www.x-lab.info/github-analysis-report/#/report)，发现了一个背后非常好用并且强大的[数据集](https://www.gharchive.org/)。所以简单介绍一下这个数据集的收集方式和使用方法。

此博客在 [GitHub](https://github.com/Rustin170506/blog) 上公开发布. 如果您有任何问题或疑问，请在此处打开一个 [issue](https://github.com/Rustin170506/blog/issues).

## 简介

前段时间过年的时候老板给我发来了一个 [X-lab](http://www.x-lab.info/) 实验室做的 GitHub 开源项目的[分析报告](http://www.x-lab.info/github-analysis-report/#/report)想让我看看我们是否可以学习和利用其中的分析方法和数据集。我就看了看这个报告，发现它其实背后是使用了另外一个开源项目 [GH Archive](https://github.com/igrigorik/gharchive.org) 归档的数据集并配合 ClickHouse 生成了分析报告。所以我就简单的看了看这个项目，下面是这个 GitHub 归档项目的实现思路和使用方法。

## GH Archive 设计和实现思路

GH Archive 采取了一个非常暴力但是非常有效的归档方式，他们将 GitHub 所有的事件的都利用 GitHub 提供的 [Event API]( https://api.github.com/events) 爬取了下来，然后按照 JSON 的格式存了下来。

他们使用 ruby 来调用 GitHub API 进行爬取，具体的代码在 [crawler.rb](https://github.com/igrigorik/gharchive.org/blob/master/crawler/crawler.rb)。有效的代码不超过一百行，所以就不具体展开解读了。**主要就是调用 API 并且将数据按照日期整理为 JSON 格式的文件。**

## 同步至 BigQuery 数据集

GH Archive 通过爬取的方式整理好数据之后提供了 JSON 数据集的下载服务，我们可以通过 HTTP 客户端下载这些数据集，例如使用 `wget`:

```sh
wget https://data.gharchive.org/2015-01-01-15.json.gz
```

这样我们就可以下载 2015 年 1 月 1 日 GitHub 所有公开仓库的所有事件。

但是这样的 JSON 数据集分析起来不是很方便，还要配合其他的工具类进行分析。所以 GH Archive 就官方提供了一个免费公开的 Google BigQuery 数据集：githubarchive。

数据集的结构主要还是按照时间范围来划分：
- day（e.g. 20150101）
- month（e.g. 201501）
- year（e.g. 2015）

GH Archive 利用[定时任务](https://github.com/igrigorik/gharchive.org/blob/master/crawler/tasks.cron)每一个小时同步一次数据到 BigQuery 数据集，他们将所有的 event 都构造成了同一个[结构](https://github.com/igrigorik/gharchive.org/blob/master/bigquery/schema.js)。

这样就可以通过 BigQuery 来查询数据了，因为所有的事件的主要信息都存放在 payload 里面，我们可以利用 BigQuery 提供的 [JSON 相关函数](https://cloud.google.com/bigquery/docs/reference/legacy-sql#jsonfunctions)来处理和提取这些字段和内容。

另外考虑到 GitHub 在不断的迭代，可能会对某些事件新增一些字段，所以 GH Archive 提供了一个 other 字段来存储这些新增的字段。

## BigQuery 数据集使用方法

通过以下几个步骤使用这个开放的数据集：
1. [登陆 Google 开发者控制台](https://console.developers.google.com/)
2. 如果没有项目的话创建一个新的项目并且**激活使用 BigQuery API**
3. [打开数据集](https://console.cloud.google.com/bigquery?project=githubarchive&page=project)或者直接在项目中新建查询
4. 执行以下查询：
```sql
/* count of issues opened, closed, and reopened on 2019/01/01 */
SELECT event as issue_status, COUNT(*) as cnt FROM (
SELECT type, repo.name, actor.login,JSON_EXTRACT(payload, '$.action') as event,
FROM `githubarchive.day.20190101`
WHERE type = 'IssuesEvent')
GROUP by issue_status;
```

这样我们就成功的用上了这个公开的 BigQuery 数据集，下面我写几个简单的例子演示一下更多的查询：

### 查询 1: 如何证明 PingCAP 节假日没上班？

取样调查😂，我们查询一下 2021 年 1 月 1 日 TiDB 仓库有没有提交代码即可：

```sql
/* 2021 年 1 月 1 日 TiDB 仓库代码 Push 次数 */
SELECT COUNT(*) as cnt
FROM `githubarchive.day.20210101`
WHERE type = 'PushEvent' and org.login = 'pingcap' and repo.name = 'pingcap/tidb';
```

结果为：`cnt: 0`

### 查询 2: 想要 TiDB 2020 年的贡献者名单？

我们查询 2020 年给 TiDB 提过 PR 的人员名单：

```sql
/* 2020 年给 TiDB 提过 PR 的人员名单（只要提交就算） */
SELECT DISTINCT actor.login as name,
FROM `githubarchive.year.2020`
WHERE type = 'PullRequestEvent' and org.login = 'pingcap' and repo.name = 'pingcap/tidb' and JSON_EXTRACT(payload, '$.action') = '"opened"';
```

结果为： 247 个人

### 查询 3：TiDB 社区[新的协作机器人](https://github.com/ti-chi-bot)上线后，3 月份至今还有多少人在手动合并 TiDB 的 PR？

我们查询 3 月份至今所有 Push 事件的 actor 即可：

```sql
/* 2021 年 3 月还在手动合并 PR 的人员名单 */
SELECT DISTINCT actor.login
FROM `githubarchive.month.202103`
WHERE type = 'PushEvent' and org.login = 'pingcap' and repo.name = 'pingcap/tidb';
```

结果为：除了机器人外，还有 6 个人。

```json
[
  {
    "login": "AndreMouche"
  },
  {
    "login": "qw4990"
  },
  {
    "login": "hanfei1991"
  },
  {
    "login": "ti-chi-bot"
  },
  {
    "login": "AilinKid"
  },
  {
    "login": "tiancaiamao"
  },
  {
    "login": "eurekaka"
  }
]
```

有了这些所有事件的归档，我们可以从很多的角度去分析社区的运行状况。**同时因为 payload 的信息非常全面，我们甚至能够通过项目的 ID 或者用户的 ID 去仔细查询分析所有的事件，不会有数据因为项目重命名或者迁移组织而无法查询。**

## 与其他工具结合

因为 GH Archive 存储的是 JSON 格式，所以我们完全可以将数据导入到其他工具中进行分析处理，比如 X-lab 就将数据导入了他们自己的 ClickHouse，并且按照他们的报告需求对数据进行了[分析](https://github.com/X-lab2017/github-analysis-report/tree/master/sqls)。另外也有开源组织创建了一个公开的 [ClickHouse 数据集](https://github.com/github-sql/explorer)并且创建了大量的查询和分析样例。

更多与其他工具结合的例子可以参考 GH Archive 官网中的[资源](https://www.gharchive.org/#resources)。

我会继续探索 GH Archive 在 TiDB 社区落地的方案，就目前来看完全可以满足我们对社区运营的数据支撑需求。后续我也会更新 GH Archive 在 TiDB 社区应用的场景和具体落地方案。

### 参考链接

[github-analysis-report](http://www.x-lab.info/github-analysis-report/#/report)

[gharchive.org](https://www.gharchive.org/)

[Everything You Always Wanted To Know
About GitHub](https://gh.clickhouse.tech/explorer/)
