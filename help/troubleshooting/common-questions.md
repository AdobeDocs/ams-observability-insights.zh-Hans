---
title: 常见问题解答
description: AEM Managed Services中的可观察性洞察的常见问题和调查起点。
feature: Operations
role: Admin
source-git-commit: 3e9cd3734665dc06a4b90902b229dffb8f5421df
workflow-type: tm+mt
source-wordcount: '216'
ht-degree: 0%

---


# 常见问题解答 {#faq}

当您不确定从何处开始或在进行积极调查期间需要快速回答时，可以使用此页作为起点。

## 为何无法访问可观察性洞察？ {#cannot-access-observability-insights}

从[访问和帐户管理](../get-started/access-and-accounts.md)开始。 如果您的配置不完整或已过时，请联系您的客户成功工程师(CSE)以请求访问或更新。

## 为什么在尝试登录时看到“正在加载权限”？ {#loading-permissions-error}

这通常表示用户配置存在问题。 联系您的客户成功工程师(CSE)，他们可以与其他相关团队一起解决访问问题。

## 如何确定问题是否与应用程序或基础架构相关？ {#application-or-infrastructure}

从[应用程序性能监控](/help/applications.md)开始，查看创作或发布的请求率、错误率和延迟。 如果应用信号被提升，请使用[主机](/help/hosts.md)来检查主机级资源压力（CPU、内存、磁盘或网络）是否说明了您所看到的内容，或使所看到的内容复杂化。

## 我应如何了解特定图形或量度？ {#understand-graph-or-metric}

使用功能板参考页面可获取逐个面板的说明、量度名称、单位和屏幕截图：

- [APM仪表板引用](../reference/apm-dashboard-reference.md)
- [基础架构仪表板引用](../reference/infrastructure-dashboard-reference.md)

## Observability Insights实际收集哪些数据？ {#what-data-is-collected}

请参阅[覆盖范围、环境和数据保留](../get-started/coverage-and-data.md)，以了解监视范围、应用程序呈现、保留期和操作影响。
