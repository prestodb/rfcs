# **RFC-0011 for Presto**

## Introduce new SPI for sql invoked functions

Proposers
* Pratik Joseph Dabre
* Tim Meehan

## Related Issues

* https://github.com/prestodb/presto/issues/24964
* https://github.com/prestodb/presto/pull/25597

## Summary

This RFC proposes the introduction of a new SPI method, `getSqlInvokedFunctions()` that enables plugin-based registration of SQL invoked functions in Presto. 
This SPI is supported via a new function namespace manager , `BuiltInPluginFunctionNamespaceManager`, which isolates SQL invoked functions loaded through a plugin from native or built-in functions.
To support different runtime(sidecar-enabled native and java/sidecar disabled native) environments, two separate plugin modules will be introduced:
- One for java-only/native deployments that includes all inlined SQL invoked functions currently registered under `BuiltInTypeAndFunctionNamespaceManager`.
- One for native-sidecar deployments that includes all inlined SQL invoked functions currently registered under `BuiltInTypeAndFunctionNamespaceManager` but excludes functions overridden natively.

This design aims to prevent conflicts in function resolution, especially when native functions are integrated into the coordinator in a sidecar-enabled cluster.


## Background

In sidecar-enabled clusters, users can configure the default function namespace that will be used to serve functions. The C++ functions are then loaded via an endpoint under this specified namespace.
Previously, inlined SQL invoked functions were registered under the `presto.default` namespace (the default Java built-in namespace). Native functions pulled from the c++ sidecar were registered under a separate namespace like `native.default`.
These native functions contain overridden optimized implementations of some of the inlined SQL invoked functions that the users would like to take advantage of. 
However, this setup led to conflict resolution complexity. Presto would need to prioritize between two namespaces `presto.default` and `native.default` when resolving functions and such namespace-based prioritization is opaque and not ideal.

To address this, the proposed change moves all inlined SQL invoked functions into a plugin-based namespace manager `BuiltInPluginFunctionNamespaceManager` which serves whatever the current default function namespace is. 
This ensures:
- All inlined SQL invoked functions are grouped together cleanly and loaded via a plugin.
- Overridden native implementations of inlined SQL invoked functions are excluded from plugin registration.
- The default serving catalog resolves functions seamlessly since all SQL invoked functions still live under the same default catalog but are now plugin loaded and managed by a plugin-based namespace manager `BuiltInPluginFunctionNamespaceManager`.

### Goals
- Introduce a new SPI method, `getSqlInvokedFunctions()` for plugin-based registration of SQL invoked functions.
- Create and wire a new plugin-based namespace manager `BuiltInPluginFunctionNamespaceManager` to manage these functions.
- Enable native sidecar environments to load only the relevant subset of inlined SQL-invoked functions (i.e., excluding ones overridden natively).
- Guarantee that inlined SQL invoked functions still work under the default catalog using the new plugin namespace manager.

### Non-goals
- This RFC does not change the execution model for inlined SQL invoked functions - they are still inlined in Java as earlier.
- It does not modify function resolution rules outside of registration-time separation (i.e., no new prioritization logic is added).
- It does not impact native (functions pulled from sidecar) or Java built-in function registration, which remains handled by the `NativeFunctionNamespaceManager` and `BuiltInTypeAndFunctionNamespaceManager` respectively.

## Proposed Implementation

This section describes the internal changes made to safely register SQL invoked functions via a new plugin SPI and isolate them from built-in functions. It includes SPI definition, 
the design of a new namespace manager, conflict detection and lazy resolution mechanisms.
- New SPI: `getSqlInvokedFunctions()`

    A new method was introduced in the plugin SPI to allow SQL invoked functions to be loaded:
    
    ```
    default Set<Class<?>> getSqlInvokedFunctions()
        {
            return emptySet();
        }
    ```
    
    This SPI is picked up by the plugin manager at Presto startup. These functions are routed to a new plugin-specific namespace manager described below.


