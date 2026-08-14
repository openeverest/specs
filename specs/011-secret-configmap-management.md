# Secret and ConfigMap Management

*   **Status:** Draft
*   **Authors:** @chilagrow
*   **Created:** 2026-06-30
*   **Last Updated:** 2026-08-12
*   **Related Issues:** https://github.com/openeverest/openeverest/issues/1798, https://github.com/openeverest/openeverest/issues/2471


---

## 1. Summary

This specification introduces namespace-scoped API endpoints for managing Secrets and ConfigMaps used by Instances and its components. Secrets and ConfigMaps created through these endpoints are labeled for identification. Providers customize the creation form fields using UI schema definitions.

## 2. Motivation

Currently, users must create Secrets and ConfigMaps using `kubectl`, breaking the unified management experience. This spec provides:

1. **API endpoints** for Create, Get, Delete operations
2. **Lifecycle management** via labels and owner references
3. **Provider-customizable UI** using the same schema mechanism as Instance creation

## 3. Goals & Non-Goals

**Goals:**
- Provide namespace-scoped API endpoints for Secrets and ConfigMaps
- Support inline creation during Instance creation
- Allow providers to define UI schema for Secrets and ConfigMaps creation forms

**Non-Goals:**
- Cross-namespace Secrets and ConfigMaps sharing
- Secret versioning or audit trails

## 4. Proposed Solution / Design

### 4.1. Managed Secrets and ConfigMaps Labels

Secrets and ConfigMaps created are identified by labels:

```yaml
apiVersion: v1
kind: Secret  # or ConfigMap
metadata:
  name: my-splithorizon-cert
  namespace: default
  labels:
    openeverest.io/managed: "true"                             # Created via OpenEverest API
    openeverest.io/provider: "provider-percona-server-mongodb" # Provider name (same as spec.provider in Instance CR)
    openeverest.io/definition: "splithorizon-tls"                # Resource definition for filtering
```

**Label meanings:**
- `openeverest.io/managed: "true"` — Secrets and ConfigMaps created by OpenEverest API
- `openeverest.io/provider` — Provider that uses Secrets and ConfigMaps type
- `openeverest.io/definition` — Secrets and ConfigMaps definition for filtering (e.g., `splithorizon-tls`, `import-credentials`)

### 4.2. API Endpoints

#### Secrets

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/clusters/{cluster}/namespaces/{ns}/secrets` | Create secret |
| GET | `/clusters/{cluster}/namespaces/{ns}/secrets` | List secrets (only metadata without content) |
| GET | `/clusters/{cluster}/namespaces/{ns}/secrets/{name}` | Get secret (only metadata data) |
| DELETE | `/clusters/{cluster}/namespaces/{ns}/secrets/{name}` | Delete secret |
| GET | `/clusters/{cluster}/providers/{name}` | Get secret UI schema definitions |

#### ConfigMaps

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/clusters/{cluster}/namespaces/{ns}/config-maps` | Create configmap |
| GET | `/clusters/{cluster}/namespaces/{ns}/config-maps` | List configmaps |
| GET | `/clusters/{cluster}/namespaces/{ns}/config-maps/{name}` | Get configmap (includes data) |
| DELETE | `/clusters/{cluster}/namespaces/{ns}/config-maps/{name}` | Delete configmap |
| GET | `/clusters/{cluster}/providers/{name}` | Get configmap UI schema definitions |

#### `POST /clusters/{cluster}/namespaces/{ns}/secrets`

The request body follows Kubernetes Secret/ConfigMap format with OpenEverest-specific labels.
The labels `openeverest.io/definition` and `openeverest.io/provider` are required.

For creating base64 encoded request:

```json
{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": {
    "name": "my-splithorizon-cert",
    "namespace": "default",
    "labels": {
      "openeverest.io/provider": "provider-percona-server-mongodb",
      "openeverest.io/definition": "splithorizon-tls"
    }
  },
  "type": "Opaque",
  "data": {
    "tls.crt": "<base64-encoded>",
    "tls.key": "<base64-encoded>"
  }
}
```

For plaintext request:

```json
{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": {
    "name": "my-splithorizon-cert",
    "namespace": "default",
    "labels": {
      "openeverest.io/provider": "provider-percona-server-mongodb",
      "openeverest.io/definition": "splithorizon-tls"
    }
  },
  "type": "Opaque",
  "stringData": {
    "tls.crt": "<plaintext>",
    "tls.key": "<plaintext>"
  }
}
```

