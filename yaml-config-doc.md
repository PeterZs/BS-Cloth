# YAML Configuration Reference

This document contains detailed explanation of all accepted key-value pairs in `config.yaml`, and what their corresponding effects are.

The simulator looks for `config.yaml` in the **current working directory** at startup (which in practice is the directory containing the built executable, or at the same location provided in the example if building using Visual Studio). The same parser is used for resume-from-step files generated under `steps/`.

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
| `continue_on_step` | bool | `false` | Resume a previously-saved simulation. The serialised `steps/step_*.yaml` files always have this set to `true`. |
| `current_step` | uint | — | Step index to resume from. Read only when `continue_on_step: true`. |
| `max_steps` | uint | `1000` | Stop the simulation after this many timesteps. |
| `max_newton_iterations` | uint | `0` (must be set) | Cap on Newton iterations per IP sub-solve. |
| `max_ls_iterations` | uint | `500` | Cap on backtracking line-search iterations per Newton step. |
| `residue_tolerance` | float | `1e-2` | Newton convergence tolerance (used as `tolerance` internally). |
| `timestep` | float | `1e-2` | Δt in seconds. |

### Quadrature

| Key | Type | Default | Description |
|---|---|---|---|
| `quad_scheme` | string | `global` | Scheme for placing Gauss quadrature points. See valid values below. |
| `quad_order` | uint | `6` | Per-element Gaussian quadrature order. For `local-*` schemes, the per-patch quadrature order is hardcoded to 2 and this field is ignored on that axis. |
| `per_patch_mesh` | uint | `2` | Number of mesh subdivisions per B-spline knot span. |
| `cache_quad_points` | bool | `false` | If `true`, write computed quadrature data under `quad_cache/` after preparation. |
| `use_cached_quad_points` | bool | `false` | If `true`, read quadrature data from `quad_data/surf_<i>.yaml` instead of recomputing. |

Valid `quad_scheme` values (parsed in `QuadStrToScheme`):

- `local` — uniform quadrature per cell.
- `local-edge-dense` — interior knot spans are 2×2; cells on a single domain edge get the perpendicular axis bumped to 3; corner cells are 3×3.
- `local-chessboard` — `local-edge-dense` with internal span switches to alternating 2-1 / 1-2 pattern.
- `local-chessboard-dense` — alternative 2-2 / 1-1 pattern.
- `local-dual-chessboard` — The quadrature scheme described in the paper.
- `global` — uniform quadrature per element.

Unknown strings warn and fall back to `global`.

### Material parameters

`stretch_stiffness` and `shear_stiffness` are *derived*, not read directly. Specify Young's moduli + ratios and the simulator computes the actual stiffnesses.

| Key | Type | Default | Description |
|---|---|---|---|
| `stretching_youngs_modulus` | float | `1e5` | Young's modulus E used for membrane stretching. |
| `shear_stretch_ratio` | float | — (required) | Ratio between shear and stretch stiffness. |
| `bending_youngs_modulus` | float | `1e6` | Young's modulus for bending. |
| `poisson_ratio` | float | `0.45` | Poisson's ratio ν (also used when computing Lamé constants λ and μ). |
| `thickness` | float | `1e-3` | Cloth thickness. |
| `density` | float | `200.` | Volumetric density of the cloth. |
| `gravity` | float | `9.81` | Gravitational acceleration (`gravConst`). |

### Contact / friction

| Key | Type | Default | Description |
|---|---|---|---|
| `disable_interpatch_contact` | bool | `false` | Toggle inter-patch contact. May require code changes in IPC implementation. |
| `contact_threshold` | float | `1e-3` | Contact-barrier activation distance (`dHat` in IPC). |
| `contact_stiffness` | float | unset | Optional explicit contact stiffness. |
| `friction_mu` | float | `0.1` | Coulomb friction coefficient (used by IPC's friction model). |
| `ground_height` | float | `-5.0` | World-space **z**-coordinate of the static ground plane for contact. |

### Output

| Key | Type | Default | Description |
|---|---|---|---|
| `screenshot` | bool | `false` | When the renderer is enabled, dump a frame to `cache/` per step. |
| `write_ipc_mesh` | bool | `false` | Write the IPC contact mesh per step. |
| `write_iter_mesh` | bool | `false` | Write the per-Newton-iteration mesh (debug). |

---

## `bs_surfaces` *(required, sequence)*

A sequence of mappings, one per B-spline cloth patch.

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
| `resolution.x`, `resolution.y` | uint | `20`, `20` | Number of B-spline control points along each parametric axis. |
| `linear_resolution.x`, `linear_resolution.y` | uint | `20`, `40` | Resolution of the auxiliary linear mesh used for contact. |
| `idx` | uint | `UINT_MAX` | Optional explicit index. |
| `rest_size.x`, `rest_size.y` | float | `0.8` | Physical size (in metres) of the rectangular rest configuration. |
| `init_ratio.x`, `init_ratio.y` | float | `1.0` | Pre-stretch ratios applied to the initial configuration. |
| `init_velocity` | float[3] | `[0,0,0]` | Uniform initial velocity for every control point. |
| `init_velocity_file` | string | unset | Specify a yaml file which sets per-control-point velocities. Overrides `init_velocity`. |
| `fixed_nodes` | uint[] | empty | Single-dim indices of control points that should be fixed. |
| `moving_nodes` | sequence of mappings | empty | See below. |
| `rest_node_file` | string | unset | Stem name; loads irregular rest-pose data from `models/<stem>.yaml`. Triggers `irregularDomain = true`. The file must contain `resolution.x * resolution.y` entries of `[u,v]` material-space coordinates. |
| `start_node_file` | string | unset | Loads initial 3D positions from `models/<stem>.yaml`. |
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
| `id` | uint | Control-point index. |
| `vel` | float[3] | Velocity vector applied each step until `until_frame`. |
| `until_frame` | uint | The velocity is applied while `curStep < until_frame`. |

---

## `fixed_trig_meshes` and `fixed_tet_meshes` *(optional, sequence)*

Static collider meshes.

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
| `name` | string | File name. Loaded as `models/<name>.obj. |
| `transformations` | sequence | Linear transformations applied to the static mesh. |

---

## `animated_meshes` *(optional, mapping)*

A single moving collider sequence. At most one moving collider sequence is supported.

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
| `directory` | string | Subdirectory under `models/` containing the animation frames. |
| `pattern` | string | Filename pattern; `"@"` is the placeholder for the integer frame index. |
| `length` | uint | Number of frames in the sequence. |
| `diff` | float | Tolerance threshold. Penalty will no longer be applied if the difference is below this value. |
| `stiffness` | float | Penalty stiffness pushing the contact mesh toward the prescribed animation pose. |
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
| `seaming_points_file` | string | Seam detail file name. Used later when loading from `seam/<stem>.yaml`. |

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
| `animated_meshes.vertices` | Cached world-space positions of the animated collider mesh at the resume step. This is necessary as at the end of each frame the current position may not coincide exactly with the prescribed positions. |

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
quad_scheme: local-dual-chessboard
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
