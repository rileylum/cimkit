# Prior Art Survey — ArcGIS Enterprise CI/CD and Service Deployment

A survey of every meaningful public attempt to automate ArcGIS Enterprise service publishing, multi-environment promotion, and deployment management.

---

## Summary Table

| Tool | Publisher | Stars | Last Active | ArcGIS Pro Required | Publishes Services | Multi-Env Promotion | Rollback | Health Check |
|---|---|---|---|---|---|---|---|---|
| arcgis-gitops | Esri (official) | 30 | Jan 2026 | No | No (infra only) | No | No | Site-level only |
| ags-service-publisher | City of Austin | 16 | Mar 2026 | Yes | Yes | No | No | No |
| agsconfig | D. Whittingham | 5 | Apr 2024 | No | Config only | No | No | No |
| agsadmin | D. Whittingham | 6 | Jul 2023 | No | No | No | No | Status API wrapper |
| arcpyext | D. Whittingham | 25 | Apr 2024 | Yes (ArcPy) | Partial | No | No | No |
| solution.js | Esri (official) | 47 | Oct 2025 | No | Hosted layers only | Partial (templates) | No | No |
| ArcGIS Python API | Esri (official) | N/A | Active | No (publish step) | Hosted only (headless) | No | No | Status endpoint |
| update-hosted-feature-service | Esri/arcpy | N/A | Archived 2020 | Yes (ArcMap) | Hosted only | No | No | No |
| rooschu/overwrite-web-layers | Community | 9 | Mar 2024 | Yes | Yes | No | No | No |
| docker-arcgis-enterprise | Wildsong | 145 | Mar 2024 | No | No (infra only) | No | No | No |
| ags_map_service_deployer | pauldzy | N/A | Abandoned | ArcGIS Server | Config only | No | No | No |

---

## Detailed Notes

### 1. Esri/arcgis-gitops

GitHub Actions workflows for provisioning and upgrading ArcGIS Enterprise infrastructure — Terraform for cloud resources, Ansible for machine configuration, Packer for base images. Covers AWS and Azure. Includes a post-provision "Enterprise Admin CLI" container that runs a test publish (a small CSV to a hosted feature layer) to confirm the site is alive after deployment.

**What it does not do:** Publishes no GIS services. Does not manage portal content, service definitions, or data sources. The health check is a smoke test at the site level, not a per-service deployment gate.

**Architecture note:** Esri's own documentation alongside this repo explicitly acknowledges the gap — IaC tools handle infrastructure well but "not for maintaining state of a system" like portal content and services. This is the divide arcgis-cicd fills.

**Complementary, not competing.** arcgis-gitops provisions the environment; arcgis-cicd deploys into it.

---

### 2. cityofaustin/ags-service-publisher

The most mature public tool for automated bulk service publishing. Used by the City of Austin (a large municipal GIS operation) in production.

**Architecture:** Two-tier YAML config — `userconfig.yml` defines all server environments; per-folder YAML files list services with property cascades at folder → environment → service level. Services are identified by name; the tool matches them to maps in an `.aprx` project, stages each one, and publishes. Supports MapServer, FeatureServer, ImageServer, GeocodeServer. Handles `.mxd` → `.aprx` auto-conversion.

**Multi-env data path translation:** `data_source_mappings` — a list of substitution rules applied to connection strings before staging. Can be simple string replacement or criteria-based (match by workspace type, then replace). This is conceptually the right model.

**Python invocable:** `Runner().run_batch_publishing_job(...)` — not just a CLI, can be embedded in a pipeline.

**What it does not do:**
- No rollback — if a service publish fails mid-batch, already-published services stay up, later services are skipped. No atomicity.
- No pre-flight validation (no `AnalyzeForSD` gate before staging begins)
- No health check after publish
- No versioning of SD artifacts
- No concept of "promote this batch from env A to env B after approval"
- Requires ArcGIS Pro on the machine running the script

**Gap relative to arcgis-cicd:** Multi-env promotion as a first-class concept, rollback, validation gates, health checks, and headless operation are all absent.

---

### 3. DavidWhittingham/agsconfig + agsadmin + arcpyext

An ecosystem of complementary libraries by a single Australian developer. Collectively the most technically sophisticated community attempt, but fragmented and under-maintained.

**agsconfig:** Edits ArcGIS Server service configuration — both sddraft files (pre-publish) and the REST Admin API JSON (post-publish). No ArcGIS Pro required. The only library found that explicitly supports modifying service configuration post-publish via REST without ArcGIS Pro. Supports MapServer and ImageServer.

**agsadmin:** Python client for the ArcGIS Server REST Admin API — start, stop, delete services; get status and statistics; manage uploads. No ArcGIS Pro required. Useful as a building block for health checks and service lifecycle management.

**arcpyext:** ArcPy extension with a publishing module that calls `AnalyzeForSD()` before staging — the only library found that wraps the analysis step as a validation gate. Also handles data source remapping for maps before staging. Requires ArcPy.

**agstools (deprecated 2017):** CLI wrapper that used all the above for a draft → stage → publish command line. Python 2.7 only, abandoned. The successor to this — a modern, maintained CLI — does not exist.

**Takeaway:** These three libraries together cover (1) pre-publish analysis, (2) sddraft editing, (3) REST-based post-publish config management, and (4) server lifecycle management. None of them is a deployment pipeline; agstools shows what that pipeline could look like but is dead. Worth borrowing from (or depending on) rather than rebuilding.

---

### 4. Esri/solution.js

TypeScript/Node.js library for transferring portal items between ArcGIS Online/Enterprise organizations via templates. The most complete existing tool for portal content promotion — supports 80+ item types: web maps, apps, dashboards, hosted feature services, notebooks, forms, StoryMaps, QuickCapture.

