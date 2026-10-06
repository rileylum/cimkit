# cimkit

Headless CI/CD for ArcGIS: version-control project files, build service
definitions without ArcGIS Pro, and publish them with validation and rollback.

cimkit is a family of small packages. Each one is useful alone; this package
will combine them into the end-to-end pipeline.

| Package | Job | Status |
|---|---|---|
| [cimkit-git](https://github.com/rileylum/cimkit-git) | Version control for Esri project files (`.aprx` today) | Released (as `aprx-tools` ≤ 0.2.1) |
| [cimkit-symbology](https://github.com/rileylum/cimkit-symbology) | CIM renderer → REST renderer JSON | Pre-alpha |
| [cimkit-sd](https://github.com/rileylum/cimkit-sd) | Build a Service Definition (`.sd`) without ArcGIS Pro | Pre-alpha |
| [cimkit-deploy](https://github.com/rileylum/cimkit-deploy) | Publish, swap, roll back and health-check services | Not started |
| **cimkit** (this repo) | The pipeline that runs the others | Not started |

## How the pieces connect

The packages share files, not code. Each one reads the previous one's output:

```
map.aprx ──cimkit-git──▶ map.aprx.src/ ──cimkit-sd──▶ map.sd ──cimkit-deploy──▶ live service
```

`cimkit` will hold the pipeline configuration (which services go to which
environments) and the CI templates that run these steps on merge.

## Local layout

All repos are checked out side by side:

```
~/dev/cimkit/
├── cimkit/             ← this repo: the pipeline + research notes
├── cimkit-git/
├── cimkit-symbology/
├── cimkit-sd/          ← depends on cimkit-symbology (resolved from ../ until published)
├── cimkit-deploy/
└── cimkit-corpus/      ← test data, not a git repo
```

## Research

`docs/research/` holds the problem statement, prior-art survey and deployment
research the suite is designed from. Start with `docs/research/PROBLEMS.md`.
