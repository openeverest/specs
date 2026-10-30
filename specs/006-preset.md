# Presets

*   **Status:** Draft
*   **Authors:** @chilagrow
*   **Created:** 2026-06-04
*   **Last Updated:** 2026-10-30
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

**Non-Goals:**
*   Cross-provider sharing of presets — Presets are scoped to a single provider and cannot be shared across providers.
*   Supporting One-click deployment via `kubectl`.
*   User-facing preset creation/editing UI in Phase 1.
*   Bulk update propagation. This may be handled in future.

## 4. Proposed Solution / Design

### Preset CRs

`ClusterInstancePreset` CR is a cluster-scoped configuration template mirroring Instance spec.
It is installed with each provider or created by admin user to have template that can be used in any namespace.
It can only be created and managed by admins using `kubectl` or equivalent.
It must not contain namespace specific references. If you need namespace specific reference, you need to create `InstancePreset`.

`InstancePreset` CR is a namespace-scoped configuration template, the spec also mirrors Instance, it may contain namespace specific references and allows users to create, update and delete them via OpenEverest API.

Instances are created by copying values from `ClusterInstancePreset` or `InstancePreset` and contain annotation references to the originating Preset.

**Example ClusterInstancePreset CR:**

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: ClusterInstancePreset
metadata:
  name: psmdb-replicaset
spec:
  providerRef:
    name: percona-server-mongodb
  version: "8.0.32"
  topology:
    type: replicaSet
  components:
    engine:
      storage:
        size: 25Gi
      resources:
        limits:
          cpu: "1"
          memory: 4Gi
      replicas: 3
      schedulingPolicy:
        affinity:
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
      service:
        serviceType: ClusterIP
  deletionPolicy: Cascade
```

**Example InstancePreset CR:**

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: InstancePreset
metadata:
  name: psmdb-default
  namespace: my-special-place
spec:
  providerRef:
    name: percona-server-mongodb
  version: "8.0.32"
  topology:
    type: replicaSet
  parameters:
    tls:
      issuer:
        kind: Issuer
      mode: preferTLS
  components:
    engine:
      storage:
        size: 25Gi
        storageClass: local-path
      resources:
        limits:
          cpu: "1"
          memory: 4Gi
      replicas: 3
      service:
        serviceType: ClusterIP
  backup:
    enabled: true
    classRef:
      name: percona-backup-mongodb
    storages:
      - storageRef:
          name: bs-msp-1 # namespace-scoped resource reference
  deletionPolicy: Cascade
```

### Fetching Presets via API

#### API Endpoints

```
GET /clusters/{cluster}/cluster-instance-presets                                 # List all presets
GET /clusters/{cluster}/cluster-instance-presets?provider={provider}             # Filter by provider
GET /clusters/{cluster}/cluster-instance-presets/{name}                          # Get specific preset
```

```
GET /clusters/{cluster}/namespaces/{ns}/instance-presets                         # List all presets
GET /clusters/{cluster}/namespaces/{ns}/instance-presets?provider={provider}     # Filter by provider
POST /clusters/{cluster}/namespaces/{ns}/instance-presets                        # Create a preset
GET /clusters/{cluster}/namespaces/{ns}/instance-presets/{name}                  # Get specific preset
PUT /clusters/{cluster}/namespaces/{ns}/instance-presets/{name}                  # Update specific preset
DELETE /clusters/{cluster}/namespaces/{ns}/instance-presets/{name}               # Delete specific preset
```

#### 1. List All ClusterInstancePresets

```
GET /clusters/{cluster}/cluster-instance-presets
```

**Response:** List object with `items` containing ClusterInstancePresets.

```json
{
    "items": [
        {
            "kind": "ClusterInstancePreset",
            "apiVersion": "core.openeverest.io/v1alpha1",
            "metadata": {
                "name": "psmdb-replicaset"
            },
            "spec": {
                "providerRef": {
                    "name": "percona-server-mongodb"
                },
                "version": "8.0.32",
                "topology": {
                    "type": "replicaSet"
                },
                "components": {
                    "engine": {
                        "storage": {
                            "size": "25Gi"
                        },
                        "resources": {
                            "limits": {
                                "cpu": "1",
                                "memory": "4Gi"
                            }
                        },
                        "replicas": 3,
                        "schedulingPolicy": {
                            "affinity": {
                                "podAntiAffinity": {
                                    "preferredDuringSchedulingIgnoredDuringExecution": [
                                        {
                                            "weight": 1,
                                            "podAffinityTerm": {
                                                "topologyKey": "kubernetes.io/hostname"
                                            }
                                        }
                                    ]
                                }
                            }
                        },
                        "service": {
                            "serviceType": "ClusterIP"
                        }
                    }
                },
                "deletionPolicy": "Cascade"
            }
        }
    ]
}
```

