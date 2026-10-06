# ArcGIS Enterprise CI/CD — Problem Statement and Project Goals

## Background

ArcGIS Enterprise is the on-premises/private-cloud deployment of the Esri platform. It hosts map services, feature services, and geoprocessing services that GIS teams build and maintain over time. Most organisations run three or more environments — dev, UAT, and production — and need to promote changes through them reliably.

The current tooling ecosystem treats service publishing as an ad-hoc manual operation. This document records the concrete pain points that make that approach unsustainable and defines the goals a CI/CD tool for ArcGIS Enterprise would need to meet.

---

## Pain Points

### 1. Publishing is manual and opaque

Publishing a service through ArcGIS Pro is the only officially supported path for map and geoprocessing services. It requires:
- An interactive ArcGIS Pro session
- A licensed install of ArcGIS Pro on the machine doing the publishing
- Manual navigation through the sharing dialog

The scripted equivalent (`StageService` + `UploadServiceDefinition` via ArcPy) inherits the same ArcGIS Pro dependency and adds its own failure modes. The most common is **Error 999999** — a catch-all that surfaces for completely unrelated root causes: IPv6/IPv4 conflicts, Windows oplock contention on file shares, token expiry mid-job, disk space during staging, certificate validation failures. The error message gives no indication which root cause applies. There is no structured logging and no way to distinguish a transient failure from a permanent one without manual investigation.

Publishing jobs can also silently queue and never complete, with no API to poll job status or kill a hung job.

### 2. Failed publishes leave services in a broken state

When a publish or overwrite fails mid-operation, ArcGIS Server does not roll back. The existing service is often left in a degraded state — stopped, partially replaced, or with corrupt configuration — while the portal item still exists pointing at it. The service does not repair itself.

Recovery requires:
1. Manually identifying which portal items and server objects are in a broken state
2. Deleting them individually through Server Manager or REST endpoints
3. Re-publishing from scratch (overwrite is no longer available once the service is broken)

This is a full manual cleanup operation with no scripted equivalent documented by Esri. There is no "restore to last known good" path.

### 3. Overwrite is destructive and not safe to automate

The overwrite workflow — the mechanism for updating a deployed service — has several sharp edges that make it dangerous in automated pipelines:

- **Layer ID reassignment**: overwriting with a new feature class can reassign layer IDs, silently breaking any layer views, web maps, or applications built on those IDs. There is no pre-flight check.
- **Schema changes break all downstream consumers**: overwriting with a mismatched schema corrupts dependent web maps immediately. The overwrite commits before any validation against downstream consumers.
- **Post-publish configuration is silently wiped**: pop-up customisations, field aliases, symbology changes, and default values applied in the Portal UI after the initial publish are lost on every overwrite.
- **Edits block overwrite**: if the feature layer has had edits since publication (sync enabled, or replicas registered), the overwrite is blocked by a prompt that cannot be bypassed programmatically without first disabling sync or unregistering replicas.

The underlying mechanism for scripted overwrites (`esriServiceDefinitionType_Replacement` in the `.sddraft` XML) is not officially documented as the primary path. Community members have reverse-engineered it from HTTP traffic.

### 4. There is no publish → test → rollback workflow

Once a service is published or overwritten, there is no concept of a deployment being "in progress" or "pending validation." The change is live immediately. If it breaks consumers:
- There is no rollback operation
- The only recovery is republishing from a previously saved `.sd` file (if one was kept) or restoring an Enterprise backup (which affects all users and all services)
- The "Rollback On Failure" property in the REST API is a database transaction rollback for feature edits, not a deployment rollback — the naming causes confusion

There is no equivalent of a blue/green deployment, a canary release, or a staged promotion with a validation gate.

### 5. Promoting across environments is not a first-class operation

The standard model for multi-environment management is three completely independent ArcGIS Enterprise sites with no automated link between them. Moving a service from dev to UAT to production involves:

- Re-staging the service definition file (requires ArcGIS Pro)
- Manually re-registering data stores on each target site, with exactly-matching client paths (the "swizzle" mechanism) — getting this wrong results in data being copied into the service rather than referenced from the registered database, with no upfront warning
- Re-publishing to each environment
- Manually updating any connection strings, URLs, or environment-specific parameters

The Python API's `clone_items()` function provides partial automation for hosted layers only. It does not handle server-registered (enterprise geodatabase-backed) feature services, which are the most common service type in organisations using a managed database backend. When cloning fails on any item, all previously cloned items are rolled back and deleted — there is no partial success or retry.

Web maps cloned between environments continue pointing to source-environment service URLs; URL rewriting is not automatic.

### 6. ArcGIS Pro is a required build dependency — no headless path exists

There is no officially supported way to create a service definition (`.sd`) file without ArcGIS Pro. The `CreateSharingDraft` and `StageService` ArcPy tools require a licensed ArcGIS Pro install on the machine running the script.

