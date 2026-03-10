# code/renderer/tr_flares.c — Enhanced Analysis

## Architectural Role

Flares occupy a specialized niche in the renderer's back-end: they implement a per-view visual effect that bridges front-end tessellation (world-space geometry processing) and back-end GPU readback (post-opaque depth testing). Unlike most renderer subsystems, which are stateless per-frame, the flare system maintains cross-frame state for smooth fade timing—a deliberate tradeoff explained in the file's own documentation. This module is part of the **Renderer** subsystem's effects pipeline and interacts closely with `tr_shade.c` (tessellator), `tr_main.c` (coordinate transforms), and the shader system.

## Key Cross-References

### Incoming (who depends on this file)

- **`tr_surface.c` (inferred):** During surface tessellation, surfaces marked with the `SF_FLARE` flag invoke `RB_AddFlare` at the front-end to register flare candidates in screen-space.
- **`tr_cmds.c` or main back-end loop (inferred):** `RB_RenderFlares()` is called once per view after opaque geometry has been rendered, relying on the populated depth buffer for visibility testing.
- **`tr_init.c`:** `R_ClearFlares()` is invoked at renderer initialization or map load to reset the flare pool.
- **`RB_AddDlightFlares` (internal):** Optionally processes dynamic light sources as flare candidates (currently disabled in the call site but available infrastructure).

### Outgoing (what this file depends on)

| Subsystem | Functions/Globals | Purpose |
|-----------|-------------------|---------|
| **Coordinate Transforms** (`tr_main.c`) | `R_TransformModelToClip`, `R_TransformClipToWindow` | Project world-space flare origins into normalized device coordinates and viewport-space |
| **Tessellator** (`tr_shade.c`) | `RB_BeginSurface`, `RB_EndSurface`, `tess.*` globals | Emit screen-space quad geometry into the GPU command buffer |
| **Math** (`q_shared.c`/`q_math.c`) | `VectorCopy`, `VectorSubtract`, `VectorNormalizeFast`, `DotProduct`, `VectorScale` | 3D vector math for intensity fade based on surface normal and view angle |
| **Memory** (`qcommon`) | `Com_Memset` | Pool initialization |
| **OpenGL** (`qgl.h` wrappers) | `qglReadPixels`, `qglOrtho`, `qglPushMatrix`, `qglMatrixMode`, `qglDisable` | GPU depth readback, projection state, matrix stack manipulation |
| **Renderer Globals** | `backEnd`, `tr`, `glState` | Per-frame back-end state; current view parameters; GL state cache |
| **CVars** | `r_flares`, `r_flareSize`, `r_flareFade` | Runtime control of flare feature, size scaling, and fade speed |

## Design Patterns & Rationale

1. **Fixed-Size Object Pool with Active/Inactive Free-Lists**  
   - 128 pre-allocated `flare_t` structs; active chain tracks currently-rendered flares; inactive chain is the free-list.
   - **Why:** Predictable memory footprint; no runtime allocation/deallocation; O(1) flare reuse.
   - **Tradeoff:** Silently drops flares if the pool exhausts (line ~157: `return` when `!r_inactiveFlares`).

2. **Cross-Frame Persistent State for Fade Interpolation**  
   - Unlike most renderer state (which is ephemeral per-frame), flares retain `visible`, `fadeTime`, and `drawIntensity` across frames.
   - **Why:** Smooth in/out transitions even if a flare visibility changes abruptly (e.g., camera pans over a light). Frame-rate-independent fading via `(currentTime - fadeTime) / 1000.0`.
   - **Rationale from code comment:** "To prevent abrupt popping, the intensity of the flare is interpolated up and down as it changes visibility. This involves scene to scene state…"

3. **Dual-Phase Processing: Front-End Registration → Back-End Visibility Test**  
   - Front-end (`RB_AddFlare`): projects world-space points to screen-space, stores metadata (window coords, depth, color).
   - Back-end (`RB_TestFlare`): reads GPU depth pixel after opaque rendering; compares depth to determine occlusion.
   - **Why:** Screen-space position must come from front-end (when camera matrix is fresh); visibility test must come after opaque geometry is rasterized.

4. **GPU Readback for Visibility ("Depth Peeling")**  
   - `qglReadPixels(f->windowX, f->windowY, 1, 1, GL_DEPTH_COMPONENT, ...)` reads a single depth texel from the rendered framebuffer.
   - **Why:** Exact occlusion test; can't be done on CPU without replicating geometry rasterization.
   - **Cost:** Forces GPU pipeline flush (implicit stall); acceptable only for low flare counts (~16–32 typical).

5. **Orthographic Final Render**  
   - After visibility and fade are computed, flares are rendered as axis-aligned quads in orthographic projection, not world-space.
   - **Why:** Screen-space effects don't need perspective; simpler and faster; consistent size onscreen regardless of view distance.

## Data Flow Through This File

