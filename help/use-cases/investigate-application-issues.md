---
title: 调查应用程序问题
description: 在AEM Managed Services中，使用可观察性分析APM可区分延迟、错误和吞吐量问题。
feature: Operations
role: Admin
redirect_url: /help/application-performance-monitoring.md
source-git-commit: beb36ab380251b8f325b1c746b72e91b38ae6d2f
workflow-type: tm+mt
source-wordcount: '22'
ht-degree: 0%

---


# 调查应用程序问题

调查。

<!-- 

This content has moved to [Application Performance Monitoring](../application-performance-monitoring.md).

## Investigation workflow {#investigation-workflow}

1. Identify the affected environment and whether the issue is on Author, Publish, or both.
2. Open the relevant APM application and review the overview KPIs.
3. Check RED metrics to determine whether the dominant signal is request volume, error rate, or latency.
4. Review traffic and endpoint-level views to isolate high-impact transactions.
5. Inspect traces to confirm where execution time is spent.
6. Correlate with infrastructure metrics if you suspect host resource pressure.

## Questions to answer during triage {#questions-to-answer-during-triage}

- Did throughput change before the issue, or only after symptoms began?
- Are failures concentrated in one HTTP status band or one endpoint family?
- Is latency elevated broadly, or only for specific transactions?
- Do traces point to repository operations, downstream systems, or application code hot spots?

## Evidence to capture {#evidence-to-capture}

Capture these items when escalating or collaborating with Adobe Managed Services:

- Environment name and time window
- Whether Author, Publish, or both are affected
- Screenshots of overview, RED metrics, and error or latency charts
- Example trace IDs or transaction names
- Any correlated infrastructure anomalies

## Supporting reference {#supporting-reference}

- [Application Performance Monitoring](../application-performance-monitoring.md)
- [APM dashboard reference](../reference/apm-dashboard-reference.md)
- [Infrastructure monitoring](../infrastructure-monitoring.md)

## Add more workflow detail here later {#add-more-workflow-detail-here-later}

Expand this page later with product-specific runbooks such as:

- JVM pressure investigation
- Slow endpoint triage
- Error spike analysis
- External dependency troubleshooting

-->