- New Class : `BuiltInPluginFunctionNamespaceManager`

   A new class, `BuiltInPluginFunctionNamespaceManager`, was introduced to hold all SQL invoked functions loaded from plugins.
   
  - Responsibilities:
      - Manage SQL invoked functions from plugin modules
      - Serve only the configured default serving namespace
      - Ensure no duplicate function signatures are registered
      - Support lazy caching and resolution in  sidecar environments

     - #### Constructor and Field initialization:

       - ```
         private final Supplier<FunctionMap> cachedFunctions =
                Suppliers.memoize(this::checkForNamingConflicts);
         ```
       - ```
         private synchronized FunctionMap checkForNamingConflicts()
         {
              Optional<FunctionNamespaceManager<?>> functionNamespaceManager =
              functionAndTypeManager.getServingFunctionNamespaceManager(functionAndTypeManager.getDefaultNamespace());
              checkArgument(functionNamespaceManager.isPresent(), "Cannot find function namespace for catalog '%s'", functionAndTypeManager.getDefaultNamespace().getCatalogName());
              checkForNamingConflicts(functionNamespaceManager.get().listFunctions(Optional.empty(), Optional.empty()));
              return functions;
         }
         ```
       This defers the conflict check until functions are actually used, particularly useful in sidecar-enabled clusters.

     - #### Function Registration workflow:
       Functions from each plugin implementing the new SPI are registered as follows:
       - ```
         public void registerPluginFunctions(List<? extends SqlFunction> functions)
         {
            builtInPluginFunctionNamespaceManager.registerPluginFunctions(functions);
         }
         ```
       - ```
          public synchronized void registerPluginFunctions(List<? extends SqlFunction> functions)
          {
            checkForNamingConflicts(functions);
            this.functions = new FunctionMap(this.functions, functions);
          }
         ```
  
       This ensures plugin-provided SQL functions are separated from built-in/native functions and properly validated.

     - #### Conflict Detection  and signature validation:

       Two layers of conflict checks are added: 
      1. During registration against SQL invoked functions from other plugins
         - ```
           private synchronized void checkForNamingConflicts(Collection<? extends SqlFunction> functions)
           {
               for (SqlFunction function : functions) {
                   for (SqlFunction existingFunction : this.functions.list()) {
                       checkArgument(!function.getSignature().equals(existingFunction.getSignature()), "Function already registered: %s", function.getSignature());
                   }
               }
           }
         ```
  
      2. Against current default serving namespace functions
    
      - ```
          private synchronized FunctionMap checkForNamingConflicts()
          {
            Optional<FunctionNamespaceManager<?>> functionNamespaceManager =
            functionAndTypeManager.getServingFunctionNamespaceManager(functionAndTypeManager.getDefaultNamespace());
            checkArgument(functionNamespaceManager.isPresent(), "Cannot find function namespace for catalog '%s'", functionAndTypeManager.getDefaultNamespace().getCatalogName());
            checkForNamingConflicts(functionNamespaceManager.get().listFunctions(Optional.empty(), Optional.empty()));
            return functions;
          }
        ```
  
         This ensures that plugin provided functions don't conflict with other registered functions , both with other plugins and also the current default namespace manager.

    - #### Sidecar-Aware Behavior and Lazy Function Resolution:
         In clusters where native-sidecar functions are enabled, function registration and resolution must be carefully orchestrated to:
        - Avoid duplicate function signatures
        - Delay conflict checks until both plugin and native functions are available.
        - This is triggered only upon first use of:
          - `SHOW FUNCTIONS`
          - `DESCRIBE FUNCTION`
          - Internal resolution calls like `resolveFunction`
        - ```
          public Collection<? extends SqlFunction> getFunctions(QualifiedObjectName functionName)
          {
              if (functions.list().isEmpty() ||
                      (!functionName.getCatalogSchemaName().equals(functionAndTypeManager.getDefaultNamespace()))) {
                  return emptyList();
              }
              return cachedFunctions.get().get(functionName);
          }
          ```
    
          For java clusters and non-sidecar enabled native clusters, the first part of the above check is helpful as unless and until the plugin functions are loaded , we don't compare them with the underlying 
          serving function namespace i.e. `JAVA_BUILTIN_NAMESPACE` or `presto.default` functions.
    
          For sidecar enabled clusters, the second part of the above check is helpful as if it's not a function under the default namespace , don't trigger the conflict validation with the sidecar fetched functions. 
          This works particularly because while starting the coordinator there are going to be functions which need to be resolved, we don't want to trigger the conflict validation then, but delay it until a function from the `native.default` namespace is requested.
          This strategy allows the coordinator to finish initializing and calls the sidecar lazily to trigger the conflict check.

      - #### Metadata and execution support:
          This section explains how SQL invoked functions registered via the plugin SPI are integrated into Presto's function resolution and execution infrastructure. Both metadata fetching and execution are handled with full support for plugin-based SQL invoked functions.

        - When a function is resolved (e.g. during analysis or planning), Presto now checks both:
            The current default serving namespace functions (handled by `BuiltInTypeAndFunctionNamespaceManager` in java/non-sidecar enabled clusters and `NativeFunctionNamespaceManager` in sidecar enabled clusters).
            The new `BuiltInPluginFunctionNamespaceManager`, which contains all plugin provided SQL invoked functions.
            This logic is encapsulated in the below method:
    
        ```
        private Collection<? extends SqlFunction> getFunctions(
                  QualifiedObjectName functionName,
                  Optional<? extends FunctionNamespaceTransactionHandle> transactionHandle,
                  FunctionNamespaceManager<?> functionNamespaceManager)
          {
              return ImmutableList.<SqlFunction>builder()
                      .addAll(functionNamespaceManager.getFunctions(transactionHandle, functionName))
                      .addAll(builtInPluginFunctionNamespaceManager.getFunctions(functionName))
                      .build();
          }
    
          /**
           * Gets the function handle of the function if there is a match. We enforce explicit naming for dynamic function namespaces.
           * All unqualified function names will only be resolved against the built-in default function namespace. We get all the candidates
           * from the current default namespace and additionally all the candidates from builtInPluginFunctionNamespaceManager.
           *
           * @throws PrestoException if there are no matches or multiple matches
           */
          private FunctionHandle getMatchingFunctionHandle(
                  QualifiedObjectName functionName,
                  Optional<? extends FunctionNamespaceTransactionHandle> transactionHandle,
                  FunctionNamespaceManager<?> functionNamespaceManager,
                  List<TypeSignatureProvider> parameterTypes,
                  boolean coercionAllowed)
          {
              Optional<Signature> matchingDefaultFunctionSignature =
                      getMatchingFunction(functionNamespaceManager.getFunctions(transactionHandle, functionName), parameterTypes, coercionAllowed);
              Optional<Signature> matchingPluginFunctionSignature =
                      getMatchingFunction(builtInPluginFunctionNamespaceManager.getFunctions(functionName), parameterTypes, coercionAllowed);
    
              if (matchingDefaultFunctionSignature.isPresent() && matchingPluginFunctionSignature.isPresent()) {
                  throw new PrestoException(AMBIGUOUS_FUNCTION_CALL, format("Function '%s' has two matching signatures. Please specify parameter types. \n" +
                           "First match : '%s', Second match: '%s'", functionName, matchingDefaultFunctionSignature.get(), matchingPluginFunctionSignature.get()));
              }
    
              if (matchingDefaultFunctionSignature.isPresent()) {
                  return functionNamespaceManager.getFunctionHandle(transactionHandle, matchingDefaultFunctionSignature.get());
              }
    
              if (matchingPluginFunctionSignature.isPresent()) {
                  return builtInPluginFunctionNamespaceManager.getFunctionHandle(matchingPluginFunctionSignature.get());
              }
    
              throw new PrestoException(FUNCTION_NOT_FOUND, constructFunctionNotFoundErrorMessage(functionName, parameterTypes,
                      getFunctions(functionName, transactionHandle, functionNamespaceManager)));
          }
        ```

