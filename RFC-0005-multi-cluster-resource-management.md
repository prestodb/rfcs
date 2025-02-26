# **RFC5 for Presto**

## [Multi Cluster Resource Management]

Proposers

* Prithviraj Pandian, Meta 
* Abhisek Saikia, Meta

## Summary

We intend to implement functionality and interfaces in Presto to integrate with an optional external queueing service. 

## Related Issues 

The idea of moving the queueing phase out of the coordinator into a separate process is an old one dating back many years. Here are a few previous proposals/discussions: 
* https://github.com/prestodb/presto/pull/12176
* https://github.com/trinodb/trino/issues/391
* https://github.com/prestodb/presto/issues/10174
* https://github.com/prestodb/presto/issues/13198

## Background

Currently, each Presto cluster owns its own resource management and queue. The resource manager of each cluster decides which query in the cluster’s queue should be executed next, based on logic described in the cluster’s [resource group configuration](https://prestodb.io/docs/current/admin/resource-groups.html). 

This status quo of cluster-level queueing works fine to manage one or a few clusters, but quickly becomes problematic when managing a deployment of tens or hundreds of Presto clusters. Some issues include: 

1. Due to the queue being fragmented across multiple clusters, and the admission decision being made at the cluster level, resource group admission decisions are not correct globally, although they could be correct locally. For example, it is possible that one cluster might admit a query of lower priority than a query in another cluster’s queue. 
2. Since query concurrency control is applied on a per-cluster basis, global concurrency control can only be achieved, if the concurrency limit is a multiple of the number of clusters. For example, it is currently impossible to enforce a resource group with a concurrency limit of 1 query, across multiple clusters. 
3. Load-balancing queries effectively across multiple clusters requires the load balancer to understand the resource group and queue state of each cluster. This is difficult to achieve, and in practice, stale information leads to poor load balancing. 
4. Since the query is queued on the cluster itself, restarting clusters for maintenance reasons requires either that the queued queries be killed, or a lengthy drain time. 

## Proposed Implementation

We propose to add support in Presto for an optional queueing service that sits upstream of multiple Presto clusters. This queueing service will serve as the global entry-point for all queries from the client and will implement the QueuedStatement protocol. This queueing service is responsible for: 

1. Authenticating the request 
2. Adding the query to the global queue 
3. Responding to clients polling for the queueing status of their query
4. Picking the next query to execute 
5. Selecting the best cluster for execution of this query
6. Dispatching this query to the cluster
7. Redirecting the client to the picked cluster

The internal details of this queueing service are still under development, and this document focuses on the changes required to Presto for such a service to work. If there is interest from the community, we can provide an open-source reference implementation of this queueing service, once mature. 

### Diagram
![Presto Github Issue GRM](https://github.com/user-attachments/assets/7b2af8fd-5d82-411d-ac97-795e54e35164)

### Work Items
Because we want this to be transparent to Presto clients, the front-end of the queueing service will reuse the existing functionality of classes in the presto-dispatcher package such as QueuedStatementResource, DispatchManager, DispatchQuery, etc. The high-level work items are:

1. Allow for the queueing service to forward all required query metadata to the cluster, such as query ID, authenticated identity, query state timings, etc. This will involve changes to QueuedStatementResource, QueryStateMachine, and DispatchManager, so that coordinator can create a query with an already populated QueryId and AuthorizedIdentity.
2. De-couple the dispatching and execution phases of the query lifecycle. This will involve separating and removing dependencies that are required for execution of the query, but not for queueing, such as Metadata class. We will create a new module, presto-dispatcher, that has all classes required for the queueing phase, such as QueuedStatementResource, DispatchManager, ResourceGroupManager, etc. The presto-main module will have a dependency on this new module. Interfaces such as ResourceGroupManager and DispatchQuery will be in presto-dispatcher, with implementations such as InternalResourceGroupManager and LocalDispatchQuery in presto-main. In addition, it could be necessary to move some classes into a common module, if they are required by both presto-main and presto-dispatcher.

## Adoption Plan 

1. There will be no client-server protocol changes. The queueing service should be able to be added without any changes to clients. 
2. There should be no impact to users of Presto who do not implement or use this queueing service.
3. There should be minimal impact to the latency of end-to-end query execution when using the service.

## Test Plan 

We will follow all software engineering best practices for testing with unit tests, etc. 