This makes it impossible to publish map or geoprocessing services from:
- A Linux CI server
- A Docker container
- Any machine without an ArcGIS Pro license

Once an `.sd` file exists, it can be uploaded and published via REST without ArcGIS Pro — but the SD generation step is the blocker for any container-based or cloud-native CI system.

### 7. No version history or audit trail for deployed services

There is no native version control for what is deployed. There is no way to answer:
- What changed between the current deployed service and the previous version?
- Who published the service, and when?
- What was the service configuration three weeks ago?

The Esri documentation and community have acknowledged this gap explicitly but there is no delivery date on a native solution. Individual teams work around it by keeping `.sd` files in file shares or SharePoint, with no standard naming scheme or diff capability.

### 8. Version compatibility between ArcGIS Pro and Enterprise is a silent failure source

The ArcGIS Pro ↔ ArcGIS Enterprise version compatibility matrix is strict — publishing from a newer Pro to an older Enterprise version produces cryptic failures or, worse, silently broken output (corrupt image services, annotation datasets that fail to open). The compatibility rules are documented in a blog post, not enforced by the tooling with a clear upfront error.

Esri's own Architecture Center documentation acknowledges that ArcGIS Enterprise is "somewhat misaligned to many organisations' existing use of CI/CD or DevOps," and describes the core problem: "ArcGIS, once deployed, begins to create state and configurations in ArcGIS Enterprise... Re-deploying ArcGIS Enterprise or a new ArcGIS Online organisation destroys that existing state unless a backup is restored."

---

## What Exists Today

| Tool | What it does | Limitation |
|---|---|---|
| ArcGIS Pro sharing dialog | Publish/overwrite services | Manual, requires interactive session, opaque failures |
| ArcPy `StageService` + `UploadServiceDefinition` | Scriptable publish | Requires ArcGIS Pro; Error 999999; no safety checks |
| ArcGIS Python API `clone_items()` | Clone portal content between orgs | Hosted layers only; all-or-nothing; URLs not rewritten |
| `PublishingTools` REST endpoint | Upload and publish an SD file | Still requires ArcGIS Pro to create the SD |
| Esri/arcgis-gitops | GitHub Actions for Enterprise infrastructure | Infrastructure standup only; no service content management |
| AGO Assistant / GEO Jobe Admin Tools | GUI content migration | Not CI/CD compatible |

No publicly maintained tool exists that covers: scripted multi-environment promotion, pre-flight validation before overwrite, deployment rollback, or headless SD generation.

---

## Project Goals

A CI/CD tool for ArcGIS Enterprise service deployment should address the following:

### G1 — Reliable scripted publishing
Publish and overwrite services from a script without requiring an interactive ArcGIS Pro session. Failures must surface a structured, actionable error — not Error 999999.

### G2 — Pre-flight validation before overwrite
Before an overwrite commits, validate that:
- The schema of the new service matches the deployed schema (or surface a diff for review)
- Layer IDs are stable (or warn that downstream consumers will break)
- Post-publish configuration that will be wiped is documented

### G3 — Safe overwrite with rollback
Deploy to a staging slot or equivalent before making the service live. If the deployment is healthy, promote it. If validation fails, roll back to the previous version without manual cleanup.

### G4 — Multi-environment promotion as a first-class operation
Define a service once and promote it through environments with environment-specific bindings (connection strings, data store paths) injected at deploy time — analogous to Helm values. No re-staging required between environments.

### G5 — Service definition versioning
`.sd` files (or their source equivalent) are versioned artifacts. Every deployed service has a traceable lineage back to a source commit. Rollback is "redeploy artifact N-1."

### G6 — CI/CD integration without an ArcGIS Pro license on the build agent
Where possible, use the ArcGIS REST API directly. Where ArcPy is unavoidable (SD generation), document the minimum viable CI environment and provide a containerised option.

### G7 — Deployment status feedback
After a publish operation, confirm the service is healthy: responding to requests, returning expected layer count, not in a stopped/failed state. Surface this as a pass/fail signal a CI pipeline can gate on.

### G8 — Compatibility pre-check
Before publishing, check the Pro version used to stage the SD against the Enterprise version of the target site and warn on known incompatibilities.

---

## Relationship to aprx-tools

[aprx-tools](https://github.com/riley-lum/aprx-tools) solves the version control problem for `.aprx` project files — making the *source* of a service diffable and mergeable in git. This project picks up where that ends: taking a versioned `.aprx` (with environment-specific connection strings applied via aprx-tools' planned substitution feature) and automating the publish, validate, and promote steps.

The natural pipeline:

```
git commit (aprx-tools)        →  .aprx.src/ tracked in git
PR merged to environment branch →  aprx-tools applies connections.json, packs .aprx
                                →  arcgis-cicd stages SD, validates, publishes to target
                                →  health check confirms service is live
                                →  rollback available if check fails
```
