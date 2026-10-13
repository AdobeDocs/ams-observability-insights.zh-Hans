---
source-git-commit: 263ba9e9b3eb1c6e4dd253ca03bd0ce8954c588b
workflow-type: tm+mt
source-wordcount: '491'
ht-degree: 1%
---
## 教程4：高级函数 {#tutorial-4-advanced-functions}

本教程涵盖了支持更深入调查的功能和实体：可变性、增长、“最坏示例”查找、占总数的份额、事务构成和任意OpenTelemetry指标。

**在本教程中：** [9.1 stddev](#91-measure-consistency-with-stddev) · [9.2派生](#92-spot-growth-with-derivative) · [9.3排名最新](#93-find-the-worst-example-with-ranked-latest) · [9.4 percentOfTotal](#94-find-where-time-goes-with-percentoftotal) · [9.5 SpanBreakdown](#95-break-down-transactions-with-spanbreakdown) · [9.6任何量度](#96-discover-and-query-any-metric) · [9.7选择实体](#97-choose-the-right-trace-entity) · [9.8嵌套聚合](#98-work-around-nested-aggregation)

### 度量值与`stddev`的一致性

两个操作可以共享一个平均值，但行为仍然非常不同。 标准偏差显示滞后时间的变化：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, stddev(duration) AS stddevMs FROM Transaction FACET name ORDER BY stddevMs DESC LIMIT 20
```

比平均偏差大的标准偏差指向偶尔请求非常缓慢的操作。 接下来看看它的p99。

### 使用`derivative`的点增长

`derivative(attr)`返回在存储段之间属性平均值的每秒更改。 它对于发现正在爬升的仪表很有用，例如在泄漏前栈的使用情况：

```sql
SELECT derivative(value) AS heapMBPerSec FROM JvmHeapUsed FACET host TIMESERIES 5 minutes
```

持续的正线意味着增长。 负值表示仪表盘在下降（例如，在垃圾收集之后）。 与`rate`一样，`derivative`需要`TIMESERIES`并且必须是唯一的聚合。

### 查找排名为`latest`的最差示例

`latest`和`earliest`的第二个参数将“最新”更改为“由此属性最高”。 `latest(traceId, duration)`返回最慢请求的跟踪ID，因此您可以将其粘贴到跟踪搜索中：

```sql
SELECT count(*) AS requests, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTrace, latest(urlPath, duration) AS slowestPath FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 15
```

使用`earliest(traceId, duration)`获取最快的示例以进行比较。

### 查找时间与`percentOfTotal`的关系

`percentOfTotal(attr)`显示每个组在总数中所占的份额。 它应用于`duration`，显示哪些操作占用了最多的服务器时间。 这通常比哪些操作速度最慢更有用：

```sql
SELECT count(*) AS requests, sum(duration) AS totalMs, percentOfTotal(duration) AS shareOfServerTime FROM Transaction FACET name LIMIT 15
```

`percentOfTotal`不能与`TIMESERIES`组合。

### 划分与`SpanBreakdown`的事务

`SpanBreakdown`在摄取时预聚合。 它记录每个事务(`rootOperation`)花费的时间（以分钟为单位），每个子操作(`segment`)。 它回答“此端点在做什么？” 低成本：

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown WHERE rootOperation LIKE '%GET%' FACET rootOperation, segment LIMIT 25
```

随时间呈现一个区段的趋势：

```sql
SELECT average(avgDurationMs) AS avgMs FROM SpanBreakdown FACET segment TIMESERIES AUTO LIMIT 8
```

&#x200B;> [!NOTE]
> `avgDurationMs`未使用设备自动标记。 将小组件&#x200B;**单元**&#x200B;设置为`ms`。

### 发现和查询任何量度

SQL没有`keyset()`或`SHOW EVENT TYPES`。 相反，列出报告的每个量度及其单位和最新值：

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricGauge FACET metric, unit LIMIT 200
```

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricSum FACET metric, unit LIMIT 200
```

然后按名称查询任何量度。 例如，每个池使用的JVM内存：

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool LIMIT 50
```

要查看为每个服务填充了哪些&#x200B;**span属性**，请计算非空值：

```sql
SELECT count(*) FILTER (WHERE http.route != '') AS withRoute, count(*) FILTER (WHERE db.system != '') AS dbCalls, count(*) FILTER (WHERE peer.service != '') AS withPeer, count(*) FILTER (WHERE error.type != '') AS withErrorType FROM Span FACET service LIMIT 50
```

&#x200B;> [!TIP]
> 当存在已策划的实体时（例如`CpuUtilization`），首选它而不是`MetricGauge`。 它采用正确的滤波器和单位转换方法，可以加速滚动。

### 选择正确的跟踪实体

| 使用 | 当您想要时 | 示例 |
| ------------- | ------------------------------------------------------------------------------------------- | --------------------------------- |
| `Transaction` | 入站HTTP/RPC请求KPI（吞吐量、延迟、错误、附录）。 汇总加速。 | 黄金信号 |
| `AppRequest` | 相同的KPI，**包括消息使用者**（Kafka和类似者） | 事件驱动型服务 |
| `Span` | 请求以下的任何内容：出站HTTP、DB、消息传送、AEM Sling/JCR/JSP、内部跨距 | 依赖性和代码路径分析 |

计算每个跨度种类以查看检测的内容：

```sql
SELECT count(*) AS spans, percentile(duration, 95) AS p95 FROM Span FACET kind
```

按外部域列出的出站调用：

```sql
SELECT count(*) AS calls, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain LIMIT 20
```

### 解决嵌套聚合问题

NRQL允许您通过子查询聚合聚合，例如“每主机最大值的平均值”。 SQL不会。 下面是备选方案：

| 目标 | SQL方法 |
| --------------------------------------- | ---------------------------------------------------------- |
| 每个组的值，然后对组进行排序 | `FACET group` + `ORDER BY alias` + `LIMIT` |
| 两个聚合的比率 | [公式小组件](#86-combine-queries-with-formula-widgets) |
| 每个主机的值，然后是“最差主机” | `FACET host ORDER BY alias DESC LIMIT 1` |
| 一段时间内每台主机的平均值 | `FACET host TIMESERIES AUTO`，并阅读系列 |

示例：按p95显示的最差主机：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET host ORDER BY p95 DESC LIMIT 1
```
