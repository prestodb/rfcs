# **RFC0 for Presto**

## Function to aggregate IP prefixes.

Proposers
* Matt Calder (mattcalder-meta)

## Summary

A function `ip_prefix_agg` which accepts an `ARRAY` of `IPPREFIX`s, collapses them into their minimal representation, and returns an `ARRAY` of new `IPPREFIX`s.

Note: The term "aggregate" here is used in the networking sense -- to group adjacent IP address space. We are proposing a scalar operator on an array, not a SQL aggregation function.

## Background




## Proposed Implementation

1. Add a new method called `aggregateIpPrefixes` within `presto-main/src/main/java/com/facebook/presto/operator/scalar/IpPrefixFunctions.java`. This method will defined the SQL function called `ip_prefix_agg`.
1. `ip_prefix_agg` will operate on an `ARRAY` of `IPPREFIX`s.
1. All IP prefixes in the array must be the same IP version (i.e., no mix of IPv4 and IPv6) or an exception will be thrown.
1. To implement effectively, we need to work operate on ranges of integers. To support IPv6 we'll need 128-bit integers, which means using BigInteger (see Other Approaches). We will need some support functions to covert back and forth from `IPPREFIX` bytes to BigInteger but these are straight forward.
1. Tests will be added in `presto-main/src/test/java/com/facebook/presto/operator/scalar/TestIpPrefixFunctions.java`.

The algorithm will be similar to what is done in Python 3's `ip_network.collapse_addresses` function which performs the same job.
1. The input array of ip prefixes are sorted.
1. `IPPREFIX` objects are converted to integer ranges -- a tuple of the first IP address and last IP address.
1. Adjacent and overlapping ranges are merged (i.e. `[100, 249]` and `[250, 300] -> [100, 300]`). This only provides a range of consecutive IP addresses, but IP prefixes can only exist on specific integer boundaries.
1. We then split up the integer range into IP prefixes along correct address boundaries which involves a bit of math and bit fiddling.
1. We convert our integer ranges back into IPPREFIX objects that get returned.

## Other Approaches Considered
In theory, our implementation can be more efficient in terms of memory consumption and execution speed if we use native Java `long` for IPv4 prefixes using `BigInteger` for everything. These types don't share an interface, so it would mean duplicating core logic for IPv4 and IPv6.

## Adoption Plan

-

## Test Plan

We will have a unit tests, code coverage verification, and a comparison with outputs of Python's well-known `collapse_addresses` implementation.
