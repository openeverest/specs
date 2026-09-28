# Generic Plugins — Architecture Design

*   **Status:** Draft
*   **Authors:** @spron-in
*   **Created:** 2026-04-09
*   **Last Updated:** 2026-09-28
*   **Related Issues:** [openeverest#2661](https://github.com/openeverest/openeverest/issues/2661) (plugin UI design), [openeverest#3076](https://github.com/openeverest/openeverest/pull/3076) (implementation)
*   **Related specs:** [001 — Modular core / Provider plugins](./001-plugins-architecture.md)

---

## 1. Summary

Spec [001](./001-plugins-architecture.md) introduced a plugin system for provisioning and lifecycle management of databases backed by Kubernetes operators. This document designs the complementary **generic plugin** layer — self-contained, installable extensions that consume OpenEverest's existing databases and data without managing them. Generic plugins can contribute UI pages, sidebar entries, database-detail panels, CLI subcommands, and backend API logic, all without rebuilding or redeploying the OpenEverest core.

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

## 3. Terminology

| Term | Meaning |
|---|---|
| **Host** | The OpenEverest core: API server, React web UI shell, and `everestctl`. |
| **Plugin** | A generic extension authored by anyone, distributed as an OCI artifact. |
| **Manifest** | The `Plugin` Kubernetes CR that describes a plugin. Always required. |
| **Backend** | An optional HTTP service that implements custom logic on behalf of the plugin. |
| **Frontend bundle** | An optional ESM JavaScript module loaded at runtime into the web UI shell. |
| **Extension point** | A named, typed slot in the host UI or CLI where a plugin registers a contribution. |
| **InstalledExtension** | A cluster-scoped CR that records an installed extension (a generic plugin or a spec 001 provider). Per-cluster plugin visibility is governed by Everest RBAC, not by fields on this CR. |
| **Infrastructure plugin** | A generic plugin that creates and manages its own Kubernetes resources (Deployments, Services, ConfigMaps) in response to lifecycle events. Its `ServiceAccount`, `Role`/`ClusterRole`, and `RoleBinding`/`ClusterRoleBinding` are shipped by the plugin bundle itself (Helm chart); the host does not generate plugin RBAC. |
| **Stateful plugin** | A generic plugin that declares custom resource schemas in its manifest. The host installs the CRDs, watches instances via a dynamic informer, and routes reconciliation events to the plugin backend over HTTP. The plugin does not run its own operator. |
| **Plugin CRD** | A `CustomResourceDefinition` declared by a stateful plugin under the `plugins.openeverest.io` API group. Installed, validated, and watched by the host; reconciled by the plugin backend via webhook-style HTTP callbacks. |

## 4. Goals & Non-Goals

**Goals:**

* Define what a generic plugin is and how it differs from a spec 001 Provider plugin.
* Design concrete manifest schema, CRDs, and lifecycle mechanics.
* Define the UI extension-point taxonomy and the frontend SDK surface.
* Define how plugin backends are hosted, reached, and authenticated.
* Integrate with the existing RBAC / auth model without introducing new auth concepts.
* Establish a clear security model for both frontend bundles and backend services.
* Provide a phased roadmap so v1 is deliverable quickly.

**Non-Goals:**

* Replacing or modifying the spec 001 Provider model.
* Defining a plugin marketplace or catalog UI.
* Tooling for plugin development (scaffolding, testing framework).
* Full CRD YAML schemas or SDK implementation code — those belong in follow-up issues.

## 5. Illustrative Use Cases

| Use Case | Backend? | Frontend? | External access needed? |
|---|---|---|---|
| SQL query browser (DBeaver-like) | Yes (query runner) | Yes (UI editor) | No — talks to in-cluster DBs |
| AI data copilot | Yes (LLM integration) | Yes (chat UI) | Yes — calls external LLM APIs |
| AWS RDS discovery | Yes (AWS API poller) | Yes (import/list UI) | Yes — calls AWS APIs |
| Data migration tool | Yes (migration job runner) | Yes (progress UI) | Possibly |
| Compliance / audit plugin | Yes (policy engine) | Yes (report UI) | Possibly |
| Read-only metrics dashboard | No | Yes (pulls from Prometheus) | Yes — calls monitoring APIs |
| CLI-only backup reporter | No | No (just `everestctl` commands) | No |

## 6. Plugin Anatomy

A plugin has three optional parts. The **manifest** is always required; the other two are present only when the plugin actually needs them.

```
plugin/
├── manifest.yaml        # Plugin CR, the single source of truth
├── main.js              # (optional) Frontend ESM bundle
└── server               # (optional) Backend binary / container image
```

### 6.1 Manifest

The manifest is a Kubernetes `Plugin` CR (cluster-scoped). It declares:

- Identity: name, version, vendor, description, icon.
- Frontend: URL or OCI/ConfigMap reference to the JS bundle.
- Backend: in-cluster `Service` reference or external HTTPS URL + credentials secret.
- Extension-point registrations (which slots in the UI/CLI the plugin fills).
- RBAC permissions the plugin requires against the OpenEverest API. Kubernetes RBAC (Roles, RoleBindings, ServiceAccount) is shipped by the plugin's Helm chart, not declared in the manifest.
- CLI contribution (optional).
- Compatibility range: which OpenEverest host versions this plugin supports.

### 6.2 Backend (optional)

Any HTTP service. The host never calls it directly from the browser — all traffic goes through the OpenEverest API proxy. Two hosting modes:

- **In-cluster**: a `Service` name + port in a declared namespace.
- **External**: an HTTPS URL with credentials stored in a `Secret` referenced by the plugin's installation (chart values / install-time arguments).

A backend can behave in one or more **behavioral patterns**, depending on what the plugin's code does:

- **Request handler** — responds to HTTP traffic proxied from the UI / `everestctl`. This is the default pattern for interactive plugins (SQL browser, AI copilot). Any plugin that declares `serviceRef` or `externalUrl` is implicitly a request handler.
- **Daemon** — runs continuously in the background with no inbound user traffic. Used for metering, periodic syncs, scheduled reports, AWS RDS discovery pollers. The plugin's container code decides whether to run as a daemon; the host does not manage this.
- **Event consumer** — holds an open subscription to the host's lifecycle event stream and reacts to resource changes (cluster created, deleted, backup completed, etc.). Used for audit, billing, external-system synchronisation. This is *pull-based*: the plugin opens the stream; the host does not push outbound HTTP to the plugin.

These patterns are not mutually exclusive: a billing plugin typically combines all three — a daemon to roll up usage, a held-open event subscription to capture lifecycle, and a request-handler endpoint to serve the in-UI invoice page.

### 6.3 Frontend bundle (optional)

A single ESM JavaScript file. It exports a `register(api)` function that calls `api.registerExtension(point, component)` to fill extension points. The React shell dynamically imports it at startup after fetching the enabled-plugins list from `GET /v1/clusters/{cluster}/plugins`.

The bundle shares only React with the host (through a browser import map) and bundles its own UI library. A plugin that uses MUI can look native with no extra work by theming it from the host's `--everest-*` design tokens via `@openeverest/plugin-theme`. See §8.1 and §9.3–§9.5.

### 6.4 CLI contribution (optional)

A container image (can be the same image as the backend) that the plugin exposes as a CLI subcommand through `everestctl`.

## 7. CRD Sketches

### 7.1 `Plugin` (cluster-scoped)

```yaml
apiVersion: extensions.openeverest.io/v1alpha1
kind: Plugin
metadata:
  name: sql-explorer
spec:
  displayName: "SQL Explorer"
  description: "Query and browse your databases directly from the OpenEverest UI."
  version: "1.2.0"
  compatibleHostVersions: ">=2.0.0 <3.0.0"        # host application version (API gate)
  compatibleUiContractVersions: ">=18.0.0 <19.0.0" # shared React major (UI gate, §9.5)
  vendor: "Acme Corp"
  icon: "https://example.com/icon.png"   # or omit for default

  # Frontend contribution (optional).
  # Exactly one of 'bundleUrl', 'bundleConfigMapRef', or 'bundleOciRef' must be set.
  frontend:
    bundleOciRef: "ghcr.io/acmecorp/sql-explorer-ui:1.2.0"
    # SRI hash is mandatory when bundleUrl is used; recommended otherwise.
    bundleIntegrity: "sha384-<hash>"
    extensionPoints:
      - type: route
        name: sql-explorer
        path: /sql-explorer              # rendered at /plugins/sql-explorer
        label: "SQL Explorer"
        icon: "database"
      - type: sidebarItem
        label: "SQL Explorer"
        icon: "database"
        routeName: sql-explorer
      - type: clusterDetailTab
        label: "Query"
        component: ClusterQueryTab       # exported name in the bundle
        # Optional: limit to specific database engine types.
        # Valid values: "postgresql", "psmdb", "pxc".
        # Omit to show for all engine types.
        providers: ["postgresql"]

  # Backend contribution (optional).
  backend:
    # In-cluster mode:
    serviceRef:
      namespace: everest-plugins
      name: sql-explorer-svc
      port: 8080
    # External mode (use instead of serviceRef):
    # externalUrl: "https://sql-explorer.saas.example.com"
    # credentialsSecretRef: "sql-explorer-creds"

  # RBAC: what OpenEverest API resources this plugin needs to call.
  permissions:
    - verb: read
      resource: database-clusters
    - verb: read
      resource: database-cluster-connection-details

  # CLI contribution (optional).
  cli:
    image: "ghcr.io/acmecorp/sql-explorer-cli:1.2.0"
    subcommand: "sql-explorer"
    description: "Interact with SQL Explorer from the terminal."
```

Kubernetes RBAC the plugin's `ServiceAccount` requires (e.g., `apps/deployments`,
`core/services`) is **not** declared in this CR. It is shipped as standard
`Role`/`ClusterRole` + `RoleBinding`/`ClusterRoleBinding` manifests inside the
plugin's Helm chart and applied at install time (see §10.7). Trust is anchored
at the plugin hub: signed/curated bundles get installed; everything else is
rejected at install time.

#### Canonical name

`metadata.name` is the plugin's canonical identity. It is referenced as a
global key by the proxy path (`/v1/clusters/{cluster}/plugins/{name}/...`), Everest RBAC
(the cluster-scoped `plugins` resource, object `{cluster}/{name}`, §11.2), frontend extension-point routes (`/plugins/{name}`),
`InstalledExtension.spec.plugin.pluginCRName`, the CLI subcommand, and the API group
of any `customResources` (`<name>.plugins.openeverest.io`, §10.8). It must
equal the chart's identity — see §10.7 for the chart-side rule. The host
rejects `Plugin` CRs whose name does not match `^[a-z][a-z0-9-]{0,62}$` or
whose name collides with an existing `Plugin` CR.

### 7.2 `InstalledExtension` (cluster-scoped)

A single cluster-scoped CR records the install state of an extension — either
a generic plugin or a spec 001 provider. It records install metadata only;
per-cluster plugin visibility and user access are governed by Everest RBAC
(see §11), not by fields on this CR.

The CR is created by `everestctl extension install` (never by the user
directly editing YAML). Cluster-admin owns it.

