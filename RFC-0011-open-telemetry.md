# **RFC0011 for Presto**


## Enhancing Open Telemetry Implementation in Presto
Proposers
* Suresh Babu Areekara
* Siddarth Ajay
* Ben Tony Joe


## [Related Issues]
* https://github.com/prestodb/presto/issues/23975


## Summary
Presto's original tracing implementation was merged in https://github.com/prestodb/presto/pull/18534. It was an experimental feature, had a limited set of telemetry data, and did not include a child span concept. This document proposes an expansion to the tracing provider SPI to
1. Support child spans in traces to make understanding nested operations simpler
2. Allow traces to be propagated to workers so that the entire lifespan of a query can be traced from analysis through execution


## Background
TracerProvider/Tracer can be extended and implemented with serviceability frameworks. It facilitates generation, collection, and management of telemetry data such as traces.


### Existing implementation
![Traces existing implementation](/RFC-0011-open-telemetry/traces-existing-implementation-oss-presto.png)


Presto has a basic `TracerProvider` and `Tracer` interfaces which makes it difficult to implement the following desirable features:
- Have only limited number of spans and covers only query state transitions like dispatching, planning, analyzing, and
scheduling. ie. it does not cover most of the internal operations during a query execution(The implementation for methods startBlock and endBlock are based on Map and cannot be used to handle large number of spans as the span name can be same for many.).
- No additional information is being passed as query attributes or events. No suitable methods in SPI to set events and attributes.
- No correlation between traceId and queryId. No suitable methods in SPI to set queryId.
- Does not identify failed queries. No suitable methods in SPI to pass Error/Exception information to the failed query spans.
- No context propagation to track worker nodes. No methods in SPI supports this feature.

## Proposed implementation
With additional changes to the SPI, we get the following advantages -
- Presto can be manually instrumented which gives more flexibility and control. Easier to customize what operations can be monitored.
- Traces with parent and child span concept which helps to get the insight on internal operations.
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
- add attributes and events, extract context and start child span , check flush trigger and export span batches are additions to the existing flow.


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
#### Additions

```java
/**
 * SPI can be extended with the specific implementation which contains the actual SDK Span. BaseSpan propagates through the main presto code. Introduced as part of new design.
 */
public interface BaseSpan
        extends AutoCloseable
{
  @Override
  default void close()
  {
    return;
  }

  default void end()
  {
    return;
  }

  /**
   * Returns the span info as string.
   *
   * @return the optional
   */
  default Optional<String> spanString()
  {
    return Optional.empty();
  }
}
```


```java
/**
 * SPI for the span operations which can be implemented by the required serviceability framework.
 *
 * @param <T> the type parameter
 * @param <U> the type parameter
 */
public interface Tracer<T extends BaseSpan, U extends BaseSpan>
{
    /**
     * Gets current context wrap.
     *
     * @param runnable the runnable
     * @return the current context wrap
     */
    Runnable getCurrentContextWrap(Runnable runnable);

    /**
     * Returns true if this Span records tracing events.
     *
     * @return the boolean
     */
    boolean isRecording();

    /**
     * Returns headers map from the input span.
     *
     * @param span the span
     * @return the headers map
     */
    Map<String, String> getHeadersMap(T span);

    /**
     * Ends span by updating the status to error and record input exception.
     *
     * @param querySpan the query span
     * @param throwable the throwable
     */
    void endSpanOnError(T querySpan, Throwable throwable);

    /**
     * Add the input event to the input span.
     *
     * @param span      the span
     * @param event the event name
     */
    void addEvent(T span, Event event);

    /**
     * Sets the attributes map to the input span.
     *
     * @param span       the span
     * @param attributes the attributes
     */
    void setAttributes(T span, Map<String, String> attributes);

    /**
     * Sets the status of the input span to success.
     *
     * @param span the query span
     */
    void setSuccess(T span);

    /**
     * Returns an invalid Span. An invalid Span is used when tracing is disabled.
     *
     * @return the invalid span
     */
    T getInvalidSpan();

    /**
     * Creates and returns the root span.
     *
     * @return the root span
     */
    T getRootSpan(String traceId);

    /**
     * Creates and returns the span with input name.
     *
     * @param spanName the span name
     * @return the span
     */
    T getSpan(String spanName);

    /**
     * Creates and returns the ScopedSpan with input name.
     *
     * @param name     the name
     * @param skipSpan the skip span
     * @return the u
     */
    U scopedSpan(String name, Boolean... skipSpan);
}
```


```java
/**
 * SPI to supply different Tracer implementations.
 *
 * @param <T> the type parameter
 */
public interface TracerProvider<T>
{
    T create();
}
```


#### Deletions

```java
public interface Tracer
{
    String tracerName = "com.facebook.presto";

    void addPoint(String annotation);

    void startBlock(String blockName, String annotation);

    void addPointToBlock(String blockName, String annotation);

    void endBlock(String blockName, String annotation);

    void endTrace(String annotation);
}
```


```java
public interface TraceProvider
{
    String getTracerType();

    Function<Map<String, String>, TracerHandle> getHandleGenerator();

    Tracer getNewTracer(TracerHandle handle);
}
```


```java
/**
 * Removed as we have a default Tracer implementation in presto-main/..tracing/DefaultTelemetryTracer
 *
 */
public interface NoopTracer
{
    public void addPoint(String annotation) {
    }

    @Override
    public void startBlock(String blockName, String annotation) {
    }

    @Override
    public void addPointToBlock(String blockName, String annotation) {
    }

    @Override
    public void endBlock(String blockName, String annotation) {
    }

    @Override
    public void endTrace(String annotation) {
    }

    @Override
    public String getTracerId() {
        return "noop_dummy_id";
    }
}
```


```java

```


```java
/**
 * Removed as not required for the current implementation.
 *
 */
public interface TracerHandle
{
    String getTraceToken();
}
```


### Configuration
#### Existing configuration
In the current Presto Telemetry can be configured by adding the below keys and values(Example) in presto-main/etc/config.properties

```properties
tracing.tracer-type=otel
tracing.enable-distributed-tracing=true
tracing.distributed-tracing-mode=ALWAYS_TRACE
```


#### New configuration
In the latest implementation trace can be configured by modifying the values in presto-main/etc/telemetry-tracing.properties

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
