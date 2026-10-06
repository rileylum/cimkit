# SD File Reverse Engineering — Headless Publishing Research

## Context

ArcGIS Pro is currently the only officially supported tool for publishing map and feature services to ArcGIS Enterprise / Portal. The `StageService` + `UploadServiceDefinition` ArcPy tools require a licensed ArcGIS Pro install, which blocks container-based or Linux-native CI pipelines.

This document records what we learned by reverse engineering the artefacts ArcGIS Pro produces during a publish, to determine whether headless SD construction is feasible.

The test case was a simple ArcGIS Pro project (`simple.aprx`) containing three feature layers and a table backed by a file geodatabase, published as a hosted FeatureServer to ArcGIS Enterprise via Portal.

---

## Artefacts examined

ArcGIS Pro produces the following during a publish (captured by setting `SaveToSDFile = TRUE`):

| File | Description |
|---|---|
| `Map.msd` | Original MSD — the aprx content extracted for the target map |
| `Updated_Map.msd` | Processed MSD — layout stripped, ready for service packaging |
| `Map.sddraft` | Service definition draft — XML configuration manifest |
| `Map.sd` | Final staged service definition — uploaded to Portal |
| `SharingInfo.json` | ArcGIS Pro internal sharing metadata |
| `SharingJobLog.log` | Step-by-step publish log |
| `Map_PublishResults.json` | Layer ID → field name mapping after publish |

---

## What ArcGIS Pro actually does

### Step 1 — Extract the map from the aprx → `Map.msd`

An `.aprx` is a standard ZIP archive of CIM JSON and XML files. An `.msd` is the same format. ArcGIS Pro extracts the target map's CIM content from the aprx and writes it into an MSD.

**Finding: the layer JSON files are byte-for-byte identical between the aprx and Map.msd.** The only difference is formatting — the aprx stores minified JSON (one line per file), the MSD stores pretty-printed JSON. No transformation of the data occurs.

### Step 2 — Strip layout content → `Updated_Map.msd`

ArcGIS Pro produces a second MSD with layout-related content removed. The changes are four small JSON edits:

| Change | Location |
|---|---|
| Remove layout view from `views` array | `GISProject.json` |
| Remove `groundElevationSurfaceLayer` reference | `map/map.json` |
| Remove layout and ground elevation nodes, renumber IDs | `Index.json` |
| Replace project metadata file with one containing a base64 thumbnail | `Metadata/*.xml` |
| Rewrite `workspaceConnectionString` from aprx-relative path to staging-folder-relative path | all `map/*.json` layer files |

All five changes are Python dict mutations on the parsed CIM JSON. No ArcGIS-specific logic required.

### Step 3 — Write the sddraft

The `.sddraft` is an XML manifest describing the service configuration: name, type (`FeatureServer`, `MapServer`), portal URL, credentials, layer list, extent, spatial reference, `maxRecordCount`, capabilities, and publish type (`esriServiceDefinitionType_New` for first publish, `esriServiceDefinitionType_Replacement` for overwrite). This is a templateable XML document with no computed content.

### Step 4 — Stage: consolidate data and package

For **hosted feature layers** (copy-data mode): ArcGIS Pro reads the source data (file GDB in this case) and copies it into the SD package. This is the 18-second "Consolidating data and staging web layer" step in the publish log.

For **enterprise GDB-backed services** (by-reference mode): no data is copied. The SD contains only the MSD and configuration. This step is trivial.

### Step 5 — Compress into SD and upload

The SD is packaged as a **7-zip archive** (not a standard ZIP) and uploaded to Portal via the REST API. Portal runs the publish job and returns the service URL and item ID.

---

## SD internal structure

```
Map.sd  (7-zip archive)
│
├── manifest.xml
│     The sddraft, promoted to State=Staged with InPackagePath references resolved.
│     This is the routing manifest — tells the server where data lives and what
│     type of service to create.
│
├── serviceconfiguration.json
│     The SVCConfiguration section of the sddraft expressed as clean JSON:
│     service name, type, capabilities, maxRecordCount, and property overrides.
│
├── tilingservice.xml
│     Tiling cache configuration. Empty/default for non-cached services.
│
├── p30/                          (Pro 3.0 format map content — two representations)
│   ├── Updated_Map.mapx
│   │     CIMMapDocument: all layer definitions inlined into a single flat JSON
│   │     document. Type: CIMMapDocument with keys mapDefinition, layerDefinitions,
│   │     tableDefinitions, binaryReferences. This is the authoritative Pro 3.0 source.
│   └── Updated_Map.msd
│         The same CIM data as a ZIP of separate JSON files per layer (legacy format).
│
├── cd/                           (consolidated data — hosted layers only)
│   └── simple.gdb/
│         Source file GDB copied verbatim. All files are binary (Esri proprietary format).
│         Not present for enterprise GDB-backed services.
│
├── servicedescriptions/
│   └── featureserver/
│       ├── featureserver.json
│       │     Pre-computed REST API descriptor — the full JSON response that
│       │     /services/Map/FeatureServer?f=json would return. Contains service-level
│       │     capability flags, and per-layer: fields, geometry type, extent,
│       │     drawing info, edit tracking config.
│       └── featureserveritemdata.json
│             Portal-native layer config: renderer (CIM symbol definitions) and
│             popup field list per layer. Used by the Portal UI.
│
└── esriinfo/                     (portal item metadata)
    ├── iteminfo.xml              title, tags, snippet, extent, spatial reference
    ├── item.pkinfo               package version, document type declaration
    ├── metadata/metadata.xml     full metadata XML + base64-encoded thumbnail PNG
    └── thumbnail/thumbnail.png   300×200 map preview image
```

