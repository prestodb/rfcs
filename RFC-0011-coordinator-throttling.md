## Coordinator throttling mechanism

Proposers

* Prashant Golash (@PRASHANT GOLASH)


## Summary

Implement a mechanism to prevent queries from being admitted when a cluster is experiencing overload. 
A cluster is considered overloaded based on aggregated metrics reported by its underlying worker nodes. 

These metrics can include CPU utilization, memory usage, and task queue lengths. By analyzing this aggregated information, the system can make informed decisions about the cluster’s current load status.
The exact criteria used to determine overload are governed by customizable and extensible policies. For example, a policy might specify that a cluster is overloaded if a certain percentage or number of worker nodes reach critical resource thresholds (“run hot”).
In our case, most of the workers get overloaded within a matter of few mins

A sample graph of clusters running hot. All the workers are running hot in this case.

![Design diagram](RFC-0011/cpuload.png)

The impact is:

- Large variance in execution time
- Memory kills / Overload on dependency services like metastore / warm-storage etc 

This provide more reasons to work on protecting cluster overload.

## Background

While not exhaustive, current throttling/possible mechanisms relevant for this discussion

#### Granular task based scheduling

**Leaf Task scheduling**

The node selection logic in Presto involves randomly selecting 10 nodes from those that do not currently have scheduled splits for the given query. Among these 10 nodes, the system then chooses the node that is the least busy. For reference, the relevant code implementation can be found [here](https://github.com/prestodb/presto/blob/master/presto-main-base/src/main/java/com/facebook/presto/execution/scheduler/nodeSelection/SimpleNodeSelector.java#L182).
Additionally, in the process of leaf task scheduling, a feedback mechanism is employed whereby splits are assigned to the nodes deemed least busy. It is important to note that in this context, "busyness" is defined by the number of splits being processed by a node, rather than its actual system load. 

While this approach is generally effective for distributing work, it may not always result in the most optimal allocation of resources.

**Intermediate task scheduling**

For intermediate task scheduling in Presto, the selection of workers is based on a hash partition count. Specifically, intermediate tasks are assigned to worker nodes according to the hash of the partition, which determines the distribution across available intermediate workers. The relevant code implementation can be found [here](https://github.com/prestodb/presto/blob/master/presto-main-base/src/main/java/com/facebook/presto/sql/planner/SystemPartitioningHandle.java#L164).
This strategy does not consider the current system load when assigning intermediate tasks. 

#### Resource Group throttling
There are certain metrics like softMemoryLimit etc which exist at RG level to queue the workload, but it does not consider the actual utilization and is out of scope for this discussion.


While we could improve task scheduling (pro-active measure to not let worker overload) through load balancing (more on this in alternative approaches), this alone would be insufficient given the nature of our overload scenario where all workers are saturated in short period of time which is due to admitting lot of heavy queries in short duration of time. 

For immediate relief, we propose implementing a holistic admission control mechanism that provides reactive throttling of the overall workload by not admitting queries.

### Goals
- Decrease duration of overload on the cluster
- Also decrease overall variance of execution time of queries (queuing is fine)

## Proposed Implementation
### Worker side changes
- Added endpoint to gather worker load (`/v1/info/nodestats`)
  - Include NodeState + worker load. Idea is to just call `/v1/info/nodestats` rather than `/v1/info/state` (eventually deprecate it). Worker load is already getting populated in [cpp workers](https://github.com/prestodb/presto/blob/master/presto-native-execution/presto_cpp/main/PrestoServer.cpp#L1537).  
- PR: https://github.com/prestodb/presto/pull/25686

### Coordinator side changes
#### Collecting worker load data
  - Update DiscoveryManager to invoke this end point rather than the `/v1/info/state`
  - Expose `getNodeLoadMetrics` on InternalNodeManager
  - PR: https://github.com/prestodb/presto/pull/25688 
#### Scheduling policies
  - Implement node overload policies that can govern the policies to determine cluster overload
  - Based on these policies and load collected from worker, improve the admission control logic globally to not queue the query (irrespective of Resource Group)
  - PR: https://github.com/prestodb/presto/pull/25689 


## Metrics

- Cluster overload
- Variance in runtime of queries


## Other Approaches Considered

### Approach 1
#### Improve Granular task scheduling
Not fully baked, but at high-level based on load, the coordinator can assign scores to the workers and selects the top N highest-scoring workers, then randomly chooses from this subset to schedule tasks or splits.

**Few Examples**:
- **Intermediate stage**: For 333 hash partitions, randomly select 333 workers from the top 500 scoring workers. (We run with 300 node clusters so this property needs to be tuned as well if we go with this approach.)
- **Leaf stage**: For leaf task, the logic in `chooseLeastBusyNode` could be augmented to pick workers with best score and out of these assign the splits.


- **Pros**
This could provide load balancing to begin with and prevent some workers getting hot

- **Cons**
In our cluster most of the workers become overloaded in a fraction of minutes due to a lot of queries getting admitted. 

Load balancing may not be immediately helpful in such cases
This could be a good direction for the future if we see only a select few workers are causing issues.

### Approach 2
Write a separate service which can consume worker load and update global RG (max queries that can run based on the load / policies)

- **Pros**
No coordinator changes

- **Cons**
Extra service to maintain
Two brains


## Adoption Plan
If this feature is enabled, this can cause queuing when cluster is loaded (which would be separate from queuing due to Resource Groups)
. For now end users can correlate queuing via looking at queuing + worker overload metrics together. If required, we can also expose separate queuing metrics due to cluster overload.
## Test Plan
Added the test cases
Also added via simulating the workload
