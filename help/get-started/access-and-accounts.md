---
title: 访问和帐户管理
description: 了解可观察性分析帐户的配置方式、管理访问权限的人员以及客户团队具有的控制级别。
feature: Operations
role: Admin
product_v2:
  - id: fd1f54a9-f50c-467d-8956-cebbaf4f3eb8
    internal-label: Experience Manager
feature_v2:
  - id: 8a70d214-ab7b-58c1-b001-2ed2e5d6303d
    internal-label: Operations
role_v2:
  - id: c66ffd68-0f65-42bb-aa23-b4020f12e0bd
    internal-label: Admin
source-git-commit: e0cc17c9d725cad021ba99da4332bca176eae6db
workflow-type: tm+mt
source-wordcount: '232'
ht-degree: 0%
---

# 访问和帐户管理 {#access-and-account-management}

Adobe将为您的AEM Managed Services组织配置和管理Observability Insights帐户。 客户团队使用Adobe IMS登录并以只读方式接收受监控数据。

## 帐户所有权模型 {#account-ownership-model}

- Adobe Managed Services拥有Observability Insights帐户。
- 客户团队被授予只读访问权限。
- 管理更改、配置和访问更新通过Adobe进行管理。

## 用户如何获取访问权限 {#how-users-get-access}

访问可观察性分析需要Adobe IMS配置。

要请求或更新访问权限，请执行以下操作：

1. 联系您的客户成功工程师(CSE)。
2. 提供Adobe IMS配置所需的用户详细信息。
3. 确认已分配正确的组织和访问范围。

配置完成后，登录到[insights.adobecqms.net](https://insights.adobecqms.net)。

## 用户可以执行哪些操作 {#what-users-can-do}

客户用户通常可以：

- 查看APM仪表板
- 查看基础结构仪表板
- 检查受监视的应用程序和主机度量
- 使用共享的仪表板和跟踪参与调查

## 用户无法执行的操作 {#what-users-cannot-do}

除非Adobe Managed Services明确说明，否则客户用户应假定管理控制仍保留在Adobe中。

常见示例包括：

- 管理帐户所有权
- 更改平台级资源调配
- 更改受管理的检测行为

## 稍后添加的信息 {#information-to-add-later}

当有更完整的进程详细信息可用时，请使用此部分：

- Adobe IMS先决条件
- 预期配置周转时间
- 上报联系人和路径
- 面向加入者和离职者的用户生命周期管理
