# q3radiant/MapInfo.cpp — Enhanced Analysis

## Architectural Role

This file implements a **map statistics dialog** for the Q3Radiant level editor, displaying compile-time metadata about the current map: total brush count, world-geometry count, and per-entity-class population. Q3Radiant is a standalone offline tool separate from the runtime engine (like q3map and bspc); this dialog exists only to give mappers visibility into structural properties before compilation. It operates entirely against editor-internal data structures and has no runtime engine counterpart.

## Key Cross-References

### Incoming
- **q3radiant UI framework**: The dialog is instantiated by the menu/UI system in `q3radiant/MainFrm.cpp` or equivalent (not visible in provided context) when the user selects a "Map Info" menu item.
- **MFC framework**: Inherits from `CDialog` and uses MFC reflection macros (`DDX_*`, `BEGIN_MESSAGE_MAP`, etc.) for lifecycle binding.

### Outgoing
- **Editor data model**: Reads from global linked-list heads `active_brushes` and `entities` (defined in `q3radiant/map.cpp` or similar, tracking the map structure being edited).
- **Entity class metadata**: Inspects `entity_t→eclass→name` to extract entity class names (populated by `q3radiant/eclass.cpp` at load time).
- **MFC controls**: Writes to `IDC_LIST_ENTITIES` listbox and text edit controls via `UpdateData()` reflection.

## Design Patterns & Rationale

- **MFC dialog pattern**: Standard Windows modal dialog using data exchange (`DoDataExchange`) to bind C++ member variables to GUI controls. Typical for early-2000s VC++ applications.
- **One-shot population**: `OnInitDialog()` runs once on dialog creation, walking the entire editor data model to build summary statistics. No real-time updates or caching; recompute on every open.
- **Map-based entity counting**: Uses MFC `CMapStringToPtr` hash table to count occurrences of each `eclass→name`. Counts are cast through `void*` pointers (type-unsafe; intentional for the era's MFC API).
- **Brush classification**: Separates world geometry (`m_nNet`, brushes with `owner == world_entity`) from total brushes via a single linear scan.

## Data Flow Through This File

1. **Input**: User clicks "Map Info" menu → dialog instantiated.
2. **Brush scan**: Linear walk of `active_brushes` doubly-linked list; each brush increments `m_nTotalBrushes`; if `owner == world_entity`, also increments `m_nNet` (world-geometry count).
3. **Entity scan**: Linear walk of `entities` list; each entity's class name is looked up in `mapEntity` hash table, count incremented, re-stored.
4. **Listbox population**: Iterate hash table, format each entry as `"EntityClassName\t<count>"` (tab-separated for right-aligned display), add to listbox.
5. **GUI update**: Call `UpdateData(FALSE)` to push C++ member vars → control values; dialog displays and blocks.

## Learning Notes

- **Editor vs. runtime separation**: This dialog has zero dependencies on qcommon, renderer, server, or game modules. It's a pure tool built against the editor's internal map representation, not the engine's runtime structures. Mappers never see this code; it informs the compilation pipeline.
- **Linked-list traversal**: The editor uses intrusive linked lists (C-style `next`/`prev` pointers in `brush_t` and `entity_t`) rather than STL containers. The sentinel-headed loop (`for (...; pBrush != &active_brushes; ...)`) is idiomatic to Quake-era C codebases.
- **Type-unsafe void* casts**: The `reinterpret_cast<void*&>(nValue)` pattern (storing an int count in a pointer slot) is a legacy MFC idiom that avoids template containers. Modern C++ would use `CMap<CString, int>` or `std::map<std::string, int>`.
- **No persistence or user interaction**: Unlike the renderer or game logic, this dialog is read-only and ephemeral—a debug/inspection tool. There's no "save map info" or user-configurable thresholds.

## Potential Issues

- **No synchronization**: If the map is edited while the dialog is open, statistics become stale (no observer pattern or real-time updates).
- **Unbounded listbox**: For maps with hundreds of unique entity classes, the listbox has no scrolling visual feedback or sorting; could be hard to navigate.
- **Silently uncounted entities**: Any entity not linked into the `entities` list head won't be counted; doesn't validate against expected entity count.
- **Type safety**: Casting int counts through `void*` is fragile and would fail under 64-bit compilation without care (though this tool is Windows-specific and was never ported to 64-bit).
