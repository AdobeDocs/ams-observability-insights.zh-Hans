---
title: 仪表板Blueprint
description: 使用Blueprint构建可观察性分析功能板以进行应用程序和基础架构监控。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '1324'
ht-degree: 1%
---

## 仪表板Blueprint {#dashboard-blueprints}

每个Blueprint都是一个完整、复杂的功能板。 其中列出了目标、选项卡和节布局，以及每个小组件及其图表类型、单位、阈值、大小和查询。 在12列网格上，大小为`width×height`。 从上到下构建每个，然后拖动进行调整。

| Blueprint | 受众 | 高亮 |
| ------------------------------------------------------------------------------------ | --------------------- | ---------------------------------------------------------------------------------- |
| [11.1 Golden Signals / SLO](#111-golden-signals--slo-board) | SRE，待命 | KPI图块，带有阈值、Apdex、错误%、饱和度 |
| [11.2错误分类](#112-error-triage-board) | 待命 | 错误类、 5xx热点、带有跟踪ID的失败请求 |
| [11.3延迟深入分析](#113-latency-deep-dive-board) | 性能工程师 | 百分点、延迟存储段、可变性、交易组成、最慢跟踪 |
| [11.4下游依赖项](#114-downstream-dependencies-board) | 服务所有者 | 出站HTTP、DB、消息传送、RPC |
| [11.5 AEM创作/发布/ Dispatcher](#115-aem-author--publish--dispatcher-board) | AEM团队 | 每层的区段、Sling/JSP/JCR、Dispatcher加载 |
| [11.6基础架构容量](#116-infrastructure-capacity-board) | 平台/容量 | CPU模式，内存，磁盘，网络速率，公式 |
| [11.7 JVM运行状况](#117-jvm-health-board) | Java团队 | 栈%公式，每个主机和池的栈，线程， JVM CPU |
| [11.8主管/NOC墙](#118-executive--noc-wall-board) | NOC，领导力 | KPI图块、可用性公式、按层流量、SLO说明 |
| [11.9比例和余量](#119-ratios--headroom-board-formula-showcase) | 每个人 | 公式小组件展示 |
| [11.10服务资源管理器](#1110-service-explorer-variable-driven) | 每个团队 | 变量驱动、可重用 |

**图表图例：** **N** =数字，**L** =折线，**A** =面积图，**B** =条形图，**P** =饼图，**T** =表格。



### Golden Signals/SLO板

**目标：**一个屏幕回答“平台当前是否正常？”  — 流量、错误、延迟、饱和度。
**时间选取器：**&#x200B;最近1小时（事件）或24小时（每日审阅）。

```text
┌──────────── Section: SLO KPIs ────────────────────────────────┐
│ N Requests │ N Error % │ N p95 │ N Apdex │ N Slow % │ N Hosts  │  (6 × 2w×2h)
├──────────── Section: Traffic & Errors ────────────────────────┤
│ A Throughput by service (6w) │ L Error % by service (6w)       │
├──────────── Section: Latency ─────────────────────────────────┤
│ L p50/p95/p99 (6w)           │ L Apdex by service (6w)         │
├──────────── Section: Saturation ──────────────────────────────┤
│ L CPU by host (4w) │ L Memory by host (4w) │ L Load 5m (4w)    │
└───────────────────────────────────────────────────────────────┘
```

**W1 ·请求** — N，2×2

```sql
SELECT count(*) AS requests FROM Transaction
```

**W2 ·错误%** — N，单位`%`，2×2。 阈值： `0 → Green`、`1 → Amber`、`5 → Red`。

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**W3 · p95延迟** — N，单位`ms`，2×2。 阈值： `0 → Green`、`800 → Amber`、`2000 → Red`。

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**W4 · Apdex** — N，2×2。 阈值： `0 → Red`、`0.7 → Amber`、`0.85 → Green` （匹配的最高频带获胜次数，因此低分数保持红色）。

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**W5 ·请求> 2 s** — N，单位`%`，2×2。 阈值： `0 → Green`、`2 → Amber`、`10 → Red`。

```sql
SELECT percentage(*, WHERE duration > 2000) AS slowPct FROM Transaction
```

**W6 ·活动主机** — N，2×2

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction
```

**W7 ·服务吞吐量** — A， 6×4

```sql
SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W8 ·服务**&#x200B;的错误% — L，单元`%`，6×4

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W9 ·延迟百分位数** — L，单位`ms`，6×4

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**W10 ·按服务排列的Apdex** — L，6×4

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction FACET service TIMESERIES AUTO LIMIT 8
```

**W11 ·主机CPU** — L，4×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO LIMIT 10
```

**W12 ·按主机列出的内存** — L， 4×4

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO LIMIT 10
```

**W13 ·加载5m** — L， 4×4

```sql
SELECT average(value) AS load5m FROM LoadAverage5m FACET host TIMESERIES AUTO LIMIT 10
```



### 错误分类展示板

**目标：**&#x200B;从“错误率已达到”到“此终结点，此异常，此跟踪”（通过三次点击）。

```text
Tab "Overview"
│ N Errors │ N Error % │ N Affected traces │ N 5xx count │                 (4 × 3w×2h)
│ L Errors by service (8w) │ P Status code mix (4w)                       │
│ T Errors vs total by service (12w)                                       │
Tab "Drill-down"
│ T 5xx hot spots: service × path × code (12w)                             │
│ T Exception types (6w) │ T Downstream failures (6w)                      │
│ T Latest failing requests – event list (12w×6h)                          │
```

**概述选项卡**

**W1 ·错误** — N，阈值`0 → Green`，`1 → Amber`，`100 → Red`

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error'
```

**W2 ·错误%** — N，单元`%`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction
```

**W3 ·受影响的跟踪** — N

```sql
SELECT uniqueCount(traceId) AS traces FROM Span WHERE status = 'Error'
```

**W4 · HTTP 5xx** — N，阈值`0 → Green`，`1 → Red`

```sql
SELECT count(*) AS http5xx FROM Transaction WHERE http.statusCode >= 500
```

**W5 ·服务错误** — L， 8×4

```sql
SELECT count(*) AS errors FROM Transaction WHERE status = 'Error' FACET service TIMESERIES AUTO LIMIT 10
```

**W6 ·状态代码组合** — P， 4×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE http.statusCode >= 400 FACET http.statusCode LIMIT 10
```

**W7 ·错误数与服务总数** — T，12×4

```sql
SELECT count(*) AS total, count(*) FILTER (WHERE status = 'Error') AS errors, percentage(*, WHERE status = 'Error') AS errorPct, count(*) FILTER (WHERE http.statusCode >= 500) AS http5xx, count(*) FILTER (WHERE http.statusCode >= 400 AND http.statusCode < 500) AS http4xx FROM Transaction FACET service ORDER BY errors DESC LIMIT 30
```

**向下钻取选项卡**

**W8 · 5xx热点** — T， 12×5

```sql
SELECT count(*) AS errors, percentile(duration, 95) AS p95, latest(traceId) AS lastTraceId FROM Transaction WHERE http.statusCode >= 500 FACET service, urlPath, http.statusCode ORDER BY errors DESC LIMIT 100
```

**W9 ·异常类型** — T， 6×5

```sql
SELECT count(*) AS errors, latest(statusMessage) AS lastMessage FROM Span WHERE status = 'Error' AND error.type != '' FACET service, error.type ORDER BY errors DESC LIMIT 50
```

**W10 ·下游故障** — T，6×5

```sql
SELECT count(*) AS failures, percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND status = 'Error' FACET service, peer.service, externalDomain ORDER BY failures DESC LIMIT 50
```

**W11 ·最新的失败请求** — T，12×6

```sql
SELECT timestamp, service, host, name, urlPath, http.statusCode, duration, statusMessage, traceId FROM Transaction WHERE status = 'Error' LIMIT 200
```



### 延迟深度讨论区

**目标：**&#x200B;查找&#x200B;_位置_&#x200B;时间 — 哪些操作、哪些区段、一致性如何。

```text
│ L p50/p95/p99 (8w)               │ N p99 (4w)                │
│ B Time share by operation (6w)   │ B Slowest ops by p95 (6w) │
│ T Latency profile per operation (12w)                         │
│ T Latency buckets per service (6w) │ B Variability (6w)       │
│ T Transaction composition – SpanBreakdown (12w)               │
│ T Slowest trace per operation (12w)                           │
```

**W1 ·百分位数** — L， `ms`， 8×4

```sql
SELECT percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99 FROM Transaction TIMESERIES AUTO
```

**W2 · p99** — N， `ms`，阈值`0 → Green`， `1500 → Amber`， `4000 → Red`， 4×4

```sql
SELECT percentile(duration, 99) AS p99 FROM Transaction
```

**W3 ·按操作共享的时间** — B， `%`， 6×4

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Transaction FACET name LIMIT 15
```

**W4 ·最慢的操作(p95)** — B， `ms`， 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction FACET name ORDER BY p95 DESC LIMIT 15
```

**W5 ·延迟配置文件** — T，12×5

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 50) AS p50, percentile(duration, 95) AS p95, percentile(duration, 99) AS p99, max(duration) AS maxMs FROM Transaction FACET service, name ORDER BY p95 DESC LIMIT 100
```

**W6 ·延迟存储段** — T，6×4

```sql
SELECT count(*) FILTER (WHERE duration <= 100) AS fast, count(*) FILTER (WHERE duration > 100 AND duration <= 500) AS ok, count(*) FILTER (WHERE duration > 500 AND duration <= 2000) AS slow, count(*) FILTER (WHERE duration > 2000) AS verySlow FROM Transaction FACET service LIMIT 20
```

**W7 ·延迟可变性** — B， `ms`， 6×4

```sql
SELECT stddev(duration) AS stddevMs FROM Transaction FACET service ORDER BY stddevMs DESC LIMIT 15
```

**W8 ·事务组合** — T，单位`ms`，12×5

```sql
SELECT sum(callCount) AS calls, average(avgDurationMs) AS avgMs FROM SpanBreakdown FACET service, rootOperation, segment ORDER BY calls DESC LIMIT 100
```

**W9 ·每个操作的最慢跟踪** — T， 12×5

```sql
SELECT count(*) AS calls, max(duration) AS slowestMs, latest(traceId, duration) AS slowestTraceId, latest(host, duration) AS slowestHost FROM Transaction FACET name ORDER BY slowestMs DESC LIMIT 50
```



### 下游依赖关系板

**目标：**&#x200B;是我们还是他们？ 我们调用的每个数据库、API、队列和RPC的运行状况。

```text
│ N Outbound calls │ N Outbound error % │ N Outbound p95 │            (3 × 4w×2h)
│ T Downstream services (12w)                                       │
│ L External domain p95 (6w) │ T External domains (6w)              │
│ B DB p95 by system/op (6w) │ T Slow DB namespaces (6w)            │
│ L Messages per destination (6w) │ T Producer vs consumer (6w)     │
│ T RPC methods (12w)                                               │
```

**W1 ·出站呼叫** — N

```sql
SELECT count(*) AS calls FROM Span WHERE kind = 'Client'
```

**W2 ·出站错误%** — N，`%`，阈值`0 → Green`，`2 → Amber`，`10 → Red`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client'
```

**W3 ·出站p95** — N，`ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client'
```

**W4 ·下游服务** — T， 12×5

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE kind = 'Client' FACET service, peer.service ORDER BY calls DESC LIMIT 100
```

**W5 ·外部域p95** — L， `ms`， 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain TIMESERIES AUTO LIMIT 8
```

**W6 ·外部域** — T， 6×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND externalDomain != '' FACET externalDomain ORDER BY calls DESC LIMIT 50
```

**W7 · DB p95** — B， `ms`， 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Span WHERE db.system != '' FACET db.system, db.operation ORDER BY p95 DESC LIMIT 20
```

**W8 ·慢速数据库命名空间** — T，6×4

```sql
SELECT count(*) AS calls, average(duration) AS avgMs, percentile(duration, 99) AS p99, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE db.system != '' FACET service, db.system, db.namespace ORDER BY p99 DESC LIMIT 50
```

**W9 ·每个目标的邮件数** — L， 6×4

```sql
SELECT count(*) AS messages FROM Span WHERE messaging.system != '' FACET messaging.destination TIMESERIES AUTO LIMIT 10
```

**W10 ·生产者和消费者** — T，6×4

```sql
SELECT count(*) FILTER (WHERE kind = 'Producer') AS produced, count(*) FILTER (WHERE kind = 'Consumer') AS consumed, percentile(duration, 95) AS p95 FROM Span WHERE messaging.system != '' FACET messaging.system, messaging.destination LIMIT 50
```

**W11 · RPC方法** — T， 12×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Span WHERE rpc.system != '' FACET service, rpc.system, rpc.method ORDER BY calls DESC LIMIT 50
```



### AEM创作/发布/Dispatcher展示板

**目标：**按层分组的AEM操作展示板，镜像经典New Relic AEM布局。
使用名为`PUBLISH`、`AUTHOR`、`DISPATCHER`的&#x200B;**节标题**&#x200B;和带`- PUBLISH` / `- AUTHOR`的结尾小组件标题以获取颜色提示。

```text
Tab "Tiers"
├─ Section: OVERVIEW
│ A Traffic by tier (8w) │ T Content mix by tier (4w)
├─ Section: PUBLISH
│ N Requests - PUBLISH │ N Error % - PUBLISH │ N p95 - PUBLISH │ N Apdex - PUBLISH
│ L p95 by host - PUBLISH (6w) │ L CPU - PUBLISH (6w)
│ L Heap - PUBLISH (6w) │ T Slowest /bin servlets - PUBLISH (6w)
├─ Section: AUTHOR
│ N Requests - AUTHOR │ N Error % - AUTHOR │ N p95 - AUTHOR │ N Active authors' hosts - AUTHOR
│ L p95 by host - AUTHOR (6w) │ L Heap - AUTHOR (6w)
├─ Section: DISPATCHER
│ L Dispatcher req/s (8w) │ L CPU - DISPATCHER (4w)
Tab "Code hot spots"
│ T Slow Sling scripts (6w) │ T Slow JSPs (6w)
│ T Hottest JCR paths (6w)  │ B JCR operation mix (6w)
│ T Sling request URIs (6w) │ T GraphQL operations (6w)
│ B Sling servlet time share (12w)
```

**概述**

**W1 ·按层**&#x200B;显示的流量 — A， 8×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix TIMESERIES AUTO LIMIT 10
```

**W2 ·按层**&#x200B;列出的内容组合 — T，4×4

```sql
SELECT count(*) FILTER (WHERE urlPath LIKE '/content/%') AS pages, count(*) FILTER (WHERE urlPath LIKE '/bin/%') AS servlets, count(*) FILTER (WHERE urlPath LIKE '/content/dam/%') AS assets FROM Transaction FACET hostTierPrefix LIMIT 20
```

**发布节**

**W3 ·请求 — 发布** — N

```sql
SELECT count(*) AS requests FROM Transaction WHERE host LIKE '%publish%'
```

**W4 ·错误% — 发布** — N，`%`，阈值`0 → Green`，`1 → Amber`，`5 → Red`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction WHERE host LIKE '%publish%'
```

**W5 · p95 - PUBLISH** — N，`ms`，阈值`0 → Green`，`1000 → Amber`，`3000 → Red`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%'
```

**W6 · Apdex - PUBLISH** — N，阈值`0 → Red`，`0.7 → Amber`，`0.85 → Green`

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction WHERE host LIKE '%publish%'
```

**W7 · p95，按主机 — PUBLISH** — L， `ms`， 6×4

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO LIMIT 10
```

**W8 · CPU — 发布** — L， `%`， 6×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO
```

**W9 ·栈 — 发布** — L， `MB`， 6×4

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed WHERE host LIKE '%publish%' FACET host TIMESERIES AUTO
```

**W10 ·最慢的/bin servlet - PUBLISH** — T，6×4

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE host LIKE '%publish%' AND urlPath LIKE '/bin/%' FACET urlPath ORDER BY p95 DESC LIMIT 30
```

**作者节** — 复制W3-W9并将`'%publish%'`更改为`'%author%'`，并将标题后缀更改为`- AUTHOR`。 附加构件：

**W11 ·作者主机处于活动状态 — AUTHOR** — N

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction WHERE host LIKE '%author%'
```

**Dispatcher部分**

**W12 · Dispatcher请求** — L， `req/s`， 8×4

```sql
SELECT rate(value) AS requestsPerSecond FROM DispatcherRequestRate FACET host TIMESERIES AUTO
```

**W13 · CPU - DISPATCHER** — L， `%`， 4×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization WHERE host LIKE '%dispatcher%' FACET host TIMESERIES AUTO
```

**代码热点选项卡**

**W14 ·慢速Sling脚本** — T

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE sling.script != '' FACET sling.script ORDER BY totalMs DESC LIMIT 50
```

**W15 ·慢速JSP** — T

```sql
SELECT count(*) AS executions, percentile(duration, 95) AS p95 FROM Span WHERE jsp.path != '' FACET jsp.path ORDER BY p95 DESC LIMIT 50
```

**W16 ·最热的JCR路径** — T

```sql
SELECT count(*) AS operations, percentile(duration, 95) AS p95, sum(duration) AS totalMs FROM Span WHERE jcr.path != '' FACET jcr.operation, jcr.path ORDER BY totalMs DESC LIMIT 50
```

**W17 · JCR操作组合** — B

```sql
SELECT count(*) AS operations FROM Span WHERE jcr.operation != '' FACET jcr.operation LIMIT 20
```

**W18 · Sling请求URI** — T

```sql
SELECT count(*) AS requests, percentile(duration, 95) AS p95 FROM Transaction WHERE sling.requestUri != '' FACET sling.requestUri ORDER BY requests DESC LIMIT 100
```

**W19 · GraphQL操作** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE graphql.operation != '' FACET graphql.operation ORDER BY calls DESC LIMIT 50
```

**W20 · Sling servlet时间共享** — B， `%`， 12×4

```sql
SELECT percentOfTotal(duration) AS timeShare FROM Span WHERE sling.servlet != '' FACET sling.servlet LIMIT 20
```



### 基础架构容量展示板

**目标：**跨主机进行容量规划和饱和度搜索。
**时间选择器：** 7天用于趋势（汇总可以快速完成），1小时用于事件。

```text
Tab "Compute"
│ N Hosts reporting │ N Peak CPU % │ N Peak memory % │ N Max load 5m │
│ L CPU by host (6w) │ L Memory by host (6w)                         │
│ L CPU user / system / iowait / steal (4 × 3w)                     │
│ T Host snapshot (12w)                                              │
│ T Top processes by CPU (6w) │ T Top processes by memory (6w)       │
Tab "Storage & Network"
│ T Disk used per mount (6w) │ N Lowest free root disk (6w)          │
│ L Disk read (6w) │ L Disk write (6w)                               │
│ L Network rx (6w) │ L Network tx (6w)                              │
│ L Total network I/O for one host – formula (12w)                   │
```

**W1 ·主机报告** — N

```sql
SELECT uniqueCount(host) AS hosts FROM CpuUtilization
```

**W2 ·峰值CPU** — N，`%`，阈值`0 → Green`，`75 → Amber`，`90 → Red`

```sql
SELECT max(value) AS peakCpu FROM CpuUtilization
```

**W3 ·峰值内存** — N，`%`，阈值`0 → Green`，`80 → Amber`，`92 → Red`

```sql
SELECT max(value) AS peakMemory FROM MemoryUtilization
```

**W4 ·最大负载5m** — N

```sql
SELECT max(value) AS maxLoad FROM LoadAverage5m
```

**W5 ·主机CPU** — L， `%`， 6×4

```sql
SELECT average(value) AS cpu FROM CpuUtilization FACET host TIMESERIES AUTO
```

**W6 ·按主机列出的内存** — L， `%`， 6×4

```sql
SELECT average(value) AS memory FROM MemoryUtilization FACET host TIMESERIES AUTO
```

**W7-W10 ·按模式CPU** — L，`%`，各3×4

```sql
SELECT average(value) AS userPct FROM CpuUserPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS systemPct FROM CpuSystemPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS iowaitPct FROM CpuIowaitPercent FACET host TIMESERIES AUTO LIMIT 10
```

```sql
SELECT average(value) AS stealPct FROM CpuStealPercent FACET host TIMESERIES AUTO LIMIT 10
```

**W11 ·主机快照** — T，12×5

```sql
SELECT average(value) AS avgCpu, percentile(value, 95) AS p95Cpu, max(value) AS peakCpu, latest(value) AS nowCpu FROM CpuUtilization FACET host ORDER BY p95Cpu DESC LIMIT 100
```

**W12 · CPU排名最前的进程** — T，6×5

```sql
SELECT average(value) AS cpuPct, max(value) AS peakPct FROM ProcessCpuUtilization FACET host, process.name ORDER BY cpuPct DESC LIMIT 30
```

**W13 ·按内存排在前面的进程** — T， `MB`， 6×5

```sql
SELECT average(value) AS memMb, max(value) AS peakMb FROM ProcessMemoryUsage FACET host, process.name ORDER BY memMb DESC LIMIT 30
```

**存储和网络选项卡**

**W14 ·每次装载使用的磁盘** — T， `%`

```sql
SELECT latest(value) AS usedPct, max(value) AS peakPct FROM DiskUtilization FACET host, mountpoint ORDER BY usedPct DESC LIMIT 50
```

**W15 ·最低可用根磁盘** — N，单位`bytes`，阈值`0 → Red`，`5000000000 → Amber`，`20000000000 → Green`

```sql
SELECT min(value) AS freeBytes FROM DiskFreeBytes
```

**W16 / W17 ·磁盘吞吐量** — L， `bytes/s`

```sql
SELECT rate(value) AS readBytesPerSec FROM DiskReadRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS writeBytesPerSec FROM DiskWriteRate FACET host TIMESERIES AUTO
```

**W18 / W19 ·网络吞吐量** — L， `bytes/s`

```sql
SELECT rate(value) AS rxBytesPerSec FROM NetworkReceiveRate FACET host TIMESERIES AUTO
```

```sql
SELECT rate(value) AS txBytesPerSec FROM NetworkTransmitRate FACET host TIMESERIES AUTO
```

**W20 ·一台主机的网络I/O总数（公式）** — 公式小组件，L， `bytes/s`，12×4。 公式不能`FACET`，且`rate()`的范围必须限制到一台主机（请参阅[策划的量度实体](#53-curated-metric-entities)），因此请固定该主机 — 将`web-publish-01`替换为W5中的实际主机名。

查询`netRx`：

```sql
SELECT rate(value) AS v FROM NetworkReceiveRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

查询`netTx`：

```sql
SELECT rate(value) AS v FROM NetworkTransmitRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

公式：

```text
netRx + netTx
```



### JVM运行状况展示板

**目标：**&#x200B;捕获栈压力、线程泄漏和JVM CPU峰值，然后才会导致中断。

```text
│ N Heap used % (formula) │ N Peak threads │ N JVM CPU % │ N Max heap MB │
│ L Heap used % over time – formula (12w)                                │
│ L Heap used by host (6w) │ T Heap limit by host (6w)                    │
│ L Threads by host (6w)   │ L JVM CPU by host (6w)                       │
│ T Memory pools (12w)                                                   │
```

**W1 ·栈已使用% （公式）** — 公式，N，`%`，阈值`0 → Green`，`75 → Amber`，`90 → Red`

查询`used`：

```sql
SELECT average(value) AS v FROM JvmHeapUsed
```

查询`limit`：

```sql
SELECT average(value) AS v FROM JvmHeapLimit
```

公式：

```text
(used / limit) * 100
```

**W2 ·峰值线程** — N

```sql
SELECT max(value) AS peakThreads FROM JvmThreadCount
```

**W3 · JVM CPU** — N， `%`

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization
```

**W4 ·已使用的最大栈数** — N， `MB`

```sql
SELECT max(value) AS maxHeapMb FROM JvmHeapUsed
```

**W5 ·栈已使用% （公式）** — 公式， L， `%`， 12×4

查询`used`：

```sql
SELECT average(value) AS v FROM JvmHeapUsed TIMESERIES AUTO
```

查询`limit`：

```sql
SELECT average(value) AS v FROM JvmHeapLimit TIMESERIES AUTO
```

公式：

```text
(used / limit) * 100
```

>[!NOTE]
>
> 若要将公式限定在一台主机中，请将`WHERE host = 'my-publish-01'`添加到&#x200B;**两个**&#x200B;查询。

**W6 ·主机使用的栈** — L， `MB`， 6×4

```sql
SELECT average(value) AS heapMb FROM JvmHeapUsed FACET host TIMESERIES AUTO
```

**W7 ·按主机列出的栈限制** — T， `MB`， 6×4

```sql
SELECT latest(value) AS heapLimitMb FROM JvmHeapLimit FACET host LIMIT 50
```

**W8 ·主机Threads** — L，6×4

```sql
SELECT latest(value) AS threads FROM JvmThreadCount FACET host TIMESERIES AUTO
```

**W9 ·主机**&#x200B;的JVM CPU — L， `%`， 6×4

```sql
SELECT average(value) AS jvmCpu FROM JvmCpuRecentUtilization FACET host TIMESERIES AUTO
```

**W10 ·内存池** — T，12×5

```sql
SELECT latest(value) AS bytesUsed, max(value) AS peakBytes FROM MetricSum WHERE metric = 'jvm.memory.used' FACET host, pool ORDER BY peakBytes DESC LIMIT 100
```



### 11.8执行/NOC墙板

**目标：**&#x200B;带有颜色编码的可浏览墙屏。 大多数情况下使用阈值的Number小组件，每行一个趋势。

```text
│ N Requests │ N Availability % (formula) │ N Apdex │ N p95 │ N 5xx │ N Hosts │
│ L Throughput (12w)                                                          │
│ B Requests by service (6w) │ P Traffic by tier (6w)                         │
│ Text: "SLO: 99.9 % availability, p95 < 800 ms. Owner: platform-sre" (12w×1h) │
```

**W1 ·请求** — N

```sql
SELECT count(*) AS requests FROM Transaction
```

**W2 ·可用性% （公式）** — 公式，N，`%`，阈值`0 → Red`，`99 → Amber`，`99.9 → Green`

查询`ok`：

```sql
SELECT count(*) AS v FROM Transaction WHERE status != 'Error'
```

查询`total`：

```sql
SELECT count(*) AS v FROM Transaction
```

公式：

```text
(ok / total) * 100
```

**W3 · Apdex** — N，阈值`0 → Red`，`0.7 → Amber`，`0.85 → Green`

```sql
SELECT apdex(duration, 500) AS apdex FROM Transaction
```

**W4 · p95** — N，`ms`，阈值`0 → Green`，`800 → Amber`，`2000 → Red`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction
```

**W5 · 5xx** — N，阈值`0 → Green`，`1 → Amber`，`50 → Red`

```sql
SELECT count(*) AS http5xx FROM Transaction WHERE http.statusCode >= 500
```

**W6 ·主机** — N

```sql
SELECT uniqueCount(host) AS hosts FROM Transaction
```

**W7 ·吞吐量** — L， 12×4

```sql
SELECT count(*) AS requests FROM Transaction TIMESERIES AUTO
```

**W8 ·服务请求** — B，6×4

```sql
SELECT count(*) AS requests FROM Transaction FACET service LIMIT 10
```

**W9 ·按层**&#x200B;的流量 — P，6×4

```sql
SELECT count(*) AS requests FROM Transaction WHERE hostTierPrefix != '' FACET hostTierPrefix LIMIT 10
```

**W10 · SLO注释** — 文本小组件，12×1。



### 比例和预留空间展示板（公式展示）

**目标：**&#x200B;派生的KPI需要多个查询。

**W1 ·每个主机的请求数** — 公式， L， `req/host`

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

**W2 · 5xx共享所有错误** — 公式， N， `%`

查询`e5xx`：

```sql
SELECT count(*) AS v FROM Transaction WHERE http.statusCode >= 500
```

查询`errors`：

```sql
SELECT count(*) AS v FROM Transaction WHERE http.statusCode >= 400
```

公式：

```text
(e5xx / errors) * 100
```

**W3 ·数据库中的时间与服务器总时间** — 公式，L， `%`

查询`db`：

```sql
SELECT sum(duration) AS v FROM Span WHERE db.system != '' TIMESERIES AUTO
```

查询`server`：

```sql
SELECT sum(duration) AS v FROM Transaction TIMESERIES AUTO
```

公式：

```text
(db / server) * 100
```

**W4 ·栈余量(MB)** — 公式，L，`MB`

查询`limit`：

```sql
SELECT average(value) AS v FROM JvmHeapLimit TIMESERIES AUTO
```

查询`used`：

```sql
SELECT average(value) AS v FROM JvmHeapUsed TIMESERIES AUTO
```

公式：

```text
limit - used
```

**W5 ·一台主机的网络出/入比率** — 公式，L （固定主机；请参阅[策划的度量实体](#53-curated-metric-entities)）

查询`rx`：

```sql
SELECT rate(value) AS v FROM NetworkReceiveRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

查询`tx`：

```sql
SELECT rate(value) AS v FROM NetworkTransmitRate WHERE host = 'web-publish-01' TIMESERIES AUTO
```

公式：

```text
tx / rx
```

**W6 ·发布与作者流量比率** — 公式，N

查询`publish`：

```sql
SELECT count(*) AS v FROM Transaction WHERE host LIKE '%publish%'
```

查询`author`：

```sql
SELECT count(*) AS v FROM Transaction WHERE host LIKE '%author%'
```

公式：

```text
publish / author
```

**W7 ·加权错误预算刻录(SLO 99.9 %)** — 公式，L

查询`errors`：

```sql
SELECT count(*) AS v FROM Transaction WHERE status = 'Error' TIMESERIES AUTO
```

查询`total`：

```sql
SELECT count(*) AS v FROM Transaction TIMESERIES AUTO
```

公式（燃耗率：1.0 =完全按计划消耗预算）：

```text
(errors / total) / 0.001
```



### 服务浏览器（变量驱动）

**目标：**&#x200B;任何团队都可以指向&#x200B;_其_&#x200B;服务和环境的一个讨论区。

>[!WARNING]
>
> 请参阅[教程3步骤8.7](#87-make-dashboards-reusable-with-variables)中的&#x200B;**当前生成限制**：尚未预览或保存构件SQL中的`{{variables}}`。 在解决此问题之前，请使用文字（例如`'checkout'`）替换每个`{{…}}`，并使用&#x200B;_复制_&#x200B;为每个服务克隆展示板。 当支持端到端替代时，以下设计已准备就绪。

**变量**

| 名称 | 标签 | 类型 | 多个 | 数字 | 定义 |
| -------- | ------------------- | ------------ | ----- | ------- | ---------------------------------------------------------- |
| `svc` | 服务 | 从查询 | ✔ |         | `SELECT count(*) FROM Transaction FACET service LIMIT 200` |
| `env` | 环境 | 从查询 | ✔ |         | `SELECT count(*) FROM Span FACET environment LIMIT 50` |
| `slowMs` | 慢速阈值（毫秒） | 文本框 |       | ✔ | 默认`2000` |

```text
Section: "{{svc}} in {{env}}"
│ N Requests │ N Error % │ N p95 │ N Slow requests │
│ L Throughput by service (6w) │ L p95 by service (6w)   │
│ T Operations (12w)                                     │
│ T Slow requests over {{slowMs}} ms – events (12w)       │
│ T Downstream calls (12w)                               │
```

**W1 ·请求 —{{svc}}** — N

```sql
SELECT count(*) AS requests FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W2 ·错误%** — N， `%`

```sql
SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W3 · p95** — N， `ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}})
```

**W4 ·请求缓慢** — N

```sql
SELECT count(*) AS slow FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) AND duration > {{slowMs}}
```

**W5 ·服务吞吐量** — L

```sql
SELECT count(*) AS requests FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service TIMESERIES AUTO
```

**W6 · p95，按服务** — L，`ms`

```sql
SELECT percentile(duration, 95) AS p95 FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service TIMESERIES AUTO
```

**W7 ·操作** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, percentage(*, WHERE status = 'Error') AS errorPct FROM Transaction WHERE service IN ({{svc}}) AND environment IN ({{env}}) FACET service, name ORDER BY calls DESC LIMIT 100
```

**W8 ·请求缓慢（事件）** — T

```sql
SELECT timestamp, service, host, name, urlPath, duration, traceId FROM Transaction WHERE service IN ({{svc}}) AND duration > {{slowMs}} ORDER BY duration DESC LIMIT 200
```

**W9 ·下游调用** — T

```sql
SELECT count(*) AS calls, percentile(duration, 95) AS p95, count(*) FILTER (WHERE status = 'Error') AS errors FROM Span WHERE kind = 'Client' AND service IN ({{svc}}) FACET peer.service, externalDomain, db.system ORDER BY calls DESC LIMIT 50
```
