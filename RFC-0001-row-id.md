# **RFC 1 for Presto**

## Row ID

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
type is not orderable. In SQL queries this will have the name `$row_id` and the
type ROW_ID, a new built-in type. The complete row ID will have two components,
the first to identify a row within a partition and the second to identify the
partition within the warehouse.

The exact format of the row ID is connector specific and not constrained by this
RFC. It might change from one warehouse to the next if different warehouses use
different metatdata servers to supply the partition half of a row ID.

The metadata server (for example, Hive Metastore) supplies the partition component in
the same way it supplies all the other information in the `Partition` object.
Typically this involves a network call to the server which returns a Thrift
object. The fields of the Thrift object are deserialized and loaded into a
`Partition` object. One of these fields is a byte array containing the
"row ID partition component".

When a connector reads rows from a file, it passes this partition component to
the reader. Assuming the SQL query explicitly references the `$row_id`
pseudo-column, the reader constructs a unique row component for each row from
the row number and the row group. It concatenates this row component with the
partition component and returns the combined value as the `$row_id` field.
Possibly instead of passing the partition component directly, it passes a function
the reader can use to construct a row ID from the row number and the row group.


## Adoption Plan

This should be completely ignorable for all existing clients that use Presto.
Everything is API compatible, and no table or SQL query should have a column named
`$row_id` (or anything else that begins with a dollar sign).

## Test Plan

Tests will be added to the existing Hive integration tests that select the 
`$row_id` column.