## Adoption Plan

Users would need to load the newly introduced plugin modules if they wish to use the inlined SQL functions that Presto currently supports. 
This change applies to both Java-based and native sidecar-enabled/non-enabled Presto clusters.
- In Java-only/ non-sidecar enabled native clusters, users can load the `presto-sql-invoked-functions-plugin` plugin module containing all the inlined SQL invoked functions.
- In native clusters with sidecar enabled, users should load the trimmed-down plugin module `presto-native-sql-invoked-functions-plugin` that excludes functions overridden by native implementations to avoid signature conflicts.

This design ensures a smooth transition, enabling explicit control over which functions are available at runtime based on the deployment context.

## Test Plan

1. Unit Tests 
   - Register plugin functions
   - Trigger signature conflicts
   - Validate new function namespace manager behaviour and caching,  ensure memoization is called once.
   - Trigger resolution via SHOW FUNCTIONS, etc.

2. Integration Tests
   - Java-only / native non-sidecar enabled clusters:
     - Register `presto-sql-invoked-functions-plugin` plugin module.
     - Run queries using the inlined sql invoked functions present under `presto-sql-invoked-functions-plugin`.

   - Sidecar-enabled clusters:
     - Use native sidecar safe plugin `presto-native-sql-invoked-functions-plugin` module.
     - Validate conflict detection (e.g. `test123` not double-registered), if conflicts, need to fail gracefully.