# Migration status

| Existing path | New home or status |
| --- | --- |
| Former `src/WM/toy_mapf` | Copied into the installable `src/MAPD_WM` pilot; historical implementation is isolated in `src/toy_mapf_legacy` |
| `runs/toy_mapf_v2_learned_task` | Indexed in `experiments/toy_directed_grid_v1/README.md`; new runs write to `outputs/toy_directed_grid_v1` |
| Former `src/WM` | Removed after splitting each model generation into an independent top-level package under `src/`; see `src/MODEL_PACKAGES.md` |
| `src/AMHSTrafficLab` | Existing simulator package; future adapter belongs in `src/MAPD_WM/simulators` |
| Former `data/generator` | Moved to the source package `src/amhs_data_generator`; generated trajectories remain under `data/` |
| `data/raw`, `data/processed`, `runs` | Existing large data and artifacts remain in place; no copies or deletions |
| `references/*.pdf` | PDFs moved to `research/papers`; relative symlinks preserve the old paths |

This is a staged migration. The standardized package is the authoritative home for the simplified pilot. Legacy AMHS modules are still directly runnable from their current paths and should be migrated per experiment with regression tests and manifests.
