## Prestissimo Runtime Metrics Capturing and reporting

Proposers

* [Karteek Murthy](https://github.com/karteekmurthys)


## Prototype PR

There are two approaches in this PR which will be discussed in detail.
https://github.com/prestodb/presto/pull/21599

## Summary

Propose design to capture Prestissimo runtime metrics and expose these metrics to a popular time series DB, Prometheus.
Also, discuss a general way to capture and serialize metrics to any custom format.

## Background
### Velox runtime metrics:
Velox (the C++ query engine) defines a set of runtime metrics to record events that give insights into worker health
status like CPU, memory, cache hits etc. Velox exposes the BaseStatsReporter interface to capture these runtime metrics.
It is users’ responsibility to capture these runtime events for monitoring purposes. The Velox system defines the
following types of metrics: Count, Sum, Avg, Rate and Histogram. Refer this document: [Velox Runtime metrics](https://github.com/facebookincubator/velox/blob/main/velox/docs/metrics.rst) for a
complete list of supported Velox runtime metrics and their definitions. There are a total of **28 metrics** as of Jan 2024
declared [here](https://github.com/facebookincubator/velox/blob/main/velox/common/base/Counters.h) in the Velox code.

### PrestoCPP Counters:
Prestissimo (the C++ Presto Worker) also defines worker related Prestissimo Counters [here](https://github.com/prestodb/presto/blob/master/presto-native-execution/presto_cpp/main/common/Counters.h). There are **102 counters**
defined as of Jan 2024. These metrics are periodically reported using the BaseStatsReporter interface as well.
Additionally, Velox exposes the following interfaces to capture task or expression completion events:

### Other Mechanisms
***TaskListener:*** This interface exposes `TaskListener::onTaskCompletion()` method that is called on completion of each task
on a specific worker. The task specific metrics are passed down to this method.

***ExprSetListener:*** Similar to TaskListener, this interface exposes `ExprSetListener::onCompletion()` interface that must
be implemented by the user in order to capture Expression level metrics.

The above mechanisms provide extremely granular stats that provide more context about a query, task or expression behaviors.

### Semantics of Velox Runtime metrics
Velox defines macros wrapping calls to BaseStatsReporter. Users can use **DEFINE_METRIC/DEFINE_HISTOGRAM_METRIC** to declare counters/histograms respectively.
To record values against the registered metrics, users invoke **RECORD_METRIC_VALUE/RECORD_HISTOGRAM_METRIC_VALUE**.

1. **COUNT:** RECORD_METRIC_VALUE calls on COUNT type STAT usually don't mention a value in the call parameters, in which case we continuously increment it by 1 and maintain the state. When there is a value passed down, then the counter is incremented by that value. Count type metrics are expected to grow with time and only reset on application restart. Example of such metrics are number of http requests, total number of http requests that errored etc.

2. **SUM:** This STAT type can be assigned to metrics which track aggregate of Operator specific counters. For instance, Velox defines a metric called `velox.spill_input_bytes`. This is a global metric tracking the total spill input bytes across all operators at that instant. Each `Operator` that needs to spill to disk maintains a `Spiller` instance which also has a `SpillStats` member. This instance of SpillStats only tracks Operator specific events. Every time an Operator’s spill stat is updated, in this case, the spill_input_bytes, a Operator thread level counter, is updated as well. At the time of reporting, the sum of `spill_input_bytes` across all threads is gathered and only report the change since the last time aggregated. Note that thread level spill_input_bytes increase with time, they are not reset when an Operator finishes. So, the delta computed at global level across all threads is positive.

3. **AVG:** This stat type can be assigned to counters that grow or decay with time. For instance System or user CPU utilization, System or User memory usage.

4. **HISTOGRAMS:** This stat type is a summary of an event over a period of time. Histograms consists of buckets as keys which represent ranges for a metric and values are the counts representing the number of times the metric was recorded in that range. For instance:

### Current Reporting Flow**

#### Event driven metrics reporting (Blocking)

Most of the Velox metrics are reported when a specific event occurs. It’s not periodic like Prestissimo counters. For instance, when memory is reclaimed from a query pool, we try to maintain a counter that is incremented each time when this event occurs:
```
uint64_t MemoryReclaimer::run(
    const std::function<uint64_t()>& func,
    Stats& stats) {
  stats.reclaimedBytes += reclaimedBytes;
  RECORD_HISTOGRAM_METRIC_VALUE(
      kMetricMemoryReclaimExecTimeMs, execTimeUs / 1'000);
  RECORD_METRIC_VALUE(kMetricMemoryReclaimedBytes, reclaimedBytes);
  return reclaimedBytes;
}
```
This type of reporting could occur in the execution path of a query and has potential to adversely affect query latency.
Luckily there aren't too many metrics that are reported with this approach. One possible way to counter this is by making an
async call to RECORD_METRIC_VALUE.

#### Periodic reporting of Counters (Non Blocking)

Majority of metrics in Prestissimo are periodically aggregated and reported via BaseStatsReporter as follows:
1. PrestoServer registers PrestoCPP counters and Velox runtime metrics at the launch of the application.
   Note that we must set the static flag in `BaseStatsReporter::registered=true` before calling register.
   **_PrestoServer::run()→registerPrestoCppCounters()→DEFINE_METRIC(<metric_name>, metric_type)_**
2. PrestoServer starts **_PeriodicTaskManager_** which registers schedulers (callbacks) to collect metrics at regular intervals and report it through BaseStatsReporter.
   ** _PrestoServer::run () → PeriodicTaskManager::start () →RECORD_METRIC_VALUE (<metric_name>, <value>)_ **
3. When RECORD_METRIC_VALUE is invoked for a metric, the implementer of BaseStatsReporter must ensure the value is appropriately recorded depending on its StatType.

### Prometheus Time Series DB:
Open source Time Series DB(TDB) for capturing and storing time series events. Prometheus supports COUNT,
GAUGE, HISTOGRAM and SUMMARY metric types. Please efer prometheus metric types [here](https://prometheus.io/docs/concepts/metric_types/#metric-types)
for more details. Depending on the metric types, we are restricted to using specific PromQl
functions. For instance, a rate() function call is only allowed on COUNT type. While we can execute sum() and avg()
functions only on GAUGE type. This database supports a Pull model to
fetch metrics from remote exporters. Also, a “PUSH” model where the metrics can be pushed to a Gateway and later pulled
by Prometheus. So, there is no way to synchronously push metrics data to Prometheus. Prometheus can be set up to trigger
alarm on metrics if they breach thresholds or if there are drop in event records. It can be integrated with
visualisation tools like Grafana. It also exposes PromQL, a query language to pull metrics. Within IBM, we use a time
series database whose data model is compatible with Prometheus. Hence, the design choice.

| Prometheus metric Type       | Velox Stat type |
| ----------- | ----------- |
| COUNT      | COUNT       |
| GAUGE   | SUM, AVG, RATE        |
| HISTOGRAMS (buckets with counts) | No mapping in velox.|
| SUMMARIES (also histograms with quantiles) | HISTOGRAM with quantiles. |

### Goals
The Presto users running C++ worker must be able to see the runtime metrics of worker nodes and generate reports on them. In this document the focus is on metrics exposed by BaseStatsReporter and the details on how to post these metrics into a time series database, Prometheus by representing metric in **Prometheus Data model.**

### Non-goals
This RFC doesn't focus on metrics exposed by TaskListener and ExprSetListener.

## Proposed Implementation

How do you intend to implement the feature? This section can be as detailed as possible with large subsections of its own, or may be a few sentences depending on the scope of the feature proposed. Explain the design in enough detail for existing users/contributors to understand. Design should include all the corner cases you can think of. Feel free to include any new SPI method signatures, class hierarchies or system contracts here. It is recommended to mention any methods, variables, classes, or SQL language additions which you think are needed to provide a broader view of the code changes. Please mention/describe the below on a high level -

1. What modules are involved
2. Any new terminologies/concepts/SQL language additions
3. Method/class/interface contracts which you deem fit for implementation.
4. Code flow using bullet points or pseudo code as applicable
5. Any new user facing metrics that can be shown on CLI or UI.

## Metrics

How can we measure the impact of this feature?

## Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
- If we are changing behaviour how will we phase out the older behaviour?
- If we need special migration tools, describe them here.
- When will we remove the existing behaviour, if applicable.
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Test Plan
