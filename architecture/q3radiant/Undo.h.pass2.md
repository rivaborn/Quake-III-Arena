# q3radiant/Undo.h — Enhanced Analysis

## Architectural Role
This file implements the undo/redo infrastructure for **Q3 Radiant** (the offline level editor), not the runtime engine. It provides a transaction-scoped command capture API that allows map editing operations (brush creation, entity placement, etc.) to be recorded, reversed, and replayed. The file defines a public API consumed by all brush and entity manipulation routines throughout the editor UI layer (`q3radiant/*.cpp`).

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant UI layer** (`Brush.cpp`, `Entity.cpp`, `Select.cpp`, `Drag.cpp`, etc.) — calls `Undo_Start/End` to bracket edit operations
- **Editor command handlers** in `MainFrm.cpp` / `Win_main.cpp` — triggers `Undo_Undo()` / `Undo_Redo()` on user input
- **Map I/O** (`Map.cpp`) — may clear undo history on load via `Undo_Clear()`
- **Preferences/options** (`PrefsDlg.cpp`) — configures memory limits via `Undo_SetMaxSize()` / `Undo_SetMaxMemorySize()`

### Outgoing (what this file depends on)
- **Brush type** (`brush_t` from `Brush.h`) — parameter to capture/restore brush state
- **Entity type** (`entity_t` from `Entity.h`) — parameter to capture/restore entity state
- No documented dependencies on runtime engine; exists purely as an offline editor utility

## Design Patterns & Rationale

**Command Pattern (implicit):** Each `Undo_Start()` / `Undo_End()` pair brackets a logical command. The Add/End pairs suggest a before-state snapshot on Add, after-state snapshot on End, enabling forward/backward replay.

**Two-Phase Capture:**
- `Undo_AddBrush()` → captures brush before modification
- `Undo_EndBrush()` → captures brush after modification (or records operation completion)
- Allows the undo system to store deltas or full snapshots without exposing implementation

**Bounded Circular Buffers:** The size/memory-limit functions hint at LRU eviction when the undo stack exceeds capacity — common in long editing sessions on large maps.

**Separate Brush/Entity Tracks:** Distinct capture APIs suggest different serialization or undo strategies (e.g., entities may have runtime-dependent state that doesn't undo the same way as geometry).

## Data Flow Through This File

1. **Undo Capture Phase:**
   - User edits in viewport (drag brush, rotate entity, etc.)
   - Edit handler calls `Undo_Start("operation_name")`
   - Handler calls `Undo_AddBrush(old_brush)` or `Undo_AddEntity(old_entity)` *before* modification
   - Handler modifies the brush/entity in-place
   - Handler calls `Undo_EndBrush(new_brush)` or `Undo_EndEntity(new_entity)` *after* modification
   - Handler calls `Undo_End()`

2. **Undo/Redo Execution:**
   - User presses Ctrl+Z → `Undo_Undo()` replays the recorded state reversal
   - User presses Ctrl+Y → `Undo_Redo()` replays the next forward command
   - Both trigger viewport refresh (implementation detail, not visible here)

3. **Memory Management:**
   - `Undo_MemorySize()` tracks footprint; once it exceeds `Undo_GetMaxMemorySize()`, oldest operations are evicted
   - Operation count is also bounded by `Undo_GetMaxSize()` (default 64)

## Learning Notes

**What's Idiomatic to This Era:**
- The two-phase Add/End pattern is typical of pre-modern undo libraries (c.f. Adobe Photoshop's history).
- Modern editors (VS Code, Sublime) often use immutable command objects and functional composition; this file hints at a more imperative, stateful approach.
- No undo grouping or nesting mechanism visible — complex multi-brush operations must wrap all brushes in a single `Undo_Start/End` pair.

**Editor Architecture Insight:**
- Unlike the **runtime engine** (which is replay-based: snapshots + delta-compressed client-server updates), the editor uses traditional **undo stacks** with explicit before/after capture.
- The separation from runtime suggests Radiant was built as a standalone tool (Visual C++ MFC / Win32 native), not integrated with engine VM or networked systems.

**Developer Learning:**
- How to design a bounded, transaction-scoped undo system for a geometry editor
- Trade-offs: storing full object copies (fast access, high memory) vs. storing deltas (less memory, slower replay)
- The API hides implementation details, allowing the backend to switch between snapshot/delta strategies without breaking callers

## Potential Issues

**No Visible Group/Nesting:**
- If a single command needs to undo multiple independent changes (e.g., "delete 3 brushes, spawn 1 entity"), there's no explicit API to group them. All three Undo_Add calls must share the same Undo_Start/End pair — if nesting is needed, there's silent corruption risk.

**No Error Recovery or Rollback:**
- If an `Undo_AddBrush` fails mid-operation, the `Undo_End()` still finalizes, potentially leaving the undo buffer in an inconsistent state.

**Memory Limit Strategy Unspecified:**
- Whether eviction is FIFO, LRU, or weighted by operation type is not documented here. Large maps could thrash the buffer unexpectedly.

---