**Conflict Handling:**

If a secret with the same name already exists, the API server applies the following logic:

**Check owner reference**: If the existing secret has an owner reference set:
- Return `409 CONFLICT` with message: "Secret is owned by another resource"
- User must choose a different name or delete the owning Instance first

**Orphaned secret**: If no owner reference and not in use:
- Delete the existing orphaned secret
- Create the new secret with the provided configuration
- Return `201 CREATED`

This prevents race conditions and ensures secrets are not accidentally overwritten while in use.

Response strips `data` and `stringData`:
```json
{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": {
    "name": "my-splithorizon-cert",
    "namespace": "default",
    "labels": {
      "openeverest.io/managed": "true", // Added by API server
      "openeverest.io/provider": "provider-percona-server-mongodb",
      "openeverest.io/definition": "splithorizon-tls"
    }
  },
  "type": "Opaque",
  // no data or stringData
}
```

#### `GET /clusters/{cluster}/namespaces/{ns}/secrets`

Response strips `data` and `stringData`:

```json
[{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": {
    "name": "my-splithorizon-cert",
    "namespace": "default",
    "labels": {
      "openeverest.io/managed": "true",
      "openeverest.io/provider": "provider-percona-server-mongodb",
      "openeverest.io/definition": "splithorizon-tls"
    }
  },
  "type": "Opaque",
  // no data or stringData
}]
```

The response list contains secrets with `"openeverest.io/managed": "true"` label. Also filters the secrets based on the user's `read` RBAC permission on `secrets`.

#### `GET /clusters/{cluster}/namespaces/{ns}/secrets/{name}`

The response is the same as an item of list:

```json
{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": {
    "name": "my-splithorizon-cert",
    "namespace": "default",
    "labels": {
      "openeverest.io/managed": "true",
      "openeverest.io/provider": "provider-percona-server-mongodb",
      "openeverest.io/definition": "splithorizon-tls"
    }
  },
  "type": "Opaque",
  // no data or stringData
}
```

If the secret was found, but does not contain the label `"openeverest.io/managed": "true"`, server returns 404.

#### `DELETE /clusters/{cluster}/namespaces/{ns}/secrets/{name}`

On success, returns 204 with no body.

If the secret has owner reference, it returns CONFLICT 409.

If the secret was found, but does not contain the label `"openeverest.io/managed": "true"`, it returns 404.

#### `GET /clusters/{cluster}/providers/{name}`

```json
{
  "metadata": {
    "name": "percona-server-mongodb",
  },
  "spec": {
    "componentTypes": { ... },
    "components": { ... },
    "topologies": { ... },
    "versions": [ ... ],
    "uiSchema": {
      "replicaSet": { ... },
      "sharded": { ... },
    },
    "secrets": {
      "splithorizon-tls": {
        "parametersSchema": {
          "openAPIV3Schema": {
            "type": "object",
            "properties": {
              "tls.crt": { "type": "string" },
              "tls.key": { "type": "string" }
            }
          }
        },
        "uiSchema": { 
          // UI schema for split horizon
          "label": "TLS Certificate",
          "componentsOrder": ["tlsCrt", "tlsKey"],
          "components": { ... }
        }
      },
      "import-credentials": {
        "parametersSchema": {
          "openAPIV3Schema": { ... }
        },
        "uiSchema": {
          // UI schema for import credentials
        }
      }
    },
    "configMaps": {
      "custom-mongod": {
        "parametersSchema": {
          "openAPIV3Schema": { ... }
        },
        "uiSchema": {
          // UI schema for custom mongod configuration
        }
      }
    }
  }
}
```

**Label examples:**
- Component secret: `"openeverest.io/definition": "splithorizon-tls"`
- Import credential: `"openeverest.io/definition": "import-credentials"`

#### Query Parameters (List)

**Filtering:**
- `provider` — Filter by provider name (e.g., `provider=percona-server-mongodb`)
- `definition` — Filter by definition (e.g., `definition=splithorizon-tls`)
  - Can be combined with provider filter

**Examples:**
- `/secrets` — All managed secrets
- `/secrets?provider=percona-server-mongodb` — Provider-specific secrets
- `/secrets?provider=percona-server-mongodb&definition=splithorizon-tls` — Definition within provider
- `/secrets?definition=splithorizon-tls` — Definition across all providers

