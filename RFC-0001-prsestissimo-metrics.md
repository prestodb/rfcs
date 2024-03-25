## **Prestissimo Runtime Metrics Capturing and reporting**


## **Proposers**

* Ahana, an IBM company.
* Uber.

## **Prototype PR**

There are two approaches in this PR:[prestodb/presto#21599](https://github.com/prestodb/presto/pull/21599) which will be discussed in detail.


## **Summary**

Propose design to capture Prestissimo runtime metrics and expose these metrics to a Prometheus Time series DB. Also, discuss a general way to capture and serialise metrics to any custom format.


## **Background**


### **Velox runtime metrics:**

Velox (the C++ runtime framework) defines a set of runtime metrics to record events that give insights into task status like CPU, memory, cache hits etc. Velox exposes the BaseStatsReporter interface to capture these runtime metrics. It is applications’ responsibility to capture these runtime events for monitoring purposes. The Velox system defines the following types of metrics: Count, Sum, Avg, Rate and Histogram. Refer this document: [Velox Runtime metrics](https://github.com/facebookincubator/velox/blob/main/velox/docs/metrics.rst) for a complete list of supported Velox runtime metrics and their definitions. There are a total of 28 metrics as of Jan 2024 declared [here](https://github.com/facebookincubator/velox/blob/main/velox/common/base/Counters.h) in the Velox code.


### **PrestoCPP Counters:**

In addition to Velox metrics, Prestissimo (the C++ Presto Worker) also defines worker related Prestissimo Counters [here](https://github.com/prestodb/presto/blob/master/presto-native-execution/presto_cpp/main/common/Counters.h). There are 102 counters defined as of Jan 2024. These metrics are periodically reported using the BaseStatsReporter interface as well. Additionally, Velox exposes the following interfaces to capture task or expression completion events:


### **Other Mechanisms**

**TaskListener**: This interface exposes TaskListener::onTaskCompletion() method that is called on completion of each task on a specific worker. The task specific metrics are passed down to this method.

**ExprSetListener**: Similar to TaskListener, this interface exposes ExprSetListener::onCompletion() interface that must be implemented by the user in order to capture Expression level metrics.

The above mechanisms provide extremely granular stats that provide more context about a query, task or expression behaviours.


### **Prometheus Time Series DB:**

Open source Time Series DB(TDB) for capturing and storing time series events. Prometheus supports COUNT, GAUGE, HISTOGRAM and SUMMARY metric types. Please refer to Prometheus metric types [here](https://prometheus.io/docs/concepts/metric_types/#metric-types) for more details. It also exposes PromQL, a query language to pull metrics. Depending on the metric types, we are restricted to using specific PromQl functions. For instance, a rate() function call is only allowed on COUNT type. While we can execute **sum()** and **avg() **functions only on GAUGE type. This database supports a Pull model to fetch metrics from remote exporters. Also, a “PUSH” model where the metrics can be pushed to a Gateway and later pulled by Prometheus. So, there is no way to synchronously push metrics data to Prometheus. Prometheus can be set up to trigger alarm on metrics if they breach thresholds or if there are drop in event records. It can be integrated with visualisation tools like Grafana. Within IBM, we use a time series database whose data model is compatible with Prometheus. Hence, the design choice.

Mapping for Velox to Prometheus types: 


<table>
  <tr>
   <td><strong>Prometheus metric Type</strong> 
   </td>
   <td><strong>Velox Stat type</strong> 
   </td>
  </tr>
  <tr>
   <td>COUNT 
   </td>
   <td>COUNT 
   </td>
  </tr>
  <tr>
   <td>GAUGE 
   </td>
   <td>SUM, AVG, RATE
   </td>
  </tr>
  <tr>
   <td>HISTOGRAMS (buckets with counts) 
   </td>
   <td>No mapping in Velox.
   </td>
  </tr>
  <tr>
   <td>SUMMARIES (also histograms with quantiles) 
   </td>
   <td>HISTOGRAM with quantiles. 
   </td>
  </tr>
</table>



#### 
**[Prometheus Data model](https://prometheus.io/docs/concepts/data_model/)**

We need to define metric labels which are used at Prometheus client end to filter metrics. For instance, a simple label in our case could be cluster name and worker IP. Since metrics are coming from each worker, we need a way to isolate and monitor them. For cluster name, we are relying on the `node.environment` config property and for worker IP, we are relying on the `HOSTNAME` environment variable. Here is a sample of metrics formatted using this data model. 

**Counter**
```
# HELP presto_cpp_http_client_presto_exchange_source_num_on_body
# TYPE presto_cpp_http_client_presto_exchange_source_num_on_body counter
presto_cpp_http_client_presto_exchange_source_num_on_body{cluster="testing",worker="Local"} 5
# TYPE presto_cpp_memory_cache_hit_bytes gauge
presto_cpp_memory_cache_hit_bytes{cluster="testing",worker="Local"} 0
```

**Gauge**


```
# HELP presto_cpp_mapped_memory_bytes
# TYPE presto_cpp_mapped_memory_bytes gauge
presto_cpp_mapped_memory_bytes{cluster="testing",worker="Local"} 806912
# HELP presto_cpp_os_system_cpu_time_micros
# TYPE presto_cpp_os_system_cpu_time_micros gauge
presto_cpp_os_system_cpu_time_micros{cluster="testing",worker="Local"} 2218
Histogram, refer to this documentation for detailed explanation.
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


**Summaries**

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



### **Goals**

The Presto users running C++ worker must be able to see the runtime metrics of worker nodes and generate reports on them. In this document the focus is on metrics exposed by BaseStatsReporter and the details on how to post these metrics into a time series database, Prometheus by representing metric in Prometheus Data model.


## **Semantics of Velox Runtime metrics**

Velox defines macros wrapping calls to BaseStatsReporter. Users can use DEFINE_METRIC/DEFINE_HISTOGRAM_METRIC to declare counters/histograms respectively. To record values against the registered metrics, users invoke RECORD_METRIC_VALUE/RECORD_HISTOGRAM_METRIC_VALUE.

1. COUNT: RECORD_METRIC_VALUE calls on COUNT type STAT usually don't mention a value in the call parameters, in which case we continuously increment it by 1 and maintain the state. When there is a value passed down, then the counter is incremented by that value. Count type metrics are expected to grow with time and only reset on application restart. Examples of such metrics are number of http requests, total number of http requests that are errored etc.
2. SUM: Tracks the sum of the inserted values. This STAT type can be assigned to metrics which track aggregate of Operator specific counters. For instance, Velox defines a metric called velox.spill_input_bytes. This is a global metric tracking the total spill input bytes across all operators at that instant. Each Operator that needs to spill to disk maintains a Spiller instance which also has a SpillStats member. This instance of SpillStats only tracks Operator specific events. Every time an Operator’s spill stat is updated, in this case, the spill_input_bytes, a Operator thread level counter, is updated as well. At the time of reporting, the sum of spill_input_bytes across all threads is gathered and only reports the change since the last time aggregated. Note that thread level spill_input_bytes increase with time, they are not reset when an Operator finishes. So, the delta computed at global level across all threads is positive.
3. AVG: Tracks the average of the inserted values. This stat type can be assigned to counters that grow or decay with time. For instance System or user CPU utilisation, System or User memory usage.
1. RATE: Tracks the sum of the inserted values per second. As of now, there are no references to this StatType in the Velox repo.
2. HISTOGRAMS: This stat type is a summary of an event over a period of time. Histograms consist of buckets as keys which represent ranges for a metric and values are the counts representing the number of times the metric was recorded in that range. For instance, kCounterHttpClientPrestoExchangeOnBodyBytes is a histogram with min value 0 and max value 1000000. The size of each bucket is 1000, this Histogram will have (max -min)/bucket-size = (1000000 - 0)/1000 = 1000 buckets. The parameters following the max value are {50,90,95,99,100} which indicate that this Histogram metric tracks 50th, 90th etc. percentiles respectively.

```
DEFINE_HISTOGRAM_METRIC(
      kCounterHttpClientPrestoExchangeOnBodyBytes,
      1000,
      0,
      1000000,
      50,
      90,
      95,
      99,
      100);
```


## **Current Reporting Flow**


### **Event driven metrics reporting (Blocking Call)**

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


This type of reporting could occur in the execution path of a query and has potential to adversely affect query latency. Luckily there aren't too many metrics that are reported with this approach. One possible way to counter this is by making an async call to RECORD_METRIC_VALUE.


#### 
**Periodic reporting of Counters (Non Blocking Call)**

Majority of metrics in Prestissimo are periodically aggregated and reported via BaseStatsReporter as follows:

1. PrestoServer registers PrestoCPP counters and Velox runtime metrics at the launch of the application. Note that we must set the static flag in `BaseStatsReporter::registered=true` before calling register. _PrestoServer::run()→registerPrestoCppCounters()→DEFINE_METRIC(&lt;metric_name>, metric_type)_
2. PrestoServer starts _PeriodicTaskManager_ which registers schedulers (callbacks) to collect metrics at regular intervals and report it through BaseStatsReporter. ** _PrestoServer::run () → PeriodicTaskManager::start () →RECORD_METRIC_VALUE (&lt;metric_name>, )_ **
3. When RECORD_METRIC_VALUE is invoked for a metric, the implementer of BaseStatsReporter must ensure the value is appropriately recorded depending on its StatType.

### 
**Design to export runtime metrics:**


#### 
**How to capture metrics BaseStatsReporter Interface**


BaseStatsReporter has following APIs that must be implemented for user to capture metrics:

`void registerMetricExportType(&lt;key>, &lt;type>)` to Register a metric of type COUNT, SUM, AVG.

`void registerHistogramMetricExportType(&lt;key>, &lt;bucketwidth>,&lt;min>,&lt;max>, &lt;vector&lt;pct>>)` to register a HISTOGRAM.

`void addMetricValue(&lt;key>, &lt;value>)` to add a value for a metric key previously registered.

`void addHistogramMetricValue(&lt;key, &lt;value>)` to add a value to a histogram type metric key.

The metrics are reported by one of the [above] reporting flows. The user implementing BaseStatsReporter must adhere to the above Metrics Semantics of Velox while storing the metrics. That implies, the COUNT type metric must not be overridden and continuously grow with time, On the other hand, the sum metrics which reports the change in aggregated stats over a period of time will overwrite the value against that metric key. Same applies to AVG, RATE and HISTOGRAMs.

At any instant, we have a snapshot of metrics maintained which is overwritten on new updates.


#### 
**How to Export:**

Prestissimo as exporter: To keep it simple, the prestissimo worker will behave like an exporter and expose the REST endpoint `/v1/info/metrics`. The Prometheus server can be configured to pull metrics from the worker itself. Again, it is up to the user if they want to introduce another layer in between the Time series DB and the worker.

A sidecar metric exporter: A sidecar HTTP server that fetches metrics from Prestissimo. This would require launching as a child process of Prestissimo or as a separate container in the same POD as Prestissimo. Pros: In case of Prestissimo crash, we still have a way to get the last set of metrics from the node. To support graceful failure handling, we may have to dump metrics to disk and the exporter must read from disk when Prestissimo is down. Cons: Spawning a new process as a sidecar would take additional memory and CPU.

Currently, we have only prototyped Prestissimo as an exporter. Both approaches can be built using the prometheus-cpp library.


### 
**Storing metric values in BaseStatsReporter:**


#### 
**Memory footprint**

We have in total 103 + 28 + 7= 138 metric keys defined in PrestoCPP and Velox. Here is the current split by metric type:


<table>
  <tr>
   <td><strong>Metric Type</strong>
   </td>
   <td><strong>PrestoCPP Count</strong>
   </td>
   <td><strong>Velox Count</strong>
   </td>
   <td><strong>Total</strong>
   </td>
  </tr>
  <tr>
   <td>COUNT
   </td>
   <td>3
   </td>
   <td>10
   </td>
   <td>13
   </td>
  </tr>
  <tr>
   <td>SUM
   </td>
   <td>19
   </td>
   <td>6
   </td>
   <td>25
   </td>
  </tr>
  <tr>
   <td>AVG
   </td>
   <td>84
   </td>
   <td>1
   </td>
   <td>85
   </td>
  </tr>
  <tr>
   <td>RATE
   </td>
   <td>0
   </td>
   <td>0
   </td>
   <td>0
   </td>
  </tr>
  <tr>
   <td>HISTOGRAM
   </td>
   <td>4
   </td>
   <td>11
   </td>
   <td>15
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>
   </td>
   <td>Total
   </td>
   <td>138
   </td>
  </tr>
</table>


We can maintain a mapping of metric key (which can be anywhere between 30 to 50 characters) and metric value in the implementation of BaseStatsReporter class. Each of the above metric values is stored in a 8 byte integer type. At any moment we may have 138 × 8 = 1104 bytes of values data held in-memory. Also, we have 138 × ~50 = 6900 ~ 7K bytes for keys. In total we have 7K + 1104 ~ 8K bytes of memory. Thus holding metrics in memory has the least impact on Prestissimo workers.

Given the above estimates, we decide to keep the stats in-memory and we can revisit this if the memory footprint grows.

Note: Estimating the size of a histogram with quantiles is not straight forward, it depends on rotation time and error tolerance settings.


### 
**Metric timestamps**

Since we overwrite metrics in our current design, it is likely that Prometheus may miss updates as it pulls metrics at regular intervals. The timestamps are assigned by the Prometheus server (this is recommended) and may not reflect the exact timestamp at which the metric was recorded. On the other hand, we can maintain a list of metric values and timestamps seen so far in Prestissimo and in the response to the Prometheus we can include these &lt;timestamp, value> pairs. Prometheus must be configured to honour these timestamps. But this approach is not recommended.

Prestissimo emits metric values without timestamps. Prometheus DB will be assigning timestamps to the metric values after it pulls from the Prestissimo endpoint.


### 
**Serialisation Format**


#### 
**Serialize using custom interface**

By default, we shall implement JSON and Prometheus Data Model serialisation of metrics. We shall expose a new Serialization interface that users can implement to customize serialization. Pros: In-house and no external dependencies. Cons: It could be challenging to keep it in sync with Prometheus data model versions. Adding support for histogram quantiles is not simple.


#### 
**Serializing using Prometheus-CPP: (Preferred)**

Pros: Popular and simplifies histogram and summary metric maintenance. The library has implemented unit tests and integration tests for serialisation and data correctness. Cons: External dependency.

Both of these approaches are prototyped in [this PR](https://github.com/prestodb/presto/pull/21599/files#).


## 
**Configuration**

Runtime configs:

1. Currently, we have `runtime-metrics-collection-enabled` configuration property in Native Presto, which when set to true, starts recording metrics.
2. (TBD) Configs to optionally enable histogram/summary type metrics which may not be useful for day to day monitoring.
3. (TBD) Mechanism to white list metrics. Compile time configs: Also, proposing a compile to time config PRESTO_ENABLE_PROMETHEUS, which when turned ON, includes prometheus-metrics directory and registers PrometheusReporter as the metrics reporter.



## **Prototype test results**

Prometheus client end visualisation of Prestissimo metrics.
<img width="1689" alt="Screenshot 2024-01-12 at 4 44 57 PM" src="https://github.com/karteekmurthys/rfcs/assets/6971561/b58074b9-5900-49b9-9941-09c1ebdd5747">


Current PR:[ https://github.com/prestodb/presto/pull/21599](https://github.com/prestodb/presto/pull/21599)
