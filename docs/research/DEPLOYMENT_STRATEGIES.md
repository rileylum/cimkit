# Deployment and Rollback Strategies

## The core constraint: URL hardcoding

Before evaluating any approach, one ArcGIS-specific constraint shapes everything:

**Portal items for server-based services store the service REST URL as a string.** Web maps, dashboards, and applications reference services by their full URL (`/rest/services/Map/FeatureServer`). Any deployment pattern that permanently changes the service URL requires a scripted update pass across every Portal item that referenced the old URL.

This rules out patterns where a URL change is expected to be transparent to consumers — unless the rename-swap approach described in Approach 1 is used, which exploits the URL-as-pointer property rather than fighting it.

---

## Approach 1 — Clone, test, overwrite (or rename-swap)

**Publish a shadow copy of the service, validate it, then replace the real service.**

### The shadow's purpose

The shadow service (`Map_shadow`) is a pre-flight validation target only — not a staging slot that consumers ever touch. Its sole purpose is to confirm that the new version publishes successfully and passes tests before the real service is touched. It is always cleaned up after the deploy, whether the deploy succeeds or fails.

### Path A — SD overwrite

1. Publish the new version as `Map_shadow`
2. Run tests — health check, query validation, schema checks
3. If tests pass:
   - GET the current service config JSON from REST Admin API → save as rollback artifact
   - GET the current service config JSON → save to git
   - Overwrite `Map` using the SD with `esriServiceDefinitionType_Replacement`
   - Run a post-deploy health check
4. Delete `Map_shadow` and its Portal item

This is the simpler path. The overwrite is destructive and in-place — brief downtime during the operation, no retained old version on the server.

### Path B — Rename-swap

The `renameService` REST Admin API endpoint renames a service within its folder. Portal items reference server-based services by their REST URL path, not by any internal service ID. This property makes a near-zero-downtime swap possible:

1. Publish `Map_shadow`, test it
2. `renameService`: `Map` → `Map_old`  ← downtime window opens
3. `renameService`: `Map_shadow` → `Map`  ← downtime window closes
4. The original Portal item stored `url: …/rest/services/Map/FeatureServer`. That URL now resolves to the newly renamed service — the Portal item is valid and unchanged without any update
5. The shadow's Portal item stored `url: …/rest/services/Map_shadow/FeatureServer` — that URL no longer exists — delete this Portal item
6. `Map_old` server service is retained on the server for the rollback period

**Rollback:** `renameService` `Map` → `Map_broken`, `Map_old` → `Map`. The original Portal item's URL resolves to the old version again. Seconds of downtime.

**Limitation:** `renameService` only works within the same folder. Cross-folder moves are not supported via the API.

### Rename-swap: Portal item behaviour by service type

The rename-swap pattern relies on Portal items referencing the service by URL path. Whether this holds depends on service type:

| Service type | URL path reference? | Rename-swap viable? |
|---|---|---|
| Server-based MapServer | Yes | Expected yes |
| Server-based FeatureServer (registered enterprise GDB) | Yes | Expected yes |
| Hosted FeatureServer (Portal-managed data store) | Portal has internal linkage beyond the URL | Uncertain — needs testing |
| ImageServer (registered raster) | Yes | Expected yes |
| GPServer | Yes | Expected yes |
| Hosted tiled layer / scene layer | Portal-managed | Uncertain |

For hosted feature layers, Portal maintains internal state about the service beyond just the URL — the hosting server registration, data store linkage, and Portal item ownership model may not update correctly when the underlying server service is renamed via the Admin API. This is a critical open question before committing to the rename-swap path for hosted layers.

**For non-hosted server services, rename-swap is the preferred deployment path.** For hosted layers, SD overwrite (Path A) is the safer default until rename-swap behaviour is verified.

### Saving service state: REST vs git

Two complementary artifacts, not alternatives:

