## Creating a Dynamically Linked Library in CPP

Proposers

* Soumya Duriseti
* Tim Meehan

## [Related Issues]

https://github.com/facebookincubator/velox/pull/11439/
https://github.com/prestodb/presto/pull/24330

## Summary
This proposed change adds the ability to load User Defined Functions (UDFs), types, and connectors without having to fork and build Prestissimo through the use of shared libraries.
The Prestissimo worker is to access said code. The dynamic shared libraries are to be loaded upon running an instance of the presto server. In the presto server instance, it will look through a user provided Json config for the specified dynamic libraries in the plugin directory and load them dynamically.
## Background
Currently, on Presto, any Java UDFs, types, and connectors can be loaded dynamically through the use of the Plugin SPI. These plugins allow Presto users to add a custom flavor of the Presto language through the introductions of functions not known to the Presto engine. To describe the current flow, it is not possible to add custom elements to Prestissimo without:
1. Adding a corresponding Java function through the use of the Plugin SPI.
2. Forking Prestissimo to manually register the custom elements. 

RFC-0003 added the ability to validate C++ functions in the Presto coordinator without creating a corresponding function in the Java SPI. RFC 0005 allows users to register custom functions dynamically without having to fork prestissimo.

### [Optional] Goals
Register custom functions, types, and connectors dynamically without requiring a fork of Prestissimo.
### [Optional] Non-goals
Security concerns: There are some security concerns associated with using the built in plugins. Using the dlopen library, we run the risk of opening unsecure unknown shared objects especially given the lack of any form of validation. On C++, we share the same limitations as the functionality in Java with a noted exception of Java Presto running in a VM environmnet while the C++ version will be run locally. 

Performance concerns: Users may write poorly-written custom elements which may degrade performance.

Reliability concerns: Users may write poorly-written custom elements which may cause instability to the Presto cluster.

This is a well-known limitation of unfenced functions. Users are aware of these risks which are also concerns of the existing Plugin SPI. The goal of this RFC is to bring feature parity with the Plugin SPI.
## Proposed Implementation
This library will be implemented in three stages, with the first one being for User Defined Functions. Type and Connectors will build onto the dynamic library following the same format in terms of the implementation and design.

### General Implementation
The user can register their custom elements dynamically by creating .dylib or .so shared libraries and dropping them in a plugin directory. This plugin directory needs to be defined with the plugin.dir property in config.properties of the prestissimo worker. This directory will be scanned and all shared libraries will be attempted to be dynamically loaded when the worker or the sidecar process starts.

#### User Defined Function Implementation
For dynamically loaded function registration, the format followed is mirrored of that of built-in function registration with some noted differences. For instance, the below example function uses the extern "C" keyword to protect against name mangling. Additionally, a registry() function call is also necessary here.

```
#include "presto_cpp/main/dynamic_registry/DynamicFunctionRegistrar.h"

namespace facebook::presto::common::dynamicRegistry {

template <typename T>
struct DynamicFunction {
  FOLLY_ALWAYS_INLINE bool call(int64_t& result) {
    result = 123;
    return true;
  }
};

} // namespace facebook::presto::common::dynamicRegistry

extern "C" {
void registry12() {
  facebook::presto::registerPrestoFunction<
      facebook::presto::common::dynamicRegistry::DynamicFunction,
      int64_t>("dynamic");
}
}
```
To turn these files into shared libraries, the CMakeLists.txt file will be used.

#### Config Json File Implementation
To aid in the experience of creating and validating the creation of these shared libraries, the below Json file format is proposed. The goal of this config format is to bring closer feature parity with the java presto UDF registerations and the Prestissimo remote function UDF registerations.

```
{
  "dynamicLibrariesUdfMap": {
    "function_registerations": {
      "dynamic": [
        {
          "docString": "",
          "functionKind": "SCALAR",
          "outputType": "INTEGER",
          "entrypoint": "registry12", 
          "fileName": "libpresto_function_my_dynamic",
          "paramTypes": [
            "INTEGER"
          ],
          "namespace": "presto.default",
          "routineCharacteristics": {
            "language": "CPP",
            "determinism": "DETERMINISTIC",
            "nullCallClause": "CALLED_ON_NULL_INPUT"
          }
        }
      ]
    }
  }
}
```

This Json format builds on the existing JsonSignatureParser.cpp format with 4 notable changes:
1. The subdirectory object where the user specifies which subdirectory their library can be found. In the above example, we can see that it can be found in the "function_registerations" subdirectory. This object allows for the user to create various subdirectories to store their dynamic libraries. If the user wishes to add shared libraries to the top-level directory, they may leave the field blank.

2. The `entrypoint` field where the user can optionally specify a custom entrypoint. The value of the entrypoint defaults to `registry` in the event that the user skips this field.

3. The `namespace` field where the user will specify the namespace they choose to give their UDF. The value defaults to the `prestoDefaultNamespacePrefix` of `presto.default` in case the user does not specify a value here.

4. The `fileName` field. This is where the name of the library will be specified.

Additionally, the config property `plugin.config` will need to be set in the workers' configurations for the PrestoServer instance to find and parse this config.

