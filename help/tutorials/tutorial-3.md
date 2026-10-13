---
title: 教程3：推进您的功能板
description: 在Observability Insights中使用变量、公式和多个构件构建仪表板。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '902'
ht-degree: 3%
---

## 教程3：推进您的功能板 {#tutorial-3-advance-your-dashboarding}

在本教程中，您将构建生产级别仪表板的组件：用户体验分数、计数器比率、案例分组、直方图、包含阈值的KPI图块、节、公式小部件和变量。

**在本教程中：** [8.1 apdex](#81-score-user-experience-with-apdex) · [8.2比率](#82-turn-counters-into-rates-with-rate) · [8.3个案例](#83-group-values-into-cases-with-filter) · [8.4直方图](#84-build-a-latency-histogram) · [8.5 KPI图块和节](#85-build-kpi-tiles-and-sections) · [8.6公式](#86-combine-queries-with-formula-widgets) · [8.7变量](#87-make-dashboards-reusable-with-variables) · [8.8将其放在一起](#88-put-it-together)

### 用`apdex`对用户体验评分

Apdex将延迟和错误转换为0到1之间的一个分数。 如果阈值为500毫秒，则请求在500毫秒或以下时&#x200B;_满足_，_容忍_&#x200B;最多2,000毫秒，且在该&#x200B;**以上&#x200B;_受挫_，或者如果失败**：

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

按服务跟踪此趋势：

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

将其放入阈值为`0 → Red`、`0.7 → Amber`、`0.85 → Green`的&#x200B;**数字**&#x200B;小部件中。 最高匹配组获胜，因此低分保持红色。

### 8.2使用`rate`将计数器转换为费率

[数据字典](#53-curated-metric-entities)中的计数器实体(✔)存储的总数不断增加。 `rate(value)`将它们转换为每秒的值。 它需要`TIMESERIES`，并且必须是唯一的聚合：

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host, device TIMESERIES AUTO LIMIT 10
```

>[!WARNING]
>
> 始终按`host`（磁盘按`device`）分面`rate()`，或筛选到一台主机。 如果没有这些信息，来自不同主机的读数会混合在一起，结果就毫无意义。

要从跟踪中获取&#x200B;**请求**&#x200B;比率，请改为对固定存储桶中的请求计数。 这将提供每分钟的请求：

```sql
SELECT count(*) AS requestsPerMinute FROM Transaction TIMESERIES 1 minute
```

### 使用`FILTER`将值分组为大小写

NRQL的`FACET CASES`将行分组到指定的存储桶中。 在SQL中，为每个大小写写一个`FILTER`列。 以下是每个服务的状态类：

```sql
SELECT count(*) FILTER (WHERE http.statusCode < 400) AS ok, count(*) FILTER (WHERE http.statusCode >= 400 AND http.statusCode < 500) AS clientErrors, count(*) FILTER (WHERE http.statusCode >= 500) AS serverErrors FROM Transaction FACET service LIMIT 20
```

随着时间推移，同样的想法也会逐渐奏效，每个案例都有一行：

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/graphql/%') AS graphql FROM Transaction TIMESERIES AUTO
```

### 构建延迟直方图

SQL没有`histogram()`函数。 使用`FILTER`自己生成存储桶，并将结果渲染为&#x200B;**栏**&#x200B;或&#x200B;**表**&#x200B;构件：

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS 'le 100ms', count(*) FILTER (WHERE duration > 100 AND duration <= 250) AS '100-250ms', count(*) FILTER (WHERE duration > 250 AND duration <= 500) AS '250-500ms', count(*) FILTER (WHERE duration > 500 AND duration <= 1000) AS '0.5-1s', count(*) FILTER (WHERE duration > 1000 AND duration <= 2000) AS '1-2s', count(*) FILTER (WHERE duration > 2000) AS 'over 2s' FROM Transaction
```

添加`FACET service`以并排比较多个服务的分布。

### 构建KPI图块和区域

一个良好的仪表板从一行KPI图块开始，然后将详细信息图表分组到各个部分。

1. 添加四个&#x200B;**Number**&#x200B;小组件，每3列宽：

   | 标题 | 查询 | 单位 | 阈值 |
   | -------- | --------------------------------------------------------------------------- | ---- | ---------------------------------------- |
   | 请求 | `SELECT count(*) AS requests FROM Transaction` |      |                                          |
   | 错误% | `SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction` | `%` | `0 → Green`, `1 → Amber`, `5 → Red` |
   | p95 | `SELECT percentile(duration, 95) AS p95 FROM Transaction` | `ms` | `0 → Green`, `800 → Amber`, `2000 → Red` |
   | Apdex | `SELECT apdex(duration, 500) AS apdex FROM Transaction` |      | `0 → Red`, `0.7 → Amber`, `0.85 → Green` |

2. 添加&#x200B;**文本**&#x200B;构件，勾选&#x200B;_使其成为节标题_，并调用它&#x200B;**延迟**。 在其下方添加延迟图表。
3. 添加另一个节标题&#x200B;**ERRORS**，后跟错误图表。 折叠带有V形标记的部分以重点关注其他部分。
4. 针对不同的受众使用&#x200B;**选项卡**，例如&#x200B;_概述_、_AEM_、_基础架构_。 只有可见选项卡运行其查询。

### 将查询与公式构件组合

SQL无法在`SELECT`中进行数学运算，并且没有子查询。 **公式小组件**&#x200B;填补该空白：它们运行2-6个命名查询并将其结果与`+ - * / ( )`合并。

**示例：每个主机在一段时间内的请求数**（公式，行）

1. **将构件添加→公式**。
2. 命名第一个查询`requests`和第二个`hosts`。
3. 输入查询和公式，然后单击&#x200B;**运行公式**。

查询`requests`：

```sql
SELECT count(*) AS v FROM Transaction TIMESERIES AUTO
```

查询`hosts`：

```sql
SELECT uniqueCount(host) AS v FROM Transaction TIMESERIES AUTO
```

公式：

```text
requests / hosts
```

**示例：在数据库**&#x200B;中花费的时间的分享（公式、数字、单位`%`）

查询`db`：

```sql
SELECT sum(duration) AS v FROM Span WHERE db.system != ''
```

查询`server`：

```sql
SELECT sum(duration) AS v FROM Transaction
```

公式：

```text
(db / server) * 100
```

公式查询的规则：

| 规则 | 详细信息 |
| --------------- | -------------------------------------------------------------------------------------------------------------------- |
| 名称 | 字母、数字和`_`，以字母或`_`开头；最多40个字符；唯一 |
| 表达式 | `+ - * / ( )`、数字和查询名称；最多500个字符 |
| 相同形状 | 所有查询均为普通总计（数字）、**或**&#x200B;均为`TIMESERIES`，具有&#x200B;**相同的存储段大小**（行/区域/条形图） |
| 每个量度一个 | 每个查询只返回一个聚合，不返回`FACET` |
| 缺少数据 | 缺少值或除以零会产生空点，从不`Infinity`或`NaN` |

>[!TIP]
>
> 在时间序列公式的每次查询中使用`TIMESERIES AUTO`。 由于查询共享一个窗口，因此它们始终解析为相同的存储段大小。

### 使功能板可通过变量重复使用

变量允许一个功能板提供多种服务、层或阈值。 每个变量都是变量栏中的一个选取器。 在小组件标题和文本小组件中将其引用为`{{name}}`（如果修复了以下限制，则在小组件SQL中引用）。

**创建变量** （_组织和超级管理员_）：单击&#x200B;**+变量**。

| 字段 | 注释 |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **名称** | 字母、数字和`_`，以字母或`_`开头。 必须是唯一的。 用作`{{name}}`。 |
| **标签** | 选取器显示的内容（可选）。 |
| **类型** | **自定义列表** （每行一个值）、**来自查询** （具有一个`FACET`的SQL查询；其不同的值变为选项）或&#x200B;**文本框** （自由文本）。 |
| **允许选择多个值** | 展开为以逗号分隔的列表。 将它与`IN (…)`一起使用。 |
| **值为数字** | 注入未引号的值（`500`而不是`'500'`）。 |

**替换的工作方式**

| 选择区域 | `{{svc}}`变为 |
| ---------------------------------- | ------------------------------------------------------- |
| 单个字符串`checkout` | `'checkout'` |
| 多个值`checkout`，`cart` | `'checkout', 'cart'` |
| 数字`500` | `500` |
| 包含`'`的值 | 转义为`''` |
| 未选择任何内容 | 该构件等待并显示&#x200B;_为{{svc}}_&#x200B;选择一个值 |

```text
WHERE service = {{svc}}            single select
WHERE service IN ({{svc}})         multiple select
WHERE duration > {{slowMs}}        numeric text box
```

对于查询驱动的变量，**始终添加`LIMIT`**。 默认情况下，多面查询仅返回前10个值。 此列表列出最多200项服务：

```sql
SELECT count(*) FROM Transaction FACET service LIMIT 200
```

当租户或时间选择器发生更改时，选项会刷新，并且它们按字母顺序排序。 当当前选择不再有效时，将自动选择第一个值（单选）或所有值（多选）。 变量的查询不能引用其他变量。

>[!WARNING]
>
> **当前生成限制。** 在仪表板网格中进行替换，但&#x200B;**添加构件**&#x200B;预览和&#x200B;**服务器端保存检查**&#x200B;接收原始`{{name}}`令牌并使用`Unexpected character '{'`将其拒绝。 实践中：
>
> - **小组件标题**&#x200B;和&#x200B;**文本小组件**&#x200B;中的`{{variables}}`逐个工作。
> - 在&#x200B;**导入**&#x200B;之后，**小组件SQL**&#x200B;中的`{{variables}}`正确呈现，但存在此类小组件的仪表板无法&#x200B;**保存**。
>
> 在解决此问题之前，请生成具有文本筛选器的每个上下文构件（例如`WHERE service = 'checkout'`），并使用&#x200B;**复制**&#x200B;来克隆它们。

### 放在一起

您现在已在[仪表板Blueprint](#dashboard-blueprints)中使用了每个构建基块：包含阈值、部分、百分点、Apdex、比率、大小写列、直方图、公式和变量的KPI图块。 蓝图显示了如何将它们组合到完整的黄金信号、错误分类、AEM、基础架构和JVM仪表板中。