| Artifact | What it captures | When to save | Use for |
|---|---|---|---|
| SD file in git | Complete service source — CIM, schema, symbology, data reference | Every publish | Re-publishing from scratch; source of truth for what was intended |
| Service config JSON from REST Admin API | Runtime state — instance counts, timeouts, capabilities, post-publish tweaks | Immediately before each deploy | Restoring runtime properties without a full re-publish |

The SD captures what was deployed. The config JSON captures what the service had become in production (which may differ if an admin tuned it post-publish). Both are needed for a complete rollback story.

### Rollback options by severity

| Scenario | Rollback action | Downtime |
|---|---|---|
| Post-publish config drift only | `editService` with saved config JSON | Seconds (service restart) |
| Broken after rename-swap | `renameService` Map → Map_broken, Map_old → Map | Seconds |
| Broken after SD overwrite | Re-publish from saved .sd file | Minutes |
| Complete corruption, .sd lost | Re-publish from git source (aprx → headless SD → publish) | Minutes + build time |

---

## Approach 2 — Proxy / traffic routing (blue-green)

**Sit a proxy between consumers and ArcGIS, publish a new service version, route traffic to it, keep the old version for rollback.**

### What's actually possible

The ArcGIS Web Adaptor does not support URL routing rules — it is a pass-through component only.

A third-party reverse proxy (nginx, Caddy, HAProxy) placed in front of the Web Adaptor could do path rewriting at the HTTP level. But:
- ArcGIS token authentication is tied to the specific resource URL — rewriting the URL on the wire breaks token validation
- CORS headers from ArcGIS Server reference the service by its internal name
- JSON response bodies from the service contain self-referencing URLs that would also need rewriting

True transparent traffic routing would require the proxy to rewrite both request paths and JSON response bodies — fragile, untested with ArcGIS auth, and explicitly warned against by Esri.

### Where it could work

For **direct REST API consumers** (custom applications that query services programmatically, not Portal web maps), URL-level routing is viable if those consumers are built to use the proxy URL rather than the ArcGIS Server URL directly.

**Infrastructure-level blue-green** (two complete Enterprise deployments with DNS cutover) is Esri's documented recommendation for zero-downtime whole-environment upgrades — not individual service deployments.

### Verdict

**Not viable for Portal-connected services at the individual service level.** The rename-swap pattern in Approach 1 achieves much of the same goal (retain old version, brief swap window, fast rollback) without the proxy complexity and without breaking token auth.

---

## Approach 3 — Direct filesystem modification

**Edit ArcGIS Server config-store files directly rather than using the REST API.**

### What the filesystem looks like

ArcGIS Server stores service configuration as JSON files in the config-store:
```
<config-store>/services/<ServiceName.ServiceType>/<ServiceName.ServiceType>.json
```

This is the same JSON structure returned by the Admin REST API.

### Hot-reload: per-service restart, not full server restart

The REST Admin API supports stopping and starting individual services without restarting the whole server:

```
POST /arcgis/admin/services/Map.FeatureServer/stop
POST /arcgis/admin/services/Map.FeatureServer/start
```

The documented `editService` workflow (GET config → modify → POST back) triggers an equivalent per-service restart under the hood — no full server restart required. This confirms that ArcGIS Server re-reads service config at service-start time rather than once at process startup.

The implication for filesystem modification: if a service is stopped via the REST API (releasing its in-memory config), a config-store JSON file edited on disk, and the service started again via the REST API, the change should be picked up. This is distinct from editing files while the service is running (which would be overwritten by the in-memory state).

The proposed workflow:
1. `POST /stop` — stop the service, release in-memory state
2. Edit `<ServiceName.ServiceType>.json` in the config-store
3. `POST /start` — server reads the updated file, service starts with new config

**This is not officially documented or supported by Esri.** The supported path is the `editService` REST endpoint. Direct filesystem editing bypasses the API's validation, may behave differently across ArcGIS Server versions, and risks LOCK file conflicts in multi-machine sites where the config-store is a shared network path.

