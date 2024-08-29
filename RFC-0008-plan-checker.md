# **RFC0 for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## [Title]

Proposers

* Tim Meehan
* Bryan Cutler
* Rebecca Schlussel

## [Related Issues]

* RFC-0003

## Summary

Add a new SPI to integrate a custom plan checker, and add a plugin to use the Presto sidecar to check if a Presto plan can be
successfully translated into a Velox plan.

## Background

The optimizer makes decisions in part based on the capabilities of the underlying evaluation engine.  With the Presto evaluation
engine being migrated to Velox while concurrently supporting the Presto Java evaluation engine, there's now differences between
what is supported between both evaluation engines.  Because of these underlying differences, the optimizer may generate plans that
can't be executed in C++ clusters, or likewise, a Presto Java cluster could be misconfigured to generate plans that only work with
C++ clusters.

If a plan is generated that can't be executed in the cluster, the query will fail with an error message during the query execution
phase.  This is a poor user experience, as the user has to wait for the query to fail before they can take corrective action, and
this might have occurred after lengthy queueing.  Additionally, this failure would occur after worker resources have already been
allocated to the query, which is wasteful.  This RFC proposes a plan checker that can be run before the query is executed to ensure
a quick validation of the plan.

### [Optional] Goals

* Provide a mechanism to validate Presto to Velox plan conversion during planning phase
* Add an SPI to allow custom validators to be added to suit individual business needs
* Validate fragmented plans prior to scheduling

### [Optional] Non-goals

* Ensure all Velox plans are executable in a Presto C++ cluster--many checks are done at runtime and may not be caught by the 
  plan checker

## Proposed Implementation

### Core SPI

A new SPI `PlanConverter` will be added to the Presto codebase that takes in a Presto `PlanFragment` and returns a data 
structure with the following fields:

* An optional error message.  Presence indicates that the plan is invalid, absence indicates that the plan is valid.
* An optional string representing the serialized converted plan fragment.  Presence indicates that the plan was successfully
  converted to a Velox plan fragment, absence indicates that the plan was not converted.

### Presto to Velox plan validation

The Presto runtime centralizes plan validation logic into the `PlanChecker` class.  There exist three phases to this class: 

* `validateIntermediatePlan`
* `validateFinalPlan`
* `validateFragmentedPlan`

It is generally useful to allow this class to be configured with an SPI that allows for custom plan validation logic to be added.
For example, a business may decide that a certain type is not allowed, or add a check to ensure that plans that are overly
complicated are killed.

An SPI will be added that will add more checks to the `PlanChecker` class that will allow additional checks for each of the planning
phases.  The SPI will contain a field indicating which phase of the plan checker to be added, and a validator that will be run during
that phase.

A new endpoint to the Presto sidecar will be added that will attempt to convert a Presto plan to a Velox plan.  In the
`presto-native-plugin` module, a new implementation of the SPI will be added which will add a check to the `PlanChecker` class
which will call the sidecar to attempt to convert the plan.  If the conversion fails, the plan checker will fail the plan.  It will
be added at the `validateFragmentedPlan` phase--this is because it's not until plan fragmentation occurs that we know which portions
of the plan will be executed in the coordinator, and which will be executed in the workers.  Plan fragments that are executed in the
coordinator, such as `COORDINATORY_ONLY` distribution types, will be skipped by the plugin.

The code for the plan checker will run `PrestoToVeloxQueryPlan`, which is used by the workers to convert the Presto plan fragment
to a Velox plan fragment.  If the conversion fails, the plan checker will fail the plan, returning with it the reason for the failure.

#### Failing the plan quickly

An additional code change will be made to allow the planner to execute prior to queuing.  This is so that the plan checker can
be run before the query is queued.  This will allow the user to get feedback on the plan before the query is executed, and will
allow the query to fail quickly if the plan is invalid.

Because the queue limits concurrency, and too much concurrency during planning may require excessive resources in the coordinator,
the plan checker will be run in a separate thread pool.  This thread pool will be configured with a maximum number of threads
that can be run concurrently.  If the thread pool is full, then planning will wait until there is a free thread to run the planner.
This will be configured with a new configuration parameter and session property.