### 4.3. Instance Creation Flow

When configuring a component that requires a Secret or ConfigMap, the UI displays a dropdown and/or add secret option decided by provider developer.

**Examples:**
- Split horizon: `GET /clusters/{cluster}/namespaces/{ns}/secrets?provider={provider}&definition=splithorizon-tls`
- Import credentials: Show add secret option

The UI shows:
- Existing managed secrets matching the provider and definition (optional)
- Option to "Create New" which opens the creation form

**Inline Creation Flow:**

```mermaid
sequenceDiagram
    participant UI
    participant API

    UI->>API: POST /secrets
    API-->>UI: Created
    UI->>API: POST /instances (refs secret)
    alt Failure
        UI->>API: DELETE /secrets/{name}
    end
```

#### Split Horizon Example

For split horizon, the component will require secret such as `my-splithorizon`.

```
apiVersion: instance.openeverest.io/v1alpha1
kind: Instance
metadata:
  name: my-mongo-cluster
  namespace: production
spec:
  provider: percona-server-mongodb
  components:
    engine:
      ...
    splithorizon:
      config:
        secretRef:
          name: my-splithorizon
```

Below is the content of an example of the secret:

```
apiVersion: v1
kind: Secret
metadata:
  name: my-splithorizon
  namespace: production
  labels:
    openeverest.io/provider: percona-server-mongodb
    openeverest.io/definition: splithorizon-tls
type: Opaque
data:
  tls.crt: "<secure-crt>"
  tls.key: "<secure-key>"
```

#### Import Example

For import, instance creation flow (or separate flow as currently done in OpenEverest v1) require additional steps to populate necessary secrets.
The example secret `my-mongo-cluster-import-creds` is used only by this instance.

```
apiVersion: instance.openeverest.io/v1alpha1
kind: Instance
metadata:
  name: my-mongo-cluster
  namespace: production
spec:
  provider: percona-server-mongodb
  topology: replica-set
  components:
    ...

  dataSource:
    type: Import # NEW
    import: # NEW
      backupClassName: psmdb-mongoimport-import
      storageName: s3-external-data
      config:
        path: /imports/users.json
        credentialsSecretName: my-mongo-cluster-import-creds
```

Below is the content of an example of the secret:

```
apiVersion: v1
kind: Secret
metadata:
  name: my-mongo-cluster-import-creds
  namespace: production
  labels:
    openeverest.io/provider: percona-server-mongodb
    openeverest.io/definition: import-credentials
type: Opaque
data:
  MONGODB_BACKUP_USER: "backup"
  MONGODB_BACKUP_PASSWORD: "<secure-password>"
  MONGODB_CLUSTER_ADMIN_USER: "clusterAdmin"
  MONGODB_CLUSTER_ADMIN_PASSWORD: "<secure-password>"
  MONGODB_CLUSTER_MONITOR_USER: "clusterMonitor"
  MONGODB_CLUSTER_MONITOR_PASSWORD: "<secure-password>"
  MONGODB_DATABASE_ADMIN_USER: "databaseAdmin"
  MONGODB_DATABASE_ADMIN_PASSWORD: "<secure-password>"
  MONGODB_USER_ADMIN_PASSWORD: "<secure-password>"
```

### 4.4. Provider Secret/ConfigMap Definitions

Providers define Secret and ConfigMap types in separate definition files, similar to BackupClass definitions:

#### Definition Structure

```
definition/
  secrets/
    import-credentials/
      definition.yaml  # Schema configuration
      ui.yaml          # UI rendering hints
      types.go         # Go types for schema validation
  configmaps/
    custom-mongod/
      definition.yaml  # Schema configuration
      ui.yaml          # UI rendering hints
      types.go         # Go types for schema validation
```

#### Secret Definition

**definition.yaml:**
```yaml
# definition/secrets/import-credentials/definition.yaml
parametersSchema:
  openAPIV3Schema: ImportCredentialsConfig
```

