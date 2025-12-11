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

**Non-Goals:**

*   Integrations with databases for data management
*   
This will be a separate set of plugins. They don't really touch the core of the product and does not really change the backend side of the house.

## 4. Proposed Solution / Design

TBD

## 5. Definition of Done

What does "completed" look like?
List the concrete, verifiable criteria that must be met for this spec to be considered fully Implemented. This is your checklist.

* Feature X is deployed to the staging environment.
* API documentation is updated in /docs.
* Integration tests with >90% coverage are passing.
* A user guide has been written.

## 6. Alternatives Considered

What other paths did we explore?
Discuss other designs, technologies, or approaches that were considered. Explain why the proposed solution was chosen over these alternatives. This demonstrates thorough thinking.

## 7. Open Questions

What do we still need to figure out?
List any unresolved issues, trade-offs, or decisions that need further discussion or research during the review phase.

## 8. References

Links to relevant documentation, research, or prior art.
