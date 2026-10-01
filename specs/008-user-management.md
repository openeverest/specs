# User Management Plugin

*   **Status:** Draft
*   **Authors:** @spron-in
*   **Created:** 2026-06-17
*   **Last Updated:** 2026-06-17
*   **Related Issues:**
*   **Related specs:** [001 — Modular core / Provider plugins](./001-plugins-architecture.md), [003 — Generic plugins](./003-generic-plugins.md)

> **Note on API groups.** This spec follows the post-rename layout: `Plugin` and `InstalledExtension` live under `extensions.openeverest.io/v1alpha1`; `Provider` and `Instance` remain in `core.openeverest.io`. Plugin-declared CRDs (including this plugin's `DatabaseUser` / `DatabaseRole` / `DatabaseGrant`) continue to live under `<pluginName>.plugins.openeverest.io`.

---

## 1. Summary

A generic plugin (per spec 003) that manages **database users, roles, and grants** across the database engines OpenEverest provisions. The plugin declares its own namespaced CRs (`DatabaseUser`, `DatabaseRole`, `DatabaseGrant`) and reconciles them through pluggable **engine adapters**. An adapter uses the provider's native facility when one exists (e.g., the `users` array on the `PerconaServerMongoDB` cluster CR) and falls back to a direct database connection when it does not. Credentials are minted by the plugin and stored as Kubernetes `Secret`s in the user's namespace.

## 2. Motivation

OpenEverest provisions database clusters but stops at the cluster boundary. Once a cluster is `ready`, a user still has to:

- `kubectl exec` into a pod and run `CREATE USER` / `db.createUser()` by hand, or
- juggle provider-specific surfaces (the Percona Operator for MongoDB exposes users as an inline `users[]` array on the `PerconaServerMongoDB` cluster CR; PostgreSQL and PXC don't expose anything equivalent), or
- script a sidecar to keep app credentials in sync with `Secret`s.

None of this is uniform across engines and none of it is visible in the OpenEverest UI. Teams end up with stale credentials, no audit trail, and no consistent way to grant least-privilege access per application.

Putting this in the **core** locks every engine into one user-management model and bloats the provider contract. Putting it in **each provider** duplicates the same CRUD/UI for every engine and re-implements the parts that are genuinely engine-agnostic (naming, secret layout, rotation cadence, UI). A **generic plugin with engine adapters** is the right shape: one CR schema and one UI for the user, engine-specific work isolated behind a small adapter interface.

## 3. Goals & Non-Goals

**Goals:**

* One declarative model — `DatabaseUser`, `DatabaseRole`, `DatabaseGrant` — that works across all OpenEverest-supported engines.
* Credentials minted by the plugin and delivered as `Secret`s the application can mount directly.
* Two adapter strategies per engine: **provider-native** (delegate to the operator's CR) and **direct** (open an authenticated connection and issue SQL).
* Reconciliation that converges: deletion of a `DatabaseUser` removes the user from the database; spec changes (rename, password rotation, grant additions) are applied in place.
* Surfaced in the OpenEverest UI as a `clusterDetailTab` (Users, Roles, Grants) on every cluster the plugin supports.
* Audit trail of every user/role/grant change via the plugin backend's logs and the OpenEverest event stream.

**Non-Goals:**

* SSO / LDAP / OIDC federation — see §7 (out of scope for v1; sketched as a later phase).
* Application-level authorization (row-level security, fine-grained policies inside the schema).
* Managing **superuser** / operator-owned system accounts (`root`, `clusterAdmin`, replication users). The plugin refuses to touch reserved names.
* Cross-cluster user federation (the same `DatabaseUser` name targeting N clusters simultaneously) — a later phase.
* Password vaulting beyond Kubernetes `Secret`s (no integration with Vault / AWS Secrets Manager in v1).

## 4. Design

### 4.1 Plugin shape

A spec-003 generic plugin with:

- **Stateful** (§10.8 of spec 003) — declares `DatabaseUser`, `DatabaseRole`, `DatabaseGrant` CRDs under `users.plugins.openeverest.io`.
- **Event consumer** (§10.5) — subscribes to `database-cluster.ready` / `.deleted` to reconcile users on cluster lifecycle changes.
- **Request handler** — backs the UI tab and the `everestctl users` CLI.
- **Frontend bundle** — registers `clusterDetailTab` (provider-filtered to engines that have an adapter) and a small set of forms.
- **`kubePermissions`** — namespace-scoped: `create/get/list/watch/update/delete` on `secrets`. Native adapters that delegate to a separate operator-owned user CR get the corresponding `apiGroups/resources` rule. Native adapters that mutate fields **inside** the provider's cluster CR (PSMDB's inline `users[]`) need a narrower path — see Open Question §8.6.

The plugin does **not** modify spec-001 resources directly. It reads `DatabaseCluster` to discover connection details. Any write to the underlying provider's CR happens through a host-mediated patch endpoint (see §8.6), never by giving the plugin blanket write access.

### 4.2 Custom resources

```yaml
apiVersion: users.plugins.openeverest.io/v1alpha1
kind: DatabaseUser
metadata:
  name: app-orders
  namespace: team-alpha
spec:
  clusterRef:
    name: orders-pg            # DatabaseCluster in the same namespace
  username: app_orders         # final DB-side name; immutable after create
  passwordSecretRef:           # optional; if absent, plugin generates one
    name: app-orders-password
    key: password
  roles:                       # references to DatabaseRole, OR built-in role names
    - readwrite_orders
  rotation:
    enabled: true
    intervalDays: 90           # null = manual rotation only
status:
  ready: true
  secretRef:                   # always written by the plugin; consumer-facing
    name: app-orders-credentials
  lastRotated: 2026-06-17T10:00:00Z
  conditions: [...]
```

```yaml
apiVersion: users.plugins.openeverest.io/v1alpha1
kind: DatabaseRole
metadata:
  name: readwrite_orders
  namespace: team-alpha
spec:
  clusterRef:
    name: orders-pg
  # Engine-neutral capability list. The adapter translates to engine syntax.
  capabilities:
    - read
    - write
  # Optional engine-specific escape hatch — used verbatim by the adapter.
  raw:
    postgresql: |
      GRANT USAGE ON SCHEMA orders TO :role;
      GRANT SELECT, INSERT, UPDATE, DELETE
        ON ALL TABLES IN SCHEMA orders TO :role;
  scope:
    databases: [orders]        # engine-specific meaning (DB, schema, keyspace…)
```

```yaml
apiVersion: users.plugins.openeverest.io/v1alpha1
kind: DatabaseGrant
metadata:
  name: app-orders-readwrite
  namespace: team-alpha
spec:
  clusterRef: { name: orders-pg }
  userRef:    { name: app-orders }
  roleRef:    { name: readwrite_orders }
```

`DatabaseGrant` is split from `DatabaseUser.spec.roles` so that one role can be granted to many users by RBAC-restricted callers without editing the user CR itself.

### 4.3 Engine adapter interface

The plugin backend selects an adapter at reconcile time based on the target `DatabaseCluster`'s `spec.engine.type`. The interface is small and HTTP-internal (Go interface, not exposed over the wire):

```go
type Adapter interface {
    EnsureUser(ctx, cluster, user) (UserStatus, error)
    DeleteUser(ctx, cluster, username) error
    EnsureRole(ctx, cluster, role) error
    DeleteRole(ctx, cluster, roleName) error
    ApplyGrants(ctx, cluster, username, roles) error
    RotatePassword(ctx, cluster, username) (newSecretRef, error)
}
```

Two implementations per engine ship in the plugin image:

| Engine | Native adapter | Direct adapter |
|---|---|---|
| Percona Server for MongoDB | Patch `users[]` on the `PerconaServerMongoDB` cluster CR (see §8.6) | `mongo` client → `db.createUser` / `grantRolesToUser` |
| Percona XtraDB Cluster | — | `mysql` client → `CREATE USER` / `GRANT` |
| Percona PostgreSQL (PGO) | `users[]` on the `PerconaPGCluster` cluster CR (similar shape to PSMDB; see §8.6) | `pgx` → `CREATE ROLE LOGIN` / `GRANT` |

Selection is per cluster, declared on the `InstalledExtension`:

```yaml
spec:
  config:
    adapters:
      psmdb: native              # use the operator CR
      pxc:   direct              # talk to the DB
      pg:    auto                # native if available, else direct
```

A new engine adds support by implementing the interface; nothing in core changes.

### 4.4 Credential flow

1. On `DatabaseUser` create, if `passwordSecretRef` is unset the plugin generates a 32-byte random password and writes it to `Secret/<user>-password`.
2. The adapter creates the user in the database.
3. The plugin writes the consumer-facing `Secret/<user>-credentials` containing `username`, `password`, `host`, `port`, `database`, and an engine-appropriate `uri`. This is the secret applications mount.
4. On rotation, the plugin generates a new password, updates the database via the adapter, then updates the credentials secret atomically. Old passwords are not retained — apps that cache must reconnect on auth failure.
5. On `DatabaseUser` delete, the adapter drops the user and the plugin garbage-collects both secrets.

The cluster connection used by the direct adapter is brokered by `GET /v1/databases/{id}/connection-details` (spec 003 §16 Q3) and is scoped to the plugin's service token, never to the requesting user.

### 4.5 Reconciliation triggers

- CR create/update/delete → host calls `POST /v1/plugins/user-management/reconcile` (spec 003 §10.8).
- `database-cluster.ready` event → the plugin replays all `DatabaseUser`s for that cluster (handles cluster recreate / restore-from-backup, which loses DB-side users).
- `database-cluster.deleted` event → the plugin marks every `DatabaseUser` for that cluster as `Orphaned`, deletes secrets, but **does not delete the CRs** (the user may want to recreate the cluster with the same users).
- Periodic resync (1h) → drift detection: lists DB-side users, compares to CRs, logs deltas. Auto-heal is opt-in per `InstalledExtension` (default off).

### 4.6 UI

A `clusterDetailTab` with three sub-tabs (Users, Roles, Grants), provider-filtered to engines with adapters. Each sub-tab is a list + create/edit form bound to the corresponding CR. The user form has a one-click "Reveal credentials" action that reads the credentials secret via the plugin backend (RBAC-checked).

### 4.7 CLI

```sh
everestctl users list   -n team-alpha --cluster orders-pg
everestctl users create -n team-alpha --cluster orders-pg --name app-orders --role readwrite_orders
everestctl users rotate -n team-alpha app-orders
everestctl users delete -n team-alpha app-orders
```

Implemented as the plugin's CLI contribution (spec 003 §12).

### 4.8 HTTP API

**CRs are the source of truth.** Declarative CRUD on `DatabaseUser` / `DatabaseRole` / `DatabaseGrant` goes through the OpenEverest API's existing Kubernetes-style endpoints (the same path the UI uses for any plugin CR per spec 003 §10.8). The UI's "create user" form is just a `POST` of a `DatabaseUser`; `everestctl users delete` is a `DELETE` of the CR. No separate REST verbs duplicate that path.

The plugin backend, mounted at `/v1/plugins/user-management/*` (spec 003 §10.1), exposes a small **imperative** surface for things that don't fit a CR — actions, queries against live DB state, and host callbacks:

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/reconcile` | Host → plugin. Standard stateful-plugin reconcile hook (spec 003 §10.8). Not called by users. |
| `POST` | `/users/{user}/rotate` | Trigger an immediate password rotation. Returns the new credentials secret ref. |
| `GET`  | `/users/{user}/credentials` | Reveal credentials (one-shot, audit-logged). RBAC-gated; backs the UI "Reveal" action. |
| `POST` | `/users/{user}/test-connection` | Open a connection with the minted credentials and report success/failure. Used by the UI after create. |
| `GET`  | `/clusters/{cluster}/db-users` | List users as the database itself reports them (not the CRs). Backs drift detection in the UI. |
| `POST` | `/clusters/{cluster}/import` | Adopt a pre-existing DB-side user by creating a matching `DatabaseUser` CR (see Open Question §8.4). |

All requests carry the caller's `X-Everest-User` JWT (spec 003 §10.1); the plugin re-checks the user's permission on the underlying `DatabaseCluster` before acting. The plugin's own service token (spec 003 §10.4) is used only for in-cluster machinery (talking to the database, reading secrets) — never to bypass the requester's RBAC.

## 5. Definition of Done

* `DatabaseUser`, `DatabaseRole`, `DatabaseGrant` CRDs installed by the plugin via spec 003 §10.8.
* Native + direct adapters shipped for PSMDB, PXC, and PG.
* Credentials secret schema documented and stable across rotations.
* Cluster-deletion and cluster-recreate paths covered by e2e tests on at least one engine per topology (replicaset, group-replication, primary/replica).
* UI tab and `everestctl users` CLI available.
* User guide covering the four common flows: app onboarding, rotation, revocation, debugging a stuck user.

## 6. Alternatives Considered

* **Build into the core.** Rejected: forces every engine through one model and grows the provider contract. Engines without a native user CR would still need the direct-connection logic — same complexity, less isolation.
* **Build into each provider.** Rejected: duplicates UI, CLI, secret layout, and rotation policy per engine. The genuinely engine-specific surface (the actual SQL / Mongo commands) is the small part.
* **Pure operator CR pass-through (no plugin CRs).** Rejected: the PSMDB inline `users[]` shape is not portable to PXC, and the user gets a different UX per engine. The whole point is a uniform model.
* **External identity provider only (LDAP/OIDC).** Rejected for v1: most production setups still need locally-defined service accounts for applications. SSO is additive, not a replacement (see §7).

## 7. Phased Roadmap

### Phase 1 — Direct adapters, core engines

CRDs, plugin backend, direct adapters for PSMDB / PXC / PG, credentials secret, UI tab, CLI. Rotation manual only.

### Phase 2 — Native adapters & scheduled rotation

PSMDB native adapter (patching `users[]` on the cluster CR, contingent on §8.6), PG native where supported, scheduled rotation, drift detection with opt-in auto-heal.

### Phase 3 — Identity federation

LDAP and OIDC adapters that delegate authentication to an external IdP. Requires engine support (PG supports both natively, PSMDB supports LDAP via `$external`, PXC requires the auth plugin). May surface new fields on `Provider` to declare which auth backends an engine supports. This is the deeper-integration phase the original motivation referenced.

### Phase 4 — Multi-cluster users & secret-store integration

One `DatabaseUser` targeting multiple clusters (replicated credentials). Optional external secret backends (Vault, AWS Secrets Manager, ESO) instead of in-cluster `Secret`s.

## 8. Open Questions

1. **Built-in role catalog.** Do we ship a small library of capability-bundles (`read`, `write`, `admin`, `readonly_*`) translated by adapters, or only support `raw` per-engine SQL? Recommendation: ship a minimal set (`read`, `write`, `admin`) plus `raw` escape hatch.
2. **Reserved username list.** Where does the canonical list live — in the plugin, or contributed per Provider? Probably per Provider, but plugin ships a default denylist.
3. **Rotation and application restart.** Some apps don't reconnect cleanly. Should the plugin support a grace period where both old and new passwords are accepted (engine-dependent — MongoDB supports it via `db.updateUser` with mechanism rotation, MySQL does not natively)?
4. **Conflict with operator-managed users.** Operators often create users (`monitor`, `xtrabackup`) the plugin must not touch. The denylist handles names, but what about a user a human creates outside the plugin and then declares a CR for? Adopt-on-import flag?
5. **Per-database scope on PG / MySQL.** `DatabaseUser` is namespace-scoped + cluster-scoped. Within a cluster, PG/MySQL have multiple logical databases. Is the role/grant scope expressive enough? `spec.scope.databases` is the current sketch — needs validation against real workloads.
6. **Mutating the provider's cluster CR for native adapters.** PSMDB (and likely PGO) manage users as an inline array on the same cluster CR that the spec-001 Provider owns. The plugin cannot get blanket write access to that CR — it would be free to change topology, versions, backup config. Options under consideration:
    * **Adapter contract on the Provider.** Extend spec 001 so a Provider declares an optional `userManagement` surface — a JSON Patch path (e.g. `/spec/users`) the host is willing to expose for plugin writes, plus a JSON-schema fragment for the value. The plugin submits patches to a new host endpoint `PATCH /v1/databases/{id}/provider-spec` that validates the patch is confined to the declared path and schema before applying it. Keeps the plugin out of the operator CR directly; keeps engine-specific knowledge in the Provider.
    * **Sub-resource on `Instance`.** Add a `users` sub-resource to the spec-001 `Instance` CR that the Provider's reconciler projects onto the underlying operator CR. Cleanest from the plugin's perspective (it writes a uniform shape) but pushes the engine-specific projection into every Provider that wants native user management.
    * **Direct adapter only for these engines.** Skip native entirely; always use the direct adapter for PSMDB and PGO. Loses the operator's idempotency and password-rotation guarantees.
   Recommendation leans toward the first option; needs a follow-up amendment to spec 001.
