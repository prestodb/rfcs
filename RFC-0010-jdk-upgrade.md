# **RFC-0010 for Presto**

## JDK Upgrade for Presto

Proposers

* ZacBlanco
* imjalpreet
* aaneja


## Summary

This RFC lists out the steps and process for upgrading the build and runtime JDK version used in Presto to a more recent LTS release like 17 and 21  
As of today (2025-01-24) the version we use is JDK 8


## Background

The Java community has slowly but surely been moving to JDK versions beyond JDK 8  

Some of our dependencies, e.g [Jetty](https://github.com/jetty/jetty.project/) have already moved to JDK 17, and declared prior versions to be out-of-support(See https://github.com/jetty/jetty.project/issues/10485. The latest supported version is Jetty 12)

Not using modern versions of our dependencies opens Presto installations to security risks through CVEs, which hurts adoption of Presto

See this [Prestocon 2024 presentation](https://www.youtube.com/watch?v=K6CjuPZtqW4&list=PLJVeO1NMmyqW_qoMMEyVq-wW8lBSxA7hP&index=10) for a deep-dive motivating the issue further (Slides [here](RFC-0010/PrestoJavaUpgradeLightningTalk.pdf))


## Proposed Migration Plan  

To migrate to newer versions of the JDK, we propose a phased approach -   

| Phase | Time Window | Release Version | Runtime JDK Support | Compilation JDK Support |
| --- | --- | --- | --- | --- |
| Dual Support | Month N | X | JDK Y and Y+1 | JDK Y |
| Dual Support & Deprecation | Month N + 2 | X+1 | JDK Y and Y+1 | JDK Y |
| Stable JDK | Month N + 4 | X+2 | JDK Y+1 | JDK Y+1 |

_Note :  
JDK Y - Current JDK version (e.g. 8)  
JDK Y+1 - Next JDK version (e.g. 17)_


The salient points - 

 - During the upgrade, the project will be built with two JDK versions , the current and the new target version (e.g JDK 8 and 17) simultaneously
 - We will make use of our CI pipelines to run the full breadth of unit, integration and end-to-end product tests on the newest JDK version that we intend to support. (e.g. in the case of 8 to 17 upgrade, we will run our CI on Java 17)
 - Users will be invited to test new builds with their production workloads to report any issues observed around correctness, performance and stability
 - As blocking issues shake out and get resolved, we will drop support for older JDK versions over one or two releases, making the new JDK version the default


## Current Blockers
We list the current blockers for the migration effort below

### Presto On Spark
[Presto on Spark](https://github.com/prestodb/presto/issues/13856) (PoS) integrates Presto's SQL engine with Apache Spark's distributed computing capabilities, allowing Presto queries to leverage Spark's scalability and fault tolerance for large-scale batch processing workloads. 

While the Spark project has been putting out releases supporting newer JDK versions, Presto's users may not have migrated to the latest Spark version

Here's is a summary of the max version that a Spark application can have, by Spark version (see [link](https://community.cloudera.com/t5/Community-Articles/Spark-and-Java-versions-Supportability-Matrix/ta-p/383669) for more details)


| Spark Version | Supported Java Version(s) |
| --- | --- | 
| 4.x (unreleased) | Java 8/11/17/21 | 
| 3.3.0 - 3.5.1 | Java 8/11/17 | 
| 3.0.0 - 3.2.4 | Java 8/11 | 
| 2.4.0 - 2.4.8 | Java 8 |


This means, for example, if we upgrade the bytecode level of Presto to that of JDK 17, users would only be able to use PoS on Spark 3.3.0+ clusters

### Workaround
The [presto-spark](https://github.com/prestodb/presto/tree/498784bb35feaf2787da865865ffe8ab3e8673ca/presto-spark) maven module has the below dependency graph -

<details>

```
com.facebook.presto:presto-spark:jar:0.291-SNAPSHOT
\- com.facebook.presto:presto-spark-base:jar:0.291-SNAPSHOT:runtime
   +- com.facebook.presto:presto-client:jar:0.291-SNAPSHOT:runtime
   |  +- com.fasterxml.jackson.core:jackson-core:jar:2.15.4:runtime
   |  +- com.fasterxml.jackson.core:jackson-databind:jar:2.15.4:runtime
   |  +- com.facebook.airlift:security:jar:0.216:runtime
   |  +- com.facebook.drift:drift-api:jar:1.40:runtime
   |  +- com.google.auth:google-auth-library-oauth2-http:jar:0.12.0:runtime
   |  |  +- com.google.auth:google-auth-library-credentials:jar:0.12.0:runtime
   |  |  +- com.google.http-client:google-http-client:jar:1.27.0:runtime
   |  |  |  \- org.apache.httpcomponents:httpclient:jar:4.5.5:runtime
   |  |  |     +- org.apache.httpcomponents:httpcore:jar:4.4.9:runtime
   |  |  |     \- commons-codec:commons-codec:jar:1.17.0:runtime
   |  |  \- com.google.http-client:google-http-client-jackson2:jar:1.27.0:runtime
   |  +- com.squareup.okhttp3:okhttp:jar:3.9.0:runtime
   |  |  \- com.squareup.okio:okio:jar:1.13.0:runtime
   |  \- com.squareup.okhttp3:okhttp-urlconnection:jar:3.9.0:runtime
   +- com.facebook.presto:presto-parser:jar:0.291-SNAPSHOT:runtime
   |  \- org.antlr:antlr4-runtime:jar:4.7.1:runtime
   +- com.facebook.presto:presto-analyzer:jar:0.291-SNAPSHOT:runtime
   +- com.github.luben:zstd-jni:jar:1.5.2-3:runtime
   +- com.facebook.presto:presto-common:jar:0.291-SNAPSHOT:runtime
   +- com.facebook.presto:presto-spark-common:jar:0.291-SNAPSHOT:runtime
   +- com.facebook.presto:presto-spi:jar:0.291-SNAPSHOT:runtime
   +- com.facebook.presto:presto-main:jar:0.291-SNAPSHOT:runtime
   |  +- com.esri.geometry:esri-geometry-api:jar:2.2.4:runtime
   |  +- com.facebook.presto:presto-geospatial-toolkit:jar:0.291-SNAPSHOT:runtime
   |  |  \- org.locationtech.jts.io:jts-io-common:jar:1.19.0:runtime
   |  |     \- com.googlecode.json-simple:json-simple:jar:1.1.1:runtime
   |  +- org.apache.commons:commons-math3:jar:3.6.1:runtime
   |  +- com.facebook.presto:presto-bytecode:jar:0.291-SNAPSHOT:runtime
   |  |  +- org.ow2.asm:asm-tree:jar:9.2:runtime
   |  |  +- org.ow2.asm:asm-util:jar:9.2:runtime
   |  |  \- org.ow2.asm:asm-analysis:jar:9.2:runtime
   |  +- io.airlift:aircompressor:jar:0.27:runtime
   |  +- com.facebook.airlift:discovery:jar:0.216:runtime
   |  +- com.facebook.airlift:event:jar:0.216:runtime
   |  +- com.facebook.airlift:http-server:jar:0.216:runtime
   |  |  +- org.eclipse.jetty.http2:http2-server:jar:9.4.56.v20240826:runtime
   |  |  +- org.eclipse.jetty:jetty-server:jar:9.4.56.v20240826:runtime
   |  |  +- org.eclipse.jetty:jetty-servlet:jar:9.4.56.v20240826:runtime
   |  |  |  \- org.eclipse.jetty:jetty-util-ajax:jar:9.4.56.v20240826:runtime
   |  |  +- org.eclipse.jetty:jetty-security:jar:9.4.56.v20240826:runtime
   |  |  \- org.eclipse.jetty:jetty-jmx:jar:9.4.56.v20240826:runtime
   |  +- com.facebook.airlift:jaxrs:jar:0.216:runtime
   |  |  +- javax.xml.bind:jaxb-api:jar:2.3.1:runtime
   |  |  |  \- javax.activation:javax.activation-api:jar:1.2.0:runtime
   |  |  +- org.glassfish.jersey.core:jersey-common:jar:2.26:runtime
   |  |  |  \- org.glassfish.hk2:osgi-resource-locator:jar:1.0.1:runtime
   |  |  +- org.glassfish.jersey.core:jersey-server:jar:2.26:runtime
   |  |  |  +- org.glassfish.jersey.core:jersey-client:jar:2.26:runtime
   |  |  |  \- org.glassfish.jersey.media:jersey-media-jaxb:jar:2.26:runtime
   |  |  +- org.glassfish.jersey.containers:jersey-container-servlet-core:jar:2.26:runtime
   |  |  +- org.glassfish.jersey.containers:jersey-container-servlet:jar:2.26:runtime
   |  |  \- org.glassfish.jersey.inject:jersey-hk2:jar:2.26:runtime
   |  |     \- org.glassfish.hk2:hk2-locator:jar:2.5.0-b42:runtime
   |  |        +- org.glassfish.hk2:hk2-api:jar:2.5.0-b42:runtime
   |  |        +- org.glassfish.hk2:hk2-utils:jar:2.5.0-b42:runtime
   |  |        \- org.javassist:javassist:jar:3.22.0-GA:runtime
   |  +- com.facebook.airlift:jmx:jar:0.216:runtime
   |  |  \- com.sun:tools:jar:1.8:system
   |  +- com.facebook.airlift:jmx-http:jar:0.216:runtime
   |  +- io.airlift.resolver:resolver:jar:1.4:runtime
   |  |  +- org.sonatype.aether:aether-spi:jar:1.13.1:runtime
   |  |  +- org.sonatype.aether:aether-impl:jar:1.13.1:runtime
   |  |  +- org.sonatype.aether:aether-util:jar:1.13.1:runtime
   |  |  +- org.sonatype.aether:aether-connector-file:jar:1.13.1:runtime
   |  |  +- org.sonatype.aether:aether-connector-asynchttpclient:jar:1.13.1:runtime
   |  |  |  \- com.ning:async-http-client:jar:1.6.5:runtime
   |  |  +- io.netty:netty:jar:3.6.2.Final:runtime
   |  |  +- org.apache.maven:maven-core:jar:3.0.4:runtime
   |  |  |  +- org.apache.maven:maven-settings:jar:3.0.4:runtime
   |  |  |  +- org.apache.maven:maven-settings-builder:jar:3.0.4:runtime
   |  |  |  +- org.apache.maven:maven-repository-metadata:jar:3.0.4:runtime
   |  |  |  +- org.apache.maven:maven-plugin-api:jar:3.0.4:runtime
   |  |  |  +- org.apache.maven:maven-model-builder:jar:3.0.4:runtime
   |  |  |  +- org.codehaus.plexus:plexus-interpolation:jar:1.14:runtime
   |  |  |  +- org.codehaus.plexus:plexus-utils:jar:2.0.6:runtime
   |  |  |  +- org.codehaus.plexus:plexus-component-annotations:jar:1.5.5:runtime
   |  |  |  \- org.sonatype.plexus:plexus-sec-dispatcher:jar:1.3:runtime
   |  |  +- org.apache.maven:maven-model:jar:3.0.4:runtime
   |  |  +- org.apache.maven:maven-artifact:jar:3.0.4:runtime
   |  |  +- org.apache.maven:maven-aether-provider:jar:3.0.4:runtime
   |  |  +- org.apache.maven:maven-embedder:jar:3.0.4:runtime
   |  |  |  +- org.apache.maven:maven-compat:jar:3.0.4:runtime
   |  |  |  |  \- org.apache.maven.wagon:wagon-provider-api:jar:2.2:runtime
   |  |  |  \- org.sonatype.plexus:plexus-cipher:jar:1.7:runtime
   |  |  +- org.codehaus.plexus:plexus-container-default:jar:1.5.5:runtime
   |  |  |  \- org.apache.xbean:xbean-reflect:jar:3.4:runtime
   |  |  +- org.codehaus.plexus:plexus-classworlds:jar:2.4:runtime
   |  |  \- org.slf4j:slf4j-api:jar:1.7.32:runtime
   |  +- com.facebook.airlift:trace-token:jar:0.216:runtime
   |  +- io.airlift:joni:jar:2.1.5.3:runtime
   |  +- com.facebook.drift:drift-server:jar:1.40:runtime
   |  +- com.facebook.drift:drift-transport-netty:jar:1.40:runtime
   |  |  +- io.netty:netty-common:jar:4.1.115.Final:runtime
   |  |  +- io.netty:netty-buffer:jar:4.1.115.Final:runtime
   |  |  +- io.netty:netty-handler-proxy:jar:4.1.115.Final:runtime
   |  |  |  +- io.netty:netty-codec-socks:jar:4.1.115.Final:runtime
   |  |  |  \- io.netty:netty-codec-http:jar:4.1.115.Final:runtime
   |  |  +- io.netty:netty-handler:jar:4.1.115.Final:runtime
   |  |  |  \- io.netty:netty-transport-native-unix-common:jar:4.1.115.Final:runtime
   |  |  +- io.netty:netty-transport-classes-epoll:jar:4.1.115.Final:runtime
   |  |  +- io.netty:netty-transport-native-epoll:jar:linux-x86_64:4.1.115.Final:runtime
   |  |  \- io.netty:netty-codec:jar:4.1.115.Final:runtime
   |  +- com.facebook.drift:drift-transport-spi:jar:1.40:runtime
   |  +- com.facebook.drift:drift-client:jar:1.40:runtime
   |  +- com.facebook.drift:drift-protocol:jar:1.40:runtime
   |  +- com.facebook.drift:drift-codec:jar:1.40:runtime
   |  |  +- io.airlift:parameternames:jar:1.3:runtime
   |  |  \- com.facebook.airlift:bytecode:jar:1.3:runtime
   |  +- com.facebook.drift:drift-codec-utils:jar:1.40:runtime
   |  +- com.teradata:re2j-td:jar:1.4:runtime
   |  +- com.facebook.airlift.discovery:discovery-server:jar:1.33:runtime
   |  |  +- com.facebook.airlift:jmx-http-rpc:jar:0.198:runtime
   |  |  +- org.iq80.leveldb:leveldb-api:jar:0.10:runtime
   |  |  \- org.iq80.leveldb:leveldb:jar:0.10:runtime
   |  +- javax.servlet:javax.servlet-api:jar:3.1.0:runtime
   |  +- javax.ws.rs:javax.ws.rs-api:jar:2.1:runtime
   |  +- com.fasterxml.jackson.module:jackson-module-afterburner:jar:2.15.4:runtime
   |  +- com.jayway.jsonpath:json-path:jar:2.9.0:runtime
   |  |  \- net.minidev:json-smart:jar:2.5.0:runtime
   |  |     \- net.minidev:accessors-smart:jar:2.5.0:runtime
   |  +- org.sonatype.aether:aether-api:jar:1.13.1:runtime
   |  +- org.ow2.asm:asm:jar:9.2:runtime
   |  +- org.jgrapht:jgrapht-core:jar:1.3.1:runtime
   |  |  \- org.jheaps:jheaps:jar:0.10:runtime
   |  +- org.apache.lucene:lucene-analyzers-common:jar:8.10.0:runtime
   |  |  \- org.apache.lucene:lucene-core:jar:8.10.0:runtime
   |  +- org.locationtech.jts:jts-core:jar:1.19.0:runtime
   |  +- io.jsonwebtoken:jjwt-api:jar:0.11.5:runtime
   |  +- io.jsonwebtoken:jjwt-impl:jar:0.11.5:runtime
   |  +- io.jsonwebtoken:jjwt-jackson:jar:0.11.5:runtime
   |  +- org.apache.datasketches:datasketches-memory:jar:2.2.0:runtime
   |  +- org.apache.datasketches:datasketches-java:jar:5.0.1:runtime
   |  +- com.facebook.presto:presto-plugin-toolkit:jar:0.291-SNAPSHOT:runtime
   |  +- io.netty:netty-transport:jar:4.1.115.Final:runtime
   |  |  \- io.netty:netty-resolver:jar:4.1.115.Final:runtime
   |  \- com.facebook.presto:presto-ui:jar:0.291-SNAPSHOT:runtime
   +- com.facebook.presto:presto-expressions:jar:0.291-SNAPSHOT:runtime
   +- com.facebook.presto:presto-matching:jar:0.291-SNAPSHOT:runtime
   +- com.facebook.presto:presto-memory-context:jar:0.291-SNAPSHOT:runtime
   +- io.airlift:slice:jar:0.38:runtime
   +- io.airlift:units:jar:1.3:runtime
   +- com.facebook.airlift:stats:jar:0.216:runtime
   |  \- org.hdrhistogram:HdrHistogram:jar:2.1.9:runtime
   +- com.facebook.airlift:json:jar:0.216:runtime
   |  +- com.fasterxml.jackson.datatype:jackson-datatype-jdk8:jar:2.15.4:runtime
   |  +- com.fasterxml.jackson.datatype:jackson-datatype-jsr310:jar:2.15.4:runtime
   |  +- com.fasterxml.jackson.datatype:jackson-datatype-guava:jar:2.15.4:runtime
   |  +- com.fasterxml.jackson.datatype:jackson-datatype-joda:jar:2.15.4:runtime
   |  +- com.fasterxml.jackson.module:jackson-module-parameter-names:jar:2.15.4:runtime
   |  \- com.fasterxml.jackson.dataformat:jackson-dataformat-smile:jar:2.15.4:runtime
   +- com.facebook.airlift:http-client:jar:0.216:runtime
   |  +- ch.qos.logback:logback-core:jar:1.2.13:runtime
   |  +- org.eclipse.jetty:jetty-client:jar:9.4.56.v20240826:runtime
   |  +- org.eclipse.jetty:jetty-io:jar:9.4.56.v20240826:runtime
   |  +- org.eclipse.jetty:jetty-util:jar:9.4.56.v20240826:runtime
   |  +- org.eclipse.jetty:jetty-http:jar:9.4.56.v20240826:runtime
   |  +- org.eclipse.jetty.http2:http2-client:jar:9.4.56.v20240826:runtime
   |  |  +- org.eclipse.jetty.http2:http2-common:jar:9.4.56.v20240826:runtime
   |  |  |  \- org.eclipse.jetty.http2:http2-hpack:jar:9.4.56.v20240826:runtime
   |  |  \- org.eclipse.jetty:jetty-alpn-client:jar:9.4.56.v20240826:runtime
   |  +- org.eclipse.jetty.http2:http2-http-client-transport:jar:9.4.56.v20240826:runtime
   |  |  \- org.eclipse.jetty:jetty-alpn-openjdk8-client:jar:9.4.56.v20240826:runtime
   |  +- com.facebook.airlift:http-utils:jar:0.216:runtime
   |  \- net.jodah:failsafe:jar:2.0.1:runtime
   +- com.fasterxml.jackson.core:jackson-annotations:jar:2.15.4:runtime
   +- com.google.guava:guava:jar:32.1.0-jre:runtime
   |  +- com.google.guava:failureaccess:jar:1.0.1:runtime
   |  +- com.google.guava:listenablefuture:jar:9999.0-empty-to-avoid-conflict-with-guava:runtime
   |  +- org.checkerframework:checker-qual:jar:3.37.0:runtime
   |  +- com.google.errorprone:error_prone_annotations:jar:2.18.0:runtime
   |  \- com.google.j2objc:j2objc-annotations:jar:2.8:runtime
   +- com.google.inject:guice:jar:4.2.2:runtime
   |  \- aopalliance:aopalliance:jar:1.0:runtime
   +- javax.annotation:javax.annotation-api:jar:1.3.2:runtime
   +- javax.inject:javax.inject:jar:1:runtime
   +- com.facebook.airlift:concurrent:jar:0.216:runtime
   +- com.facebook.airlift:configuration:jar:0.216:runtime
   |  +- org.apache.bval:bval-jsr:jar:2.0.0:runtime
   |  \- cglib:cglib-nodep:jar:3.2.5:runtime
   +- com.facebook.airlift:bootstrap:jar:0.216:runtime
   +- com.facebook.airlift:node:jar:0.216:runtime
   +- com.facebook.airlift:log:jar:0.216:runtime
   +- com.facebook.airlift:log-manager:jar:0.216:runtime
   |  +- org.slf4j:slf4j-jdk14:jar:1.7.32:runtime
   |  +- org.slf4j:log4j-over-slf4j:jar:1.7.32:runtime
   |  \- org.slf4j:jcl-over-slf4j:jar:1.7.32:runtime
   +- com.google.code.findbugs:jsr305:jar:3.0.2:runtime
   +- org.pcollections:pcollections:jar:2.1.2:runtime
   +- org.weakref:jmxutils:jar:1.19:runtime
   +- org.openjdk.jol:jol-core:jar:0.2:runtime
   +- joda-time:joda-time:jar:2.12.7:runtime
   +- javax.validation:validation-api:jar:2.0.1.Final:runtime
   +- it.unimi.dsi:fastutil:jar:8.5.2:runtime
   \- org.apache.commons:commons-text:jar:1.10.0:runtime
      \- org.apache.commons:commons-lang3:jar:3.12.0:runtime

```  
  
</details>

By carefully limiting how and when a dependency is upgraded, we can ensure that Spark jobs using Presto are still runnable on older Spark clusters

For example, the Jetty dependency comes in via `presto-main`. If we can figure out a way to either  

1. Remove this dependency or   
1. Replace it with an equivalent implementation,  

we can migrate `presto-main` to a higher JDK version
In this particular case, we could identify that code in PoS that uses Jetty's HttpClient can be deprecated. See [TODO issue link]() for more details on this

For the modules that *cannot be upgraded* to a higher bytecode level, we can refactor code to generate smaller modules that can be kept at lower bytecode levels.  
This can prove to be beneficial for other modules too, e.g `presto-client` can be kept at JVM 8 bytecode level to allow for a broader support for Java based clients

## Timeline for the project

A detailed timeline is out-of-scope for this RFC.  
We will track project milestones and timeline on https://github.com/orgs/prestodb/projects/31/

We have the below high-level timeline targets - 
- Migrate Presto-On-Spark modules to JDK 17 starting early 2026

_Note: Active discussions on timelines are happening on the slack channel `#jdk21-upgrade`. Please join the [PrestoDB Slack](https://communityinviter.com/apps/prestodb/prestodb) to participate_