```yaml
apiVersion: extensions.openeverest.io/v1alpha1
kind: InstalledExtension
metadata:
  name: sql-explorer
spec:
  type: plugin                       # provider | plugin
  catalogId: openeverest-official    # optional; empty for manual installs
  channel: stable                    # optional
  version: "1.2.0"
  chartDigest: "sha256:..."          # optional in early phases

  plugin:                            # required when type=plugin
    pluginCRName: sql-explorer       # references the Plugin CR
    frontendDigest: "sha256:..."     # optional
    backendImageDigest: "sha256:..." # optional

  # Mutually exclusive with spec.plugin.
  provider:                          # required when type=provider
    providerName: percona-server-mongodb

status:
  phase: Installed                   # Installed | Upgrading | Failed | Uninstalling
  conditions:
    - type: Ready
    - type: BundleServed             # plugin-only
    - type: BackendReachable         # plugin-only
    - type: TokenIssued              # plugin-only, daemon mode
    - type: CRDsInstalled            # plugin-only, stateful plugins
    - type: ProviderRegistered       # provider-only
  installedAt: "2026-05-08T12:00:00Z"
  availableUpgrade:
    version: "1.3.0"
    chartDigest: "sha256:..."
```

## 8. UI Extension Points

Each extension point is a named, versioned slot in the React shell. The host exports the full taxonomy from `@openeverest/plugin-sdk` as TypeScript types so plugin authors get compile-time safety.

| Extension point | Where it appears | Props passed to component | Provider filter |
|---|---|---|---|
| `route` | Top-level React Router route at `/plugins/{name}/{path}` | `{ pluginName, params }` | — |
| `sidebarItem` | Main navigation sidebar | `{ navigate, currentPath }` | — |
| `clusterDetailTab` | Extra tab on a `DatabaseCluster` detail page | `{ cluster, namespace }` | ✓ |
| `clusterAction` | Context-menu action in the clusters table | `{ cluster, namespace, onClose }` | ✓ |
| `clusterCard` | Widget card on the cluster overview | `{ cluster, namespace }` | ✓ |
| `instanceCreateFormSection` | Collapsible section in the create-instance wizard | `{ formValues, onChange, namespace }` | ✓ |
| `instanceEditFormSection` | Collapsible section in the edit-instance page | `{ instance, formValues, onChange, namespace }` | ✓ |
| `globalDashboardWidget` | Card on the home / dashboard page | `{ namespaces }` | — |
| `settingsPanel` | Tab inside the Settings page | `{ currentUser }` | — |
| `themeOverride` | MUI theme override (logos, palette) | `{ defaultTheme }` — Phase 4 | — |

Extension points are **additive** — a plugin can register for multiple points. A host version that does not recognise a point silently skips it.

#### Provider filtering

Extension points that render in a database context (`clusterDetailTab`, `clusterAction`, `clusterCard`) support an optional `providers` filter. When declared, the host only renders the contribution for clusters whose `spec.engine.type` matches one of the listed values. Valid values are `"postgresql"`, `"psmdb"` (MongoDB), and `"pxc"` (MySQL).

The filter is expressed in two complementary places:

- **Plugin CR** (`spec.frontend.extensionPoints[].providers`) — documents the intent in the manifest; the value is forwarded by `GET /v1/clusters/{cluster}/plugins` to the frontend shell.
- **Bundle registration** (`registerExtension` call, `providers` field on the extension object) — the runtime gate; the host skips rendering the component if the current cluster's engine type is not in the list.

Omitting `providers` (or leaving it empty) means "show for all engine types". Existing plugins that do not set the field are unaffected.

#### Instance creation / edit form sections

The `instanceCreateFormSection` and `instanceEditFormSection` extension points let a plugin inject a collapsible configuration section into the instance creation wizard and the instance edit page respectively. This enables infrastructure plugins to let users opt in to plugin-managed features (e.g., "Enable ProxySQL") and configure them (exposure mode, resources, custom config) as part of the normal instance lifecycle.

**Data flow:**

1. The plugin registers a React component via `registerExtension({ type: 'instanceCreateFormSection', ... })`. The component receives `formValues` (current form state) and an `onChange(pluginConfig)` callback.
2. The user fills in the plugin section. The host collects the plugin config as an opaque JSON blob keyed by plugin name.
3. On form submission the host includes the plugin configs in a `POST` to the plugin backend: `POST /v1/clusters/{cluster}/plugins/{name}/instance-config` with `{ instance, namespace, config }`. The plugin backend stores the config (e.g., as a `ConfigMap` or in its own state) and acts on it — creating Deployments, Services, etc.
4. The host does **not** store the plugin config on the `Instance` CR. The plugin owns its own state. The host is only a messenger between the UI form and the plugin backend.

**Props:**

| Prop | Type | Description |
|---|---|---|
| `formValues` | `Record<string, unknown>` | Current form state (read-only snapshot). |
| `onChange` | `(config: Record<string, unknown>) => void` | Callback to update the plugin's config section. |
| `namespace` | `string` | Target namespace for the instance. |
| `instance` | `Instance \| undefined` | The existing instance (edit mode only; `undefined` during create). |

### 8.1 UI/UX consistency

Plugins **should** look and feel like the core, so plugin-contributed UI sits
naturally next to core UI. This is a recommendation, not a gate: a plugin that
looks different is still a valid plugin. The value is on the author's side —
with the host's theming bridge, a plugin gets the core palette, typography and
live light/dark switching without building any of that itself.

