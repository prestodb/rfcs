# Prestissimo Runtime Metrics Capturing and reporting

## Proposers

* Ahana, an IBM company.

## Prototype PR
There are two approaches in this PR:https://github.com/prestodb/presto/pull/21599 which will be discussed in detail.

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

### Semantics of Velox Runtime metrics
Velox defines macros wrapping calls to BaseStatsReporter. Users can use **DEFINE_METRIC/DEFINE_HISTOGRAM_METRIC** to declare counters/histograms respectively.
To record values against the registered metrics, users invoke **RECORD_METRIC_VALUE/RECORD_HISTOGRAM_METRIC_VALUE**.

1. **COUNT:** RECORD_METRIC_VALUE calls on COUNT type STAT usually don't mention a value in the call parameters, in which case we continuously increment it by 1 and maintain the state. When there is a value passed down, then the counter is incremented by that value. Count type metrics are expected to grow with time and only reset on application restart. Example of such metrics are number of http requests, total number of http requests that errored etc.

2. **SUM:** This STAT type can be assigned to metrics which track aggregate of Operator specific counters. For instance, Velox defines a metric called `velox.spill_input_bytes`. This is a global metric tracking the total spill input bytes across all operators at that instant. Each `Operator` that needs to spill to disk maintains a `Spiller` instance which also has a `SpillStats` member. This instance of SpillStats only tracks Operator specific events. Every time an Operator’s spill stat is updated, in this case, the spill_input_bytes, a Operator thread level counter, is updated as well. At the time of reporting, the sum of `spill_input_bytes` across all threads is gathered and only report the change since the last time aggregated. Note that thread level spill_input_bytes increase with time, they are not reset when an Operator finishes. So, the delta computed at global level across all threads is positive.

3. **AVG:** This stat type can be assigned to counters that grow or decay with time. For instance System or user CPU utilization, System or User memory usage.

4. **HISTOGRAMS:** This stat type is a summary of an event over a period of time. Histograms consists of buckets as keys which represent ranges for a metric and values are the counts representing the number of times the metric was recorded in that range. For instance:

### Current Reporting Flow

#### Event driven metrics reporting (Blocking Call)

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

#### Periodic reporting of Counters (Non Blocking Call)

Majority of metrics in Prestissimo are periodically aggregated and reported via BaseStatsReporter as follows:
1. PrestoServer registers PrestoCPP counters and Velox runtime metrics at the launch of the application.
   Note that we must set the static flag in `BaseStatsReporter::registered=true` before calling register.
   **_PrestoServer::run()→registerPrestoCppCounters()→DEFINE_METRIC(<metric_name>, metric_type)_**
2. PrestoServer starts **_PeriodicTaskManager_** which registers schedulers (callbacks) to collect metrics at regular intervals and report it through BaseStatsReporter.
   ** _PrestoServer::run () → PeriodicTaskManager::start () →RECORD_METRIC_VALUE (<metric_name>, <value>)_ **
3. When RECORD_METRIC_VALUE is invoked for a metric, the implementer of BaseStatsReporter must ensure the value is appropriately recorded depending on its StatType.

### Design to export runtime metrics:

#### How to capture metrics BaseStatsReporter Interface

BaseStatsReporter has following APIS that must be implemented for user to capture metrics:

`void registerMetricExportType(<key>, <type>)` to Register a metric of type COUNT, SUM, AVG.

`void registerHistogramMetricExportType(<key>, <bucketwidth>,<min>,<max>, <vector<pct>>)`  to register a HISTOGRAM.

`void addMetricValue(<key>, <value>)` to add a value for a metric key previously registered.

`void addHistogramMetricValue(<key, <value>)` to add a value to a histogram type metric key.

The metrics are reported by one of the [above] reporting flows. The user implementing BaseStatsReporter must adhere to the above Metrics Semantics of Velox while storing the metrics. That implies, the COUNT type metric must not be overridden and continuously grow with time,  On the other hand, the sum metrics which reports the change in aggregated stats over a period of time will overwrite the value against that metric key. Same applies to AVG, RATE and HISTOGRAMs.

At any instant, we have a snapshot of metrics maintained which is overwritten on new updates.

#### How to Export:
**Prestissimo as exporter:**
To keep it simple, the prestissimo worker will behave like an exporter and exposes the REST endpoint `/v1/info/health/metrics`. The Prometheus server can be configured to pull metrics from the worker itself. Again, it is up to the user if they want to introduce another layer in between the Time series DB and the worker.

**A sidecar metric exporter:**
A sidecar HTTP server that fetches metrics from Prestissimo. This would require launching as a child process of Prestissimo or as a separate container in the same POD as Prestissimo.
Either way, this would take additional resources on the node which we would rather use for Query processsing.

### Storing metric values in BaseStatsReporter:
#### Memory footprint
We have in total 103 + 28 +  7= 138 metric keys defined in PrestoCPP and Velox. Here is the current split by metric type:

| Metric Type | PrestoCPP Count |Velox Count | Total|
| ----------- | --------------- |------------|------|
| COUNT|3| 10| 13   |
|SUM|19|6|25|
|AVG|84|1|85|
|RATE|0|0|0|
|HISTOGRAM|4|11|15|
|||Total|138|

We can maintain a mapping of metric key (which can be anywhere between 30 to 50 characters) and metric value in the impelementation of BaseStatsReporter class.
Each of the above metric value is stored in a 8 byte integer type. At any moment we may have 138 × 8 = 1104 bytes of values data held in-memory. Also, we have 138 × ~50 = 6900 ~ 7K bytes for keys. In total we have 7K + 1104 ~ 8K bytes of memory. Thus holding metrics in memory has least impact on Prestissimo worker.