**ui.yaml:**
```yaml
# definition/secrets/import-credentials/ui.yaml
label: "Import Credentials"
componentsOrder:
  - backupUser
  - backupPassword
  - clusterAdminUser
  - clusterAdminPassword
  - clusterMonitorUser
  - clusterMonitorPassword
  - databaseAdminUser
  - databaseAdminPassword
  - userAdminPassword
components:
  backupUser:
    uiType: text # TODO: use base64 encoded text
    path: "data.MONGODB_BACKUP_USER" # use stringData for text, data for base64 encoded text
    fieldParams:
      label: "Backup User"
    validation:
      required: true
  backupPassword:
    uiType: text # TODO: use base64 encoded text
    path: "data.MONGODB_BACKUP_PASSWORD" # use stringData for text, data for base64 encoded text
    fieldParams:
      label: "Backup Password"
    validation:
      required: true
  clusterAdminUser:
    uiType: text # TODO: use base64 encoded text
    path: "data.MONGODB_CLUSTER_ADMIN_USER" # use stringData for text, data for base64 encoded text
    fieldParams:
      label: "Cluster Admin User"
    validation:
      required: true
  clusterAdminPassword:
    uiType: text # TODO: use base64 encoded text
    path: "data.MONGODB_CLUSTER_ADMIN_PASSWORD" # use stringData for text, data for base64 encoded text
    fieldParams:
      label: "Cluster Admin Password"
    validation:
      required: true
  clusterMonitorUser:
    uiType: text # TODO: use base64 encoded text
    path: "data.MONGODB_CLUSTER_MONITOR_USER" # use stringData for text, data for base64 encoded text
    fieldParams:
      label: "Cluster Monitor User"
    validation:
      required: true
  clusterMonitorPassword:
    uiType: text # TODO: use base64 encoded text
    path: "data.MONGODB_CLUSTER_MONITOR_PASSWORD" # use stringData for text, data for base64 encoded text
    fieldParams:
      label: "Cluster Monitor Password"
    validation:
      required: true
  databaseAdminUser:
    uiType: text # TODO: use base64 encoded text
    path: "data.MONGODB_DATABASE_ADMIN_USER" # use stringData for text, data for base64 encoded text
    fieldParams:
      label: "Database Admin User"
    validation:
      required: true
  databaseAdminPassword:
    uiType: text # TODO: use base64 encoded text
    path: "data.MONGODB_DATABASE_ADMIN_PASSWORD" # use stringData for text, data for base64 encoded text
    fieldParams:
      label: "Database Admin Password"
    validation:
      required: true
  userAdminPassword:
    uiType: text # TODO: use base64 encoded text
    path: "data.MONGODB_USER_ADMIN_PASSWORD" # use stringData for text, data for base64 encoded text
    fieldParams:
      label: "User Admin Password"
    validation:
      required: true
```

**types.go:**
```go
// definition/secrets/import-credentials/types.go
package importcredentials

// ImportCredentialsConfig describes the expected data keys for this secret type.
// +k8s:openapi-gen=true
type ImportCredentialsConfig struct {
    MongoDBBackupUser           string `json:"MONGODB_BACKUP_USER"`
    MongoDBBackupPassword       string `json:"MONGODB_BACKUP_PASSWORD"`
    MongoDBClusterAdminUser     string `json:"MONGODB_CLUSTER_ADMIN_USER"`
    MongoDBClusterAdminPassword string `json:"MONGODB_CLUSTER_ADMIN_PASSWORD"`
    MongoDBClusterMonitorUser     string `json:"MONGODB_CLUSTER_MONITOR_USER"`
    MongoDBClusterMonitorPassword string `json:"MONGODB_CLUSTER_MONITOR_PASSWORD"`
    MongoDBDatabaseAdminUser     string `json:"MONGODB_DATABASE_ADMIN_USER"`
    MongoDBDatabaseAdminPassword string `json:"MONGODB_DATABASE_ADMIN_PASSWORD"`
    MongoDBUserAdminPassword     string `json:"MONGODB_USER_ADMIN_PASSWORD"`
}
```

#### Component UI Schema Reference

Components reference secret definitions.
Below is an example, but final UI schema components will change.

```yaml
# definition/topologies/<topology>/topology.yaml
ui:
  sections:
    configuration:
      label: "Import Configuration"
      components:
        importCredentials:
          uiType: secret # New - may change
          path: spec.dataSource.import.config.credentialsSecretName # New: does not exist yet and may change
          definition: import-credentials  # New: References definition/secrets/import-credentials
          fieldParams:
            label: "Import Credentials"
            createLabel: "+ Add New Credentials" # New - may change
          dataSource: # Fetch from `GET /secrets?provider=provider-percona-server-mongodb&definition=import-credentials`
            # TODO: to match v1 implementation dataSource won't be necessary
          validation:
            required: true
```

