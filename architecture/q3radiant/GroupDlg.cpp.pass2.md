# q3radiant/GroupDlg.cpp — Enhanced Analysis

## Architectural Role

This file implements the **Group system**, an editor-only organizational feature for Radiant level designers. Groups are purely a UI/persistence construct—they exist to help designers organize brush hierarchies within the level editor and serialize as `group_info` entities into `.map` files. The runtime engine discards these entities, never reading or interpreting group information; they serve no gameplay function. The code bridges the editor's in-memory group representation (linked list of `group_t` + tree UI widget) with the map file's entity-based serialization format.

## Key Cross-References

### Incoming (who depends on this file)
- **Radiant editor main loop** (via `CGroupDlg` dialog class and global `g_pGroupDlg`) — user UI events (button clicks, tree edits), map load/save triggers
- **Map I/O pipeline** — `Group_Init()` called on map load to reconstruct hierarchy from saved `group_info` entities; `Group_Save()` called on map write to serialize groups back
- **Brush lifecycle management** (elsewhere in Radiant) — calls `Group_AddToProperGroup()`, `Group_RemoveBrush()`, `Group_AddToWorld()` when brushes are created/deleted/moved
- **BSP entity parsing** — `Group_Add()` invoked when world entity loading encounters `group_info` classname

### Outgoing (what this file depends on)
- **Editor globals**: `g_qeglobals.m_bBrushPrimitMode` (feature gate; groups only active in brush-primitives mode), `world_entity` (loaded BSP entity state), `active_brushes`/`selected_brushes` (brush doubly-linked lists)
- **Entity/brush structures**: reads/writes `epair_t` (key-value pairs via `ValueForKey()`, `SetKeyValue()`, `DeleteKey()`), `brush_t` (brush pointers stored in tree via `SetItemData()`), `entity_t` (group metadata)
- **Memory**: `qmalloc()` for heap allocation
- **Debugging**: `Sys_Printf()` for debug warnings
- **MFC framework**: `CDialog`, `CTreeCtrl`, `TVINSERTSTRUCT` for tree UI rendering and event dispatch

## Design Patterns & Rationale

**Dual-representation architecture**: Groups live in two places simultaneously—`g_pGroups` linked list (C data structure for queries) and `m_wndTree` tree control UI (visual representation for user interaction). They must stay synchronized. When the map loads, `Group_Init()` reconstructs both from saved entities. When the user edits groups via the tree UI, handlers update `g_pGroups`.

**Epair-based serialization**: Groups are stored as entities (`classname="group_info"`) with a single `"group"` key holding the hierarchical name delimited by `@`. This reuses the existing BSP entity string format, avoiding a custom binary storage mechanism. The hierarchical naming scheme (`"root@subfolder@group"`) encodes the tree structure without requiring explicit parent pointers.

**Mode-gating**: Every public function checks `g_qeglobals.m_bBrushPrimitMode`. Groups are a brush-primitives-specific feature; the code silently no-ops when the flag is off, preventing corruption if groups are accessed in incompatible editor modes.

## Data Flow Through This File

**Load-time flow:**
1. Map file loaded; BSP entity strings parsed (elsewhere)
2. Entity with `classname="group_info"` encountered → `Group_Add(entity_t*)` called
3. Creates `group_t` struct, stores epairs, inserts tree node under "World" root
4. `Group_Init()` called after all entities loaded; iterates `active_brushes`/`selected_brushes`
5. For each brush, reads its `"group"` epair and calls `Group_AddToProperGroup()` to link brush into correct tree node

**Save-time flow:**
1. `Group_Save(FILE*)` walks `g_pGroups` linked list
2. For each group, writes a `group_info` entity with the group name as epair

**Runtime (editor interaction):**
1. User drags brush onto tree node → tree event fires → message handler updates brush's `"group"` epair
2. User renames group → tree edit handler updates group's epair
3. Brush/group deletions update tree and epair state atomically

## Learning Notes

**Editor vs. engine**: This file is a teaching example of **editor-only subsystems**. The runtime engine (`code/game`, `code/client`, `code/server`) never executes this code. Map compilers (`q3map`, `bspc`) strip `group_info` entities as non-gameplay data. Only the human-facing editor cares about groups—a clean separation of concerns.

**Idiomatic to Quake III era**: The design reflects early-2000s practices:
- Win32 MFC for GUI (now considered heavyweight; modern engines use Qt or Dear ImGui)
- Entity-based metadata serialization (pragmatic for a BSP-centric engine, but verbose)
- Hierarchical delimiter naming (`@`) to encode trees without explicit parent/child pointers—a workaround for limited data structures in the map format

**Comparison to modern engines**: Unreal Editor and Unity use dedicated scene-graph data structures (not entities) to store editor-only metadata (groups, layers, locks). They serialize to separate `.umap` binary formats, not the gameplay entity format. Radiant predates this; it conflates editor and gameplay data in the same entity pool, then filters at compile time.

## Potential Issues

1. **Memory leak in `Group_Init()`**: Lines 255–265 iterate `g_pGroups` to free epairs with a `TODO` comment: "FIXME: we leak, delete the groups on the way." The epair cleanup runs, but the group structs themselves are never freed. Minor issue (typically small dataset), but indicates incomplete implementation.

2. **Unguarded tree access**: `Group_AddToItem()` directly calls `m_wndTree.InsertItem()` without null-checking the tree control, assuming it's always initialized. If called before dialog creation, this crashes.

3. **Synchronization risk**: The linked list (`g_pGroups`) and tree UI (`m_wndTree`) must stay synchronized. If brushes are deleted by other code paths that don't call `Group_RemoveBrush()`, the tree will become inconsistent with reality.
