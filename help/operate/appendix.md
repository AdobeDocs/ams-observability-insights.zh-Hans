---
title: 附录
description: 有关可观察性见解SQL查询和仪表板的参考补充信息。
source-git-commit: 143a9567da06f0137ef690795cfbc41bda1780ca
workflow-type: tm+mt
source-wordcount: '145'
ht-degree: 5%
---

## 附录 {#appendix}

**在此文档中：** [14.1限制](#141-limits) · [14.2导出文件格式](#142-export-file-format) · [14.3一页备忘单](#143-one-page-cheat-sheet)

### 限制

| 项目 | 限制 |
| -------------------------------- | ----------------------------------------------------------------------- |
| 查询文本 | 4 KB |
| `FACET`属性 | 5 |
| `LIMIT` | 1 - 5 000（默认值：数字1，彩块化10，事件100，时间序列5 000） |
| `IN (…)`值 | 1 - 100 |
| 时间窗口 | 31天 |
| 查询超时 | 15秒（加上字节读取预算） |
| `TIMESERIES AUTO`最小分段 | 2 分钟 |
| 公式 | 2 - 6个查询；表达式≤500个字符；名称≤40个字符 |
| 仪表板 | ≤ 40个选项卡·每个选项卡≤ 200个小组件· ≤ 50个变量 |
| 小组件 | 标题≤ 200 ·子标题≤ 500 ·单元≤ 40 · ≤ 10阈值 |
| 文本小组件 | ≤ 4 000个字符 |
| 变量 | 名称≤ 64个字符· ≤ 500个选项/值·查询≤ 4 KB |

### 导出文件格式

使用&#x200B;**导出**&#x200B;生成此内容；使用&#x200B;**导入**&#x200B;加载此内容。 对于版本控制功能板或在租户之间提升功能板很有用。

```json
{
  "name": "Golden Signals",
  "templateVariables": [
    {
      "id": "var-1",
      "name": "svc",
      "label": "Service",
      "kind": "query",
      "multi": true,
      "query": "SELECT count(*) FROM Transaction FACET service LIMIT 200",
      "value": null
    }
  ],
  "pages": [
    {
      "id": "page-1",
      "name": "Overview",
      "widgets": [
        {
          "id": "sec-kpis",
          "title": "SLO KPIs",
          "queryLanguage": "text",
          "text": "",
          "isSection": true,
          "collapsed": false,
          "layout": { "x": 0, "y": 0, "w": 12, "h": 1 }
        },
        {
          "id": "w-error-rate",
          "title": "Error %",
          "queryLanguage": "SQL",
          "query": "SELECT percentage(*, WHERE status = 'Error') AS errorRate FROM Transaction",
          "vizType": "stat",
          "unit": "%",
          "thresholds": [
            { "value": 0, "status": "ok" },
            { "value": 1, "status": "warning" },
            { "value": 5, "status": "error" }
          ],
          "layout": { "x": 0, "y": 1, "w": 3, "h": 2 }
        },
        {
          "id": "w-throughput",
          "title": "Throughput by service",
          "queryLanguage": "SQL",
          "query": "SELECT count(*) AS requests FROM Transaction FACET service TIMESERIES AUTO LIMIT 8",
          "vizType": "area",
          "layout": { "x": 3, "y": 1, "w": 9, "h": 4 }
        },
        {
          "id": "w-availability",
          "title": "Availability %",
          "queryLanguage": "SQL-formula",
          "formulaQueries": [
            {
              "name": "ok",
              "query": "SELECT count(*) AS v FROM Transaction WHERE status != 'Error'"
            },
            {
              "name": "total",
              "query": "SELECT count(*) AS v FROM Transaction"
            }
          ],
          "formulaExpression": "(ok / total) * 100",
          "vizType": "stat",
          "unit": "%",
          "layout": { "x": 0, "y": 3, "w": 3, "h": 2 }
        }
      ]
    }
  ],
  "exportedAt": "2026-10-01T00:00:00.000Z"
}
```

`vizType`值： `timeseries` （行）、`area`、`bar`、`pie`、`stat` （数字）、`table`。 阈值`status`： `ok` （绿色）、`warning` （琥珀色）、`error` （红色）。

### 单页备忘单

```text
SHAPE        SELECT agg FROM Entity                         -> Number
             ... FACET a[, b..e] LIMIT n                     -> Bar / Pie / Table
             ... TIMESERIES AUTO                             -> Line / Area
             ... FACET a TIMESERIES AUTO LIMIT n             -> top-n series
             SELECT attr1, attr2 FROM Entity LIMIT n         -> event Table

AGGREGATES   count(*)  average(x)  sum(x)  min(x)  max(x)  uniqueCount(x)
             percentile(x, 95)  stddev(x)  latest(x[, rank])  earliest(x[, rank])
             percentage(*, WHERE c)  apdex(duration, 500)  percentOfTotal(x)
             rate(value)        -- counters, TIMESERIES only
             derivative(value)  -- gauges,   TIMESERIES only
             agg FILTER (WHERE c) AS alias

WHERE        = != < <= > >=   LIKE / NOT LIKE ('%', '_')   IN / NOT IN (...)
             IS [NOT] NULL   AND OR NOT ( )   strings in 'single quotes'
             string attr present?  ->  attr != ''

ORDER/LIMIT  ORDER BY alias [ASC|DESC]   LIMIT 1..5000

VARIABLES    = {{name}}   IN ({{name}})   numeric: > {{name}}

FORMULA      2-6 named queries, one metric each, no FACET, same shape
             expression: + - * / ( ) numbers names
```
