# **RFC1 for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## SPI for native workers

Proposers

* Tim Meehan
* Aditi Pandit
* Deepak Majeti
* Yi-Hong Wang

## [Related Issues]

N/A

## Summary

Prestissimo is currently hard-coded to work with Hive and TPC-H connectors, and has a hard-coded list of functions, types and operators that are incongruent with the default built-ins found in the Java coordinator.  We propose a new SPI that will allow plugin authors to customize Prestissimo in ways that are currently available in Java clusters.  Additionally, we believe an enhanced SPI can allow better integration with the underlying native evaluation engine, and also potentially open up the possibility for large amounts of code deletion and cleanup in the Presto codebase.

## Background

Brief description of any existing issues, user requests or feature comparison with competitors which explains why the proposed feature might be needed. How the proposed feature helps our users? Explain the impact and value of this feature.

### Goals

* The coordinator must fail fast if it encounters a query which is not compatible with its underlying evaluation engine.
* All `SHOW` commands must return consistent information to the user. In particular:
  *  `SHOW SESSION` should only show session properties which are compatible with its underlying evaluation engine.
  *  `SHOW CATALOGS` should only show catalogs which are compatible with its underlying evaluation engine.
  * `SHOW FUNCTIONS` should only show functions which are compatible with its underlying evaluation engine. 
* DDL statement compatibility must be consistent with evaluation engine compatibility. 
* Presto must fail fast if one attempts to register a connector which is incompatible with its underlying evaluation engine. 
* Presto must fail to start if one attempts to use a configuration property which is not compatible with its underlying evaluation engine. 
* Presto must fail fast if one attempts to use a session property which is not compatible with its underlying evaluation engine. Presto must fail fast if one attempts to use a connector session property which does not have a corresponding session property in the worker. 
* The connector authoring experience must be updated to reflect the new evaluation engine. In particular:
  * Connector authors should be able to register connectors without deploying custom builds of Presto. 
  * Connectors should always have a way of communicating to the coordinator what types, functions, operators, and session properties are supported. 
  * Connector authors must not be expected to use classes outside of an established SPI. 
  * The Connector SPI must be updated to remove interfaces that are not used by the underlying evaluation engine.


### Non-goals
* It is not a goal to prevent all instances of mismatch/split-brain between Java coordinators and C++ workers.  This will significantly close the gap, however there is a long tail of subtle differences that will need to be worked out over time.

## Proposed Implementation

[For more details on the implementation, see this design document.](https://docs.google.com/document/d/1d8pDKtayun0QKGNjYz9Slaor3-aVYr99k7ZBmrKSOOA/edit?usp=sharing)

1. What modules are involved
   2. There will be a new plugin interface to support the additional functionality of registering native functions, supporting their evaluation, registering native types and session properties, and registering and communicating with connectors.  This will affect the Presto SPI, the Presto engine, and the native worker module, with additional work in Velox to support dynamic loading of code to register functions and connectors.
2. Any new terminologies/concepts/SQL language additions
   3. There will be a new SPI footprint, which will eventually be used to deprecate the older Java-specific SPI.
3. Method/class/interface contracts which you deem fit for implementation.
   4. Please see design document for details.
4. Code flow using bullet points or pseudo code as applicable
5. Any new user facing metrics that can be shown on CLI or UI.
   6. There will be no new user facing metrics as a part of this work. 

## Metrics

N/A

## Other Approaches Considered

* Hybrid clusters -- rather than exposing the differences between C++ and Java to the user, one could run hybrid clusters with both Java and C++ workers.  It is anticipated that the resource management and performance challenges this would create would be non-trivial and would not result in a productive outcome for Prestissimo adoption.  It also doesn't solve the egonomic challenges of C++ connector and function authoring.
* Coprocessors -- one could run colocated Java workers on the same nodes as C++ workers, and use coprocessors to communicate between the two.  This would create significant resource management challenges as Presto clusters presume singular pools of memory--such memory would need to dynamically shift between native and Java processes, a challenge made difficult by the Java worker's use of Java heap memory as opposed to off-heap memory.  It also doesn't solve the egonomic challenges of C++ connector and function authoring.
* Dual clusters -- one can attempt to run two separate clusters, one that is purely Java, and one that is purely C++.  This would waste resources in due to the always-on nature of Presto clusters and the anticipated unequal load between the clusters.  Additionally, it would result in a poor user experience when queries which differ only by, e.g. a single unsupported function, have drastically different performance owing to their running on the Java cluster.  It also doesn't solve the egonomic challenges of C++ connector and function authoring.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
  - There will be a migration path to the new SPI.  The Presto project will be required to maintain both for a period of time.
- If we are changing behaviour how will we phase out the older behaviour?
  - We believe that C++ is a foundational change in Presto's architecture, and as such there will be small behavior differences with Java.  We will document such minor differences and encourage users to migrate to the new architecture at will.
- If we need special migration tools, describe them here.
  - There will be a new framework to translate SPI v1 connectors to SPI v2 connectors.  Such a migration will be documented and publicly available for connector authors.  Ideally, the work required to complete this migration will be minimal and configuration based. 
- When will we remove the existing behaviour, if applicable.
  - No existing behavior will be removed.
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
  - There are several ways we will spread awareness of this architecture:
    - We will add documentation for all new SPI interfaces and additions.
    - We will add blog posts and articles describing the change and its benefits.
    - We will announce progress through various forums, such as the TSC meeting and working group meetings.
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
  - While this RFC will allow us to deprecate the Java worker, a separate RFC should be made that describes the deprecation process.

## Test Plan

Integration tests will be made for various ad-hoc connectors in Prestissimo, new functions in Prestissimo, and new types and session properties.  We will also add benchto tests as needed, and product tests to prove end to end reliability.