# **RFC0 for Presto**

## Function to check if IPADDRESS is private

Proposers
* Matt Calder (mattcalder-meta)

## Summary

Create `is_private_ip` UDF that checks whether an IP address is within private or reserved range that is not globally reachable. We take our definitions from what IANA considers not "globally reachable" in
https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml and
https://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xhtml.

## Background

When working with network data, filtering private IP address space is a common requirement. Without a standard function to handle it, it has led to numerous instances of:
* Incorrect/partial detection because of the scattered nature of Internet RFCs.
* Code duplication and general clutter to handle ~25 IP address block conditions.

## Proposed Implementation

This feature can be implemented in SQL, so we propose to:
1. Create a new `IPSqlFunctions.java` class under `presto-main/src/main/java/com/facebook/presto/operator/scalar/sql/`.
2. Implement a function called `is_private_ip`. The function will accept a single argument of type `IPADDRESS` that will return `true`
if the argument is a private IP address and `false` otherwise. The actual checks can be completed in two ways:
    * Some IPv4 private ranges align nicely with dotted-decimal string representation and it is more efficient to use a `STARTS_WITH` than to check if it is equal to a private `IPPREFIX`.
    * For other private ranges, we'll need to cast to an `IPPREFIX` and see if it equals one of the private ranges.
3. Add a new test class `presto-main/src/test/java/com/facebook/presto/operator/scalar/sql/TestIPSqlFunctions.java`.


## Other Approaches Considered

This function is more efficient to implement on integer representations of IP addresses, but we lack 128-bit integers.
We could write a new Java UDF that uses `BigInteger` but it seems like the SQL functions are preferred at this point.

## Adoption Plan

- Function documentation and release notes.

## Test Plan

Unit tests will check that the boundaries of all include private ranges return `true` while non-private ranges return `false`.
