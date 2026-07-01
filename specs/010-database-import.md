# Database Import

*   **Status:** Draft
*   **Authors:** @chilagrow
*   **Created:** 2026-06-26
*   **Last Updated:** 2026-06-30
*   **Related Issues:** https://github.com/openeverest/openeverest/issues/2471

---

## 1. Summary

Enable data import functionality in OpenEverest v2 by extending the Instance CR and BackupClass CR to support initial data population from external sources. Instead of creating separate DataImporter/DataImportJob CRs (as in v1), treat data imports as instance initialization operations — allowing users to create a new database Instance with pre-populated data by specifying an external data source during Instance creation.

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

A data import is conceptually an **instance initialization operation** where a new instance is created and populated with data from an external source. The v2 BackupClass architecture with `ExecutionMode=Job` already provides:
- Job execution with custom images/commands
- RBAC permission management
- Payload secret creation and mounting
- Status observation and lifecycle management
- OpenAPI schema validation for configuration

By extending the Instance CR and BackupClass CR to support initial data sources, we can reuse this infrastructure while maintaining the correct semantic: creating a new instance with initial data.

## 3. Goals & Non-Goals

**Goals:**
- Enable data import operations during Instance creation
- Support multiple import methods per provider (e.g., mongorestore, mongoimport, pg_restore, psql)
- Reuse BackupStorage CRs for S3 credentials and endpoint configuration
- Reuse BackupClass CR infrastructure for import job execution
- Eliminate the need for separate DataImporter/DataImportJob CRs

**Non-Goals:**
- Supporting non-S3 storage types in the initial implementation (future: Azure, GCS)
- Automatic schema detection or data transformation during import
- Bi-directional sync or continuous data replication

## 4. Proposed Solution / Design

### 4.1 Architecture Overview

```mermaid
graph TD
    User[User] --> Instance[Create Instance CR]
    Instance --> Check{dataSource.type = External?}
    Check -->|No| CreateDB[Create DB Resources]
    Check -->|Yes| CreateDBImport[Create DB Resources]
    CreateDB --> EmptyReady[Instance Ready]
    CreateDBImport --> WaitHealthy[Wait for Instance Healthy]
    WaitHealthy --> ResolveBC[Resolve BackupClass and Validate Config]
    ResolveBC --> FetchS3[Fetch S3 Creds from BackupStorage]
    FetchS3 --> FetchDB[Fetch DB Creds from connectionSecretRef]
    FetchDB --> PayloadSecret[Create Payload Secret]
    PayloadSecret --> CreateJob[Create Import Job - phase = Restoring]
    CreateJob --> JobDone{Job Status?}
    JobDone -->|Succeeded| DataReady[Instance Ready with Data]
    JobDone -->|Failed| ImportFailed[Import Failed]
```

### 4.2 API Changes

#### 4.2.1 Extend DataSource

**File:** `api/core/v1alpha1/instance_types.go`

For the instance initialization use case (creating an instance with pre-populated data), the Instance CR uses `dataSource` field where `*backupv1alpha1.DataSource` is defiend in `restore_types.go`:

```go
type InstanceSpec struct {
    // DataSource specifies a data source to seed the Instance from on creation.
    // If set, the instance will be created and then a Restore operation will
    // be automatically triggered to populate data before marking the instance as Ready.
    // Immutable once set.
    // +optional
    DataSource *backupv1alpha1.DataSource `json:"dataSource,omitempty"`

    // ... other fields ...
}
```

**File:** `api/backup/v1alpha1/restore_types.go`

The existing `RestoreSpec.DataSource` currently only supports `Backup` type via `DataSourceBackup`. Extend `DataSource` to support `External` type:

