# q3radiant/EntityListDlg.cpp — Enhanced Analysis

## Architectural Role

`EntityListDlg` is a level-editor UI panel that provides hierarchical browsing and property inspection of map entities. It bridges the map's in-memory entity graph (populated during BSP load) with the editor's selection/manipulation subsystem. As part of q3radiant's editing interface, it has **no role in the runtime engine**; it's a tool-only component consumed during offline level design.

## Key Cross-References

### Incoming (who depends on this file)
- **MainFrm.cpp** / window manager: instantiates and docks the dialog window; routes entity-selection events
- **RadiantDoc.cpp** / document model: notifies dialog when entity graph is modified (spawned, deleted, properties changed)
- **EntityW.cpp** / entity inspector: sibling UI component; both read from same global `entities` linked list

### Outgoing (what this file depends on)
- **Global entity state**: `world_entity` (root singleton), `entities` (doubly-linked entity list from `Entity.cpp`)
- **Entity types** (`Entity.h`): `entity_t` (contains `eclass`, `brushes`, `epairs` list) and `epair_t` (key–value properties)
- **Selection subsystem**: calls `Select_Deselect()` and `Select_Brush()` (from `Select.cpp`) to integrate tree selection with the editor's 3D viewport
- **UI refresh**: `Sys_UpdateWindows(W_ALL)` triggers redraw of all viewport panels after selection change

## Design Patterns & Rationale

**Two-pane hierarchical browser:**
- Tree pane groups entities by `eclass->name` (e.g., "light", "info_player_deathmatch") using a `CMapStringToPtr` temporary map
- List pane (LVN) displays raw epair key–value pairs of the selected entity
- Pattern mirrors classic outline/properties inspector seen in 3D modeling tools (e.g., Maya Outliner + Attribute Editor)

**Direct pointer storage:** Uses `reinterpret_cast<DWORD>(entity_t*)` to embed entity pointers in tree item data (`SetItemData`/`GetItemData`), avoiding separate id→entity lookup map. This is efficient for small entity counts (~100–500 typical maps).

**Global entity iteration:** Linear scan of `entities` linked list on init; no caching or lazy loading. Acceptable for offline tools but diverges from engine's sector-tree spatial partition (`SV_LinkEntity` / `SV_UnlinkEntity` in server).

## Data Flow Through This File

1. **Init phase** (`OnInitDialog`):
   - Walk global `entities` linked list
   - Group by classname; insert into tree
   - Each leaf stores a `entity_t*` pointer for later lookup

2. **Selection phase** (`OnSelchangedTreeEntity`):
   - Extract `entity_t*` from selected tree item
   - Enumerate its `epairs` linked list
   - Populate list control with key–value strings

3. **Interaction phase** (`OnSelect`, `OnDblclkTreeEntity`):
   - Retrieve entity pointer from tree
   - Call `Select_Brush(entity->brushes.onext)` to highlight in 3D viewport
   - Trigger `Sys_UpdateWindows()` to refresh views

## Learning Notes

- **No runtime footprint**: Like all of q3radiant, this code is compile-out entirely for shipped game binaries
- **Pre-BSP world model**: Entities exist as in-memory graph during editing; at export time, they are flattened into BSP entity string lump (`cl_parse.c` / `CM_LoadMap` rebuild them on load)
- **Contrast with engine**: The runtime engine's `SV_Snapshot` traces PVS to cull visible entities; the editor has no visibility/culling, only selection and property panels
- **MFC idiom (Windows-specific)**: Uses `CDialog`, `CTreeCtrl`, `CListCtrl` from Microsoft Foundation Classes — this entire editor is Windows/MSVC-only (no cross-platform port)

## Potential Issues

- **No validation on epair iteration**: If `entity->epairs` list is corrupted, the iteration in `OnSelchangedTreeEntity` will crash. Offline tools typically assume correct data from their own serialization.
- **Linear entity list scan**: Scales poorly beyond ~1000 entities; a cached `vector<entity_t*>` would be faster, but typical maps are far smaller.
- **Tab-widening heuristic** (line ~125): Splits "Key\tValue" with extra tabs if key < 8 chars — brittle and not consistently used. Modern UIs would use column auto-sizing.
- **No undo/redo of entity selection**: Selection changes don't push undo stack; this is likely by design (selection ≠ edit), but differs from some editors.
