---
title: 疑难解答
description: 在可观察性分析中，对SQL查询、图表和功能板进行故障诊断。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '337'
ht-degree: 5%
---

## 疑难解答 {#troubleshooting}

当您&#x200B;**验证**&#x200B;或&#x200B;**运行查询**&#x200B;时，编辑器下将显示错误消息，在仪表板上的小组件本身也会显示错误消息。 在第一列中查找消息。

| 消息/症状 | 原因 | 修复 |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------- |
| `Unknown entity 'X'` | 拼写错误或非目录名称 | 检查[数据字典](#data-dictionary)/查询指南面板。 |
| `Attribute 'x' is not available on Entity` | 属性不在该实体上（例如`CpuUtilization`上的`duration`） | 跟踪属性和量度属性 — 请参阅[数据字典](#data-dictionary)。 |
| `Raw attributes cannot be mixed with aggregates; use FACET to group results` | `SELECT service, count(*)` | `SELECT count(*) … FACET service`. |
| `FACET and TIMESERIES require an aggregate query` | 带有FACET/TIMESERIES的事件查询 | 使用聚合。 |
| `FACET supports at most five attributes` | > 5个Facet | 减少。 |
| `average requires a numeric attribute` | e.g. `average(service)` | 使用`uniqueCount`/`latest`或数字属性。 |
| `rate requires TIMESERIES` / `rate is only available for counter measures` | `rate()`不含TIMESERIES或在仪表上 | 添加`TIMESERIES AUTO`；使用✔计数器实体（[策划的度量实体](#53-curated-metric-entities)）；对于仪表盘使用`derivative()`。 |
| `rate cannot be combined with other aggregates` | 多个具有`rate()`的项目 | 每个小组件一个`rate()`；与公式组合。 |
| `percentOfTotal cannot be combined with TIMESERIES` | — | 删除时间序列。 |
| `apdex requires an entity with a status attribute` | 量度实体上的`apdex` | 使用`Transaction`/`Span`/`AppRequest`。 |
| `ORDER BY 'x' must reference a selected value or alias` | 按未选择的内容排序 | 按别名对聚合和顺序进行别名。 |
| `IN requires between 1 and 100 values` | — | 拆分或使用`LIKE`。 |
| `Use IS NULL or IS NOT NULL for null comparisons` | `attr = NULL` | 使用`IS NULL`。 |
| `Unexpected character '{'` | 在“添加构件”预览中或“保存”时`{{variable}}` | 请参阅[Tutorial 3，步骤8.7](#87-make-dashboards-reusable-with-variables)限制。 |
| `Unexpected character ';'` / `'"'` | 分号或双引号 | 删除`;`；使用单引号。 |
| `Expected a positive duration such as 30 minutes` | `TIMESERIES 5m` | 写入`TIMESERIES 5 minutes`。 |
| `Time range cannot exceed 31 days` | 选取器> 31天 | 缩小窗口。 |
| `Select a customer before running a SQL query` | 无租户/演示租户 | 在租户切换器中选择客户。 |
| `All queries in a formula must be the same kind…` | 总计和时序或彩块化的组合 | 使所有子查询为纯查询或所有`TIMESERIES AUTO`；删除FACET。 |
| `Query 'x' must select exactly one metric…` | 子查询有两个聚合 | 每个子查询一个聚合。 |
| `Unknown query name 'x' - must be one of: …` | 公式引用缺少的名称 | 完全匹配名称（区分大小写）。 |
| **保存失败** | 构件查询在服务器端无效（通常为`{{var}}`） | 打开每个最近编辑的小组件并&#x200B;**运行**&#x200B;它；修复或删除无效的小组件。 |
| _选择{{x}}_&#x200B;的值 | 变量无选择项 | 在变量栏中选取一个值。 |
| _过去……无活动_ | 窗口中没有行 | 扩大窗口；使用发现查询（[数据发现](#101-data-discovery)）验证筛选器。 |
| 线条破碎为点 | 存储段小于数据节奏 | 使用`TIMESERIES AUTO` （≥2分钟）或条形图。 |
| Facet显示空字符串组 | 某些行上缺少属性 | 添加`WHERE attr != ''`。 |
| _此旧版SQL面板不再运行_ | 旧原始SQL生成器中的面板 | 在SQL中编辑→重建。 |