Plugins **must**, however, keep working when the host upgrades its UI stack
(MUI, theme internals) without being rebuilt, and must never break the host or
other plugins. The model below
([openeverest#2661](https://github.com/openeverest/openeverest/issues/2661))
serves both by sharing only React and design tokens.

**Delivery model:**

| Owned by | What | How it reaches the plugin |
|---|---|---|
| Host | React runtime (`react`, `react-dom`, `react/jsx-runtime`) | Browser import map resolving to the host's single React instance (§9.3) |
| Host | Design tokens: palette, typography, corner radius, light/dark | `--everest-*` CSS variables on `:root` and `data-everest-color-scheme` on `<html>` (§9.4) |
| Plugin | MUI, Emotion, icons, any other UI library | Bundled into the plugin at the version the plugin pins |
| Plugin | Its MUI theme | Built at runtime by `PluginThemeProvider` from `@openeverest/plugin-theme`, reading the host tokens |

Consequences:

- The host can upgrade MUI or change its theme internals; already-built plugins
  keep working and keep following the host palette and dark mode.
- Each plugin upgrades MUI on its own schedule. The only runtime shared with the
  host is the React major — the **UI contract** (§9.5).
- Each plugin downloads its own MUI and Emotion (roughly 75–110 kB gzip per
  bundle). This is the price of upgrade independence (see §19, bundle size).

**Rules every plugin must follow** — they protect the host and other plugins,
whatever UI library the plugin uses:

- **No global CSS.** Do not import `.css` files that produce global selectors,
  add `CssBaseline` or `GlobalStyles`, inject a CSS reset or normalise
  stylesheet, write custom properties to `:root`, or manipulate
  `document.styleSheets`. The host owns document-level styles. A third-party
  library's own stylesheet whose selectors are all scoped under that library's
  classes (e.g. `.react-flow`) may be inlined as a `<style nonce={api.cssNonce}>`
  inside the plugin tree, because library-mode builds emit no CSS file the host
  would load.
- **No overrides of host elements**, including `!important` rules targeting
  host markup.
- **Stay inside the mount point.** Do not append to `document.body` or other
  host DOM directly; overlays go through a portal (MUI `Portal`, or MUI
  components that portal such as `Dialog`, `Menu`, `Tooltip`).
- **Isolated, CSP-compliant styles.** Styles are injected under a plugin-unique
  Emotion cache key and carry `api.cssNonce` (`PluginThemeProvider` does both,
  §9.4).
- **Depend only on the public contract.** Do not read the host's `--mui-*`
  variables or rely on the host's React theme context; they are MUI internals
  of the host and change on a host MUI upgrade. `--everest-*` is the stable
  contract.
- **Do not enable MUI `cssVariables`** in a plugin theme. It writes `--mui-*`
  variables to `:root`, where they collide with the host's.
- **Bundle rules** in §9.3 (React external, one copy of each UI library).

**Recommended for a native look** — each of these saves the author work:

- **MUI with `@openeverest/plugin-theme`.** Build UI with `@mui/material` and
  wrap every registered component in `PluginThemeProvider` (§9.4). The plugin
  then follows the host palette, typography, radius and light/dark mode with no
  code of its own. Avoid wrapping plugin UI in your own `ThemeProvider` /
  `createTheme()`: the palette would stop following the host.
- **`sx` prop and `styled()` for styling.** They go through the plugin's own
  Emotion cache. A second CSS-in-JS runtime (styled-components, Stitches, a
  Tailwind runtime) adds bundle size and specificity conflicts.
- **Design tokens over magic values.** Reference `theme.palette`,
  `theme.spacing`, `theme.typography`, and `theme.shape` rather than
  hard-coding pixel values, hex colours, or font families, so the plugin
  respects light/dark mode and future theme changes.
- **Using a different UI library?** The `--everest-*` variables and
  `data-everest-color-scheme` are plain CSS, so any library, chart or canvas can
  follow the host colours and mode (§9.4).
- **Layout patterns.** Use MUI layout primitives (`Box`, `Stack`, `Grid`,
  `Container`) for page structure and the host's spacing rhythm (typically
  `theme.spacing(2)` / `theme.spacing(3)` between sections).
- **Icons.** Use `@mui/icons-material`, installed at the same major as the
  plugin's `@mui/material`. For custom icons, use inline SVG wrapped in
  `SvgIcon`.
- **Host fonts.** Rely on the host font stack (delivered through the tokens)
  rather than bundling custom fonts.

**What the plugin inherits from the host today:**

| Inherited (via `--everest-*`) | Not inherited yet |
|---|---|
| Palette `primary`, `secondary`, `error`, `warning`, `info`, `success` (main/dark/light) | Component style overrides of the core theme (e.g. the core's pill-shaped `Button`, `Chip`, `Card`, `Dialog`, input and table styling) |
| Text (primary/secondary/disabled), background (default/paper), divider | Custom typography variants (e.g. `helperText`) |
| Typography per variant: family, size, weight, line height, letter spacing, text transform | Shadows/elevation, grey scale, action states (hover/selected/disabled) |
| Corner radius (`theme.shape.borderRadius`) | Breakpoints, z-index, transitions (plugins get MUI defaults, which currently match the host) |
| Light/dark mode, switching live without a reload | |

Tokens are additive: new groups can be published without breaking existing
plugins (§19).

**Tooling:**

- The `everestctl extension lint` command (P1 DX tooling) will statically check
  the bundle against the rules above and in §9.3 (global CSS injection, only
  host-provided bare imports, no CommonJS `require("react")`, a single copy of
  MUI and Emotion). The look-and-feel recommendations are not linted.
- The `@openeverest/plugin-sdk/testing` mock host publishes the `--everest-*`
  tokens, so plugin unit tests render with the host look.
- The Plugin SDK's TypeScript types guide authors toward the recommended
  patterns at compile time (e.g., extension-point components receive
  `sx`-compatible props rather than `className`).
- A reference compatibility suite (`ui/plugin-compat` in the core repo, branch
  `poc/plugin-mui-isolation`) loads prebuilt plugin bundles under hosts built
  with different MUI majors and checks theming, dark mode, portals, style
  isolation and that the host is left untouched.

## 9. Frontend SDK & Loading Model

### 9.1 Loading strategy

**Decision: dynamic ESM module loading (Headlamp model), not iframes.**

Rationale:
- Tight UX integration — host-themed UI, shared React context, router state.
- Iframes break deep-link navigation, inject a separate auth session, and cannot contribute sidebar entries or theme overrides in a seamless way.
- A browser import map lets the host provide its single React instance (`react`, `react-dom`, `react/jsx-runtime`), so hooks and context work across the host/plugin boundary. MUI and Emotion are deliberately **not** shared: each plugin bundles its own copy (§8.1, §9.3), so host UI upgrades never break already-built plugins.

At shell startup:

```
1. GET /v1/clusters/{cluster}/plugins  →  [{ name, bundleUrl, extensionPoints,
                                              compatibleHostVersions,
                                              compatibleUiContractVersions }, ...]
2. For each enabled plugin:
     skip it (console error) if its UI-contract or host-version range excludes this host (§9.5)
     const mod = await import(bundleUrl)   // dynamic ESM import
     (mod.default ?? mod.register)(pluginApi)
3. Plugin calls api.registerExtension({ type: "clusterDetailTab", component: MyComponent, ... })
4. Shell renders registered components at the declared extension points.
```

All enabled bundles are currently loaded eagerly after login. Loading them per
extension point, only when one is about to render, is a planned follow-up (§19).

#### Host-component rendering & isolation

The shell never mounts a plugin component directly into a core page. Every
registered contribution is rendered through a dedicated **host component** that
owns the mount point, injects the extension-point props, and provides the shared
React context (router, auth). The theme is not shared through React context: the
plugin supplies its own via `PluginThemeProvider` (§9.4). Each extension-point type has its own host
wrapper — e.g., a route host for `route`, a tab host for `clusterDetailTab`, a
settings host for `settingsPanel`.

Each host wrapper is required to isolate plugin failures behind a **plugin error
boundary**: a plugin component that throws must render a contained fallback in
its own slot and must never crash the host shell or sibling plugins. The host
also filters registrations against the extension points declared in the plugin's
manifest — a bundle that registers for a point it did not declare is ignored.

### 9.2 `@openeverest/plugin-sdk`

New package at `ui/packages/plugin-sdk`. Public surface:

```ts
// Registration — extension object shape determines the contribution type.
// Database-context extensions (clusterDetailTab, clusterAction, clusterCard)
// accept an optional 'providers' field to restrict rendering by engine type.
registerExtension(extension: Extension): void

// Example: PostgreSQL-only detail tab
api.registerExtension({
  type: 'clusterDetailTab',
  label: 'SQL Query',
  path: 'sql-query',
  component: SqlQueryTab,
  providers: ['postgresql'],   // omit to show for all engine types
});

// Proxy base path for this plugin, injected by the host:
// `/v1/clusters/{cluster}/plugins/{pluginName}`. Build backend and bundle-asset
// (e.g. icon) URLs from this — never reconstruct the path by hand.
basePath: string

// Hooks — bridge to the host's React context
useEverestApi(): EverestApiClient    // pre-authed HTTP client for /v1/...
useCurrentUser(): User
useCluster(id: string): DatabaseCluster | undefined
useNamespaces(): string[]
useRBAC(): { can: (verb: string, resource: string) => boolean }

// Also on the PluginApi passed to register(api):
React: typeof import('react')        // the host React (same instance the import map serves)
fetch(path, init?): Promise<Response> // authenticated call to this plugin's backend via the proxy
cssNonce: string                     // CSP nonce for <style> tags; pass to PluginThemeProvider
hostVersion: string                  // host application version, "dev" when unknown
uiContractVersion: string            // shared React major, e.g. "18" (§9.5)
```

The SDK is types plus the `register` contract; it has no runtime dependency on
MUI. Theming lives in a separate package, `@openeverest/plugin-theme` (§9.4).

### 9.3 Bundle requirements

Plugin authors produce a single ESM file:

- **Entry.** Default export `register(api: PluginApi): void` (a named
  `register` export is also accepted). Conventionally `main.js`, served at
  `spec.frontend.bundlePath`.
- **Format.** ES module, `esnext` target, production build. The import map does
  not provide `react/jsx-dev-runtime`, so development builds do not load.

**External vs. bundled:**

| Import | Treatment | Why |
|---|---|---|
| `react`, `react-dom`, `react/jsx-runtime` | **External** | The host import map resolves them to the host's React. A second React copy breaks hooks and context. |
| `react-dom/client`, `react-dom/server`, `react/jsx-dev-runtime`, other React subpaths | **Do not import** | Not in the import map; the host owns the React root. |
| `@mui/*`, `@emotion/*`, `@openeverest/plugin-theme` | **Bundle** | The plugin owns its UI stack and its version. |
| Everything else (data fetching, charts, dates, …) | **Bundle** | — |

Only the public React 18 API is available through the import map. Plugins must
not rely on React internals.

**Dependencies.** `@openeverest/plugin-theme` declares its UI stack as peer
dependencies, so the plugin installs and pins them itself:

| Peer | Range |
|---|---|
| `@mui/material` | `^5.15.0 \|\| ^6.0.0 \|\| ^7.0.0 \|\| ^9.0.0` |
| `@emotion/react`, `@emotion/cache` | `^11.11.0` |
| `react`, `react-dom` | `^18.0.0` (types and tests only; never bundled) |

```json
{
  "dependencies": {
    "@emotion/cache": "^11.11.0",
    "@emotion/react": "^11.11.0",
    "@emotion/styled": "^11.11.0",
    "@mui/icons-material": "7.3.11",
    "@mui/material": "7.3.11",
    "@openeverest/plugin-theme": "^0.1.0"
  },
  "devDependencies": {
    "@openeverest/plugin-sdk": "^0.4.0",
    "@types/react": "^18.3.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "typescript": "^5.2.0",
    "vite": "^8.0.0"
  }
}
```

Neither package version above is published yet: `@openeverest/plugin-theme`
0.1.0 is new, and `cssNonce`, `hostVersion` and `uiContractVersion` land in the
next plugin-sdk minor. Until then, link both from a core checkout with `file:`
paths.

- npm rejects a MUI version outside the peer range (`ERESOLVE`). Do not bypass
  it with `--legacy-peer-deps` or `--force`.
- Every other MUI package (`@mui/icons-material`, `@mui/x-date-pickers`,
  `@mui/lab`, …) must be on the **same major** as `@mui/material`. Otherwise
  npm installs a second `@mui/material` for it and the bundle carries two MUI
  copies.
- The built bundle must contain exactly one copy of `@mui/material`,
  `@mui/system` and `@emotion/react`.

**Reference Vite configuration:**

```ts
import { defineConfig } from 'vite';
import type { Plugin } from 'vite';
import react from '@vitejs/plugin-react-swc';

// Resolved by the host import map; never bundle them.
const HOST_PROVIDED = ['react', 'react-dom', 'react/jsx-runtime'];

// A bundled CommonJS dependency that require()s React compiles to a stub that
// throws on load, and the host only logs plugin load errors to the console.
const failOnHostRequire = (): Plugin => ({
  name: 'fail-on-host-require',
  apply: 'build',
  renderChunk(code, chunk) {
    for (const id of HOST_PROVIDED) {
      if (code.includes(`__require("${id}")`)) {
        this.error(`${chunk.fileName} calls require("${id}"); alias that dependency to its ESM build`);
      }
    }
    return null;
  },
});

export default defineConfig(({ command }) => ({
  plugins: [react(), failOnHostRequire()],
  resolve: {
    // Only needed when @openeverest/* packages are linked from a local checkout.
    dedupe: ['@mui/material', '@emotion/react', '@emotion/styled', '@emotion/cache'],
  },
  // Library mode leaves process.env untouched, but MUI and Emotion read NODE_ENV.
  define: command === 'build' ? { 'process.env.NODE_ENV': JSON.stringify('production') } : undefined,
  build: {
    lib: { entry: 'src/main.tsx', formats: ['es'], fileName: () => 'main.js' },
    rollupOptions: { external: HOST_PROVIDED },
  },
}));
```

**Common build pitfalls:**

- **CommonJS dependencies that `require("react")`** become a stub that throws
  when the bundle loads; the plugin then silently fails to register. Alias them
  to their ESM builds. Known cases: `use-sync-external-store/shim/with-selector`
  (pulled in by zustand and `@xyflow/react`), and on MUI 5 the deep
  `@mui/system/*` imports (alias to `@mui/system/esm/*`).
- **CSS files.** Library-mode builds emit no CSS file the host would load.
  Import third-party CSS with `?inline` and render it as
  `<style nonce={api.cssNonce}>` inside the plugin tree (scoped selectors only,
  §8.1).
- **Static assets** (images, icons). Serve them from the plugin backend and
  reference them via `api.basePath` (e.g. `${api.basePath}/icon.png`), or
  inline them as `data:` URIs.

**Browser constraints (host Content Security Policy):**

| Directive | Value | What it means for plugins |
|---|---|---|
| `script-src` | `'self'` + nonce | No `eval` / `new Function`, no inline `<script>`, no scripts from other origins. |
| `style-src` | `'self'` + nonce | Every `<style>` the plugin injects needs `api.cssNonce`; `PluginThemeProvider` passes it to Emotion. |
| `connect-src` | `'self'` (+ the OIDC issuer) | The browser can only call the host. Use `api.fetch` for the plugin backend; external APIs are called from the backend. |
| `img-src`, `font-src` | `'self'` `data:` | Images and fonts only from the host origin (via `api.basePath`) or `data:` URIs. |

The host also sends `Cross-Origin-Embedder-Policy: require-corp` and
`Cross-Origin-Resource-Policy: same-origin`, so cross-origin resources are
blocked regardless.

### 9.4 Theming with `@openeverest/plugin-theme`

`@openeverest/plugin-theme` (core repo, `ui/packages/plugin-theme`) is the
bridge between the host design tokens and the plugin's own MUI. It exports only
`PluginThemeProvider`, `useHostColorMode` and the `PluginThemeProviderProps`
type; plugins import MUI components from their own `@mui/material`.

`PluginThemeProvider` reads the host's `--everest-*` variables, builds a MUI
theme from them with the plugin's MUI, gives the plugin its own namespaced
Emotion cache, and rebuilds the theme whenever the host switches between light
and dark. It renders no `CssBaseline`: document-level styles belong to the host.

When using the package, wrap every registered component once, typically with a small root component:

```tsx
import type { ReactNode } from 'react';
import { PluginThemeProvider } from '@openeverest/plugin-theme';
import { Button, Paper, Typography } from '@mui/material';
import type { PluginApi, PluginRegisterFn } from '@openeverest/plugin-sdk';

// Emotion keys allow only lowercase letters and "-"; must be unique per plugin.
const CACHE_KEY = 'sql-explorer';

let pluginApi: PluginApi;

const PluginRoot = ({ children }: { children: ReactNode }) => (
  <PluginThemeProvider cacheKey={CACHE_KEY} nonce={pluginApi.cssNonce}>
    {children}
  </PluginThemeProvider>
);

const SqlExplorerPage = () => (
  <PluginRoot>
    <Paper sx={{ p: 3 }}>
      <Typography variant="h5">SQL Explorer</Typography>
      <Button variant="contained">Run query</Button>
    </Paper>
  </PluginRoot>
);

const register: PluginRegisterFn = (api) => {
  pluginApi = api;
  api.registerExtension({ type: 'route', label: 'SQL Explorer', component: SqlExplorerPage });
};

export default register;
```

**`PluginThemeProvider` props:**

| Prop | Required | Value |
|---|---|---|
| `cacheKey` | Yes | The plugin's Emotion cache key; also the class-name prefix and the `data-emotion` attribute of its `<style>` tags. Use the plugin's `metadata.name`, with anything other than lowercase letters and `-` removed or replaced (Emotion rejects digits and `_`). It must be unique across plugins and must not be the host's key (`percona-css`). Use the same key for every root of one plugin. |
| `nonce` | Yes (under the host CSP) | `api.cssNonce`. Without it the browser blocks the plugin's styles. |
| `children` | Yes | The plugin UI. |

`useHostColorMode(): 'light' | 'dark'` returns the host's current mode and
re-renders on change. Use it for code that doesn't go through the MUI theme
(chart libraries, canvas drawing).

**Token contract.** The host publishes these variables on `:root` and keeps them
in sync with its theme, including on light/dark switches:

| Variable | Value |
|---|---|
| `--everest-color-{primary,secondary,error,warning,info,success}-{main,dark,light}` | Palette colours |
| `--everest-color-text-{primary,secondary,disabled}` | Text colours |
| `--everest-color-background-{default,paper}` | Background colours |
| `--everest-color-divider` | Divider colour |
| `--everest-radius` | Corner radius, unitless pixels (e.g. `4`) |
| `--everest-font-{variant}-{family,size,weight,line-height,letter-spacing,transform}` | Typography per variant, for `h1`–`h6`, `subtitle1`, `subtitle2`, `body1`, `body2`, `button`, `caption`, `overline` |

and the active mode as `data-everest-color-scheme="light" | "dark"` on `<html>`.

Outside MUI (plain CSS, third-party components) plugins may use the variables
directly, e.g. `color: var(--everest-color-text-secondary)`. Inside MUI code,
prefer `theme.palette.*`: MUI's colour helpers (`alpha`, `darken`, `lighten`)
cannot parse `var(...)` values.

The variable names are a **public contract**. New variables can be added at any
time without breaking plugins; renaming or removing one is a breaking host
change.

### 9.5 Compatibility & versioning

A plugin frontend is checked on two independent axes when the host loads it:

| Axis | Plugin declares (`Plugin` CR) | Host exposes | Guards against |
|---|---|---|---|
| UI contract | `spec.compatibleUiContractVersions` | `api.uiContractVersion` (React major) | The shared React major changing |
| Host API | `spec.compatibleHostVersions` | `api.hostVersion` | Host API / extension-point changes |

- Ranges use npm semver syntax with **full `x.y.z` versions**, e.g.
  `">=18.0.0 <19.0.0"` or `"^18.0.0"`. Shorthand such as `">=19"` or `"18.x"` is
  currently not enforced (the check passes), so always spell out full versions.
- An empty range always passes. On a development host (`hostVersion` is `dev`
  or `0.0.0`) the host-version check is skipped.
- A plugin that fails either check is skipped with a console error; the rest of
  the UI keeps working.

**Host UI upgrades.** Because each plugin bundles its own MUI, a host MUI upgrade
needs no action from plugin authors. This was verified by loading plugin bundles
built with MUI 5.18, 6.5, 7.3 and 9.4, unchanged, under hosts built with MUI 6.5,
7.3 and 9.4: theming, live dark mode, portals, style isolation and the host's own
styling behaved the same on every host. A host **React major** upgrade is
different: it changes the UI contract, and plugins must declare the ranges they
support.

**Plugin MUI upgrades** are the plugin's own migration, done on its own
schedule; MUI removes deprecated APIs between majors (e.g. `PaperProps` and
`inputProps` are gone in MUI 9). `@openeverest/plugin-theme`'s peer range says
which MUI majors it supports; widening that range is a non-breaking release of
the package.

## 10. Backend Model

### 10.1 Proxy architecture

The OpenEverest API server exposes:

```
/v1/clusters/{cluster}/plugins/{pluginName}/*   →   proxied to plugin backend
```

The browser never calls a plugin backend directly. All requests are:

1. Authenticated by the host (session cookie / OIDC token validated).
2. RBAC-checked: the requesting user must have `use` on the `plugins` resource for the object `{cluster}/{pluginName}`.
3. Forwarded to the backend with an `X-Everest-User` header containing a short-lived, signed JWT carrying `{ sub, namespaces, pluginName, exp }`.
4. Audit-logged by the host.

This gives the host complete visibility over plugin traffic and prevents plugins from acting outside the user's own RBAC scope.

### 10.2 Backend authentication to OpenEverest

When a plugin backend needs to call OpenEverest APIs on behalf of the user, it uses the `X-Everest-User` JWT as a bearer token. The host validates it and enforces the user's own permissions — the plugin cannot escalate privilege.

### 10.3 Hosting modes

| Mode | How it works |
|---|---|
| **In-cluster** | `serviceRef` names a Kubernetes `Service`. Host resolves DNS internally. No public ingress needed. |
| **External** | `externalUrl` is an HTTPS endpoint. Credentials (e.g. API key) from a `Secret` are passed as `Authorization` header. |

### 10.4 Daemon mode

A daemon backend has no user session driving it. It runs continuously and typically polls or reacts to events. To support this:

**Autonomous identity.** The host issues each daemon a long-lived **plugin service token** — a JWT bound to the plugin name (not to any user) and scoped to the permissions declared in `spec.permissions`. The token:

- Is mounted into the backend pod via a projected `Secret` named `<plugin-name>-token`, refreshed automatically before expiry (default TTL 24 h).
- Carries claims `{ sub: "plugin:<name>", permissions: [...], exp }`.
- Is accepted by the OpenEverest API as a bearer token. RBAC checks the declared permissions, not any user identity.
- Can read but **cannot write** to spec-001 resources, regardless of declared permissions (enforced by a hard-coded denylist — see §16 Q9).

**Why not just call the kube API directly?** A daemon could `watch` CRs natively from inside the cluster. We deliberately route everything through the OpenEverest API so:

- Plugins remain portable (work the same way against in-cluster and remote-SaaS OpenEverest deployments).
- The audit trail is centralised.
- The hard "no writes to spec-001 resources" guarantee can be enforced uniformly.

**Lifecycle.** When an `InstalledExtension` for a plugin that behaves as a daemon reaches `Ready`, the host (in a future Phase 3 implementation):

1. Generates the service token and writes it to the projected `Secret`.
2. Ensures the backend `Deployment` has at least one replica.
3. Polls `healthPath` to track liveness; surfaces status on the `InstalledExtension` as the `TokenIssued` / `BackendReachable` conditions.
4. On `InstalledExtension` deletion, scales to zero and revokes the token.

### 10.5 Event stream

The host exposes lifecycle events for resources it manages over a single **stateless streaming endpoint**. Plugins (and any other client) consume the stream by holding an open HTTP connection. There is no outbound push, no delivery queue, and no per-plugin server-side state.

This design is a thin wrapper over Kubernetes' native watch mechanism: OpenEverest's reconcilers already learn about state changes via kube watches on the underlying CRs. The event endpoint normalises those watch events into a stable, plugin-facing schema and streams them out. State lives in etcd, where it already does — not in OpenEverest.

**Endpoint.**

```
GET /v1/events?since=<resourceVersion>&types=<csv>&namespaces=<csv>
   Accept: text/event-stream
```

Delivered as Server-Sent Events (SSE). Each event carries the underlying Kubernetes `resourceVersion` as its cursor. The connection is held open by the client; the host streams events as they happen.

**Initial event taxonomy** (extensible — plugins ignore types they don't know):

| Event type | Triggered when |
|---|---|
| `database-cluster.created` | A `DatabaseCluster` resource is created. |
| `database-cluster.ready` | A cluster transitions to `ready` state. |
| `database-cluster.updated` | Spec changes on an existing cluster (resize, version upgrade). |
| `database-cluster.deleted` | A cluster is deleted (post-finalizer). |
| `database-cluster.failed` | A cluster transitions to a failed state. |
| `backup.started` | A backup job starts. |
| `backup.completed` | A backup job completes successfully. |
| `backup.failed` | A backup job fails. |
| `restore.started` / `restore.completed` / `restore.failed` | Same for restores. |
| `instance.created` / `instance.deleted` | spec-001 `Instance` lifecycle. |
| `user.login` | A user successfully authenticates (local or OIDC). |
| `user.login-failed` | An authentication attempt fails. |
| `user.logout` | A user session is invalidated. |
| `plugin.installed` | A `Plugin` CR is created. |
| `plugin.uninstalled` | A `Plugin` CR is deleted. |
| `plugin.enabled` | A plugin becomes ready (enabled via `Plugin` CR or `InstalledExtension` created). |
| `plugin.disabled` | A plugin becomes not-ready (disabled via `Plugin` CR or `InstalledExtension` deleted). |
| `namespace.added` | A namespace is registered with OpenEverest. |
| `namespace.removed` | A namespace is removed from OpenEverest. |
| `settings.updated` | Platform settings are modified. |

Events are sourced from two mechanisms:

1. **Kubernetes watches** — the event hub watches `DatabaseCluster`, `Backup`, `Restore`, `Instance`, `Plugin`, and `InstalledExtension` CRs. Changes are normalised into the event envelope and broadcast to subscribers.
2. **Direct publish** — API handlers that do not correspond to a watched CR (session create/delete, settings update) call `Hub.Publish()` directly to emit events into the same fan-out pipeline.

**Event envelope.**

```json
{
  "resourceVersion": "482719",
  "type": "database-cluster.deleted",
  "occurredAt": "2026-05-08T12:00:00Z",
  "namespace": "team-alpha",
  "resource": {
    "kind": "DatabaseCluster",
    "name": "orders-prod",
    "uid": "a1b2c3d4-...",
    "engine": "postgresql",
    "version": "15.5"
  },
  "prevState": { "phase": "ready" },
  "newState":  { "phase": "deleting" },
  "actor":     { "type": "user", "id": "alice@example.com" }
}
```

**Authentication.** The stream is authenticated like any other `/v1` endpoint — a daemon plugin uses its service token (§10.4); a user-facing tool uses the user's session token. The events the client receives are filtered by the token's permissions (the user's accessible namespaces, or the daemon token's declared scope).

**Why pull and not push?** A push model requires the host to track which events have been delivered to which plugin, manage retry queues, and persist state across restarts. By pulling, the plugin owns its cursor and the host stays stateless. This mirrors how Kubernetes itself exposes change streams and how every kube controller already handles restarts.

### 10.6 Catch-up & restart semantics

The stream uses `resourceVersion` as its cursor — the same mechanism Kubernetes watch already provides. Plugins follow the standard kube-watch restart pattern.

**Normal operation.** The plugin opens `GET /v1/events?since=<rv>` with the last `resourceVersion` it persisted, receives any events that occurred since, and then streams new events live. After processing each event the plugin persists the new cursor (typically to its own local storage — PVC, embedded KV, etc.).

**Stream drop.** If the connection drops (network, host restart, scale event), the plugin reconnects with its last cursor. As long as the cursor is still within the kube watch cache window (default 5 min, configurable on the kube API server), the stream resumes from exactly that point with no gap.

**Cold start or stale cursor.** If the plugin has no cursor yet, or its cursor is older than the watch cache, the plugin must:

1. Call `GET /v1/databases`, `/v1/backups`, etc. to snapshot current state.
2. Note the `resourceVersion` returned in the list response.
3. Open `GET /v1/events?since=<that rv>` to resume streaming.

This is exactly how the Kubernetes client-go informer handles the same problem; the SDK provides a helper that wraps the dance.

**Implications.**

- No event store or dead-letter queue in the host. No state to back up, no state to migrate during host upgrades.
- The plugin must stay connected (or reconnect quickly) to capture events. Plugins that don't tolerate stream gaps must persist their cursor before acknowledging work, and must implement the snapshot-then-watch fallback.
- A plugin that holds the connection but processes events slowly will back-pressure the stream; the host will drop the slowest connections under memory pressure (advertised via a configurable per-connection buffer size). This is acceptable: dropped clients reconnect with `since=` and catch up.

### 10.7 Infrastructure plugins — Kubernetes resource management

Some plugins need to create and manage their own Kubernetes resources in response to database-cluster lifecycle events. Examples include ProxySQL (SQL proxy deployed per cluster), connection poolers, or monitoring sidecars. These are called **infrastructure plugins**.

#### Plugins own their Kubernetes RBAC

Plugin Kubernetes RBAC is **shipped by the plugin, not generated by the host**. The plugin's bundle is a Helm chart; the chart's templates include the plugin's `ServiceAccount`, `Role`/`ClusterRole`, and `RoleBinding`/`ClusterRoleBinding`. `everestctl extension install` installs the chart, which applies these objects alongside the plugin's `Deployment`, `Service`, and any plugin-owned `ConfigMap`s.

The host does not:

- Declare a `spec.kubePermissions` field on the `Plugin` CR.
- Validate plugin RBAC against a denylist.
- Generate `Role`/`ClusterRole` objects.
- Reconcile or repair plugin RBAC.

Trust is anchored at the **plugin hub**: only signed, curated bundles are allowed to install. A plugin whose chart asks for unreasonable RBAC is rejected at hub-vetting time, not at runtime by the host. Unsigned or untrusted plugins are refused at install time (see the hub design — out of scope for this section).

Note: the daemon service-token denylist in §10.4 — which blocks plugin tokens from writing spec-001 resources via `/v1` — is a separate guarantee and remains in force. It governs API calls to OpenEverest, not direct Kubernetes API calls.

#### Example chart layout

```
proxysql-plugin/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── plugin.yaml             # the Plugin CR
    ├── installed-extension.yaml # the InstalledExtension CR
    ├── serviceaccount.yaml
    ├── role.yaml               # (or clusterrole.yaml)
    ├── rolebinding.yaml        # (or clusterrolebinding.yaml)
    ├── deployment.yaml         # the plugin backend Deployment
    └── service.yaml            # the plugin backend Service
```

The chart author decides scope: a single `ClusterRole` + `ClusterRoleBinding` for a cluster-wide plugin, or a `Role` + `RoleBinding` per target namespace. The chart is the single source of truth.

#### Plugin CR naming

The `Plugin` and `InstalledExtension` CRs are cluster-scoped singletons keyed by the plugin's canonical id (§7.1). The chart **must** pin their `metadata.name` to `{{ .Chart.Name }}` — never `{{ .Release.Name }}` or `{{ include "chart.fullname" . }}`. Namespaced resources (Deployment, Service, ServiceAccount, RoleBinding, ConfigMap) may still use the release name; their names are not load-bearing.

```yaml
# templates/plugin.yaml
apiVersion: extensions.openeverest.io/v1alpha1
kind: Plugin
metadata:
  name: {{ .Chart.Name }}            # canonical, not .Release.Name
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
  # ...
```

Consequences:

- Installing the same chart twice under different release names fails on the second install (`Plugin` CR already exists). This is intentional: two installs would race for the same `/v1/clusters/{cluster}/plugins/{name}` route, RBAC resource, and CRD group.
- Hub vetting rejects charts whose `Plugin` / `InstalledExtension` templates do not pin `metadata.name` to `.Chart.Name`.
- `everestctl extension install` enforces the same rule on the rendered manifest before applying, covering the raw-`helm install` and GitOps paths via the hub-vetting gate and covering the CLI path directly.

#### Lifecycle integration — ProxySQL example

1. **Vetting.** The hub publishes a signed ProxySQL chart whose templates include a `ServiceAccount`, a `Role` granting `apps/deployments` and `core/services,configmaps` in the target namespaces, and the corresponding `RoleBinding`. The hub has reviewed the chart before allowing it to be installed.

2. **Installation.** Admin runs `everestctl extension install proxysql`. The CLI fetches the chart, renders it with the cluster-admin's values, and applies all rendered objects — including the `Plugin` CR, the `InstalledExtension` CR, and the plugin's RBAC. Per-cluster end-user access is governed by Everest RBAC (`use` on `plugins` for `{cluster}/proxysql`, §11.2).

3. **Cluster creation.** User creates a new PXC cluster. The ProxySQL plugin's `instanceCreateFormSection` renders an "Enable ProxySQL" toggle and configuration fields (exposure mode, resource limits, custom rules).

4. **Config handoff.** On submission the host POSTs the plugin config to `POST /v1/clusters/{cluster}/plugins/proxysql/instance-config` with the instance name, namespace, and config blob. The plugin backend stores this config (e.g., in a ConfigMap).

5. **Event-driven deployment.** The plugin daemon receives a `database-cluster.ready` event via SSE and creates a ProxySQL `Deployment`, `Service`, and `ConfigMap` in the target namespace using its bundle-shipped `ServiceAccount`.

6. **Detail tab.** The plugin registers a `clusterDetailTab` showing ProxySQL status, metrics, and a config editor. Changes submitted via the tab's UI are sent to the plugin backend, which updates the ProxySQL ConfigMap and triggers a rolling restart.

7. **Cluster deletion.** The plugin receives a `database-cluster.deleted` event and cleans up its resources. As a safety net, the plugin sets `ownerReferences` on all created resources pointing to the `DatabaseCluster` CR so Kubernetes GC catches anything missed.

#### Security boundaries

- The plugin runs in its own pod with its own `ServiceAccount`. It never shares the host's credentials.
- The plugin's RBAC reach is whatever its chart granted — Helm uninstall removes those objects on `everestctl extension uninstall`.
- The host does not proxy or relay Kubernetes API calls. The plugin talks directly to the Kubernetes API server using its own bound credentials.
- All plugin-created resources should carry standard labels (`app.kubernetes.io/managed-by: everest-plugin-<name>`) for auditability.

### 10.8 Stateful plugins — Plugin-declared Custom Resources

Some plugins need persistent, structured, namespace-scoped state that goes beyond what a ConfigMap or a plugin-managed database provides. Examples:

- **Presets** — named configuration templates that pre-fill `InstanceSpec` fields during instance creation.
- **Database user management** — CRs representing logical database users, reconciled by a plugin that provisions credentials in the target engines.
- **Migration configs** — declarative schema migration state for a schema-management plugin.

These plugins are called **stateful plugins**. They declare custom resource schemas in the `Plugin` manifest; the host installs, validates, and watches those CRDs; reconciliation events are forwarded to the plugin backend as HTTP callbacks. **No per-plugin operator is required.**

#### Why not a separate operator per plugin?

Forcing every stateful plugin to ship its own controller-runtime binary introduces:

- An additional pod + ServiceAccount + leader election per plugin.
- Race conditions between the plugin operator and the host controller when both react to Instance lifecycle events.
- High DX barrier — plugin authors must learn controller-runtime, build Go binaries, manage CRD versions, and handle upgrades.

The host-managed reconciliation model avoids all of this. The plugin author writes an HTTP handler; the host does the Kubernetes plumbing.

#### Declaring custom resources

A new `spec.customResources[]` field on the `Plugin` CR:

```yaml
apiVersion: extensions.openeverest.io/v1alpha1
kind: Plugin
metadata:
  name: presets
spec:
  displayName: "Presets"
  version: "1.0.0"
  # ... frontend, backend, permissions as usual ...

  # Custom resource declarations.
  customResources:
    - kind: Preset
      # API group is always <pluginName>.plugins.openeverest.io
      # (auto-derived; cannot be overridden)
      scope: Namespaced
      # OpenAPI v3 schema for spec validation (same format as CRD
      # structural schemas). The host generates the full CRD from this.
      schema:
        type: object
        properties:
          provider:
            type: string
          version:
            type: string
          topology:
            type: object
            x-kubernetes-preserve-unknown-fields: true
          components:
            type: object
            x-kubernetes-preserve-unknown-fields: true
          backup:
            type: object
            x-kubernetes-preserve-unknown-fields: true
      # Status subresource is always enabled.
      # Additional printer columns (optional).
      additionalPrinterColumns:
        - name: Provider
          jsonPath: .spec.provider
          type: string
        - name: Version
          jsonPath: .spec.version
          type: string
```

The generated CRD will have:
- Group: `presets.plugins.openeverest.io`
- Version: `v1alpha1` (auto-assigned; plugin controls via manifest version)
- Kind: `Preset`
- Scope: Namespaced (the plugin CR can be created in any namespace of a cluster where the caller has Everest RBAC to `use` the plugin)

#### Host reconciliation model

When a `Plugin` with `customResources` is installed, the host:

1. **Generates the CRD** from the declared schema and applies it to the cluster. The CRD is owned by the `Plugin` CR (via `ownerReferences`) so it is garbage-collected on plugin uninstall.
2. **Starts a dynamic informer** (using `dynamic.Interface` + `cache.Informer`) watching instances of the declared Kind in namespaces where the plugin is installed.
3. **On CR create/update/delete**, the host calls the plugin backend:

   ```
   POST /v1/clusters/{cluster}/plugins/{pluginName}/reconcile
   Content-Type: application/json

   {
     "event": "create" | "update" | "delete",
     "object": { <full CR JSON> },
     "old": { <previous version, on update only> },
     "namespace": "team-alpha"
   }
   ```

4. **Backend responds** with status + optional requeue:

   ```json
   {
     "status": { "ready": true, "message": "Validated against provider schema" },
     "requeue": false,
     "requeueAfter": "0s"
   }
   ```

5. **Host writes** the status subresource onto the CR and requeues if requested.

If the backend is unreachable, the host retries with exponential backoff and sets a `Reconciling` condition on the CR.

#### Access scoping

Plugin CRs are permitted in any namespace of a cluster where the caller has Everest RBAC to `use` the plugin (the cluster-scoped `plugins` resource, object `{cluster}/{name}`, §11.2). The host rejects (via a validating webhook or informer-level filter) any CR created by a user without that grant. There is no separate per-namespace enable list on the `InstalledExtension` — access scoping is a pure RBAC concern, consistent with how other cluster-scoped resources are gated.

#### Security & validation

| Concern | Mitigation |
|---|---|
| API group hijacking | Plugin CRDs must live under `<pluginName>.plugins.openeverest.io`. The host rejects any other group. |
| Schema size DoS | Maximum schema size: 64 KB (compressed). Enforced at admission. |
| Kind collision | Kind names are globally unique within `plugins.openeverest.io`. The host rejects duplicates at Plugin create time. |
| CRD manipulation | The plugin itself cannot modify or delete the CRD — only the host controller manages CRD lifecycle. Plugin charts that ship verbs on `apiextensions.k8s.io` are rejected at hub vetting. |
| Orphaned CRs on uninstall | On Plugin deletion, the host deletes the CRD. Kubernetes cascades deletion to all CRs. Admin receives a warning if CRs exist. |
| Reconcile endpoint abuse | The `/reconcile` call carries the host's internal service token, not user identity. The plugin backend verifies the token before acting. |

#### Interaction with other plugin capabilities

Stateful plugins typically combine custom resources with other capabilities:

- **Frontend bundle** — UI components to create/edit/list plugin CRs (e.g., a Preset editor, a database-user manager).
- **`instanceCreateFormSection`** — integrate plugin CRs into the Instance creation flow (e.g., "Select a Preset" dropdown).
- **Event consumer** — react to Instance lifecycle events to trigger reconciliation of related plugin CRs.
- **Request handler** — serve API endpoints that operate on plugin CRs (e.g., `POST /v1/clusters/{cluster}/plugins/presets/apply` to copy a Preset into a new Instance spec).

#### Example: Presets plugin

1. Admin installs the Presets plugin. The host creates the `Preset` CRD under `presets.plugins.openeverest.io`.
2. Admin grants the `use` verb on `plugins` for `{cluster}/presets` to the `team-alpha` role (or specific users) via Everest RBAC. Those users can now create `Preset` CRs in any namespace of that cluster.
3. A user creates a `Preset` named `production-large` with topology, component sizing, and backup config for their PXC clusters.
4. The Presets backend receives the `/reconcile` call, validates the Preset content against the Provider schema (via `GET /v1/providers/{name}`), and returns status `{ ready: true }`.
5. During Instance creation, the Presets plugin's `instanceCreateFormSection` shows a "Select Preset" dropdown. On selection, it fetches the Preset CR via the OpenEverest API and pre-fills the form.
6. The host submits the filled form as a normal `Instance` create — the Preset is a one-time copy, not a live binding.

## 11. RBAC Integration

OpenEverest already uses a Casbin policy model (`pkg/rbac`). Generic plugins slot into it without new concepts.

### 11.1 Plugin install permission

Installing a plugin (creating a `Plugin` CR) requires the `create` verb on the new resource type `plugins` at cluster scope. Only `admin` has this by default.

### 11.2 Per-user plugin access

Plugin consumption is gated by the `use` verb on the cluster-scoped `plugins`
resource, with the object identifying the target cluster and plugin as
`{cluster}/{name}`. Admins grant it to roles or individual users:

```
p, role:viewer, plugins, use, prod/sql-explorer
```

The host checks this before proxying any request to the plugin backend and
before rendering extension-point components for the user. Because the object is
`{cluster}/{name}`, a grant scopes access to a plugin **within a specific
Kubernetes cluster**, not per Everest namespace.

### 11.3 Defense in depth

Even if a plugin backend attempts to call OpenEverest APIs directly, it can only act under the user identity carried in the JWT, which is bound to that user's existing RBAC policy. A plugin cannot read resources the user cannot read, and cannot mutate resources the user cannot mutate.

For **daemon mode**, the autonomous plugin service token (§10.4) is bound to the declared `spec.permissions` and to the plugin name — it is not a user identity. It cannot be used to assume a user's privileges, and it is unconditionally denied write access to spec-001 resources. The token is rotated automatically and revoked on `InstalledExtension` deletion.

## 12. `everestctl` Integration

Go's `plugin` package is Linux-only and fragile across compiler versions. WASM toolchains are promising but immature. The pragmatic choice is **subcommand-via-shellout**.

### How it works

The plugin manifest declares a `cli.image` (OCI). When the user runs:

```sh
everestctl extension run sql-explorer -- query --db my-db "SELECT 1"
```

`everestctl`:
1. Pulls/caches the CLI image locally (or finds it in a local cache).
2. Execs the container, passing `--host`, `--token` (short-lived API token) and the user-supplied arguments via `stdin`/`stdout`.
3. Streams output back to the terminal.

Discovery & lifecycle:

```sh
everestctl extension list                          # installed extensions (plugins + providers)
everestctl extension info     <name>               # show install metadata and conditions
everestctl extension install  <oci-ref>
everestctl extension uninstall <name>
```

Per-cluster plugin access is granted via Everest RBAC (`use` on the `plugins` resource for `{cluster}/{name}`, §11.2) — there is no separate `extension namespace add/remove` subcommand.

There is no `everestctl plugin` subcommand. Plugins and providers are both managed via `everestctl extension`; the `--type` filter on `list` distinguishes them when needed.

## 13. Lifecycle & Distribution

### 13.1 OCI artifact as the canonical unit

A plugin is distributed as a single OCI artifact containing:

```
layers/
  manifest.yaml            # Plugin CR
  main.js                  # Frontend bundle (if any)
  annotations:
    org.opencontainers.image.title: "sql-explorer"
    org.everest.plugin.schema-version: "v1"
```

Backend and CLI images are separate OCI images referenced by digest from `manifest.yaml`. This keeps the plugin artifact small and lets image layers be cached independently.

### 13.2 Versioning

- Plugins are versioned with SemVer.
- `spec.compatibleHostVersions` is a semver range (same syntax as npm).
- `spec.compatibleUiContractVersions` is a semver range over the host's React major (§9.5). Both ranges are also checked when the UI loads the bundle.
- The host rejects installation of a plugin whose range does not include the running host version.
- Plugins are upgraded independently of the host: `everestctl extension upgrade <name>`.

### 13.3 Air-gapped environments

```sh
# Export on an internet-connected machine
everestctl extension export sql-explorer --output sql-explorer.tar

# Import on an air-gapped machine
everestctl extension install --from-tar sql-explorer.tar
```

The `--from-tar` flag pushes images into the in-cluster registry if one is configured (e.g., Harbor), then creates the `Plugin` CR.

## 14. Security Considerations

| Concern | Mitigation |
|---|---|
| Malicious frontend bundle | Bundles served through host proxy (not a CDN); SRI hash in manifest verified before serving; strict CSP allows only host origin. |
| XSS via plugin code | Plugin JS runs in the same origin — normal XSS mitigations apply (React escaping, CSP). Considered acceptable given admin-only install gate. |
| Plugin tampering with the shared runtime | The React runtime the host shares through the import map (`window.__EVEREST_PLUGIN_RUNTIME__`) is frozen and non-writable, so one plugin cannot swap React for the others. This is a robustness measure, not a security boundary: plugin code still runs with the user's full session, so trust is anchored at install time (admin-only install, curated catalog, and later bundle signing / SRI). |
| Style interference between plugins and the host | Each plugin styles through its own Emotion cache (unique `cacheKey`, §9.4) and may not inject global CSS or write to `:root` (§8.1). |
| Credential exfiltration | Plugin backends never receive raw DB credentials; only short-lived, scoped JWTs. DB connection details brokered on-demand via `/v1/databases/{id}/connection-details`. |
| Privilege escalation | All plugin API calls re-checked against the acting user's own RBAC. Plugin cannot escalate beyond the user's permissions. |
| In-cluster lateral movement | Plugin backend runs in its own `ServiceAccount` with a minimal `Role` auto-generated from `spec.permissions`. `NetworkPolicy` restricts egress to declared endpoints only. |
| Supply chain | Manifest must include OCI image digests. Host verifies cosign signatures when `spec.signatureVerification: true` is set on the cluster. |
| Secrets in manifests | Credentials for external backends are stored exclusively in `Secret` resources, never in the `Plugin` CR itself. |
| Admin-only install | Creating a `Plugin` CR and the corresponding `InstalledExtension` requires cluster-admin RBAC. Per-cluster access for end users is granted via Everest RBAC (`use` on the `plugins` resource for `{cluster}/{name}`, §11.2); regular users only get `use` on the specific plugin/cluster combinations the admin allows. |
| Daemon token theft | Plugin service token is mounted via a projected `Secret` (short-lived, auto-rotated, default TTL 24 h). Token is bound to plugin name + declared permissions, never a user identity, and revoked on `InstalledExtension` deletion. |
| Forged events | The event stream is delivered over the plugin's authenticated HTTPS connection to OpenEverest — there is no inbound push the plugin needs to validate. |
| Event-driven privilege escalation | Events are informational only — they do not authorise the plugin to perform any action. Any follow-up API call still goes through normal RBAC checks. |
| Slow event consumer / DoS on host | Per-connection bounded buffer; slow consumers are dropped and reconnect with `since=`. No unbounded queue grows in the host. |
| Infrastructure plugin kube access | Plugin Kubernetes RBAC is shipped by the plugin's Helm chart, not the host. The plugin runs in its own pod with its own `ServiceAccount` bound to the plugin's `Role`/`ClusterRole`. Trust is anchored at the plugin hub: only signed, curated bundles are admitted; the hub rejects bundles asking for unreasonable RBAC at vetting time. Plugin never shares the host's `ServiceAccount`. |
| Plugin creates orphaned resources | Plugin must handle `database-cluster.deleted` events to clean up. As a safety net, plugin-created resources should carry `ownerReferences` pointing to the `DatabaseCluster` CR for Kubernetes GC. |

## 15. Reference Architecture

```mermaid
graph TB
    subgraph Browser
        Shell[React UI Shell]
        Bundle[Plugin Frontend Bundle\nloaded via dynamic import]
    end

    subgraph OpenEverest Core
        API["API Server\n/v1/clusters/{cluster}/plugins/{name}/*"]
        Auth[Auth / Session]
        RBAC[RBAC Engine\nCasbin]
        Proxy[Plugin Proxy]
        EventStream[GET /v1/events\nSSE over kube watch\nstateless]
        TokenSvc[Plugin Token Service\nautonomous identities]
        PluginAPI["GET /v1/clusters/{cluster}/plugins\nGET /v1/clusters/{cluster}/plugin-context\nGET /v1/databases/{id}/connection-details"]
    end

    subgraph Kubernetes
        PluginCR[Plugin CR]
        PIcr[InstalledExtension CR]
        Secret[Secret\nCredentials]
    end

    subgraph Plugin Backend
        InCluster[In-Cluster Service]
        External[External HTTPS Endpoint]
        Daemon[Daemon Pod\nautonomous, no user session]
    end

    Shell -->|1. fetch plugin list| PluginAPI
    Shell -->|2. dynamic import| Bundle
    Bundle -->|3. API calls via SDK| API
    API --> Auth
    Auth --> RBAC
    RBAC -->|4. proxy if allowed| Proxy
    Proxy -->|X-Everest-User JWT| InCluster
    Proxy -->|X-Everest-User JWT| External
    InCluster -->|5. call back with JWT| PluginAPI
    Daemon -->|GET /v1/events?since=rv\nheld-open SSE| EventStream
    EventStream -->|kube watch| PluginCR
    TokenSvc -->|projected Secret\nplugin service token| Daemon
    Daemon -->|API calls with service token| API
    API --> PluginCR
    API --> PIcr
    API --> Secret
```

## 16. Answering Prior Open Questions

> The earlier draft of this spec (§7) raised ten open questions. This section records the decisions.

1. **Can generic plugins ship their own Kubernetes CRDs?**
   Yes, with constraints. Plugins may declare custom resource schemas in their manifest under `spec.customResources[]`. The host generates, installs, and watches the resulting CRDs; reconciliation is handled by the plugin backend via HTTP callbacks (see §10.8). Plugins do **not** run their own operators. All plugin CRDs live under the `<pluginName>.plugins.openeverest.io` API group — plugins cannot declare CRDs in the `core.openeverest.io`, `extensions.openeverest.io`, or any other core API group. CRDs for spec 001 Providers remain host-exclusive.

2. **What is the minimal backend surface OpenEverest must expose?**
   The existing v1 API, plus four new additions:
   - `GET /v1/clusters/{cluster}/plugins` — plugin discovery (list enabled plugins + bundle URLs).
   - `GET /v1/clusters/{cluster}/plugin-context` — current user identity, accessible namespaces.
   - `GET /v1/databases/{id}/connection-details` — brokered, short-lived credentials.
   - `GET /v1/events?since=<resourceVersion>` — stateless SSE event stream over kube watch (§10.5).
   No outbound push from the host; no separate "plugin API" needed; everything else goes through `/v1`.

3. **How do plugins get DB credentials?**
   OpenEverest brokers them on demand via `GET /v1/databases/{id}/connection-details`, gated by the user's existing RBAC. Tokens are short-lived (15 min). Plugins never cache or store credentials.

4. **Single plugin model or distinct shapes?**
   Single model with optional parts. A plugin that sets only `frontend` is a UI-only extension. One that sets only `backend` (with no `frontend`) is a headless integration or CLI tool. A full plugin sets both. `everestctl` already handles all cases.

5. **Can a plugin interact with spec 001 resources (`DatabaseCluster`, `Instance`)?**
   Read-only, via the OpenEverest API only. Plugins may `GET` cluster and instance resources through `/v1/...`; they may not call the Kubernetes API directly and may not mutate spec 001 resources.

6. **Is dynamic JS module loading viable for OpenEverest's frontend?**
   Yes. ESM dynamic `import()` is supported by all modern browsers. Import maps handle shared dependency deduplication. The React shell already uses Vite; the ESM loader is a natural extension. (See §9 for details.)

7. **What does the `everestctl` extension surface look like?**
   Subcommand-via-shellout to a plugin-declared OCI image. `everestctl extension run <name> -- <args>` execs the container with a short-lived API token injected. (See §12 for details.)

8. **How are plugins versioned independently of the host?**
   SemVer on the plugin; `spec.compatibleHostVersions` semver range in the manifest; host enforces the range at install time and again when the UI loads the bundle, together with `spec.compatibleUiContractVersions` (§9.5). Plugins upgrade independently via `everestctl extension upgrade`, and bundle their own MUI so host UI upgrades don't force a plugin rebuild (§8.1).

9. **Are there categories of plugins we would explicitly disallow?**
   Yes: plugins may not write to `DatabaseCluster`, `Instance`, `Provider`, or any other spec 001 CRs. The RBAC policy for the auto-generated plugin `ServiceAccount` excludes `create`, `update`, `patch`, and `delete` verbs on those resource types unconditionally. The same denylist applies to the daemon plugin service token (§10.4) regardless of what the manifest declares.

10. **Air-gapped / regulated environments?**
    Supported via `everestctl extension export` / `--from-tar`. Images pushed to the in-cluster registry; no internet egress required after initial export.

## 17. Phased Roadmap

### Phase 1 — MVP

Deliver the minimal complete path for a plugin author to ship a UI page.

- `Plugin` and `InstalledExtension` CRDs (both cluster-scoped; install metadata only — no per-namespace enable list).
- `GET /v1/clusters/{cluster}/plugins` discovery endpoint.
- `GET /v1/installed-extensions` list endpoint.
- Dynamic ESM loader in the React shell.
- `@openeverest/plugin-sdk` stub: `registerExtension`, `useEverestApi`.
- Extension points: `route` and `sidebarItem` only.
- Simple backend proxy (`/v1/clusters/{cluster}/plugins/{name}/*`) with session auth.
- Admin-only `InstalledExtension` create gate in RBAC.
- `everestctl extension install / list / uninstall`.

### Phase 2 — Multi-tenant & access control

- `plugins` resource in Casbin model (object `{cluster}/{name}`) — per-user, per-cluster `use` grants are the sole control over which users can invoke which plugin in which cluster.
- Per-tenant config secrets: plugin authors who need per-namespace runtime config consume a `ConfigMap`/`Secret` named by convention (e.g., `<plugin>-config` in the target namespace) — no host-side wiring.
- In-cluster backend `serviceRef` discovery (DNS resolution, health check).
- Credentials broker: `GET /v1/databases/{id}/connection-details`.
- `GET /v1/clusters/{cluster}/plugin-context` endpoint.

### Phase 3 — Daemon mode & event stream

Unlocks the metering / billing / audit / external-sync class of plugins.

- Plugin token service — mint, mount, and rotate autonomous service tokens.
- Daemon mode: host-managed `Deployment` lifecycle, health tracking, status conditions on `InstalledExtension`.
- `GET /v1/events` SSE endpoint backed by a kube watch on the relevant CRs (`DatabaseCluster`, `Instance`, `Backup`, `Restore`).
- Event normaliser: maps kube watch events into the plugin-facing schema (§10.5).
- SDK helpers for the snapshot-then-watch restart pattern (§10.6).
- Hard denylist on writes to spec-001 resources from daemon tokens.
- **No event store, no delivery worker, no DLQ in the host.** State lives in etcd; cursor lives in the plugin.

### Phase 4 — Infrastructure plugins & form extension points

- Helm-based plugin install: `everestctl extension install` fetches the plugin chart and applies it, creating the `Plugin` CR, the plugin's `ServiceAccount`/`Role`/`RoleBinding` (or `ClusterRole`/`ClusterRoleBinding`), the backend `Deployment`/`Service`, and the matching `InstalledExtension`.
- Plugin hub integration hooks: chart digest pinning, signature checks at install time (the full trust model is a separate spec).
- `instanceCreateFormSection` and `instanceEditFormSection` extension points.
- `POST /v1/clusters/{cluster}/plugins/{name}/instance-config` endpoint for plugin config handoff.
- ProxySQL reference plugin as the canonical infrastructure plugin example.

### Phase 5 — Rich UI extension points & distribution

- Extension points: `clusterDetailTab`, `clusterAction`, `clusterCard`, `globalDashboardWidget`, `settingsPanel`.
- `everestctl extension run` shellout for CLI extensions.
- OCI artifact packaging: `everestctl extension export` / `--from-tar`.
- Bundle SRI verification + optional cosign signature check.
- Auto-generated `ServiceAccount` + `Role` + `NetworkPolicy` for backend pods.

### Phase 6 — Stateful plugins & plugin CRDs

Unlocks plugins that need persistent, structured, namespace-scoped state without running their own operator.

- `spec.customResources[]` declaration on the `Plugin` CR.
- Dynamic CRD generation + installation from declared schemas.
- Dynamic informer for plugin CRs (unstructured client, namespace-filtered).
- `POST /v1/clusters/{cluster}/plugins/{name}/reconcile` endpoint — host calls plugin backend on CR create/update/delete; backend returns status + requeue.
- Validating webhook (or informer filter) restricting plugin CRs to clusters where the caller has the `use` verb on the `plugins` resource for `{cluster}/{name}`.
- Kind uniqueness enforced across all plugins; `apiextensions.k8s.io` access rejected at hub vetting (plugins do not own their CRD lifecycle).
- Presets reference plugin as the canonical stateful-plugin example.
- SDK helpers: `usePluginResources(kind)` hook for the frontend, `PluginResourceClient` for the backend.

### Phase 7 — Polish & ecosystem

- `themeOverride` extension point (branding / logos).
- Plugin-to-plugin event bus (opt-in pub/sub via the SDK).
- External backend support (`externalUrl` + `credentialsSecretRef`) for daemon and event-consumer modes (the external backend opens a held SSE connection back to the host — no inbound push required).
- Marketplace / catalog UI (`Plugin` browser in the web UI).
- `everestctl extension upgrade` with version-range enforcement.

## 18. Implementation Cost & Plugin Author DX

This section captures the engineering reality of delivering this design in the existing OpenEverest codebase, and what's needed for plugin authors to actually enjoy building plugins.

### 18.1 Impact on the existing codebase

The host stack is already well-shaped for this work — Echo + oapi-codegen for the API, controller-runtime for CRDs, Casbin for RBAC, Vite + React Router v6 for the UI, Cobra for `everestctl`. Most of the plugin system slots in cleanly; only two areas require genuinely new infrastructure.

**Existing infrastructure that we reuse as-is:**

- **HTTP / middleware** ([internal/server/](internal/server/)) — Echo handler chain (`newHandlerChain(valH, rbacH, k8sH)`) is the natural place to drop in the plugin proxy and discovery handler.
- **Auth / JWT** ([pkg/session/](pkg/session/), [pkg/oidc/](pkg/oidc/)) — already mints and validates JWTs; the plugin token service can reuse the same signing key infrastructure.
- **CRDs / controllers** ([api/extensions/v1alpha1/](api/extensions/v1alpha1/), [internal/controller/](internal/controller/)) — kubebuilder-style; adding `Plugin` and `InstalledExtension` follows the same pattern as existing CRDs.
- **RBAC** ([pkg/rbac/](pkg/rbac/), [data/rbac/model.conf](data/rbac/model.conf)) — Casbin model already supports glob matching on the object field. Adding the cluster-scoped `plugins` resource (object `{cluster}/{name}`) is a constants change plus a few policy lines, no model rewrite.
- **CLI** ([commands/](commands/)) — Cobra; `everestctl plugin ...` slots in alongside existing subcommand groups like [accounts/](commands/accounts/).
- **UI build** ([ui/apps/everest/](ui/apps/everest/)) — Vite is ESM-native, dynamic `import()` works out of the box.

**Genuinely new infrastructure that we have to build:**

1. **Event stream endpoint**. OpenEverest today has no plugin-facing change stream. The Phase-3 work is a `GET /v1/events` SSE handler that opens a kube watch on the relevant CRs (`DatabaseCluster`, `Instance`, `Backup`, `Restore`), normalises each watch event into the plugin-facing schema (§10.5), filters by the caller's RBAC and namespace scope, and writes the result as SSE frames. **No event bus, no delivery worker, no queue, no store** — state lives in etcd where it already does, and the cursor lives in the plugin. The reconcilers themselves don't need any modification because the kube watch sees the same status transitions they do.
2. **Dynamic UI route registration**. React Router v6 prefers compile-time route trees. The pragmatic approach is a single wildcard route `/plugins/:pluginName/*` that dispatches to a `<PluginHost>` component which mounts whichever extension component the plugin registered. Sidebar items are pulled from React state populated at startup. Some Vite plumbing for an import map is also needed so plugin bundles can `import 'react'` and resolve to the host singleton.
3. **Dynamic CRD management for stateful plugins** (Phase 6). The host must generate `CustomResourceDefinition` objects from plugin-declared schemas, install them, and watch instances via `dynamic.Interface` + custom informers. On CR mutations the host calls the plugin backend's `/reconcile` endpoint and writes back status. This is genuinely new infrastructure — `controller-runtime` does not natively support dynamic type registration, so a raw dynamic informer layer (similar to Crossplane's composite-resource watches) is needed. Estimated scope: ~1,500–2,500 LoC on top of the base plugin system.

**Estimated change scope** (rough, not a commitment):

| Area | New / modified files | Approx. LoC |
|---|---|---|
| `Plugin` + `InstalledExtension` CRDs ([api/extensions/v1alpha1/](api/extensions/v1alpha1/)) | new | 300–500 |
| Plugin reconciler ([internal/controller/](internal/controller/)) | new | 400–600 |
| Plugin proxy + discovery + token service ([internal/server/](internal/server/)) | new | 500–800 |
| `GET /v1/events` SSE handler + kube-watch normaliser (`internal/server/events.go`) | new | 250–400 |
| RBAC additions ([pkg/rbac/](pkg/rbac/)) | modified | 100–150 |
| `everestctl plugin ...` ([commands/](commands/)) | new | 400–600 |
| `@openeverest/plugin-sdk` ([ui/packages/](ui/packages/)) | new package | 400–600 |
| UI dynamic loader + `<PluginHost>` ([ui/apps/everest/](ui/apps/everest/)) | modified | 300–500 |
| Dynamic CRD manager + reconcile proxy (Phase 6) | new | 1,500–2,500 |
| Tests + fixtures | new | 800–1200 |
| **Total** | | **~4,950–7,850 LoC** |

Dropping the durable event-bus subsystem trims roughly 700–1,100 LoC and a significant chunk of operational complexity (no store to back up, no DLQ to monitor, no delivery state to migrate during host upgrades). The phased roadmap (§17) defers the most invasive remaining parts (daemon mode, dynamic UI loader) to Phase 3+ so a Phase-1 MVP can ship with route + sidebar extension points only and validate the architecture.

### 18.2 Highest-risk integration points

1. **Slow event consumers**. A plugin that holds the SSE connection but processes events slowly back-pressures the host's per-connection buffer. **Mitigation**: bounded buffer per connection (configurable, default ~1k events); drop the slowest connections; clients reconnect with `since=` and resume. No unbounded queue can grow on the host.
2. **Watch cache window**. Plugins disconnected longer than the kube watch cache (default 5 min) must do a snapshot-then-watch fallback to resume. **Mitigation**: ship the snapshot-then-watch helper as a first-class SDK function (§10.6); document the pattern as the standard restart flow.
3. **RBAC scoping for the `plugins` resource**. The current Casbin model uses glob matching on the object field. A naive `{cluster}/*` (or `*/*`) object would grant access to all plugins. **Mitigation**: use specific per-plugin `{cluster}/{name}` object lines, not wildcards, and reject wildcard plugin policies in the policy editor.
4. **Dynamic React Router**. Workable but easy to get wrong (history scope, error boundaries, nested routes). **Mitigation**: prototype the `<PluginHost>` wrapper early in Phase 1.
5. **API backward compatibility**. Once plugins ship, breaking response shapes (including the event envelope in §10.5) breaks plugins. **Mitigation**: lock the plugin-facing subset of `/v1` early (discovery, plugin-context, connection-details, events) and treat it as a stability boundary; version the event envelope explicitly.
6. **Dynamic informer lifecycle** (Phase 6). Plugin CRDs are installed at runtime; informers must be started/stopped as plugins are installed or removed. A stale informer watching a deleted CRD will error-loop. **Mitigation**: wrap dynamic informers in a manager that tracks CRD existence via a watch on `apiextensions.k8s.io/v1/customresourcedefinitions`; tear down informers when the CRD disappears. Crossplane solves this same problem with `engine.Start()/Stop()` per composite resource.
7. **Reconcile endpoint reliability** (Phase 6). If the plugin backend is down, CRs pile up in a pending state. **Mitigation**: exponential backoff with jitter; status condition `Reconciling=Unknown, reason=BackendUnreachable`; surface on `InstalledExtension` status. No silent data loss — CRs stay in etcd, the informer retries.

### 18.3 Plugin author developer experience

The architecture is only as valuable as the number of plugins built on it. If authoring a plugin requires reading 80 pages of docs, hand-writing a YAML CRD manifest, configuring an import map, and standing up a local OpenEverest cluster to test against — nobody will write plugins.

The DX investments below are **as important as the architecture itself** and should be tracked alongside the implementation roadmap.

**P0 — minimum required for any external plugin author to succeed:**

- **`everestctl plugin scaffold <name>`** — generates a working plugin in one command: `manifest.yaml`, a TypeScript frontend stub with the SDK wired up, an optional Go backend stub, a `Makefile`, and a sample `InstalledExtension`. This is the single highest-leverage DX item.
- **Plugin SDK with strong types** — every extension point's props are typed in `@openeverest/plugin-sdk`. Event payloads are typed. The `EverestApi` client is generated from the OpenAPI spec so the client and server types can never drift. Plugin authors never write `any`.
- **A "Hello World" reference plugin** in the [openeverest/plugin-examples](https://github.com/openeverest/plugin-examples) repo (to be created) covering a UI-only plugin, a daemon plugin, and an event-subscriber plugin. Each example is a working, tested codebase, not a snippet in docs.
- **A working dev-mode loop**: `everestctl plugin dev` runs the plugin's Vite dev server and patches the host's plugin discovery to point at `http://localhost:3001/main.js`. Hot-reload works in the browser without rebuilding/redeploying anything.

**P1 — significantly improves authoring quality:**

- **Manifest linter**: `everestctl plugin lint` validates the manifest against the CRD schema, checks that declared permissions exist in the OpenAPI spec, verifies SemVer ranges, and warns on unsigned bundles.
- **Mock SDK for unit tests**: `@openeverest/plugin-sdk/testing` exports `mockEverestApi()`, `mockUser()`, `renderInPluginHost()` so plugin components can be unit-tested in isolation without spinning up a cluster.
- **Backend SDK packages** (`@openeverest/plugin-backend-sdk` for Node, plus a Go module): wraps JWT verification, event signature verification, the service-token bootstrap, and the OpenEverest API client. Plugin backends shouldn't have to reimplement these.
- **Compatibility check at install time**: `everestctl plugin install` refuses to install plugins whose `compatibleHostVersions` excludes the running host, with a clear error message.

**P2 — ecosystem-grade polish:**

- **CI matrix template**: a reusable GitHub Actions workflow that runs a plugin's tests against multiple OpenEverest versions in kind clusters.
- **E2E test harness**: a Playwright fixture (`@openeverest/plugin-sdk/e2e`) that installs the plugin under test into a kind cluster and exposes the host UI for assertions.
- **Versioned event schema**: every event payload carries a `schemaVersion` field; the SDK exposes per-version typed accessors so plugins can adopt new event versions incrementally.

**Tooling cost estimate:**

| Item | Priority | Effort |
|---|---|---|
| `everestctl plugin scaffold` | P0 | 1–2 weeks |
| Plugin SDK + generated types + mock utils | P0 | 2–3 weeks |
| Reference plugins repo + docs | P0 | 1–2 weeks |
| `everestctl plugin dev` (hot-reload) | P0 | 2–3 weeks |
| Manifest linter | P1 | 1–2 weeks |
| Backend SDK (Node + Go) | P1 | 1–2 weeks |
| Install-time compatibility check | P1 | < 1 week |
| E2E test harness (Playwright fixture) | P2 | 2–3 weeks |
| CI matrix template | P2 | 1 week |
| Versioned event schema | P2 | 1 week |

Total DX investment: **roughly 4–6 person-weeks for P0, plus another 4–6 for P1, on top of the architecture work itself**. Treating DX as a first-class deliverable — rather than a "we'll write some docs later" item — is the single biggest predictor of plugin ecosystem adoption.

### 18.4 Recommended sequencing

1. Land Phase 1 + the P0 DX bundle together. A scaffolder, SDK, and one reference plugin shipping with the MVP make the difference between a feature nobody uses and a feature people start building on immediately.
2. Use the reference plugins as the canary for Phase 2 / Phase 3 changes — if the metering reference plugin breaks under a daemon-mode change, the spec is wrong, not the plugin.
3. Don't ship Phase 3 (daemon mode + events) without the corresponding SDK updates and a working metering reference plugin. The architecture and the reference implementation must land together.

## 19. Open Questions

1. **OCI media type**: custom `application/vnd.openeverest.plugin.v1` media type vs. reusing a Helm chart. Custom type is cleaner semantically; Helm is more familiar for GitOps workflows. Needs decision before Phase 4.

2. **Bundle hosting**: serve plugin bundles from a `ConfigMap` (size-limited to ~1 MB after compression) or store them in an in-cluster object store / PVC? For Phase 1 a `ConfigMap` suffices; Phase 4 distribution needs OCI artifact storage or an in-cluster registry.

3. **React shell router sandboxing**: expose the host React Router `<Outlet>` to plugins directly, or wrap plugin routes in a sandboxed sub-router with a restricted history scope? Direct exposure is simpler; sandboxing gives better isolation for plugin navigation errors.

4. **Plugin-to-plugin communication**: should plugins be allowed to call each other's backends via `/v1/clusters/{cluster}/plugins/{otherName}/*`? If yes, the requesting plugin must have `use` on the target plugin and carry a valid user session. Decision deferred to Phase 5.

5. **Bundle size / performance**: no size limit defined yet. Each bundle carries its own MUI and Emotion (roughly 75–110 kB gzip), and all enabled bundles are currently loaded eagerly after login, so N installed plugins cost N downloads and N style caches even on pages that show none of them. Planned follow-up: build sidebar entries and routes from the descriptor's `extensionPoints` and `import()` a bundle only when one of its extension points is about to render (cached, loaded at most once), with an eager fallback for plugins that declare no extension points.

6. **Event retention window**: bounded by the kube watch cache window (default 5 minutes on the kube API server, configurable). Plugins that have been disconnected longer must do the snapshot-then-watch fallback (§10.6). No separate retention policy needed in OpenEverest — etcd is the source of truth and the watch cache covers the gap.

7. **Event delivery model**: pull-based SSE stream over kube watch (decided, §10.5). The earlier draft proposed a push-based bus with a server-side queue; that introduced unwanted state in the host (delivery log, retry queue, DLQ, durability across restarts). The pull model puts the cursor on the plugin side and keeps the host stateless. Open sub-question: do we ever need a push variant for SaaS / off-cluster consumers that cannot hold an inbound connection? If so, a small webhook bridge plugin ("event forwarder") could be built on top of the pull stream without bringing state into the core.

8. **Synchronous (pre-) hooks**: this design covers post-hoc, fire-and-forget events only. Should we also support **synchronous validating hooks** (e.g., "before creating a cluster, ask the policy plugin to approve")? That class of plugin sits closer to a Kubernetes admission webhook and may justify a separate spec; explicitly out of scope for v1. *Note:* the `instanceCreateFormSection` extension point (§8, §10.7) partially addresses the "user opts in at creation time" use case — it lets a plugin collect configuration during instance creation and act on it asynchronously, without requiring a synchronous pre-hook in the host.

9. **Daemon scaling**: does the host enforce single-replica daemons (simpler, no distributed-lock concerns) or allow plugin authors to declare a replica count? Multi-replica daemons need event delivery to be load-balanced across replicas in a partition-aware way — non-trivial. Single-replica is the recommended Phase 3 starting point.

10. **Plugin CRD schema evolution**: when a plugin upgrades and its declared schema changes, existing CRs may become invalid. Options: (a) require backward-compatible schema changes only (additive fields, no removals), (b) support multiple CRD versions with conversion (complex, mirrors kube-native CRD versioning), (c) plugin owns migration via a one-time reconcile pass on upgrade. Leaning toward (a) with (c) as an escape hatch. Needs decision before Phase 6.

11. **Dynamic informer lifecycle**: `controller-runtime` assumes static type registration at manager startup. Plugin CRDs require either restarting the manager (disruptive) or using raw `dynamic.Interface` + custom informers outside the manager. The latter is feasible (Crossplane, KubeVela use this pattern) but loses some controller-runtime ergonomics. Prototype needed in Phase 6.

12. **Presets: core CRD vs. plugin CRD**: Presets could be shipped as either a first-class core CRD (like Instance, Provider) or as the first stateful plugin exercising the Phase 6 mechanism. Core CRD ships faster and integrates tighter with Instance validation; plugin CRD validates the extensibility model. Decision: start with a core `Preset` CRD to unblock the feature quickly, then optionally migrate to a plugin once Phase 6 lands — or keep it core if the tight validation integration proves essential.

13. **Sharing more of the core look**: plugins inherit palette, typography, radius and dark mode (§8.1), but not the core theme's component style overrides (e.g. the pill-shaped `Button`) or shadows, grey scale and action states. Options: (a) publish more `--everest-*` tokens (additive, independent of MUI version); (b) have `@openeverest/plugin-theme` ship the core's component overrides as theme options, which ties those releases to MUI's theme format and may narrow its peer range; (c) a separate package of core-styled components on a pinned MUI. (a) first, (b) when a plugin needs the component look.

## 20. Definition of Done

> To be defined once Phase 1 implementation begins.

## 21. Alternatives Considered

> To be populated as the design discussion progresses.

## 22. References

* [001 — Plugins Architecture](./001-plugins-architecture.md)
* [Headlamp plugin system](https://headlamp.dev/docs/latest/development/plugins/building-and-deploying/)
