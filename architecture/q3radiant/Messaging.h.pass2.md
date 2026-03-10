# q3radiant/Messaging.h — Enhanced Analysis

## Architectural Role

This header defines the client-side implementation wrapper for the XY viewport messaging/snapping subsystem in Q3Radiant. The `CXYWndWrapper` class adapts the XY window (top-down 2D editor view) interface (`IXYWndWrapper`) to provide grid-snapping services for brush editing. While minimal in scope, it bridges the generic viewport abstraction with concrete grid-snapping logic, enabling the editor's core interaction model: constraint-based placement and alignment of geometry.

## Key Cross-References

### Incoming (who depends on this file)
- `q3radiant/Messaging.cpp` — Implementation file; likely calls `SnapToGrid` from event handlers
- `q3radiant/XYWnd.h` and `q3radiant/XYWnd.cpp` — The XY viewport window class; may use or compose `CXYWndWrapper`
- UI event handlers in mainframe/dialog code that respond to user drag/placement operations

### Outgoing (what this file depends on)
- `IXYWndWrapper` — Abstract interface (likely defined elsewhere in q3radiant); defines the contract for viewport messaging
- `vec3_t` — Shared 3D vector type from `code/game/q_shared.h` (imported indirectly); the grid-snapped point is emitted as a 3D world coordinate

## Design Patterns & Rationale

**Adapter/Wrapper Pattern**: `CXYWndWrapper` inherits from `IXYWndWrapper` to fulfill a contractual interface while delegating grid-snapping logic to `SnapToGrid`. This allows the editor's command/messaging system to treat viewport operations uniformly regardless of which window (XY, Z profile, camera) is active.

**Minimal Coupling**: By defining only a single public method, the wrapper keeps the interface surface small. This reduces the number of edit operations that must be aware of grid constraints—only the XY window (which operates in 2D) needs explicit snapping; the Z and camera views inherit differently.

## Data Flow Through This File

1. **Input**: Screen-space mouse coordinates (`x1, y1`; likely in 640×480 or editor-unit scale) from a drag/placement event
2. **Transformation**: `SnapToGrid` quantizes and rounds to the nearest grid unit (typically configurable in Q3Radiant, e.g., 8/16/32 units)
3. **Output**: World-space 3D point (`pt` as `vec3_t`) snapped to grid; passed to brush manipulation or entity placement logic

This is a **critical UX bottleneck**: every brush movement or entity placement in the XY view flows through this snapping function, ensuring visual/logical alignment with the grid.

## Learning Notes

- **Editor ≠ Engine**: Unlike the runtime engine (`code/`, which is platform-abstracted, VM-hosted, and streaming-oriented), Q3Radiant is a **stateful GUI application** with synchronous event dispatch. The "messaging" terminology reflects MFC-style (Microsoft Foundation Classes) message passing, not network packets or command queues.

- **Grid as First-Class Concept**: Modern engines often treat snapping as a per-tool parameter; Q3Radiant bakes it into the viewport wrapper, reflecting 1990s/2000s modeling philosophy where grid is mandatory for BSP-based geometry.

- **Interface Segregation**: The `IXYWndWrapper` abstraction suggests the editor anticipated multiple viewport types. Only the XY window overrides snapping (likely Z and camera views inherit a no-op or different snapping strategy).

- **Shared Types Across Boundary**: Use of `vec3_t` from the game VM's `q_shared.h` in the editor shows **no hard separation** between tool and engine code; both share coordinate systems, enabling direct drag-and-drop of brushes into game coordinate space.

## Potential Issues

- **No validation** of input coordinates or grid size (division by zero if grid is 0?) — assumes caller ensures valid state
- **Static/global grid state** likely; this wrapper doesn't store grid size, suggesting it reads from global config (e.g., `g_grid` or similar)
- **3D point from 2D input**: The method accepts only 2D (x1, y1) but outputs 3D vec3_t. The Z component must come from elsewhere (current brush height, selection center, or implicit camera plane) — interface doesn't clarify this contract
