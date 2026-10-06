# Test-Suite Expansion Plan — `arcgis_cicd` `.sd` generator

Goal: grow from one fixture (`simple.aprx` → hosted FeatureServer, at parity) to a
**matrix-covering test suite** we can develop the generator against. This plan maps
the target matrix (Esri docs), assesses the existing corpus, identifies the
environment/data gaps, and phases the work — including the env setup + build scripts
needed to generate golden-master artifacts.

Sources: matrix from Esri docs (see "Citations"); corpus survey of
`~/dev/aprx-corpus` (346 `.aprx`, 137 in-scope Pro 3.x). Empirically confirmed on the
federated Server 11.5 VM: file-gdb→referenced-feature = error 00134; map-image *can*
reference a file gdb; `FEDERATED_SERVER`+`FEATURE` is rejected.

---

## 1. The target matrix

Two artifact **families** — this is the load-bearing scope boundary:

- **Family A — `.sd`-emitting** (our generator's contract): FEATURE, MAP_IMAGE,
  MAP_SERVICE (standalone), TILE, IMAGE_SERVICE, GP. Staged via
  `getWebLayerSharingDraft`/`CreateSharingDraft` → `exportToSDDraft` → `StageService`.
- **Family B — package-only, NO `.sd`**: VECTOR_TILE (`.vtpk`) and SCENE (`.slpk`)
  drafts have **no `exportToSDDraft`** — they publish only via `arcpy.sharing.Publish`
  against a live portal, or ship as packages. *(Verify against our Pro/Server build.)*
  These are out of the `.sd` generator's contract and need a separate package path.
  **DECIDED (user): in scope, but deferred to Phase 5 / later** — bound the core to
  `.sd`-emitting services first.

### Family A cells to support (service × data source × binding)

| Service type | Valid sources | Binding | Staging call | Status |
|---|---|---|---|---|
| **Hosted FeatureServer** | file gdb, shp, csv, sde, DB query | copy only (hosting server) | `getWebLayerSharingDraft(HOSTING_SERVER, FEATURE)` | ✅ done (file gdb) |
| **Map image layer** (MapServer) | file gdb, sde, shp, raster, query | **copy OR reference** (file gdb referenceable!) | `getWebLayerSharingDraft(FEDERATED_SERVER, MAP_IMAGE)` | ✅ ref done (file gdb), binding tested |
| **Referenced FeatureServer** | **enterprise gdb ONLY** (single connection) | reference | federated MAP_IMAGE + `extension.feature` | ⛔ needs SDE |
| **Standalone map service** | file gdb, sde, shp, raster | copy/reference | `CreateSharingDraft(STANDALONE_SERVER, MAP_SERVICE)` (+`.ags`) | — |
| **Tile layer** (cached) | any vector/feature | copy (cache built) | `getWebLayerSharingDraft(HOSTING_SERVER, TILE)` | — |
| **Image service** | raster / **mosaic dataset** | copy/reference | `CreateImageSDDraft` or `CreateSharingDraft(…, IMAGE_SERVICE)` | ⛔ needs mosaic |
| **Hosted/standalone table** | file gdb, sde, csv | copy / (ref=SDE only) | as feature | partial |

### Negative cells (the suite must assert these FAIL with the right code)

| Combo | Expected | Why |
|---|---|---|
| file gdb → referenced FeatureServer | **error 00134** | referenced feature needs enterprise gdb |
| file-gdb table → referenced feature svc | **error 00135 / 00033** | same |
| referenced layer over **unregistered** data | **error 00231/00232** (hard) | must register first |
| copy-eligible source unregistered | **warning 24011** (soft, copies) | registration optional for copy |
| raster catalog → image service | invalid | convert to mosaic dataset |

The **valid/invalid pairing per data source** is what makes the suite drive the
constraint logic — every "valid" fixture gets a mirror "invalid" source.

---

## 2. Corpus coverage vs gaps (`~/dev/aprx-corpus`, 137 Pro-3.x projects)

**Covered well** (FileGDB-dominant — 1562 refs):
- File-gdb feature classes + standalone tables — the happy path. ✅
- Rasters on disk (34 projects), subtypes (14), multi-map (22), feature datasets (30).
- Web-service-backed layers (64 projects) — the **"skip/exclude" case** to verify.
- Mobile/SQLite gdb (Utility Network family) — SDE-*syntax* connection strings.

**Gaps — NOT in the corpus** (must be synthesized / provisioned):
- ❌ **Real enterprise SDE geodatabase** — zero `.sde`, no server connections. Blocks
  the entire *referenced-feature* + *query-layer* column.
- ❌ **Mosaic datasets** (`esriDTMosaicDataset` = 0) — blocks image services.
- ❌ **SQL/query layers**, **CIM join/relate layers**, **cloud stores**, **OGC (WMS/WFS)**.
- 207 Pro **2.x XML** projects are out of scope (generator consumes CIM JSON).

**Two corpus facts that affect generator design:**
- **Geometry type is NOT in the `.aprx`** — it lives in the source dataset. Confirms
  the `sd_gdb.read_schema` seam (read the gdb) is mandatory, not optional.
- Licensing/size: the corpus is 31 GB of Esri "Learn" content. Don't commit it or
  derived masters wholesale — see §5 decision.

---

## 3. Filling the gaps — environments + data

The user's instinct ("we probably don't have postgres, but we could fix that with the
existing data") is exactly right — migrate existing file-gdb data into new stores:

1. **Enterprise geodatabase (PostgreSQL).** Stand up PostgreSQL (container or on the
   AWS VM) + the ArcGIS PostgreSQL client libs; `arcpy.management.CreateEnterpriseGeodatabase`;
   register `.sde` with the federated server; load `simple.gdb` (and a richer corpus
   project) via `FeatureClassToGeodatabase`. → unlocks **referenced FeatureServer**,
   **query layers**, **branch-versioned editing**. *Biggest effort; highest value.*
2. **Mosaic dataset.** Build a mosaic from corpus rasters (`CreateMosaicDataset` +
   `AddRastersToMosaicDataset`). → unlocks **image services**.
3. **Query layer.** Once PostgreSQL exists, add a DB query layer fixture.
4. Cloud store / OGC: defer (low priority, needs external infra).

---

## 4. The artifact-generation harness (build scripts)

Generalize `scripts/stage_service.py` into a **corpus-driven, declarative** harness:

- A **matrix manifest** (`fixtures.yaml`/`.json`): each row = `{aprx, map, service_type,
  binding, data_source, target (hosting/federated/standalone), expected: ok|error:00134}`.
- A driver that, per row, runs the right `getWebLayerSharingDraft`/`CreateSharingDraft`
  with the right params, `exportToSDDraft` (prefer **`offline=True`** to avoid a live
  portal where the cell allows it), `StageService`, and captures the `.sd` (+ `.sddraft`).
  For **negative** rows it captures the **analyzer error code** as the expected outcome.
- Runs on the VM (arcpy). Output: golden masters + an outcomes log, copied back into
  `tests/fixtures/`.
- Reuse the fixups already built: gdb-path repair, display-field fix, data-store
  registration (`register_datastore.py`).

This is the "build scripts to create the artefacts" the user flagged — it's real dev
effort, but it's the multiplier: once declarative, adding a matrix cell = one manifest
row + one run.

---

## 5. The test harness (what we develop against)

- Per golden master: `compare_sd(generated, reference, ignore_xml_guids=True)` — the
  oracle we already have.
- **Parametrized** test over the matrix manifest: `build_sd(service_type=…,
  by_reference=…, gdb=…)` vs each master. Drives the **orthogonal axes** refactor of
  `build_sd` (service type × binding × data source).
- **Negative** tests: assert the generator *refuses or flags* invalid combos with the
  matching reason code — mirroring the staging analyzer.

### Open decisions (need a call)
- **Fixtures: synthetic vs corpus-derived. DECIDED (user): synthetic, license-clean**
  fixtures (extend the `simple` family — one per matrix cell) committed to the repo;
  the 31 GB corpus stays **external** as a *stress-test + reference* source — it's a
  great catalogue of "everything you might do in ArcGIS Pro," used to design the
  synthetic fixtures and later to stress them. We don't ship Esri Learn data.
- **Master storage:** `.sd`/`.sddraft` are small (commit directly); source `.gdb` are
  larger (git or git-LFS?). Decide before bulk-adding.
- **Determinism:** staged masters vary by arcpy/Server version — pin the version that
  produced each master in the manifest; treat version bumps as expected re-baselines.

---

## 6. Phasing

- **Phase 0 — file-gdb cells, no new infra (NOW — build out FULLY; this is the core
  every later path is built off).** Refactor `build_sd` into **orthogonal axes**
  (service type × binding × data source), then cover, to whole-tree `compare_sd`
  parity, every file-gdb cell:
  - hosted FeatureServer (copy) ✅ done
  - **referenced map image / MapServer** — golden master in hand (`sd_byref/`),
    binding done; finish the MapServer service-type shell to parity *(immediate next)*
  - **hosted map image / MapServer (copy)** — same shell, copy binding
  - standalone map service (`.ags`), tile layer (cache), standalone tables,
    web-service "skip" layers
  Each synthetic fixture extends the `simple` family; masters staged on the federated VM.
- **Phase 1 — enterprise gdb.** Stand up PostgreSQL, migrate data, register; referenced
  FeatureServer + the 00134/00135 negatives + query layers. *Infra-heavy.*
- **Phase 2 — imagery.** Mosaic dataset → image service.
- **Phase 3 — scale/edge.** Utility Network, subtypes, multi-map, large projects.
- **Phase 5 — package family (B).** Vector tile / scene → separate `.vtpk`/`.slpk` path.
  In scope, deferred (user) — build the `.sd` core first.

---

## Citations (load-bearing)
- Reference vs copy / registration: doc.esri.com … `understanding-reference-registered-data-and-copy-all-data`, `24011` warning, `register-data-with-arcgis-server`.
- 00134 file-gdb-unsupported (web feature layer); 00135 standalone table; 00231 must-register.
- Map image references file gdb: `file-geodatabases-and-arcgis-enterprise`.
- Staging classes: `Map.getWebLayerSharingDraft`, `arcpy.sharing.CreateSharingDraft`, `MapServiceDraft`, `MapImageSharingDraft`, `ImageSharingDraft`, `CreateImageSDDraft`.
- Package-only (no `.sd`): VectorTile/Scene sharing-draft classes + `arcpy.sharing.Publish`.