#### Function Registeration Validation
Due to the nature of `dlopen()`, there is no way to know before loading the shared library what to expect inside the file. To combat this ambiguity and to ensure functions are properly loaded, registeration validations will make for a smoother user experience.

Driven by the config, the data will be parsed and put into two maps:

```
// To group possible functions with the same name but different
// signatures.
  folly::Synchronized<std::unordered_map<std::string, std::vector<velox::exec::FunctionSignaturePtr>>>
      functionMap_;

// (non duplicate entrypoints consisting of subdirectory/filename) -> entrypoint string
folly::Synchronized<std::unordered_map<std::string, std::string>> entrypointMap_;
```

`entrypointMap_` is used before the dylib method is called. By iterating through this map and filtering out any config signatures with invalid paths, all loadable shared libraries coupled with entrypoints are loaded through the dynamic loading library. This implementation provides additional flexibility for the user where one shared library is allowed to have more than one unique entrypoint. Moreover, we can assume that these entrypoints will be unique in every shared library due to the nature of creating the shared libraries where these files need to remain compilable.

`functionMap_` will hold the name of the function as the key and all the various signatures associated with this function name as the value. After loading all dynamic libraries, this map will be used to check if every function signature mentioned in the Json config got registered. At this time, if there is any discrepancy between the number of functions actually registered and the number projected by the config, those missing signatures will be logged and shown to the user.


There is a limitation with this setup where any user-made typos in the `fileName`, `entrypoint`, or `subdirectory` will lead to omission of some shared libraries. With that said, the user might omit other libraries in the same directory purposefully, so it is important to decide if the additional flexibility outweighs the cost of potentially missing a few libraries from being registered.

#### Error and Signal Handling
Whenever there is a signal thrown such as SIGSEGV, SIGBUS, or SIGABRT while loading a shared library, it can be extremely disruptive as it would kill the worker. Prioritizing minimal disruptions, the proposed DynamicSignalHandler.cpp creates a custom signal handler which uses sigaction to redirect SIGSEGV, SIGBUS, SIGILL, and SIGABRT signals. Once all dynamic libraries are loaded, the signal handler is restored to the original one. The error handling would look like this in action:

if the dynamic library looked like below:
```
...
extern "C" {
void registry12() {
  raise(SIGSEGV);
}
}
```
Then during the run, the error would get caught and then ignored to avoid crashing the worker.
```
E0319 22:14:14.478333 154519 DynamicSignalHandler.cpp:55] Caught signal 6 in dynamic library loading. Preventing Crash.
E0319 22:14:14.478353 154519 DynamicSignalHandler.cpp:57] Fault address: 0x25b97
```

With this signal handling method using sigaction, it is important to note that only the redirected signals will be handled and any other ones will crash the system. Depending on the architecture there can be different codes for different errors. Also, the behavior of the process is undefined after
it ignores a SIGFPE, SIGILL, or SIGSEGV signal that was not generated by kill(2) or raise(3). There is also an inherent risk to ignoring signals such as being stuck in endless loops. Finally, It is not possible to block SIGKILL or SIGSTOP.


1. What modules are involved
    dlfcn.h (for dynamic linking library)
    signal.h (for signal handling)
    Registerer.h (for function registerations)
    Type.h (for type registeration)
    Connector.h (for connector registration)

2. Any new terminologies/concepts/SQL language additions
    None.
3. Method/class/interface contracts which you deem fit for implementation.
    None.
4. Code flow using bullet points or pseudo code as applicable
    1. User is to create a shared library (in the form of a .so or .dylib file) for the UDF they wish to register. This is freeform with the noted exception that they follow the velox function registry API to create their UDF. This will be noted in the documentation. One way to create shared libraries is through CMake using the SHARED keyword in the CMakeLists.txt.
    2. User will also create a Json config for any of the shared libraries they would like to be registered. They will copy over the function signature, metadata, and file information into this format.
    3. User will proceed to place their .so/.dylib files into the plugin directory.
    4. User will set the plugin.dir and plugin.config with the location of their plugin directory and json config respectively.
    5. Upon running the sidecar or the Prestissimo worker, we will go through all the function signatures in the configs and register their respective libraries located in plugin directory. Here, we will load the .so/.dylib files dynamically using a call to loadDynamicLibraryFunctions(). This function uses dlopen() to dynamically load these files.

5. Any new user facing metrics that can be shown on CLI or UI.
    None.
## [Optional] Metrics

We indend to use Pbench for performing performance testing. This library's effectiveness can be measured by successful completion of registering all of UDFs and validating their proper registration using the CLI with a call to SHOW FUNCTIONS and a SQL query invoking the registered function will indicate successful completion of the process.

## [Optional] Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
No impact as this is a new offering.
- If we are changing behaviour how will we phase out the older behaviour?
- If we need special migration tools, describe them here.
- When will we remove the existing behaviour, if applicable.
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
A presto-docs entry with these changes will be included to explain to users how to properly use the dylib functionality.
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
None.

## Test Plan

An E2E test will go through the entire process and validate the function registering with a call to SHOW FUNCTIONS and a SQL query invoking the registered function in a containerized test.