# Presets

*   **Status:** Draft
*   **Authors:** @chilagrow
*   **Created:** 2026-06-04
*   **Last Updated:** 2026-06-04
*   **Related Issues:** [openeverest/openeverest#1820]

---

## 1. Summary

A **Preset** is a provider-specific default configuration for an OpenEverest Instance. It replaces the current ad-hoc configuration patterns (split horizon config, load balancer annotations, pod scheduling policy) with a unified mechanism, enabling one-click deployment with sensible defaults while allowing users to override values as needed.

## 2. Motivation

Currently, deploying an Instance requires users to manually configure each component (replicas, storage, resources, backup, proxy, load balancer annotations, scheduling, etc.), which is error-prone and time-consuming. There is no unified way to ship or share default configurations. Presets address this by providing a structured, provider-scoped default configuration that can be installed with OpenEverest and applied automatically during Instance creation.

## 3. Goals & Non-Goals

**Goals:**
- Phase 1: One-click deployment with defaults
- Phase 2: Override values, preset management in UI
- Phase 3: Bulk updates

**Non-Goals:**
*   Cross-provider sharing of presets — Presets are scoped to a single provider and cannot be shared across providers.
*   Supporting One-click deployment via `kubectl`.
*   User-facing preset creation/editing UI in Phase 1.
*   Bulk update propagation in Phase 1.

## 4. Proposed Solution / Design

### Preset CR

Preset is a cluster-scoped configuration template mirroring Instance spec. It is not a namespace-scoped resource to avoid users from defining Presets for each namespace. Naturally, Preset does not contain namespace-scoped resources and those fields are left empty. Preset API provides ways to pre-fill empty namespace-scoped fields (StorageClass, MonitoringConfig, Secrets) when fetched with namespace parameter. Instances are created by copying values from Preset and contain annotation references to the originating Preset. 

**Example Preset CR:**

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: Preset
metadata:
  name: mongodb-production
  annotations:
    openeverest.io/preset: "mongodb-production"
spec:
  provider: percona-server-mongodb
  
  components:
    engine:
      replicas: 3
      resources:
        limits:
          cpu: "1"
          memory: 4Gi
      storage:
        size: 25Gi
        storageClass: ""  # Empty - resolved from cluster default
    monitoring:
      customSpec:
        monitoringConfigName: ""  # Empty - resolved from namespace default once namespace is supplied
    
  topology:
    type: replicaSet
  
  version: "8.0.12"
```

#### Default Resource Pre-filling

When fetching a Preset via API with a namespace parameter, empty fields are pre-filled using annotation-based defaults from that namespace/cluster. This is the approach used for Kubernetes `PVC` to discover `StorageClass` by looking for annotation `storageclass.kubernetes.io/is-default-class: "true"`. If more than one `StorageClass` is set default, the most recently created `StorageClass` is selected.

Preset pre-filling determines which resource to look for by the field name:
- Field `storageClass`, `storageClassRef`, `storageClassName` → Look for `storageclass.kubernetes.io` `StorageClass` resource
- Field `monitoringConfig`, `monitoringConfigRef`, `monitoringConfigName` → Look for `monitoring.openeverest.io/v1alpha1` `MonitoringConfig` CRD
- Field `secret`, `secretRef`, `secretName` → Look for `Secret` resource
- Field `configMap`, `configMapRef`, `configMapName` → Look for `ConfigMap` resource

While default annotation is sufficient for CRs, secrets and configMaps require additional information to determine which field they are used by. Instance CR may reference multiple secrets and configMaps. For Kubernetes resource `StorageClass`, the annotation set by Kubernetes is used. For OpenEverest CRs, `openeverest.io/is-default: "true"` is used.

- `StorageClass` → `storageclass.kubernetes.io/is-default-class: "true"`
- `MonitoringConfig` → `openeverest.io/is-default: "true"`

For secrets and configMaps, additional path information is included in the annotation using `openeverest.io/is-default-{field-path}` format. The default annotation for the field `spec.components.splithorizon.customSpec.tls.secretName` is `openeverest.io/is-default-components-splithorizon-customspec-tls: "true"`.

The annotation creation is done using CLI: 

```
kubectl annotate MonitoringConfig <monitoring-config-name> openeverest.io/is-default="true" --overwrite
```

#### Fetching Presets via API

```
GET /clusters/{cluster}/presets
GET /clusters/{cluster}/presets?provider={provider}
GET /clusters/{cluster}/presets/{name}
GET /clusters/{cluster}/presets/{name}?namespace={namespace}
```

Presets are fetched through a RESTful API that pre-fills namespace/cluster defaults and prepares Instance configuration for the UI.

When `namespace` query parameter is provided, the API returns the Preset CR with namespace/cluster defaults pre-filled:

```
GET /clusters/{cluster}/presets/{name}?namespace={namespace}
```

```json
{
  "apiVersion": "core.openeverest.io/v1alpha1",
  "kind": "Preset",
  "metadata": {
    "name": "mongodb-production",
    "annotations": {
      "openeverest.io/preset": "mongodb-production"
    }
  },
  "spec": {
    "provider": "percona-server-mongodb",
    "components": {
      "engine": {
        "replicas": 3,
        "resources": {
          "limits": {
            "cpu": "1",
            "memory": "4Gi"
          }
        },
        "storage": {
          "size": "25Gi",
          "storageClass": "local-path"
        },
      },
      "monitoring": {
        "customSpec": {
          "monitoringConfigName": "config"
        }
      }
    },
    "topology": {
      "type": "replicaSet"
    },
    "version": "8.0.12"
  }
}
```

### Instance CR

Instance CR spec is explicitly defined from Preset CR. The `openeverest.io/preset: {name}` annotation is added for tracking.

**Example Instance CR:**

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: Instance
metadata:
  name: my-mongodb
  namespace: prod
  annotations:
    # Preset tracking
    openeverest.io/preset: "mongodb-production"

spec:
  # Full resolved configuration (self-contained, no references to Preset)
  provider: percona-server-mongodb
  
  components:
    engine:
      replicas: 5  # USER OVERRIDE - different from preset's 3
      resources:
        limits:
          cpu: "1"          # FROM PRESET
          memory: 4Gi       # FROM PRESET
      storage:
        size: 25Gi
        storageClass: local-path  # RESOLVED from cluster default
    monitoring:
      customSpec:
        monitoringConfigName: config  # RESOLVED from namespace default
    
  topology:
    type: replicaSet
  
  version: "8.0.12"
```

### OpenEverest UI

The OpenEverest UI fetches available Presets from a provider.

**Phase 1 UI Flow for creating Instance:**

1. **List Presets**: UI fetches available presets for selected provider
   ```
   GET /clusters/{cluster}/presets?provider=percona-server-mongodb
   ```

2. **User Selects Preset and namespace**: UI fetches preset with defaults pre-filled
   ```
   GET /clusters/${cluster}/presets/mongodb-production?namespace=prod
   ```

3. **Pre-fill Form**: UI populates Instance creation form with Preset values

4. **User Overrides** (Phase 1: read-only preview, Phase 2: editable): User can review values

5. **Create Instance**: UI constructs and submits Instance CR

### Phased Rollout

#### 🚀 Phase 1 (Initial Release)

**Goal:** One-click deployment with pre-configured presets

### Validation
- The pre-installed Preset CR ships with known-valid values.

**User Stories:**
- As a user, I can deploy an Instance with one click, and the system applies a preset automatically
- As a user, I can select which preset to use when multiple presets exist
- As a user, I can install OpenEverest with presets already configured

**What's NOT in Phase 1:**
- Editing instance from preset values in UI (form is read-only/preview)
- Creating custom presets in UI
- Bulk updates when preset changes

---

#### 🔮 Phase 2 (Custom Values & Preset Management)

**Goal:** Users can edit Instance values during Instance creation from preset and admins can manage custom presets

**Overview:**
Users are able to edit Instances populated from presets.
Users can create a preset from an Instance.
It also gives users the ability to manage presets on OpenEverest UI. While UI specific for presets can be integrated into OpenEverest, an alternative is implementing preset management via Generic Plugin.

### Validation
- A Preset CR follows the same validation rules as an Instance CR.
- Namespace-scoped and cluster-scoped fields are left empty and disabled from modifying.

**User Stories:**
- As a user, I can edit preset values before creating an Instance
- As a user, I can create a new preset based on a running Instance
- As an admin, I can create and manage presets in the UI
- As an admin, I can control which presets are visible to which users/roles

---

#### 📦 Phase 3 (Bulk Updates)

**Goal:** Help users apply preset updates

**Overview:**
Phase 1 & 2 keep Instances fully detached from Presets after creation. Phase 3 adds detection and user-approval or automatic workflow for preset updates.

The default flow for updating Preset is not to update Instances using the Preset. Updating Instance is disruptive and should not be updated without user approval. The instance may be assigned a preset policy which explicitly allows updates upon Preset change. 

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: Instance
metadata:
  name: my-mongodb
  annotations:
    openeverest.io/preset: "mongodb-production"
    openeverest.io/preset-policy: "auto-update" # manual (default) or auto-update
spec:
  # ... instance configuration
```

Manual approval flow in UI may be implemented (upon needs):

1. Upon an admin updates a Preset, lists Instances with `openeverest.io/preset: {preset-name}` annotation
2. Shows admin impact analysis before saving: "3 instances will be affected"
3. Admin reviews proposed changes for their Instances and approves/rejects per Instance
4. Updating Preset triggers updating approved instances

**User Stories:**
- As an admin, I can update a preset and see which Instances reference it
- As a user, I can review what would change in my Instance
- As a user, I can approve or reject the update per Instance
- As an admin, I can see which Instances are "out of sync" with latest preset

## 5. Definition of Done

**Phase 1:** Preset CRD + pre-installed CRs + API with default resolution + UI selector

**Phase 2:** Editable Instance UI (or Generic Plugin) + Preset CRUD API + RBAC + "Create Preset from instance"

**Phase 3:** Controller + policy annotations

## 6. Alternatives Considered

**Rejected:**
- ConfigMap-based: No validation, versioning
- Preset reference in Instance.spec: Not self-contained requiring merging Preset and Instance.Spec in webhook or in-memory in controller
- Client-side pre-filling: Complex UI logic, inconsistent

**Chosen:** Cluster-scoped CR mirroring Instance spec, backend API pre-fills defaults, annotation tracking, instances detached after creation.

## 7. Open Questions

- Preset versioning during Helm upgrades
- Phase 3: Default policy (`manual` vs `auto-update`), rollback mechanism, controller scope
- Error handling: Deleted defaults, secret validation, deleted preset

## 8. References

- https://github.com/openeverest/openeverest/issues/1820
