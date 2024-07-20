# **RFC0 for Presto**

## Function to aggregate IP prefixes.

Proposers
* Matt Calder (mattcalder-meta)

## Summary

A function `ip_prefix_agg` which accepts an `ARRAY` of `IPPREFIX`s, collapses them into their minimal representation, and returns an `ARRAY` of new `IPPREFIX`s.

Note: The term "aggregate" here is used in the networking sense -- to group adjacent IP address space. We are proposing a scalar operator on an array, not a SQL aggregation function.

## Background
An IP prefix is a notation for representing a *network*, or group of IP addresses. For example, a */24*, such as 192.0.1.0/24, is 256 IP addresses in the range of 192.0.1.0 to 192.168.1.255. The /24 specifies a *prefix length* of 24-bits. To be a valid IP prefix, the prefix length part must fall on a valid IP address boundary. For example, 192.168.5.7/24 is not valid. This is enforced by the existing `IPPREFIX` type.

In the network data world it is common to group blocks of adjacent address space into a more compact representation. As a trivial example, 192.0.2.0/25, 192.0.2.128/25 can be aggregated to 192.0.2.0/24. However, we cannot just merge arbitrary neighboring IP blocks with the same prefix lengths because the starting prefix may not fall on a valid network boundary. For example, 192.168.1.0/24 and 192.168.2.0/24 cannot be aggregated to 192.168.1.0/23.

## Motivation
This above example is simple but in reality we cannot accomplish this with a `REDUCE` for arbitrary sets of IP addresses. This is primarily because the set of required merges cannot be computed in a single iteration. Network data practictioners often need to aggregate sets of prefixes based on things like all the prefixes in a geographical area within the same ASN, for example, Comcast [https://bgp.he.net/AS7922#_prefixes](https://bgp.he.net/AS7922#_prefixes).

Take this example of sorted input prefixes:
* 192.168.1.0/26
* 192.168.1.64/26
* 192.168.4.0/24
* 192.168.8.0/24
* 192.168.9.0/25
* 192.168.9.128/25
* 192.168.10.0/23

The correct aggregated output set is: 192.168.1.0/25, 192.168.4.0/24, 192.168.8.0/22.

Iteratively, here is how we get there:

**Pass 1:**
1. 192.168.1.0/26 and 192.168.1.64/26 can get merged into a /25 because they are the same prefix length and adjacent in the IP address space.
2. 192.168.4.0/24 has nothing to be merged with.
3. 192.168.8.0/24 has nothing to be merged with.
4. 192.168.9.0/25 and 192.168.9.128/25 can be merged into a /24.
5. 192.168.10.0/23 has nothing to be merged with.

**Result after pass 1:**
* 192.168.1.0/25
* 192.168.4.0/24
* 192.168.8.0/24
* 192.168.9.0/24
* 192.168.10.0/23

**Pass 2:**
1. 92.168.1.0/25 has nothing to merge with.
2. 192.168.4.0/24 has nothing to merge with.
3. 192.168.8.0/24 and 192.168.9.0/24 can get merged into a /23.
4. 192.168.10.0/23 can be merged with the previous /23 to create a /22 assuming you have state and can look back. Otherwise, requires a third pass.

**Result after pass2:**
* 92.168.1.0/25
* 192.168.4.0/24
* 192.168.8.0/22

In practice the set of input prefixes is arbitrary so the set prefix lengths are unknown before hand. The general solution in SQL for IPv4 will require 32 queries/nested reduces since there are 32 possible prefix lengths (i.e., IPv4 is 32-bit -- one for each bit). IPv6 -- with 128 bits, will of course require more. Technically, you can emit a new table for each prefix length and run a chain of 32 or 128 nested queries but this is terrible. The iterative approach is simple, runs in linear time, and intuitive which is why a UDF solution is desirable.

## Proposed Implementation

1. Add a new method called `aggregateIpPrefixes` within `presto-main/src/main/java/com/facebook/presto/operator/scalar/IpPrefixFunctions.java`. This method will defined the SQL function called `ip_prefix_agg`.
1. `ip_prefix_agg` will operate on an `ARRAY` of `IPPREFIX`s.
1. All IP prefixes in the array must be the same IP version (i.e., no mix of IPv4 and IPv6) or an exception will be thrown.
1. To implement effectively, we need to work operate on ranges of integers. To support IPv6 we'll need 128-bit integers, which means using BigInteger (see Other Approaches). We will need some support functions to covert back and forth from `IPPREFIX` bytes to BigInteger but these are straight forward.
1. Tests will be added in `presto-main/src/test/java/com/facebook/presto/operator/scalar/TestIpPrefixFunctions.java`.

The algorithm will be similar to what is done in Python 3's `ipaddress.collapse_addresses` function which performs the same job.
1. The input array of ip prefixes are sorted.
1. `IPPREFIX` objects are converted to integer ranges -- a tuple of the first IP address and last IP address.
1. Adjacent and overlapping ranges are merged (i.e. `[100, 249]` and `[250, 300] -> [100, 300]`). This only provides a range of consecutive IP addresses, but IP prefixes can only exist on specific integer boundaries.
1. We then split up the integer range into IP prefixes along correct address boundaries which involves a bit of math and bit fiddling.
1. We convert our integer ranges back into IPPREFIX objects that get returned.

## Other Approaches Considered
In theory, our implementation can be more efficient in terms of memory consumption and execution speed if we use native Java `long` for IPv4 prefixes using `BigInteger` for everything. These types don't share an interface, so it would mean duplicating core logic for IPv4 and IPv6.

## Adoption Plan
- Documentation and release notes

## Test Plan

We will have a unit tests, code coverage verification, and a comparison with outputs of Python's well-known `collapse_addresses` implementation.
