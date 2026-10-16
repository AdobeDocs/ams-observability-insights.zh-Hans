---
title: SQL引用
description: 引用可观察性分析支持的SQL语法、子句和函数。
source-git-commit: 898e474e3ab7914f8de2df286c67ac046d3b22b0
workflow-type: tm+mt
source-wordcount: '1504'
ht-degree: 11%
---

## SQL引用 {#sql-reference}

本参考介绍了SQL语法、子句、函数和运算符。 展开每个条目以查看详细信息和示例。

### 查询组件

```text
SELECT <select-item> [, <select-item> ...]
FROM   <Entity>
[WHERE      <condition>]
[FACET      <attribute> [, <attribute> ...]]          -- up to 5
[SINCE      <n> <unit> [AGO]]
[UNTIL      NOW | <n> <unit> [AGO]]
[TIMESERIES AUTO | MAX | <n> <unit>]
[ORDER BY   <alias-or-selected-name> [ASC | DESC]]
[LIMIT      <1..5000>]

<select-item> := <function>(<args>) [FILTER (WHERE <condition>)] [AS <alias>]
               | <attribute> [AS <alias>]
<unit>        := second | seconds | minute | minutes | hour | hours | day | days | week | weeks
```

>[!NOTE]
>
> 上述`--`注释仅为文档。 SQL本身不接受注释。

### 必需条款

#### `SELECT` — 要返回的内容

+++查看详细信息

`SELECT`列出了&#x200B;**原始属性** （事件查询）或&#x200B;**聚合器函数** （聚合查询）。 你不能把两者混在一起。

| 表单 | 示例 | 结果 |
| ------------ | -------------------------------------------------- | ------------------------------------------------- |
| 属性 | `SELECT timestamp, name, duration` | 单个事件的行数（默认为100） |
| 聚合 | `SELECT count(*), average(duration)` | 每个聚合一个值（每个方面/时段） |
| 带有别名 | `SELECT count(*) AS requests` | 列标签`requests` |
| 带引号的别名 | `SELECT percentile(duration, 95) AS 'p95 latency'` | 带空格的标签 |

事件列表：

```sql
SELECT timestamp, service, name, duration, status, traceId FROM Transaction WHERE status = 'Error' LIMIT 50
```

多个聚合：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, max(duration) AS maxMs FROM Transaction
```

规则：

- 不支持`SELECT *`。 列出所需的属性。
- 不支持属性或聚合(`sum(a) / count(*)`)之间的数学运算。 使用[公式小组件](#86-combine-queries-with-formula-widgets)。
- 混合属性和聚合失败，因为&#x200B;_原始属性不能与聚合混合；请使用FACET对结果_&#x200B;进行分组。

+++

#### `FROM` — 要查询的实体

+++查看详细信息

`FROM`从[数据字典](#data-dictionary)中恰好为&#x200B;**一个**&#x200B;实体命名。 实体名称不区分大小写。

```sql
SELECT count(*) FROM Transaction
```

```sql
FROM Span SELECT count(*) FACET kind
```

每个实体都有一个固定的范围。 例如，`Transaction`始终是服务器跨度，`CpuUtilization`始终是CPU忙碌量度，其中`value`已转换为百分比。 不能在单个查询中查询多个实体。 若要合并实体，请使用[公式小组件](#86-combine-queries-with-formula-widgets)。

+++

### 可选子句

#### `AS` — 标记列

+++查看详细信息

使用`AS`为聚合或属性提供一个可读的名称。 别名将变为列标题、图例标签以及您在`ORDER BY`中引用的名称。

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET service ORDER BY p95 DESC LIMIT 10
```

为带有空格的标签使用单引号字符串：

```sql
SELECT percentile(duration, 95) AS 'p95 latency (ms)' FROM Transaction TIMESERIES AUTO
```

+++

#### `FACET` — 组结果

+++查看详细信息

`FACET`最多按&#x200B;**5**&#x200B;个属性（如SQL `GROUP BY`）对结果分组。

