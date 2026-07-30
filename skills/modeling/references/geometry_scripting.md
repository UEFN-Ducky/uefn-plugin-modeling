---
description: "GeometryScriptingCore recipes — booleans, remesh, repair, UVs, bake, bone weights"
metadata:
  label: "Geometry Scripting"
  default_enabled: false
  load_condition: "Doing procedural mesh edits — booleans, remesh, hole-fill, UV layout, baking, or skin weights"
---

# Geometry Scripting recipes

`GeometryScriptingCore` operates on a **DynamicMesh** you load from an asset,
edit, then write back. The load → edit → write-back frame is mandatory: edits to
the in-memory mesh do nothing until copied back.

## Load / write-back frame

```python
# FIRST: get_project_info() → content_root (e.g. /VideoTest/)
sm = unreal.EditorAssetLibrary.load_asset("/VideoTest/Meshes/rock")
dyn = unreal.DynamicMesh()
opts = unreal.GeometryScriptCopyMeshFromAssetOptions()
lod = unreal.GeometryScriptMeshReadLOD()
dyn, _ = unreal.GeometryScript_AssetUtils.copy_mesh_from_static_mesh(sm, dyn, opts, lod)

# ... edit `dyn` here ...

copy_opts = unreal.GeometryScriptCopyMeshToAssetOptions()
unreal.GeometryScript_AssetUtils.copy_mesh_to_static_mesh(dyn, sm, copy_opts, unreal.GeometryScriptMeshWriteLOD())
unreal.EditorAssetLibrary.save_loaded_asset(sm)
```

Confirm class/method names with `describe_class` before running — geometry
scripting names are long and version-sensitive.

## Booleans

```python
dyn, _ = unreal.GeometryScript_MeshBooleans.apply_mesh_boolean(
    dyn, unreal.Transform(), other_dyn, unreal.Transform(),
    unreal.GeometryScriptBooleanOperation.SUBTRACT,
    unreal.GeometryScriptMeshBooleanOptions())
```

Operations: `UNION`, `SUBTRACT`, `INTERSECTION`. Also `apply_mesh_plane_cut`,
`apply_mesh_mirror`, `apply_mesh_self_union`.

## Repair / cleanup (post-import)

- `GeometryScript_MeshRepair.fill_all_mesh_holes(...)`
- `resolve_t_junctions(...)`, `weld_mesh_edges(...)`
- `remove_degenerate_triangles(...)` / `remove_small_components(...)`

## Simplify / remesh (LOD authoring)

- `GeometryScript_MeshRemeshing.apply_uniform_remesh(...)`
- `GeometryScript_MeshSimplification.apply_simplify_to_triangle_count(...)`

Run per LOD with different targets, writing each into its own `WriteLOD`.

## UVs and baking

- `GeometryScript_MeshUVFunctions.recompute_mesh_uvs(...)`,
  `layout_mesh_uvs(...)` (XAtlas auto-unwrap).
- `GeometryScript_MeshBake` — bake normals / AO / curvature to textures.

## Skeletal: bone weights

- `compute_smooth_bone_weights(...)` — auto-weight vertices.
- `transfer_bone_weights_from_mesh(...)` — copy weights between compatible meshes.
- `copy_bones_from_skeleton(...)` — seed a mesh's bone hierarchy.

These pair with the `animation` skill: build the mesh here, rig/retarget there.

## Rule of thumb

If a call fails, `describe_class` the library (e.g.
`GeometryScript_MeshBooleans`) and read the exact static method names and struct
types — nearly every geometry-scripting error is a wrong method name or a
missing options struct, not a logic bug.
