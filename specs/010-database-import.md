# Database Import via BackupClass Expansion

*   **Status:** Draft
*   **Authors:** @chilagrow
*   **Created:** 2026-06-26
*   **Last Updated:** 2026-06-29
*   **Related Issues:** https://github.com/openeverest/openeverest/issues/2471

---

## 1. Summary

Enable data import functionality in OpenEverest v2 by extending the existing BackupClass and Restore CRs to support external data sources. Instead of creating separate DataImporter/DataImportJob CRs (as in v1), treat data imports as "restores from external sources" — reusing the existing backup/restore infrastructure with a new `DataSourceType.External` that references an external storage location.

## 2. Motivation

### Current State (v1 openeverest-operator)

The v1 operator implements data import through dedicated CRs:
- **DataImporter** (cluster-scoped): Defines an import method (image, command, config schema, RBAC)
- **DataImportJob** (namespaced): Represents an active import operation with inline S3 source details

This creates:
- **Duplicate infrastructure**: Job execution, RBAC management, payload contracts, and status tracking are reimplemented separately from backup/restore
- **API surface bloat**: Additional RBAC resources (`data-importers`, `data-import-jobs`) and distinct lifecycle management
- **Inconsistent UX**: Different concepts for "restore from backup" vs "import from external source" despite similar underlying operations

### Why Change?

A data import is conceptually a **restore operation** where the source happens to be external storage rather than a managed Backup CR. The v2 BackupClass architecture with `ExecutionMode=Job` already provides:
- Job execution with custom images/commands
- RBAC permission management
- Payload secret creation and mounting
- Status observation and lifecycle management
- OpenAPI schema validation for configuration

Reusing this infrastructure eliminates duplication and provides a unified experience.

## 3. Goals & Non-Goals

**Goals:**
- Enable data import operations using the existing Restore CR and BackupClass infrastructure
- Support multiple import methods per provider (e.g., mongorestore, mongoimport, pg_restore, psql)
- Reuse BackupStorage CRs for S3 credentials and configuration
- Maintain compatibility with existing backup/restore workflows
- Eliminate the need for separate DataImporter/DataImportJob CRs

**Non-Goals:**
- Replacing existing ProviderManaged backup/restore functionality
- Supporting non-S3 storage types in the initial implementation (future: Azure, GCS)
- Automatic schema detection or data transformation during import
- Bi-directional sync or continuous data replication

## 4. Proposed Solution / Design

### 4.1 Architecture Overview

```mermaid
graph TD
    User[User] --> Restore[Create Restore CR]
    Restore --> DataSource{DataSource.Type?}
    DataSource -->|Backup| ExistingBackup[Reference Backup CR]
    DataSource -->|External| ExternalSource[Reference BackupStorage + Path]
    ExistingBackup --> BackupClass1[BackupClass<br/>restoreJob]
    ExternalSource --> BackupClass2[BackupClass<br/>importJob]
    BackupClass1 --> Job1[K8s Job: Restore]
    BackupClass2 --> Job2[K8s Job: Import]
```

### 4.2 API Changes

#### 4.2.1 Extend Restore CR DataSource

**File:** `api/backup/v1alpha1/restore_types.go`

Add new DataSourceType and supporting types:

```go
const (
    DataSourceTypeBackup   DataSourceType = "Backup"
    DataSourceTypeExternal DataSourceType = "External"  // NEW
)

// DataSourceExternal references an external storage location for import
type DataSourceExternal struct {
    // StorageName references a BackupStorage CR in the same namespace
    // that contains S3 credentials and endpoint configuration
    StorageName string `json:"storageName"`
    
    // Path is the absolute file or directory path within the storage bucket
    Path string `json:"path"`
    
    // BackupClassName references the BackupClass that defines the import method
    BackupClassName string `json:"backupClassName"`
}

type DataSource struct {
    Type     DataSourceType      `json:"type"`
    Backup   *DataSourceBackup   `json:"backup,omitempty"`
    External *DataSourceExternal `json:"external,omitempty"`  // NEW
}

type RestoreSpec struct {
    InstanceName    string                `json:"instanceName"`
    DataSource      DataSource            `json:"dataSource"`
    Config          *runtime.RawExtension `json:"config,omitempty"`
}
```

#### 4.2.2 Extend BackupClass for Import Operations

**File:** `api/backup/v1alpha1/backupclass_types.go`

