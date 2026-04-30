# YAML Configuration Reference

This document contains detailed explanation of all accepted key-value pairs in `config.yaml`, and what their corresponding effects are.

The simulator looks for `config.yaml` in the **current working directory** at startup (which in practice is the directory containing the built executable). The same parser is used for resume-from-step files generated under `steps/`.

---

## File-path conventions

Several keys reference files by a stem name. The simulator resolves each stem against a fixed directory and appends `.yaml`:

| Key | Resolved as |
|---|---|
| `rest_node_file: <name>` | `models/<name>.yaml` |
| `start_node_file: <name>` | `models/<name>.yaml` |
| `init_velocity_file: <name>` | `models/<name>.yaml` |
| `seaming_points_file: <name>` | `seam/<name>.yaml` |

For mesh-name keys (`fixed_trig_meshes[*].name`, `fixed_tet_meshes[*].name`, `animated_meshes.directory`), the value is a path stem under `models/`; the loader for fixed/animated meshes appends the appropriate extension and (for animations) the per-frame index.

All paths are relative to the working directory at runtime, **not** to the YAML file itself.

---

## Top-level scalar keys

Every key in this section is **optional** unless explicitly noted; defaults come from `SolverConfig::DefaultInfo()` in `SolverConfig.cpp`.

### Simulation control

| Key | Type | Default | Description |
|---|---|---|---|
| `continue_on_step` | bool | `false` | Resume a previously-saved simulation. If `true`, the loader also reads `current_step` and the per-step state (`control_nodes`, `est_velocity`). The serialised `steps/step_*.yaml` files always have this set to `true`. |
| `current_step` | uint | — | Step index to resume from. Read only when `continue_on_step: true`. |
| `max_steps` | uint | `1000` | Stop the simulation after this many timesteps. |
| `max_newton_iterations` | uint | `0` (must be set) | Cap on Newton iterations per IP sub-solve. Not initialised by `DefaultInfo()`; with C++20 designated-initializer rules it value-initialises to `0`. Note: the Newton loop is `do { … if (cnt >= max) break; } while (residue > tol);`, so `0` does not skip the iteration entirely — exactly one Newton pass runs, then the post-iteration cap check breaks. Always set this to a meaningful positive value. |
| `max_ls_iterations` | uint | `500` | Cap on backtracking line-search iterations per Newton step. |
| `residue_tolerance` | float | `1e-2` | Newton convergence tolerance (used as `tolerance` internally). |
| `timestep` | float | `1e-2` | Δt in seconds. |

### Quadrature

| Key | Type | Default | Description |
|---|---|---|---|
| `quad_scheme` | string | `global` (in practice) | Scheme for placing Gauss quadrature points. See valid values below. **Caveat:** `DefaultInfo()` initialises this to `local-edge-dense`, but `LoadExperimentLayout` calls `QuadStrToScheme` unconditionally — when the key is omitted, the empty string hits the warning fallback and the scheme is overwritten with `global` (`GLOBAL_UNIFORM`). Set this key explicitly if you want a `local-*` scheme. |
| `quad_order` | uint | `6` | Per-element Gaussian quadrature order. For `local-*` schemes, the per-patch quadrature order is hardcoded to 2 and this field is ignored on that axis. |
| `per_patch_mesh` | uint | `2` | Number of mesh subdivisions per B-spline patch (used to construct the contact / render triangle mesh). |
| `cache_quad_points` | bool | `false` | If `true`, write computed quadrature data under `quad_cache/` after preparation. |
| `use_cached_quad_points` | bool | `false` | If `true`, read quadrature data from `quad_data/surf_<i>.yaml` instead of recomputing. **Note the asymmetry with `cache_quad_points`:** the cache is *written* to `quad_cache/` but *read* from `quad_data/`. The expected workflow (per the inline comment in `Solver.cpp`) is to run once with `cache_quad_points: true`, then manually rename `quad_cache/` → `quad_data/` and rerun with `use_cached_quad_points: true`. |

Valid `quad_scheme` values (parsed in `QuadStrToScheme`):

- `local` — uniform quadrature per cell.
- `local-edge-dense` — denser on edges/corners (3-3, otherwise 2-2).
- `local-chessboard` — `local-edge-dense` plus alternating 2-1 / 1-2 pattern across cells.
- `local-chessboard-dense` — alternative 2-2 / 1-1 pattern.
- `local-dual-chessboard` — edge 1-1, interior 1-2/2-1 in the dual grid.
- `global` — uniform quadrature per element.
- `global-limited` — global with limited 2-2 per cell.
- `global-edge-dense` — global with denser edges (3-3).
- `global-chessboard` — global with alternating 2-1 / 1-2 chessboard pattern.

