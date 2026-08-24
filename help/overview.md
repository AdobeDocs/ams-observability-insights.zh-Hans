---
title: 使用 [!DNL Synoptryx]监控您的AEM Managed Services环境
description: 在Adobe [!DNL Experience Manager] Managed Services上 [!DNL Synoptryx] 监控概述 — Adobe监控的内容、您的帐户设置方式以及您的团队获取访问权限的方式。
feature: Operations
role: Admin
source-git-commit: e8de2213d91e09da68a8f7014b075f81bd7f07ef
workflow-type: tm+mt
source-wordcount: '618'
ht-degree: 0%

---


# 使用[!DNL Synoptryx]监控您的AEM Managed Services环境 {#synoptryx-monitoring}

[!DNL Synoptryx]使您的团队能够了解应用程序性能、基础架构运行状况和最终用户体验，而无需设置单独的监控平台。

>[!NOTE]
>
> [!DNL Synoptryx]产品概述白皮书包含完整的AEM Managed Services可观察性和监视概述，非常适合于与利益相关者共享或离线查看。

## 概述 {#overview}

[!DNL Synoptryx]是Adobe的新一代可观察性平台，旨在跨应用程序性能、基础架构运行状况和综合监控提供统一的可视性。 它通过单一、集成的体验，实现关键业务服务的主动监控。 [!DNL Synoptryx]结合了Application Performance Monitoring (APM) 、 Infrastructure Monitoring和Synthetic User历程监视，以帮助在问题影响最终用户之前发现并解决问题。 该平台提供了深入的事务跟踪、 JVM洞察、基础架构遥测和高级诊断，从而加快根本原因分析。 它基于现代可观察性技术构建，可在复杂的企业环境中提供可扩展的安全监控。 [!DNL Synoptryx]提供扩展的数据保留、丰富的仪表板和智能分析，以支持卓越的运营。 通过[!DNL Adobe IMS]无缝登录体验可确保安全访问和治理。 该平台旨在提高服务可靠性、加快故障排除速度并增强客户体验。 作为Adobe的战略可观察性解决方案，[!DNL Synoptryx]为跨托管服务环境的监控、自动化和运营见解提供了一个为未来做好准备的基础。

[!DNL Synoptryx]包含在Adobe [!DNL Experience Manager] Managed Services中 — 不需要单独的监视平台或许可证。 Adobe作为我们标准产品的一部分，监控环境的可用性和性能，并且[!DNL Synoptryx]是您的团队可以用来了解Adobe [!DNL Experience Manager] (AEM)应用程序和支持基础架构执行情况的专用平台。

本指南介绍所监控的内容、您的[!DNL Synoptryx]帐户设置方式，以及如何导航用于日常分析和故障排除的仪表板。

## 概览 {#at-a-glance}

在AEM Managed Services中，您将收到：

- **专用[!DNL Synoptryx]帐户** — 由Adobe Managed Services配置和监督，对您的团队具有只读访问权限。
- **深层AEM事务监视** — [!DNL Synoptryx] APM代理跟踪有意义的事务到方法调用（包括行号）、外部依赖项和存储库操作。
- **统一的应用程序和基础结构视图** — 将APM和主机级别的度量结合起来，以全面优化性能。

## 使用[!DNL Synoptryx]监视的Adobe内容 {#what-we-monitor}

Adobe使用[!DNL Synoptryx] APM Java插件监视AEM **创作**&#x200B;和&#x200B;**发布**&#x200B;层。 使用[!DNL Synoptryx]基础结构代理监视拓扑中的所有托管服务器。 在非生产和生产Managed Services环境中均启用了自定义APM和基础架构监控。

![显示跨AEM Author、Publish和托管服务器的Synoptryx APM和基础架构监视的图表](assets/image6.png)

### 您帐户中的应用程序 {#applications-in-your-account}

您的[!DNL Synoptryx]帐户已链接到单个Adobe主帐户，并且可从多个应用程序接收数据，包括：

- 每个AEM Managed Services环境一个&#x200B;**作者**&#x200B;层的APM应用程序
- 每个AEM Managed Services环境一个&#x200B;**发布**&#x200B;层的APM应用程序

每个应用程序都有自己的许可证密钥。 Managed Services合同中的所有拓扑都报告到一个[!DNL Synoptryx]帐户中。 APM和基础结构量度和事件最多可保留&#x200B;**30天**。

## 访问和您的帐户 {#access}

监控数据整合到Adobe配置和管理的[!DNL Synoptryx]帐户中。 您的团队将收到对代理收集的所有APM和基础结构度量的&#x200B;**完全只读访问权限**。 Adobe Managed Services保留对帐户的所有权和管理控制权。

>[!NOTE]
>
> **获取访问权限：**&#x200B;访问[!DNL Synoptryx]需要[!DNL Adobe IMS]设置。 您的客户成功工程师(CSE)可以配置和管理组织的用户访问权限。

在CSE配置帐户后，您可以在[synoptryx.adobecqms.net](https://synoptryx.adobecqms.net)登录。

## 后续内容 {#whats-next}

继续使用您的团队每天使用的监控功能板：

- [应用程序性能监控(APM)](application-performance-monitoring.md) — 跟踪AEM事务、分析JVM行为和检查外部服务。
- [基础架构监控](infrastructure-monitoring.md) — 审查主机级系统、网络、进程和存储指标。

