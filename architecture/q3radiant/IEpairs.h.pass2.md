# q3radiant/IEpairs.h — Enhanced Analysis

## Architectural Role

This header defines a pure virtual C++ interface (`IEpair`) that abstracts entity property access for Q3Radiant plugins. Entity key-value pairs ("epairs") are the fundamental data structure through which all Quake entity metadata flows—from map authorship through game logic. By exposing this interface via a stable vtable, the editor allows third-party plugins to read/write entity properties without exposing internal editor data structures or coupling plugins to the level-editor binary's private implementation.

## Key Cross-References

### Incoming (who depends on this file)
- **IEpairs.cpp** (`q3radiant/IEpairs.cpp`): The concrete implementation backing this virtual interface; likely wraps the editor's internal entity representation (see `entity_s` structures in `ENTITY.H`).
- **Plugin interfaces** (`IPluginEntities.h`, `PlugIn.cpp`, `PlugInManager.cpp`): Plugins receive `IEpair*` pointers when querying selected or targeted entities; the interface decouples plugin code from editor internals.
- **Entity dialogs and property editors** (e.g., `DialogInfo.cpp`, `EntityListDlg.cpp`): The editor's own UI uses `IEpair` to display and modify selected entities.

### Outgoing (what this file depends on)
- **Quake math types** (`vec3_t` from `q_shared.h`, inherited into q3radiant's common header chain): Vector and bounds representation.
- **No runtime engine calls**: This is a pure interface definition; no dependencies on renderer, server, or collision subsystems.

## Design Patterns & Rationale

**Virtual Interface / Abstract Base Class**: The `IEpair` class is a pure virtual interface with no implementation. This decouples:
- Plugins from the editor's entity representation details
- Multiple potential backend implementations (could support different entity data stores)
- ABI stability: plugins can be recompiled without relinking against the editor executable if the vtable order is preserved

**Reference Counting** (`IncRef`/`DecRef`): Classic COM-style lifetime management. Plugins increment the ref count when obtaining a pointer, decrement when done. The editor retains ownership and destroys the backing object when all external references drop to zero. This prevents use-after-free and simplifies memory ownership across the plugin/editor boundary.

**Key-Value Accessors**: The interface mirrors how Quake entities are fundamentally structured in BSP entity strings and the game VM (`epair_t` chains in `g_local.h`). Methods like `ValueForKey`, `FloatForKey`, `GetVectorForKey` directly map to the entity property lookups used throughout the engine (`G_SpawnString`, trap calls in cgame, etc.).

## Data Flow Through This File

1. **Plugin obtains interface**: Editor creates an `IEpair` instance (via concrete `IEpairs.cpp` implementation) wrapping a selected or queried entity.
2. **Property queries**: Plugin calls `ValueForKey("classname")`, `GetVectorForKey("origin")`, etc. to read entity configuration.
3. **Property mutations**: Plugin calls `SetKeyValue("targetname", "new_name")` to modify the entity. Underlying editor UI and undo/redo system likely observe/record changes.
4. **Computed properties**: `CalculateRotatedBounds` and `GetEntityOrigin` are computed accessors that synthesize derived values (e.g., rotating bounding box per entity's `angle` or `angles` keys—standard Quake entity orientation).
5. **Cleanup**: Plugin calls `DecRef` when done. Editor destroys backing `IEpair` when count reaches zero.

## Learning Notes

**Idiomatic Quake Entity Representation**: This interface encodes how Quake maps and entities work:
- All entity data is **key-value pairs** (strings) parsed from BSP entity lumps and synchronized to clients via configstrings.
- **Angles** (`angle` scalar or `angles` vector) control rotation, and derived geometry (bounding boxes, model offsets) must rotate accordingly—see `CalculateRotatedBounds`.
- **Origin** is the entity's world position; special handling for origin may apply (e.g., movers, relative to parent, etc.).

**Plugin Architecture Pattern**: Q3Radiant's plugin system (visible here via IEpairs and sibling interfaces in `IPluginEntities.h`, `IShaders.h`, etc.) is a **facade** pattern: multiple virtual interfaces provide a stable, versioned API that hides the editor's internal architecture. This is similar to how the Quake engine itself uses `refexport_t` (renderer), `cgame_export_t` (cgame VM), etc.—**vtable-based capability negotiation** is a recurring pattern in this codebase.

**Modern Comparison**: ECS engines expose components; traditional game editors expose scene graphs or property inspectors. Quake's entity system is **property-centric**: entities are bags of properties, behaviors emerge from composition of properties (e.g., `"type" "trigger_hurt"` + `"damage" "10"` defines behavior). The `IEpair` interface directly reflects this philosophy.

## Potential Issues

**No type safety**: All properties are stored/retrieved as strings (`ValueForKey` returns `char*`). Type conversions are caller's responsibility; typos in keys silently return defaults (empty string, 0.0f, zero vector). Modern type-safe property systems prevent this.

**No change notification**: The interface doesn't signal when a property changes. If multiple subsystems read the same entity, they may cache stale values. The concrete implementation likely uses an observer pattern internally, but it's not expressed in this interface.

**Incomplete rotation representation**: `CalculateRotatedBounds` computes axis-aligned bounds from rotated geometry. For editor visualization and collision checks during placement, this suffices, but **Quake's runtime uses angle + bounding box directly** (no separate rotation matrix in entity_t); the editor must consistently apply this same rotation logic or preview vs. runtime behavior will diverge.
