## Prestissimo Runtime Metrics Capturing and reporting

Proposers

* Karteek Murthy[https://github.com/karteekmurthys]

## Prototype PR

The approach in this prototype will be discussed in detail.
https://github.com/prestodb/presto/pull/21599

## Summary

Propose design to capture Prestissimo runtime metrics and expose these metrics to a popular time series DB, Prometheus.
Also, discuss a general way to capture and serialize metrics to any custom format.

## Background
### Velox runtime metrics:

Velox (the C++ query engine) defines a set of runtime metrics to record events that give insights into worker health
status like CPU, memory, cache hits etc. Velox exposes the BaseStatsReporter interface to capture these runtime metrics.
It is users’ responsibility to capture these runtime events for monitoring purposes. The Velox system defines the
following types of metrics: Count, Sum, Avg, Rate and Histogram. Refer this document: Velox Runtime metrics for a
complete list of supported Velox runtime metrics and their definitions. There are a total of 28 metrics as of Jan 2024
declared here in the Velox code.

### PrestoCPP Counters:
Prestissimo (the C++ Presto Worker) also defines worker related Prestissimo Counters here. There are 102 counters
defined as of Jan 2024. These metrics are periodically reported using the BaseStatsReporter interface as well.
Additionally, Velox exposes the following interfaces to capture task or expression completion events:

*TaskListener:* This interface exposes TaskListener::onTaskCompletion() method that is called on completion of each task
on a specific worker. The task specific metrics are passed down to this method.

*ExprSetListener:* Similar to TaskListener, this interface exposes ExprSetListener::onCompletion() interface that must
be implemented by the user in order to capture Expression level metrics.

### Prometheus Time Series DB:
Open source Time Series DB(TDB) for capturing and storing time series events. This database supports a Pull model to
fetch metrics from remote exporters. Also, a “PUSH” model where the metrics can be pushed to a Gateway and later pulled
by Prometheus. So, there is no way to synchronously push metrics data to Prometheus. Prometheus can be set up to trigger
alarm on metrics if they breach thresholds or if there are drop in event records. It can be integrated with
visualisation tools like Grafana. It also exposes PromQL, a query language to pull metrics. Within IBM, we use a time series database whose data model is compatible with Prometheus. Hence, the design choice.Prometheus supports COUNT, GAUGE, HISTOGRAM and SUMMARY metric types. Depending on the metric types, we are restricted to using specific PromQl functions. For instance, a rate() function call is only allowed on COUNT type. While we can execute sum() and avg() functions only on GAUGE type.

### Goals

### Non-goals

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
