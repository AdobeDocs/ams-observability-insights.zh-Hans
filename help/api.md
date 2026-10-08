---
title: 可观察性分析公共API
description: 可观察性分析公共API允许您直接将自己的可观察性数据（请求概述、服务目录、跟踪和量度）提取到您自己的工具、脚本和仪表板中。
product_v2:
  - id: fd1f54a9-f50c-467d-8956-cebbaf4f3eb8
    internal-label: Experience Manager
source-git-commit: e0cc17c9d725cad021ba99da4332bca176eae6db
workflow-type: tm+mt
source-wordcount: '1135'
ht-degree: 7%
---

# 可观察性分析公共API

可观察性分析公共API允许您直接将自己的可观察性数据（请求概述、服务目录、跟踪和量度）提取到您自己的工具、脚本和仪表板中。

- **API基本URL (API_BASE_URL)：** `https://insights.adobecqms.net/`
- **格式：** HTTPS上的JSON
- **身份验证：** API密钥（持有者令牌）

> 使用您的Observability Insights实例的API主机（例如`https://insights.adobecqms.net/`）替换本文档中的`{{API_BASE_URL}}`。

## &#x200B;1. 获取API密钥

API密钥是与您的帐户关联的个人凭据，其作用域限于单个组织。 密钥只能读取属于其创建对象的组织的租户的数据 — 它永远看不到其他组织的数据。

### 生成密钥

