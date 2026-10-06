# Validation Experiments

Concrete experiments to validate Approach 1 (clone, test, overwrite/rename-swap) and Approach 3 (direct filesystem modification) before committing to either as the deployment strategy.

---

## Approach 1 — What needs validating

Three separate questions, each a distinct experiment.

### Experiment 1A — SD overwrite (Path A baseline)

The simplest thing to confirm first. Establishes that a headless-built SD is accepted by Portal and that overwrite preserves the Portal item ID.

**Steps:**
1. Publish `Test_Map` from `Map.sd` (existing artifact from reverse engineering)
2. Record the Portal item ID and service URL
3. Modify the SD trivially — change the description in `esriinfo/iteminfo.xml`, repack as 7-zip
4. Upload and publish the modified SD with `esriServiceDefinitionType_Replacement`
5. Verify: Portal item ID unchanged, description reflects the change

**Proves:** SD overwrite preserves item ID. Headless-built SD is accepted by Portal.

---

### Experiment 1B — Rename-swap for non-hosted services

**Steps:**
1. Publish `Test_Map` (non-hosted service — MapServer or database-backed FeatureServer)
2. Record: Portal item ID, Portal item `url` field value
3. Publish `Test_Map_shadow` from a modified SD (different description so versions are distinguishable)
4. Record: shadow Portal item ID
5. `POST /admin/services/Test_Map.MapServer/rename` → `Test_Map_old`
6. `POST /admin/services/Test_Map_shadow.MapServer/rename` → `Test_Map`
7. Check: `GET /rest/services/Test_Map/MapServer?f=json` — does description match the new version?
8. Check: original Portal item (by stored item ID) — does `url` field still point to `…/Test_Map/MapServer` and return HTTP 200?
9. Check: shadow Portal item (by stored item ID) — is its URL now broken?
10. Rollback test: rename `Test_Map` → `Test_Map_broken`, `Test_Map_old` → `Test_Map`
11. Check: original Portal item URL resolves to old version again?

**Proves:** rename-swap works for non-hosted services, Portal item URL pointer follows the renamed service transparently, rollback is clean.

---

### Experiment 1C — Rename-swap for hosted feature layers

Same sequence as 1B but publish `Test_Map` as a hosted FeatureServer (use `Map.sd` from the reverse engineering — it is already a hosted FeatureServer SD).

Additional checks after rename:
- Can you query features from the renamed service? (`/Test_Map/FeatureServer/0/query?where=1=1&f=json`)
- Does the Portal item page still show layer count and data correctly?
- Are there any errors in ArcGIS Server logs about data store connection after the rename?

**Proves or disproves:** whether `renameService` is safe for hosted layers. Portal maintains internal linkage between hosted services and the managed data store beyond just the URL — this experiment determines whether a server-level rename breaks that linkage.

**This is the critical unknown.** If this fails, rename-swap is only usable for non-hosted services and SD overwrite (Path A) remains the default for hosted layers.

---

## Approach 3 — What needs validating

### Experiment 3A — Per-service restart works

Prerequisite for everything else. Confirms that stop/start operates at the individual service level without affecting other services or requiring a full server restart.

**Steps:**
1. `POST /admin/services/Test_Map.FeatureServer/stop`
2. `GET /admin/services/Test_Map.FeatureServer/status` → expect `configuredState: STOPPED`
3. Check another running service is unaffected
4. `POST /admin/services/Test_Map.FeatureServer/start`
5. `GET /admin/services/Test_Map.FeatureServer/status` → expect `configuredState: STARTED`

**Proves:** per-service stop/start works independently without touching other services or requiring a server restart.

---

### Experiment 3B — Filesystem edit picked up on service start

Requires SSH/RDP access to the ArcGIS Server machine to reach the config-store directory.

**Steps:**
1. `GET /admin/services/Test_Map.FeatureServer?f=json` → note current `description` value
2. `POST /admin/services/Test_Map.FeatureServer/stop`
3. On the server filesystem: locate and edit `<config-store>/services/Test_Map.FeatureServer/Test_Map.FeatureServer.json` — change `description` to a distinct value
4. `POST /admin/services/Test_Map.FeatureServer/start`
5. `GET /admin/services/Test_Map.FeatureServer?f=json` → is `description` the edited value or the original?

**Proves:** whether config-store files are read fresh on service start, or whether the server process holds configs in memory above the service level. A positive result (edited value appears) opens filesystem modification as a viable fast-path for config-only changes.

---

### Experiment 3C — Edit while service is running (failure case)

