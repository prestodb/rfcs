# **RFC-0014 for Presto**

## Mixed case identifiers

Proposers

* Reetika Agrawal

## Summary

Improve Presto's identifier (schema, table & column names) handling to align with SQL standards, ensuring better interoperability with case-sensitive
and case-normalizing databases while minimizing SPI-breaking changes.

## Background

Presto treats all identifiers as case-insensitive, normalizing them to lowercase. This creates issues when
querying databases that are case-sensitive (e.g., MySQL, PostgreSQL) or case-normalizing to uppercase (e.g., Oracle,
DB2). Without a standard approach, identifiers might not match the actual names in the underlying data sources, leading
to unexpected query failures or incorrect results.

The goal here is to improve interoperability with storage engines by aligning identifier handling with SQL standards
while ensuring a seamless user experience. Ideally, the change should be implemented in a way that minimizes
breaking changes to the SPI, i.e. allowing connectors to adopt the new approach without significant impact.

### Goals

- Align Presto’s identifier handling with SQL standards to improve interoperability with case-sensitive and
  case-normalizing databases.
- Minimize SPI-breaking changes to maintain backward compatibility for existing connectors.
- Introduce a mechanism for connectors to define their own identifier normalization behavior.
- Allow identifiers to retain their original case where necessary, preventing unexpected query failures.
- Ensure Access Control SPI can correctly normalize identifiers.
- Preserve a seamless user experience while making these changes.

### Proposed Plan

Presto's default behavior is -

- Identifiers are converted to lowercase by default unless a connector enforces a specific behavior.
- Identifiers are normalized when:
  - Resolving schemas, tables, columns, views.
  - Retrieving metadata from connectors.
  - Displaying entity names in metadata introspection commands like SHOW TABLES and DESCRIBE.

Presto uses identifiers in several ways:

  - Matching identifiers to locate entities such as catalogs, schemas, tables, views.
  - Resolving column names based on table metadata provided by connectors.
  - Passing identifiers to connectors when creating new entities.
  - Processing and displaying entity names retrieved from connectors, including column resolution and metadata introspection commands like SHOW and DESCRIBE.

## Proposed Implementation

#### Core Changes

* In the presto-spi, add new API to pass original identifier (Schema, table and column names)
* Introduce new API in Metadata for preserving lower case identifier by default to preserve backward compatibility
* Introduce new Connector specific API in ConnectorMetadata

Metadata.java

```java
    String normalizeIdentifier(Session session, String catalogName, String identifier);
```

MetadataManager.java

```java
    @Override
    public String normalizeIdentifier(Session session, String catalogName, String identifier)
    {
        Optional<CatalogMetadata> catalogMetadata = getOptionalCatalogMetadata(session, transactionManager, catalogName);
        if (catalogMetadata.isPresent()) {
            ConnectorId connectorId = catalogMetadata.get().getConnectorId();
            ConnectorMetadata metadata = catalogMetadata.get().getMetadataFor(connectorId);
            return metadata.normalizeIdentifier(session.toConnectorSession(connectorId), identifier);
        }
        return identifier.toLowerCase(ENGLISH);
    }
```

ConnectorMetadata.java
```java

    /**
     * Normalize the provided SQL identifier according to connector-specific rules
    */
    default String normalizeIdentifier(ConnectorSession session, String identifier)
    {
        return identifier.toLowerCase(ENGLISH);
    }
```

JDBC Connector specific implementation

JdbcMetadata.java

```java
    @Override
    public String normalizeIdentifier(ConnectorSession session, String identifier)
    {
        return jdbcClient.normalizeIdentifier(session, identifier);
    }
```

JdbcClient.java
```java
    String normalizeIdentifier(ConnectorSession session, String identifier);
```

BaseJdbcClient.java
```java
    @Override
    public String normalizeIdentifier(ConnectorSession session, String identifier)
    {
        return identifier.toLowerCase(ENGLISH);
    }
```

Example - Connector specific implementation -
MySqlClient.java

```java
    @Override
    public String normalizeIdentifier(ConnectorSession session, String identifier)
    {
        return identifier;
    }
```

#### Example Queries

#### MySQL Table Handling