Unknown strings warn and fall back to `global`.

### Material parameters

`stretch_stiffness` and `shear_stiffness` are *derived*, not read directly. Specify Young's moduli + ratios and the simulator computes the actual stiffnesses.

| Key | Type | Default | Description |
|---|---|---|---|
| `stretching_youngs_modulus` | float | `1e5` | Young's modulus E used for membrane stretching. Setting this triggers recomputing `stretch_stiffness = E / (2(1+ν))` and `shear_stiffness = shear_stretch_ratio * stretch_stiffness`. |
| `shear_stretch_ratio` | float | (hard-required when `stretching_youngs_modulus` is set) | Ratio between shear and stretch stiffness. Loaded with the non-optional `LoadDataFromYaml`: if you set `stretching_youngs_modulus` but omit this key, the loader throws an exception that is silently caught at the top level, the message "No config specified, using default config." is logged, and **all subsequent config keys in `LoadExperimentLayout` are skipped** — the partially-loaded config retains whatever was read before this point. |
| `bending_youngs_modulus` | float | `1e6` | Young's modulus for bending. Setting this triggers `bending_stiffness = E·t³ / (12(1−ν²))`. **Caveat:** when this key is *omitted*, `DefaultInfo()` precomputes `bending_stiffness` using a divisor of `24` instead of `12`, so the resulting stiffness is half of what the documented formula would give. Set this key explicitly to get the documented behaviour. |
| `poisson_ratio` | float | `0.45` | Poisson's ratio ν (also used when computing Lamé constants λ and μ). |
| `thickness` | float | `1e-3` | Cloth thickness in metres. Affects bending stiffness, volume, and ground/contact behaviour. |
| `density` | float | `200.` | Volumetric density of the cloth. |
| `gravity` | float | `9.81` | Gravitational acceleration (`gravConst`). |

### Contact / friction

