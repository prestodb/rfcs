# **RFC0011 for Presto**


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


### Existing Open Telemetry
![Traces existing implementation](/RFC-0011-open-telemetry/traces-existing-implementation-oss-presto.png)


The OSS Presto had a basic implementation of Open Telemetry with few limitations listed below.
- Have only less number of spans and covers only query state transitions like dispatching, planning, analyzing, and
scheduling. ie. it does not cover most of the internal operations during a query execution.
- No additional information is being passed as query attributes or events.
- No correlation between traceId and queryId.
- Does not identify failed queries
- No context propagation to track worker nodes.

## Proposed Implementation
With additional changes to the OpenTelemetry SPI, we get the following advantages -
- Presto can be manually instrumented which gives more flexibility and control. Easier to customize what operations can be monitored.
- More number of spans with parent and child span concept which helps to get the insight on internal operations.
- Able to pass queryId as attribute so that traces and query get connected each other.
- Can identify failed queries and filter it out.
- Ability to pass additional information as span attributes and events.
- Context propagation from coordinator to worker nodes.


### Traces with new implementation
![Traces with new implementation](/RFC-0011-open-telemetry/traces-with-new-implementation.png)


### Tracing Instrumentation Flow
![Instrumentation flow](/RFC-0011-open-telemetry/tracing-instrumentation-flow.png)
- Here the Open Telemetry SDK provides libraries and API's for instrumenting the applications.
- Backend is a system to store, analyse and visualize this telemetry data. Common backends include systems like Jaeger, Instana, Grafana stack, etc.


### Context propagation from Coordinator to Worker
![Context propagation](/RFC-0011-open-telemetry/context-propagation-coordinator-to-worker.png)

- Using context propagation, Signals can be correlated with each other, regardless of where they are generated.
- Context contains the information for the sending and receiving service, or execution unit, to correlate one signal with another. For example, if service A calls service B, then a span from service A whose ID is in context can be used as the parent span for the next span created in service B.
- Propagation is the mechanism that moves context between services and processes. It serializes or deserializes the context object and provides the relevant information to be propagated from one service to another.
- Propagation is usually handled by instrumentation libraries and is transparent to the user. In the event that you need to manually propagate context, you can use the Propagators API.

- OpenTelemetry maintains several official propagators. The default propagator is using the headers specified by the W3C TraceContext specification.
  - In Presto in areas where REST calls involved, we use the header for context propagation as per the above image.
  - Presto Coordinator fetch the current span context and inject as the traceparent http header. Which is then extracted from the Worker side and use to create the child spans with the parent context.
  - In some other areas parent context is available in child context and we directly set the parent context in child spans.


### Low level design
![Trace flow](/RFC-0011-open-telemetry/open-telemetry-trace-flow-diagram.png)


## [Optional] Other Approaches Considered
Based on the discussion, this may need to be updated with feedback from reviewers.


## Adoption Plan
### SPI Changes
```java
class BaseSpan {}
```
SPI can be extended with the specific implementation which contains the actual SDK Span. BaseSpan propagates through the main presto code. Introduced as part of new design.

#### Additions
```java
default void close()
{
    return;
}

default void end()
{
    return;
}
```


```java
class Tracer {}
```
SPI for the span operations which can be implemented by specific serviceability framework.

#### Additions
```java
void loadConfiguredOpenTelemetry();
Runnable getCurrentContextWrap(Runnable runnable);
boolean isRecording();
Map<String, String> getHeadersMap(T span);
void endSpanOnError(T querySpan, Throwable throwable);
void addEvent(T span, String eventName);
void setAttributes(T span, Map<String, String> attributes);
void recordException(T span, String message, RuntimeException runtimeException, ErrorCode errorCode);
void setSuccess(T span);
T getInvalidSpan();
T getRootSpan(String traceId);
T getSpan(String spanName);
U scopedSpan(String name, Boolean... skipSpan);
Optional<String> spanString(T span);
```

#### Deletions
```java
String tracerName = "com.facebook.presto";
void addPoint(String annotation);
void startBlock(String blockName, String annotation);
void addPointToBlock(String blockName, String annotation);
void endBlock(String blockName, String annotation);
void endTrace(String annotation);
```


```java
class TraceProvider {}
```
SPI to supply different Tracer implementations.

#### Additions
```java
T create();
```

#### Deletions
```java
String getTracerType();
Function<Map<String, String>, TracerHandle> getHandleGenerator();
Tracer getNewTracer(TracerHandle handle);
```

```java
class NoopTracer {}
```
Removed as we have a default Tracer implementation in presto-main/..tracing/DefaultTelemetryTracer

#### Deletions
```java
public void addPoint(String annotation)
{
}

@Override
public void startBlock(String blockName, String annotation)
{
}

@Override
public void addPointToBlock(String blockName, String annotation)
{
}

@Override
public void endBlock(String blockName, String annotation)
{
}

@Override
public void endTrace(String annotation)
{
}

@Override
public String getTracerId()
{
    return "noop_dummy_id";
}
```


```java
class TracerHandle {}
```
Removed as not required for the current implementation.

#### Deletions
```java
String getTraceToken();
```


### Configuration
Trace can be configured by modifying the values in presto-main/etc/telemetry-tracing.properties

```properties
tracing-factory.name=otel
tracing-enabled=false
tracing-backend-url=<backend endpoint>
max-exporter-batch-size=256
max-queue-size=1024
schedule-delay=1000
exporter-timeout=1024
trace-sampling-ratio=1.0
span-sampling=true
```

***tracing-factory.name***: Unique identifier for factory implementation to be registered.

***tracing-enabled***: Boolean value controlling if tracing is on or off.

***tracing-backend-url***: Points to backend for exporting telemetry data.

***max-exporter-batch-size***: Maximum number of spans that will be exported in one batch.

***max-queue-size***: Maximum number of spans that can be queued before being processed for export.

***schedule-delay***: Delay between batches of span export, controlling how frequently spans are exported.

***exporter-timeout***: How long the span exporter will wait for a batch of spans to be successfully sent before timing out.

***trace-sampling-ratio***: Double between 0.0 and 1.0 to specify the percentage of queries to be traced.

***span-sampling***: Boolean to enable/disable sampling. If enabled, spans are only generated for major operations.


## Test Plan
We have added UT cases for all the OTel implementations and UT span assertion for few major classes where the spans are actually getting generated.