```sql
SELECT count(*) FROM Transaction FACET service, http.method, http.statusCode LIMIT 30
```

| 查询 | `LIMIT n`的含义 | 默认 |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| 不带`TIMESERIES`的`FACET` | 行数 | 10 |
| `FACET`与`TIMESERIES` | **前n个系列**，按整个窗口中的第一个聚合的总和排名。 这些系列的每一段时间都会保留。 | 所有系列（上限为5,000行） |

多面时间序列，前5项：

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 5
```

Facet值可以是字符串或数字。 空白字符串构成他们自己的组。 通过`WHERE attr != ''`排除它们。

+++

#### `FILTER` — 条件聚合

+++查看详细信息

将`FILTER (WHERE condition)`附加到任何聚合以对该行的子集进行计算。 这使您能够一次性比较多个条件。

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentile(duration, 95) FILTER (WHERE http.method = 'GET') AS getP95 FROM Transaction FACET service LIMIT 20
```

`FILTER`不允许与`apdex`、`rate`或`derivative`一起使用。 `percentage`已采用自己的条件。

+++

#### `LIMIT` — 有多少行或系列

+++查看详细信息

`LIMIT`采用正整数。 超过5000的值将固定为5000。

| 结果形状 | 默认 |
| ------------ | ---------- |
| 广告牌 | 1 |
| Facet | 10 |
| 时间序列 | 5,000行 |
| 事件 | 100 |

```sql
SELECT count(*) FROM Transaction FACET urlPath LIMIT 50
```

>[!TIP]
>
> 始终在查询驱动变量和高基数Facet (`urlPath`， `traceId`， `jcr.path`， `url`)上设置`LIMIT`。

+++

#### `ORDER BY` — 排序结果

+++查看详细信息

`ORDER BY`按&#x200B;**选定的别名或选定的属性**&#x200B;排序。 默认方向为`DESC`。

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

| 结果形状 | 默认顺序 |
| ----------------- | --------------------------------------------------------------------------- |
| 广告牌/彩块化 | 第一个聚合，`DESC` |
| 时间序列 | 时间，`ASC` |
| 事件 | 第一个选择的列`DESC` （因此，将`timestamp`放在“最新优先”的首位） |

规则：

- 引用&#x200B;_ORDER BY &#39;x&#39;未选定的内容失败，必须引用选定的值或别名_。
- 不要将`ORDER BY`添加到`TIMESERIES`查询。 它覆盖图表所需的时间顺序。

+++

#### `SINCE` — 窗口开始

+++查看详细信息

`SINCE n unit [AGO]`设置窗口开始的时间。 单位： `second`、`minute`、`hour`、`day`、`week`（单数或复数）。 `AGO`是可选的。

```sql
SELECT count(*) FROM Transaction SINCE 6 hours ago
```

- 默认为 `SINCE 1 hour ago`。
- 时间范围不能超过&#x200B;**31天**（_时间范围不能超过31天_）。
- 不支持绝对日期(`SINCE '2026-01-01'`)和`TODAY` / `YESTERDAY`。

