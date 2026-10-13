---
source-git-commit: 263ba9e9b3eb1c6e4dd253ca03bd0ce8954c588b
workflow-type: tm+mt
source-wordcount: '565'
ht-degree: 0%
---
## 查询指南 {#query-cookbook}

指南是一个现成查询库，按主题分组。 每个查询前面都有一个最适合它的图表类型，以&#x200B;**bold**&#x200B;表示。 时间子句被省略，因为时间选择器控制窗口。

**在此文档中：** [10.1数据发现](#101-data-discovery) · [10.2吞吐量](#102-throughput) · [10.3延迟](#103-latency) · [10.4错误](#104-errors) · [10.5 Apdex](#105-apdex--user-experience) · [10.6服务和主机](#106-services-hosts--instances) · [10.7依赖项](#107-downstream-dependencies) · [10.8 AEM](#108-aem-adobe-experience-manager) · [10.9错误基础架构](#109-infrastructure) · [10.10 JVM](#1010-jvm--runtime)

&#x200B;> [!TIP]
> 将示例值（如`'checkout'`或`'web-publish-01'`）替换为您租户中的实际值。 下面的数据发现查询列出了它们。

### 数据发现

**表** — 哪些服务报告流量：

```sql
SELECT count(*) AS requests, uniqueCount(host) AS hosts FROM Transaction FACET service LIMIT 200
```

**表** — 存在哪些主机/层：

```sql
SELECT count(*) AS requests FROM Transaction FACET hostTierPrefix, host LIMIT 200
```

**表** — 存在哪些环境和区域：

```sql
SELECT count(*) AS spans FROM Span FACET environment, region LIMIT 50
```

**表** — 每个量规量度报告及其最新值：

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricGauge FACET metric, unit LIMIT 200
```

**表** — 每个总和/计数器量度：

```sql
SELECT count(*) AS samples, latest(value) AS latestValue FROM MetricSum FACET metric, unit LIMIT 200
```

**表** — 填充了哪些span属性（AEM示例）：

```sql
SELECT count(*) FILTER (WHERE sling.script != '') AS slingScripts, count(*) FILTER (WHERE jcr.operation != '') AS jcrOps, count(*) FILTER (WHERE db.system != '') AS dbCalls, count(*) FILTER (WHERE graphql.operation != '') AS graphql FROM Span FACET service LIMIT 50
```

### 吞吐量

**数字** — 请求总数：

```sql
SELECT count(*) AS requests FROM Transaction
```

**行** — 吞吐量趋势：

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

**区域** — 按服务划分的吞吐量，前8项：

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**栏** — 通过HTTP方法的吞吐量：

```sql
SELECT count(*) AS requests FROM Transaction FACET http.method LIMIT 10
```

**表** — 最繁忙的URL路径：

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction FACET urlPath ORDER BY requests DESC LIMIT 50
```

**数字** — 不同的跟踪：

```sql
SELECT uniqueCount(traceId) AS traces FROM Span
```

### 延迟

**数字** — p95响应时间（单元自动=毫秒）：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**线** — p50 / p95 / p99在一个图表上：

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**行** — 每个服务的p95，最繁忙的10个：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**栏** — p95执行的最慢操作：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 20
```

**表** — 每个操作的延迟配置文件：

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99, max(duration) AS maxMs, stddev(duration) AS stddevMs FROM Transaction FACET name ORDER BY p95 DESC LIMIT 50
```

**栏** — 每个操作所花费的总时间的份额（时间预算所在的位置）：

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Transaction FACET name LIMIT 20
```

**表** — 每个操作的单个最慢请求的跟踪ID：

```sql
SELECT count(*) AS calls, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTraceId FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 50
```

**数字** — 共享少于2秒的请求：

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowShare FROM Transaction
```

**表** — 通过筛选计数（直方图替代）的延迟存储段：

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS under100ms, count(*) FILTER (WHERE duration > 100 AND duration <= 500) AS to500ms, count(*) FILTER (WHERE duration > 500 AND duration <= 2000) AS to2s, count(*) FILTER (WHERE duration > 2000) AS over2s FROM Transaction FACET service LIMIT 20
```

### 错误数

**数字** — 错误率%：

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**行** — 每个服务的错误率%：

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**行** — 错误计数趋势：

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error' TIMESERIES AUTO
```

**饼图** — HTTP状态类组合：

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode IS NOT NULL FACET http.statusCode LIMIT 15
```

**表** — 5xx热点：

```sql
SELECT count(*) AS errors5xx FROM Transaction WHERE http.statusCode >= 500 FACET service, urlPath, http.statusCode LIMIT 50
```

**表** — 按异常类型分类的错误：

```sql
SELECT count(*) AS errors, uniqueCount(traceId) AS traces, latest(statusMessage) AS lastMessage FROM Span WHERE status = 'Error' FACET service, error.type LIMIT 50
```

**表** — 错误与错误总数并排：

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction FACET service ORDER BY errors DESC LIMIT 20
```

**表** — 最新的失败请求（事件列表）：

```sql
SELECT timestamp, service, name, urlPath, http.statusCode, duration, traceId FROM Transaction WHERE status = 'Error' LIMIT 100
```

### Apdex和用户体验

**数字** — Apdex （T = 500毫秒）：

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**行** — 每个服务的约：

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 10
```

**栏** — 按URL路径最差的Apdex（更严格的目标为300毫秒）：

```sql
SELECT apdex(duration, 300) AS apdex, count(*) AS requests FROM Transaction WHERE urlPath LIKE '/content/%' FACET urlPath ORDER BY apdex ASC LIMIT 20
```

### 服务、主机和实例

**表** — 服务记分卡：

```sql
SELECT count(*) AS requests, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95, apdex(duration, 500) AS apdex, uniqueCount(host) AS hosts FROM Transaction FACET serviceKey ORDER BY requests DESC LIMIT 50
```

**表** — 每个实例（AppRequest包含使用者）：

```sql
SELECT count(*) AS requests, percentage(*, WHERE status = 'Error') AS errorPct, percentile(duration, 95) AS p95 FROM AppRequest FACET appInstance ORDER BY requests DESC LIMIT 50
```

**行** — 每个主机的吞吐量：

```sql
SELECT count(*) AS requests FROM Transaction FACET host TIMESERIES AUTO LIMIT 10
```

**栏** — 负载平衡检查（每个主机的请求共享）：

```sql
SELECT count(*) AS requests FROM Transaction FACET host LIMIT 30
```

### 下游依赖关系

**表** — 已调用下游服务：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client' AND peer.service != '' FACET service, peer.service LIMIT 50
```

**表** — 已调用的外部域：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain ORDER BY calls DESC LIMIT 50
```

**栏** — 系统和操作的数据库延迟：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95 FROM Span WHERE db.system != '' FACET db.system, db.operation LIMIT 30
```

**表** — 数据库命名空间缓慢：

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 99) AS p99 FROM Span WHERE db.system != '' FACET db.system, db.namespace ORDER BY p99 DESC LIMIT 30
```

**行** — 每个目标的消息传送吞吐量：

```sql
SELECT count(*) AS messages FROM Span WHERE messaging.system != '' FACET messaging.destination TIMESERIES AUTO LIMIT 10
```

**表** — 每个主题的生产者与消费者：

```sql
SELECT count(*) FILTER (WHERE kind = 'Producer') AS produced, count(*) FILTER (WHERE kind = 'Consumer') AS consumed, percentile(duration, 95) AS p95 FROM Span WHERE messaging.system != '' FACET messaging.system, messaging.destination LIMIT 30
```

**表** — RPC方法：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE rpc.system != '' FACET rpc.system, rpc.method LIMIT 30
```

