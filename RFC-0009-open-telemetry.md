# **RFC0009 for Presto**

## Enhancing Open Telemetry Implementation in Presto

Proposers

* Suresh Babu Areekara
* Siddarth Ajay
* Ben Tony Joe

## [Related Issues]

* https://github.com/prestodb/presto/issues/23975

## Summary

The existing Open Telemetry implementation https://github.com/prestodb/presto/pull/18534 was an experimental feature, had a limited set of telemetry data(Query state changes) and did not include a child span concept. The recent implementation will make Presto more flexible, allowing support for both parent and child spans. Additionally, traces can now be propagated to the worker nodes as well.

## Background

OpenTelemetry is a powerful serviceability framework that helps to gain insights into the performance and behaviour of the systems. It facilitates generation, collection, and management of telemetry data such as traces.

The OSS Presto had a basic implementation of Open Telemetry.

![Traces existing implementation](/RFC-0009-open-telemetry/traces-existing-implementation-oss-presto.png)

## Proposed Implementation

Presto can be manually instrumented and will have the following advantages.
- More flexibility and control over instrumentation
- Easier to customize what operations can be monitored
- Ability to pass additional information as span attributes and events

![Instrumentation flow](/RFC-0009-open-telemetry/tracing-instrumentation-flow.png)
- Open Telemetry SDK provides libraries for instrumenting applications to capture telemetry data(traces). It includes built-in integrations for common frameworks and supports custom instrumentation.
- Presto application is getting instrumented using OpenTelemetry API.
- After instrumentation Presto starts the span and register with OpenTelemetry SDK.
- SDK creates context which is the actual association to the flow and attach to the current span(parent).
- While performing any operations, Presto adds the required attributes and events to the respective span.
- In case of sub operations (child span), Presto creates child span, extract the parent context and attach to the child span as parent context so that all parent and child spans get connected.
- After the operation spans will get ended in the order of creation and update the span state.
- The SDK continuously checks the flush trigger, and when it reaches the batch size, all spans are batched and sent to the backend.
- Backend is a system to store, analyse and visualize this telemetry data. Common backends include systems like Jaeger, Instana, Grafana stack, etc.

![Context propagation](/RFC-0009-open-telemetry/context-propagation-coordinator-to-worker.png)

Using context propagation, Signals can be correlated with each other, regardless of where they are generated.

Context contains the information for the sending and receiving service, or execution unit, to correlate one signal with another. For example, if service A calls service B, then a span from service A whose ID is in context can be used as the parent span for the next span created in service B.

Propagation is the mechanism that moves context between services and processes. It serializes or deserializes the context object and provides the relevant information to be propagated from one service to another.

Propagation is usually handled by instrumentation libraries and is transparent to the user. In the event that you need to manually propagate context, you can use the Propagators API.

OpenTelemetry maintains several official propagators. The default propagator is using the headers specified by the W3C TraceContext specification.
- In Presto in areas where REST calls involved, we use the header for context propagation as per the above image.

- Presto Coordinator fetch the current span context and inject as the traceparent http header. Which is then extracted from the Worker side and use to create the child spans with the parent context.

- In some other areas parent context is available in child context and we directly set the parent context in child spans.

### Low level design
![Trace flow](/RFC-0009-open-telemetry/open-telemetry-trace-flow-diagram.png)

## [Optional] Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

## Adoption Plan
Tracing providers can be implemented by extending the `Tracing Factory` interface in SPI module and provide the implementation.

As of now we have one implementation for Open Telemetry can be configured by modifying the values in presto-main/etc/telemetry.properties

```properties
tracing-factory.name=otel
tracing-enabled=false
tracing-backend-url=<backend endpoint>
max-exporter-batch-size=256
max-queue-size=1024
schedule-delay=1000
exporter-timeout=1024
span-sampling=true
```

***tracing-factory.name***: unique identifier for factory implementation to be registered

***tracing-enabled***: boolean value controlling if tracing is on or off

***tracing-backend-url***: points to otel collector or backend for exporting telemetry data

***max-exporter-batch-size***: maximum number of spans that will be exported in one batch

***max-queue-size***: maximum number of spans that can be queued before being processed for export

***schedule-delay***: delay between batches of span export, controlling how frequently spans are exported

***exporter-timeout***: how long the span exporter will wait for a batch of spans to be successfully sent before timing out

***span-sampling***: boolean to enable/disable sampling. If enabled, spans are only generated for major operations

## Test Plan

We have added UT cases for all the OTel implementations and UT span assertion for few major classes where the spans are actually getting generated.
