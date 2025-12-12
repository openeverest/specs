# Modular core for plugin architecture

*   **Status:** Draft
*   **Authors:** @spron-in
*   **Created:** 2025-12-11
*   **Last Updated:** 2025-12-11
*   **Related Issues:** openeverest/roadmap#1

---

## 1. Summary
As per vision described [here](https://vision.openeverest.io), we propose to transform the core of OpenEverest into a modular and extensible one. 
This will allow developers, the community, and users to easily add new features (such as database engines).

## 2. Motivation
OpenEverest currently relies on hardcoded, vendor-specific solutions for core functionalities. 
For instance, database management is exclusively handled by Percona Operators for MySQL, PostgreSQL, and MongoDB, 
and users are constrained to using Percona Monitoring and Management (PMM) for observability.

While community contributions are technically possible, integrating alternative technologies requires deep, 
specific knowledge of the OpenEverest core, Kubernetes architecture, Operator SDK, Golang, and the JS/React UI. 
This high barrier to entry translates into integration timelines of weeks or even months for adding a new database technology.

This tightly coupled architecture severely limits community engagement, stifles contributions, 
and undermines our fundamental claim of no-vendor lock-in.

## 3. Goals & Non-Goals
There are a few types of plugins that we can think of:

**Database and data technologies**

This includes adding new database technology or storage plugin.

**Integrations with databases for data management**

A few examples:
* Data management plugin in the UI (like built-in DBeaver)
* An AI copilot - it connects to DB endpoints to fetch the data, but does not really change the core.
* Showing observability metrics in the UI

**Goals:**

* Database and data technologies plugin system

The goal will be for a user to write a minimal amount of code, define UI artifacts through YAML manifest or similar and immediately see the plugin in the UI.

**Non-Goals:**

* Integrations with databases for data management

This will be a separate set of plugins. They don't really touch the core of the product and does not really change the backend side of the house.

## 4. Proposed Solution / Design

### Pluggable storage architecture

This design is driven by the need for extensibility and universality. While plugin developers need to still provide custom reconciliation logic, it is equally important to maintain a single, universal `DatabaseCluster` CRD that can represent the configuration of any database system.

Since configuration requirements differ across database vendors and topologies, plugin developers must be given a mechanism to dynamically extend the schema. This ensures the DatabaseCluster CRD can express operator-specific configurations while still remaining consistent.

**DatabaseClusterDefinition**:
Cluster-scoped CRD. Describes available components, versions, images, and topologies.

**Provider**:
Cluster-scoped CRD. References a DatabaseClusterDefinition and aggregates information among other things that are not included in this document.

**DatabaseCluster**:
Namespace-scoped CRD. References Provider and drives configuration of components and topologies at runtime.

#### Technical Details

##### 1. DatabaseClusterDefinition

Represents the available building blocks (components) and valid topologies for a database system/operator.

```yaml
apiVersion: everest.percona.com/v2alpha1
kind: DatabaseClusterDefinition
metadata:
  name: psmdb-operator
spec:
  # Defines a list of component types for this provider.
  # Each component type has a list of versions and images.
  componentTypes:
    mongod:
      versions:
        - version: 8.0.4-1
          image: percona/percona-server-mongodb:8.0.4-1-multi
        - version: 8.0.8-3
          image: percona/percona-server-mongodb:8.0.8-3
          default: true
    backup:
      versions:
        - version: 2.9.1
          image: percona/percona-server-mongodb-backup:2.9.1
    pmm:
      versions:
        - version: 2.44.1
          image: percona/pmm-server:2.44.1

    prometheus:
      versions:
        ...

  # Defines a list of components for this provider.
  # Each component has a type from componentTypes.
  # One or more components may specify the same component type.
  components:
    engine:
      type: mongod
    configServer:
      type: mongod
    proxy:
      type: mongod
    backupAgent:
      type: backup
    monitoring:
      type: prometheus

  # Defines a list of topologies for this provider.
  # Each topology defines the components that are supported by that topology.
  topologies:
    standard:
      components:
        engine:
          defaults:
            replicas: 3
        backupAgent:
          optional: true
        monitoring:
          optional: true

    sharded:
      components:
        engine:
          defaults:
            replicas: 3
        proxy: {}
        configServer: {}
        backupAgent:
          optional: true
        monitoring:
          optional: true
```

##### 2. Provider

Maps to a single DatabaseClusterDefinition. A DatabaseCluster must belong to a single Provider in order for OpenEverest to be able to derive the correct structure of the components and topologies.

```yaml
apiVersion: everest.percona.com/v2alpha1
kind: Provider
metadata:
  name: psmdb-operator
spec:
  # ...
  databaseClusterDefinition: psmdb-operator
```
##### 3. DatabaseCluster

Represents an operational database cluster instance, referencing a Provider for structure and validation.
```yaml
apiVersion: everest.percona.com/v2alpha1
kind: DatabaseCluster
metadata:
  name: psmdb-cluster
  namespace: default
  # DatabaseCluster CRD design
spec:
  provider: psmdb-operator
  topology:
    type: sharded
    config: {}
  components:
    versions:
      mongod:
        version: 8.0.8-3
      backup:
        version: 2.9.1
      prometheus:
        version: x.y.z
  instances:
    engine: {}
    proxy: {}
    configServer: {}
    backupAgent: {}
    monitoring: {}
```

## 5. Definition of Done

* Contributors can add new database engines and technologies within days.
* The path to add new plugin as well documented and semi-automated
* Existing database technologies (Percona Operators) are using the new plugin system
* * Upgrade path is described and well tested

## 6. Alternatives Considered

TBD

## 7. Open Questions

There are lots of questions regarding UI and everything else.

## 8. References

TBD
