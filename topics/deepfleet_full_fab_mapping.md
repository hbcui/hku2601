# DeepFleet Graph-Floor mapping of the native AMHS fab

Primary reference: Agaskar et al., [*DeepFleet: Multi-Agent Foundation Models for Mobile Robots*](https://arxiv.org/html/2508.08574v3), §§3 and 8. The paper uses a directed graph of legal floor locations, global vertex coordinates, static vertex purposes, and dynamic robot attributes. Its Graph-Floor model attaches robots to graph vertices, sends messages over spatial and temporal edges, predicts move/wait and other state attributes, then applies a deterministic legal floor transition. The paper also permits a move of multiple vertices. It does **not** require every location to be a four-neighbor chessboard cell.

## Our full-fab graph

The source of truth is `maps/fab_01/topology_native_v5.json`. The existing `maps/fab_01/full_grid_v1.json` samples **all** of its directed RailPaths at at most 0.25 m along native geometry. The compact [manifest](../../maps/fab_01/deepfleet_floor_v1_manifest.json) records source hashes, counts, and the vertex sets for machines, buffers, Bay regions, station codes and junctions. It references the full grid instead of duplicating its 11 MB edge list. Vertex slots are sorted grid node IDs. A physical XY coincidence does not create an edge: separate crossing lanes remain separate vertices.

The grid's original XLSX SHA-256 and topology SHA-256 both exactly match the processed v17 seed-0 metadata. Thus this projection is tied to the same native map used to collect that trajectory, rather than a lookalike synthetic floor.

| Native object | Graph-Floor mapping | Why |
| --- | --- | --- |
| RailPath | Ordered directed vertex chain; each edge retains RailPath ID, native arc length and speed limit | Preserves one-way connectivity, curvature sampling and per-rail physics |
| Native graph node | Shared junction vertex `n:<id>` | Adjacent RailPaths meet only where the original topology says they meet |
| Machine / station code | Static vertex purpose and station-code-to-vertex lookup | Pickup and delivery locations retain their exact graph identity |
| Buffer | Static buffer vertex | Preserves non-machine service/storage locations |
| Bay | **Set** of vertices from member native nodes and RailPaths, plus a separate dynamic resource token | Bay admission, occupancy and granted/waiting OHT relations affect a region, not one point |
| Area | Set of vertices from an explicit rail/node/Bay catalog | v17 has zero Areas; a future dataset without the catalog fails instead of inventing a location |
| Merge / queue | Dynamic resource at its native node vertex, with capacity, closure, holders and ordered directional queues | Queue order and grants cannot be represented by plain cell occupancy |
| OHT | Identity-bearing dynamic record at `(vertex, within-cell phase)`, with global pose/heading, speed, acceleration, mode, task, target, route, requests, grants and its separate static physical configuration | 30 Hz motion inside a cell must remain observable; multiple OHTs can share a quantized vertex |
| Task | Dynamic record with pickup/delivery vertex, lifecycle, priority and assignment edge | Task arrival and assignment persist beyond one frame |
| Route action | Ordered native junctions **expanded to every directed floor vertex**, plus target vertex, OHT ID and event time | The controller's route commitment remains the sole high-level action and persists across frames |
| External task event | Pickup/delivery vertices or lifecycle/assignment relation | Exogenous task stream is an input, not a routing action |

The projection is implemented in [`floor_scene.py`](../../src/deepfleet_hybrid_v1/floor_scene.py); legal position and movement conversion are in [`vertex_floor.py`](../../src/deepfleet_hybrid_v1/vertex_floor.py). Original raw rows are retained as `source` in the projected records for any fields not selected as model features. The projection does not choose new routes, reorder a Merge queue or simulate traffic policy.

## What real v17 data says about the time step

The [seed-0 scene audit](../../outputs/deepfleet_floor_scene_seed0_audit.json) maps all 300 OHTs, all 40 Bays, 545 machines, 899 buffers, 4,014 RailPaths and 30,787 directed edges. At frame 30,000 it maps 204 active tasks and 130 Merge resources. At frame 152 it maps eight route commits and ten task lifecycle events in their observed order. Some projected vertices contain two OHTs (12 shared vertices in the initial state), so a single-ID vertex raster would lose identities and a hard one-OHT-per-cell collision rule would change the native physics.

The audit also attaches all 300 records from `oht_static_config.json`, finds no dangling task/OHT relations in its sampled frames, and maps every sampled task to both graph endpoints. The v17 map has no configured Areas; the mapper accepts an explicit Area rail/node/Bay catalog for a future dataset and refuses to invent an Area location when that catalog is absent.

The [seed-0](../../outputs/deepfleet_vertex_floor_seed0_audit.json) and [seed-1](../../outputs/deepfleet_vertex_floor_seed1_audit.json) movement audits found **zero illegal transitions at a one-frame, 30 Hz interval** among 3,600 and 7,200 sampled OHT pairs. Every observed pair stays at its vertex or crosses one directed edge. But 90–91% of those vertex labels say “wait” and many of those OHTs have positive speed. A vertex-only state would therefore confuse motion inside a 0.25 m cell with stopping. The cell phase is retained as a continuous state variable, and training should supervise both edge crossing and phase/speed. A longer fixed stride can traverse several RailPaths; one must retain its intervening native frames rather than connect endpoint vertices directly.

Against native curved geometry, a vertex alone has about 0.064 m mean position error in both samples. Interpolating with cell phase reduces the mean to 0.00046 m (seed 0) and 0.00040 m (seed 1); the largest sampled error is about 0.015 m. These are mapping errors on sampled observed states, not model forecast errors.

## Architectural boundary

The existing `deepfleet_hybrid_v1/model_v2.py` still uses **RailPath-level** floor tokens and the prior continuous motion projection. The new map, scene and movement adapters are implemented and audited, but previous checkpoints are not vertex-floor models. A training/inference revision must consume these vertex/phase states, inject current occupancy and resource/route relations into graph nodes and edges, predict legal successor/phase updates, and feed predicted dynamic state back each step. The static graph should be reused, not predicted. Free-rollout evaluation should compare actual OHT identity, physical pose, route progress, task and resource relations and congestion, including shared-vertex cases.
