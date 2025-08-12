# **RFC0005 for Presto - Presto API Spec**

Proposers

* Zac Blanco
* Tim Meehan

## [Related Issues]

- https://github.com/prestodb/presto/issues/22567
- https://github.com/prestodb/presto/pull/22982

## Summary

Currently, Presto's REST API and serializable objects sent between coordinator
and worker are sparsely documented, if documented at all. In order to improve
the adoption of Presto in the ecosystem and promote the availability of
third-party tools and implementations, we should create a well-defined
specification for Presto's REST API endpoints and serializable components.

Presto should declare all of its serializable components and API endpoints in a
well-defined specification language. The Java and C++ objects used in Presto's
implementation should be generated from this specification rather than be
written by hand or generated through a custom which attempts to parse java
source code[^1].

## Background

### Presto's Current Approach

Presto's REST API endpoints for interacting with the coordinator and workers is
not well-documented. We have a page for the basic REST API
endpoints[^2] in the
documentation. This covers the `/node`, `/query`, `/stage`, `/statement`, and
`/task` resources.

There are also pages which superficially cover the client[^3] and worker[^4]
interactions. None of these pages actually have well-defined examples of all
request and response parameters. It also doesn't describe fully when and how
each endpoint should be invoked.

This lack of documentation poses a few challenges for the Presto community as a
whole

1. Lack of documentation about architecture and endpoints can make it
more difficult to onboard new developers, slowing Presto development
velocity.
2. Lack of well-defined schemas for serializable objects contributes to a higher
chance of runtime bugs when implementing connector support Prestissimo.
3. Lack of documentation makes it more difficult to create clients or
third-party tools which connect to existing Presto endpoints.

### History of Presto's Endpoints and Serialization

Currently, Presto defines which objects are serializable through FasterXML's
Jackson serialization library[^5]. This library provides a set of annotations
which can be placed on Java class, member, method, constructor, and constructor
arguments. These annotations are interpreted by Jackson's serialization
framework to execute the proper methods for serialization and deserialization of
objects.

```java
@JsonCreator
public NamedTypeSignature(
        @JsonProperty("fieldName") Optional<RowFieldName> fieldName,
        @JsonProperty("typeSignature") TypeSignature typeSignature)
{ /* implementation */ }

@JsonProperty
public Optional<RowFieldName> getFieldName()
{
    return fieldName;
}

@JsonProperty
public TypeSignature getTypeSignature()
{
    return typeSignature;
}
```

There are classes and APIs defined with Jackson in the `presto-main` module.
Though not explicitly documented, connector implementations are required to
specify a number of SPI classes as JSON serializable in order for the
coordinator to send information to workers which is required to perform table
scans. Usually this information is limited to splits, and table reader and
writer metadata. However depending on the complexity of the connector (such as
hive) this info may extend to encryption or optimization metadata.

The original approach to serialization worked fine when Presto was written
solely in Java. However, with the advent of new systems such as Velox and
Prestissimo in C++ as well as projects such as substrate[^6] and a desire to
make Presto appealing to a wider variety of users and external systems it would
be beneficial to make some of the internal components of Presto more accessible
such that others users and systems can more easily develop their systems to
connect to Presto.

Converting endpoints and serializable structures to C++ has posed challenges
around maintaining synchronization between Java and C++ compatibility as well as
integrating other execution environments. A specification with code generation
capabilities would be highly beneficial to the community and for enabling more
rapid development.

### Goals

In summary, Presto should:

1. Make all structs/classes which are serialized between coordinator and workers
both inside of `presto-main` and in connectors defined by a specification
language
2. Define all API endpoints in a specification language
3. Publish all specifications for the entire system for each release
4. Utilize code generation capabilities of the specification language to
generate Java, C++, and any other code required for Presto to function.

### A Proposal to Utilize OpenAPI Specification for Presto

Presto can achieve goals (1)-(4) declared above in a phased approach utilizing
the OpenAPI specification language. An initial prototype within a specification
for a portion of the API surface and struct definitions was merged in PR
22982[^7].

OpenAPI is a mature specification language used by a wide variety of companies
for their APIs. It is a well defined language that has a number of contributors.
There are also numerous implementations which provide code generation
capabilities. One of the more well known tools Swagger Codegen[^8]. More
recently a fork of the Swagger codegen tool has been created, now known as
OpenAPI-Generator[^9]. Both provide support for generating structs, clients, and
servers from OpenAPI specifications.

The original attempt in PR #22982 chose to use OpenAPI-Generator for it's more
active community and wider support of different languages and frameworks.
Additionally OpenAPI-Generator has support for custom codegen templates.

The rest of this section details each phase for achieving full specification of
Presto

#### Phase 1 - Spec Documentation

In the first phase of specification, we should make a best effort attempt to
specify and document the entirety of the Presto internal and external endpoints
and all structs. After documenting, we should publish and make available the
spec and a generated HTML page for the entirety of the spec within the Presto
documentation.

Since the entire Presto API and struct surface is large, this allows us to
complete a large chunk of the work without being bound by having 100%
correctness. Work can be parallelized and regular PR review should take place to
catch as many errors as possible.

