# **RFC11 for Dynamic Catalog Modification**

## Dynamic Catalog Endpoints

Proposers

* Hazmi

## [Related Issues]

https://github.com/prestodb/presto/issues/2445
https://github.com/prestodb/presto/pull/24587
https://github.com/prestodb/presto/pull/12605

## Summary

This proposed change adds the ability to load, delete & update catalogs without restarting the cluster

## Background

Previously, we had a PR with the implementation for this feature. After a TSC meeting discussion, the following concerns
were brought up:
- Native support for this feature
- Syncing new configs to workers without requiring a script to run parallely on each worker to apply the catalog changes individually
- Ideally we should persist the configs to on-disk - this will solve the problem of how configs persist on coordinator/worker restart


## Proposed Implementation

Currently, on startup, Presto will create an instance of a StaticCatalogStore. Then, the StaticCatalogStore's loadCatalog function is called.
This will then load the catalog files located in the catalog configuration directory. 
Then, for each catalog, the ConnectorManager will have a CatalogManager that contains a Map of connectors.

Our proposal has two goals:
1. Add an option of having catalogs stored remotely as opposed to locally.
2. Add a way to interact with the external store via Presto

### Dynamic Catalog Store

We can add another CatalogStore type to that retrieves catalog info from an external database such as etcd or Zookeeper.
Both etcd and Zookeeper contain a watch query to listen for any changes. In a cluster, the resource manager
can send a watch request and wait until any changes occur. Once it receives that request, it can inform all workers
to refresh their catalog stores. Workers then retrieve the latest catalog info and send their results back to the resource manager.
Once the resource manager verifies that all workers retrieved the same information, it can send a signal to replace their catalogs.
This will ensure that the catalog info is consistent across all workers.

### External Store API

We can add several REST endpoints to interact with the external store.
Since the database is standalone, this is not required to dynamically modify catalogs.


## Adoption Plan

- What impact (if any) will there be on existing users? Are there any new session parameters, configurations, SPI updates, client API updates, or SQL grammar?
    - We will add a new configuration to either enable the external store or stick to the old StaticCatalogStore implementation.
    - We will add a new CatalogStore interface & have the old StaticCatalogStore implement it, along with the new DynamicCatalogStore.
    - We might add new REST endpoints to interact with the external catalog database if the dynamic catalog store is enabled.
- If we are changing behaviour how will we phase out the older behaviour?
    - Old behaviour will remain
- If we need special migration tools, describe them here.
    - N/A
- When will we remove the existing behaviour, if applicable.
    - N/A
- How should this feature be taught to new and existing users? Basically mention if documentation changes/new blog are needed?
    - A new page in the documentation will explain the available APIs
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
    - N/A

## Test Plan

How do we ensure the feature works as expected?
1. Test catalogs being added correctly
2. Test catalogs are properly added to all worker nodes
3. Test if catalogs are correctly removed from all worker nodes
