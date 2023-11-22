# **RFC1 for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## Loading error messages from resource bundles / localization of error messages

Proposers

* Elbin Pallimalil

## [Related Issues]

https://github.com/prestodb/presto/issues/21199

## Summary

Move all the inline error messages to resource bundles. Take advantage of this change to serve client locale specific error messages from Presto.

## Background

Presto now has inline error messages written in multiple locations. If we need to review the error messages we don't have a central location to review all the messages. We can move the inline error messages to resource files to keep them in a central location.

When we are moving the error messages to resource files, we can also load locale specific error messages during server startup so that we can serve client locale specific error messages from Presto. 

## Proposed Implementation

 1. Create a default Messages.properties resource under src/main/resources/error folder in presto-main module 
 2. Add all error messages used in Presto code in Messages.properties 
 3. Deprecate inline error messages in Presto code and load message from Messages.properties 
 4. Check if there is default property file or if there are Messages files for different locales in etc/resources/error folder during Presto startup. If yes load those locale specific resources. 
 5. When loading messages from resources, load from the locale specific resource. The locale to be used should be determined from the client session. 
 6. Each plugin can define additional error messages under src/main/resources/error folder in the corresponding plugin module. 
 7. These resource files from plugins will be read on server startup and a combined resource file for each locale will be created. 
 8. On server startup, read default and locale specific Messages.properties files from plugin/<plugin-name>/resources/error 
 9. Load localized error messages in QueuedStatementResource.toQueryError method by getting the client locale from sessionContext.language 
 10. Warn when the default resource bundle entries != the translated entries 
 11. When a translation is not available, a locale message is transcribed that explains an error message is not available, and then prints the message in the default locale (English).  That makes it embarrassing to the deployer of Presto, but less bad than preventing debugging. 
 12. Presto exceptions in query logs contain both the default and the locale-specific exception messages 
 13. Compile time validation of error message keys. 
        1. Unit test validation 
            1. All error message keys should be defined as enums 
            2. Write a unit test to ensure all the defined enums have a corresponding entry in default Messages.properties file. 
        2. Run time validation 
            1. All error message keys should be defined as enums 
            2. On server startup verify that all the defined enums have a corresponding entry in default Messages.properties file. 
 14. OSS community will maintain English bundle for presto-main and the other connectors. 
 15. OSS community can optionally maintain additional bundles for different locales but Presto can also read available bundles from a specified path at runtime, so sys admins can choose to maintain their own version of localized error bundles. 
 16. The error bundles for English and localized bundles for both presto-main and other connectors can be verified to be consistent in step 13. 2. mentioned above. (If we choose that approach) 

## [Optional] Metrics

How can we measure the impact of this feature?

## [Optional] Other Approaches Considered

Based on the discussion, this may need to be updated with feedback from reviewers.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
- If we are changing behaviour how will we phase out the older behaviour?
- If we need special migration tools, describe them here.
- When will we remove the existing behaviour, if applicable.
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Test Plan

How do we ensure the feature works as expected? Mention if any functional tests/integration tests are needed. Special mention for product-test changes. If any PoC has been done already, please mention the relevant test results here that you think will bolster your case of getting this RFC approved.