### AEM (Adobe Experience Manager)

**区域** — 按AEM层（创作/发布/调度程序前缀）的流量：

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix TIMESERIES AUTO LIMIT 10
```

**行** — 发布层p95：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO LIMIT 10
```

**行** — 创作层p95：

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%author%' FACET host TIMESERIES AUTO LIMIT 10
```

**表** — 页面请求(`/content`)与servlet调用(`/bin`)：

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/etc.clientlibs/%') AS clientlibs, count(*) FILTER (WHERE urlPath LIKE '/content/dam/%') AS assets FROM Transaction FACET hostTierPrefix LIMIT 20
```

**表** — 最慢的`/bin` servlet：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE urlPath LIKE '/bin/%' FACET urlPath ORDER BY p95 DESC LIMIT 30
```

**表** — Sling请求URI：

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction WHERE sling.requestUri != '' FACET sling.requestUri ORDER BY requests DESC LIMIT 100
```

**表** — Sling脚本缓慢：

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE sling.script != '' FACET sling.script ORDER BY totalMs DESC LIMIT 50
```

**栏** — 按时间共享列出的Sling servlet：

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Span WHERE sling.servlet != '' FACET sling.servlet LIMIT 20
```

**表** — 慢速JSP：

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95 FROM Span WHERE jsp.path != '' FACET jsp.path ORDER BY p95 DESC LIMIT 50
```

**栏** — JCR操作组合：

```sql
SELECT count(*) AS operations FROM Span WHERE jcr.operation != '' FACET jcr.operation LIMIT 20
```

**表** — 最热的JCR路径：

```sql
SELECT count(*) AS operations, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE jcr.path != '' FACET jcr.operation, jcr.path ORDER BY totalMs DESC LIMIT 50
```

**表** — GraphQL (headless)操作：

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE graphql.operation != '' FACET graphql.operation ORDER BY calls DESC LIMIT 50
```

