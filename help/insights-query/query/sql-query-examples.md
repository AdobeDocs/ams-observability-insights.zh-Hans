---
title: SQL查询示例
description: 浏览Observability Insights遥测的SQL查询示例。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '306'
ht-degree: 10%
---

## SQL查询示例

下面是一个更完整的示例。 对于此查询：

- 您选择`Transaction`作为实体。
- 您使用`SELECT`计算第95百分位数持续时间。
- 您使用`FACET`按服务对结果分组。
- 您使用`TIMESERIES AUTO`在自动调整大小的间隔内绘制数据。

```sql
FROM Transaction SELECT percentile(duration, 95) AS p95 FACET service TIMESERIES AUTO
```

这会在折线图上为每个服务生成一行，默认显示10个最繁忙的服务。

单击下面的每个示例以展开它。

<details>
<summary><strong>入站请求的基本查询</strong></summary>

所有入站请求的平均响应时间（以毫秒为单位）：

```sql
SELECT average(duration) FROM Transaction
```

</details>

<details>
<summary><strong>包含点的属性名称</strong></summary>

点状名称不需要引号：

```sql
SELECT count(*) FROM Transaction WHERE http.statusCode >= 500 FACET http.route
```

</details>

<details>
<summary><strong>查询返回多个列</strong></summary>

一个查询中的多个聚合会成为表中的列：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, percentile(duration, 95) AS p95, max(duration) AS maxMs FROM Transaction FACET service
```

</details>

<details>
<summary><strong>一次条件聚合</strong></summary>

总计、错误和5xx并排：

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, count(*) FILTER (WHERE http.statusCode >= 500) AS http5xx FROM Transaction FACET service
```

</details>

<details>
<summary><strong>查询策划的指标</strong></summary>

一段时间内每台主机的平均CPU数：

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

</details>

<details>
<summary><strong>按名称</strong>查询任何OpenTelemetry度量</summary>

将`MetricGauge` / `MetricSum`用于没有策划实体的量度：

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool
```

</details>

<a id="what-sql-doesnt-do"></a>

## SQL不能执行的操作

SQL故意很小。 这保证了查询的安全性、快速性和租户隔离性。 当您需要其中一项功能时，请使用替代功能。

| 功能 | 状态 | 替代 |
| ------------------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------ |
| `SELECT *` | 不受支持 | 列出您需要的属性： `SELECT timestamp, name, duration, traceId …`。 |
| 混合原始属性和聚合 | 不受支持 | 使用`FACET attr`或`latest(attr)` / `earliest(attr)`。 |
| `SELECT`中的数学(`a - b`， `count(*) / 60`) | 不受支持 | **公式**&#x200B;小组件（仪表板）。 |
| 联接和子查询 | 不受支持 | **公式**&#x200B;构件，或两个并排构件。 |
| `HAVING` | 不受支持 | `ORDER BY alias` + `LIMIT`. |
| `COMPARE WITH` | 不受支持 | 更改时间选取器，或将两个小组件放在不同的选项卡上。 |
| `FACET CASES` | 不受支持 | 每个案例一个`count(*) FILTER (WHERE …)`列（请参阅[教程3](#83-group-values-into-cases-with-filter)）。 |
| `histogram()` | 不受支持 | 分段`FILTER`列（请参阅[教程3](#84-build-a-latency-histogram)）。 |
| `uniques()` / `keyset()` | 不受支持 | `FACET attr LIMIT n`列出了不同的值。 发现查询列出量度和填充的属性。 |
| `RLIKE` （正则表达式） | 不受支持 | 包含`%`和`_`通配符的`LIKE` / `NOT LIKE`，或`IN (…)`。 |
| 评论(`--`， `//`， `/* */`) | 不受支持 | 将备注放入小组件标题或&#x200B;**文本**&#x200B;小组件中。 |
| `SINCE`中的绝对日期 | 不受支持 | 使用时间选取器的绝对范围。 |
| 类型转换(`numeric()`， `string()`) | 不需要 | 在目录中键入属性。 |