#### 2. Filter by Provider

Returns presets for a specific provider.

```
GET /clusters/{cluster}/cluster-instance-presets?provider=percona-server-mongodb
```

**Response:** Same as above, filtered by provider.

#### 3. Get Specific Preset

```
GET /clusters/{cluster}/cluster-instance-presets/{name}
```

**Response:** Same as an item of list

#### 4. List all InstancePreset

```
GET /clusters/{cluster}/namespaces/{namespace}/instance-presets
```

**Response:** List object with `items` containing InstancePresets. `bs-msp-1` is a namespace-scoped resource reference.

```json
{
    "items": [
        {
            "kind": "InstancePreset",
            "apiVersion": "core.openeverest.io/v1alpha1",
            "metadata": {
                "name": "psmdb-default",
                "namespace": "my-special-place"
            },
            "spec": {
                "providerRef": {
                    "name": "percona-server-mongodb"
                },
                "version": "8.0.32",
                "topology": {
                    "type": "replicaSet"
                },
                "parameters": {
                    "tls": {
                        "issuer": {
                            "kind": "Issuer"
                        },
                        "mode": "preferTLS"
                    }
                },
                "components": {
                    "engine": {
                        "storage": {
                            "size": "25Gi",
                            "storageClass": "local-path"
                        },
                        "resources": {
                            "limits": {
                                "cpu": "1",
                                "memory": "4Gi"
                            }
                        },
                        "replicas": 3,
                        "service": {
                            "serviceType": "ClusterIP"
                        }
                    }
                },
                "backup": {
                    "enabled": true,
                    "classRef": {
                        "name": "percona-backup-mongodb"
                    },
                    "storages": [
                        {
                            "storageRef": {
                                "name": "bs-msp-1"
                            }
                        }
                    ]
                },
                "deletionPolicy": "Cascade"
            }
        }
    ]
}
```

#### 5. Create InstancePreset

```
POST /clusters/{cluster}/namespaces/{namespace}/instance-presets
```

**Request:**

```json
{
    "apiVersion": "core.openeverest.io/v1alpha1",
    "kind": "InstancePreset",
    "metadata": {
        "name": "psmdb-default"
    },
    "spec": {
        "providerRef": {
            "name": "percona-server-mongodb"
        },
        "topology": {
            "type": "replicaSet"
        },
        "backup": {
            "classRef": {
                "name": "percona-backup-mongodb"
            },
            "enabled": true,
            "storages": [
                {
                    "name": "bs-msp-1",
                    "storageRef": {
                        "name": "bs-msp-1"
                    }
                }
            ]
        },
        "components": {
            "engine": {
                "service": {
                    "serviceType": "ClusterIP"
                },
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
                }
            }
        },
        "version": "8.0.32"
    }
}
```

**Response:** Created InstancePreset

#### 6. Get InstancePreset

```
GET /clusters/{cluster}/namespaces/{namespace}/instance-presets/{name}
```

**Response:** An item of the list response

#### 7. Update InstancePreset

```
PUT /clusters/{cluster}/namespaces/{namespace}/instance-presets/{name}
```

**Request:** Same as the create request body.

#### 8. Delete InstancePreset

```
DELETE /clusters/{cluster}/namespaces/{namespace}/instance-presets/{name}
```

**Response:** No content

### Instance CR

Instance CR spec is explicitly defined from InstancePreset CR or ClusterInstancePreset CR.
The `openeverest.io/instance-preset: {name}` and `openeverest.io/cluster-instance-preset: {name}` annotations are used for RBAC check for users who can create Instance from preset defaults but cannot change from the defaults.

**Example Instance CR:**

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: Instance
metadata:
  name: my-mongodb
  namespace: my-special-place
  annotations:
    # Preset tracking
    openeverest.io/instance-preset: "psmdb-default"
# Full configuration copied from the preset (self-contained, no references to Preset)
spec:
  providerRef:
    name: percona-server-mongodb
  version: "8.0.32"
  topology:
    type: replicaSet
  parameters:
    tls:
      issuer:
        kind: Issuer
      mode: preferTLS
  components:
    engine:
      storage:
        size: 25Gi
        storageClass: local-path
      resources:
        limits:
          cpu: "1"
          memory: 4Gi
      replicas: 3
      service:
        serviceType: ClusterIP
  backup:
    enabled: true
    classRef:
      name: percona-backup-mongodb
    storages:
      - storageRef:
          name: bs-msp-1
  deletionPolicy: Cascade