1. 登录到[可观察性分析仪表板](https://insights.adobecqms.net/)。
2. 打开&#x200B;**API密钥**→配置文件菜单（右上方）。
   ![API密钥菜单](v2-assets/api-key.png)
3. 在&#x200B;**API密钥**&#x200B;选项卡中，单击&#x200B;**生成密钥**。
   ![生成API密钥](v2-assets/api-key-gen.png)
4. 为其指定一个描述性名称（例如`CI pipeline`、`Grafana datasource`），选择其作用域应涵盖的组织，并（可选）设置过期日期。
5. 单击&#x200B;**生成密钥**。 您的密钥以&#x200B;**一次显示**，格式为：

   ```
   synx_9pQ2v6f1WYbLZk3n0aRtEo4jXcHsVmDgUiPq7B8l1yc
   ```

   **立即复制它并将其存储在安全位置**（密钥管理器、CI密钥存储等）  — 仪表板无法再次向您显示。 如果您丢失了它，请撤销它并生成一个新的存储库。

### 管理现有密钥

API密钥部分列出了您创建的每个密钥，包括其组织、创建日期、到期和上次使用时间戳。 单击&#x200B;**撤销**&#x200B;键旁边的垃圾桶图标 — 撤销是立即的，无法撤消。

### 密钥安全性

- 将API密钥完全视为密码。 拥有密钥的任何人都可以读取其作用域所属组织中每个租户的所有可观察性数据，直到数据被吊销或过期。
- 切勿向源代码管理提交密钥或以纯文本（聊天、电子邮件、票证）共享密钥。
- 定期轮换密钥并撤销任何不再使用的密钥。
- 如果某个密钥被泄漏，请立即从&#x200B;**组织设置→API密钥**&#x200B;中撤销它，并生成替换。


## &#x200B;2. 验证请求

对公共API的每个请求都必须在`Authorization`标头中包含您的密钥：

```
Authorization: Bearer synx_9pQ2v6f1WYbLZk3n0aRtEo4jXcHsVmDgUiPq7B8l1yc
```

没有有效密钥或密钥已过期/已撤消的请求将接收`401 Unauthorized`。 会话登录（浏览器Cookie/令牌）在此API中&#x200B;**未被接受**。

## &#x200B;3. 基本概念

### 租户

每个端点都需要一个`tenant_id`查询参数，用于标识要读取的租户数据。 键只能查询属于创建它的组织的租户；请求该组织外部的租户将返回`403 Forbidden`。 此API上没有“所有租户”模式 — 始终传递特定`tenant_id`。

不确定您的密钥可以使用哪些`tenant_id`值？ 调用[`GET /public/v1/tenants`](#get-publicv1tenants) — 它列出了您的密钥有权查询的租户。

### 时间范围

接受`from` / `to`参数的端点采用Unix时间戳（秒）、毫秒时间戳或ISO 8601日期时间字符串，例如：

```
from=1735689600
from=2025-01-01T00:00:00Z
```

如果忽略，则大多数端点将默认使用最近滚动的时段（请参阅下面的每个端点）。

### 速率限制

请求具有每个API密钥的速率限制。 如果超过限制，您将收到：

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60

{ "error": "Too Many Requests", "message": "Rate limit of 300 requests/60s exceeded" }
```

在`Retry-After`标头中的秒数后回退并重试。 如果您的用例需要更高的限制，请联系支持人员。

### 错误数

错误以JSON格式返回，带有`error`字段，通常为人类可读的`message`：

```json
{ "error": "Bad Request", "message": "tenant_id is required" }
```

| 状态 | 含义 |
| ------------------------- | ------------------------------------------------------------------ |
| `400 Bad Request` | 参数缺失或无效（例如，无`tenant_id`，错误的时间范围） |
| `401 Unauthorized` | API密钥缺失、无效、已过期或已撤销 |
| `403 Forbidden` | 该密钥未授权给所请求的租户 |
| `429 Too Many Requests` | 超出速率限制 — 请参阅`Retry-After` |
| `502 Bad Gateway` | 上游查询失败 — 可安全重试 |
| `503 Service Unavailable` | 数据后端暂时不可用 |

## &#x200B;4. 端点

### `GET /public/v1/tenants`

列出您的密钥有权查询的租户ID。 首先调用此项 — 其他每个端点都需要这些值之一作为`tenant_id`。

```bash
curl -s "{{API_BASE_URL}}/public/v1/tenants" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{ "tenants": ["tenant1", "tenant2"] }
```

### `GET /public/v1/overview`

租户在一段时间内的高级别运行状况KPI：请求量、错误率和延迟百分点。

| 参数 | 必需 | 描述 |
| ------------ | -------- | ------------------------------------------------------------------------- |
| `tenant_id` | 是 | 要查询的租户 |
| `from`, `to` | 否 | 时间范围（请参阅[时间范围](#time-ranges)） |
| `minutes` | 否 | 如果未提供`from`/`to`，则为“最近N分钟”的简写（默认`15`） |

```bash
curl -s "{{API_BASE_URL}}/public/v1/overview?tenant_id=<tenant_id>&minutes=30" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "from": 1735689000,
  "to": 1735690800,
  "total_spans": 48213,
  "errors": 112,
  "error_rate_pct": 0.23,
  "p50_ms": 34,
  "p95_ms": 210,
  "p99_ms": 480,
  "service_count": 12,
  "trace_count": 9021
}
```

### `GET /public/v1/services`

列出租户的不同服务名报告。

| 参数 | 必需 | 描述 |
| ------------ | -------- | --------------------------------------------------------------------- |
| `tenant_id` | 是 | 要查询的租户 |
| `from`, `to` | 否 | 限制在此窗口中看到的服务；默认为过去7天 |

```bash
curl -s "{{API_BASE_URL}}/public/v1/services?tenant_id=<tenant_id>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "services": ["checkout-api", "payments-worker", "web-frontend"]
}
```

### `GET /public/v1/traces`

使用可选过滤器搜索租户的最近跟踪。

| 参数 | 必需 | 描述 |
| ----------------- | -------- | ------------------------------------------------- |
| `tenant_id` | 是 | 要查询的租户 |
| `from`, `to` | 否 | 时间范围；默认为过去24小时 |
| `limit` | 否 | 要返回的最大行数（1-200，默认为100） |
| `offset` | 否 | 分页偏移（默认为0） |
| `service` | 否 | 按服务名称筛选 |
| `app_name` | 否 | 按应用程序/实例名称筛选 |
| `status` | 否 | 按跟踪状态筛选： `ok`、`error`或`unset` |
| `search` | 否 | 跨跨跨范围/操作名称的自由文本搜索 |
| `min_duration_ms` | 否 | 仅跟踪达到或超过此持续时间 |

```bash
curl -s "{{API_BASE_URL}}/public/v1/traces?tenant_id=<tenant_id>&status=error&limit=25" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "data": [
    {
      "TraceId": "4bf92f3577b34da6a3ce929d0e0e4736",
      "ServiceName": "checkout-api",
      "DurationMs": 812,
      "StatusCode": "Error",
      "Timestamp": "2026-08-30T09:12:44Z"
    }
  ],
  "rows": 137,
  "limit": 25,
  "offset": 0
}
```

将`rows` （总匹配计数）与`limit`/`offset`一起使用以翻阅结果。

### `GET /public/v1/traces/:traceId`

返回单个跟踪的完整跨度瀑布。

| 参数 | 必需 | 描述 |
| ----------- | -------- | ---------------------------------------- |
| `tenant_id` | 是 | 跟踪所属的租户 |
| `limit` | 否 | 要返回的最大跨距（1-500，默认为500） |
| `offset` | 否 | 非常大的跟踪的分页偏移 |

```bash
curl -s "{{API_BASE_URL}}/public/v1/traces/4bf92f3577b34da6a3ce929d0e0e4736?tenant_id=<tenant_id>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "spans": [
    {
      "SpanId": "00f067aa0ba902b7",
      "Name": "POST /checkout",
      "DurationMs": 812,
      "children": []
    }
  ],
  "totalDurationMs": 812,
  "spanCount": 14,
  "limit": 500,
  "offset": 0
}
```

### `GET /public/v1/metrics`

返回租户的原始量度数据点。

| 参数 | 必需 | 描述 |
| ---------------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `tenant_id` | 是 | 要查询的租户 |
| `metric` | `metric`/`like`之一 | 确切量度名称 |
| `like` | `metric`/`like`之一 | SQL `LIKE`模式匹配多个度量名称 |
| `type` | 否 | `gauge` （默认）或`sum` |
| `from`, `to` | 否 | 时间范围；默认为过去24小时 |
| `service` | 否 | 按服务名称筛选 |
| `host` | 否 | 按主机名筛选。 下面的基础架构主机量度需要 — 如果没有它，租户中每个主机的读数会混合在一起 |
| `attribute_key`, `attribute_value` | 否 | 按特定量度属性过滤（必须一起使用） |

```bash
curl -s "{{API_BASE_URL}}/public/v1/metrics?tenant_id=<tenant_id>&metric=jvm.memory.used&type=gauge" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "data": [
    {
      "TimeUnix": "2026-08-30T09:00:00Z",
      "MetricName": "jvm.memory.used",
      "Value": 512482816,
      "ServiceName": "checkout-api",
      "host": ""
    }
  ],
  "rows": 1
}
```

#### 基础架构主机指标

同一端点还提供基础架构仪表板（CPU、内存、平均负载、磁盘I/O、网络I/O）上显示的主机级指标。 使用这些确切的`metric` / `attribute_key` / `attribute_value`组合，始终使用`host`：

| 仪表板小组件 | `metric` | `attribute_key` | `attribute_value` |
| --------------------- | ------------------------------------- | --------------- | ------------------------------------------------------------------------------------------- |
| CPU % | `system.cpu.utilization` | `state` | `idle` （从1减去“使用中”），或分别查询`user`/`system`/`iowait`并求和 |
| 内存使用率% | `system.memory.utilization` | `state` | `used` |
| 平均负载（1分钟） | `system.cpu.load_average.1m` | — | — |
| 磁盘读取I/O | `system.disk.io` (`type=sum`) | `direction` | `read` |
| 磁盘写入I/O | `system.disk.io` (`type=sum`) | `direction` | `write` |
| 磁盘读取操作 | `system.disk.operations` (`type=sum`) | `direction` | `read` |
| 磁盘写入操作 | `system.disk.operations` (`type=sum`) | `direction` | `write` |
| 中的网络 | `system.network.io` (`type=sum`) | `direction` | `receive` |
| 网络输出 | `system.network.io` (`type=sum`) | `direction` | `transmit` |

```bash
curl -s "{{API_BASE_URL}}/public/v1/metrics?tenant_id=<tenant_id>&metric=system.cpu.utilization&type=gauge&attribute_key=state&attribute_value=idle&host=<host_name>" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

