---
title: 教程2：控制您的数据
description: 了解如何在Observability Insights中筛选、分组和控制查询结果。
source-git-commit: 09c8edf64e744ff0c16acd0d5905fc6796490f82
workflow-type: tm+mt
source-wordcount: '336'
ht-degree: 0%
---

## 教程2：控制您的数据 {#tutorial-2-control-your-data}

在本教程中，您将学习将原始计数转换为答案的函数和子句：非重复计数、最新值、百分比、百分点、通配符、排序和多维分组。

### 使用`uniqueCount`对非重复值进行计数

每个服务有多少台主机和不同的跟踪？

```sql
SELECT uniqueCount(host) AS hosts, uniqueCount(traceId) AS traces FROM Transaction FACET service
```

`uniqueCount`是精确的，而不是估计值。

要&#x200B;**列出**&#x200B;不同的值本身（NRQL调用`uniques()`），请对属性进行分面：

```sql
SELECT count(*) AS requests FROM Transaction FACET host LIMIT 100
```

### 查找最新值和最早值

`latest(attr)`返回最近的值，而`earliest(attr)`返回最旧的值。 它们可用于“当前状态”表：

```sql
SELECT latest(value) AS currentCpu FROM CpuUtilization FACET host
```

```sql
SELECT earliest(timestamp) AS firstSeen, latest(timestamp) AS lastSeen FROM Transaction FACET service
```

### 计算百分比

`percentage(*, WHERE condition)`返回符合条件的行的共享。 这是错误率：

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction
```

按服务跟踪此趋势：

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

并跟踪慢速请求的份额：

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction FACET service
```

### 了解`percentile`的延迟

平均数隐藏离群值。 百分位数显示大多数用户体验的内容(p50)以及不幸用户体验的内容(p95、p99)：

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

>[!TIP]
>
> `Transaction.duration`的p50、p95和p99通过长窗口从汇总中提供，因此每周查看时它们保持较快。

### 使用标签使结果可读

为包含空格的标签使用带单引号的`AS`：

```sql
SELECT count(*) AS 'Requests', percentile(duration, 95) AS 'p95 (ms)' FROM Transaction FACET service
```

>[!NOTE]
>
> 别名仅适用于`SELECT`项。 Facet列始终使用属性名称进行标记。

### 匹配模式和列表

`LIKE`将`%`用于任意数量的字符，将`_`用于唯一一个字符：

```sql
SELECT count(*) AS requests FROM Transaction WHERE urlPath LIKE '/content/%.html' FACET urlPath LIMIT 20
```

排除具有`NOT LIKE`的静态资源：

```sql
SELECT count(*) AS requests FROM Transaction WHERE urlPath NOT LIKE '%.js' AND urlPath NOT LIKE '%.css' AND urlPath NOT LIKE '%.png' FACET urlPath LIMIT 20
```

与具有`IN`和`NOT IN`的列表匹配（最多100个值）：

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.method IN ('POST', 'PUT', 'DELETE') FACET service
```

>[!NOTE]
>
> sql没有正则表达式(`RLIKE`)。 将`LIKE`模式与`OR`合并，或使用`IN`。

### 使用`ORDER BY`排序和排名

默认情况下，多面结果按第一个聚合排序。 使用`ORDER BY`按其他条件排名，例如p95最慢的操作：

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

或者先使用最少服务：

```sql
SELECT count(*) AS requests FROM Transaction FACET service ORDER BY requests ASC LIMIT 10
```

> **提示**
>
> 没有`HAVING`。 要仅查找“忙”操作，请按照`requests`排名，并保留前N位，或者添加可消除噪音的`WHERE`（例如`WHERE http.method != 'OPTIONS'`）。

### 按多个属性分组

`FACET`最多接受五个属性。 每个组合将变成一个行或系列：

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode >= 400 FACET service, http.method, http.statusCode LIMIT 30
```

### 查看过去的窗口

没有`COMPARE WITH`。 在功能板和浏览中，更改&#x200B;**时间选取器**。 当您调用没有明确范围的API时，`SINCE … UNTIL …`会为您提供过去的任何窗口：

```sql
SELECT count(*) AS lastWeekSameHour FROM Transaction SINCE 169 hours ago UNTIL 168 hours ago
```
