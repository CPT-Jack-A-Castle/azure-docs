---
title: Log-based and pre-aggregated metrics in Azure Application Insights | Microsoft Docs
description: Why to use log-based versus pre-aggregated metrics in Azure Application Insights
ms.topic: conceptual
ms.date: 09/18/2018
ms.reviewer: mbullwin
---

# Log-based and pre-aggregated metrics in Application Insights

This article explains the difference between "traditional" Application Insights metrics that are based on logs, and pre-aggregated metrics. Both types of metrics are available to the users of Application Insights, and each brings a unique value in monitoring application health, diagnostics and analytics. The developers who are instrumenting applications can decide which type of metric is best suited to a particular scenario, depending on the size of the application, expected volume of telemetry, and business requirements for metrics precision and alerting.

## Log-based Metrics

In the past, the application monitoring telemetry data model in Application Insights was solely based on a small number of predefined types of events, such as requests, exceptions, dependency calls, page views, etc. Developers can use the SDK to either emit these events manually (by writing code that explicitly invokes the SDK) or they can rely on the automatic collection of events from auto-instrumentation. In either case, the Application Insights backend stores all collected events as logs, and the Application Insights blades in the Azure portal act as an analytical and diagnostic tool for visualizing event-based data from logs.

Using logs to retain a complete set of events can bring great analytical and diagnostic value. For example, you can get an exact count of requests to a particular URL with the number of distinct users who made these calls. Or you can get detailed diagnostic traces, including exceptions and dependency calls for any user session. Having this type of information can significantly improve visibility into the application health and usage, allowing to cut down the time necessary to diagnose issues with an app.

At the same time, collecting a complete set of events may be impractical (or even impossible) for applications that generate a large volume of telemetry. For situations when the volume of events is too high, Application Insights implements several telemetry volume reduction techniques, such as [sampling](./sampling.md) and [filtering](./api-filtering-sampling.md) that reduce the number of collected and stored events. Unfortunately, lowering the number of stored events also lowers the accuracy of the metrics that, behind the scenes, must perform query-time aggregations of the events stored in logs.

> [!NOTE]
> In Application Insights, the metrics that are based on the query-time aggregation of events and measurements stored in logs are called log-based metrics. These metrics typically have many dimensions from the event properties, which makes them superior for analytics, but the accuracy of these metrics is negatively affected by sampling and filtering.

## Pre-aggregated metrics

In addition to log-based metrics, in late 2018, the Application Insights team shipped a public preview of metrics that are stored in a specialized repository that is optimized for time series. The new metrics are no longer kept as individual events with lots of properties. Instead, they are stored as pre-aggregated time series, and only with key dimensions. This makes the new metrics superior at query time: retrieving data happens much faster and requires less compute power. This consequently enables new scenarios such as [near real-time alerting on dimensions of metrics](../alerts/alerts-metric-near-real-time.md), more responsive [dashboards](./overview-dashboard.md), and more.

> [!IMPORTANT]
> Both, log-based and pre-aggregated metrics coexist in Application Insights. To differentiate the two, in the Application Insights UX the pre-aggregated metrics are now called "Standard metrics (preview)", while the traditional metrics from the events were renamed to "Log-based metrics".

