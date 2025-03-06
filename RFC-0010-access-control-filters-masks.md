# **RFC-0010 for Presto**

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions on creating your RFC and the process surrounding it.

## [Access Control Row Filters and Column Masks]

Proposers

* Tim Meehan
* Bryan Cutler

## [Related Issues]

Current Proposal <br>
https://github.com/prestodb/presto/issues/24278

Past Discussions and PRs
<br>
https://github.com/prestodb/presto/issues/20572
<br>
https://github.com/prestodb/presto/issues/19041
<br>
https://github.com/prestodb/presto/pull/21913
<br>
https://github.com/prestodb/presto/pull/18119

## Summary

Add access control support for row filtering and column masking, and apply them to queries by rewriting the plan.

## Background

As a part of governance requirements, Presto is needed to support data compliance coming from a set of defined rules. For a
given query, these rules are made into expressions for row filters and column masks. A row filter is used to prevent display of
certain rows with data the user does not have access to, while allowing remaining rows that are allowed. A column mask can be
used to mask or obfuscate sensitive data that a user is forbidden to view, such as a credit card number. The expressions can
then be applied to the query during a rewrite before it is run.

### [Optional] Goals

* Add SPIs to AccessControl classes to allow retrieval of row filters and column masks
* Add functionality for Presto to apply filters and masks to a query

## Proposed Implementation

The proposed implementation follows the design from TrinoDB. The filters and masks are retrieved in the `StatementAnalyzer`
and the query is rewritten with the `RelationPlanner`. For reference, orginal TrinoDB commits:

* Adding support for row filters trinodb/trino@fae3147
* Adding support for column masking trinodb/trino@7e0d88e

### Core SPI

Add methods to access control interfaces to retrieve a list of row filters and columns masks for all relevant columns in a
table.

AccessControl.java

```java
    default List<ViewExpression> getRowFilters(TransactionId transactionId, Identity identity, AccessControlContext context, QualifiedObjectName tableName)
    {
        return Collections.emptyList();
    }

    default Map<ColumnMetadata, ViewExpression> getColumnMasks(TransactionId transactionId, Identity identity, AccessControlContext context, QualifiedObjectName tableName, List<ColumnMetadata> columns)
    {
        return Collections.emptyMap();
    }
```

ConnectorAccessControl.java

```java
    /**
     * Get row filters associated with the given table and identity.
     * <p>
     * Each filter must be a scalar SQL expression of boolean type over the columns in the table.
     *
     * @return the list of filters, or empty list if not applicable
     */
    default List<ViewExpression> getRowFilters(ConnectorTransactionHandle transactionHandle, ConnectorIdentity identity, AccessControlContext context, SchemaTableName tableName)
    {
        return Collections.emptyList();
    }

    /**
     * Bulk method for getting column masks for a subset of columns in a table.
     * <p>
     * Each mask must be a scalar SQL expression of a type coercible to the type of the column being masked. The expression
     * must be written in terms of columns in the table.
     *
     * @return a mapping from columns to masks, or an empty map if not applicable. The keys of the return Map are a subset of {@code columns}.
     */
    default Map<ColumnMetadata, ViewExpression> getColumnMasks(ConnectorTransactionHandle transactionHandle, ConnectorIdentity identity, AccessControlContext context, SchemaTableName tableName, List<ColumnMetadata> columns)
    {
        return Collections.emptyMap();
    }
```

SystemAccessControl.java
```java

    /**
     * Get row filters associated with the given table and identity.
     * <p>
     * Each filter must be a scalar SQL expression of boolean type over the columns in the table.
     *
     * @return a list of filters, or empty list if not applicable
     */
    default List<ViewExpression> getRowFilters(Identity identity, AccessControlContext context, CatalogSchemaTableName tableName)
    {
        return Collections.emptyList();
    }

    /**
     * Bulk method for getting column masks for a subset of columns in a table.
     * <p>
     * Each mask must be a scalar SQL expression of a type coercible to the type of the column being masked. The expression
     * must be written in terms of columns in the table.
     *
     * @return a mapping from columns to masks, or an empty map if not applicable. The keys of the return Map are a subset of {@code columns}.
     */
    default Map<ColumnMetadata, ViewExpression> getColumnMasks(Identity identity, AccessControlContext context, CatalogSchemaTableName tableName, List<ColumnMetadata> columns)
    {
        return Collections.emptyMap();
    }
```

