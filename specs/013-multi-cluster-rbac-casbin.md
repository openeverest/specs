# Multi-Cluster RBAC

*   **Status:** Draft
*   **Authors:** @recharte
*   **Created:** 2026-05-07
*   **Last Updated:** 2026-09-16
*   **Related Issues:** [openeverest#3009](https://github.com/openeverest/openeverest/issues/3009), [openeverest#3104](https://github.com/openeverest/openeverest/pull/3104)

---

## 1. Summary

OpenEverest v2 introduces multi-cluster management via the `/clusters/{cluster}/...` API surface. This spec extends the existing Casbin-based RBAC system to cluster-aware authorization by embedding the cluster name in the hierarchical object field of the 4-tuple policy model (`sub, res, act, obj`). The Casbin model file remains unchanged — only the object format evolves from `namespace/name` to `cluster/namespace/name`.

This spec is the normative definition of v2 authorization: the complete resource and action vocabulary, object arity rules and policy validation, enforcement semantics (including derived checks on writes and credential gating), failure modes, and the threat model.

## 2. Motivation

v1 policies use a two-level object (`namespace/name`) with no cluster dimension — there is no way to express "user X can manage instances on cluster A but not B". Without cluster-aware objects, adding a second managed cluster grants every user implicit access to every resource on every cluster.

The v2 enforcement code was built ahead of this spec, and its gaps trace back to the missing normative definition:

- The restore endpoints and the `/events` SSE stream perform no RBAC checks at all.
- The 3-tier object format was never written down: `namespaces` is enforced with 1-segment objects while its admin policy grants 2 segments (namespace lists come back empty under RBAC), and `instance-presets` is uncategorized and works only via a legacy fallback.
- Plugins started as a PoC outside the generated chain ([#3009](https://github.com/openeverest/openeverest/issues/3009)): `plugins` was absent from the generated resource list — admins were locked out, and any `plugins` policy line invalidated the entire ConfigMap — and the plugin name was encoded in the *resource* string (`plugin/<name>`) instead of the object. [#3104](https://github.com/openeverest/openeverest/pull/3104) moves the list/context endpoints onto the chain; this spec settles the model those endpoints and the proxy must enforce.
- v1's derived checks (e.g., backup-storage read when creating an instance with schedules) were not ported, and v1's separate credentials permission was silently collapsed into `instances:read`.

### Current Architecture (kept by this spec)

1. **Casbin model** (`data/rbac/model.conf`) — 4-tuple `(sub, res, act, obj)`; `globMatch` on res/act/obj; `g(sub, role)` role links.
2. **Policy storage** — ConfigMap `everest-rbac` in the system namespace (`enabled` flag + `policy.csv`), hot-reloaded via informer.
3. **Handler chain** — Rate Limiting → JWT Auth → **RBAC handler** → Validation → Kubernetes handler. The RBAC handler enforces with the user's subject and each OIDC group as Casbin subjects.
4. **Post-filtering** — LIST fetches all items, then silently omits those failing `enforce()`.

## 3. Goals & Non-Goals

**Goals:**

*   Cluster-aware authorization with the Casbin model file unchanged.
*   Gate clusters themselves as a protected resource; support per-cluster, cross-cluster, and per-resource-instance roles.
*   Define the complete, normative v2 resource/action vocabulary and object formats — every endpoint covered, including restores and `/events`.
*   Define policy validation (object arity, unknown resources/actions) and fail-closed failure modes.
*   Preserve v1's credential-access granularity and its derived write-side checks.
*   Document the threat model and the v1 → v2 policy migration path.

**Non-Goals:**

*   Backward compatibility with v1 endpoints — v1 API endpoints and their RBAC resource names are removed.
*   Deny rules — the model is allow-only (§4.6).
*   Multi-cluster policy synchronization — policies live only on the management cluster.
*   Kubernetes-native RBAC on managed clusters; per-cluster OIDC.
*   Audit logging of RBAC decisions (future spec).

## 4. Design

### 4.1 Object Format and Glob Semantics

The object is a `/`-separated hierarchical path whose segment count (arity) is fixed by the resource's scope. Casbin's `globMatch` is Go's `path.Match`, which gives this scheme both its power and its one trap:

> **`*` matches within a single path segment and never crosses `/`.**

| Pattern | Matches | Does NOT match |
|---------|---------|----------------|
| `*` | `prod` | `prod/db`, `prod/ns/db` |
| `*/*` | `prod/db` | `prod`, `prod/ns/db` |
| `*/*/*` | `prod/ns/db` | `prod`, `prod/db` |
| `prod/*/*` | everything on the prod cluster | any 1- or 2-segment object |
| `*/dev/*` | the dev namespace on any cluster | — |

A policy whose object arity does not match its resource's scope **silently never matches** — it grants nothing. Validation therefore rejects arity mismatches (§4.6), and "access to everything" takes one line per tier (§4.9).

Cluster, namespace, and resource names are Kubernetes object names (DNS-1123): they cannot contain `/` or glob metacharacters, so names cannot inject separators or patterns into objects. This is an invariant, not a convention.

### 4.2 Resource Scopes

Every v2 resource has exactly one scope. The categorization is exhaustive and machine-enforced: resource-name generation and admin-policy generation MUST fail on an uncategorized resource instead of falling back to a default (§4.7).

| Scope | Object format | Wildcard | Resources |
|-------|--------------|----------|-----------|
| Global | `name` | `*` | `clusters` |
| Cluster-scoped | `cluster/name` | `*/*` | `namespaces`, `providers`, `backup-classes`, `instance-presets`, `plugins` |
| Cluster+namespace | `cluster/namespace/name` | `*/*/*` | `instances`, `backups`†, `restores`†, `backup-storages`, `monitoring-configs`, `config-maps`, `secrets` |

† `backups` and `restores` use the **owning instance's name** as the last segment, not their own (§4.5).

Endpoints outside the resource model:

| Endpoint | Treatment |
|----------|-----------|
| `/events` (SSE) | Enforced per event (§4.5) — currently unenforced; must be fixed |
| `GET /clusters/{cluster}/plugins/{name}/*` | Unauthenticated by design — serves static frontend bundles; bundle contents are public |
| non-GET `/clusters/{cluster}/plugins/{name}/*` | Authenticated reverse proxy to the plugin backend; every request requires `plugins:use` on `cluster/name` (§4.3) |
| `/clusters/{cluster}/plugin-context` | Authenticated, no per-plugin gate; the namespace list is filtered by `namespaces:read` |
| `/permissions`, `/cluster-info`, `/resources`, `/version` | RBAC skip list — authenticated, self-describing or non-sensitive |
| `/session`, `/auth/token`, `/auth/revoke`, `/settings` | Unauthenticated by design (login bootstrap); `/settings` must expose only what the login page needs |

### 4.3 Actions

| Action | Meaning |
|--------|---------|
| `read` | GET and LIST; also gates SSE event delivery and preset `resolve`¹ |
| `create` | POST |
| `update` | PUT / PATCH |
| `delete` | DELETE |
| `create-from-preset` | Create an instance **only as an exact materialization of an `InstancePreset`** (`instances` only)² |
| `read-connection` | GET `/instances/{i}/connection` — returns credentials. **Not implied by `read`**; implied by `*` |
| `use` | Plugin backend invocation through the proxy (`plugins` only)³ |
| `*` | All of the above |

¹ `POST .../instance-presets/{name}/resolve` is a computation, not a mutation: it requires `instance-presets:read` plus `namespaces:read` on the target namespace.

² `create-from-preset` is the governance counterpart to `create`: platform teams publish vetted presets, and preset-only users can materialize them without the power to craft arbitrary instances. Creating an instance that references a preset (`openeverest.io/instance-preset` annotation) requires `instance-presets:read` on the preset, then either `instances:create` (any customization allowed) or `instances:create-from-preset` (the instance must match the resolved preset exactly). `create` is strictly stronger — holders may also create from presets, with customization — while `create-from-preset` grants the constrained capability alone; `*` implies both (and the pattern `create*` matches both). Two deliberate properties: preset *visibility* (`read`) doubles as *deployability* — restrict deployment by restricting which presets a role can read; and the exact-match check must cover behavior-bearing metadata, not just the spec — unknown `openeverest.io/*` annotations are rejected on the preset-only path.

³ Every authenticated proxied request to a plugin backend requires `plugins:use` on the plugin's `cluster/name` object; `read` gates catalog visibility only, and neither implies the other. The PoC-era `plugin/<name>`/`plugin/*` *resource-string* encoding is removed — the plugin name belongs in the object like every other resource — and validation (§4.6) rejects it.

`read-connection` restores v1's separate `database-cluster-credentials` permission: seeing an instance must not imply seeing its password. It is an action rather than a separate resource (the v1 shape) so that the ubiquitous read-only role — `p, role:viewer, *, read, …` — can never accidentally grant credentials via the resource wildcard; only an explicit grant or full control (`act: *`) does.

### 4.4 v1 → v2 Resource Mapping & Migration

| v1 resource | v1 object | v2 resource | v2 object |
|---|---|---|---|
| `database-clusters` | `ns/name` | `instances` | `cluster/ns/name` |
| `database-cluster-backups` | `ns/db-cluster` | `backups` | `cluster/ns/instance` |
| `database-cluster-restores` | `ns/db-cluster` | `restores` | `cluster/ns/instance` |
| `database-cluster-credentials` | `ns/db-cluster` | `instances` with `read-connection` | `cluster/ns/name` |
| `database-engines` | `ns/name` | `providers` | `cluster/name` |
| `monitoring-instances` | `ns/name` | `monitoring-configs` | `cluster/ns/name` |
| `backup-storages` | `ns/name` | `backup-storages` | `cluster/ns/name` |
| `namespaces` | `name` | `namespaces` | `cluster/name` |
| `pod-scheduling-policies`, `load-balancer-configs`, `data-importers`, `data-import-jobs`, `enginefeatures/*` | — | *removed — no v2 endpoints; delete the constants* | — |
| *(new)* | — | `clusters`, `backup-classes`, `instance-presets`, `config-maps`, `secrets`, `plugins` | per §4.2 |

**Migration is a manual rewrite.** Auto-migration is impossible: resources were renamed, scopes changed arity, and some resources have no successor. Upgrade docs ship this table, and `everestctl settings rbac validate` flags v1 resource names and arity mismatches so admins can iterate until the policy is clean.

### 4.5 Enforcement Semantics

*   **Enforce before fetch.** Permission checks run before any backend lookup, so unauthorized requests get a uniform 403 whether or not the resource exists — no existence oracle.
*   **LIST post-filtering.** Unauthorized items are silently omitted from list responses (no 403).
*   **Writes fail loud.** CREATE/UPDATE/DELETE without permission returns 403 naming the missing `resource:action:object`.
*   **`/events` (SSE).** Every event is filtered with the same resource/object keys under `read` before delivery — a subscription is not a bypass around per-resource checks.
*   **Backups and restores are keyed by their owning instance** (`cluster/ns/instance`), not their own name. Backup names are typically generated (`generateName`), making name-based grants unusable; one grant covers an instance's whole backup history; this matches v1 semantics and the nested `/instances/{i}/backups` routes.
*   **The cluster gate is explicit.** `clusters:read` controls only cluster visibility (list/get). Nested resource checks are self-sufficient — `instances:read` on `prod/dev/*` works without `clusters:read` on `prod` — but UI navigation effectively requires the explicit grant, so roles should include it (§4.9).
*   **Derived checks run on writes only.** Creating/updating a resource that references others requires read on the referenced resources — otherwise `instances:create` alone lets a user point backups at someone else's storage:

    | Operation | Additional checks |
    |---|---|
    | Create/update instance | `providers:read` on the referenced provider; `backup-storages:read` on each referenced storage; `backups:create` (obj = instance) when schedules are declared; `restores:create` + `backups:read` (obj = source instance) when `dataSource` is set |
    | Create instance from preset | `instance-presets:read` on the preset, then `instances:create` (any customization) or `instances:create-from-preset` (exact preset match only) — see §4.3. Exact-preset creations **waive** the reference checks above: trust derives from the vetted preset, whose author's authority covers its provider/storage references (the Service Catalog launch-role pattern) |
    | Create backup | `backups:create` (obj = instance) + `backup-storages:read` on the target storage |
    | Create restore | `restores:create` (obj = target instance) + `backups:read` (obj = source backup's instance) |

    Reads deliberately do **not** fan out to referenced resources. v1 gated instance *reads* on read access to every referenced storage and monitoring config, which made resources invisibly disappear from lists and multiplied enforcement cost per item; a reference leaks only a name, not the object.
*   **Credentials.** `instances:read` never returns connection secrets; the connection endpoint requires `read-connection` (§4.3). The generic `secrets` resource independently gates raw Secret access.

### 4.6 Policy Validation & Failure Modes

**The model is allow-only.** The policy definition has no `eft` column, so every `p` line is an allow rule and the `!some(where (p.eft == deny))` clause in the effect expression is inert — deny rules cannot be expressed. The model file stays unchanged; effective access is the union of allow rules.

**Validation** runs in `everestctl settings rbac validate` and again on every ConfigMap (re)load. A policy is rejected when:

*   a `p` line does not have exactly 4 values, or a `g` line exactly 2;
*   the resource is not a known v2 resource name and not a glob;
*   the action is not in the vocabulary (§4.3) and not a glob;
*   the object arity does not match the resource's scope (§4.1–§4.2) — the silent-never-matches trap.

**Failure modes are fail-closed:**

| Condition | Behavior |
|---|---|
| Reload yields invalid policy | Keep last-known-good policy and log an error. Never fail open, never wipe to empty. |
| ConfigMap missing/unreadable at startup | Only the built-in generated admin policy is active; all other subjects are denied. |
| `enabled: "false"` | Enforcement disabled entirely — the documented escape hatch. The Helm chart currently defaults to disabled; this spec recommends default-on for v2 GA (§7). |

### 4.7 Admin Policy Generation

`loadAdminPolicy()` grants the built-in admin role action `*` on every resource, with the wildcard arity of the resource's scope (`*`, `*/*`, or `*/*/*`). Two hard requirements:

*   The generated `AllResources` list and the scope categorization must cover the same set. Generation MUST fail on an uncategorized resource instead of falling back to a default — `instance-presets` is uncategorized today (works only via a legacy 2-segment fallback), and `plugins` was missing from `AllResources` entirely (admin lockout) until [#3104](https://github.com/openeverest/openeverest/pull/3104).
*   A CI check asserts `AllResources` ⊆ (Global ∪ ClusterScoped ∪ ClusterNamespaced).

### 4.8 Threat Model

| Boundary / vector | Position |
|---|---|
| Everest RBAC is the **sole tenant boundary** | The Everest service account is fully privileged on managed clusters; nothing below this layer separates tenants. |
| ConfigMap write = RBAC root | Write access to `everest-rbac` grants control of all authorization. Protect it with Kubernetes RBAC; manage the policy via GitOps for review, versioning, and rollback. |
| IdP claims are trusted input | Group claims are enforced as Casbin subjects, and `g(x, x)` identity-matches — an OIDC group literally named `role:super-admin` would inherit that role. User extraction MUST reject subject/group claims carrying the `role:` prefix, and docs must call out IdP group hygiene. |
| Object injection | Impossible — all object segments are DNS-1123 Kubernetes names (§4.1). |
| Plugin backends | Reached only through the proxy after `plugins:use`; plugin daemon service tokens are additionally hard-denied writes to core resources regardless of policy. |
| UI filtering | UX only, never a control. The server-side handler chain is the enforcement point. |

### 4.9 Policy Examples

Because `*` never crosses `/`, "access to everything" takes one line per scope tier:

```csv
# Full admin (equivalent to the built-in admin role)
p, role:super-admin, *, *, *
p, role:super-admin, *, *, */*
p, role:super-admin, *, *, */*/*

# Read-only everywhere (read does NOT include read-connection)
p, role:viewer, *, read, *
p, role:viewer, *, read, */*
p, role:viewer, *, read, */*/*

# Prod-cluster admin
p, role:prod-admin, clusters, read, prod
p, role:prod-admin, *, *, prod/*
p, role:prod-admin, *, *, prod/*/*

# Dev team: the dev namespace on any cluster
p, role:dev, clusters, read, *
p, role:dev, namespaces, read, */dev
p, role:dev, providers, read, */*
p, role:dev, instances, *, */dev/*
p, role:dev, backups, *, */dev/*
p, role:dev, backup-storages, read, */dev/*
# catalog visibility + backend invocation for one plugin, any cluster
p, role:dev, plugins, read, */sql-explorer
p, role:dev, plugins, use, */sql-explorer

# Fine-grained: one instance, credentials included
p, alice, clusters, read, prod
p, alice, instances, read, prod/default/my-db
p, alice, instances, read-connection, prod/default/my-db

# Preset-only deployer: may materialize vetted presets, not craft instances
p, role:deployer, clusters, read, *
p, role:deployer, instance-presets, read, */*
p, role:deployer, instances, read, */*/*
p, role:deployer, instances, create-from-preset, */*/*

# Role assignments (users or OIDC groups)
g, admin-user, role:super-admin
g, dev-group, role:dev
```

Multi-tenant isolation (SaaS per-customer clusters, regional sharding) is the prod-admin pattern with a different cluster name per tenant; namespace-based tenancy on a shared cluster is the dev-team pattern.

### 4.10 Permissions Discovery & UI

`GET /permissions` returns the caller's resolved policy tuples (roles expanded) with cluster-prefixed objects. The UI uses it to filter cluster/namespace/resource views and disable unavailable actions. Short-TTL caching is acceptable because the result is advisory — stale UI state only changes which 403s a user can provoke, never what the server allows.

## 5. Definition of Done

**Model & vocabulary:**
*   `data/rbac/model.conf` unchanged; allow-only semantics documented.
*   Every v2 resource categorized per §4.2, with a CI check on categorization completeness; `plugins` present in `AllResources` and cluster-scoped ([#3104](https://github.com/openeverest/openeverest/pull/3104)); `instance-presets` categorized as cluster-scoped.
*   Dead v1 resource constants removed from `pkg/rbac`.

**Enforcement:**
*   Every endpoint in §4.2 enforces — including restores (the restore handler interface gains the cluster parameter) and per-event filtering on `/events`.
*   The connection endpoint requires `read-connection`; `instances:read` alone never returns credentials.
*   Derived write-side checks (§4.5) implemented for instances, backups, and restores; backups/restores keyed by owning instance; exact-preset creations waive reference checks.
*   The `create-from-preset` exact-match check rejects unknown `openeverest.io/*` annotations (metadata is part of the match, not just spec); the action is renamed from the current `deploy` in code.
*   The plugin proxy enforces `plugins:use` on `cluster/name` objects, replacing the PoC `plugin/<name>`/`plugin/*` resource strings and the `ActionAll` admin shortcut with a 1-segment object (regression flagged in the [#3104](https://github.com/openeverest/openeverest/pull/3104) review); plugin catalog listing post-filters on `plugins:read`.
*   Uniform 403 before fetch; LIST post-filtering; namespace listing works for non-admin users (2-segment objects).

**Validation, tooling & failure modes:**
*   Arity/vocabulary validation in `everestctl settings rbac validate` and at ConfigMap load; `everestctl settings rbac can` handles all three arities.
*   Invalid reload keeps last-known-good policy (tested).
*   Token subject/group claims with the `role:` prefix are rejected (tested).

**Tests & docs:**
*   Unit tests for every RBAC handler file; an e2e authorization matrix in api-tests covering resources × actions × allow/deny.
*   User docs rewritten with the v2 vocabulary and the §4.4 migration table.
*   Helm default posture decided and documented.
*   `make test` and `make check` pass with no regressions.

## 6. Alternatives Considered

**Casbin domains (`g = _, _, _`)** — native multi-tenancy with per-domain role assignments. Rejected: breaks every existing `g` line, domain matching doesn't use `globMatch` by default (poor cross-cluster wildcards), and `GetImplicitPermissionsForUser` doesn't resolve cross-domain roles without extra configuration.

**5th tuple field (`r = sub, res, act, obj, cluster`)** — cluster as an explicit dimension. Rejected: touches the model, every policy, every enforce call site, and migration tooling, for the same expressiveness the hierarchical object already provides.

**Per-cluster RBAC deployments** — an enforcer and policy store per managed cluster. Rejected: no centralized management, cross-cluster roles impossible, `/permissions` would need cross-enforcer aggregation.

**Custom matcher where `*` crosses `/`** — would make `p, role:x, *, *, *` mean "everything" and remove the one-line-per-tier idiom. Rejected: diverges from the `path.Match`/ArgoCD conventions operators already know, and hides scope mistakes that strict arity validation (§4.6) surfaces loudly at policy-load time.

## 7. Open Questions

1.  **Default posture** — should the Helm chart ship `enabled: "true"` at v2 GA? Recommended by this spec, but it changes behavior for installs upgrading without a policy; needs a product decision.
2.  **Audit logging** — logging every RBAC decision has real performance and storage implications; deferred to a dedicated spec.
3.  **Performance at scale** — post-filtered LIST is O(items × subjects × policies). If profiling shows problems, consider Casbin `BatchEnforce` or compiled per-user permission sets.

## 8. References

**Industry RBAC Patterns:**
*   [Casbin globMatch function](https://casbin.org/docs/function#globmatch) — implemented as Go's [`path.Match`](https://pkg.go.dev/path#Match); `*` never crosses `/`.
*   [ArgoCD RBAC documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/) — `project/application` scoping with glob patterns.
*   [Rancher RBAC architecture](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/authentication-permissions-and-global-configuration/manage-role-based-access-control-rbac) — global + cluster + project roles.
*   [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) — `Role` vs. `ClusterRole` scope separation.

**OpenEverest Documentation:**
*   [OpenEverest v2 API spec](openeverest/api/openapi/http-api.yaml) — `/clusters/{cluster}/...` endpoint definitions.
*   [Core API types](openeverest/api/core/v1alpha1/) — `Instance`, `Provider` CRD definitions for the v2 API.
*   [Plugin Architecture spec](001-plugins-architecture.md) — how providers and plugins work in OpenEverest.