```
[Front-End Phase]
  Surface marked SF_FLARE encountered
    ↓
  RB_AddFlare(surface, fogNum, worldPoint, color, normal)
    ├─ R_TransformModelToClip() → eye-space and clip-space coords
    ├─ R_TransformClipToWindow() → normalized, window-space coords
    ├─ Allocate or find matching flare in active chain (keyed by surface + frameSceneNum + inPortal)
    ├─ Apply view-angle fade: d = dot(eyeToPoint, normal); color *= d
    └─ Store: windowX/Y, eyeZ, color, addedFrame

[Back-End Phase, After Opaque Render]
  RB_RenderFlares()
    ├─ Prune stale flares (not added in last frame)
    ├─ For each flare matching current scene/portal:
    │   ├─ RB_TestFlare(f)
    │   │   ├─ qglReadPixels() → reads depth at (windowX, windowY)
    │   │   ├─ Reconstructs scene-space Z via inverse projection
    │   │   ├─ Compares -eyeZ vs -screenZ; if diff < 24 units, visible = true
    │   │   ├─ Lerp drawIntensity between 0 and 1 based on fadeTime, r_flareFade
    │   │   └─ Stores result in f->drawIntensity
    │   │
    │   └─ RB_RenderFlare(f) if drawIntensity > 0
    │       ├─ Scale color by drawIntensity * identityLight
    │       ├─ Compute screen size: depends on viewport width, r_flareSize cvar, and eyeZ (proximity boost)
    │       ├─ Emit 4 vertices (quad) + 6 indices (two triangles) into tess
    │       └─ RB_BeginSurface(tr.flareShader) + RB_EndSurface() → GPU submission
    │
    └─ Restore GL state (pop matrix, re-enable clipping planes)
```

**Key State Transitions:**
- **Inactive → Active:** When a new flare surface is added and no active flare exists for that (surface, scene, portal) tuple.
- **Active → Inactive:** When not added for ≥1 frame (line ~322), or when `drawIntensity` reaches 0 after fade-out (line ~336).
- **Visible flag:** Toggles based on depth test result; controls fade-in vs fade-out direction.

## Learning Notes

**Idiomatic Q3A Renderer Patterns:**
- **Temporal coherence:** Unlike modern engines (which batch all effect types), Q3A processes flares separately in the back-end per-view, trading versatility for simplicity.
- **Manual matrix management:** `qglPushMatrix` / `qglPopMatrix` (lines ~342–347) is the fixed-function era approach; modern engines use uniform matrices or compute shaders.
- **Screen-space rendering after main pass:** Flares are rendered after the main scene with orthographic projection—efficient for 2D overlays but limits parallax/depth interaction.

**Contrast with Modern Engines:**
- Modern engines (Unreal, Unity) implement lens flares via post-process shaders (starburst via FFT, bloom via Gaussian blur), not depth readback.
- Deferred renderers can extract bright pixel clusters from the G-buffer without per-flare GPU stalls.
- Screen-space effects in modern engines often use compute shaders or framebuffer reads with less per-effect overhead.

**Game Engine Concept Connections:**
- **Visibility culling:** The depth readback is a per-object visibility determination (occlusion query equivalent, but not async).
- **Cross-frame state:** Common in particle systems and animation blending; unusual for visual effects in renderer back-ends.
- **Orthographic overlay:** Standard technique for UI and screen-space effects (HUDs, crosshairs, minimaps).

## Potential Issues

1. **GPU Stall from `qglReadPixels`**  
   - Each flare causes a pipeline flush (implicit `glFinish` equivalent). With 20+ visible flares, this can cause measurable frame-time spikes.
   - **Mitigation:** Use async readback (ARB_pixel_buffer_object) or defer visibility testing to next frame.

2. **Depth Reconstruction Formula Assumptions**  
   - Line ~288: `screenZ = backEnd.viewParms.projectionMatrix[14] / ( ( 2*depth - 1 ) * ... )`  
   - Assumes standard OpenGL projection matrix format; breaks if custom projection is used (e.g., stereoscopic rendering).
   - **24-unit threshold (line ~291)** is hardcoded; appropriate for typical FOVs (90°) but may incorrectly cull/include flares at extreme FOVs or resolutions.

3. **Portal Depth Buffer Artifact (Documented Issue)**  
   - Code comment (lines ~318–322): "The resulting artifact is that flares in mirrors or portals don't dim properly when occluded by something in the main view…"
   - Root cause: Depth buffer must be reset between portal views; can't test main-view occlusion from portal view.
   - **Not easily fixable** without architectural change (e.g., deferred visibility or separate occlusion buffers).

4. **Silent Flare Drop on Pool Exhaustion**  
   - Line ~157: If `r_inactiveFlares` is NULL, the new flare is silently discarded with no warning or fallback.
   - **Visible symptom:** Critical light sources may suddenly stop flaring if many temporary flares (e.g., explosions) exhaust the pool.

5. **Free-List Fragmentation**  
   - Over long gameplay sessions, active/inactive chains can become interleaved, reducing cache locality.
   - **Unlikely to matter** for 128 flares, but observable in profile data.