---

## Binary content audit

| Path | Binary? | Notes |
|---|---|---|
| `cd/simple.gdb/*` | Yes | Esri file GDB format. Not present for enterprise GDB services. |
| `esriinfo/thumbnail/thumbnail.png` | Yes | PNG — generatable from a blank image or map render |
| `p30/Updated_Map.msd` | Yes (ZIP) | Standard ZIP of JSON/XML — we know this format (aprx-tools) |
| Everything else | No | Pure XML or JSON |

For enterprise GDB-backed services, the only binary is the MSD ZIP and the thumbnail PNG. No opaque binary formats.

---

## Feasibility of headless generation

### What is fully generatable without ArcGIS Pro

| Component | How |
|---|---|
| `p30/Updated_Map.msd` | Extract CIM from aprx, apply 5 JSON edits, repack as ZIP |
| `p30/Updated_Map.mapx` | Read all `map/*.json` files from aprx, inline into `CIMMapDocument` wrapper |
| `manifest.xml` | Template from sddraft; update `State` to `Staged` and fill `InPackagePath` |
| `serviceconfiguration.json` | Extract SVCConfiguration from sddraft; write as JSON |
| `servicedescriptions/featureserver/featureserveritemdata.json` | Translate CIM renderer and field definitions from aprx layer JSON |
| `esriinfo/iteminfo.xml` | Template with service name, title, tags, description, extent |
| `esriinfo/item.pkinfo` | Static template |
| `esriinfo/metadata/metadata.xml` | Template with base64 thumbnail |
| `tilingservice.xml` | Static template |
| 7-zip packaging | `py7zr` Python library |

### `featureserver.json` — fully generatable

This file contains three categories of data:

**Static capability flags** — `currentVersion`, `cimVersion`, `syncEnabled`, `supportedExportFormats`, `advancedEditingCapabilities`, etc. These are server version constants. Template values.

**Schema-derived** — field names, types, aliases, lengths, `geometryType`, `objectIdField`, `displayField`. All present in the CIM layer JSON in the aprx under `featureTable`.

**Extent** — the layer bounding box. This is the only part that requires reading the actual data, but it is straightforwardly solvable:

```python
from osgeo import ogr, osr

ds = ogr.Open("simple.gdb")
layer = ds.GetLayerByName("test_pts")
xmin, xmax, ymin, ymax = layer.GetExtent()
# reproject to EPSG:3857 with osr.CoordinateTransformation if needed
```

`ogr.GetExtent()` works identically for file GDB, enterprise GDB (SDE driver), shapefile, GeoPackage, and PostGIS — every realistic source type.

For the overwrite / CI deploy case, the extent can also be fetched from the existing live service:
```
GET /server/rest/services/Hosted/Map/FeatureServer?f=json
```
and used directly for the next deploy, avoiding any data read.

### `cd/` — the only real constraint

For hosted feature layers (copy-data mode), the source GDB must be copied into `cd/`. The GDB files are binary (Esri proprietary format) but they are copied verbatim — no parsing or transformation required.

For **enterprise GDB-backed services** (the primary production use case for this tool), `cd/` does not exist. The SD contains no binary data files whatsoever.

---

## The publish pipeline headlessly

```
aprx (CIM JSON)
    │
    ├─ strip layout → Updated_Map.msd      (JSON edits + ZIP)
    ├─ inline layers → Updated_Map.mapx    (JSON restructure)
    ├─ write manifest.xml                  (XML template)
    ├─ write serviceconfiguration.json     (JSON template)
    ├─ build featureserver.json            (CIM fields + ogr extent)
    ├─ build featureserveritemdata.json    (CIM renderer translation)
    ├─ copy cd/ (hosted only)             (verbatim file copy)
    └─ build esriinfo/                     (XML/PNG templates)
           │
           └─ 7-zip → Map.sd
                  │
                  └─ POST /sharing/rest/content/users/{user}/addItem
                         │
                         └─ POST /sharing/rest/content/items/{id}/publish
```

No ArcGIS Pro. No ArcPy. No Windows. Standard Python (`zipfile`, `json`, `xml`, `py7zr`, `osgeo`).

---

## Open questions

1. **Does Portal require `featureserver.json` and `featureserveritemdata.json` to be present, or does it generate them at publish time?** The fastest way to test: build a minimal SD without them and attempt a publish. The error message (if any) will be specific.

2. **Does the `mapx` need to match the `msd` exactly?** Both contain the same CIM data in different shapes. If Portal uses only one as authoritative, the other may be omittable.

3. **What does a non-hosted (enterprise GDB, MapServer) SD look like?** The test case is a hosted FeatureServer. The SD structure for a server-registered map service will differ — particularly `manifest.xml` (no `cd/` database entries, different `ServerType`, `ByReference=true`). A second test case with an enterprise GDB source is needed to confirm.

4. **7-zip compression settings** — does Portal accept any 7-zip compression level, or does it expect a specific method/level? To be confirmed by testing a rebuilt SD.