```go
// DataSourceType identifies the kind of data source for restore.
// +kubebuilder:validation:Enum=Backup;External
type DataSourceType string

const (
    DataSourceTypeBackup   DataSourceType = "Backup"
    DataSourceTypeExternal DataSourceType = "External"  // NEW
)

// DataSourceExternal references an external storage location for import.
type DataSourceExternal struct {
    // BackupClassName references the BackupClass that defines the import method
    BackupClassName string `json:"backupClassName"`

    // StorageName references a BackupStorage CR in the same namespace
    // that contains S3 credentials and endpoint configuration.
    StorageName string `json:"storageName"`

    // Config contains all import configuration including:
    // - path: S3 file/directory path
    // - credentialsSecretName: Secret with database credentials
    // Validated against BackupClass.spec.importConfig.openAPIV3Schema
    // +kubebuilder:validation:Required
    Config *runtime.RawExtension `json:"config"`
}

// DataSource defines the source for a Restore operation.
type DataSource struct {
    Type     DataSourceType        `json:"type"`
    Backup   *DataSourceBackup     `json:"backup,omitempty"`
    External *DataSourceExternal   `json:"external,omitempty"`  // NEW
}
```

#### 4.2.2 Extend BackupClass for Import Operations

**File:** `api/backup/v1alpha1/backupclass_types.go`

Add import-specific fields (this part remains the same):

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
    ImportSecret        BackupClassConfig              `json:"importSecret,omitempty"` // NEW
    InstanceConstraints BackupClassInstanceConstraints `json:"instanceConstraints,omitempty"`
    UISchema            *runtime.RawExtension          `json:"uiSchema,omitempty"`
    Job                 *JobExecution                  `json:"job,omitempty"`
    RestoreJob          *JobExecution                  `json:"restoreJob,omitempty"`
    ImportJob           *JobExecution                  `json:"importJob,omitempty"`    // NEW
}
```

`importJob` must be unset when `executionMode` is `ProviderManaged`, consistent with the existing constraint on `job` and `restoreJob`.

**How to Define Import Secret Schema:**

Similar to `config` and `restoreConfig`, the `importSecret` is defined in the BackupClass YAML as a reference to a Go type in the provider's `backupclasses/<name>/types.go`:

```yaml
# backupclasses/psmdb-import/class.yaml
importSecret: PSMDBImportCredentials
```

```go
// backupclasses/psmdb-import/types.go
package psmdb_import

// PSMDBImportCredentials defines the required Secret keys for PSMDB import operations.
// The provider-sdk generate command extracts this as an OpenAPI schema.
type PSMDBImportCredentials struct {
    // MONGODB_BACKUP_USER is the MongoDB backup user
    // +kubebuilder:validation:Required
    MONGODB_BACKUP_USER string `json:"MONGODB_BACKUP_USER"`

    // MONGODB_BACKUP_PASSWORD is the MongoDB backup user password
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Format=password
    MONGODB_BACKUP_PASSWORD string `json:"MONGODB_BACKUP_PASSWORD"`

    // MONGODB_CLUSTER_ADMIN_USER is the MongoDB cluster admin user
    // +kubebuilder:validation:Required
    MONGODB_CLUSTER_ADMIN_USER string `json:"MONGODB_CLUSTER_ADMIN_USER"`

    // MONGODB_CLUSTER_ADMIN_PASSWORD is the MongoDB cluster admin password
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Format=password
    MONGODB_CLUSTER_ADMIN_PASSWORD string `json:"MONGODB_CLUSTER_ADMIN_PASSWORD"`

    // MONGODB_CLUSTER_MONITOR_USER is the MongoDB cluster monitor user
    // +kubebuilder:validation:Required
    MONGODB_CLUSTER_MONITOR_USER string `json:"MONGODB_CLUSTER_MONITOR_USER"`

    // MONGODB_CLUSTER_MONITOR_PASSWORD is the MongoDB cluster monitor password
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Format=password
    MONGODB_CLUSTER_MONITOR_PASSWORD string `json:"MONGODB_CLUSTER_MONITOR_PASSWORD"`

    // MONGODB_DATABASE_ADMIN_USER is the MongoDB database admin user
    // +kubebuilder:validation:Required
    MONGODB_DATABASE_ADMIN_USER string `json:"MONGODB_DATABASE_ADMIN_USER"`

    // MONGODB_DATABASE_ADMIN_PASSWORD is the MongoDB database admin password
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Format=password
    MONGODB_DATABASE_ADMIN_PASSWORD string `json:"MONGODB_DATABASE_ADMIN_PASSWORD"`

    // MONGODB_USER_ADMIN_PASSWORD is the MongoDB user admin password
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Format=password
    MONGODB_USER_ADMIN_PASSWORD string `json:"MONGODB_USER_ADMIN_PASSWORD"`
}
```

The `provider-sdk generate` command converts this Go type to an OpenAPI v3 schema that gets embedded in the generated BackupClass manifest.

### 4.3 Example: Import Methods

Each import method gets its own BackupClass.

The default `importer` binary will be shipped inside the `provider-percona-server-mongodb` provider image,
as it was in v1.
It reads the mounted `request.json` payload, creates a `PerconaServerMongoDBRestore` CR against the Kubernetes API, and waits for the restore to reach a terminal state before exiting.
See https://github.com/openeverest/openeverest-operator/blob/main/internal/data-importer/cmd/psmdb/import.go.

#### BackupClass:

```yaml
apiVersion: backup.openeverest.io/v1alpha1
kind: BackupClass
metadata:
  name: psmdb-mongorestore-import
