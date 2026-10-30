# Multi-Cluster RBAC

*   **Status:** Draft
*   **Authors:** @recharte
*   **Created:** 2026-05-07
*   **Last Updated:** 2026-09-16
*   **Related Issues:** [openeverest#3009](https://github.com/openeverest/openeverest/issues/3009), [openeverest#3104](https://github.com/openeverest/openeverest/pull/3104)

---

## 1. Summary

OpenEverest v2 is multi-cluster (`/clusters/{cluster}/...`). This spec makes the Casbin RBAC system cluster-aware by embedding the cluster name in the hierarchical object of the 4-tuple model (`sub, res, act, obj`): objects evolve from `namespace/name` to `cluster/namespace/name`. It is the normative definition of v2 authorization: vocabulary, object arity and validation, enforcement semantics, failure modes, threat model, and the GA audit minimum.

## 2. Motivation

v1 objects have no cluster dimension — a second managed cluster would be implicitly open to every user. The v2 enforcement code predates this spec; its known gaps:

- Restores and the `/events` SSE stream enforce nothing.
- The plugin proxy's GET route is unauthenticated and forwards the caller's `Authorization` header to plugin backends, including arbitrary `externalUrl` hosts. ([#3104](https://github.com/openeverest/openeverest/pull/3104) fixed plugin *discovery*: list/context ride the generated chain with `plugins:read` post-filtering on `cluster/name` objects and a namespace-filtered context; the proxy still enforces the PoC `plugin/<name>` resource-string encoding from [#3009](https://github.com/openeverest/openeverest/issues/3009).)
- `namespaces` is enforced at the wrong arity (namespace lists come back empty under RBAC); `instance-presets` works only via a legacy fallback.
- v1's derived write-side checks were not ported, and v1's separate credentials permission was collapsed into `instances:read`.

### Current Architecture

1. **Casbin model** (`data/rbac/model.conf`) — 4-tuple; `globMatch` on res/act/obj; `g(sub, role)`. Kept, except the effect-expression fix (§4.6).
2. **Policy storage** — ConfigMap `everest-rbac` (`policy.csv`), hot-reloaded. A **pre-GA transitional** format; the GA store is an open decision (§4.11, §7.1).
3. **Handler chain** — Rate Limiting → JWT → **RBAC** → Validation → Kubernetes; enforced for the subject and each OIDC group. RBAC deliberately precedes validation: the validation handler dials request-supplied endpoints with request-supplied credentials.
4. **LIST post-filtering** of items failing `enforce()`.

## 3. Goals & Non-Goals

**Goals:** cluster-aware objects with the model's definitions unchanged; clusters gated as a resource; the complete normative vocabulary and object formats — every endpoint, including restores and `/events`; fail-closed validation and failure modes; v1's credential granularity and derived write checks preserved; threat model; the v1→v2 migration path; a GA audit minimum for privileged events (§4.12).

**Non-Goals:** v1 endpoint compatibility (removed); deny rules (allow-only, §4.6); policy sync to managed clusters; Kubernetes-native RBAC on managed clusters; per-cluster OIDC; full per-decision audit logging (deferred — §4.12 is the GA minimum).

## 4. Design

### 4.1 Object Format and Glob Semantics

The object is a `/`-separated path whose segment count (arity) is fixed by the resource's scope. Casbin's `globMatch` delegates to `doublestar.Match` (**not** Go's `path.Match`), which shares the one property this scheme depends on:

> **A single `*` matches within one path segment and never crosses `/`.**

doublestar's extra forms (`**`, `?`, `[…]`, `{…}`) are rejected in every term position (§4.6) — `**` has no arity and would defeat arity validation and §4.2's reserved-name exclusion. A characterisation test in `pkg/rbac` pins the dialect, so a Casbin upgrade that changes matching fails CI instead of silently rewidening every policy.

| Pattern | Matches | Does NOT match |
|---------|---------|----------------|
| `*` | `prod` | `prod/db`, `prod/ns/db` |
| `*/*` | `prod/db` | `prod`, `prod/ns/db` |
| `*/*/*` | `prod/ns/db` | `prod`, `prod/db` |
| `prod/*/*` | everything on the prod cluster | any 1- or 2-segment object |
| `*/dev/*` | the dev namespace on any cluster | — |

A policy whose object arity does not match its resource's scope **silently never matches** — it grants nothing. Validation therefore rejects arity mismatches (§4.6), and "access to everything" takes one line per tier (§4.9).

Cluster, namespace, and resource names are DNS-1123 Kubernetes names, so they cannot inject separators or patterns into objects — a requirement, not a convention: the OpenAPI declares DNS-1123 `pattern`s on the `cluster`/`namespace`/`name` path parameters, and object construction rejects empty segments (doublestar's `*` matches the empty string, the one non-fail-closed direction).

### 4.2 Resource Scopes

Every v2 resource has exactly one scope. The categorization is exhaustive and machine-enforced: resource-name generation and admin-policy generation MUST fail on an uncategorized resource instead of falling back to a default (§4.7).

| Scope | Object format | Wildcard | Resources |
|-------|--------------|----------|-----------|
| Global | `name` | `*` | `clusters` |
| Cluster-scoped | `cluster/name` | `*/*` | `namespaces`, `providers`, `backup-classes`, `instance-presets`, `plugins`‡ |
| Cluster+namespace | `cluster/namespace/name` | `*/*/*` | `instances`, `backups`†, `restores`†, `backup-storages`, `monitoring-configs`, `config-maps`, `secrets` |

† `backups` and `restores` use the **owning instance's name** as the last segment, not their own (§4.5).

‡ Cluster-scoped as of [#3104](https://github.com/openeverest/openeverest/pull/3104): plugins are advertised and enabled per managed cluster, so the leading segment has a real referent even though `Plugin` CRs live on the management cluster. The proxy must converge on the same `cluster/name` objects (§4.3).

**The slash convention.** A `/` in a *resource name* means “excluded from resource wildcards”: glob matching applies to the resource term too, so `res: *` never matches a slashed name — only the exact name or `<prefix>/*` does. The convention is normative **now**, ahead of any resource using it: any future resource that gates the security boundary itself (policy-management endpoints, for instance) MUST take a slashed name — an unslashed one would silently widen every existing wildcard policy on the upgrade that introduces it. Never add a slashed name cosmetically. The concrete management resource names are deliberately **not reserved yet**; they are settled together with the policy-store decision (§4.11, §7.1). Adding *any* resource to a tier widens wildcard policies on upgrade, and **re-tiering** one is a breaking policy change ([#3104](https://github.com/openeverest/openeverest/pull/3104) moved `plugins`) — both need release-note treatment.

Endpoints outside the resource model:

| Endpoint | Treatment |
|----------|-----------|
| `/events` (SSE) | Per-event filtering (§4.5) — currently unenforced; must be fixed |
| `/clusters/{cluster}/plugins/{name}/assets/*` | Serves the plugin's declared static bundle only, never proxied; may stay unauthenticated (bundle contents are public by declaration) |
| `/clusters/{cluster}/plugins/{name}/*` | Reverse proxy to the plugin backend: JWT + `plugins:use` on `cluster/name`, **every method, GET included**; inbound `Authorization`/`Cookie` stripped and replaced with a per-plugin credential (§4.8). *GET is currently unauthenticated with the caller's JWT forwarded — a live vulnerability this spec removes* |
| `/clusters/{cluster}/plugin-context` | Authenticated via the generated chain ([#3104](https://github.com/openeverest/openeverest/pull/3104)); namespaces filtered by `namespaces:read`. Advisory metadata only, never an authorization input (§4.8) |
| `/permissions`, `/cluster-info`, `/resources`, `/version` | RBAC skip list — authenticated, self-describing or non-sensitive |
| `/auth/token`, `/auth/revoke`, `/settings` | Unauthenticated by design (login bootstrap); `/settings` exposes only what the login page needs |

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

² The governance counterpart to `create`: platform teams publish vetted presets; preset-only users materialize them without the power to craft arbitrary instances. Creating an instance referencing a preset (`openeverest.io/instance-preset` annotation) requires `instance-presets:read`, then `instances:create` (customization allowed) or `create-from-preset` (exact materialization only). `create` is strictly stronger; `*` implies both. Preset *visibility* doubles as *deployability* — scope with name globs (`names: ["small-*"]`). The exact match covers `spec` plus an explicit **allowlist** of metadata keys — any other label or annotation fails the preset-only path (a denylist would miss behavior-bearing metadata). A mismatch 403 names the offending field paths, distinguishably from a missing-permission 403.

³ Every proxied request — every method, GET included — requires `plugins:use` on the plugin's `cluster/name`; `read` gates catalog visibility only ([#3104](https://github.com/openeverest/openeverest/pull/3104)); neither implies the other. **`plugins:use` is a reachability gate, not a data-access boundary**: the proxy forwards opaque plugin-defined paths, so the plugin's own calls back through the Everest API are the data boundary, and installing a plugin that holds its own credentials is a privileged act (§4.8). Docs must state this limitation where data-access plugins are evaluated; per-namespace scoping is post-GA (§7.4). The PoC `plugin/<name>` resource-string encoding — still live in the proxy — is removed; validation (§4.6) rejects it.

**Action terms are exact names or the literal `*` — no globs.** Prefix globs are traps: `read*` silently grants `read-connection`, and `create*` picks up every future `create-…` action on upgrade. Validation (§4.6) rejects anything else; docs show `read` (exact) as the read-only idiom.

`read-connection` restores v1's separate `database-cluster-credentials` permission: seeing an instance must not imply seeing its password. It is an action, not a resource, because both resource encodings are dominated: an unslashed resource is swept up by the ubiquitous `*, read, …` wildcard (v1 genuinely leaked credentials this way), while a slashed name drops out of `res: *`, so every hand-written full-control role would silently *lack* credential access. As an action it composes correctly: `read` never grants it, `act: *` always does.

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

**Migration is a documented manual rewrite.** An automated v1→v2 translator is **out of scope**: v2 is a new deployment (no in-place upgrades), v1 policies are small hand-authored files, and the semantic deltas below mean any generated output would need full human review anyway — revisit only if real migration demand appears. Two safety nets ship instead: the mapping above is a **machine-readable table in `pkg/rbac`** consumed by `everestctl settings rbac validate`, so a v1 resource name produces a targeted error naming its v2 successor and object arity (and a removed resource says so); and the upgrade docs carry this table plus the deltas below, prominently.

**Semantic deltas — a faithful transcription is *wider* than v1 enforced.** Docs must state these where the rewrite is described, since no tool prints them: v1 fanned instance *reads* out to referenced storages/monitoring configs, v2 does not (§4.5); credentials become an action (`instances:read-connection`); the cluster segment is new information the author must supply (never default it to `*`); dropped resources lose their grants.

### 4.5 Enforcement Semantics

*   **Enforce before fetch — on read paths.** Checks run before any backend lookup: uniform 403 whether or not the resource exists — no existence oracle. Two bounded exceptions: the flat `/backups/{name}` and `/restores/{name}` routes are **fetch-then-enforce** (keying by owning instance requires reading the object; the pre-fetch uses the server's identity, is never returned or leaked into the error body, and "not found" = "denied" = 403; the nested `/instances/{i}/backups` routes are canonical), and CREATE's `409 AlreadyExists` is an accepted name-collision side-channel.
*   **LIST post-filtering.** Unauthorized items are silently omitted (no 403). Filtered counts are never returned — a count is a cardinality oracle. An enforce error omits the item (deny-and-continue); single-object paths return 500 (§4.6).
*   **Writes fail loud.** CREATE/UPDATE/DELETE without permission returns 403 naming the missing `resource:action:object`.
*   **`/events` (SSE).** Every event is filtered under `read` before delivery — a subscription is not a bypass. The envelope gains a `cluster` field; each `events.Type` maps to exactly one `(resource, object)` pair or an explicit non-resource gate, in a table in `pkg/events` with a CI completeness check; backup/restore events carry the owning instance; `settings.updated` and `plugin.*` require the resource's `read`; `user.*` events (incl. `user.login_failed`, today broadcast to everyone) require `clusters:read` on `*`. An unmapped `Type` is **not delivered**.
*   **Backups and restores are keyed by their owning instance** (`cluster/ns/instance`): backup names are typically generated (`generateName`), one grant covers an instance's history, and it matches v1 and the nested routes.
*   **The cluster gate is explicit.** `clusters:read` gates only cluster visibility. Nested checks are self-sufficient — `instances:read` on `prod/dev/*` works without `clusters:read` on `prod` — but UI navigation effectively requires it, so roles should include it (`validate` warns when it's missing, §4.6).
*   **Derived checks run on writes only.** Writing a resource that references others requires read on the referenced resources. The general rule: **every `*Ref` in `InstanceSpec` requires `read` on the referenced resource**; current enumeration:

    | Operation | Additional checks |
    |---|---|
    | Create instance | `providers:read` on the provider; `backup-storages:read` on each storage; `secrets:read` on `userSecretRef`; `backups:create` (obj = instance) when schedules are declared; `restores:create` + `backups:read` (obj = source instance) when `dataSource` is set |
    | Update instance | Same checks, **delta-only** — only references added or changed by the request (else losing one grant permanently locks the instance). The update path has no derived checks today and must gain them. **PATCH**: compute the post-merge object (dry-run or in-process merge) before checking; if infeasible, reject PATCH for objects with cross-references |
    | Create from preset | `instance-presets:read`, then `instances:create` or `create-from-preset` (§4.3). The waiver covers **only references pinned by the resolved preset** (the preset author's authority — the Service Catalog launch-role pattern); requester-supplied references (e.g. `userSecretRef`) are never waived |
    | Create backup | `backups:create` (obj = instance) + `backup-storages:read` on the target storage + `backup-classes:read` on the referenced class (a class selects an execution identity — §4.8) |
    | Create restore | `restores:create` (obj = target instance) + `backups:read` (obj = source backup's instance) |

    Derived checks authorize the *submitted* object atomically (the Kubernetes `escalate`/`bind` shape) — no TOCTOU. Reads deliberately do **not** fan out to referenced resources: v1 did, making resources invisibly vanish from lists; a reference leaks only a name.
*   **Credentials.** `instances:read` never returns connection secrets; the connection endpoint requires `read-connection` (§4.3). The `secrets` resource gates Secret **metadata** only — the API strips `data`/`stringData` on every path; the only credential-disclosing endpoint is `/instances/{i}/connection`. (Over-restricting `secrets:read` breaks instance creation: it is also the `userSecretRef` derived check.)

### 4.6 Policy Validation & Failure Modes

**The model is allow-only, permanently.** The policy definition has no `eft` column — deny cannot be expressed — and the effect expression drops its inert deny clause: `e = some(where (p.eft == allow))`. Semantically identical, operationally significant: the two-clause form selects Casbin's `AllowAndDenyEffect`, which suppresses first-match short-circuit and forces a full policy scan on **every** `Enforce()`; the single-clause form returns at the first match. (`matchedBy[]` provenance in §4.10 is therefore computed from the resolved permission set, not the enforcer's explain path.) Request/policy/role definitions and the matcher are unchanged. Allow-only keeps “prove X cannot do Y” a *local* question — the absence of a matching allow — where deny makes it whole-policy reasoning (Kubernetes RBAC is additive-only for the same reason). If deny ever becomes necessary, it is a new policy-format major version, not an `eft` column.

**Validation** runs in `everestctl settings rbac validate` and on every ConfigMap (re)load. A policy is rejected when:

*   a `p` line does not have exactly 4 values, or a `g` line exactly 2;
*   a term uses an illegal pattern form. **Legal forms:** the only wildcard is a single `*` — `**`, `?`, `[…]`, `{…}`, `\` are rejected in every position. A resource term is a known v2 name or the literal `*` (slashed names, when introduced per §4.2, are matchable only exactly or as `<prefix>/*`); an action term is a bare action name or `*` (§4.3); an object term has exactly the tier's segment count, each segment a DNS-1123 name or `*`;
*   the object arity does not match the resource's scope — the silent-never-matches trap;
*   a term contains characters outside its position's set. The current pattern `^[/*-_:a-zA-Z0-9]+$` hides a `*-_` character-class *range* admitting `;` `<` `=` `>` `?` `@` `[` `\` `]` `^` — an **availability defect, not hygiene**: `[` passes validation, doublestar returns `ErrBadPattern`, and the enforce error aborts list handlers — one accepted line turns every request into a 500. Fix: per-position sets (subjects add `.`/`@`; other terms are lowercase alphanumerics plus `-`, `/`, `:`, `*`). Property-tested invariant: **no policy that passes validation can cause `Enforce` to return an error**.

An unbound role is a **no-op, not an error** (Kubernetes allows a `ClusterRole` without bindings, and GitOps-applied policy arrives in arbitrary order). The current `checkRoles`, which rejects the whole policy for one unbound role, is deleted.

`validate` additionally **warns** (not errors) on: grants without matching `clusters:read` (the empty-UI trap, §4.5); `backup-classes` write grants, including via `*` (the §4.8 escalation).

**`defaultRole` replaces the `enabled` flag.** `enabled: "false"` actually means *every identity that can authenticate is a full Everest admin* — most dangerous under SSO — and `IsEnabled` returns true only for the literal `"true"`, so a typo lands in everyone-is-admin: fail-open. Replaced by one key:

| `defaultRole` | Behavior |
|---|---|
| `role:admin` | Exactly the old `enabled: "false"` — but the config now says what it does |
| `role:viewer` | SSO onboarding: every authenticated user reads. **Single-tenant convenience only** — where the authenticating population spans tenants, use `""` and scoped group bindings |
| `""` (or unset) | Deny by default — the old `enabled: "true"` |

The migration is mechanical (`false` → `role:admin`, `true`/absent → `""`). The enforcer is always on, so `/permissions` is always truthful; `UserPermissions.Enabled` and the UI's “RBAC disabled” branch are removed. **The chart ships `defaultRole: ""`** with a specified bootstrap: a binding rendered into the active policy store from `server.rbac.bootstrapAdmins` (default: the local `admin` account; SSO-first installs name an IdP group) binds the built-in admin role, so a fresh install is administrable through the product. A Day-1 recipe ships in §4.9, post-install NOTES name the next command, and zero resolvable bindings are surfaced loudly (Kubernetes Event + `/cluster-info` degraded flag). `defaultRole` maps 1:1 to a binding on the synthetic subject `everest:authenticated`; the `everest:` prefix is rejected from token claims at parse time (§4.8). Permissive/audit mode is deliberately **not** shipped at GA — it would reintroduce the `enabled: false` fail-open as a supported flag, for a population (in-place upgrades) that does not exist; the §4.12 denial counter is the data that would justify it later.

**Failure modes are fail-closed:**

| Condition | Behavior |
|---|---|
| Reload yields invalid policy | Keep last-known-good and log. Never fail open, never wipe. |
| ConfigMap missing/unreadable at startup | Built-in admin policy only; all other subjects denied. |
| `defaultRole` missing or empty | Deny by default — misconfiguration can only narrow access. |
| Enforce errors on one item | LIST: item omitted. Single-object: 500. Prevented at source by the no-enforce-errors invariant. |
| Zero resolvable admin bindings | Kubernetes Event + `/cluster-info` degraded flag. |

These semantics apply to the ConfigMap artifact. A store with per-object granularity (the CR candidate, §4.11) narrows “invalid policy” to a per-object condition — rejected at admission or excluded via status — never poisoning the whole enforcer. The fail-closed principle is store-independent; only the granularity varies.

### 4.7 Admin Policy Generation

`loadAdminPolicy()` grants the built-in admin role `*` on every resource, at the wildcard arity of its scope. Requirements:

*   Generation MUST fail on an uncategorized resource instead of falling back to a default (`instance-presets` relies on such a fallback today; `plugins` was missing from `AllResources` entirely — admin lockout — until [#3104](https://github.com/openeverest/openeverest/pull/3104) added it, cluster-scoped).
*   `AllResources` is generated from the OpenAPI path map, which cannot surface resources without endpoints; if non-endpoint resources are ever introduced (e.g. reserved management names, §4.2), the generator must support hand-merging them into the list.
*   A CI check asserts `AllResources` ⊆ (Global ∪ ClusterScoped ∪ ClusterNamespaced) over the merged list. One tier table feeds `loadAdminPolicy`, the validator, and the 014 compiler.

### 4.8 Threat Model

| Boundary / vector | Position |
|---|---|
| Everest RBAC is the **sole tenant boundary** | The Everest service account is fully privileged on managed clusters; nothing below this layer separates tenants. |
| ConfigMap write = RBAC root | Protect `everest-rbac` with Kubernetes RBAC; manage policy via GitOps for review and rollback. |
| IdP claims are trusted input | Group claims are Casbin subjects and `g(x, x)` identity-matches — an OIDC group named `role:super-admin` would inherit that role; reserved-prefix rejection is the *only* thing separating subjects from roles. `role:`/`everest:` claims are rejected at every subject source — OIDC `sub`, OIDC `groups` (every value), local-account sessions, plugin daemon tokens — applied to the *resolved* subject (after the internal issuer's colon-truncation), plus at local-account creation. `everest:authenticated` is appended inside the enforcer wrapper, after rejection. Table test over all four sources; docs call out IdP group hygiene. |
| Users and Groups share one Casbin namespace | A local account named like an IdP group (or vice versa) silently inherits its roles — with self-service IdP group creation, an escalation. Subject-kind prefixing (`user:`/`group:`, designed in draft spec 014 §4.3) closes this structurally; until it lands, operators MUST ensure the IdP forbids self-asserted group values. |
| `backup-classes:create/update` is not a routine grant | `BackupClass.spec.job.*.permissions`/`clusterPermissions` are arbitrary `rbacv1.PolicyRule`s minted into a job ServiceAccount (the manager holds `escalate`/`bind` for this) — effectively **cluster-admin on the targeted managed cluster**, executed via `backups:create`/`restores:create`. Treat as privileged; `validate` warns on it (§4.6). Bounding the primitive is a separate workstream. |
| Object injection | Impossible — all segments are DNS-1123 names, enforced per §4.1. |
| Plugin backends | Proxy-only after JWT + `plugins:use`, every method (§4.2); `Authorization`/`Cookie` stripped for a per-plugin credential — forwarding the caller's JWT (especially to `externalUrl`) is credential exfiltration. External backends are an egress boundary; a plugin with its own credentials is bounded only by its own authorization — installing one is privileged. `plugin-context` is advisory, never an authorization input, and must not disclose the caller's group list. Plugin daemon tokens are hard-denied writes to core resources — the denylist is re-keyed to the v2 vocabulary, applied at one chokepoint the proxy also passes through, and fails closed on unknown names (today: keyed to v1 constants and bypassed by the proxy). |
| Delegated policy write = latency amplification | Policy size multiplies every enforce call, so a delegated policy author can degrade authorization globally. Any store offering delegation must cap compiled policy size (the CR candidate does — draft 014 §4.10). |
| `clusters` write actions (future) | Cluster registration attaches remote-cluster credentials — a `backup-classes`-class grant. Read-only today; priced now so the endpoints don't land ungated. |
| Management API (planned) | Policy-editing endpoints need a privilege-escalation guard (grant only what you hold — `escalate`/`bind` semantics) and a last-admin lockout guard. |
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
# NOTE: *-on-resources includes backup-classes writes — effectively cluster-admin
# on the managed cluster (§4.8). Enumerate resources if that is not intended.
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

Multi-tenant isolation (per-customer clusters) is the prod-admin pattern with a different cluster name per tenant; namespace tenancy on a shared cluster is the dev-team pattern.

**Day 1.** Deny-by-default ships with a recipe: the chart's bootstrap-admin binding (§4.6), a copy-paste docs bundle (bind the IdP admin group; optionally bind `everest:authenticated` to `role:viewer` — single-tenant only, no `read-connection`; grant one team a namespace), post-install NOTES naming the next command, and `everestctl settings rbac init --template editor|namespace-admin|deployer` emitting commented policy scaffolds — in the active store's format — to name, review, and commit.

### 4.10 Permissions Discovery & UI

Three contracts, all specified in OpenAPI now (even where implementation lands post-GA) — the UI must never parse raw tuples, because two matcher implementations in two languages will drift:

*   **`GET /permissions`** — *resolved, structured* permissions per resource/action/object-pattern, each carrying the granting role (provenance). **Action terms are always expanded, never a literal `*`** (so `read-connection` inside a wildcard grant is visible; action axis only — expanding objects would be an enumeration oracle). Same expansion in `rbac can`/`validate --dry-run`. A zero-grant result is explicit; the UI renders a dedicated "no access — ask your administrator" empty state, not a generic "nothing here". Optional `?cluster=&namespace=`; short-TTL caching is fine (advisory).
*   **`POST /self-access-reviews`** — any authenticated user; a batch of `(resource, action, object)` triples (`maxItems: 256`, 400 over-cap) returning per-triple decisions: one call per rendered page decides every button, no client-side matcher. Scope views can't express per-item grants; embedding `allowedActions` in list items breaks the no-DTO rule. Implementation: resolve the caller's permission set **once**, match locally (`BatchEnforce` is a bare loop — not a mitigation, §7.3); the local matcher is the same `GlobMatch`, differential-tested against `Enforce`.
*   **`POST /access-reviews`** — admin-gated, same bound; evaluates *any* subject, returns `{allowed, matchedBy[]}` — the "why is this denied" answer. The self/any split mirrors `SelfSubjectAccessReview`/`SubjectAccessReview`: reviewing others is itself a permission.

No “RBAC disabled → show everything” UI branch: the enforcer is always on (§4.6). Migration off the UI's in-browser `casbin.js` matching (~40 files) is phased **in work, not contract**: `/self-access-reviews` first, page-by-page; `/permissions` changes shape once (own client, no GA install to strand); “no `casbin.js` import in `ui/`” is the exit criterion.

### 4.11 Policy Store

**The GA policy store is an open decision** (§7.1). The ConfigMap `policy.csv` is the current, pre-GA transitional store and is never documented as the GA interface. The leading candidate is Kubernetes CRs (`AccessRole`/`AccessBinding`), fully designed in **draft spec 014** — structured schema with compiler-owned arity, per-object failure granularity, delegation via Kubernetes RBAC; the alternative is hardening the ConfigMap, whose costs (1MiB ceiling, whole-file failure model, no delegation) are catalogued in 014 §2.

Enforcement is deliberately storage-agnostic: handlers call `enforce()` and the store sits behind the adapter, so nothing else in this spec depends on the outcome — the vocabulary, enforcement semantics, and failure principles bind any store.

### 4.12 GA Audit Minimum

Full per-decision audit logging stays deferred (§7.2). Four cheap, bounded items ship at GA:

*   **Every successful `read-connection`** logs one structured line `{ts, subject, cluster, namespace, instance}` — one low-frequency endpoint, no LIST amplification; otherwise credential *use* is untracked.
*   **Structured denial logs on single-object and write paths** with the missing `resource:action:object`. Per-item LIST denials are excluded — counted, not logged (the per-row warning logged today is a defect this fixes).
*   **Policy-change auditability** = a documented Kubernetes audit-policy stanza for writes to the policy-store objects (the ConfigMap today; CRs if the 014 candidate is adopted), with approver/diff/revert via GitOps.
*   Prometheus counter `everest_rbac_denials_total{resource,action}` on every deny — turns deny-by-default friction from anecdote into data.

## 5. Definition of Done

**Model & vocabulary:**
*   `model.conf`: request/policy/role definitions and matcher unchanged; effect expression reduced to `some(where (p.eft == allow))` (§4.6) — benchmark regression on 5,000 rules + decision-equality test over the CSV corpus.
*   Characterisation test pinning doublestar matching semantics (§4.1).
*   Every resource categorized (§4.2) with a CI completeness check; `instance-presets` categorized, the `default:` fallback removed (§4.7); one tier table for all consumers.
*   The slash convention documented as normative for future security-boundary resources (§4.2).
*   Dead v1 constants removed, with `plugin_denylist.go` re-keyed to v2 in the same change (§4.8) — tested per resource, fails closed, ⊆ `AllResources` (CI).

**Enforcement:**
*   Every §4.2 endpoint enforces — including restores (handler gains the cluster parameter) and `/events` per the §4.5 mapping table. CI keeps route registration and the enforcement map in sync (`/events` and the proxy remain hand-registered — how [#3009](https://github.com/openeverest/openeverest/issues/3009) happened).
*   Connection endpoint: `read-connection` + the §4.12 log line; `instances:read` never returns credentials.
*   Derived checks (§4.5): instance create and update (delta-only; PATCH post-merge), backups (incl. `backup-classes:read`), restores; backups/restores keyed by owning instance (flat routes fetch-then-enforce); preset waiver limited to preset-pinned references.
*   `create-from-preset` exact-match = `spec` + metadata allowlist; mismatch 403s name field paths; action renamed from `deploy` in code.
*   Plugin proxy: JWT + `plugins:use` on `cluster/name`, every method; bundles from the declared asset path only; `Authorization`/`Cookie` stripped; the PoC `plugin/<name>` string and `ActionAll` shortcut removed. (Catalog/context filtering: [#3104](https://github.com/openeverest/openeverest/pull/3104).)
*   Uniform 403 before fetch on reads; LIST post-filtering with deny-and-continue; namespace listing works for non-admins (2-segment objects).
*   One shared enforcer accessor (handler chain, plugin proxy — today a second independent enforcer — and `/permissions`); reloads swap a fresh enforcer via `atomic.Pointer`, never `LoadPolicy` in place; informer handles Add/Update/Delete.

**Validation, tooling & failure modes:**
*   Pattern-form/arity/vocabulary validation (§4.6) in `validate` and at load; `can` handles all three arities; per-position charsets; the no-enforce-errors property test.
*   `validate`: targeted v1-name errors from the machine-readable mapping (§4.4); warns on missing `clusters:read` (the empty-UI trap) and on `backup-classes` write grants (§4.8); prints its vocabulary version.
*   `defaultRole` replaces `enabled`; the `IsEnabled` fail-open, `UserPermissions.Enabled`, and the UI disabled-branch are gone.
*   Invalid reload keeps last-known-good (tested).
*   Reserved-prefix rejection at all four subject sources + local-account creation (table-tested, §4.8); synthetic group injected post-rejection.
*   The §4.12 audit minimum implemented.

**Tests & docs:**
*   Unit tests per RBAC handler; e2e authorization matrix (resources × actions × allow/deny); install-then-login e2e for the bootstrap admin.
*   The three §4.10 contracts in OpenAPI; no client-side matching — “no `casbin.js` import in `ui/`” is the exit criterion.
*   Docs: v2 vocabulary, §4.4 table + semantic deltas, Day-1 bundle (§4.9), group-binding onboarding with the `role:viewer` multi-tenant caveat, `backup-classes` and `plugins:use` warnings (§4.8).
*   Helm ships `defaultRole: ""` + bootstrap-admin binding + NOTES (§4.6).
*   GA docs present the decided policy store (§7.1) as the policy interface; `policy.csv` is pre-GA only.
*   `make test` and `make check` pass.

## 6. Alternatives Considered

**Casbin domains (`g = _, _, _`)** — native multi-tenancy with per-domain role assignments. Rejected: breaks every existing `g` line, domain matching doesn't use `globMatch` by default (poor cross-cluster wildcards), and `GetImplicitPermissionsForUser` doesn't resolve cross-domain roles without extra configuration.

**5th tuple field (`r = sub, res, act, obj, cluster`)** — cluster as an explicit dimension. Rejected: touches the model, every policy, every enforce call site, and migration tooling, for the same expressiveness the hierarchical object already provides.

**Per-cluster RBAC deployments** — an enforcer and policy store per managed cluster. Rejected: no centralized management, cross-cluster roles impossible, `/permissions` would need cross-enforcer aggregation.

**Custom matcher where `*` crosses `/`** — rejected: doublestar already supplies `**` for this and validation bans it (§4.6); a friendlier wildcard hides scope mistakes that arity validation surfaces loudly. (Argo CD is the cautionary example, not precedent: its glob does not treat `/` as a separator, and its docs must tell authors to "always use four slashes" — unenforced discipline where arity validation enforces it.)

## 7. Open Questions

1.  **Policy store decision** (§4.11). The CR store (draft spec 014) is the leading candidate, **not a commitment**. Decision inputs: the compiler prototype (property tests + CSV-equivalence) as the feasibility artifact, and product runway. If CRs are adopted post-GA rather than at GA, the guardrails are: fail-loud CSV/CR coexistence, migrate shipping in the flipping release, one dual-read release, tuple-set verification. The reserved management resource names (§4.2) and subject-kind prefixing (§4.8) are settled in the same decision. Nothing else in this spec depends on the outcome.
2.  **Full audit logging** — deferred (LIST volume); §4.12 is the GA minimum.
3.  **Performance at scale** — post-filtered LIST is O(items × subjects × policies). Mitigations, in order: first-match short-circuit (§4.6); resolve-once-match-locally (§4.10); compiled-policy caps if the CR store is adopted (draft 014 §4.10). Benchmarks (5,000 tuples × 500 items) gate GA.
4.  **Per-namespace plugin scoping** — `plugins:use` is a reachability gate (§4.3); requires plugin cooperation; post-GA.

## 8. References

**Industry RBAC Patterns:**
*   [Casbin globMatch](https://casbin.org/docs/function#globmatch) — delegates to [`doublestar.Match`](https://github.com/bmatcuk/doublestar) (v4); single `*` never crosses `/`; extended forms banned by §4.6.
*   [ArgoCD RBAC](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/) — glob-scoped objects (no `/` separator; see §6).
*   [Rancher RBAC](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/authentication-permissions-and-global-configuration/manage-role-based-access-control-rbac) — global + cluster + project roles.
*   [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) — `Role` vs. `ClusterRole` scope separation.

**OpenEverest Documentation:**
*   [OpenEverest v2 API spec](openeverest/api/openapi/http-api.yaml) — `/clusters/{cluster}/...` endpoint definitions.
*   [Core API types](openeverest/api/core/v1alpha1/) — `Instance`, `Provider` CRD definitions for the v2 API.
*   [Plugin Architecture spec](001-plugins-architecture.md) — how providers and plugins work in OpenEverest.