This method will likely introduce some human-error in the spec translation. This
should be okay. It will be fixed as we transition portions of the code base to
utilize spec-generated code in later phases.

#### Phase 2 - `presto-main` Struct Codegen

In this phase, we should integrate the codegen for all structs from the
`presto-main` module in the specification into the build process.

This would involve removing all existing Java classes which are checked source
that exist in the spec. We would create a new maven module `presto-api` which
contains all objects and interfaces generated from the spec. Specifically in
this phase, Java classes for each object in the spec would then be generated in
this maven module. Generated files will be stored in the maven target output
directory under `generated-sources` similar to how we have generated file
for the `presto-parquet` module. These generated files would not be checked into
git.

One downside to this approach Code generation would be required all clean
builds, which potentially could down the compilation process. However, this
prevents the complicated and tricky task of keeping spec and source in-sync. An
effort should be made to focus on optimizing the build time of the `presto-api`
module.

Additionally, this phase would involve removing all generated C++ code from
`presto-native-execution/presto_protocol` and require generating the relevant
C++ code during the maven `generate-sources` phase of either the
`presto-openapi` module or during the `presto-native-execution` module.

Success in this phase would mean removing all Java structures and C++ generated
source code from the repository and relying solely on the API specification for
code generation.

Depending on the scope, it may be desirable to split this part into two phases
2a and 2b for removing Java and C++ source code respectively.

#### Phase 3 - Connector Struct Codegen

In this phase, we would focus on creating spec components for each connector
and implementing code generation parts of the build pipeline for the connectors.

One of the neat parts about the Java-based Presto API is that when a connector
is loaded at runtime, the relevant serializable classes for the connector such
as `ColumnHandle` and `TableHandle` are automatically registered. This is
convenient in Java because serialization is done implicitly without having to
specify any classes to the server or serialization framework. However, when
integrating with Prestissimo we need to know all of the serializable classes for
each connector, which then requires us to rely on the `presto_protocol`
framework.

It will be easier for future development if the connectors specify all of their
serializable components through an OpenAPI spec so that we can generate the
relevant code in Java and C++. It makes it clearer to the developers and
prevents having to rely on the existing `presto_protocol` framework.

#### Phase 4 - `presto-main` Server Endpoint Codegen

In this phase, the focus is on using the spec to generate interfaces which
represent the current HTTP API surface for the server side of the Presto
coordinator and workers. Ideally, most of the code should just be "lift and
shift" without much editing. 

`presto-openapi` will generate an interface which represents the HTTP API. Then,
the existing implementation will be shifted to satisfy the interface. The
OpenAPI-generator has a number of existing generators which utilize JAX-RS
annotations[^10] which is what Presto already uses for the servers. So we
shouldn't need to make major modifications to the existing code to satisfy the
interface from the OpenAPI-generator.

#### Phase 5 - `presto-main` Client Endpoint Codegen

In this phase, we similarly generate interfaces and shift the existing Presto
client code to satisfy the new interfaces.

### Foreseen Challenges

In order to reduce the likelihood of breaking compatibility or reducing
performance we will likely need to write some of our own code generation
templates for the OpenAPI-generator in order to generate code which is
compatible with the existing code.

For basic structures and interfaces, there shouldn't be many issues. However,
there may be compatibility challenges implementing existing polymorphic
structures. One example of a polymorphic structure is the `ValueSet` type:

```java
@JsonTypeInfo(
        use = JsonTypeInfo.Id.NAME,
        include = JsonTypeInfo.As.PROPERTY,
        property = "@type")
@JsonSubTypes({
        @JsonSubTypes.Type(value = EquatableValueSet.class, name = "equatable"),
        @JsonSubTypes.Type(value = SortedRangeSet.class, name = "sortable"),
        @JsonSubTypes.Type(value = AllOrNoneValueSet.class, name = "allOrNone")})
public interface ValueSet
```

Some of the code generators don't support these types of structures, so we may
either need to (1) define them differently in the spec, or (2) use a custom
codegen template, or (3) implement support in the generator.


## [Optional] Other Approaches Considered

- TBD

## Test Plan

Through each stage we will ensure compatibility by making sure all existing
tests pass. After each phase, we should do a performance comparison benchmark
with the newly generated code to make sure there are no performance degredations
due to new serialization code or indirection in server of client interfaces.

## References

[^1]: https://github.com/prestodb/presto/blob/ddc33d6ab6ec78688e1c10e8b0c01dfebb024165/presto-native-execution/presto_cpp/presto_protocol/java-to-struct-json.py
[^2]: https://prestodb.github.io/docs/current/rest.html
[^3]: https://prestodb.github.io/docs/current/develop/client-protocol.html
[^4]: https://prestodb.github.io/docs/current/develop/worker-protocol.html
[^5]: https://github.com/FasterXML/jackson
[^6]: https://substrait.io/
[^7]: https://github.com/prestodb/presto/pull/22982
[^8]: https://github.com/swagger-api/swagger-codegen
[^9]: https://github.com/OpenAPITools/openapi-generator
[^10]: https://openapi-generator.tech/docs/generators/