spec:
  displayName: "MongoDB Import"
  description: "Import BSON dumps created by mongodump"
  supportedProviders: [percona-server-mongodb]
  executionMode: Job

  importConfig:
    openAPIV3Schema:
      type: object
      required:
        - path
        - credentialsSecretName
      properties:
        path:
          type: string
          description: "S3 path to import file/directory. For mongorestore, point to a directory containing BSON dump files. For mongoimport, point to a single JSON/CSV/TSV file."
        credentialsSecretName:
          type: string
          description: "Name of Secret containing database credentials. Required keys defined by BackupClass.spec.importSecret."

  importSecret:
    openAPIV3Schema:
      type: object
      required:
        - MONGODB_BACKUP_USER
        - MONGODB_BACKUP_PASSWORD
        - MONGODB_CLUSTER_ADMIN_USER
        - MONGODB_CLUSTER_ADMIN_PASSWORD
        - MONGODB_CLUSTER_MONITOR_USER
        - MONGODB_CLUSTER_MONITOR_PASSWORD
        - MONGODB_DATABASE_ADMIN_USER
        - MONGODB_DATABASE_ADMIN_PASSWORD
        - MONGODB_USER_ADMIN_PASSWORD
      properties:
        MONGODB_BACKUP_USER:
          type: string
          description: "MongoDB backup user"
        MONGODB_BACKUP_PASSWORD:
          type: string
          description: "MongoDB backup user password"
          format: password
        MONGODB_CLUSTER_ADMIN_USER:
          type: string
          description: "MongoDB cluster admin user"
        MONGODB_CLUSTER_ADMIN_PASSWORD:
          type: string
          description: "MongoDB cluster admin password"
          format: password
        MONGODB_CLUSTER_MONITOR_USER:
          type: string
          description: "MongoDB cluster monitor user"
        MONGODB_CLUSTER_MONITOR_PASSWORD:
          type: string
          description: "MongoDB cluster monitor password"
          format: password
        MONGODB_DATABASE_ADMIN_USER:
          type: string
          description: "MongoDB database admin user"
        MONGODB_DATABASE_ADMIN_PASSWORD:
          type: string
          description: "MongoDB database admin password"
          format: password
        MONGODB_USER_ADMIN_PASSWORD:
          type: string
          description: "MongoDB user admin password"
          format: password

  importJob:
    jobSpec:
      image: percona/provider-percona-server-mongodb:0.1.0
      command: ["/importer", "psmdb"]
    permissions:
      - apiGroups: [""]
        resources: [secrets]
        verbs: [get, create, update, delete]
      - apiGroups: ["psmdb.percona.com"]
        resources: [perconaservermongodbrestores]
        verbs: [get, create, update]
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
    credentialsSecretName: my-s3-creds  # user-created Secret with AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY
```

#### Step 2: Create Database Credentials Secret

The import job requires database credentials to connect to the Instance. Users must create a Secret containing the necessary credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-mongo-cluster-import-creds
  namespace: production
type: Opaque
stringData:
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

> **Note:** The required credential keys are defined by the BackupClass's `spec.importSecret`. For the PSMDB import BackupClass, all MongoDB user credentials listed in the schema must be provided.

#### Step 3: Create Instance with DataSource

```yaml
apiVersion: instance.openeverest.io/v1alpha1
kind: Instance
metadata:
  name: my-mongo-cluster
  namespace: production
