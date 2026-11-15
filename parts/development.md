# Development conventions

- Add a map specification under `maps/` and point a scenario config to it.
- Keep generated data out of Git; document each dataset in `data/manifests/`.
- Put reusable transition, topology and evaluation code under `src/MAPD_WM/`; keep CLI parsing under `scripts/`.
- Evaluate on held-out trajectories with predictions fed back at every step. Record legal-move violations and compare against a simple baseline.
- Keep large or running legacy AMHS experiments at their existing paths until their imports, data schema and result checks have been migrated together.

The current pilot separates map indexing (`topology`), label generation (`simulators`), feature construction (`features`), learned transition (`models`), legal-state projection (`state`), supervised training (`training`) and free rollout (`evaluation`). `domain`, `actions`, `inference` and `utils` are reserved for stabilized interfaces in later benchmarks.
