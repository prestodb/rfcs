

## [Register Native Sidecar Functions in default namespace]

Proposers

* Kevin Tang
* Feilong Liu

## [Related Issues]


RFC for new SPI for SQL invoked functions
https://github.com/prestodb/rfcs/pull/40/files

## Summary

Until we can do all the constant folding on the sidecar we need to be able to do constant folding in Java, even when getting the function list from the sidecar.  Some functions will be cpp only functions, and those we want to skip constant folding for, since those will fail.

The goal is to be able to enable the sidecar while preserving the constant folding functionality for functions that have java implementation and SQL implementation. Once this is finished, sidecar can be enabled internally and inline_sql_functions can be fully enabled in C++ clusters. There is currently no support for constant folding in C++ functions, but this will be addressed in a followup design.


## Background

### [Requirements]

* All functions with Java implementations get constant folded on the coordinator
* All functions without Java implementations skip constant folding
* Both functions with and without Java implementations get registered as "builtin", meaning that they don't need to be fully qualified to be used

Breakdown by case
* Function with only java implementation: constant fold, existing behavior
* Function with only C++ implementation: no constant fold
* Function with java and c++ implementation: constant fold, existing behavior
* Function with only SQL implementation: use sql implementation, existing behavior
* Function with SQL and C++ implementation: use c++ implementation, new behavior


### Non-goals

[Question]
* Use presto.default namespace for every built in function or have separate namespace for native sidecar functions?

[Answer]
* If we choose to use presto.default. prefix for every function (built in functions from coordinator and functions from sidecar), then we would need to resolve naming collisions by only choosing one implementation when there exists a coordinator implementation and a worker implementation.

* If we choose to have separate namespaces for built in functions from coordinator and functions from sidecar, then this would require extra changes on the worker code. Currently the worker code is using one prefix for every function, and this prefix is configurable. However, you cannot set a different prefix for different functions coming from the sidecar; they would all be the same prefix.

* Since adding more worker changes adds extra complexity and confusion, we decided to use presto.default. prefix for every function from sidecar that is registered.

[Question]
* What is the behavior for `SHOW FUNCTIONS`? Will it show both implementations in the returned list?

[Answer]
* Both implementations (worker and coordinator implementation) will be shown to the user if both exist. This is similar to existing behavior of have native sidecar enabled with 'presto.default.' prefix as the default prefix. However, there will be followup to apply the same conflict resolution logic to ensure that the implementation that will be invoked is shown in `SHOW FUNCTIONS`.

## Proposed Implementation

The approach will be somewhat similar to this PR: https://github.com/prestodb/presto/pull/25597

Instead of adding a BuiltInPluginFunctionNamespaceManager, add a BuiltInNativeFunctionNamespaceManager. Register the functions from native sidecar under the presto.default namespace. During function evaluation inside of FunctionAndTypeManager#resolveFunctionInternal and getMatchingFunctionHandle, add some logic to determine whether the function in BuiltInTypeAndFunctionNamespaceManager or the function in BuiltInNativeFunctionNamespaceManager is used. This is similar to the PR, except the logic in PR will throw an error if it sees that there is a conflict. If there is a conflict, follow this logic to determine which functions to use:


* If the one already inside the BuiltInTypeAndFunctionNamespaceManager is a Java implementation, then use the one in BuiltInTypeAndFunctionNamespaceManager
* If the one already inside the BuiltInTypeAndFunctionNamespaceManager is a SQL implementation, then use the one in BuiltInNativeFunctionNamespaceManager


## Other Approaches Considered
One other approach that was considered involved selectively registering either the worker or coordinator implementation within BuiltInTypeAndFunctionNamespaceManager. This can be done by adding the Built in functions in getBuiltInFunctions to the function map as well as adding the ones pulled in from sidecar. After that, deduplicate the functions using the same logic adobe when there is a conflict. This will ensure that a function name can only refer to exactly one implementation; each function name has a unique implementation.

The downside of this approach is that it can add complexity to the logic inside of the BuiltInTypeAndFunctionNamespaceManager. Another concern is that the native sidecar requires that there is at least one active worker sidecar node to read the function registry from. However, during coordinator server startup, all the builtin functions and all the plugin functions are registered before the node manager is ready, which means no worker nodes can be used to pull the function registry.

```
// PrestoServer.java

// this is where the built in functions are registered because they are loaded when ServerMainModule is installed
Bootstrap app = new Bootstrap(modules.build());

....

injector.getInstance(PluginManager.class).loadPlugins();

...

// This is where NodeManager is setup. Only after this point can the sidecar successfully fetch the functions from worker

NodeInfo nodeInfo = injector.getInstance(NodeInfo.class);
PluginNodeManager pluginNodeManager = new PluginNodeManager(nodeManager, nodeInfo.getEnvironment());
planCheckerProviderManager.loadPlanCheckerProviders(pluginNodeManager);

```

This should show why the approach is challenging. We need access to the sidecar functions when the server modules are built in order to know whether to keep the coordinator implementation or the worker implementation, but we can only have access to the sidecar functions after node manager setup. This issue does not affect the initially proposed approach because the BuiltInNativeFunctionNamespaceManager can just do something similar to NativeFunctionNamespaceManager by wrapping the FunctionMap in a memoize so that the functions are only attempted to be loaded when needed.

```
// NativeFunctionNamespaceManager.java

this.memoizedFunctionsSupplier = Suppliers.memoize(this::bootstrapNamespace);

```

## Adoption Plan

In order to control rolling out this feature, add a new server property to enable this feature or not.

However, if this use case of only overriding specific functions is not needed anymore, the feature can be removed, and we can set native.default to be default namespace to just use the existing native function namespace manager.

More details in this migration plan: https://github.com/prestodb/presto/issues/25905

## Test Plan

1. Unit test to invoke a SQL function that has a C++ implementation
2. Unit test to invoke a Java function that has Java implementation
3. All other cases should just use coordinator implementation