# Generic Plugins (Extensions)

*   **Status:** Draft
*   **Authors:** @spron-in
*   **Created:** 2026-04-09
*   **Last Updated:** 2026-04-09
*   **Related Issues:** 

---

## 1. Summary

Spec [001](./001-plugins-architecture.md) introduced a plugin system for provisioning and lifecycle management of databases on Kubernetes, backed by Operators and Providers. This spec proposes a complementary, more general extension model — **generic plugins** — that allows anyone to build and ship packages that extend the OpenEverest UI and interact with data, databases, and external systems, without needing deep knowledge of Kubernetes operators, reconciliation loops, or the OpenEverest core.

## 2. Motivation

Database provisioning and lifecycle management (spec 001) is only one layer of what users need around their data. Once a database exists, a large number of valuable use cases remain unaddressed:

- A developer wants to **query and browse their data** from within the OpenEverest UI, without switching to a separate SQL client or DBeaver instance.
- A data engineer wants an **AI assistant** that can introspect the schema, suggest queries, and answer questions about the data.
- A platform team wants to **discover and import visibility** of databases they don't manage directly — e.g., AWS RDS instances, managed services on GCP, or self-hosted databases outside the cluster.
- A team wants to **migrate data** between two database clusters — potentially across providers, versions, or cloud regions.
- A security team wants a **compliance plugin** that audits access patterns, scans for exposed credentials, or enforces tagging policies.

None of these fit the spec 001 model. They are not about provisioning or managing database operators. They are about doing things with databases and data, consuming OpenEverest resources, and potentially bringing in data from outside. Yet they all share a natural home in OpenEverest: this is where the user's databases live.

Without an extension model for this class of capability, every such feature must be built into the OpenEverest core — an approach that does not scale, creates tight coupling, and shuts out community contributions.

**Headlamp** is a widely referenced example of how this can work in the Kubernetes UI space: plugins are self-contained packages that ship their own frontend code, register themselves in the UI at well-defined extension points, and can talk to the Kubernetes API, other plugins, or arbitrary backends.

## 3. Goals & Non-Goals

**Goals:**

* Define what a "generic plugin" is and how it differs from a spec 001 Provider plugin.
* Identify the key scenarios and user archetypes that generic plugins need to serve.
* Identify the surface area a generic plugin needs access to:
  * OpenEverest API (list databases, get connection strings, etc.)
  * Other plugins and their data
  * Kubernetes API
  * External APIs and services
* Identify the two structural parts of a generic plugin: **backend** (an API, deployable anywhere) and **frontend** (UI and/or CLI integration).
* Identify trust, security, and isolation concerns at a high level.

**Non-Goals:**

* Defining the implementation — API contracts, plugin manifests, SDK design, UI framework specifics. This is an early spec.
* Replacing spec 001 — generic plugins complement, not replace, the Provider model.
* Defining a plugin marketplace or distribution mechanism.
* Tooling for plugin development (scaffolding, testing framework, etc.) — out of scope for this spec.

## 4. Problem Space

> **Note:** This is an early draft. No solution has been selected. The section below maps the problem space and raises open questions.

### 4.1 What is a Generic Plugin?

A generic plugin is a self-contained, installable extension to OpenEverest that:

1. **Does not manage database lifecycle** — it consumes databases and data, it does not provision or reconcile them.
2. **Has two optional parts:**
   - A **backend** — a service that implements custom logic, exposes an API, and may run anywhere (in-cluster, sidecar, or as a remote SaaS endpoint).
   - A **frontend** — a UI contribution to the OpenEverest web interface, and/or an extension to `everestctl`, that renders information and calls backend or OpenEverest APIs.
3. **Can be authored and shipped by anyone** — the OpenEverest core team, third-party vendors, or individual community members.

### 4.2 Illustrative Use Cases

| Use Case | Backend? | Frontend? | External access needed? |
|---|---|---|---|
| SQL query browser (DBeaver-like) | Yes (query runner) | Yes (UI editor) | No — talks to in-cluster DBs |
| AI data copilot | Yes (LLM integration) | Yes (chat UI) | Yes — calls external LLM APIs |
| AWS RDS discovery | Yes (AWS API poller) | Yes (import/list UI) | Yes — calls AWS APIs |
| Data migration tool | Yes (migration job runner) | Yes (progress UI) | Possibly |
| Compliance / audit plugin | Yes (policy engine) | Yes (report UI) | Possibly |
| Read-only metrics dashboard | No | Yes (pulls from Prometheus) | Yes — calls monitoring APIs |
| CLI-only backup reporter | No | No (just `everestctl` commands) | No |

