---
source-git-commit: ec98f6bd132cadaba1da7c4fb727d7e031790645
workflow-type: tm+mt
source-wordcount: '217'
ht-degree: 2%
---
## 性能 {#performance}

每个查询以15秒的超时和字节读取预算运行。 本文档介绍了可观察性分析如何自动加快查询，以及如何设计在任何时间范围内保持快速状态的仪表板。

### 汇总加速（自动）

`Transaction`个图表读取预聚合表，当这些表的&#x200B;**所有**&#x200B;保持时：

| 条件 | 要求 |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------ |
| 形状 | 查询具有`TIMESERIES` （数字/表格总计始终为准确读取原始数据） |
| 窗口 | ≥ 3 h（1分钟层） · ≥ 1 d（5分钟层） · ≥ 7 d（1小时层） |
| 聚合 | 仅`count(*)`，`percentage(*, WHERE status = 'Error')`，`apdex(duration, 500)`，`percentile(duration, 50 \| 95 \| 99)` |
| `FACET` / `WHERE` | 仅在`service`，`appInstance`，`kind`上 |

包含`FACET host` / `WHERE host …`的`CpuUtilization`和`MemoryUtilization` (`average(value)`)和`JvmCpuRecentUtilization` (`average(value)`)图表≥3小时显示为1分钟的汇总。

**设计提示：**&#x200B;对于7至31天的趋势小组件，请遵循易于汇总的设置（p95，错误%，Apdex @ 500，计数，FACET服务） — 它们在任何范围内都保持快速。 保存较短的窗口或表的`FACET host`/`name`、`average(duration)`或`urlPath`筛选器。

### 一般提示

- 在每个方面查询上使用`LIMIT`，特别是`FACET urlPath`、`traceId`、`jcr.path`（高基数）。
- 在低基数属性(`service`、`kind`、`status`)上通过`WHERE`提前筛选。
- 当只需要入站请求时，更喜欢`Transaction`而不是`Span` — 行数少得多。
- `uniqueCount`是精确的；对于长窗口内的高基数属性，它是最昂贵的聚合。
- 除几分钟的窗口外，请避免`TIMESERIES MAX`。
- 功能板构件缓存结果30秒；刷新按钮强制重新运行。
- 一个遇到15秒超时或字节读取预算错误的构件 — 可缩小窗口、添加过滤器或减少Facet。
