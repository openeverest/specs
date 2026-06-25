# Presets

*   **Status:** Draft
*   **Authors:** @chilagrow
*   **Created:** 2026-06-04
*   **Last Updated:** 2026-06-25
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

### InstancePreset CR

InstancePreset is a cluster-scoped configuration template mirroring Instance spec. It is not a namespace-scoped resource to avoid users from defining InstancePresets for each namespace.
InstancePreset does not contain namespace-scoped resources and namespace-scoped fields are left empty.
InstancePreset API provides ways to pre-fill empty namespace-scoped fields (MonitoringConfig, Secrets) when fetched with namespace parameter.
Instances are created by copying values from InstancePreset and contain annotation references to the originating InstancePreset.

**Example InstancePreset CR:**

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: InstancePreset
metadata:
  name: mongodb-production
spec:
  provider: percona-server-mongodb
  
  components:
    engine:
      type: mongod
      replicas: 3
      resources:
        limits:
          cpu: "1"
          memory: 4Gi
      storage:
        size: 25Gi
        storageClass: "local-path"
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
      service: # Example to showcase usage for component spec not yet available
        serviceType: LoadBalancer
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-internal: "true"
        loadBalancerService:
          sourceRanges:
            - "192.168.1.100"

    monitoring:
      type: pmm
      customSpec:
        monitoringConfigName: "" # Namespace-scoped MonitoringConfig is empty - resolve from namespace default with annotation "openeverest.io/is-default-components-monitoring"
    
    splithorizon: # Example to showcase usage for component not yet available
      type: splithorizon
      config:
        secretRef:
          name: ""  # Namespace-scoped secret is empty - resolve from namespace default with annotation "openeverest.io/is-default-components-splithorizon"
      customSpec:
        domain: example.com
    
  topology:
    type: replicaSet
  
  version: "8.0.12"
```

#### Default Resource Pre-filling

When fetching a InstancePreset via API with a namespace parameter, namespace-scoped fields are pre-filled using annotation-based defaults from that namespace. If more than one default is set, the most recently created is selected.
This is the same approach used for Kubernetes `PVC` to discover `StorageClass` by looking for annotation `storageclass.kubernetes.io/is-default-class: "true"`.

Preset pre-filling determines which resource to look for by the field name:
- Field `monitoringConfig`, `monitoringConfigRef`, `monitoringConfigName` → Look for `monitoring.openeverest.io/v1alpha1` `MonitoringConfig` CRD
- Field `secret`, `secretRef`, `secretName` → Look for `Secret` resource

Instance may contain references to secrets such as Split Horizon Certificates.
The secret annotation requires additional information to determine which field they are used by.
The annotation uses `openeverest.io/is-default-components-{component-name}` format. The default annotation for secret used by `spec.components.splithorizon` is `openeverest.io/is-default-components-splithorizon: "true"`.

- `MonitoringConfig` → `openeverest.io/is-default-components-monitoring: "true"`
- `Secret` (for splithorizon) → `openeverest.io/is-default-components-splithorizon: "true"`

**Example: Annotating default**

```bash
kubectl annotate Secret <secret-name> \
  openeverest.io/is-default-components-splithorizon="true" \
  --overwrite
```

Or using a CR:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: splithorizon-tls
  namespace: prod
  annotations:
    openeverest.io/is-default-components-splithorizon: "true"
type: Opaque
data:
  ca.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpN...
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURGRENDQWZ5Z0F3SUJBZ0lVWTlLTWtlTC82RVhQaitWTjQ...
```

### Fetching Presets via API

#### API Endpoints

```
GET    /clusters/{cluster}/instance-presets                                 # List all presets
GET    /clusters/{cluster}/instance-presets?provider={provider}             # Filter by provider
GET    /clusters/{cluster}/instance-presets/{name}                          # Get specific preset
GET    /clusters/{cluster}/instance-presets/{name}/resolve?namespace={ns}   # Get preset with namespace defaults pre-filled
POST   /clusters/{cluster}/instance-presets                                 # Create instance preset
POST   /clusters/{cluster}/instance-presets/from-instance                   # Create instance preset from existing instance
PUT    /clusters/{cluster}/instance-presets/{name}                          # Update instance preset
DELETE /clusters/{cluster}/instance-presets/{name}                          # Delete instance preset
```

#### 1. List All Presets

Returns all installed presets with:
- Namespace-scoped fields empty (e.g., `monitoringConfigName` → "")

```
GET /clusters/{cluster}/instance-presets
```

**Response:** Array of preset objects.

