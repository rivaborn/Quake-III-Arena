# code/botlib/be_aas_debug.h — Enhanced Analysis

## Architectural Role

This header exposes the **debug visualization interface for AAS spatial and navigation structures**. While not part of the botlib public API (`botlib_export_t`), these functions serve as introspection tools for developers debugging the navigation graph, pathfinding, and spatial decomposition. They accumulate debug geometry into internal lists managed by `be_aas_debug.c`, which in turn delegates actual rendering to engine callbacks (`botimport.debug*`) registered via the `botlib_import_t` vtable—a pattern that isolates botlib's debug output from rendering specifics.

## Key Cross-References

### Incoming (who depends on this file)
- **`be_aas_debug.c`** — implements all declared functions; manages internal debug line/polygon queues
- **`be_aas_reach.c`** / **`be_aas_route.c`** — call visualization functions (inferred from reachability/routing context) to debug pathfinding
- **`be_aas_move.c`** / **`be_aas_sample.c`** — likely call geometry visualization during movement prediction or area sampling
- **`be_interface.c`** — may expose subset of debug functions through botlib console commands (not visible in this header but plausible given botlib architecture)

### Outgoing (what this file depends on)
- **`q_shared.h`** — `vec3_t` type definitions
- **`be_aas_def.h`** / **`be_aas_reach.h`** — `aas_reachability_s` struct definition (forward-declared here)
- **`botlib_import_t` callbacks** — `botimport.debug*` functions (not declared in this header but called by `be_aas_debug.c`) for actual rendering

## Design Patterns & Rationale

**Deferred debug rendering via callback accumulation:**
Functions like `AAS_DebugLine` and `AAS_PermanentLine` do not immediately invoke rendering. Instead, they accumulate geometry into internal lists, allowing multiple visualization calls to batch together. `AAS_ClearShownDebugLines` / `AAS_ClearShownPolygons` manage lifecycle—a pattern that avoids immediate GL calls from deep within pathfinding code and isolates botlib from rendering specifics.

**Dual transient/permanent geometry streams:**
The header distinguishes between temporary debug lines (cleared each frame/interval) and permanent lines (persistent across clears). This mirrors real-time visualization workflows where developers want to inspect both instantaneous state and longer-term traces.

**Visualization by decomposition layer:**
Functions cluster into three categories:
1. **Primitive rendering** (`AAS_DebugLine`, `AAS_DrawCross`, `AAS_DrawArrow`) — low-level geometry
2. **Data structure visualization** (`AAS_ShowFace`, `AAS_ShowArea`, `AAS_ShowBoundingBox`) — mid-level AAS spatial hierarchy
3. **Navigation graph visualization** (`AAS_ShowReachability`, `AAS_ShowReachableAreas`, `AAS_DrawPlaneCross`) — high-level pathfinding structure

This layering allows different subsystems to visualize at appropriate abstraction levels.

## Data Flow Through This File

```
Developer code / Debug console → AAS_ShowReachability / AAS_ShowArea / etc.
                                ↓
                        be_aas_debug.c (implementation)
                                ↓
                    Accumulate to internal debug lists
                                ↓
                    botimport.debug* callbacks
                                ↓
                        Engine renderer
```

**State transitions:**
- **Transient**: `AAS_DebugLine` → internal list → `AAS_ClearShownDebugLines` clears
- **Persistent**: `AAS_PermanentLine` → separate permanent list → survives clears

## Learning Notes

**Idiomatic to this era (pre-modern debug UI):**
This is a low-level visualization layer typical of late-1990s/early-2000s game engines. Modern engines use in-process debug renderers or separate debug visualization tools; Quake 3 bakes debug drawing into the engine's GL command stream via callbacks. The separation of "what to draw" (botlib) from "how to draw" (engine callbacks) is a clean architectural boundary.

**Inverse dependency pattern:**
Notably, botlib does *not* depend on the renderer or even directly on GL. Instead, the engine provides a `botlib_import_t` vtable with debug callbacks, making botlib a true dependency-inverse layer. This is a precursor to plugin architectures.

**Reachability and navigation debugging:**
The concentration of functions around `aas_reachability_s` (`AAS_ShowReachability`, `AAS_ShowReachableAreas`, `AAS_DrawArrow`) reveals pathfinding as the central concern of botlib's introspection. Developers could visualize the entire navigation graph at runtime—essential for tuning bot traversability and AI goal selection.

**No ECS or modern scene graph:**
Unlike modern engines, this is pure C with explicit struct pointers and integer indices (face/area numbers). Debugging is manual, explicit function calls rather than component queries or retained-mode drawing systems.

## Potential Issues

- **Typo in header comment** (line 52): "draw a cros" should be "draw a cross"  
- **No error handling**: Functions return `void` and do not report failure if internal lists overflow. Silent failures could cause missed debug visualization without warning.  
- **Color parameter semantics unclear**: `int color` is passed throughout but never documented—likely a palette index into the engine's debug color table, but consumers of this header cannot know without reading `be_aas_debug.c`.