spec:
  provider: percona-server-mongodb
  topology: replica-set
  resources:
    cpu: "2"
    memory: 4Gi
  storage:
    size: 50Gi
    class: standard

  dataSource:
    type: External
    external:
      backupClassName: psmdb-mongoimport-import
      storageName: s3-external-data
      config:
        path: /imports/users.json
        credentialsSecretName: my-mongo-cluster-import-creds
```

#### Step 4: Controller Creates Instance and Import Job

The Instance controller:
1. Creates the database instance as normal (StatefulSet, Services, etc.)
2. Waits for the instance to become healthy
3. Once healthy, resolves the `psmdb-mongoimport-import` BackupClass from `dataSource.external.backupClassName`
4. Validates `dataSource.external.config` against `BackupClass.spec.importConfig.openAPIV3Schema`
5. Extracts `path` and `credentialsSecretName` from `dataSource.external.config`
6. Validates that the Secret named by `config.credentialsSecretName` exists and contains all required keys defined in `BackupClass.spec.importSecret.openAPIV3Schema`
7. Fetches S3 credentials from the BackupStorage named by `dataSource.external.storageName`
8. Reads DB connection info (host, port) from `instance.status` — populated by the provider once the instance is healthy
9. Reads DB credentials from the user-provided Secret named by `config.credentialsSecretName`
10. Creates a payload Secret with key `request.json` containing the normalized import contract (matching the `dataimporterspec.Spec` shape from v1):

```json
{
  "source": {
    "s3": {
      "bucket": "my-data-imports",
      "region": "us-east-1",
      "endpointURL": "https://s3.amazonaws.com",
      "accessKeyID": "***",
      "secretKey": "***",
      "verifyTLS": true,
      "forcePathStyle": false
    },
    "path": "/imports/users.json"
  },
  "target": {
    "databaseClusterRef": {"name": "my-mongo-cluster", "namespace": "production"},
    "host": "my-mongo-cluster.svc",
    "port": "27017",
    "user": "databaseAdmin",
    "password": "***",
    "type": "mongodb"
  }
}
```

> **Note:** The `user` and `password` in the payload are extracted from the user-provided Secret referenced in `config.credentialsSecretName`. For PSMDB, `MONGODB_DATABASE_ADMIN_USER` and `MONGODB_DATABASE_ADMIN_PASSWORD` are used.

11. Creates a Kubernetes Job using `BackupClass.spec.importJob.jobSpec`, with the payload Secret mounted as a volume at `/payload/request.json`
12. Sets `status.importJobName`, `ConditionDataSourceReady=False`, reason=`Importing`, phase=`Restoring`
13. Observes Job until terminal:
    - **Succeeded**: sets `ConditionDataSourceReady=True`, reason=`Succeeded`, phase=`Ready`, clears `importJobName`
    - **Failed**: sets `ConditionDataSourceReady=False`, reason=`ImportFailed`, message=job error, phase=`Failed`

### 4.5 UI Support

The import feature integrates into the existing **create instance wizard** as an optional section.

#### Create Instance Wizard: Data Import Section

A new collapsible optional "Data Import" section is added as a step of the create instance wizard.

1. **Import Method** — a `select` dropdown populated by:
   ```
   GET /clusters/{cluster}/backup-classes
   ```
   Filtered client-side to only show BackupClasses where:
   - `spec.importJob` is set
   - `spec.supportedProviders` includes the selected instance provider

2. **Storage** — a `select` dropdown populated by:
   ```
   GET /clusters/{cluster}/namespaces/{namespace}/backup-storages
   ```

3. **Path** — a free-text input for the file/directory path within the bucket.

4. **Database Credentials** — dynamic password fields rendered from the selected BackupClass's `spec.importSecret.openAPIV3Schema`. The schema defines required Secret keys and their types/descriptions. The schema is fetched via:
   ```
   GET /clusters/{cluster}/backup-classes/{backupClass}
   ```
   For the PSMDB import BackupClass, the schema defines 9 required credential fields (MongoDB users and passwords). The UI renders input fields for each property in the schema, respecting the `format: password` hint to use password input fields.

   These credentials are used to create a Secret `{instance-name}-import-creds` before creating the Instance.

5. **Import Config** — dynamic fields rendered from the selected BackupClass's `spec.importConfig.openAPIV3Schema`, using the same JSON schema → form field renderer used for backup/restore config forms. The schema is fetched via:
   ```
   GET /clusters/{cluster}/backup-classes/{backupClass}
   ```
   Rendering hints can optionally be provided via `BackupClass.spec.uiSchema` under an `import` key, mirroring the `backup` and `restore` keys already used there.

**On submit**, the wizard:

1. Creates a Secret containing the database credentials:
   ```
   POST /clusters/{cluster}/namespaces/{namespace}/secrets
   ```
   With body:
   ```json
   {
     "metadata": {
       "name": "{instance-name}-import-creds"
     },
     "type": "Opaque",
     "stringData": {
       "MONGODB_BACKUP_USER": "...",
       "MONGODB_BACKUP_PASSWORD": "...",
       "MONGODB_CLUSTER_ADMIN_USER": "...",
       "MONGODB_CLUSTER_ADMIN_PASSWORD": "...",
       "MONGODB_CLUSTER_MONITOR_USER": "...",
       "MONGODB_CLUSTER_MONITOR_PASSWORD": "...",
       "MONGODB_DATABASE_ADMIN_USER": "...",
       "MONGODB_DATABASE_ADMIN_PASSWORD": "...",
       "MONGODB_USER_ADMIN_PASSWORD": "..."
     }
   }
   ```

2. Creates the Instance with the `dataSource` block:
   ```
   POST /clusters/{cluster}/namespaces/{namespace}/instances
   ```
   With body:
   ```json
   {
     "spec": {
       ...
       "dataSource": {
         "type": "External",
         "external": {
           "backupClassName": "psmdb-mongorestore-import",
           "storageName": "s3-external-data",
           "config": {
             "path": "/imports/dump",
             "credentialsSecretName": "{instance-name}-import-creds",
             ...
           }
         }
       }
     }
   }
   ```

#### Instance Detail Page: Import Status

While `instance.status.phase == "Restoring"` and `ConditionDataSourceReady` is present with `status=False`, the instance detail page shows an import progress banner with:
- The reason and message from `ConditionDataSourceReady`
- A link to Job logs using `status.importJobName` (same pattern as restore job log links)

Once the import completes, the condition flips to `status=True` and the banner is dismissed.

## 5. Definition of Done

- [ ] Controller supports initial data import workflow
- [ ] UI form supports creating Instances with initial data import
- [ ] Integration tests for Instance creation with initial data import

## 6. Alternatives Considered

### Alternative 1: Keep Separate DataImporter/DataImportJob CRs

**Decision:** Rejected. The duplication cost outweighs the semantic clarity benefit.

### Alternative 2: Extend Restore CR's DataSource with External Type

**Decision:** Rejected. Restore CR semantically implies restoring to an **existing** instance, but database import creates a **new** instance. Modifying `restore_types.go` to add `External` would couple the restore and import concerns. Instead, all new types are self-contained in `instance_types.go`.

### Alternative 3: Instance Controller Creates a Restore CR Internally

**Decision:** Rejected. While this avoids duplicating execution logic, it creates a Restore CR that the user never requested. This phantom CR:
- Appears in `kubectl get restores` and confuses operators
- Creates unclear ownership (can the user delete it? interact with it?)
- Splits failure diagnosis across two controllers and two status objects

Instead, execution logic is extracted into a shared `pkg/importer` package, keeping the Instance controller as the single owner of the import lifecycle with no hidden side effects.

### Alternative 4: Separate Import CR that creates an Instance

**Decision:** Rejected. Having two ways to create an instance (Instance CR vs Import CR) creates confusion. Better to have one way with optional initial data.

## 7. Open Questions

1. **Import retry mechanism**: Should there be a way to retry a failed import without recreating the entire Instance?

2. **Import progress reporting**: Should we expose Job pod logs or progress metrics in Instance status?

## 8. References

- [v1 openeverest-operator DataImporter types](https://github.com/openeverest/openeverest-operator/blob/main/api/everest/v1alpha1/dataimporter_types.go)
- [v1 openeverest-operator DataImportJob types](https://github.com/openeverest/openeverest-operator/blob/main/api/everest/v1alpha1/dataimportjob_types.go)
- [v1 default importer](https://github.com/openeverest/openeverest-operator/blob/main/internal/data-importer/cmd/psmdb/import.go)