### When it might be useful

- Bulk property changes across many services (faster than individual API round-trips)
- Emergency recovery when the Admin REST API is unavailable
- Programmatic generation of a service config in environments where running the full publish pipeline is not feasible

### Verdict

**Not the default path, but not completely ruled out.** The per-service restart capability makes it more tractable than a full server restart. Worth a targeted test on dev Enterprise: stop a service via REST, edit its JSON, start it, verify the change was picked up. If reliable, it opens a fast-path for config-only changes that avoids a full SD re-publish.

---

## Approach 4 — Versioned service folders

**Maintain `v1/Map`, `v2/Map` etc. as separate services. Rollback = point consumers at the previous version.**

### Why it is limited for Portal-based deployments

Web maps and Portal items that reference `v1/Map/FeatureServer` cannot be silently redirected to `v2/Map/FeatureServer` without updating every Portal item that references them — the same URL hardcoding problem.

Works cleanly for application-controlled consumers that resolve the service URL from configuration (a web app that reads its service URL from an environment variable or config file). Rollback is just changing a config value — no ArcGIS API interaction required.

### Verdict

**Viable for application-controlled consumers; not viable for Portal-native consumers.** A useful convention for new services being designed from scratch, but cannot be retrofitted onto existing deployments.

---

## Recommended pipeline

```
┌─────────────────────────────────────────────────────────┐
│  BEFORE DEPLOY                                          │
│  GET /admin/services/Map.FeatureServer?f=json           │
│  → save config JSON to git (rollback artifact)          │
│  → retain previous .sd in artifact registry             │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│  SHADOW PUBLISH + VALIDATION                            │
│  Publish new SD as Map_shadow                           │
│  Run health check, query test, schema validation        │
│  Pass? → continue. Fail? → delete shadow, abort         │
└────────────────────────┬────────────────────────────────┘
                         │
          ┌──────────────┴──────────────┐
          │ Non-hosted service           │ Hosted layer
          ▼                             ▼
┌─────────────────────┐     ┌──────────────────────────┐
│  RENAME-SWAP        │     │  SD OVERWRITE            │
│  renameService:     │     │  esriServiceDefinition   │
│  Map → Map_old      │     │  Type_Replacement        │
│  Map_shadow → Map   │     │  brief downtime          │
│  seconds downtime   │     └──────────────────────────┘
│  Map_old retained   │
└──────────┬──────────┘
           │
┌──────────▼────────────────────────────────────────────┐
│  POST-DEPLOY VALIDATION                               │
│  Health check on Map                                  │
│  Pass? → success. Fail? → trigger rollback            │
└──────────┬────────────────────────────────────────────┘
           │
     ┌─────┴──────────────────┐
     │ SUCCESS                │ ROLLBACK
     │ delete shadow          │ rename-swap: reverse renames
     │ delete Map_old after   │ OR re-publish from saved .sd
     │ retention period       │
     └────────────────────────┘
```

---

## Open questions

1. **Rename-swap for hosted feature layers**: does `renameService` via the ArcGIS Server Admin API correctly update Portal's internal linkage for hosted layers (data store reference, Portal item ownership), or does it only rename the server-side service leaving Portal state inconsistent? Must be tested before using rename-swap for hosted layers.

2. **Direct filesystem edit + per-service restart**: if a service is stopped via REST, its config-store JSON edited on disk, and the service started via REST — does the change take effect? Test on dev Enterprise with a trivial property change (e.g. `description`).

3. **Rename timing**: how long does `renameService` actually take in practice? The gap between the two renames in the swap is the downtime window.

4. **Shadow resource cost**: does `Map_shadow` consume ArcGIS Server instance capacity while running? Shadow lifetime should be minimised — publish, test, swap or abort within a single pipeline run rather than leaving it running.

5. **SD artifact retention policy**: how many previous SD versions to retain and where. Options: git LFS, S3/Azure Blob, or Portal itself as uploaded `.sd` items.
