# code/cgame/tr_types.h — Enhanced Analysis

## Architectural Role
This file defines the shared interface contract between the **cgame VM** (client-side game logic) and the **renderer module** (OpenGL backend). Every renderable frame, cgame populates `refEntity_t` instances describing what to draw and submits them via syscalls; the renderer receives these immutable structures, batches them into a depth/shader-sorted command list, and executes to OpenGL. `refdef_t` encapsulates the per-frame camera setup and scene metadata (time, area visibility, deform text). `glconfig_t` is a read-only snapshot of GPU capabilities, queried once at init and cached for lifetime. This is the public API boundary between two independently-compiled modules.

## Key Cross-References

### Incoming (who depends on this file)
- **code/cgame/** - Builds `refEntity_t` arrays per frame from server snapshots (`cg_ents.c`, `cg_players.c`); submits via `RE_AddRefEntityToScene` syscall
- **code/renderer/tr_*.c** - Receives `refEntity_t` and `refdef_t` as opaque input; implements all type dispatch, transform, and rendering logic
- **code/client/cl_cgame.c** - Routes syscalls between cgame VM and renderer `refexport_t` vtable; bridges module boundary
- **code/game/bg_public.h** references - Game VM's shared `entityState_t` definition must align with what cgame translates into `refEntity_t`
- **code/ui/** - Queries `glconfig_t` via `trap_R_GetGlconfig` for resolution and capability detection (stereo, compression, driver type)

### Outgoing (what this file depends on)
- **code/q_shared.h** - Provides `vec3_t` (3-float array), `qhandle_t` (32-bit opaque handle), `qboolean`, `byte`, string size constants, and area byte array bounds
- **Platform defines** (`Q3_VM`, `_WIN32`) - Conditionals select OpenGL driver DLL names (Win32: `"opengl32"` vs `"3dfxvgl"`; Unix: `"libGL.so.1"` vs `"libMesaVoodooGL.so"`)

## Design Patterns & Rationale

**1. Opaque Handle Abstraction**  
`qhandle_t` for shaders, models, and custom skins hides resource identity from cgame. Renderer can use any internal representation (pointer, hash, index) without cgame knowing. Classic module decoupling from the 2000s pre-reflection era.

**2. Immutable Capability Snapshot**  
`glconfig_t` is written once at renderer init (`R_Init`) and queried read-only thereafter. No runtime capability probing; prevents runtime divergence between cgame assumptions and renderer state. Fits the "frozen at startup" philosophy of early-2000s GL drivers.

**3. Bit-Flag Limits as Architecture Constraint**  
`MAX_DLIGHTS 32` exists because renderer uses a 32-bit surface bitmask to track which dlights affect each surface (`glFrontEnd.c`). `MAX_ENTITIES 1023` is constrained by drawsurf sort-key packing (entity ID occupies ~10 bits, shader occupies ~13 bits, etc.). These limits are *architectural*, not arbitrary—changing them requires renderer recompilation. Modern engines use dynamic arrays or unbounded pools to avoid this coupling.

**4. Dual-Frame Interpolation Built-In**  
`frame`/`oldframe`, `origin`/`oldorigin`, `backlerp` in `refEntity_t` embed client-side prediction interpolation directly into the submission struct. Server sends 10 Hz snapshots; client interpolates to 60+ Hz locally. This is idiomatic Quake: the entity *carries* both endpoints for the renderer to blend.

**5. Transform + Scale Flexibility**  
`axis[3]` (rotation basis) + `nonNormalizedAxes` flag is a microoptimization: if flag is false, axis is guaranteed normalized (no scale), so renderer skips renormalization in lighting calculations. If true, axis carries scale (e.g., for shrink/grow powerups), and renderer must renormalize. Typical mid-2000s space/time tradeoff.

**6. Per-Entity Lighting Origin Override**  
`lightingOrigin` + `RF_LIGHTING_ORIGIN` flag allows multi-part models (e.g., player with separate legs/torso meshes) to be lit from a single shared point, preventing seams. Without this, each part would be lit from its own origin, causing discontinuities.

## Data Flow Through This File

**Outbound (cgame → renderer per frame):**
1. cgame reads server snapshot (entity states, player state)
2. cgame translates each `entityState_t` to `refEntity_t` (type dispatch, transform, animation frame, customizations)
3. cgame calls `RE_AddRefEntityToScene(refEntity_t*)` syscall for each entity
4. Client forwards to renderer's `AddRefEntityToScene()`, which appends to the scene's entity list
5. cgame calls `RE_RenderScene(refdef_t*)` with viewpoint, FOV, time, area visibility; renderer processes entire batched list

**Renderer-side processing:**
- For each `refEntity_t`, renderer dispatches on `reType`: `RT_MODEL` → mesh transform + bind-pose or skeletal animation; `RT_SPRITE` → billboarded quad; `RT_BEAM` → extruded line; `RT_RAIL_CORE`/`RT_RAIL_RINGS` → specialized rail effect
- Renderer applies `shaderRGBA` modulation and `shaderTime` offset to all surfaces of the entity
- Renderer uses `lightingOrigin` if flag set, else uses `origin`, for dynamic light radius queries
- `backlerp` weight blends between `frame` and `oldframe` skeletal poses (or sprite frames)
- Projected shadow renders at `shadowPlane` depth if `RF_SHADOW_PLANE` set

## Learning Notes

**Idiomatic to Q3A / Early-2000s:**
- All transforms are 32-bit float. No double precision; distant objects exhibit precision loss. Modern engines use double-precision transforms for large worlds.
- Vertex shader effects don't exist; all time-based deformations (waves, tcMod, turb) are pre-computed or evaluated per-surface in the shader system (`.shader` directives), not per-vertex.
- `glHardwareType_t` enumerates specific GPU quirks (Voodoo Banshee can't interpolate alpha, RagePro can't modulate alpha on alpha textures). Modern engines query capabilities via GL extensions or query objects.
- Stereo support (`stereoFrame_t`) targets anaglyph and shutter-glass stereoscopy (2000s tech), not modern VR. Frame is rendered three times: center, left-eye, right-eye.
- **No VBO/VAO abstraction**: renderer internally manages a global `tess` (tessellator) buffer, not per-entity VBOs. All surfaces flush into one interleaved vertex/index buffer per frame. Efficient for state-change minimization on 2000s GL, but cache-inefficient on modern GPUs.

**Modern Contrast:**
- Modern engines use **hierarchical component systems** (transform trees, LOD culling) rather than flat submission of monolithic transforms.
- **Capability probing** happens at runtime (e.g., GLSL version, extension queries) rather than driver-class enums.
- **Unbounded entity counts** via dynamic arrays instead of bit-packed IDs constrained to 1023.

## Potential Issues

**Hard Architectural Limits:**  
Increasing `MAX_DLIGHTS` or `MAX_ENTITIES` requires renderer-side bit-packing redesign. Any fork that hits these limits faces non-trivial refactoring.

**No Validation Boundary:**  
`qhandle_t` are unvalidated integers. Renderer trusts cgame never passes a stale or out-of-range handle. A corrupted cgame can cause renderer crashes (null dereference in handle lookup). No handle versioning or validity bits.

**Float Precision at Distance:**  
32-bit floats lose precision beyond ~100k units from origin. Large outdoor maps exhibit vertex snapping or Z-fighting at horizon. Modern engines use origin shifting or double precision.

**Implicit Time Dependency:**  
`shaderTime` and `refdef_t.time` are floating-point wall-clock times, not ticks. Frame-rate-dependent effects if not carefully authored. Modern engines use discrete ticks or delta-time for determinism.