**行** — 每个主机的Dispatcher请求率：

```sql
SELECT rate(value) AS requestsPerSecond FROM DispatcherRequestRate FACET host TIMESERIES AUTO
```

**表** — 事务由什么组成（区段划分；设置单元`ms`）：

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown WHERE rootOperation LIKE 'GET /content%' FACET rootOperation, segment ORDER BY calls DESC LIMIT 50
```

### 基础架构

**行** — 每个主机的CPU：

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

**行** — CPU用户时间：

```sql
SELECT average(value) AS userPct FROM CpuUserPercent TIMESERIES AUTO
```

&#x200B;> [!NOTE]
> SQL为每个查询读取一个实体，因此user/system/iowait/steke需要四个并排小组件（请参阅[blueprint 11.6](#116-infrastructure-capacity-board) W7-W10）。 公式会将它们折叠为一个数字，而不是四个系列。

**栏** — 按主机列出的CPU p95：

```sql
SELECT percentile(value, 95) AS cpuP95 FROM CpuUtilization FACET host LIMIT 30
```

**表** — 主机运行状况快照：

```sql
SELECT average(value) AS avgCpu, max(value) AS peakCpu, latest(value) AS nowCpu FROM CpuUtilization FACET host ORDER BY peakCpu DESC LIMIT 50
```

**行** — 已用内存%：

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**行** — 内存增长率（泄漏检测）：

```sql
SELECT derivative(value) AS memoryTrend FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**表** — 每次装载使用的磁盘：

```sql
SELECT latest(value) AS usedPct FROM DiskUtilization FACET host, mountpoint ORDER BY usedPct DESC LIMIT 50
```

**数字** — 所有主机可用的最少的根磁盘（以字节为单位的值；设置单位`bytes`）：

```sql
SELECT min(value) AS freeBytes FROM DiskFreeBytes
```

**行** — 磁盘读取和写入吞吐量（各一个）：

```sql
SELECT rate(value) AS readBytesPerSec FROM DiskReadRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host TIMESERIES AUTO
```

**线路** — 网络接收/传输：

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS txBytesPerSec FROM NetworkTransmitRate FACET host TIMESERIES AUTO
```

**行** — 网络错误：

```sql
SELECT rate(value) AS rxErrors FROM NetworkReceiveErrorRate FACET host TIMESERIES AUTO
```

**行** — 平均负载：

```sql
SELECT average(value) AS load5m FROM LoadAverage5m FACET host TIMESERIES AUTO
```

**表** — 按CPU和内存排在前面的进程：

```sql
SELECT average(value) AS cpuPct, max(value) AS peakCpuPct FROM ProcessCpuUtilization FACET host, process.name ORDER BY cpuPct DESC LIMIT 30
```

```sql
SELECT average(value) AS memMb FROM ProcessMemoryUsage FACET host, process.name ORDER BY memMb DESC LIMIT 30
```

**表** — 按名称划分的任何原始仪表盘（不需要策划的实体）：

```sql
SELECT average(value) AS avgValue, max(value) AS maxValue FROM MetricGauge WHERE metric = 'system.cpu.utilization' FACET host, state LIMIT 50
```

### JVM和运行时

**行** — 每个主机使用的栈(MB)：

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed FACET host TIMESERIES AUTO
```

**表** — 使用的栈与限制（两个小部件或公式 — 请参阅[Blueprint 11.7](#117-jvm-health-board)）：

```sql
SELECT latest(value) AS heapLimitMb FROM JvmHeapLimit FACET host LIMIT 50
```

**行** — JVM线程：

```sql
SELECT latest(value) AS threads FROM JvmThreadCount FACET host TIMESERIES AUTO
```

**行** — JVM进程CPU：

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization FACET host TIMESERIES AUTO
```

**表** — JVM内存池（通用求和量度）：

```sql
SELECT latest(value) AS bytesUsed FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool LIMIT 100
```
