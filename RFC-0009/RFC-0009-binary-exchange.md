# Add support for Binary Exchange Protocol

Proposers
* Daniel Bauer (dnb@zurich.ibm.com)
* Zoltan Nagy (nag@zurich.ibm.com)
* Andrea Giovannini (agv@zurich.ibm.com)

## Related Issues

[Redesign Exchange protocol to reduce query latency](https://github.com/prestodb/presto/issues/21926)

Above protocol enhancement is integrated into the proposed binary exchange protocol.

## Summary

The binary exchange protocol (BinX) is an alternative for the existing HTTP-based exchange protocol that
runs between Prestissimo worker nodes. It offers the same functionality and API
but uses binary encoding that can be more efficiently parsed than HTTP messages.
This translates into a performance benefit for exchange-intensive queries.
BinX does not replace the control protocol that runs between the coordinator and the
worker nodes. The control protocol continues to use HTTP.

## Background

The exchange protocol provides remote-procedure-call semantics for obtaining
data from a remote worker, acknowledging data receipt and terminating (aborting) the exchange.
The implementation on top of HTTP uses a small subset of the features that HTTP offers.
Transaction- and session multiplexing are not needed. The parsing of the generic HTTP messages
is more complex than decoding binary encoded messages.

### Goals

The proposal is to use a binary exchange protocol as a light-weight alternative to the existing HTTP exchange protocol.
As a prototypical implementation shows that such a protocol reduces query run-time of exchange heavy queries by
20% to 30%.

A further goal is to open the way to enable network accelerators, i.e. support for smart network interface
cards that offload the transport stack onto the NIC.

## Description of the Prototype Implementation

The aim is to minimize changes to the existing code base by adding light-weight
components to the Prestissimo workers without changing the coordinator. BinX is an optional feature
disabled by default that can be activated in the Prestissimo worker configuration.
Once activated, the data exchange between all worker nodes is done using BinX. Communication
between worker and coordinator continue to use the existing HTTP exchange protocol.
Mixed-mode operation where only some worker nodes use BinX is not supported.

BinX communicates using its own, separate port number. When enabled, the `http://` and `https://`
scheme provided by the coordinator are re-mapped and a BinX exchange source is instantiated instead
of the default HttpExchangeSource (formerly known as PrestoExchangeSource).
The communication to the coordinator node and other exchange types such as broadcast exchange and unsafe row exchange
are not affected.

The semantics of the exchange protocol remains unchanged with the following properties:
- Exchanges are initiated by the data consumer for a remote buffer. Once the buffer has
been transferred or the owning remote task has been stopped, the exchange is terminated.
- The exchange is implemented as a request/reply protocol, i.e. it is pull based.
- A data request is answered with a data reply. The reply contains zero, one or more
  "chunks", depending on the availability of data on the serving side.
- A timeout mechanism with exponential back-off provides robustness in the face of failures.
- Acknowledge requests are used to free memory on the serving side, acknowledge replies
exist but are ignored on the requesting side.
- An explicit delete (or abort) request informs the serving side that the transfer is
stopping and that associated resources can be freed. Delete requests are acknowledged.

Requests and replies are implemented as binary encoded protocol data units on top of TCP.
This minimizes the protocol overhead.

### Configuration Options

BinX has two configuration properties that are part of the Prestissimo (native) worker
configuration:

|Property name | Type| Values| Effect |
|--------|------|------|------|
|binx.enabled|bool| True or false, defaults to false | Enables BinX when set to true|
|binx.server.port|uint64_t|Valid port number > 1000, defaults to 8091| The port number used by BinX|

The configuration must be homogeneous across all worker nodes. BinX is disabled by default
and must be explicitly enabled.

If enabled, the worker's logs will contain the following messages:
```
I20240828 14:57:11.361124    19 PrestoServer.cpp:599] [PRESTO_STARTUP] Binary Server IO executor 'BinSrvIO' has 64 threads.
I20240828 14:57:11.364262    19 BinaryExchangeServer.cpp:203] Starting binary exchange server on port 8091
```

## Implementation Overview

The implementation covers the protocol design, the BinX server implementation and the BinX exchange
source implementation.

### Binary Exchange Protocol

The protocol consists of requests and replies that are implemented by the `BinRequest` and the `BinReply` classes defined
in `BinRequestReply.h`. All three request types for data, acknowledge and delete share the same structure.
A request consists of the following fields:
|Field name| Type | Semantic |
| ---- | ---- | ---- |
|requestType| enum (DATA, ACK, DELETE) | The type of request |
|getDataSize| bool | When true, requests the sizes of the remaining available data pages; when false requests data |
|maxSizeOctets| uint64_t | The maximum allowed size of the data pages|
|maxWaitMicroSeconds|uint64_t| The maximum wait time in microseconds|
|taskId| std::string | The unique ID of the remote task from which data is requested|
|bufferId| uint64_t | The ID of the buffer within the remote task|
|pageToken| uint64_t | The sequence number of the data page that is requested|

Acknowledge and delete requests don't need all fields and set `getDataSize`, `maxSizeOctets`,
and `maxWaitMicroSeconds` to false and 0, respectively.

A reply has the following fields:

|Field name| Type | Semantic |
| ---- | ---- | ---- |
|replyType| enum (DATA, ACK, DELETE) | The type of reply |
|status| enum (OK, NODATA, SERVER_ERROR, TIMEOUT) | The reply status |
|bufferComplete| bool | True if the entire buffer has been transferred |
|pageToken|uint64_t| The token (sequence number) of the first data page in the reply|
|pageNextToken|uint64_t| The token (sequence number) of the next available page that can be requested|
|remainingBytes| std::vector<uint64_t> | Contains the sizes of available pages when requested|
|data| IOBuf | Contains the data payload |

`BinRequest` and `BinReply` are in-memory representations. The serialization and deserialization on top
of TCP is implemented by the `ServerSerializeHandler` and `ClientSerializeHandler`. On the server side,
requests are read and replies are written and vice-versa on the client side. During serialization, a
length field is prepended that is needed to correctly de-serialize the protocol data units.

The implementation uses Meta's [Wangle](https://github.com/facebook/wangle) framework. 

### Binary Exchange Server

The binary exchange server is a self-contained component that is started when `binx.enabled` is configured.
It implements the exchange service using the BinX protocol. Incoming requests are forwarded to the
`TaskManager` and the result is marshalled into a reply and sent back.

The binary exchange server is started in `PrestoServer::run()`. It uses a dedicated
IO thread pool in order to not interfere with the HTTP IO thread pool. The CPU thread pool is shared
with the HTTP exchange.

#### Implementation Notes

Like Proxygen, BinX uses Wangle as its underlying networking library. Whereas Proxygen
implements a generic HTTP client and server, BinX provides a specific binary protocol
for Prestissimo's exchange.
The BinX server is implemented in the file `BinaryExchangeServer.h` and consists of
several components:

* The `BinaryExchangeServer` is a controller for starting and stopping the Wangle protocol stack.
It takes the port number, the IO thread pool and the CPU thread pool as construction parameters.
The `start()` method creates a Wangle factory for the BinX protocol stack and binds this factory
to a listening socket. Connection- and protocol-handing is then done by the Wangle framework.
The `stop()` method destroys the protocol stack and joins in the threads again.
* The `ExchangeServerFactory` defines the BinX protocol stack. The stack consists of an asynchronous TCP socket handler, a threading component called `EventBaseHandler` that makes sure that all operations are carried out in the same IO thread, the server-side serialization and deserialization handler, and the service implementation on top of the stack.
* The `BinaryExchangeService` processes incoming requests and calls the appropriate TaskManager methods. The results from the TaskManager are packaged into replies and sent back to the requesting BinX exchange source. This exchange service follows the design of the existing `TaskResource` service.

All of above components are templated to allow for different TaskManager implementations. In the production code, the Prestissimo TaskManager is used while for unit testing, a mock task manager is deployed.

### Binary Exchange Source and Binary Exchange Client

The client side of BinX is implemented in two components. The `BinaryExchangeSource` implements the "higher level" parts
that gets called by the Velox exchange client and interfaces with Velox's page memory management. It also implements
the exchange protocol logic.
The `BinaryExchangeSource` uses the `BinaryExchangeClient` that is responsible for the protocol mechanics and
implements connection setup, sending and receiving of requests and replies, and timeouts when replies don't arrive.

The `PrestoServer` registers a factory method for creating exchange sources. This factory method has been extended
such that `BinaryExchangeSource`s are created instead of HTTP exchanges when enabled by configuration.
One exception are connections to the
Presto coordinator that always uses the HTTP based exchange protocol. In a Kubernetes environment with its virtual
networking, it is unfortunately not straight forward to detect whether the target host is the Presto coordinator
since the connector's service IP used in the Presto configuration doesn't correspond to the IP address used by the
pod running the coordinator. In order to circumvent this problem, a helper class called `CoordinatorInfoResolver`
uses the node status endpoint of the coordinator to retrieve the coordinator's IP address. Using this address
allows the factory method to create `HttpExchangeSource`s when connecting to the coordinator and `BinaryExchangeSources`
when connecting to worker nodes.


#### Binary Exchange Client

The `BinaryExchangeClient` is a Wangle client. The `ExchangeClientFactory` is an inner class of the client that
defines the protocol stack. On top of an asynchronous socket, an `EventBaseHandler` controls threading, followed by the
`ClientSerializeHandler` that is responsible for serializing requests and de-serializing replies. The top of the client
protocol stack is formed by the `ExchangeClientDispatcher`. Its main task is to keep track of outstanding data-,
acknowledge- and delete-requests. The client dispatcher maintains hash maps for mapping the requests' sequence numbers
to the outstanding promises. These promises are fulfilled whenever a corresponding reply arrives or when the
associated timeout fires.

The `BinaryExchangeClient` provides a single `request()` method for sending a request. It returns a future that becomes
valid once the reply arrives or an error or timeout occurs. The connection to the remote node is set up lazily on the first request.
Since connection setup is asynchronous, incoming requests are queued until the connection setup has completed.
While normally only a single data request can be outstanding at any point in time, the queue is nevertheless necessary in the case
that the exchange client immediately closed after the first request and a delete request is issued before the connection setup could complete.

#### Binary Exchange Source

The `BinaryExchangeSource` class provides the same interface as the existing `HttpExchangeSource`. It transfers the contents of a remote
buffer into a series of Velox pages and appends these pages to the provided `ExchangeQueue`. Once the buffer is transferred, the
exchange source is discarded. Its implementation is similar to that of the `HttpExchangeSource` with the following differences:
- Instead of an `HttpClient`, a `BinaryExchangeClient` is instantiated.
- The `BinaryExchangeSource` doesn't use a session- or connection pool.
- `BinRequest` and `BinReply` are used rather than HTTP requests and HTTP replies.
- All buffer transfers are immediate, i.e. there is no support for `exchange.immediate-buffer-transfer=false`.


## Metrics / Benchmarks

The following heatmap shows the results of a micro-benchmark conducted on two servers, each providing 128 hardware cores.
One server is running the exchange client and the other the exchange server.
The machines are connected back-to-back using 200 Gbit/s network interfaces.
The exchanged buffers always consisted of 128 chunks (slices) with chunk sizes ranging from 32 bytes to 16 MBytes.
Only one chunk at a time was sent per data request.
The client spawned between 1 and 128 parallel exchanges. Both client and server used thread pools of 129 threads.

![Microbenchmark speedup](binx_speedup.png)

The binary exchange protocol shows a performance advantage over the HTTP protocol of a factor of around 2.
For parallel exchanges of more than 48 exchanges, the binary exchange protocol was close to saturating the
available network capacity.

## Other Approaches Considered

Both Thrift and gRPC have been considered but abandoned as the overhead of a generic RPC mechanism wasn't worth
the additional complexity.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?

  - There is one additional configuration option to enable BinX. Otherwise, there is no impact on session parameters, no API changes
  and no changes to SQL.

- If we are changing behavior how will we phase out the older behavior?

  - The HTTP stack is still required for the control message. The cost of keeping the HttpExchangeSource is minimal.

- If we need special migration tools, describe them here.

  - No tools required.

- When will we remove the existing behavior, if applicable.

  - Existing behavior will remain as the default option.

- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?

  - The documentation should mention that an alternative exchange protocol is available.

- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

  - Support for SmartNICs should be considered.

## Test Plan

Test plan involves running performance measurements using TPC-DS and TPC-H benchmarks that compare the performance of HTTP versus BinX.

The TPC-DS benchmark test has been conducted using a dataset with scale factor 1000 on an on-premise cluster with 8 nodes. The results
for this 1TB dataset have shown that overall runtime for the 99 queries was ~56 minutes when using HTTP compared to ~43 minutes for BinX.
