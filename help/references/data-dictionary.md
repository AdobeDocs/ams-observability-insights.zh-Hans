---
title: 数据词典
description: 引用可用于Observability Insights SQL查询的实体和属性。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '598'
ht-degree: 20%
---

## 数据词典 {#data-dictionary}

这是您可以查询的每个实体和属性。 实时目录（含说明）也在“浏览”的&#x200B;**查询指南**&#x200B;面板和“添加构件”对话框中。

### 跟踪实体

| 实体 | 包含 | 固定范围 | 汇总 |
| ------------- | --------------------------------------------------------- | ---------------------------- | ------------------------------------------ |
| `Transaction` | 入站服务器请求 | `kind = Server` | 1米/ 5米/ 1小时（[性能](#performance)） |
| `AppRequest` | 入站服务器&#x200B;**和**&#x200B;消息使用者请求 | `kind IN (Server, Consumer)` | — |
| `Span` | 每个跨度（服务器、客户端、生产者、消费者、内部） | — | — |

<details open>
<summary><strong>跟踪属性</strong> （在所有三个实体上相同）</summary>

| 属性 | 类型 | 描述 |
| --------------------------------------------------- | ----------- | ----------------------------------------------------------------------------------------------- |
| `timestamp` | 时间戳 | 跨越开始时间 |
| `traceId`, `spanId`, `parentSpanId` | 字符串 | 标识符（根范围具有`parentSpanId = ''`） |
| `name` | 字符串 | 工序或事务处理名称 |
| `kind` | 字符串 | `Server`, `Client`, `Producer`, `Consumer`, `Internal` |
| `service` | 字符串 | 正在发送服务名称 |
| `serviceKey` | 字符串 | 有效服务标识（当`service`为空或粗父级时，回退到`appName`） |
| `appName` | 字符串 | Observability Insights应用程序名称 |
| `appInstance` | 字符串 | 每个实例标识：依次选择作者/发布主机前缀、应用程序名称和服务 |
| `hostTierPrefix` | 字符串 | 主机名在其`author` / `publish` / `dispatcher`角色后缀处被截断 |
| `duration` | 数字（毫秒） | 跨度持续时间（以毫秒为单位） |
| `status` | 字符串 | `Ok`, `Error`, `Unset` |
| `statusMessage` | 字符串 | 状态描述 |
| `host` | 字符串 | 主机名称 |
| `environment` | 字符串 | 部署环境 |
| `region` | 字符串 | 云区域 |
| `cloud.provider` | 字符串 | AWS / AZURE / GCP |
| `http.method` | 字符串 | `GET`、`POST`、... |
| `http.statusCode` | 数字 | HTTP响应状态 |
| `http.route` | 字符串 | 标准化路由模板 |
| `url` | 字符串 | 完整URL |
| `urlPath` | 字符串 | URL (`/content/site/en.html`)的路径部分 |
| `externalDomain` | 字符串 | 出站URL的域（用于客户端范围） |
| `peer.service` | 字符串 | 已调用下游服务 |
| `db.system`, `db.operation`, `db.namespace` | 字符串 | 数据库技术、操作和模式 |
| `messaging.system`, `messaging.destination` | 字符串 | Kafka、RabbitMQ...和主题或队列 |
| `rpc.system`, `rpc.method` | 字符串 | gRPC和类似的 |
| `error.type` | 字符串 | 异常或错误类型 |
| `sling.script`, `sling.servlet`, `sling.requestUri` | 字符串 | AEM Sling详细信息 |
| `jsp.path` | 字符串 | AEM JSP已执行 |
| `jcr.operation`, `jcr.path` | 字符串 | AEM JCR存储库操作和路径 |
| `graphql.operation` | 字符串 | GraphQL操作名称 |

</details>

### 通用量度实体

| 实体 | 包含 |
| ------------- | -------------------------------------------------- |
| `MetricGauge` | 每个时间点OpenTelemetry量规示例 |
| `MetricSum` | 每个累积或增量OpenTelemetry总和示例 |

将它们用于&#x200B;**发现**，或用于没有策划实体的指标： `WHERE metric = 'jvm.gc.duration'`。

<details open>
<summary><strong>量度属性</strong>（由所有量度实体共享）</summary>

| 属性 | 类型 | 描述 |
| ----------------------------------------------------------------------- | --------- | ------------------------------------------------------- |
| `timestamp` | 时间戳 | 示例时间 |
| `metric` | 字符串 | OpenTelemetry度量名称 |
| `value` | 数字 | 示例值（在策划的实体上转换；请参阅下文） |
| `unit` | 字符串 | 量度单位 |
| `service`, `appName`, `host`, `environment`, `region`, `cloud.provider` | 字符串 | 资源标识 |
| `state` | 字符串 | CPU模式或内存状态 |
| `cpu`, `device`, `direction`, `mountpoint` | 字符串 | 每个CPU、每个设备、I/O方向、文件系统 |
| `pool` | 字符串 | JVM内存池或连接池 |
| `topic` | 字符串 | Kafka主题 |
| `process.pid`, `process.name`, `process.owner` | 字符串 | 进程标识 |

</details>

### 策划的指标实体

每个策划的实体都封装一个具有正确过滤器和单位转换的OpenTelemetry量度，因此`value`已经具有意义。 **计数器**&#x200B;实体(✔)包含累积总计。 使用`rate(value)`和`TIMESERIES`查询它们。

| 实体 | `value`表示 | 单位 | 计数器 | 注释 |
| ------------------------------------------------------------------------------ | ----------------------------------- | --------- | ------- | -------------------------------------------- |
| `CpuUtilization` | CPU忙 | % |         | 按主机为`average(value)`进行1分钟汇总 |
| `CpuUserPercent` / `CpuSystemPercent` / `CpuIowaitPercent` / `CpuStealPercent` | CPU按模式显示时间 | % |         |                                              |
| `MemoryUtilization` | 已使用的内存 | % |         | `average(value)`的1分钟汇总 |
| `LoadAverage1m` / `LoadAverage5m` / `LoadAverage15m` | 平均负载 | — |         |                                              |
| `LoadAverageWithFallback` | 500万负荷，降至100万 | — |         |                                              |
| `DiskUtilization` | 每次装载使用的磁盘 | % |         | 按`mountpoint`显示Facet |
| `DiskFreeBytes` | 无根文件系统 | 字节 |         |                                              |
| `DiskReadRate` / `DiskWriteRate` | 磁盘吞吐量 | 字节/秒 | ✔ |                                              |
| `DiskReadOpsRate` / `DiskWriteOpsRate` | 磁盘IOPS | ops/s | ✔ |                                              |
| `NetworkReceiveRate` / `NetworkTransmitRate` | eth0吞吐量 | 字节/秒 | ✔ |                                              |
| `NetworkReceiveRateAllDevices` / `NetworkTransmitRateAllDevices` | 所有设备 | 字节/秒 | ✔ |                                              |
| `NetworkIoRate` | 所有方向和设备组合 | 字节/秒 | ✔ |                                              |
| `NetworkReceiveErrorRate` / `NetworkTransmitErrorRate` | 网络错误 | 个错误/秒 | ✔ |                                              |
| `NetworkReceivePacketRate` / `NetworkTransmitPacketRate` | 数据包 | 数据包 | ✔ |                                              |
| `ProcessCpuUtilization` | 每个进程的CPU | % |         | 按`process.name`显示Facet |
| `ProcessMemoryUsage` | 每个进程的内存 | MB |         |                                              |
| `JvmHeapUsed` / `JvmHeapLimit` | JVM栈 | MB |         | 按内存池报告 |
| `JvmThreadCount` | JVM线程 | — |         |                                              |
| `JvmCpuRecentUtilization` | JVM进程CPU | % |         | 1分钟汇总 |
| `DispatcherRequestRate` | AEM Dispatcher HTTP请求 | 需要 | ✔ |                                              |

>[!NOTE]
>
> 每个内存池&#x200B;**报告JVM内存量度**，因此`JvmHeapUsed` / `JvmHeapLimit`上的`average(value)`跨池平均值。 使用`FACET host, pool`可获取准确的每个池视图。

### `SpanBreakdown`：预聚合的交易构成

`SpanBreakdown`保存每个交易的每分钟子范围组合。 它是在摄取时计算的，所以查询起来很便宜。

| 属性 | 类型 | 描述 |
| --------------- | --------- | ---------------------------------------------------------------------- |
| `timestamp` | 时间戳 | 一分钟存储桶 |
| `service` | 字符串 | 服务 |
| `rootOperation` | 字符串 | 父交易记录名称 |
| `segment` | 字符串 | 子范围名称 |
| `callCount` | 数字 | 分段中此区段的调用 |
| `avgDurationMs` | 数字 | 分段中的平均区段持续时间（手动将单位设置为`ms`） |
