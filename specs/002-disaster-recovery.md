# Disaster Recovery for OpenEverest

*   **Status:** Draft
*   **Authors:** @spron-in
*   **Created:** 2026-04-09
*   **Last Updated:** 2026-04-09
*   **Related Issues:** 

---

## 1. Summary

When a Kubernetes cluster running OpenEverest is lost, users can restore their databases from S3 backups. However, critical metadata is missing: the database name in OpenEverest, its version, how it was exposed, monitored, what its backup schedule was, and other configuration parameters. This spec explores how OpenEverest can ensure this metadata is preserved and recoverable, so users can fully restore their database infrastructure after a cluster failure.

## 2. Motivation

OpenEverest manages databases on Kubernetes via CRDs (e.g., `DatabaseCluster`). When the Kubernetes cluster goes down, the data itself may be recoverable from S3 backups via `DataImporter`. However, the Kubernetes control plane — and with it all OpenEverest custom resources — is lost.

A user facing this scenario is left asking:
- Which S3 snapshot belongs to which database?
- What was the database engine version at the time of backup?
- How was the database exposed (services, ingress, etc.)?
- What monitoring was configured?
- What was the backup schedule and retention policy?
- What secrets, presets, and templates were associated with the database?

Without this information, restoring from backup is manual, error-prone, and potentially incomplete. The user must reconstruct the full configuration from memory or external documentation — an unacceptable experience for a managed database platform.

## 3. Goals & Non-Goals

**Goals:**

* Define a clear recovery story: what a user must do (or what OpenEverest does automatically) to recover their full database configuration after a cluster loss.
* Identify all the configuration state that needs to be preserved beyond raw data backups, including but not limited to:
  * `DatabaseCluster` resources
  * S3/backup storage configuration and credentials
  * Secrets (DB users, TLS certs, etc.)
  * Monitoring configuration
  * Presets and templates
* Evaluate multiple approaches for preserving this state and understand their trade-offs.

**Non-Goals:**

* Defining the actual implementation of any specific approach — this is an early-stage spec.
* Database data recovery mechanism itself (handled by existing `DataImporter` and backup tooling).
* High availability or active-active cluster setups — this spec is scoped to full cluster loss and cold recovery.
* Recovery of the Kubernetes cluster itself.

## 4. Proposed Solution / Design

> **Note:** This is an early draft. No solution has been selected. The section below outlines the problem space and candidate approaches for discussion.

### 4.1 What Needs to Be Preserved

For a full recovery, the following categories of state must be available after cluster loss:

| Category | Examples |
|---|---|
| Database configuration | `DatabaseCluster` CR, engine version, topology |
| Connectivity | Service type, exposed ports, ingress config |
| Credentials & secrets | DB user passwords, TLS certificates |
| Backup configuration | Schedule, retention policy, S3 bucket reference |
| Monitoring configuration | PMM server endpoint, credentials |
| Presets & templates | Any named presets applied to the cluster |
| S3 storage configuration | Bucket name, region, credentials/IAM role |

### 4.2 Candidate Approaches

#### Option 1: User-managed Kubernetes backup (e.g., Velero)

Users take responsibility for backing up all Kubernetes resources, including OpenEverest CRs, using a standard k8s backup tool such as Velero. After cluster recovery, they restore Kubernetes objects before or alongside restoring OpenEverest.

**Key open questions:**
- Does restoring a `DatabaseCluster` CR automatically trigger OpenEverest to reconcile and recreate the database, or does it conflict with the data restored from S3?
- Is the ordering of CR restore vs. data restore well-defined?
- Is the user experience consistent and predictable across different k8s backup tools?

#### Option 2: Per-plugin / per-provider integration

Each OpenEverest provider plugin is responsible for implementing its own DR strategy — e.g., exporting metadata alongside data backups.

**Key open questions:**
- How does the user discover which plugin supports DR and in what format?
- What happens when a user runs multiple plugins? Is the DR experience consistent?
- Does this create an unacceptable fragmentation of the recovery workflow?

#### Option 3: OpenEverest-owned state export

OpenEverest itself exports all relevant configuration state (CRs, secrets, presets, S3 config, etc.) to a dedicated external store — likely an S3 bucket — either continuously or on a schedule.

**Key open questions:**
- What triggers an export — every change, periodic schedule, on backup creation?
- Which resources need to be exported? Is the list static or dynamic (e.g., plugin-provided)?
- How are secrets handled? Stored encrypted? Excluded and documented separately?
- How does a user bootstrap a new OpenEverest instance to point at the state export bucket?
- Would this require a dedicated "OpenEverest state" bucket, separate from data backup buckets?
- How does a user reconcile partial state (e.g., some databases restored, others not yet)?

#### Option 4: IaC / GitOps as source of truth

Users who interact with OpenEverest via CRDs are required (or strongly recommended) to manage those resources through Infrastructure as Code tooling (e.g., Terraform, Pulumi, Helm) or GitOps pipelines (e.g., Flux, ArgoCD). The desired state of all OpenEverest resources lives in a Git repository or IaC state backend — not as a backup, but as the authoritative definition of what should exist.

A complementary module that talks to the OpenEverest API could also actively sync or validate the live state against this stored definition, further ensuring the IaC representation stays current.

Recovery in this model means re-applying the stored definitions against a new OpenEverest instance, rather than restoring Kubernetes objects from a snapshot.

This is distinct from Option 1: the state is not periodically snapshotted from the cluster — it is the primary source of truth that the cluster was built from in the first place.

**Key open questions:**
- Does OpenEverest need to prescribe or officially support specific IaC/GitOps tooling, or just document the pattern?
- How does the API-facing module stay in sync if users make changes directly via the UI or kubectl (i.e., drift)?
- Secrets are typically not stored in Git — how are they handled in this model (e.g., sealed secrets, external secrets operator)?
- Is this approach viable for users who do not already use IaC tooling? Should it be optional or required for DR coverage?
- How does this interact with resources OpenEverest manages internally (e.g., auto-created secrets, operator-managed CRs)?

#### Option 5: (Open)

> Are there other approaches worth exploring? E.g., integration with external secret managers, etc.

## 5. Definition of Done

> To be defined once an approach is selected.

## 6. Alternatives Considered

> To be populated as the design discussion progresses.

## 7. Open Questions

1. What is the minimum viable recovery story? Can we start with Option 1 (user-managed) as a documented workaround while we build a first-party solution?
2. Are secrets in-scope for the first iteration? They add significant complexity (encryption, rotation, vault integration).
3. Should recovery be a fully automated process, or is a guided manual process acceptable for v1?
4. How do we associate an S3 backup snapshot with the `DatabaseCluster` CR that produced it? Is there currently any metadata written alongside backups?
5. What is the expected RTO (Recovery Time Objective) that users need? This may influence the approach.
6. Do presets and templates need to be restored before databases can be recreated, or can they be applied after?
7. How should the spec handle multi-namespace OpenEverest deployments?

## 8. References

* [OpenEverest vision](https://vision.openeverest.io)
* [001 - Plugins Architecture](./001-plugins-architecture.md)