Add import-specific fields:

```go
type BackupClassSpec struct {
    DisplayName         string                         `json:"displayName,omitempty"`
    Description         string                         `json:"description,omitempty"`
    SupportedProviders  ProviderNameList               `json:"supportedProviders,omitempty"`
    ExecutionMode       BackupExecutionMode            `json:"executionMode"`
    ProviderManaged     *ProviderManagedSpec           `json:"providerManaged,omitempty"`
    Config              BackupClassConfig              `json:"config,omitempty"`
    RestoreConfig       BackupClassConfig              `json:"restoreConfig,omitempty"`
    ImportConfig        BackupClassConfig              `json:"importConfig,omitempty"`  // NEW
    InstanceConstraints BackupClassInstanceConstraints `json:"instanceConstraints,omitempty"`
    UISchema            *runtime.RawExtension          `json:"uiSchema,omitempty"`
    Job                 *JobExecution                  `json:"job,omitempty"`
    RestoreJob          *JobExecution                  `json:"restoreJob,omitempty"`
    ImportJob           *JobExecution                  `json:"importJob,omitempty"`    // NEW
}
```

### 4.3 Example: Multiple Import Methods

Each import method gets its own BackupClass:

#### BackupClass 1: mongorestore (BSON dumps)

```yaml
apiVersion: backup.openeverest.io/v1alpha1
kind: BackupClass
metadata:
  name: psmdb-mongorestore-import
spec:
  displayName: "MongoDB Restore (mongorestore)"
  description: "Import BSON dumps created by mongodump"
  supportedProviders: [percona-server-mongodb]
  executionMode: Job
  
  importJob:
    jobSpec:
      image: percona/percona-server-mongodb:7.0
      command: ["/usr/bin/mongorestore"]
    permissions:
      - apiGroups: [""]
        resources: [secrets]
        verbs: [get]
  
  importConfig:
    openAPIV3Schema:
      type: object
      properties:
        database:
          type: string
          description: "Target database name"
        drop:
          type: boolean
          description: "Drop collections before import"
        numParallelCollections:
          type: integer
          description: "Number of parallel restore threads"
```

#### BackupClass 2: mongoimport (JSON/CSV files)

```yaml
apiVersion: backup.openeverest.io/v1alpha1
kind: BackupClass
metadata:
  name: psmdb-mongoimport-import
spec:
  displayName: "MongoDB Import (mongoimport)"
  description: "Import JSON, CSV, or TSV files"
  supportedProviders: [percona-server-mongodb]
  executionMode: Job
  
  importJob:
    jobSpec:
      image: percona/percona-server-mongodb:7.0
      command: ["/usr/bin/mongoimport"]
    permissions:
      - apiGroups: [""]
        resources: [secrets]
        verbs: [get]
  
  importConfig:
    openAPIV3Schema:
      type: object
      properties:
        collection:
          type: string
          description: "Target collection"
        database:
          type: string
          description: "Target database"
        file:
          type: string
          description: "File name in the source path"
        type:
          type: string
          enum: [json, csv, tsv]
          description: "Input file format"
        headerline:
          type: boolean
          description: "Use first line as field names (CSV/TSV)"
        drop:
          type: boolean
          description: "Drop collection before import"
```

### 4.4 Example: End-to-End Import Workflow

#### Step 1: Create BackupStorage (S3 credentials)

```yaml
apiVersion: backup.openeverest.io/v1alpha1
kind: BackupStorage
metadata:
  name: s3-external-data
  namespace: production
spec:
  type: s3
  s3:
    bucket: my-data-imports
    region: us-east-1
    endpointURL: https://s3.amazonaws.com
    credentialsSecretName: aws-creds
```

#### Step 2: Create Restore with External DataSource

```yaml
apiVersion: backup.openeverest.io/v1alpha1
kind: Restore
metadata:
  name: import-user-data
  namespace: production
spec:
  instanceName: my-mongo-cluster
  dataSource:
    type: External
    external:
      storageName: s3-external-data
      path: /imports/users.json
      backupClassName: psmdb-mongoimport-import
  config:
    collection: users
    database: production
    type: json
    drop: true
```

#### Step 3: Controller Creates Job

The Restore controller:
1. Resolves the `psmdb-mongoimport-import` BackupClass
2. Validates `config` against `BackupClass.spec.importConfig.openAPIV3Schema`
3. Fetches S3 credentials from the `s3-external-data` BackupStorage
4. Creates a payload Secret containing:

```json
{
  "instance": {"name": "my-mongo-cluster", "namespace": "production"},
  "connection": {
    "type": "mongodb",
    "provider": "percona-server-mongodb",
    "host": "my-mongo-cluster.svc",
    "port": "27017",
    "username": "admin",
    "password": "***",
    "uri": "mongodb://admin:***@my-mongo-cluster.svc:27017"
  },
  "storage": {
    "s3": {
      "bucket": "my-data-imports",
      "region": "us-east-1",
      "endpointURL": "https://s3.amazonaws.com",
      "accessKeyID": "***",
      "secretAccessKey": "***"
    },
    "path": "/imports/users.json"
  },
  "config": {
    "collection": "users",
    "database": "production",
    "type": "json",
    "drop": true
  }
}
```

5. Creates a Kubernetes Job using `BackupClass.spec.importJob.jobSpec`
6. Observes Job status → updates Restore.status

### 4.5 Controller Logic

**File:** `internal/controller/backup/restore_controller.go`

The RestoreReconciler branches on `dataSource.type`:

| DataSource.Type | BackupClass Resolution | Job Spec | Storage Resolution |
|----------------|------------------------|----------|-------------------|
| `Backup` | Via `Backup.spec.backupClassName` | `restoreJob` | From `Backup.spec.storageName` |
| `External` | Via `DataSourceExternal.backupClassName` | `importJob` (fallback to `restoreJob`) | From `DataSourceExternal.storageName` |

Key reconciler changes:
- `resolveBackupClass()`: Support direct backupClassName reference for External type
- `ensurePayloadSecret()`: Build storage details from BackupStorage instead of Backup
- `validateConfig()`: Use `BackupClass.spec.importConfig` for External type
- Job container name: Use `"importer"` instead of `"restorer"` for External type

### 4.6 Validation Rules (Webhook)

**File:** `internal/webhook/backup/restore_webhook.go`

- If `dataSource.type == External`:
  - `dataSource.external` is required
  - `dataSource.external.backupClassName` is required
  - Referenced BackupClass must have `importJob` defined (or `restoreJob` as fallback)
  - `config` validated against `BackupClass.spec.importConfig.openAPIV3Schema`
- If `dataSource.type == Backup`:
  - `dataSource.backup` is required
  - `dataSource.external.backupClassName` must be omitted (inferred from Backup CR)
  - `config` validated against `BackupClass.spec.restoreConfig.openAPIV3Schema`

## 5. Definition of Done

- [ ] API types updated in `api/backup/v1alpha1/` (restore_types.go, backupclass_types.go, jobspec/spec.go)
- [ ] Restore controller supports `DataSourceType.External`
- [ ] Webhook validation implemented for External data sources
- [ ] UI form refactored to create Restore CRs for imports
- [ ] Integration tests for External data source imports

## 6. Alternatives Considered

### Alternative 1: Keep Separate DataImporter/DataImportJob CRs

**Decision:** Rejected. The duplication cost outweighs the semantic clarity benefit.

### Alternative 2: Extend Backup CR to Support External Sources

**Decision:** Rejected. Using Restore CR with External DataSource is more semantically correct.

### Alternative 3: Separate Import CR

**Decision:** Rejected. Extending Restore is simpler and more consistent.

## 7. Open Questions

1. **Fallback behavior**: If `BackupClass.spec.importJob` is nil, should we fall back to `restoreJob`, or fail validation?
   - **Recommendation**: Fail validation. Import and restore may have different requirements.

2. **PITR support for imports**: Should `DataSourceExternal` support PITR-style "restore to timestamp"?
   - **Recommendation**: No. PITR is backup-specific. External sources are static snapshots.

3. **Path validation**: Should we validate that `DataSourceExternal.path` exists before creating the Job?
   - **Recommendation**: No. The job container handles errors. Pre-validation adds complexity and latency.

## 8. References

- [v1 openeverest-operator DataImporter types](https://github.com/openeverest/openeverest-operator/blob/main/api/everest/v1alpha1/dataimporter_types.go)
- [v1 openeverest-operator DataImportJob types](https://github.com/openeverest/openeverest-operator/blob/main/api/everest/v1alpha1/dataimportjob_types.go)