>[!IMPORTANT]
>
> 在功能板和浏览器中，**时间选取器覆盖** `SINCE`和`UNTIL`。 请参阅[时间窗口优先级](#time-window-precedence)。

+++

#### `TIMESERIES` — 绘制时间段

+++查看详细信息

`TIMESERIES`为每个时段（每个方面值）返回一行，可用于折线图、面积图或条形图。

| 表单 | 分段大小 |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `TIMESERIES 5 minutes` | 正是您所要求的内容（任何单位） |
| `TIMESERIES AUTO` | 大约窗口÷120，舍入到1秒/5秒/10秒/15秒/30秒/1米/5米/10米/15米/3小时/3小时/6小时/12小时/1天。 **从不低于2分钟。** |
| `TIMESERIES MAX` | 1秒钟。 仅需几分钟即可使用。 |

典型的`AUTO`存储段：

| 窗口 | 1小时 | 6小时 | 24小时 | 7天 | 30天 |
| ------ | ----- | ----- | ------ | --- | ---- |
| 分段 | 2分钟 | 5分钟 | 15分钟 | 3小时 | 6小时 |

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction TIMESERIES AUTO
```

```sql
SELECT count(*) FROM Transaction TIMESERIES 1 hour SINCE 1 day ago
```

>[!TIP]
>
> 偏好仪表板上的`AUTO`。 不管时间选择器怎么说，它都能保持大约120点。 它还避免了“碎片”的每主机行，因为主机量度大约每90秒刮取一次。

+++

#### `UNTIL` — 窗口结束

+++查看详细信息

`UNTIL NOW` （默认）或`UNTIL n unit [AGO]`。 将其与`SINCE`结合查看过去的窗口：

```sql
SELECT count(*) AS yesterday FROM Transaction SINCE 2 days ago UNTIL 1 day ago
```

`SINCE`必须早于`UNTIL`。 否则，查询将失败，并显示&#x200B;_SINCE必须早于UNTIL_。

+++

#### `WHERE` — 筛选行

+++查看详细信息

`WHERE`只保留符合条件的行。 将条件与`AND`、`OR`、`NOT`和括号相结合。 请参阅[运算符和条件](#45-operators-and-conditions)。

```sql
SELECT count(*) FROM Transaction WHERE service = 'checkout' AND (status = 'Error' OR http.statusCode >= 500)
```

`WHERE`只能出现一次。 把所有条件都放进去。

+++

### 聚合器函数

| 函数 | 参数 | 返回 | 工作于 |
| -------------------------------------- | ----------------------------------- | ---------------------- | ------------------------------------- |
| [`apdex`](#fn-apdex) | 数字持续时间，阈值`T` | 0-1 | 跟踪实体 |
| [`average` / `avg`](#fn-average) | 数值 | 与输入相同的单位 | 任意 |
| [`count`](#fn-count) | `*`或属性 | 行数 | 任意 |
| [`derivative`](#fn-derivative) | 数值 | 每秒更改 | 任何（需要`TIMESERIES`） |
| [`earliest`](#fn-latest) | 任意[，排名] | 属性值 | 任意 |
| [`latest`](#fn-latest) | 任意[，排名] | 属性值 | 任意 |
| [`max` / `min`](#fn-minmax) | 任何 | 与输入相同的单位 | 任意 |
| [`percentage`](#fn-percentage) | `*`或属性，`WHERE`条件 | % | 任意 |
| [`percentile`](#fn-percentile) | 数值，级别`p` | 与输入相同的单位 | 任意 |
| [`percentOfTotal`](#fn-percentoftotal) | 数值 | % | 任何（无`TIMESERIES`） |
| [`rate`](#fn-rate) | 数值计数器 | 每秒 | 计数器实体（需要`TIMESERIES`） |
| [`stddev`](#fn-stddev) | 数值 | 与输入相同的单位 | 任意 |
| [`sum`](#fn-sum) | 数值 | 与输入相同的单位 | 任意 |
| [`uniqueCount`](#fn-uniquecount) | 任何 | 不同的值 | 任意 |

<a id="fn-apdex"></a>

#### `apdex(attribute, T)`

+++查看详细信息

使用属性单位中的阈值`T`（对于`duration`为毫秒），计算[Apdex](https://www.apdex.org/)分数，从0（失败）到1（满足）。

| 分段 | 条件 | 粗细 |
| ---------- | ------------------------------------ | ------ |
| 已满足 | `duration ≤ T`，不是错误 | 1 |
| 容忍 | `T < duration ≤ 4T`，不是错误 | 0.5 |
| 沮丧 | `duration > 4T`、**或任何错误** | 0 |

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service
```

- 仅适用于跟踪实体(`Transaction`、`AppRequest`、`Span`)。
- `T`必须大于0。 不允许`FILTER`。
- `Transaction`上的`apdex(duration, 500)`在较长的窗口内被汇总加速。

+++

<a id="fn-average"></a>

#### `average(attribute)` / `avg(attribute)`

+++查看详细信息

数值属性的算术平均值。

```sql
SELECT average(duration) AS avgMs FROM Transaction FACET service TIMESERIES AUTO
```

+++

<a id="fn-count"></a>

#### `count(*)` / `count(attribute)`

+++查看详细信息

- `count(*)`计入行。
- `count(attribute)`计算属性为&#x200B;**不是NULL**&#x200B;的行。 对于字符串属性，缺少的值为空白(`''`)而不是NULL，因此仍会计算它们（请参阅[类型处理](#46-type-handling-blank-vs-null)）。

```sql
SELECT count(*) AS requests, count(http.statusCode) AS withStatusCode FROM Transaction
```

+++

<a id="fn-derivative"></a>

#### `derivative(attribute)`

+++查看详细信息

连续存储段（此存储段−上一个存储段）之间`average(attribute)`的每秒更改÷存储段秒数。 结果可以是负数。 将其用于测量仪表，以查看某个值是正在增长还是正在缩减。

```sql
SELECT derivative(value) AS heapGrowthPerSec FROM JvmHeapUsed WHERE host = 'publish-1' TIMESERIES 5 minutes
```

- 需要`TIMESERIES`，并且必须是&#x200B;**仅限**&#x200B;聚合。 不允许`FILTER`。
- 第一个存储段没有前置任务。

+++

<a id="fn-latest"></a>

#### `latest(attribute [, rankAttribute])` / `earliest(…)`

+++查看详细信息

- `latest(attr)`返回最近一行的值。 `earliest(attr)`从最早的行返回值。
- 对于第二个参数，该行由该属性选择，而不是由时间选择。 `latest(traceId, duration)`返回&#x200B;**最慢**&#x200B;请求的跟踪ID。 `earliest(traceId, duration)`返回最快的值。

```sql
SELECT latest(value) AS current FROM MemoryUtilization FACET host
```

```sql
SELECT max(duration) AS slowestMs, latest(traceId, duration) AS slowestTrace FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 10
```

+++

<a id="fn-minmax"></a>

#### `max(attribute)` / `min(attribute)`

+++查看详细信息

最大或最小值。 适用于数字、字符串和时间戳。

```sql
SELECT min(duration) AS fastestMs, max(duration) AS slowestMs, max(timestamp) AS lastSeen FROM Transaction FACET service
```

+++

<a id="fn-percentage"></a>

#### `percentage(*, WHERE condition)`

+++查看详细信息

匹配`condition`的行的共享，从0到100。

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service TIMESERIES AUTO
```

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction
```

`percentage`有自己的条件，因此不需要`FILTER`。

+++

<a id="fn-percentile"></a>

#### `percentile(attribute, p)`

+++查看详细信息

近似第p个百分位数，其中`0 < p ≤ 100`。 每个级别使用一个`percentile`。

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

`Transaction`上的p50、p95和p99在长窗口上向上滚动。

+++

<a id="fn-percentoftotal"></a>

#### `percentOfTotal(attribute)`

+++查看详细信息

每个Facet行占数值属性的&#x200B;**总和**&#x200B;的份额。 行数总计为100%。

```sql
SELECT sum(duration) AS totalMs, percentOfTotal(duration) AS shareOfTime FROM Transaction FACET name LIMIT 15
```

不允许与`TIMESERIES`一起使用。

+++

<a id="fn-rate"></a>

#### `rate(attribute)`

+++查看详细信息

**累积计数器**&#x200B;的每秒速率。 在每个存储桶中，`rate`采用每个系列的最高计数器读数，并将其与前一个存储桶进行比较。 计数器重置的浮点数为0，第一个存储桶为0。

```sql
SELECT rate(value) AS bytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

- 仅适用于计数器实体（在[数据字典](#53-curated-metric-entities)中标记为✔）。
- 需要`TIMESERIES`，并且必须是&#x200B;**仅限**&#x200B;聚合。 不允许`FILTER`。

>[!WARNING]
>
> 始终为`FACET host`，或筛选到单个主机。 许多主机上未分面的`rate()`将比较不同主机的计数器，并生成无意义的数字。 磁盘计数器是按设备计算的，因此`FACET host, device`是最精确的。

+++

<a id="fn-stddev"></a>

#### `stddev(attribute)`

+++查看详细信息

样本标准偏差。 将其与`average`一起使用，以判断延迟的一致程度。

```sql
SELECT average(duration) AS avgMs, stddev(duration) AS stddevMs FROM Transaction FACET name LIMIT 20
```

+++

<a id="fn-sum"></a>

#### `sum(attribute)`

+++查看详细信息

数值属性的总计。

```sql
SELECT sum(callCount) AS calls FROM SpanBreakdown FACET segment LIMIT 20
```

+++

<a id="fn-uniquecount"></a>

#### `uniqueCount(attribute)`

+++查看详细信息

非重复值的&#x200B;**完全**&#x200B;数量。

```sql
SELECT uniqueCount(host) AS hosts, uniqueCount(traceId) AS traces FROM Transaction FACET service
```

+++

函数约束概览：

| 约束 | 功能 |
| ------------------------------------- | ----------------------------------------------------------------------------------------- |
| 需要数值属性 | `average`, `sum`, `percentile`, `stddev`, `percentOfTotal`, `rate`, `derivative`, `apdex` |
| 无`FILTER (WHERE …)` | `apdex`, `rate`, `derivative`, `percentage` |
| 需要`TIMESERIES`，仅聚合 | `rate`, `derivative` |
| 不与`TIMESERIES`一起使用 | `percentOfTotal` |
| 仅跟踪实体 | `apdex` |
| 仅计数器实体 | `rate` |

### 运算符和条件

| 运算符 | 含义 | 示例 |
| ------------------------- | -------------------------------------------------------------------------- | ---------------------------------------- |
| `=` | 等号 | `status = 'Error'` |
| `!=`, `<>` | 不等于 | `kind != 'Internal'` |
| `<`, `<=`, `>`, `>=` | 比较 | `http.statusCode >= 500` |
| `LIKE` | 通配符匹配： `%` =任何字符，`_` =一个字符。 区分大小写。 | `urlPath LIKE '/content/%'` |
| `NOT LIKE` | 否定通配符 | `urlPath NOT LIKE '%.css'` |
| `IN (…)` | 匹配1-100个值中的任意值 | `service IN ('checkout', 'cart')` |
| `NOT IN (…)` | 不匹配任何值 | `http.method NOT IN ('OPTIONS', 'HEAD')` |
| `IS NULL` / `IS NOT NULL` | 空检查 | `http.statusCode IS NOT NULL` |
| `AND`, `OR`, `NOT` | 逻辑 | `NOT (status = 'Ok')` |
| `( … )` | 分组 | `(a = 1 OR b = 2) AND c = 3` |

```sql
SELECT count(*) FROM Transaction WHERE (status = 'Error' OR duration > 2000) AND NOT host LIKE '%dispatcher%' AND http.method IN ('GET', 'POST')
```

> **警告**
>
> `attr = NULL`是一个错误。 使用`attr IS NULL`。

### 类型处理：空白与NULL

目录中的每个属性都有一个类型： `string`、`number`、`boolean`或`timestamp`。 SQL类型检查函数（例如，`average(service)`被拒绝）。 不需要或没有类型转换函数。

> **重要**
>
> **Blank与NULL。** 大多数字符串属性来自OpenTelemetry属性映射，其中&#x200B;_missing_&#x200B;键读取为空字符串`''`，而不是`NULL`。 要仅保留带有属性的范围，请使用&#x200B;**`attr != ''`**&#x200B;而不是`IS NOT NULL`。 数值属性（如`http.statusCode`）不存在时实际上是`NULL`，因此`IS NOT NULL`适用于它们。

```sql
SELECT count(*) AS jcrCalls FROM Span WHERE jcr.operation != '' FACET jcr.operation
```

### 将SQL转换为SQL

| SQL | SQL |
| ------------------------------------------------------------ | ------------------------------------------------------------------- |
| `WHERE ts > now() - INTERVAL 1 HOUR` | 无任何内容；时间选择器（或`SINCE 1 hour ago`）设置窗口 |
| `GROUP BY service` | `FACET service` |
| `GROUP BY toStartOfInterval(ts, INTERVAL 5 MINUTE), service` | `FACET service TIMESERIES 5 minutes` |
| `quantile(0.95)(duration)` | `percentile(duration, 95)` |
| `countIf(status = 'Error')` | `count(*) FILTER (WHERE status = 'Error')` |
| `100 * countIf(x) / count()` | `percentage(*, WHERE x)` |
| `count(DISTINCT host)` | `uniqueCount(host)` |
| `argMax(traceId, duration)` | `latest(traceId, duration)` |
| `x / sum(x) OVER ()` | `percentOfTotal(x)` |
| `ORDER BY p95 DESC LIMIT 20` | `ORDER BY p95 DESC LIMIT 20` |
| `HAVING count() > 100` | _不支持_。 改为排序和`LIMIT`。 |
| `JOIN` /子查询 | _不支持_。 使用&#x200B;**公式**&#x200B;构件。 |
| `CASE WHEN` | _不支持_。 使用`FILTER`列，或`WHERE … LIKE` + `FACET`。 |

### 将现有功能板导入分析

如果您知道现有查询，下面是迁移详细信息

| 现有查询 | SQL | 注释 |
| ------------------------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `FROM Transaction SELECT count(*)` | 相同 | 支持`FROM`第一顺序 |
| `SINCE 1 day ago` | 相同 | 在仪表板上忽略；选取器将获胜 |
| `TIMESERIES AUTO` / `MAX` / `5 minutes` | 相同 | `AUTO`楼层为2分钟 |
| `FACET appName` | `FACET appName` / `FACET service` |                                                                                  |
| `percentage(count(*), WHERE error IS true)` | `percentage(*, WHERE status = 'Error')` |                                                                                  |
| `filter(count(*), WHERE …)` | `count(*) FILTER (WHERE …)` | SQL标准表单 |
| `apdex(duration, t: 0.5)` | `apdex(duration, 500)` | `duration`在&#x200B;**毫秒**&#x200B;内 |
| `latest(x)` / `earliest(x)` | 相同 | 加上排名表单： `latest(x, rankAttr)` |
| `uniqueCount(x)` | 相同 | 精确 |
| `rate(count(*), 1 minute)` | `count(*)`与`TIMESERIES 1 minute` | SQL `rate()`仅适用于计数器 |
| `derivative(x, 1 minute)` | `derivative(x)` + `TIMESERIES` | 每秒 |
| `uniques(x)` | `count(*) … FACET x LIMIT n` |                                                                                  |
| `keyset()` / `SHOW EVENT TYPES` | [发现查询](../examples/query-cookbook.md#data-discovery)或查询指南面板 |                                                                                  |
| `histogram(duration, …)` | `FILTER`桶 | 请参阅[教程3](../tutorials/tutorial-3.md#build-a-latency-histogram) |
| `FACET CASES (…)` | `FILTER`列 | 请参阅[教程3](../tutorials/tutorial-3.md#group-values-into-cases-with-filter) |
| `COMPARE WITH 1 week ago` | 不受支持 | 更改时间选取器，或通过API使用`SINCE … UNTIL …` |
| `RLIKE` | `LIKE` / `IN` |                                                                                  |
| `SELECT`中的数学/嵌套聚合 | **公式**&#x200B;构件 |                                                                                  |
| `WITH TIMEZONE`, `EXTRAPOLATE`, `SLIDE BY` | 不受支持 |                                                                                  |
