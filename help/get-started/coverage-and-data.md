---
title: 覆盖范围、环境和数据保留
description: 查看Observability Insights在AEM Managed Services中监控的内容、应用程序的表示方式以及监控数据保留多长时间。
feature: Operations
role: Admin
source-git-commit: 1d54a6a398360b040221db5b2780d301722894bf
workflow-type: tm+mt
source-wordcount: '267'
ht-degree: 1%

---


# 覆盖范围、环境和数据保留 {#coverage-environments-and-data-retention}

本页总结了AEM Managed Services的可观察性分析中收集哪些数据以及这些数据的组织方式。

## 监控范围 {#monitoring-coverage}

Adobe监视器：

- 使用Observability Insights APM Java插件的AEM创作层
- 使用Observability Insights APM Java插件的AEM发布层
- 使用Observability Insights Infrastructure代理托管拓扑中的托管服务器

在非生产和生产Managed Services环境中均启用了自定义APM和基础架构监控。

## 应用程序的表示方式 {#how-applications-are-represented}

每个AEM Managed Services环境通常包括：

- 一个APM应用程序供作者使用
- 一个APM应用程序用于发布

Managed Services合同中的所有拓扑都报告到一个Observability Insights帐户中。

## 数据保留 {#data-retention}

APM量度、基础结构量度和相关事件最多可保留&#x200B;**30天**。

## 汇总表 {#summary-tables}

| 覆盖区域 | 受监控的内容 |
| -------------- | ------------------------------------------ |
| APM | AEM创作和发布应用程序 |
| 基础架构 | 托管拓扑中的所有托管服务器 |

| 项目 | 呈现 |
| ------------------------------ | ------------------------------------------------------------- |
| AEM环境 | 一个创作APM应用程序和一个发布APM应用程序 |
| 可观察性分析帐户 | 每个Managed Services客户范围一个Adobe管理的帐户 |

| 数据类型 | 保留 |
| --------------------------------- | ------------- |
| APM量度和事件 | 最多30天 |
| 基础架构量度和事件 | 最多30天 |

## 这在操作上意味着什么 {#what-this-means-operationally}

- 可观察性分析适用于运行分析、活动事件和最近的趋势比较。
- 如果需要，保留窗口以外的历史分析应通过其他报告或归档过程处理。
- 调查重复出现的问题时，应在数据过期之前捕获屏幕快照或导出证据。
