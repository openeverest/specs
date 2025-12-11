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

TBD

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