```

### OpenEverest UI

The OpenEverest UI fetches available Presets from a provider.

**Phase 1 UI Flow for creating Instance:**

1. **Select field with list of presets**: UI fetches both `ClusterInstancePresets` and `InstancePresets` for selected provider. (Phase 1: support cluster-scope `ClusterInstancePresets` only, Phase 2: also support namespace-scope `InstancePresets` and user selection of namespace happens before fetching `InstancePreset`)
   ```
   GET /clusters/{cluster}/cluster-instance-presets?provider=percona-server-mongodb
   ```

   ```
   GET /clusters/{cluster}/namespaces/{namespace}/instance-presets?provider=percona-server-mongodb
   ```

2. **User Selects Namespace**:

3. **User Selects Preset**:

4. **Pre-fill Form**: UI populates Instance creation form with the selected Preset values

5. **User Overrides** (Phase 1: read-only preview, Phase 2: editable): User can review values

6. **Create Instance**: UI constructs and submits Instance CR

#### Prevent users from modifying defaults on One-click deployment

Some users can deploy Instance with one click, but they are unable to edit Instance from the Preset defaults.
The new Casbin RBAC permission `deploy` Instance is added for users who can create instances from presets only.
This is a lower level permission than `create` Instance permission which allows users to define custom instance specs.

Users with `deploy` permissions require the `openeverest.io/cluster-instance-preset` annotation for selected `ClusterInstancePreset` or `openeverest.io/instance-preset` annotation for `InstancePreset` before submitting the Instance CR.
Any deviation from Preset defaults is checked by the server.

**Permission Levels for Instance:**
- **read** - Can view instances
- **deploy (NEW)** - Can create instances from presets only (subset of create)
- **create** - Can create any instance (includes deploy)
- **update** - Can modify existing instances
- **delete** - Can delete instances

The following checks are made upon creating an Instance if the `openeverest.io/instance-preset` or `openeverest.io/cluster-instance-preset` annotation is defined.

1. User with create Instance permission? Yes -> create Instance
2. User with read permission on the referenced InstancePreset or ClusterInstancePreset? No -> reject
3. User with deploy Instance permission? No -> reject
4. Check Instance spec, is it modified from Preset? Yes -> reject, No -> create Instance

##### Example 1

Alice has access to Preset A and Bob has access to Preset B.

1. Give Alice `read` permission to Preset A, give Bob `read` permission to Preset B.

```
p, role:teama, cluster-instance-presets, read, {cluster}/preset-a
p, role:teamb, cluster-instance-presets, read, {cluster}/preset-b
g, alice, role:teama
g, bob, role:teamb
```

2. Give Alice and Bob `deploy` permission to Instances.

```
p, role:deployonly, instances, deploy, {cluster}/{namespace}/*
g, alice, role:deployonly
g, bob, role:deployonly
```

#### Example 2

Chris can modify Instance from Preset defaults.

1. Give Chris `read` permission to Preset C.

```
p, chris, instance-presets, read, {cluster}/{namespace}/preset-c
```

2. Give Chris `create` permission to Instances.

```
p, chris, instances, create, {cluster}/{namespace}/*
```

### Phased Rollout

#### 🚀 Phase 1 (Initial Release)

**Goal:** One-click deployment with pre-configured presets

**Validation**
- The pre-installed ClusterInstancePreset CR ships with known-valid values.

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
Users can create `Instance` from namespace-scoped `InstancePreset`.
Users are able to edit Instances populated from `ClusterInstancePreset` and `InstancePreset`.
Users can create a `InstancePreset` from an Instance.
It also gives users the ability to manage presets on OpenEverest UI. While UI specific for presets can be integrated into OpenEverest, an alternative is implementing preset management via Generic Plugin.

**Validation**
- A `InstancePreset` CR follows the same validation rules as an Instance CR.

**User Stories:**
- As a user, I can edit preset values before creating an Instance
- As a user, I can create a new preset based on a running Instance
- As an admin, I can create and manage presets in the UI
- As an admin, I can control which presets are visible to which users/roles

## 5. Definition of Done

**Phase 1:** InstancePreset CRD + pre-installed CRs + API + UI selector

**Phase 2:** Editable Instance UI (or Generic Plugin) + InstancePreset CRUD API + RBAC + "Create Preset from instance"

## 6. Alternatives Considered

**Rejected:**
- ConfigMap-based: No validation, versioning
- Preset reference in Instance.spec: Not self-contained requiring merging Preset and Instance.Spec in webhook or in-memory in controller
- Client-side pre-filling: Complex UI logic, inconsistent
- Cluster-scoped CR with namespace-specific values filled from annotation defaults.

**Chosen:** Cluster-scoped and namespace-scoped CRs mirroring Instance spec (namespace-specific values live in InstancePreset instead of being pre-filled by the backend), annotation tracking, instances detached after creation.

## 7. Open Questions

- Preset versioning during Helm upgrades
- Error handling: Deleted defaults, secret validation, deleted preset

## 8. References

- https://github.com/openeverest/openeverest/issues/1820