**How it works:**
1. Dropdown populated via `GET /secrets?provider=provider-percona-server-mongodb&definition=import-credentials`
2. Shows existing secrets matching the definition
3. "Add New" button opens creation modal rendered from `definition/secrets/import-credentials/ui.yaml`
4. Schema validation uses `definition/secrets/import-credentials/secret.yaml` config

### 4.5. Settings

**Lifecycle Management:**

Some Secrets and ConfigMaps are shared across multiple Instances (e.g., SplitHorizon TLS certificates) and should persist after individual Instances are deleted.
Others are Instance-specific (e.g., database user credentials) and should be deleted when their owning Instance is removed.
Providers control lifecycle behavior by configuring whether to set owner references on Secrets and ConfigMaps.
Either owner reference to Instance or finalizers are set by the providers.
Resources with owner references are automatically garbage-collected when the owning Instance is deleted.

A Settings page is provided in the OpenEverest UI to manage Secrets and ConfigMaps that are no longer needed.
It allows viewing and deleting Secrets and ConfigMaps shared by multiple Instances.

Under Settings, a dedicated management page allows users to view and manage Secrets and ConfigMaps:

**Location:**
- Settings → Secrets
- Settings → ConfigMaps

**Layout:**
- **Tabs**: resources are grouped by:
  1. **Provider** (e.g., "provider-percona-server-mongodb")
  2. **Definition** within each provider (e.g., "splithorizon-tls", "import-credentials")

The list view displays each item and following actions: 
- **View**: GET (Secret data is not shown for security)
- **Delete**: DELETE with validation

### 4.6. RBAC

Access to Secret and ConfigMap management endpoints uses Casbin RBAC for authorization:

#### Casbin Policy Model

OpenEverest uses Casbin's RBAC with resource-based access control:
- `secrets` RBAC resource name for Secrets
- `config-maps` RBAC resource name for ConfigMaps

#### Casbin Policies

**Actions:**
- `create` - Create new Secret/ConfigMap
- `read` - List and view metadata
- `delete` - Delete Secret/ConfigMap

**Resource Pattern Format:**
```
{cluster}/{namespace}/{name}
```

**Wildcard Rules:**
- `*` matches any value at that level
- `**` not supported (use explicit wildcards at each level)

**Policy Examples:**
- `p, role:test, secrets, read, prod/*/*` - All secrets in a namespace
- `p, role:test, secrets, read, prod/ns1/*`  - Secrets in specific namespace
- `p, role:test, secrets, read, prod/ns1/secret1`  - Specific secret in specific namespace
- `p, role:test, config-maps, read, prod/*/*` - All ConfigMaps in a namespace

## 5. Definition of Done

- [ ] API endpoints implemented for Secrets (Create, List, Update, Delete)
- [ ] API endpoints implemented for ConfigMaps (Create, List, Get, Delete)
- [ ] Labels applied correctly on Secrets and ConfigMaps creation
- [ ] Management UI in Settings for viewing, updating, and deleting Secrets/ConfigMaps
- [ ] Delete prevents deletion of in-use resources
- [ ] RBAC roles and permissions configured for Secrets and ConfigMaps
- [ ] UI renders forms from provider UI schema
- [ ] PSMDB provider defines resource types for splithorizon component
- [ ] Integration tests for API and RBAC

## 6. Alternatives Considered

| Alternative | Decision | Reason |
|-------------|----------|--------|
| Embed secret config in Instance spec | Rejected | No reuse across Instances; large specs |
| Cross-provider shared secrets | Rejected | Adds complexity; each provider must declare identical schemas; fragile if schemas differ between providers; can be revisited if strong use case emerges |

## 7. Open Questions

1. **Secret data in GET:** Should GET return secret data?
   - No, only metadata (use kubectl for data)
2. **Creation of Secrets and ConfigMaps:** Should we allow creation of ConfigMap and Secret in Settings?
   - No for now, this may be requested later.

## 8. References

- https://github.com/openeverest/openeverest/issues/1798
- https://github.com/openeverest/openeverest/issues/2471