Given the above estimates, we decide keep the stats in-memory and we can revisit this if the memory footprint grows.

Note: Estimating the size of a histogram with quantiles is not straight forward, it depends on rotation time and error tolerance settings.

### Metric timestamps
Since we overwrite metrics in our current design, it is likely that Prometheus may miss updates as it pulls metrics at regular intervals. The timestamps are assigned by Prometheus server (this is recommended) and may not reflect the exact timestamp at which the metric was recorded.
On other hand, we can maintain a list of metric values and timestamps seen so far in Prestissimo and in the response to the Prometheus we can include this <timestamp, value> pairs. Prometheus must be configured to honor these timestamps. But this approach is not recommended.

### Serialization Format (TBD)
#### Prometheus Data model
We need to define metric labels which are used at Prometheus client end to filter metrics. For instance, a simple label in our case could be cluster name and worker IP. Since metrics are coming from each worker, we need a way to isolate and monitor them. For cluster name, we are relying on the `node.environment` config property and for worker IP, we are relying on the `HOSTNAME` environment variable. Here is a sample of metrics formatted using this data model.
Counter
```
# HELP presto_cpp_http_client_presto_exchange_source_num_on_body
# TYPE presto_cpp_http_client_presto_exchange_source_num_on_body counter
presto_cpp_http_client_presto_exchange_source_num_on_body{cluster="testing",worker="Local"} 5
# TYPE presto_cpp_memory_cache_hit_bytes gauge
presto_cpp_memory_cache_hit_bytes{cluster="testing",worker="Local"} 0
```
Gauge
```
# HELP presto_cpp_mapped_memory_bytes
# TYPE presto_cpp_mapped_memory_bytes gauge
presto_cpp_mapped_memory_bytes{cluster="testing",worker="Local"} 806912
# HELP presto_cpp_os_system_cpu_time_micros
# TYPE presto_cpp_os_system_cpu_time_micros gauge
presto_cpp_os_system_cpu_time_micros{cluster="testing",worker="Local"} 2218
```
Histogram
```
# TYPE velox_hive_file_handle_generate_latency_ms histogram
velox_hive_file_handle_generate_latency_ms_count{cluster="testing",worker=""} 0
velox_hive_file_handle_generate_latency_ms_sum{cluster="testing",worker=""} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="10000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="20000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="30000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="40000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="50000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="60000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="70000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="80000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="90000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="100000"} 0
velox_hive_file_handle_generate_latency_ms_bucket{cluster="testing",worker="",le="+Inf"} 0
```

Summaries
```
# TYPE presto_cpp_http_client_presto_exchange_source_on_body_bytes_summary summary
presto_cpp_http_client_presto_exchange_source_on_body_bytes_summary_count{cluster="testing",worker=""} 0
presto_cpp_http_client_presto_exchange_source_on_body_bytes_summary_sum{cluster="testing",worker=""} 0
presto_cpp_http_client_presto_exchange_source_on_body_bytes_summary{cluster="testing",worker="",quantile="0.5"} Nan
presto_cpp_http_client_presto_exchange_source_on_body_bytes_summary{cluster="testing",worker="",quantile="0.9"} Nan
presto_cpp_http_client_presto_exchange_source_on_body_bytes_summary{cluster="testing",worker="",quantile="0.95"} Nan
presto_cpp_http_client_presto_exchange_source_on_body_bytes_summary{cluster="testing",worker="",quantile="0.99"} Nan
presto_cpp_http_client_presto_exchange_source_on_body_bytes_summary{cluster="testing",worker="",quantile="1"} Nan
# TYPE presto_cpp_presto_exchange_source_serialized_page_size_summary summary
presto_cpp_presto_exchange_source_serialized_page_size_summary_count{cluster="testing",worker=""} 0
presto_cpp_presto_exchange_source_serialized_page_size_summary_sum{cluster="testing",worker=""} 0
presto_cpp_presto_exchange_source_serialized_page_size_summary{cluster="testing",worker="",quantile="0.5"} Nan
presto_cpp_presto_exchange_source_serialized_page_size_summary{cluster="testing",worker="",quantile="0.9"} Nan
presto_cpp_presto_exchange_source_serialized_page_size_summary{cluster="testing",worker="",quantile="0.95"} Nan
presto_cpp_presto_exchange_source_serialized_page_size_summary{cluster="testing",worker="",quantile="0.99"} Nan
presto_cpp_presto_exchange_source_serialized_page_size_summary{cluster="testing",worker="",quantile="1"} Nan
```

#### Serialize using custom interface
By default, we shall implement JSON and Prometheus Data Model serialization of metrics. We shall expose a new Serialization interface that users can implement to customize serialization.
Pros: In-house and no external dependencies.
Cons: It could be challenging to keep it in sync with Prometheus data model versions.
      Adding support for histogrma quantiles is not simple.

#### Serializing using Prometheus-CPP:
Pros: Popular and simplifies histogram and summary metric maintenance.
      The library has implemented unit tests and integration tests for serialization and data correctness.
Cons: External dependency.

Both of these approaches are prototyped in [this PR](https://github.com/prestodb/presto/pull/21599/files#).

## Configuration
Currently, we have `runtime-metrics-collection-enabled` configuration property in Native Presto, which when set to true, starts recording metrics.
Also, proposing a compile to time config PRESTO_ENABLE_PROMETHEUS, which when turned ON, inlcudes prometheus-metrics directory and registers
PrometheusReporter as the metrics reporter.


## Test Plan (TBD)
1. Circle ci job that launches prestissimo
