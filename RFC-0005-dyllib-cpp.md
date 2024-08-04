## Creating a Dynamically Linked Functions Library in CPP

Proposers

* Soumya Duriseti
* Tim Meehan

## [Related Issues]

https://github.com/facebookincubator/velox/pull/1005

## Summary
This proposed change adds the ability to load User Defined Functions (UDFs) without having to fork and build Prestissimo through the use of shared libraries.
The Prestissimo worker is to access said code. The dynamic functions are to be loaded upon running an instance of the presto server. In the presto server instance, it will search for any .so or .dylib files and load them using this library.
## Background
Currently, on Presto, any Java UDFs can be loaded dynamically through the use of the Plugin SPI. These plugins allow Presto users to add a custom flavor of the Presto language through the introductions of functions not known to the Presto engine. Currently, it is not possible to add custom functions to Prestissimo without:
1. Adding a corresponding Java function through the use of the Plugin SPI.
2. Forking Prestissimo to manually register the custom function. 

RFC-0003 added the ability to validate C++ functions in the Presto coordinator without creating a corresponding function in the Java SPI. RFC 0005 allows users to register custom functions dynamically without having to fork prestissimo.

### [Optional] Goals
Register custom functions dynamically without requiring a fork of Prestissimo.
### [Optional] Non-goals
Security concerns: There are some security concerns associated with using the built in plugins. Using the dlopen library, we run the risk of opening unsecure unknown shared objects especially given the lack of any form of validation. On C++, we share the same limitations as the functionality in Java with a noted exception of Java Presto running in a VM environmnet while the C++ version will be run locally. 

Performance concerns: Users may write poorly-written functions which may degrade performance.

Reliability concerns: Users may write poorly-written functions which may cause instability to the Presto cluster.

This is a well-known limitation of unfenced functions. Users are aware of these risks which are also concerns of the existing Plugin SPI. The goal of this RFC is to bring feature parity with the Plugin SPI.
## Proposed Implementation
The user can register their functions dynamically by creating .dylib or .so shared libraries and dropping them in a plugin directory. This plugin directory needs to be defined with the plugin.dir property in config.properties of the prestissimo worker. This directory will be scanned and all shared libraries will be attempted to be dynamically loaded when the worker or the sidecar process starts.

For dynamically loaded function registration, the format followed is mirrored of that of built-in function registration with some noted differences. For instance, the below example function uses the extern "C" keyword to protect against name mangling. Additionally, a registry() function call is also necessary here.
```
namespace facebook::presto::functions {

template <typename TExecParams>
struct Dynamic123Function {
  FOLLY_ALWAYS_INLINE bool call(int64_t& result) {
    result = 123;
    return true;
  }
};

} // namespace facebook::presto::functions

extern "C" {

void registry() {
  facebook::velox::registerFunction<
      facebook::presto::functions::Dynamic123Function,
      int64_t>({"dynamic_123"});
}
}
```

The general process is as follows:

1. What modules are involved
    dlfcn.h (Dynamic linking library)
    udf.h (for function registerations)
2. Any new terminologies/concepts/SQL language additions
    None.
3. Method/class/interface contracts which you deem fit for implementation.
    None.
4. Code flow using bullet points or pseudo code as applicable
    1. User is to create a shared library (in the form of a .so or .dylib file) for the UDF they wish to register. This is freeform with the noted exception that they follow the velox function registry API to create their UDF. This will be noted in the documentation. One way to create shared libraries is through CMake using the SHARED keyword in the CMakeLists.txt.
    2. User will proceed to place their .so/.dylib files into the plugin directory.
    3. Upon running the sidecar or the Prestissimo worker, we will scan the plugin directory to load the .so/.dylib files dynamically using a call to loadDynamicLibraryFunctions(). This function uses dlopen() to dynamically load these files.
5. Any new user facing metrics that can be shown on CLI or UI.
    None.
## [Optional] Metrics

We indend to use Pbench for performing performance testing. This library's effectiveness can be measured by successful completion of registering all of UDFs and validating their proper registration using the CLI with a call to SHOW FUNCTIONS and by successful completion of the process.

## [Optional] Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
No impact as this is a new offering.
- If we are changing behaviour how will we phase out the older behaviour?
- If we need special migration tools, describe them here.
- When will we remove the existing behaviour, if applicable.
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
A README with these changes will be included to explain to users how to properly use the dyllib functionality.
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
None.

## Test Plan

An E2E test will go through the entire process and validate the function registering with a SHOW FUNCTIONS call. Performance testing using pbench will also be done.