```json
[{
  "apiVersion": "core.openeverest.io/v1alpha1",
  "kind": "InstancePreset",
  "metadata": {
    "name": "mongodb-production"
  },
  "spec": {
    "provider": "percona-server-mongodb",
    "components": {
      "engine": {
        "type": "mongod",
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
        },
        "service": { // Example to showcase usage for component spec not yet available
          "serviceType": "LoadBalancer",
          "annotations": {
            "service.beta.kubernetes.io/aws-load-balancer-internal": "true"
          },
          "loadBalancerService": {
            "sourceRanges": [
              "192.168.1.100"
            ]
          }
        }
      },
      "monitoring": {
        "type": "pmm",
        "customSpec": {
          "monitoringConfigName": "" // empty when namespace is unknown
        }
      },
      "splithorizon": { // Example to showcase usage for component not yet available
        "type": "splithorizon",
        "config": {
          "secretRef": {
            "name": ""  // empty when namespace is unknown
          }
        },
        "customSpec": {
          "domain": "example.com"
        }
      }
    },
    "topology": {
      "type": "replicaSet"
    },
    "version": "8.0.12"
  }
}]
```

#### 2. Filter by Provider

Returns presets for a specific provider.

```
GET /clusters/{cluster}/instance-presets?provider=percona-server-mongodb
```

**Response:** Same as above, filtered by provider.

#### 3. Get Specific Preset

Returns a specific preset with:
- Namespace-scoped fields empty (e.g., `monitoringConfigName` → "")

```
GET /clusters/{cluster}/instance-presets/{name}
```

**Response:** Preset object with empty namespace-scoped fields.