### 4.3 What Does a Plugin Need Access To?

For generic plugins to be useful, they need to reach into OpenEverest's world. This raises the question of what the platform must expose and what must remain private.

**From OpenEverest:**
- List and describe managed `DatabaseCluster` instances (name, engine, version, namespace, status)
- Connection details (host, port, credentials — with appropriate authorization)
- Backup and restore status
- Events and audit log

**From Kubernetes:**
- Possibly direct kube-API access for advanced plugins (e.g., a plugin that manages its own CRDs)
- Or a sandboxed subset via OpenEverest's proxy

**From other plugins:**
- Can plugins call each other? (e.g., an AI plugin calls a query plugin)
- Is there a plugin-to-plugin communication model, or only plugin-to-OpenEverest?

**From external systems:**
- AWS, GCP, Azure APIs
- External LLM APIs
- Remote databases outside the cluster

### 4.4 Trust and Security Considerations

Generic plugins introduce significant security surface area. At this stage, we need to at least identify the concerns:

- A malicious or compromised plugin backend could exfiltrate database credentials or data.
- A plugin deployed in-cluster can potentially access any service in the namespace.
- Frontend plugins run in the same browser origin as OpenEverest — XSS, CSRF, and supply chain risks apply.
- Plugins may call external endpoints — data leaving the cluster boundary is a concern for regulated environments.
- Who can install a plugin? All users? Only admins?

### 4.5 Candidate Structural Models

This section does not propose a solution, but identifies the main structural questions:

**Q: Where does the plugin backend run?**
- Always in the same Kubernetes cluster as OpenEverest
- Optionally as a remote endpoint (SaaS plugin)
- Either (plugin author declares)

**Q: How does the frontend reach the backend?**
- Directly (the plugin frontend calls the plugin backend URL)
- Proxied through OpenEverest (OpenEverest routes `/plugins/{name}/...` to the backend)
- Both

**Q: How is the frontend delivered?**
- Plugin ships a JS bundle that is loaded into the OpenEverest UI at runtime (Headlamp model)
- Plugin frontend is an iframe
- Plugins register routes/pages/widgets at declared extension points
- A combination

**Q: How are plugins discovered and installed?**
- A plugin manifest (YAML/JSON) is applied to OpenEverest
- A dedicated plugin registry or catalog
- Direct URL/image reference

**Q: How does the plugin authenticate with OpenEverest APIs?**
- It uses the logged-in user's session/token (delegated auth)
- It uses a service account with scoped permissions
- It uses a plugin-specific API key

## 5. Definition of Done

> To be defined once an approach is selected.

## 6. Alternatives Considered

> To be populated as the design discussion progresses.

## 7. Open Questions

1. Should generic plugins be able to contribute their own Kubernetes CRDs, or is that reserved for spec 001 Providers?
2. What is the minimal backend surface OpenEverest must expose for plugins to be useful? Is the existing API enough, or does a plugin-facing API need new endpoints?
3. How do we handle plugins that need credentials to connect to the databases OpenEverest manages? Does OpenEverest broker that, or does the user configure it separately per plugin?
4. Is there a single plugin model, or do we distinguish between "UI-only extensions" (no backend), "backend-only integrations" (CLI/automation, no UI), and "full plugins" (both)?
5. Can a plugin interact with resources created by spec 001 providers (e.g., read the underlying PSMDB CR)? Should it?
6. Headlamp uses a dynamic JS module loading approach — is that viable for OpenEverest's frontend stack?
7. What does the `everestctl` extension surface look like? Plugin-contributed subcommands?
8. How do we version and update plugins independently of OpenEverest core releases?
9. Are there categories of plugins we would explicitly disallow (e.g., plugins that modify `DatabaseCluster` resources directly)?
10. In air-gapped or regulated environments, how does a user install plugins without internet access?

## 8. References

* [001 - Plugins Architecture](./001-plugins-architecture.md)
* [Headlamp plugin system](https://headlamp.dev/docs/latest/development/plugins/building-and-deploying/)
