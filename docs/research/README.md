# Research notes

Problem statement, prior-art survey, and deployment and testing research for the
cimkit suite. They were written before the split into separate packages, so
they use the old names:

| Old name | Now |
|---|---|
| `aprx-tools`, `aprx_tools` | [cimkit-git](https://github.com/rileylum/cimkit-git) |
| `arcgis-cicd`, `arcgis_cicd` (SD generation) | [cimkit-sd](https://github.com/rileylum/cimkit-sd) |
| `arcgis-cicd` (publish, rollback, promotion) | cimkit-deploy and cimkit |
| `aprx-corpus` | `cimkit-corpus` |

| File | Covers |
|---|---|
| `PROBLEMS.md` | Pain points and project goals G1–G8 |
| `PRIOR_ART.md` | Existing tools and what doesn't exist yet |
| `DEPLOYMENT_STRATEGIES.md` | Overwrite, rename-swap and rollback options; the recommended pipeline |
| `VALIDATION_EXPERIMENTS.md` | Experiments that need a real Enterprise site |
| `SD_REVERSE_ENGINEERING.md` | What Pro writes into an `.sd`, and what can be generated headlessly |
| `TESTING.md`, `TEST_SUITE_PLAN.md` | Test tiers and the SD test matrix |

`Map.sd`, `*.zip` and `analysis/` are the Pro-generated artifacts these notes
were written from. They are git-ignored and exist only in the local checkout.