The newer SDKs ([Application Insights 2.7](https://www.nuget.org/packages/Microsoft.ApplicationInsights/2.7.2) SDK or later for .NET) pre-aggregate metrics during collection. This applies to  [standard metrics sent by default](../essentials/metrics-supported.md#microsoftinsightscomponents) so the accuracy isn't affected by sampling or filtering. It also applies to custom metrics sent using [GetMetric](./api-custom-events-metrics.md#getmetric) resulting in less data ingestion and lower cost.

For the SDKs that don't implement pre-aggregation (that is, older versions of Application Insights SDKs or for browser instrumentation) the Application Insights backend still populates the new metrics by aggregating the events received by the Application Insights event collection endpoint. This means that while you don't benefit from the reduced volume of data transmitted over the wire, you can still use the pre-aggregated metrics and experience better performance and support of the near real-time dimensional alerting with SDKs that don't pre-aggregate metrics during collection.

It is worth mentioning that the collection endpoint pre-aggregates events before ingestion sampling, which means that [ingestion sampling](./sampling.md) will never impact the accuracy of pre-aggregated metrics, regardless of the SDK version you use with your application.  

### SDK supported pre-aggregated metrics table

| Current Production SDKs | Standard Metrics (SDK Pre Aggregation) | Custom Metrics (without SDK Pre-Aggregation) | Custom Metrics (with SDK Pre Aggregation)|
|------------------------------|-----------------------------------|----------------------------------------------|---------------------------------------|
| .NET Core and .NET Framework | Supported (V2.13.1+)| Supported via [TrackMetric](api-custom-events-metrics.md#trackmetric)| Supported (V2.7.2+) via [GetMetric](get-metric.md) |
| Java                         | Not Supported       | Supported via [TrackMetric](api-custom-events-metrics.md#trackmetric)| Not Supported                           |
| Node.js                      | Supported (V2.0.0+) | Supported via  [TrackMetric](api-custom-events-metrics.md#trackmetric)| Not Supported                           |
| Python                       | Not Supported       | Supported                                 | Partially supported via [OpenCensus.stats](opencensus-python.md#metrics) |  

> [!NOTE]
>  The metrics implementation for Python using OpenCensus.stats is different from GetMetric. For details see [the Python documentation on metrics](./opencensus-python.md#metrics).

### Codeless supported pre-aggregated metrics table

| Current Production SDKs | Standard Metrics (SDK Pre Aggregation) | Custom Metrics (without SDK Pre-Aggregation) | Custom Metrics (with SDK Pre Aggregation)|
|-------------------------|--------------------------|-------------------------------------------|-----------------------------------------|
| ASP.NET                 | Supported <sup>1<sup>    | Not Supported                             | Not Supported                           |
| ASP.NET Core            | Supported <sup>2<sup>    | Not Supported                             | Not Supported                           |
| Java                    | Not Supported            | Not Supported                             | [Supported](java-in-process-agent.md#metrics) |
| Node.js                 | Not Supported            | Not Supported                             | Not Supported                           |

1. ASP.NET codeless attach on App Service only emits metrics in "full" monitoring mode. ASP.NET codeless attach on App Service, VM/VMSS, and On-Premises emits standard metrics without dimensions. SDK is required for all dimensions.
2. ASP.NET Core codeless attach on App Service emits standard metrics without dimensions. SDK is required for all dimensions.

## Using pre-aggregation with Application Insights custom metrics

You can use pre-aggregation with custom metrics. The two main benefits are the ability to configure and alert on a dimension of a custom metric and reducing the volume of data sent from the SDK to the Application Insights collection endpoint.

There are several [ways of sending custom metrics from the Application Insights SDK](./api-custom-events-metrics.md). If your version of the SDK offers the [GetMetric and TrackValue](./api-custom-events-metrics.md#getmetric) methods, this is the preferred way of sending custom metrics, since in this case pre-aggregation happens inside of the SDK, not only reducing the volume of data stored in Azure, but also the volume of data transmitted from the SDK to Application Insights. Otherwise, use the [trackMetric](./api-custom-events-metrics.md#trackmetric)  method, which will pre-aggregate metric events during data ingestion.

## Custom metrics dimensions and pre-aggregation

All metrics that you send using [trackMetric](./api-custom-events-metrics.md#trackmetric) or [GetMetric and TrackValue](./api-custom-events-metrics.md#getmetric) API calls are automatically stored in both logs and metrics stores. However, while the log-based version of your custom metric always retains all dimensions, the pre-aggregated version of the metric is stored by default with no dimensions. You can turn on collection of dimensions of custom metrics on the [usage and estimated cost](../usage-estimated-costs.md#usage-and-estimated-costs) tab by checking "Enable alerting on custom metric dimensions": 

![Usage and estimated cost](./media/pre-aggregated-metrics-log-metrics/001-cost.png)

## Quotas

Pre-aggregated metrics are stored as time series in Azure Monitor, and [Azure Monitor quotas on custom metrics](../essentials/metrics-custom-overview.md#quotas-and-limits) apply.

> [!NOTE]
> Going over the quota might have unintended consequences. Azure Monitor might become unreliable in your subscription or region. To learn how to avoid exceeding the quota, see [Design limitations and considerations](../essentials/metrics-custom-overview.md#design-limitations-and-considerations).
  
## Why is collection of custom metrics dimensions turned off by default?

The collection of custom metrics dimensions is turned off by default because in the future storing custom metrics with dimensions will be billed separately from Application Insights, while storing the non-dimensional custom metrics will remain free (up to a quota). You can learn about the upcoming pricing model changes on our official [pricing page](https://azure.microsoft.com/pricing/details/monitor/).

## Creating charts and exploring log-based and standard pre-aggregated metrics

Use [Azure Monitor Metrics Explorer](../essentials/metrics-getting-started.md) to plot charts from pre-aggregated and log-based metrics, and to author dashboards with charts. After selecting the desired Application Insights resource, use the namespace picker to switch between standard (preview) and log-based metrics, or select a custom metric namespace:

![Metric namespace](./media/pre-aggregated-metrics-log-metrics/002-metric-namespace.png)

## Pricing models for Application Insights metrics

Ingesting metrics into Application Insights, whether log-based or pre-aggregated, will generate costs based on the size of the ingested data, as described in [Azure Monitor Logs pricing details](../logs/cost-logs.md#application-insights-billing). Your custom metrics, including all its dimensions, are always stored in the Application Insights log-store; additionally, a pre-aggregated version of your custom metrics (with no dimensions) is forwarded to the metrics store by default.

Selecting the [Enable alerting on custom metric dimensions](#custom-metrics-dimensions-and-pre-aggregation) option to store all dimensions of the pre-aggregated metrics in the metric store, can generate **additional** costs based on [Custom Metrics pricing](https://azure.microsoft.com/pricing/details/monitor/).

## Next steps

* [Near real-time alerting](../alerts/alerts-metric-near-real-time.md)
* [GetMetric and TrackValue](./api-custom-events-metrics.md#getmetric)