```
presto> show schemas from mysql;
       Schema       
--------------------
 Test               
 TestDb             
 information_schema 
 performance_schema 
 sys                
 testdb             
(6 rows)

presto> show tables from mysql.TestDb;
   Table   
-----------
 Test      
 TestTable 
 testtable 
(3 rows)

presto> SHOW CREATE TABLE mysql.TestDb.Test;
             Create Table             
--------------------------------------
 CREATE TABLE mysql."TestDb"."Test" ( 
    "id" integer,                     
    "Name" char(10)                   
 )                                    
(1 row)

presto> select * from mysql.TestDb.Test;
 id |    Name    
----+------------
  2 | Tom        
(1 row)
```

## Behavioral Examples with `case-sensitive-name-matching` Flag

Presto will allow the connector to handle identifier normalization if the `case-sensitive-name-matching` configuration flag is
supported by the connector. For example, if the Postgres connector does not normalize identifiers to lowercase, the
original case from the Presto DDL is preserved — including for unquoted identifiers.

**When case-sensitive-name-matching = false (default behavior)**
This is the default behavior in Presto. Identifiers are normalized to lowercase, regardless of quoting.

Presto DDL:

```sql
CREATE TABLE TeSt1 (
    ID INT,
    "Name" VARCHAR
);
```

Underlying DDL sent to Postgres, since Postgres identifierQuote is double quotes:

```sql
CREATE TABLE "test1" (
    "id" INT,
    "name" VARCHAR
);
```

* Table and column names are normalized to lowercase.
* Quoting is added as needed by the connector, but the case is not preserved.

**When case-sensitive-name-matching = true**

Connector is responsible for identifier normalization, allowing case preservation or other casing.

```sql
CREATE TABLE test1 (
    UPR INTEGER,
    lwr INTEGER,
    "Mixed" INTEGER
);
```

Resulting Postgres DDL:

```sql
CREATE TABLE "test1" (
    "UPR" INTEGER,
    "lwr" INTEGER,
    "Mixed" INTEGER
);

```

* Table name is preserved as "test1" (unquoted input becomes quoted).
* Column names retain their original case — whether quoted or unquoted.
* This behavior aligns with SQL standard semantics and matches user intent.

If users prefer lowercase identifiers, they can write:

Presto DDL:

```sql
CREATE TABLE test1 (
    upr INTEGER,
    lwr INTEGER,
    mixed INTEGER
);
```

Underlying DDL sent to Postgres:

```sql
CREATE TABLE "test1" (
    "upr" INTEGER,
    "lwr" INTEGER,
    "mixed" INTEGER
);
```
### Rationale

This behavior gives users full control over identifier casing, matching SQL standard semantics and improving
compatibility with case-sensitive engines like Postgres. It also ensures a smooth migration path by defaulting to
existing behavior (case-sensitive-name-matching = false), avoiding surprises for current users.

## Backward Compatibility Considerations

* Existing connectors that do not implement normalizeIdentifier will default to lowercase normalization.
* Any connectors requiring case preservation can override the default behavior.
* A configuration flag could be introduced to allow backward-compatible identifier handling at the catalog level.

## Test Plan

* Ensure that existing CI tests pass for connectors where no specific implementation is added.
* Add unit tests for testing mixed-case identifiers support in a JDBC connector (e.g., MySQL, PostgreSQL).
* Cover cases such as:
  - Queries with mixed-case identifiers.
  - Metadata retrieval commands (SHOW SCHEMAS, SHOW TABLES, DESCRIBE).
  - Joins, subqueries, and alias usage with mixed-case identifiers.

To ensure backward-compatibility current connectors where connector specific implementation is not added, existing CI tests should pass.
Add support for mixed case for a JDBC connector (ex. mysql, postgresql etc) and add relevant Unit tests for same.

## Modules involved
- `presto-main`
- `presto-common`
- `presto-spi`
- `presto-parser`
- `presto-base-jdbc`

## Final Thoughts

This RFC enhances Presto's identifier handling for improved cross-engine compatibility. The proposed changes ensure
better adherence to SQL standards while maintaining backward compatibility. Implementing connector-specific identifier
normalization will help prevent unexpected query failures and improve user experience when working with different
databases.
Would appreciate feedback on any additional cases or edge scenarios that should be covered!

## WIP - Draft PR Changes
https://github.com/prestodb/presto/pull/24551
