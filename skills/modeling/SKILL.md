---
source_plugin_id: modeling
name: modeling
description: "Model and process meshes in UEFN — import FBX/glTF, LODs, collision, UVs, Nanite, and Geometry Scripting (booleans, remesh, repair, bake)"
license: MIT
metadata:
  label: UEFN Modeling
  version: 8
  author: UEFN-Ducky
  copyright: Copyright 2026 Mindful Path Company, LLC
  allow_redistribute: true
---

# UEFN Modeling — meshes and geometry

**Epic UEFN MCP:** Settings → MCPs → **UEFN MCP (Epic)** (`unreal-mcp`). Bridge tools: `unreal__list_toolsets` → `unreal__describe_toolset` → `unreal__call_tool` (toolsets — not flat `unreal__create_entity`). Map: `skill_read_subskill("uefn", "epic_mcp")`. Ducky tools below stay for this skill's domain when Epic does not cover it.

**SERIAL:** never parallel `spawn_actor` / `save_current_level` with other heavy
editor calls in the same turn (`skill_read_subskill("uefn", "batch_commands")`).

Mesh and geometry work in UEFN prefers the thin modeling tools. `execute_python`
with `StaticMeshEditorSubsystem` / `GeometryScriptingCore` is a **labelled last
resort** for one reflection call that those tools cannot express — never a
spawn/import/LOD loop (freezes UEFN). One editor mutator per assistant message.

## Hard rule — Static Meshes only (never Blueprints as the model)

When **making a model** in UEFN:

1. Deliverable is always a **Static Mesh** asset (`SM_…`), never a Blueprint /
   Blueprint assembly as the finished model.
2. Build from temporary mesh **pieces** (separate actors/meshes while working).
3. **Merge** all non-moving parts into one Static Mesh when done.
4. Leave separate **only** moving, swappable, or intentionally reusable parts.
5. Finish UVs / textures / materials on the mesh (use the `materials` skill pack).
6. Remove temporary construction actors; save the named `SM_` asset.

Gameplay Blueprint or Verse behavior is a **separate, explicitly requested**
task that references the finished mesh — never substitute a BP for the model.

Load the full piece/merge/UV/material checklist with
`skill_read_subskill("modeling", "mesh_build_workflow")`.

## Tool ladder (in order)

1. **Pipeline/modeling tools**: `import_asset` / `export_asset`;
   `get_static_mesh_info`, `set_mesh_collision`.
2. Load geometry recipes with
   `skill_read_subskill("modeling", "geometry_scripting")` (booleans, remesh,
   repair, bone-weight transfer). Never IDE-Read `~/.claude/skills` /
   `~/.cursor/skills` / `references/*.md`.
3. **Last resort:** `execute_python` for a single `StaticMeshEditorSubsystem` /
   geometry-scripting call the thin tools cannot express. Confirm names with
   `describe_class(class_name="StaticMeshEditorSubsystem")` first. Never loop
   spawn/import/LOD in that script.

## Golden path (import + prep a static mesh)

```
# FIRST: get_project_info() → content_root (e.g. /MyProject/)
import_asset({"source_file": "C:/models/rock.fbx",
                          "destination_path": "/MyProject/Meshes"})
# or omit destination_path / pass "" and let the listener auto-pin
get_static_mesh_info({"asset_path": "/MyProject/Meshes/rock"})   # verify import
set_mesh_collision({...})
# Last resort: one execute_python call on StaticMeshEditorSubsystem if the
# thin tools cannot express the LOD/UV edit — never a loop
save_current_level()
```

Verify with `get_asset_info` / `does_asset_exist` — never assume import worked.
Full FBX/scale/Nanite/reimport playbook:
`skill_read_subskill("modeling", "fbx_import_pipeline")`.

## Golden path (build from pieces → merge)

1. Place / create temporary Static Mesh actors for construction pieces.
2. Align pivots and transforms; keep purposeful material slots.
3. Merge non-moving pieces with `EditorLevelLibrary.merge_static_mesh_actors`
   into a new `SM_` package under `{content_root}Meshes` (e.g. `/MyProject/Meshes`).
4. Delete temporary construction actors if merge did not destroy them.
5. UV layout → create/assign materials (`materials` pack) → collision / LODs
   or Nanite → `save_asset` → `save_current_level()`.

Confirm method names with `describe_class` before running — see
`mesh_build_workflow` for the merge options skeleton.

## Last-resort operations (`execute_python`)

Only when `set_mesh_collision` / `get_static_mesh_info` cannot express the edit.
One script, one asset — never a folder loop. Confirm names with `describe_class`
first. This freezes UEFN if you iterate many meshes.

```python
sm = unreal.EditorAssetLibrary.load_asset("/MyProject/Meshes/rock")  # project path
sub = unreal.get_editor_subsystem(unreal.StaticMeshEditorSubsystem)
# Auto LODs
opts = unreal.EditorScriptingMeshReductionOptions()
sub.set_lods(sm, opts)
# Collision
sub.add_simple_collisions(sm, unreal.ScriptingCollisionShapeType.BOX)
# Nanite
sub.enable_nanite(sm)
unreal.EditorAssetLibrary.save_loaded_asset(sm)
```

Use `describe_class` to confirm method/enum names for this build before running.

## Hard rules

- **Naming**: `SM_` static meshes, `SK_` skeletal meshes, `T_` textures,
  `M_` / `MI_` materials. Folders under `get_project_info().content_root`
  (e.g. `/MyProject/Meshes`) — **never invent `/Game/Meshes`**.
- **Import is not persisted until saved** — after `import_asset`, save the asset
  and `save_current_level()` or the island may not reference it.
- **Nanite meshes ignore custom LODs and can't have per-vertex edits at runtime**
  — decide Nanite vs LOD chain up front.
- Geometry Scripting edits a mesh **copy** in memory; you must copy the result
  back to the asset (`copy_mesh_to_static_mesh`) — see
  `skill_read_subskill("modeling", "geometry_scripting")`.

## After ANY mesh change

`save_asset` (or `save_loaded_asset`) → `save_current_level()`.

## Reference files

- `references/mesh_build_workflow.md` — piece → merge Static Mesh, no Blueprints as models
- `references/geometry_scripting.md` — booleans, remesh, repair, bake
- `references/fbx_import_pipeline.md` — FBX/glTF import, scale, Nanite/LOD, collision, reimport
