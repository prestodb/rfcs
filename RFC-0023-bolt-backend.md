# **RFC0023 for Presto**

## Introduce Bolt Backend for Presto Native Execution

Proposers

* Weixin Xu
* 

## Related Issues

N/A

---

## Summary

Introduce Bolt as an additional backend for the Presto native execution engine.

The initial implementation provides a Bolt-based native worker that implements the Presto worker protocol and integrates with the existing Presto coordinator.

To support the Bolt backend build and dependency requirements, a Conan-based dependency flow is introduced for this worker module. Standardizing dependency management across all native backends is out of scope for this RFC.

---

## Background

Bolt is a C++ acceleration library for analytical data processing, designed to serve as a backend for data processing frameworks.
See: https://github.com/bytedance/bolt

It has been validated across multiple frameworks and supports execution across diverse processors and hardware architectures. Bolt provides execution optimizations such as JIT compilation for hotspot expressions, and includes support for common analytical data formats and table abstractions.

Bolt can be integrated as a Presto worker by implementing the existing worker protocol, without requiring changes to the coordinator or query protocol.

---

## Goals

* Support a Bolt-based backend for the Presto native execution engine.
* Keep backend selection operationally simple for developers and operators.
* Reuse existing native execution components where practical.

### Non-Goals

* Runtime switching between Bolt and Velox backends (e.g., per query or within the same worker).
* Large-scale refactoring of `presto-native-execution` into shared modules as a prerequisite for Bolt integration.

---

## Proposed Implementation

The Bolt backend is introduced as a separate native worker module (`presto-bolt-execution`) that implements the Presto worker protocol and integrates with the coordinator.

At the protocol level, the Bolt worker can be deployed as an alternative to the existing native worker.

The implementation reuses engine-agnostic components where applicable and keeps execution-specific logic within the Bolt backend.

### Component Overview

| Component                 | Approach               |
|---------------------------|------------------------|
| http                      | reused                 |
| runtime-metrics           | reused                 |
| thrift / protocol         | reused (preferred)     |
| types / plan conversion   | backend-local          |
| operators                 | backend-local          |
| presto_server_lib         | backend-local          |
| common                    | backend-local          |
| tests                     | backend-local          |

---

## Protocol and Thrift Strategy

The implementation reuses existing Presto protocol and thrift definitions wherever possible.

Separate protocol or thrift generation is used only if required by build or toolchain constraints.

---

## Adoption Plan

The Bolt backend is introduced as an optional native worker implementation.

It does not affect existing deployments. Users can deploy the Bolt worker alongside or in place of the existing native worker.

---

## Test Plan

Testing covers unit validation, worker-level integration, end-to-end query correctness, and CI integration.

### 1. Unit Testing

Bolt provides its own unit tests for core execution components.

Additional tests are added for integration-specific logic, including:
* plan conversion
* expression translation
* split and connector adaptation
* backend-specific configuration handling

---

### 2. Worker Integration Testing

Integration tests exercise key lifecycle and execution paths, including:
* worker startup and registration
* discovery and heartbeat
* task submission and execution
* result retrieval
* basic failure and cancellation scenarios

---

### 3. End-to-End Query Testing

End-to-end validation is performed using the external worker execution model.

Primary coverage includes:
* TPCH smoke tests
* Hive connector tests

---

### 4. CI Integration

CI covers:
* build
* unit tests
* basic integration validation