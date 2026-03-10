# q3radiant/IEpairs.cpp — Enhanced Analysis

## Architectural Role

This file provides a plugin-facing abstraction layer for manipulating entity key-value pairs (epairs) in the Q3Radiant level editor. It implements `CEpairsWrapper`, which bridges plugins and the editor's internal entity model. By delegating to global functions rather than exposing raw `entity_t` structures, it allows the editor to evolve its internal representation while maintaining plugin stability—a key design pattern for tools with extensible plugin systems.

## Key Cross-References

### Incoming (who depends on this file)
- **Plugin system** (via `IEpairs.h` interface): Any loaded plugin needing to query or modify entity properties calls these methods
- **Editor core entity dialogs**: Likely uses this wrapper to populate/update property panels in the UI
- **IMessaging/IPluginEntities subsystem**: Probably instantiates `CEpairsWrapper` to hand to plugins

### Outgoing (what this file depends on)
- **Global entity accessor functions** (`::GetVectorForKey`, `::FloatForKey`, `::ValueForKey`, `::SetKeyValue`): Defined elsewhere in radiant (likely `entity.cpp` or similar), providing single point of truth for BSP entity data
- **Radiant entity model** (`entity_t` struct): The underlying in-memory BSP entity representation
- **Shared math library** (`q_shared.c` / `q_math.c`): `VectorCopy`, `AngleVectors`, `ClearBounds`, `AddPointToBounds`, `DotProduct`

## Design Patterns & Rationale

1. **Adapter Pattern**: `CEpairsWrapper` wraps `entity_t` with a clean, type-aware interface instead of exposing raw pointer access. This decouples plugins from internal struct details.

2. **Delegation**: All key-value access routes through global functions (`::GetVectorForKey`), not direct struct field access. This allows the editor to:
   - Maintain consistency across all property access paths
   - Add caching/validation later without breaking plugins
   - Synchronize changes with the undo system or viewport

3. **Rotation Matrix Construction**: `CalculateRotatedBounds` follows a classic 3D editor pattern:
   - Normalize Euler angles to 0–359 range (editor cleanup)
   - Handle special Quake pitch values (-1 = -90°, -2 = +90°)
   - Decompose angles into orthonormal basis vectors
   - Rotate all 8 corners of the original AABB
   - Recompute AABB in rotated space (no longer axis-aligned, but its bounding volume is)

This is necessary because the editor stores bounding boxes axis-aligned, but must display/select entities with arbitrary yaw/pitch/roll.

## Data Flow Through This File

1. **Plugin initialization**: Editor creates `CEpairsWrapper(entity_t*)` and passes to plugin
2. **Query phase**: Plugin calls `ValueForKey`, `FloatForKey`, `GetVectorForKey` to read entity properties (e.g., "origin", "angle", "classname")
3. **Modification phase**: Plugin calls `SetKeyValue` to alter properties; each call goes through global `::SetKeyValue`, likely triggering undo registration
4. **Bounds recalculation**: If entity has rotation, `CalculateRotatedBounds` is called to update viewport visualization

## Learning Notes

- **Editor-specific rotation handling**: The special cases (`angle == -1`, `angle == -2`) are Quake engine conventions; the editor must mirror them for correct visualization. This tight coupling shows why editors and engines often share math code.
- **Angle normalization before storage**: The `changed` flag pattern suggests the editor normalizes angles on read, ensuring consistency. This is defensive against malformed BSP files.
- **2D-to-3D bounding volume**: Q3A uses 2D bounding boxes on disk but editors need 3D frustum culling and collision; the 8-corner rotation trick is the standard solution.
- **Idiomatic to the era**: Direct global function calls and manual 3x3 matrix construction reflect late-1990s C++ tool patterns before ubiquitous linear algebra libraries.

## Potential Issues

1. **Unsafe `sprintf` at line 130**: No buffer bounds check on `tempangles[128]`. If angle values produce output >128 bytes, this overflows. (Not critical given expected int range, but poor practice.)
2. **Silent angle "cleanup"**: Normalizing angles and writing them back mid-read could surprise plugins expecting round-trip consistency (write then read). No warning or callback signals this.
3. **Rotation matrix correctness**: The `trans[i][j]` matrix construction is hand-coded; no formal verification that forward/right/up are being placed in the right order. The layout must match `DotProduct` semantics downstream.
4. **No error handling**: `::GetVectorForKey`, `::FloatForKey`, etc. could fail silently if the entity or key doesn't exist. The wrapper provides no validation or fallback.