ViewExpression class to hold a filter/mask expression

```java
public ViewExpression(String identity, Optional<String> catalog, Optional<String> schema, String expression)
```

Analysis.java will hold filters and masks for the table with additional methods

```java
void registerTableForRowFiltering(QualifiedObjectName table, String identity)

boolean hasRowFilter(QualifiedObjectName table, String identity)

void addRowFilter(Table table, Expression filter)

List<Expression> getRowFilters(Table node)

void registerTableForColumnMasking(QualifiedObjectName table, String column, String identity)

boolean hasColumnMask(QualifiedObjectName table, String column, String identity)

void addColumnMask(Table table, String column, Expression mask)

Map<String, Expression> getColumnMasks(Table table)
```
#### Example expressions

Examples of row filter expressions, given a table `orders` with columns `orderkey`, `nationkey`:

- a simple predicate:
```
expression := "orderkey < 10"
```

- a subquery:
```
expression := "EXISTS (SELECT 1 FROM nation WHERE nationkey = orderkey)"
```

A column mask will apply an operation on a specific column, given the column values as input produce the masked output.
- example to nullify values, whatever column this is applied to will produce a NULL value:
```
expression := "NULL"
```

- example to negate a column integer values, when applied to column "custkey":
```
expression := "-custkey"
```

#### Additional information
1. What modules are involved
    - `presto-main`
    - `presto-spi`
    - `presto-analyzer` to hold masks and filters
    - `presto-hive` for legacy and sql access control
2. Any new terminologies/concepts/SQL language additions
    - NA
3. Method/class/interface contracts which you deem fit for implementation.
    - NA
4. Code flow using bullet points or pseudo code as applicable
    - During analysis phase, access control apis used to retrieve and analyze row filters and column masks.
    - Analyzed filters and masks are stored in `Analysis`.
    - `RelationPlanner` will the get the filters and masks from `Analysis` and rewrite a new plan with them applied.
    - By default no filters or masks are added and the plan will not be rewritten.
5. Any new user facing metrics that can be shown on CLI or UI.
    - NA

### Notes on Table Names with Versioning

The proposed SPI will identify a table resource as a `QualifiedObjectName` that includes
* Catalog name
* Schema name
* Table name

This does not explicitly provide table version information when a connector in use supports versioning.
For now, it is left to the plugin implementation to handle any additional versioning added to the table
name. It is recommended to further discuss the possibility of adding such information to `QualifiedObjectName`
so that the plugin can easily be aware of any table versioning, or schema evolution, when providing
row filters or column masks.

## [Optional] Metrics

This is a 0 to 1 feature and will not have any metrics.

## [Optional] Other Approaches Considered

Discussed at https://github.com/prestodb/presto/pull/21913#issuecomment-2050279419 is an approach to use the existing SPI for connector
optimization to rewrite plan during the optimization phase. The benefits of the proposed design over this approach is that it applies
globally to all connectors. Since it is an existing design that has already been in use, it is known to be working, stable and
conforms with the Trino SPI which will help to ease migration.

## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
  - No impact to users. SPI additions will include a default to keep exiting behaviour. AccessControl plugin can be used to enable the functionality.
- If we are changing behaviour how will we phase out the older behaviour?
   - NA
- If we need special migration tools, describe them here.
  - NA
- When will we remove the existing behaviour, if applicable.
  - NA
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
  - This feature will be documented in the Presto documentation.
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
  - NA

## Test Plan

Unit tests will be added to ensure that row filter and column mask expressions can be added to a query and give the expected result. The
`TestingAccessControlManager` will be modified to allow for addition of row filters and column masks to be used in testing.