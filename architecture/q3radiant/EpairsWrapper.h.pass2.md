# q3radiant/EpairsWrapper.h — Enhanced Analysis

## Architectural Role

This file implements a **reference-counted virtual proxy** for BSP entity properties in the Q3Radiant level editor's plugin extension system. It bridges the editor's internal `entity_t` representation with a standardized, managed interface (`IEpair`) that can be safely shared with plugins. The reference-counting pattern suggests it's used in contexts where ownership is distributed or temporary, typical of scripting/plugin sandboxing where the editor must revoke entity access when entities are freed.

## Key Cross-References

### Incoming (who depends on this file)
- **Plugin system** (`q3radiant/PlugIn*`) — plugins receive `IEpair` pointers to safely manipulate entity properties without direct `entity_t` access
- **Editor logic** in `q3radiant/` — interior code creates `CEpairsWrapper` instances to provide managed access to entities
- **Entity dialogs** — likely used by entity property UI panels to batch-edit epairs

### Outgoing (what this file depends on)
- **`IEpair` interface** (`q3radiant/IEpairs.h`) — abstract base class defining the public contract
- **`entity_t` type** (from Q3Radiant's internal entity representation, likely in `ENTITY.H` or similar) — raw entity structure holding key-value pairs
- **Math/utility functions** called by virtual methods (e.g., `GetVectorForKey` must parse BSP epair strings into 3D vectors)

## Design Patterns & Rationale

**Reference-Counted Proxy (Vtable + refCount)**
- Decouples plugin code from internal `entity_t` layout
- `IncRef()`/`DecRef()` pattern allows safe shared ownership — plugins increment when storing pointers, decrement when releasing; destructor fires when count hits zero
- Typical of **COM-style** or **Lua-style** C++ interfaces where plugins are untrusted or dynamically loaded

**Virtual Interface**
- All entity property access flows through virtual methods, enabling substitution or logging at call time (common in editors for undo/redo integration)
- Prevents plugins from directly mutating `entity_t`, enforcing validation or change tracking

## Data Flow Through This File

1. **Creation:** Editor code instantiates `CEpairsWrapper(entity_t* ep)` when passing entity to plugin
2. **Read-path:** Plugin calls `ValueForKey("key")` → method interprets BSP epair string → returns value
3. **Write-path:** Plugin calls `SetKeyValue("key", "value")` → method updates underlying `entity_t` → likely triggers editor change notifications (undo, redraw)
4. **Spatial queries:** `GetVectorForKey("origin")`, `GetEntityOrigin()`, `CalculateRotatedBounds()` extract positional data for 3D operations
5. **Lifetime:** When plugin reference count drops to zero, `delete this` called; entity remains valid (wrapper was merely proxy)

## Learning Notes

**Editor-Centric Patterns:**
- Unlike the runtime engine (which is data-oriented and immutable), the level editor uses **object-oriented, mutable proxy patterns** to safely bridge external code (plugins) with internal state
- The `entity_t` is likely a mutable struct (unlike runtime `entityState_t` snapshots); this wrapper enforces change isolation

**Plugin Safety:**
- No plugins can corrupt `entity_t` memory by pointer casting; all access is method-gated
- Reference counting enables the editor to revoke access when entities are deleted: wrapper becomes a dangling reference (safe no-op) rather than a use-after-free vulnerability

**BSP Entity Representation:**
- Epairs are key-value string pairs embedded in BSP entity data (raw Q3 engine format)
- Methods like `ValueForKey()`, `GetVectorForKey()`, `FloatForKey()` are **adapters** converting from editor's internal representation to common query types (idiomatic to Q3 map format, where all entity data is stored as strings)

**Rotated Bounds:**
- `CalculateRotatedBounds()` suggests entities can be rotated; this is a **rich 3D transform** feature typical of modern editors (not the runtime engine, which uses fixed-axis rotation angles). The wrapper makes this spatial math available to plugins.

## Potential Issues

- **Missing `const`:** Methods like `ValueForKey()` are non-const but appear to be read-only; should likely be `const` for clearer API semantics.
- **No thread-safety:** Reference counting is not atomic; if plugins run on separate threads (unlikely in Q3Radiant but possible in extended versions), concurrent `IncRef()`/`DecRef()` would race.
- **Implicit dependency:** The `entity_t*` pointer is never validated; if an entity is deleted behind the wrapper's back, subsequent virtual calls will dereference invalid memory. Editor must enforce external lifetime guarantees (e.g., wrapper lifetime ⊆ entity lifetime).