### EXPLAIN (TYPE NATIVE)

The `EXPLAIN (TYPE NATIVE)` command will be updated to run the `PlanConverter` over all fragments which are not `COORDINATOR_ONLY`.  
This will allow the user to see the plan that will be executed in the workers, and will allow the user to see if the plan can be 
converted to Velox.

Explain plans take in a format parameter.  The format parameters that exist today (`TEXT`, `GRAPHVIZ`, and `JSON`) will be added
to the SPI, and the `PlanConverter` will be run with the appropriate format parameter.  When the call to the sidecar is made, the
format parameter will match to an appropriate content type and added to the `accept` header in the request.  For example, if the
format is `JSON`, then the `accept` header will be set to `application/json`, and the server will be expected to return a JSON
object.  The response's content type header will be validated to be `application/json`, and if it is not, the call will fail.

### Sidecar endpoint

> Endpoint: /v1/velox/plan
>
> HTTP verb: POST
>
> Request body: serialized plan fragment
>
> Response body: serialized Velox plan fragment or error message if conversion failed (along with an HTTP 400 status code)

The request and response formats will be dictated by the `content-type` header.  Initially, the only supported content type for 
the request will be `application/json`.  The response will initially be `text/plain`, but in the future can support other formats
such as `application/json` and `application/graphviz`.  The client can specify the response format by setting the `accept` header.
E.g. `accept: application/json` if the client wants the response in JSON format.

#### Additional information

1. What modules are involved
   2. `presto-native-sidecar` (note: this is a new module that will be added to the Presto codebase)
   3. `presto-main`
   4. `presto-spi`
2. Any new terminologies/concepts/SQL language additions
   3. NA
3. Method/class/interface contracts which you deem fit for implementation.
   4. A new PlanChecker class will be added which can be implemented in the Java SPI.  A default implementation will be added that
      will validate using the Presto sidecar.
4. Code flow using bullet points or pseudo code as applicable
   5. A query is fragmented.  The fragmented query is sent to the `PlanChecker`, which runs a series of checks
      against the fragmented plan.
   6. The `PlanChecker` runs the new plan fragment checks which have been registered to be included after all 
      preexisting checks have been run.
   7. If the `presto-native-sidecar` module has been registered, then the `PlanChecker` will call the checker code
        in the `presto-native-sidecar` module.
   8. The `presto-native-sidecar` module will marshall the plan fragment into JSON and send to the Presto sidecar.
   9. The Presto sidecar will attempt to convert the plan fragment to a Velox plan fragment.  If it succeeds, a 200
      response is sent.  If it fails, a 400 response is sent with the reason for the failure as a JSON object.
5. Any new user facing metrics that can be shown on CLI or UI.
   1. NA

## [Optional] Metrics

This is a 0 to 1 feature and will not have any metrics.

## [Optional] Other Approaches Considered

https://github.com/prestodb/presto/pull/23423 added a hook for a similar plan validation.  However, this hook
is added at the plan conversion level at the worker.  This RFC proposes a plan validation at the coordinator level
to provide a quicker feedback loop to the user, and to allow this logic to be composed in other components such as
a load balancer or external queueing service.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
  - No impact to users.  Because the plan checker is implemented as a plugin, the plugin must explicitly be added to a deployment
    in order to be used.
- If we are changing behaviour how will we phase out the older behaviour?
   - NA
- If we need special migration tools, describe them here.
  - A migration to use the Presto Sidecar will be needed, which entails additional infrastructure; specifically,
    deployments will need to deploy the sidecar with the coordinator.
- When will we remove the existing behaviour, if applicable.
  - NA
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
  - This feature will be documented in the Presto documentation.
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
  - It is not in scope to catch all runtime errors in the plan checker.  This is a best effort to catch as many errors as possible
    before the query is executed.

## Test Plan

Infrastructure tests will be added that proves the end to end capability of the plan checker.  This will include a test that
validates that a plan that can be converted to Velox will pass, and a plan that can't be converted to Velox will fail.  Additionally,
unit tests will be added to the `PlanChecker` class to ensure that the SPIs are run in the correct order, and that the Presto sidecar
is called when the SPI is registered.