| Key | Type | Default | Description |
|---|---|---|---|
| `enable_contact` | bool | `true` | **Despite the name, this does NOT globally toggle the IPC contact barrier.** Barrier energy, friction energy, spatial-hash construction and `BuildCollisionSets` all run unconditionally. The flag only gates the call to `UpdateContactMesh()` after a control-point update (`Solver.cpp:4752`). Setting it to `false` produces stale contact-mesh state, not a barrier-free simulation. |
| `disable_interpatch_contact` | bool | `false` | If `true`, suppress contact tests between patches that belong to the same B-spline surface. |
| `contact_threshold` | float | `1e-3` | Contact-barrier activation distance (`d̂` in IPC). |
| `contact_stiffness` | float | unset | Optional explicit contact stiffness; if absent, the solver auto-tunes it. |
| `friction_mu` | float | `0.1` | Coulomb friction coefficient (used by IPC's friction model). |
| `ground_height` | float | `-5.0` | World-space **z**-coordinate of the static ground plane. The ground is constructed as `Plane(normal=(0,0,1), offset=ground_height)` (`Solver.cpp:29`), so the plane is `z = ground_height`, not `y = ground_height` as the name might suggest. |

### Seams

| Key | Type | Default | Description |
|---|---|---|---|
| `seam_strength` | float | `1.0` | **Currently dead config.** Originally a penalty multiplier for the seam-energy formulation in `Energy/SeamPenalty.cpp` (`SeamValAt`, `SeamGradAt`, `SeamHessAt`). The active solver path treats seams as **KKT equality constraints** via `PrecomputeKKTComponents` (`Solver.cpp:4057+`) and never reads `seamStrength`; every call site of the SeamPenalty energy in `Solver.cpp` is commented out. Setting this has no effect on the simulation in the current code. |

### Output

| Key | Type | Default | Description |
|---|---|---|---|
| `screenshot` | bool | `false` | When the renderer is enabled, dump a frame to `cache/` per step. |
| `write_ipc_mesh` | bool | `false` | Write the IPC contact mesh per step. |
| `write_iter_mesh` | bool | `false` | Write the per-Newton-iteration mesh (debug). Not initialised by `DefaultInfo()` but value-initialises to `false` via designated-initializer rules. |

---

## `bs_surfaces` *(required, sequence)*

A sequence of mappings, one per B-spline cloth patch. Parsed in `LoadSurfaceInfos`.

```yaml
bs_surfaces:
  - resolution:
      x: 34
      y: 34
    linear_resolution:
      x: 34
      y: 34
    rest_size:
      x: 0.8
      y: 0.8
    rest_node_file: skewed-square/skew_mesh_skew_only_30deg_sd_3x3_control
    transformations:
      - type: translation
        content: [-0.3, 0.3, 0.0]
```

| Key | Type | Default | Description |
|---|---|---|---|
| `resolution` | mapping | — | **Required mapping.** Loaded with the non-optional `LoadNodeFromYaml` (`SolverConfig.cpp:233`), so the `resolution:` block itself must be present. |
| `resolution.x`, `resolution.y` | uint | `20`, `20` | Number of B-spline control points along each parametric axis. Each child key is loaded with `LoadDataFromYamlOptional`; if either is omitted from an existing `resolution:` block, the field stays at the `BSSurfaceInfo` constructor default (`20`). |
| `linear_resolution` | mapping | — | **Required mapping.** Same parsing pattern as `resolution`. |
| `linear_resolution.x`, `linear_resolution.y` | uint | `20`, `40` | Resolution of the auxiliary linear mesh used for contact. Children are optional and fall back to constructor defaults (note the asymmetric default: `x=20`, `y=40`). |
| `idx` | uint | `UINT_MAX` | Optional explicit index. The parser does **not** auto-assign from sequence order — if you omit this key the field stays at the constructor default (`std::numeric_limits<UInt>::max()`). Set it explicitly if anything downstream relies on it. |
| `rest_size.x`, `rest_size.y` | float | `0.8` | Physical size (in metres) of the rectangular rest configuration. Ignored if `rest_node_file` is set. |
| `init_ratio.x`, `init_ratio.y` | float | `1.0` | Pre-stretch ratios applied to the initial configuration. |
| `start_config` | string | `flat` | **Currently dead config.** Parsed (`flat` or `vertical`) and serialised back to step logs, but `BSSurface` construction (`BSSurface.cpp:10`) builds the initial control points from `restSize`, `resolution`, `initRatio`, `restPos` and `transformations` only — `info.startConfig` is never read by the constructor or by anything else in the simulator. Setting this has no effect on the initial pose. |
| `init_velocity` | float[3] | `[0,0,0]` | Uniform initial velocity for every control point. Mutually exclusive with `init_velocity_file`. |
| `init_velocity_file` | string | unset | Stem name; the loader reads `models/<stem>.yaml`, expecting a sequence of `[vx,vy,vz]` triples (one per control point). When this key is present, `init_velocity` is not read at all and the uniform `info.initVel` field stays at the constructor default `(0,0,0)`. Per-control-point velocities are stored separately in `info.initVelocities`. |
| `rest_pos` | float[3] | `[0,0,0]` | Uniform translation applied to the rest configuration. |
| `fixed_nodes` | uint[] | empty | Single-dim indices of control points that should be Dirichlet-fixed. |
| `moving_nodes` | sequence of mappings | empty | See below. |
| `rest_node_file` | string | unset | Stem name; loads irregular rest-pose data from `models/<stem>.yaml`. Triggers `irregularDomain = true`. The file must contain `resolution.x * resolution.y` entries of `[u,v]` material-space coordinates. |
| `start_node_file` | string | unset | Stem name; loads initial 3D positions from `models/<stem>.yaml` (one `[x,y,z]` per control point). Independent of irregular-domain mode. |
| `transformations` | sequence | empty | Linear transformations applied to the surface. See **Transformations** below. |

`moving_nodes` entries — Dirichlet conditions with prescribed velocity until a given frame:

```yaml
moving_nodes:
  - id: 17
    vel: [0.0, -0.5, 0.0]
    until_frame: 200
```

| Key | Type | Description |
|---|---|---|
| `id` | uint | Single-dim control-point index. |
| `vel` | float[3] | Velocity vector applied each step until `until_frame`. |
| `until_frame` | uint | Final frame at which the velocity is applied. |

---

## `fixed_trig_meshes` and `fixed_tet_meshes` *(optional, sequence)*

Static collider meshes. Parsed in `LoadFixedMeshInfo`. `fixed_trig_meshes` are surface meshes; `fixed_tet_meshes` are tetrahedral. Both share the same schema:

```yaml
fixed_trig_meshes:
  - name: armadillo_highres
    transformations:
      - type: scaling
        content: [2.0, 2.0, 2.0]
      - type: translation
        content: [0.0, 0.0, -0.5]
```

| Key | Type | Description |
|---|---|---|
| `name` | string | Stem name; the actual mesh is loaded from `models/<name>.<ext>` (extension determined by mesh kind). The YAML parser only stores the string verbatim in `info.meshName`; any special interpretation of paths like `<dir>/<frame>` for picking a single animation frame is performed by the downstream mesh loader, not by `LoadFixedMeshInfo`. |
| `transformations` | sequence | Linear transformations applied to the static mesh. |

---

## `animated_meshes` *(optional, mapping)*

A single moving collider sequence. Parsed in `LoadAnimatedMeshInfo`. Note this is a **mapping**, not a sequence — the simulator currently supports at most one animated mesh per scene.

```yaml
animated_meshes:
  directory: Hiphop-animation
  pattern: "@"
  length: 600
  diff: 0.0001
  stiffness: 5e3
  transformations:
    - type: rotation
      content: [90.0, 0.0, 0.0]
    - type: translation
      content: [0.0, 0.0, -0.5]
```

| Key | Type | Description |
|---|---|---|
| `directory` | string | Subdirectory under `models/` containing the animation frames. **Required**. |
| `pattern` | string | Filename pattern; `"@"` is the placeholder for the integer frame index. **Required**. |
| `length` | uint | Number of frames in the sequence. **Required**. |
| `diff` | float | Tolerance threshold used to determine when the cloth has "reached" the target mesh pose. **Required**. |
| `stiffness` | float | Penalty stiffness pushing the contact mesh toward the prescribed animation pose. **Required**. |
| `transformations` | sequence | Linear transformations applied uniformly to every frame. |

---

## `seams` *(optional, sequence)*

Inter-patch seam constraints. Each entry pairs two surface indices and references a seam-points file.

```yaml
seams:
  - idx_pair: [0, 1]
    seaming_points_file: Hiphop/seam_0-1
```

| Key | Type | Description |
|---|---|---|
| `idx_pair` | uint[2] | Indices into `bs_surfaces`. |
| `seaming_points_file` | string | Stem name; the actual seam definition is loaded from `seam/<stem>.yaml` (see structure below). |

The referenced seam file must itself contain:

```yaml
in_canonical_coord: false
seams:
  - [u1, v1, u2, v2]   # parametric coordinates of one seam point on each surface
  - ...
```

Where `[u1, v1]` is in the parametric space of `idx_pair[0]` and `[u2, v2]` in that of `idx_pair[1]`. If `in_canonical_coord: true`, the simulator normalises coordinates from canonical [0,1] into the surface's parametric range.

---

## `transformations` block (reused)

Wherever a section accepts `transformations:`, it is a sequence of mappings, applied in the listed order:

```yaml
transformations:
  - type: scaling
    content: [s_x, s_y, s_z]
  - type: rotation
    content: [angle_x_deg, angle_y_deg, angle_z_deg]
  - type: translation
    content: [t_x, t_y, t_z]
```

Valid `type` values: `translation`, `rotation`, `scaling`. Rotations are specified in **degrees** as Euler angles `(x, y, z)`. `content` must always be a sequence of three floats.

Unknown types emit a warning and the offending entry is skipped.

---

## Resume-only keys

These appear only in the per-step `steps/step_*.yaml` files emitted by the simulator and are read back when `continue_on_step: true`:

| Key | Description |
|---|---|
| `current_step` | The step index to resume from. |
| `control_nodes` | Per-surface 3D positions of every control point. |
| `est_velocity` | Per-surface estimated velocities of every control point. |
| `animated_meshes.vertices` | Cached world-space positions of the animated collider mesh at the resume step. Written to the step log when an animated mesh is active (`Solver.cpp:3591`) and read back during resume (`Solver.cpp:3674`). May be either an inline sequence of `[x,y,z]` triples or a string filename pointing at such a sequence. |

You normally don't author these by hand — copy a generated `steps/step_*.yaml` to `config.yaml` to resume.

---

## Worked example

A minimal single-patch configuration with two corners pinned:

```yaml
continue_on_step: false
max_ls_iterations: 10
max_newton_iterations: 200
stretching_youngs_modulus: 5e5
shear_stretch_ratio: 0.005
bending_youngs_modulus: 1e1
seam_strength: 1e1
friction_mu: 0.1
contact_stiffness: 1e5
max_steps: 31
density: 472.6
quad_order: 6
per_patch_mesh: 2
screenshot: false
write_ipc_mesh: true
write_iter_mesh: false
enable_contact: true
contact_threshold: 0.0003
poisson_ratio: 0.243
thickness: 0.00518
timestep: 0.01
residue_tolerance: 0.01
ground_height: -5.0
quad_scheme: local-edge-dense
cache_quad_points: true
use_cached_quad_points: false
disable_interpatch_contact: true
bs_surfaces:
- resolution:
    x: 50
    y: 50
  linear_resolution:
    x: 50
    y: 50
  fixed_nodes: [0, 49]
```

For more end-to-end examples covering animated meshes, seamed garments, and multi-surface scenes, see `B-Spline-IPC/sample-configs/`.