**重要信息 — 磁盘和网络值是原始值，不断增加的计数器，而不是速率。** 仪表板的“字节数/秒”和“操作数/秒”图表是通过计算两个连续的计数器读数并除以经过的时间：

```
rate = (value_at_t2 - value_at_t1) / (t2 - t1_in_seconds)
```

### `GET /public/v1/pages`

每个Dispatcher实例最常请求的内容页面(`.html`)，按请求数量排名。 受`dispatcher.httpd.requests`指标支持 — 此端点特定于AEM Dispatcher/CDN样式访问日志，而不是一般的页面分析工具。

| 参数 | 必需 | 描述 |
| ------------ | -------- | -------------------------------------- |
| `tenant_id` | 是 | 要查询的租户 |
| `from`, `to` | 否 | 时间范围；默认为过去24小时 |
| `limit` | 否 | 要返回的最大行数（1-500，默认为50） |

```bash
curl -s "{{API_BASE_URL}}/public/v1/pages?tenant_id=<tenant_id>&limit=50" \
  -H "Authorization: Bearer $Observability_Insights_API_KEY"
```

```json
{
  "tenant_id": "<tenant_id>",
  "from": 1735689000,
  "to": 1735690800,
  "data": [
    {
      "instance": "<instance_name>",
      "domain": "www.abc.com",
      "path": "/join-us/insights.html",
      "full_url": "https://www.abc.com/join-us/insights.html",
      "requests": 7
    }
  ],
  "rows": 1
}
```

## &#x200B;5. 此API没有执行的操作

- **无原始SQL访问权限。** 所有端点都返回策划的专门构建的数据形状 — 不能直接查询基础数据存储。
- **没有跨租户查询。** 每个请求的范围恰好为一个`tenant_id`。
- **没有写入权限。** 公共API是只读的。

## &#x200B;6. 支持

如果您遇到意外错误，或遇到这些端点未涵盖的用例，请联系您的客户成功/支持工程师以获取进一步帮助。
