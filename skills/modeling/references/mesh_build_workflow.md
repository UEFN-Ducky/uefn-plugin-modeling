---
description: "Piece → merge Static Mesh workflow — no Blueprints, UVs, textures, materials, collision"
metadata:
  label: "Mesh build workflow"
  default_enabled: false
  load_condition: "Building, combining, or finishing a model from mesh pieces; UVs/textures/materials on a Static Mesh; user asks to make a model in UEFN"
---

# Mesh build workflow (Static Meshes only)

## Absolute rules

- **Deliverable = Static Mesh asset** (`SM_…`). Never ship a Blueprint /
  Blueprint assembly as the finished model.
- Temporary pieces are OK while constructing; **merge** all non-moving parts
  before you call the job done.
- Leave separate meshes **only** when a part must move, swap, or stay reusable
  on its own.
- Gameplay BP / Verse logic is out of scope unless the user explicitly asks —
  and even then it must **reference** the finished `SM_`, not replace it.

## Piece decisions

| Situation | Action |
|-----------|--------|
| Fixed body / walls / prop shell | Temporary pieces → **merge** into one `SM_` |
| Door, lid, wheel, rotating bit | Keep as its own `SM_` (moving) |
| Modular kit piece reused many times | Keep as its own `SM_` (reusable) |
| Swappable skin / attachment | Keep as its own `SM_` (swappable) |

Name assets clearly: `SM_Chest_Body`, `SM_Chest_Lid` — not generic `Cube` /
`StaticMeshActor`.

## Build steps

1. **Survey** — decide which parts merge vs stay separate.
2. **Construct** — place or create temporary Static Mesh actors; snap / align.
3. **Pivots** — set a sensible pivot (usually bottom-center or hinge) before merge.
4. **Normals** — fix flipped faces; remesh/repair if needed
   (`skill_read_subskill("modeling", "geometry_scripting")`).
5. **Merge** non-moving pieces (see below) into `{content_root}Meshes/SM_Name`
   (e.g. `/MyProject/Meshes/SM_Name`).
6. **UVs** — UV0 for textures; UV1 / lightmap channel if needed.
7. **Materials** — plan slots (wood / metal / glass…), create `M_` / `MI_` /
   `T_` assets via the `materials` skill, assign to slots.
8. **Collision** — simple box/sphere/capsule or complex as appropriate.
9. **LOD vs Nanite** — pick one path; Nanite skips custom LOD chains.
10. **Cleanup** — destroy temporary construction actors; verify asset exists;
    `save_asset` → `save_current_level()`.

## Merge (verified UEFN path)

Use `unreal.EditorLevelLibrary.merge_static_mesh_actors` with
`unreal.MergeStaticMeshActorsOptions`. Confirm fields with
`describe_class({"class_name": "MergeStaticMeshActorsOptions"})` first.

```python
import unreal

actors = [...]  # StaticMeshActors to merge (non-moving pieces only)
opts = unreal.MergeStaticMeshActorsOptions()
# FIRST: get_project_info() → content_root (e.g. /MyProject/)
opts.set_editor_property("base_package_name", "/MyProject/Meshes/SM_MyProp")
opts.set_editor_property("new_actor_label", "SM_MyProp")
opts.set_editor_property("spawn_merged_actor", True)
opts.set_editor_property("destroy_source_actors", True)
# Preserve purposeful material slots via mesh_merging_settings when needed —
# describe_class MeshMergingSettings / merge options before changing them.

merged = unreal.EditorLevelLibrary.merge_static_mesh_actors(actors, opts)
# Verify: does_asset_exist / get_static_mesh_info on the new SM_ package
```

Also available: `join_static_mesh_actors` (combine components without a new
merged asset) — prefer **merge** when the goal is one reusable `SM_` asset.

Do **not** create a Blueprint to "hold the pieces together" as the model.

## UVs and texturing around the mesh

- **UV0**: texture layout — unwrap / layout so seams are hidden and shells
  don't overlap unless intentional (tiling).
- **UV1**: lightmap / secondary channel when the island needs unique lighting.
- Geometry Scripting helpers:
  `GeometryScript_MeshUVFunctions.recompute_mesh_uvs`,
  `layout_mesh_uvs` (XAtlas) — write back with `copy_mesh_to_static_mesh`.
- Bake helpers (normals / AO / curvature): `GeometryScript_MeshBake` when
  generating detail textures.

## Materials and textures

Use the **materials** skill pack (do not invent Custom/HLSL nodes in UEFN):

```
create_material / create_material_instance  # Materials Store plugin
assign_material_to_mesh  # actor or mesh slot
set_material_instance_texture / _scalar / _vector
recompile_material → save_asset → save_current_level
```

Naming: `T_` textures, `M_` materials, `MI_` instances. Keep ≤ 500 instructions
per material. Load recipes with
`skill_read_subskill("materials", "creating_materials")` when building graphs.

Plan material **slots** before merge so wood/metal/glass stay distinct; avoid
unneeded slot explosion after merge.

## Final checklist

- [ ] Finished asset is `SM_…` (or separate justified `SM_` parts) — **no BP model**
- [ ] Non-moving pieces merged; temps removed
- [ ] Pivot, normals, UV0 (and UV1 if needed) OK
- [ ] Materials/textures assigned and recompiled
- [ ] Collision + LOD/Nanite decided
- [ ] Asset + level saved; verified with `get_static_mesh_info` / `does_asset_exist`