**Template-based promotion:** Source items are serialized to templates (JSON blobs describing the item and its dependencies). Templates can be parameterized with environment-specific values at deploy time. Target-environment item IDs are tracked so re-deploys are idempotent (`search_existing_items=True`).

**What it does not do:** Cannot republish map services or image services to ArcGIS Server — non-hosted services are just URL references in templates, not republished artifacts. No rollback. No health check. No validation.

**Relevance:** If arcgis-cicd eventually handles the portal content side of a deployment (web maps, dashboards) alongside the server service side, solution.js is the most complete prior art. For now, it solves the portal item promotion problem but not the service publishing problem.

---

### 5. ArcGIS Python API (`arcgis` on PyPI)

The official Esri Python client. Not a deployment tool itself but the foundation all tools build on. Key capabilities:

**Headless SD upload and publish (no ArcGIS Pro at publish time):**
```python
gis.content.add(item_properties, data='/path/to/service.sd')
published_item.publish()
```
Caveat: generating the `.sd` file still requires ArcGIS Pro or ArcGIS Server + ArcPy. The publish step can be headless; the stage step cannot.

**Headless hosted feature layer overwrite:**
```python
FeatureLayerCollection.fromitem(item).manager.overwrite(data_file_path)
```
Hosted feature layers only. Preserves itemID. Cannot change schema. Not usable for server-based map services.

**Service status:**
```python
server.services['myservice/MapServer'].status
```

**Idempotent cloning:**
```python
target_gis.content.clone_items(items, search_existing_items=True)
```

**Gaps:** No `AnalyzeForSD` wrapper. No rollback. No pre-flight validation. No promotion pipeline. `clone_items()` does not work for non-hosted (enterprise geodatabase-backed) services.

---

### 6. Wildsong/docker-arcgis-enterprise

Docker Compose setup for a complete ArcGIS Enterprise stack (Server + Portal + Data Store + PostgreSQL) on Linux. 145 stars — the most popular tool in this survey, but for a different problem.

**Relevance:** Provides a disposable test environment for CI pipelines. The fastest way to stand up an Enterprise instance for integration testing without a permanent installation. Worth knowing for building arcgis-cicd's own test infrastructure.

---

### 7. pauldzy/ags_map_service_deployer

Addresses one specific gap that broader tools miss: **preserving dynamic service settings that ArcGIS resets on republish** — instance counts, timeouts, capabilities configured in Server Manager after initial publish. The script reads the current deployed configuration before overwriting and re-applies the settings after. This problem is real and the instinct is right; the implementation is legacy ArcGIS Server (not ArcGIS Pro) and abandoned.

**Takeaway:** Any overwrite workflow needs a settings-preservation strategy. Options: (1) read current settings before overwrite, re-apply after; (2) maintain all settings in version-controlled config and apply at every deploy.

---

### 8. Community Patterns from Forum Research

From Esri Community threads and blog posts, the real-world landscape:

- Most organisations use **cron / Windows Task Scheduler + Python + ArcPy** for nightly service republish. No promotion concept — they just republish to each environment separately.
- **Azure DevOps Pipelines** is being explored, but the ArcGIS Pro dependency on build agents is a blocker — ArcGIS Pro can't run on Linux agents.
- **WebGISDR** (built-in Enterprise backup/restore) is used as a blunt promotion mechanism: restore a full dev backup to UAT. Promotes everything — users, groups, content — not surgical service promotion.
- **Jenkins** mentioned in one thread, no detail.
- Blue-green deployment for Enterprise services is an open question with no public answer.

---

## What Genuinely Does Not Exist

After exhaustive search, the following capabilities have no public implementation:

| Gap | Why it matters |
|---|---|
| Headless SD file creation (no ArcGIS Pro on CI) | Hard blocker for container-based CI — every tool requires licensed ArcGIS Pro to create the `.sd` artifact |
| Pre-flight validation as a pipeline gate | `AnalyzeForSD()` exists in ArcPy; no tool wraps it as a CI gate with exit codes and reporting |
| Settings preservation on overwrite | Identified by ags_map_service_deployer; never generalized; admins lose post-publish config silently |
| Safe overwrite with rollback | FishersGIS says "take a snapshot first" — manual advice, never automated |
| Multi-environment promotion pipeline | All tools publish to one environment; none model approve-then-promote across envs |
| Post-publish health check as pass/fail gate | REST status endpoint exists; no tool gates deployment on it with retry, timeout, rollback-on-fail |
| Service definition artifact versioning | No tool versions `.sd` files as first-class artifacts in a registry |
| Blue-green / zero-downtime service update | Completely unsolved — community asks the question, Esri has no answer |

---

## Patterns Worth Borrowing

| Pattern | Source |
|---|---|
| YAML config with folder → environment → service cascade | ags-service-publisher |
| Connection-neutral SD files as the deployment artifact, env values injected at publish time | Esri official model |
| `search_existing_items=True` idempotent re-deploy | ArcGIS Python API / solution.js |
| itemID preservation on overwrite | arcpy/update-hosted-feature-service |
| `AnalyzeForSD()` before staging as a validation gate | arcpyext |
| sddraft XML edits (`esriServiceDefinitionType_Replacement`) as the overwrite mechanism | Community convention — not officially documented |
| Read current settings → overwrite → re-apply settings pattern | ags_map_service_deployer |
| Server + Portal health check in sequence | Esri REST API design |
| Docker-based disposable Enterprise environment for integration tests | Wildsong/docker-arcgis-enterprise |
