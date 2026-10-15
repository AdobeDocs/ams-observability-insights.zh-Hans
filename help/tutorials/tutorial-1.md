---
title: 教程1：您的第一个查询
description: 针对可观察性分析遥测编写您的第一个SQL查询。
source-git-commit: 52cc24792c47727700428be58b5bbb2a5c7223d2
workflow-type: tm+mt
source-wordcount: '552'
ht-degree: 0%
---

## 教程1：您的第一个查询 {#tutorial-1-your-first-queries}

欢迎使用SQL。 在本教程中，您将编写第一个查询，控制时间窗口，绘制趋势，筛选和分组结果，并将第一个图表放在仪表板上。 每个步骤都基于前一个步骤。

### 开始之前

- 您需要访问可观察性分析，并且至少要有一个客户租户发送跟踪或量度。
- 在租户切换器中选择该&#x200B;**客户**。 选择&#x200B;**所有帐户**&#x200B;时，SQL未运行。
- 打开&#x200B;**浏览**&#x200B;并切换到&#x200B;**SQL**&#x200B;选项卡。 在每次查询后使用&#x200B;**运行** (⌘/Ctrl + Enter)。

### 查询具有`SELECT`、`FROM`和`LIMIT`的事件

每个查询都需要两个子句：`SELECT`指示要返回的内容，而`FROM`指示要读取的实体。 从入站请求开始：

```sql
SELECT timestamp, service, name, duration FROM Transaction
```

这是&#x200B;**事件查询**。 每一行都是一个请求。 如果没有`LIMIT`，您最多有100行。 要求更少：

```sql
SELECT timestamp, service, name, duration FROM Transaction LIMIT 10
```

行按第一列降序排序，因此，将`timestamp`放在第一会为您提供最新的请求。

> **注释**
>
> SQL没有`SELECT *`。 将属性列入列表可保持结果的小型性和可读性。 [数据字典](#data-dictionary)和编辑器中的自动完成功能显示了可用的内容。

### 汇总数据

大多数功能板不显示单个事件。 他们总结一下。 将属性替换为&#x200B;**聚合器函数**：

```sql
SELECT count(*) FROM Transaction
```

即窗口中的入站请求数。 再试一次：

```sql
SELECT average(duration) FROM Transaction
```

```sql
SELECT max(duration) FROM Transaction
```

```sql
SELECT uniqueCount(host) FROM Transaction
```

### 一次返回多个聚合

用逗号分隔聚合，并用`AS`为每个聚合命名：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs, max(duration) AS maxMs FROM Transaction
```

> **提示**
>
> 别名成为列标题和图例标签，它们就是您在`ORDER BY`中引用的内容。 养成命名每个聚合的习惯。

### 设置时间窗口

默认情况下，查询涵盖&#x200B;**前一个小时**。 使用`SINCE`进一步回顾：

```sql
SELECT count(*) AS requests FROM Transaction SINCE 1 day ago
```

使用`UNTIL`结束过去的窗口。 这会计算昨天从48小时到24小时前的流量：

```sql
SELECT count(*) AS requests FROM Transaction SINCE 2 days ago UNTIL 1 day ago
```

单位可以是`second`、`minute`、`hour`、`day`或`week`。 窗口最长31天。

> **重要**
>
> 在&#x200B;**浏览**&#x200B;和&#x200B;**仪表板**&#x200B;中，页面顶部的&#x200B;**时间选取器**&#x200B;控制窗口并覆盖`SINCE`/`UNTIL`。 若要查看其他句点，请更改选取器。 因此，本指南的其余部分将`SINCE`保留在仪表板查询之外。

### 使用`TIMESERIES`绘制趋势

添加`TIMESERIES`以随时间将单个数字转换为行：

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

`AUTO`选择分段大小，因此图表具有大约120点（1小时窗口为2分钟，24小时窗口为15分钟）。 您还可以明确设置存储段：

```sql
SELECT average(duration) AS avgMs FROM Transaction TIMESERIES 5 minutes
```

### 使用`WHERE`进行筛选

`WHERE`仅保留您关注的行。 仅对失败的请求计数：

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error'
```

将条件与`AND`、`OR`和括号相结合：

```sql
SELECT count(*) AS slowOrFailed FROM Transaction WHERE service = 'checkout' AND (status = 'Error' OR duration > 2000)
```

将`LIKE`与`%`一起用于部分匹配：

```sql
SELECT count(*) AS contentRequests FROM Transaction WHERE urlPath LIKE '/content/%'
```

> **注释**
>
> 字符串值区分大小写，并使用单引号： `'Error'`，而不是`"Error"`或`'error'`。 将`checkout`替换为租户提供的服务。 下一步说明如何列出这些变量。

### 具有`FACET`的组

`FACET`将结果分组，如SQL的`GROUP BY`。 列出您的服务以及每个服务的繁忙程度：

```sql
SELECT count(*) AS requests FROM Transaction FACET service
```

默认情况下，多面查询返回前10个组。 使用`LIMIT`查看更多：

```sql
SELECT count(*) AS requests, average(duration) AS avgMs FROM Transaction FACET service LIMIT 50
```

将`FACET`和`TIMESERIES`合并为每个组获取一行：

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 5
```

在这里，`LIMIT 5`保留五个最繁忙的系列。

### 将查询保存到仪表板

1. 打开&#x200B;**仪表板→创建仪表板→生成自定义仪表板**，并将其命名为&#x200B;_我的第一个仪表板_。
2. 单击&#x200B;**添加构件**，选择&#x200B;**单个查询**，然后将切换切换开关切换到&#x200B;**SQL**。
3. 粘贴步骤6.7中的查询并单击&#x200B;**运行查询**。
4. 将&#x200B;**Title**&#x200B;设置为&#x200B;_服务请求_&#x200B;并选择&#x200B;**折线**&#x200B;图表类型。
5. 单击顶部栏中的&#x200B;**保存构件**，然后单击&#x200B;**保存**。

您已构建第一个仪表板。
