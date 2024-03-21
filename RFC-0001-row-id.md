# **RFC 2 for Presto**

## Hive Row IDs

Proposers

* Brian Dahmen
* Devang Shah
* Elliotte Rusty Harold
* Serge Druzkin
* Sreeni Viswanadha

## Related Issues

* Row ID support #22078
* Binary IDs for Hive partitions #22103
* Binary rowID type #22053
* Start adding row\_id hidden column to Hive #22008


## Summary

A row ID is a binary key for a row that is globally unique across the warehouse.
It identifies a specific row at a specific version of a partition’s data. A row
ID will continue to locate the same row in the warehouse until the underlying
partition is mutated.

## Background

Queries can produce lists of row IDs that can be saved and used as input to
future queries for quick selection. A list of row IDs can be used to include or
exclude rows in way that identifies a subset of a table.

### Goals

Row IDs resolve to the same value across compute engines. In this way they can
be used for strong-equality comparison across compute engines.

### Non-goals

Comparison and sorting of row IDs.

## Proposed Implementation

A row ID is a VARBINARY pseudo-column, also known as a hidden column. Each row
will have a unique value that can be compared for equality, byte by byte. The
type is not orderable. In SQL queries this will have the name `$row_id` and
the type `VARBINARY`. In Java it will have the type `SQLVarBinary`.
The complete row ID will have two components,
the first to identify a row within a partition and the second to identify the
partition within the warehouse.

The exact format of the row ID is connector specific and not constrained by this
RFC. It might change from one warehouse to the next if different warehouses use
different metatdata servers to supply the partition half of a row ID.

The metadata server (for example, Hive Metastore) supplies the partition component
in the same way it supplies all the other information in the `Partition` object.
Typically this involves a network call to the server which returns a Thrift
object. The fields of the Thrift object are deserialized and loaded into a
`Partition` object. One of these fields is a byte array containing the
"row ID partition component". This byte array will be passed down the chain into a
`HiveSplit` and eventually into a `PageSource`.

If the SQL query explicitly references the `$row_id`
pseudo-column, when a page source reads rows from a file, it sets `appendRowNumber` to true 
so it receives row numbers from the reader. Then the page source coerces the 
row number block (type `long`) into a row ID block (type `SqlVarBinary`) by concatenating
the bytes of the row number, the row group ID, and the partition ID into a single 
blob.

## Adoption Plan

This should be completely ignorable for all existing clients that use Presto.
Everything is API compatible, and no table or SQL query should have a column named
`$row_id` (or anything else that begins with a dollar sign that is not a documented
Presto pseudo-column).

## Test Plan

Tests will be added to the existing Hive integration tests that select the 
`$row_id` column.
