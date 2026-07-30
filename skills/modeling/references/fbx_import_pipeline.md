---
description: "FBX/glTF import into UEFN — static vs skeletal vs anim, scale/axis, Nanite vs LOD, collision, lightmaps, reimport, fixup_redirectors, failure table"
metadata:
  label: "FBX import pipeline"
  default_enabled: false
  load_condition: "User imports FBX/glTF/OBJ, mesh is tiny/huge/wrong axis, asks about Nanite, LODs, collision, lightmaps, reimport, or skeletal/morph import"
---

# FBX import pipeline (UEFN)

Bring DCC meshes into Content cleanly. Blender export settings:
`skill_read_subskill("blender", "uefn_export")` or `skeletal_export`. Materials after
import: materials pack `blender_handoff` / `texture_import`.

## Golden path (static mesh)

```
# FIRST: get_project_info() → content_root (e.g. /VideoTest/)
import_asset({"source_file": "C:/models/SM_Prop.fbx",
              "destination_path": "/VideoTest/Meshes"})
# or omit destination_path / pass "" and let the listener auto-pin
get_static_mesh_info({"asset_path": "/VideoTest/Meshes/SM_Prop"})   # or actual imported name
# LODs / Nanite / collision via StaticMeshEditorSubsystem (see modeling SKILL.md)
set_mesh_collision({...})   # when the thin tool covers the case
save_asset → save_current_level()
```

Verify with `does_asset_exist` / `get_asset_info` — never assume import worked.
Always use `get_project_info().content_root` — never invent `/Game/Meshes`.

## What you're importing

| Source | Expect | Notes |
|--------|--------|-------|
| Static prop FBX | `StaticMesh` | Combine meshes in DCC when one prop |
| Skeletal FBX | `SkeletalMesh` + Skeleton | Bone names matter for retarget |
| Anim-only FBX | `AnimSequence` on target skeleton | See animation pack |
| glTF/GLB | Similar; materials often need rebuild | Prefer FBX for UEFN habit |

Import options (combine meshes, import morphs, skeleton reuse) are Interchange /
FBX dialog settings — when `import_asset` exposes them, set explicitly; otherwise
`execute_python` with import task APIs after `uefn_editor_python_hints` /
`describe_class`. Don't invent option names.

## Scale & axis triage

| Symptom | Cause | Fix |
|---------|-------|-----|
| Microscopic mesh | DCC meters, UE cm without scale | Scale ×100 or re-export with correct unit |
| Gigantic mesh | Double-applied scale | Re-export; apply transforms in Blender |
| Lying on side | Axis forward/up mismatch | Blender FBX: forward `-Z`, up `Y` |
| Pivot in wrong place | Origin not at ground/hinge | Fix in DCC; reimport |

UEFN: **1 uu = 1 cm**. Blender skills use **meters** — expect ×100 unless export
baked scale. After import: check bounds with `get_static_mesh_info` / place and
`get_actor_bounds`.

## Nanite vs LOD

| Choose Nanite when | Choose classic LODs when |
|--------------------|---------------------------|
| Static, dense, no need for custom LOD chain | Need hand LODs / mobile-style budgets |
| No per-vertex runtime edits | Vertex paint / morph-heavy workflows |

Nanite meshes **ignore custom LODs** and limit some edits — decide before investing
in a LOD chain (`modeling` SKILL.md). Generate LODs via
`StaticMeshEditorSubsystem.set_lods` after `describe_class`.

## Collision

- Prefer simple shapes (box/sphere/capsule) via `set_mesh_collision` /
  `add_simple_collisions`.
- Complex-as-simple only when needed — costlier.
- UCX from Blender: name proxies clearly; engine may still rebuild — verify in PIE.

## Lightmap UVs

- Channel 0 = unique/lightmap-friendly for static lighting when required.
- Generate/unwrap lightmap UVs in-engine via mesh editor APIs if DCC channel is bad
  (`describe_class` on `StaticMeshEditorSubsystem`).
- Overlapping UV0 on unique-lit heroes → shadow/light artifacts.

## Reimport & redirectors

1. Keep source path stable for reimport.
2. After move/rename: `fixup_redirectors` on the folder.
3. Always `save_loaded_asset` / `save_directory` then `save_current_level()`.
4. Skeletal reimport onto **same Skeleton** when updating a character mesh.

## Morphs / skeletal extras

- Morph targets: enable on import if the FBX has shape keys; verify on skeletal mesh info.
- Missing morphs → re-export with shape keys + import morph option on.
- Wrong skeleton → animation pack retargeting / don't force mismatched bones.

## Failure table

| Symptom | Fix |
|---------|-----|
| Import "succeeded" but no asset | Wrong destination; check `get_project_info` path |
| Materials magenta/wrong | Rebuild (`blender_handoff`); import textures |
| No collision | `set_mesh_collision` / simple collisions |
| Faceted shading | Normals / smooth groups; rebuild materials with normal map |
| Anim doesn't play on mesh | Skeleton mismatch; retarget or reimport onto correct SK |
| Old mesh in level | Level still references old; save + fixup redirectors |

## Don'ts

- Don't leave Nanite + hand LOD chain as if both apply.
- Don't skip save after import.
- Don't import a whole Blender scene when you meant one `SM_`.
- Don't skin/retarget until scale and skeleton are verified.

## Related

- Piece merge / UV finish → `mesh_build_workflow`
- Booleans / remesh → `geometry_scripting`
- Materials → `texture_import`, `blender_handoff`
- Anims → animation `retargeting`, `skeletal_export` (Blender)