Confirms the assumption that editing config-store files while the service is running has no effect and is not overwritten with corruption.

**Steps:**
1. With `Test_Map` running, edit its config-store JSON on disk (change `description`)
2. Wait 30 seconds
3. `GET /admin/services/Test_Map.FeatureServer?f=json` → has the in-memory value changed?
4. `POST /admin/services/Test_Map.FeatureServer/restart`
5. `GET /admin/services/Test_Map.FeatureServer?f=json` → does the restarted service pick up the file change?

**Proves:** the boundary condition — edits while running are ignored (server wins), but a restart after editing picks up the change. Also confirms whether a restart alone (rather than stop + edit + start) is sufficient.

---

### Experiment 3D — Filesystem edits via a GP service

A custom Python toolbox published as a GP service runs on the ArcGIS Server machine under the server's own service account, giving it filesystem access to the config-store without requiring external SSH/RDP. The GP service is callable via the REST API from any machine that can reach the server — including a CI pipeline with no direct server access.

**The proposed GP tool:**
A single-tool Python toolbox that accepts a service name and a JSON patch (key/value pairs to update), then performs the stop → edit → start cycle internally:

```python
import arcpy, json, pathlib, requests

class EditServiceConfig(object):
    def execute(self, parameters, messages):
        service_name = parameters[0].valueAsText   # e.g. "Map.FeatureServer"
        patch_json   = parameters[1].valueAsText   # e.g. '{"description": "v2"}'
        server_url   = parameters[2].valueAsText   # e.g. "https://server.example.com/arcgis"
        token        = parameters[3].valueAsText   # caller-supplied token

        config_store = pathlib.Path(arcpy.GetInstallInfo()["InstallDir"]) \
                       / "usr/config-store/services" / service_name
        config_file  = config_store / f"{service_name}.json"

        admin = f"{server_url}/admin"
        requests.post(f"{admin}/services/{service_name}/stop",   data={"token": token, "f": "json"})

        config = json.loads(config_file.read_text())
        config.update(json.loads(patch_json))
        config_file.write_text(json.dumps(config))

        requests.post(f"{admin}/services/{service_name}/start",  data={"token": token, "f": "json"})
```

**Steps:**
1. Build and publish the toolbox as a GP service (admin-only access)
2. From a machine with no direct server access, call via REST:
   ```
   POST /rest/services/Admin/EditServiceConfig/GPServer/EditServiceConfig/submitJob
   Body: service_name=Test_Map.FeatureServer&patch_json={"description":"gp-edit-test"}
   ```
3. Poll the job until complete
4. `GET /admin/services/Test_Map.FeatureServer?f=json` → verify description changed

**Additional checks:**
- Does the GP job complete successfully when calling the external Admin API URL with a token from within a running GP execution context?
- Does the service account have write permission to the config-store by default, or does it need explicit grant?
- In a multi-machine site, does the GP tool run on a specific machine, and does that machine's local Admin API (`localhost:6080`) control services across all machines?

**Proves:** whether filesystem modification is achievable without direct server access, purely via the REST API. If this works it is the most CI-friendly path — no SSH keys, no VPN to the server host, just Portal credentials.

**Security note:** this GP service is effectively a remote config-modification endpoint. It must be restricted to admin-level Portal users and should validate the service name input to prevent path traversal. Not something to leave publicly accessible.

**Dependency:** only worth running if Experiment 3B confirms that stop → filesystem edit → start is actually picked up by the server. If 3B fails, 3D fails for the same reason.

---

## Access requirements

| Experiment | Portal REST | Server Admin REST | Server filesystem (SSH/RDP) |
|---|---|---|---|
| 1A — SD overwrite | Yes | Yes | No |
| 1B — Rename-swap non-hosted | Yes | Yes | No |
| 1C — Rename-swap hosted | Yes | Yes | No |
| 3A — Per-service restart | No | Yes | No |
| 3B — Filesystem edit | No | Yes | Yes |
| 3C — Edit while running | No | Yes | Yes |
| 3D — GP service filesystem edit | Yes | Yes | No (GP service acts as the agent) |

## Recommended order

Run **1A first** — it validates the headless SD generation path end-to-end and gives you a working test service for all subsequent experiments. If 1A fails, the SD format needs fixing before anything else is meaningful.

Then **3A** — quick to run, no filesystem access needed, and unblocks 3B/3C.

Then **1B and 1C in parallel** if you have two test services available, otherwise 1B then 1C.

Then **3B and 3C** once filesystem access is arranged, and **3D** in parallel with or after 3B — it depends on 3B's result but can be set up independently.