```json
{
  "apiVersion": "core.openeverest.io/v1alpha1",
  "kind": "InstancePreset",
  "metadata": {
    "name": "mongodb-production"
  },
  "spec": {
    "provider": "percona-server-mongodb",
    "components": {
      "engine": {
        "type": "mongod",
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
        },
        "service": { // Example to showcase usage for component spec not yet available
          "serviceType": "LoadBalancer",
          "annotations": {
            "service.beta.kubernetes.io/aws-load-balancer-internal": "true"
          },
          "loadBalancerService": {
            "sourceRanges": [
              "192.168.1.100"
            ]
          }
        }
      },
      "monitoring": {
        "type": "pmm",
        "customSpec": {
          "monitoringConfigName": "" // empty when namespace is unknown
        }
      },
      "splithorizon": { // Example to showcase usage for component not yet available
        "type": "splithorizon",
        "config": {
          "secretRef": {
            "name": ""  // empty when namespace is unknown
          }
        },
        "customSpec": {
          "domain": "example.com"
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

#### 4. Get InstancePreset with Namespace Defaults

Presets API has mechanism to pre-fill namespace defaults.
When `namespace` query parameter is provided, the API returns the InstancePreset CR with namespace defaults pre-filled.

Returns a specific preset with:
- Namespace-scoped fields resolved (e.g., `monitoringConfigName` → "config")

```
GET /clusters/{cluster}/instance-presets/{name}/resolve?namespace={ns}
```

**Response:** InstancePreset object with pre-filled namespace-scoped fields.

```json
{
  "apiVersion": "core.openeverest.io/v1alpha1",
  "kind": "InstancePreset",
  "metadata": {
    "name": "mongodb-production"
  },
  "spec": {
    "provider": "percona-server-mongodb",
    "components": {
      "engine": {
        "type": "mongod",
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
        },
        "service": { // Example to showcase usage for component spec not yet available
          "serviceType": "LoadBalancer",
          "annotations": {
            "service.beta.kubernetes.io/aws-load-balancer-internal": "true"
          },
          "loadBalancerService": {
            "sourceRanges": [
              "192.168.1.100"
            ]
          }
        }
      },
      "monitoring": {
        "type": "pmm",
        "customSpec": {
          "monitoringConfigName": "config" // pre-filled from MontoringConfig CR with default annotation "openeverest.io/is-default-components-monitoring"
        }
      },
      "splithorizon": { // Example to showcase usage for component not yet available
        "type": "splithorizon",
        "config": {
          "secretRef": {
            "name": "splithorizon-tls" // pre-filled from Secret with default annotation "openeverest.io/is-default-components-splithorizon"
          }
        },
        "customSpec": {
          "domain": "example.com"
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

#### 5. Create InstancePreset

Creates a new InstancePreset.
InstancePreset does not accepts namespace-scoped resource reference.
If the instance contains non-empty namespace-scoped resource reference such as MonitoringConfig name, it returns 400 code.

```
POST /clusters/{cluster}/instance-presets
```

**Request Body:** Complete InstancePreset specification.

```json
{
  "apiVersion": "core.openeverest.io/v1alpha1",
  "kind": "InstancePreset",
  "metadata": {
    "name": "mongodb-production"
  },
  "spec": {
    "provider": "percona-server-mongodb",
    "components": {
      "engine": {
        "type": "mongod",
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
        },
        "service": {
          "serviceType": "LoadBalancer",
          "annotations": {
            "service.beta.kubernetes.io/aws-load-balancer-internal": "true"
          },
          "loadBalancerService": {
            "sourceRanges": [
              "192.168.1.100"
            ]
          }
        }
      },
      "monitoring": {
        "type": "pmm",
        "customSpec": {
          "monitoringConfigName": ""
        }
      },
      "splithorizon": {
        "type": "splithorizon",
        "config": {
          "secretRef": {
            "name": ""
          }
        },
        "customSpec": {
          "domain": "example.com"
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

**Response:** Created InstancePreset object.

#### 6. Create InstancePreset from Instance

Creates a new InstancePreset based on an existing Instance configuration.

```
POST /clusters/{cluster}/instance-presets/from-instance
```

**Request Body:** Preset name and source instance details.

```json
{
  "name": "mongodb-custom",
  "sourceInstanceName": "my-mongodb",
  "sourceInstanceNamespace": "prod"
}
```

**Response:** Created InstancePreset object with namespace-scoped fields stripped.

```json
{
  "apiVersion": "core.openeverest.io/v1alpha1",
  "kind": "InstancePreset",
  "metadata": {
    "name": "mongodb-custom"
  },
  "spec": {
    "provider": "percona-server-mongodb",
    "components": {
      "engine": {
        "type": "mongod",
        "replicas": 5,
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
        },
        "service": {
          "serviceType": "LoadBalancer",
          "annotations": {
            "service.beta.kubernetes.io/aws-load-balancer-internal": "true"
          },
          "loadBalancerService": {
            "sourceRanges": [
              "192.168.1.100"
            ]
          }
        }
      },
      "monitoring": {
        "type": "pmm",
        "customSpec": {
          "monitoringConfigName": ""
        }
      },
      "splithorizon": {
        "type": "splithorizon",
        "config": {
          "secretRef": {
            "name": ""
          }
        },
        "customSpec": {
          "domain": "example.com"
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

**Behavior:**
1. Server fetches the specified Instance from the given namespace
2. Server extracts the Instance spec
3. Server strips namespace-scoped fields (e.g., `monitoringConfigName`, secret references)
4. Server validates the preset specification
5. Server creates InstancePreset CR

#### 7. Update InstancePreset

Updates an existing InstancePreset specification.
If the instance contains non-empty namespace-scoped resource reference such as MonitoringConfig name, it returns 400 code.

```
PUT /clusters/{cluster}/instance-presets/{name}
```

**Request Body:** Updated InstancePreset specification.

```json
{
  "apiVersion": "core.openeverest.io/v1alpha1",
  "kind": "InstancePreset",
  "metadata": {
    "name": "mongodb-production"
  },
  "spec": {
    "provider": "percona-server-mongodb",
    "components": {
      "engine": {
        "type": "mongod",
        "replicas": 5,
        "resources": {
          "limits": {
            "cpu": "2",
            "memory": "8Gi"
          }
        },
        "storage": {
          "size": "50Gi",
          "storageClass": "local-path"
        }
      },
      "monitoring": {
        "type": "pmm",
        "customSpec": {
          "monitoringConfigName": ""
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

**Response:** Updated InstancePreset object.

#### 8. Delete InstancePreset

Deletes an existing InstancePreset.

```
DELETE /clusters/{cluster}/instance-presets/{name}
```

**Response:** 204 No Content on success.

### Instance CR

Instance CR spec is explicitly defined from InstancePreset CR. The `openeverest.io/preset: {name}` annotation is used for RBAC check for users who can create Instance from InstancePreset defaults but cannot change from the defaults.

**Example Instance CR:**

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: Instance
metadata:
  name: my-mongodb
  namespace: prod
  annotations:
    # Preset tracking
    openeverest.io/instance-preset: "mongodb-production"

spec:
  # Full resolved configuration (self-contained, no references to Preset)
  provider: percona-server-mongodb
  
  components:
    engine:
      replicas: 5  # USER OVERRIDE - different from preset's 3
      resources:  # FROM PRESET
        limits:
          cpu: "1"
          memory: 4Gi
      storage:  # FROM PRESET
        size: 25Gi
        storageClass: local-path
      affinity:  # FROM PRESET
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                topologyKey: kubernetes.io/hostname
      service: # Example to showcase usage for component spec not yet available
        serviceType: LoadBalancer
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-internal: "true"
        loadBalancerService:
          sourceRanges:
            - "192.168.1.100"b

    monitoring:
      customSpec:
        monitoringConfigName: config  # RESOLVED from namespace default
    
    splithorizon: # Example to showcase usage for component not yet available
      type: splithorizon
      config:
        secretRef:
          name: splithorizon-tls  # RESOLVED from namespace default
      customSpec:
        domain: example.com
    
  topology:
    type: replicaSet
  
  version: "8.0.12"
```

### OpenEverest UI

The OpenEverest UI fetches available Presets from a provider.

**Phase 1 UI Flow for creating Instance:**

1. **Select field with list of presets**: UI fetches available presets for selected provider
   ```
   GET /clusters/{cluster}/instance-presets?provider=percona-server-mongodb
   ```

2. **User Selects Namespace**:

3. **User Selects Preset**: UI fetches preset with defaults pre-filled
   ```
   GET /clusters/${cluster}/instance-presets/mongodb-production/resolve?namespace=prod
   ```

4. **Pre-fill Form**: UI populates Instance creation form with Preset values

5. **User Overrides** (Phase 1: read-only preview, Phase 2: editable): User can review values

6. **Create Instance**: UI constructs and submits Instance CR

### InstancePreset Management UI

InstancePreset management (creating, editing, deleting presets) can be delivered through two approaches:

**Option 1: Settings Integration**
- Add a "Presets" section in the Settings UI
- Provides centralized management alongside other cluster-wide resources
- Users can CRUD presets from Settings → Presets

**Option 2: Generic Plugin**
- Deliver preset management as a Generic Plugin
- Keeps core UI simpler, optional installation
- Plugin provides dedicated preset management interface

The choice between these approaches can be made during Phase 2 implementation based on UI complexity and maintainability considerations.

Both approaches use the same underlying API endpoints (POST, PUT, DELETE `/instance-presets`).

#### Prevent users from modifying defaults on One-click deployment

Some users can deploy Instance with one click, but they are unable to edit Instance from the Preset defaults.
The new Casbin RBAC permission `deploy` Instance is added for users who can create instances from presets only.
This is a lower level permission than `create` Instance permission which allows users to define custom instance specs.

Users with `deploy` permissions require the `openeverest.io/instance-preset` annotation before submitting the Instance CR.
Any deviation from Preset defaults is checked by the server.

**Permission Levels for Instance:**
- **read** - Can view instances
- **deploy (NEW)** - Can create instances from presets only (subset of create)
- **create** - Can create any instance (includes deploy)
- **update** - Can modify existing instances
- **delete** - Can delete instances

Following checks are made upon creating an Instance is InstancePreset is defined in the annotation.

1. User with read InstancePreset permission? No -> reject
2. User with create Instance permission? Yes -> creation Instance
3. User with deploy Instance permission? No -> reject
4. Check Instance spec, is it modified from Preset? Yes -> reject, No -> create Instance

##### Example 1

Alice has access to Preset A and Bob has access to Preset B.

1. Give Alice `read` permission to Preset A, give Bob `read` permission to Preset B.

```
p, role:teama, instance-presets, read, {cluster}/preset-a
p, role:teamb, instance-presets, read, {cluster}/preset-b
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
p, chris, instance-presets, read, {cluster}/preset-c
```

2. Give Chris `create` permission to Instances.

```
p, chris, instances, create, {cluster}/{namespace}/*
```

### Phased Rollout

#### 🚀 Phase 1 (Initial Release)

**Goal:** One-click deployment with pre-configured presets

**Validation**
- The pre-installed InstancePreset CR ships with known-valid values.

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

**Validation**
- A InstancePreset CR follows the same validation rules as an Instance CR.
- Namespace-scoped and cluster-scoped fields are left empty and disabled from modifying.

**User Stories:**
- As a user, I can edit preset values before creating an Instance
- As a user, I can create a new preset based on a running Instance
- As an admin, I can create and manage presets in the UI
- As an admin, I can control which presets are visible to which users/roles

## 5. Definition of Done

**Phase 1:** InstancePreset CRD + pre-installed CRs + API with default resolution + UI selector

**Phase 2:** Editable Instance UI (or Generic Plugin) + InstancePreset CRUD API + RBAC + "Create Preset from instance"

## 6. Alternatives Considered

**Rejected:**
- ConfigMap-based: No validation, versioning
- Preset reference in Instance.spec: Not self-contained requiring merging Preset and Instance.Spec in webhook or in-memory in controller
- Client-side pre-filling: Complex UI logic, inconsistent

**Chosen:** Cluster-scoped CR mirroring Instance spec, backend API pre-fills defaults, annotation tracking, instances detached after creation.

## 7. Open Questions

- Preset versioning during Helm upgrades
- Error handling: Deleted defaults, secret validation, deleted preset

## 8. References

- https://github.com/openeverest/openeverest/issues/1820
