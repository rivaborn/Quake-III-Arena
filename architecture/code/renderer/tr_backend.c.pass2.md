# code/renderer/tr_backend.c — Enhanced Analysis

## Architectural Role

`tr_backend.c` is the sole consumer of the renderer's double-buffered command queue, sitting at the boundary between the CPU-side front end (scene traversal, sort, culling in `tr_main.c`, `tr_world.c`, `tr_scene.c`) and the GPU via OpenGL. It is the only file in the renderer that issues `qgl*` draw-state calls directly in bulk; all other renderer files either feed the command queue or operate on the `tess` tessellator buffer that this file flushes. In SMP mode it runs entirely on a dedicated render thread, making it the engine's one concurrency boundary inside the renderer DLL. It also owns the 2D/3D projection switching that allows the same frame to contain both a full 3D world view and orthographic UI/cinematic overlays.

---

## Key Cross-References

### Incoming (who depends on this file)

- **`tr_init.c`** — allocates `backEndData[SMP_FRAMES]` from the hunk and spawns `RB_RenderThread` via `GLimp_SpawnRenderThread`; sets `backEnd.smpFrame` before first use.
- **`tr_cmds.c`** — the only producer of render commands; appends `RC_SET_COLOR`, `RC_STRETCH_PIC`, `RC_DRAW_SURFS`, `RC_DRAW_BUFFER`, `RC_SWAP_BUFFERS`, `RC_SCREENSHOT` into `backEndData[n]->commands`.
- **`tr_main.c`** — calls `RB_ExecuteRenderCommands` (synchronous path) or signals the render thread; reads `backEnd.pc` perf counters for `r_speeds` reporting.
- **`code/client/cl_cin.c`** — calls `RE_StretchRaw` and `RE_UploadCinematic` (defined here) through the `refexport_t` vtable to upload decoded RoQ video frames to `tr.scratchImage[]`.
- **Platform GL layer** (`win32/win_glimp.c`, `unix/linux_glimp.c`) — calls `RB_RenderThread` as the thread entry point in SMP mode; drives `GLimp_RendererSleep`/`GLimp_WakeRenderer` handshaking.
- **`GL_Bind`, `GL_State`, `GL_Cull`, `GL_TexEnv`, `GL_SelectTexture`** are called pervasively from `tr_shade.c`, `tr_sky.c`, `tr_surface.c`, `tr_shadows.c`, `tr_flares.c` — this file is the shared GL-state-management library for the entire renderer.

### Outgoing (what this file depends on)

- **`tr_shade.c`** — `RB_BeginSurface`, `RB_EndSurface`, `RB_CheckOverflow`: the tessellator sink that accumulates vertex/index data.
- **`tr_surface.c`** — `rb_surfaceTable[]`: the surface-type dispatch table invoked per draw surface inside `RB_RenderDrawSurfList`.
- **`tr_main.c`** — `R_DecomposeSort`, `R_RotateForEntity`, `R_TransformDlights`: sort-key unpacking and per-entity matrix math.
- **`tr_shadows.c`** — `RB_ShadowFinish`: stencil shadow pass invoked at the end of the draw-surface loop.
- **`tr_flares.c`** — `RB_RenderFlares`: lens flare pass appended after all surfaces.
- **`tr_image.c`** — `RB_TakeScreenshotCmd`: screenshot capture invoked by `RC_SCREENSHOT`.
- **`GLimp_*`** — `GLimp_EndFrame` (buffer swap), `GLimp_LogComment` (GL call logging), `GLimp_RendererSleep`/`GLimp_FrontEndSleep`/`GLimp_WakeRenderer` (SMP synchronization).
- **`ri` import table** — `ri.Milliseconds` for `backEnd.pc.msec`; `ri.Printf`/`ri.Error` for diagnostics; `ri.Hunk_Alloc` indirectly via `tr_init.c`.
- **Globals read:** `tr` (`trGlobals_t`), `glConfig`, `glState`, `tess` (`shaderCommands_t`), `r_nobind`, `r_finish`, `r_fastsky`, `r_measureOverdraw`, `r_shadows`, `r_speeds`.

---

## Design Patterns & Rationale

**Producer/consumer command queue.** The front end writes typed render commands into `backEndData[n]->commands` and the back end walks the buffer linearly. This decouples scene traversal from GL submission and is the prerequisite for SMP — the two phases can operate on different frames simultaneously.

**Diff-based state cache.** `GL_State` XORs incoming `stateBits` against `glState.glStateBits` and only calls `qgl*` for bits that changed. This was essential in the late 1990s when `glEnable`/`glBlendFunc` carried measurable overhead. The pattern is the conceptual ancestor of modern Pipeline State Objects (PSOs) in D3D12/Vulkan, but implemented manually in software.

**Double buffering for SMP.** `backEndData[SMP_FRAMES]` (where `SMP_FRAMES == 2`) lets the front end write frame N+1 while the back end renders frame N. `backEnd.smpFrame` indexes the active frame on the render thread. The `volatile renderThreadActive` flag is the only shared mutable state between threads; the actual synchronization is delegated to `GLimp_*` condition variables.

**Surface-type dispatch table.** `rb_surfaceTable[SF_*]` (defined in `tr_surface.c`, indexed here) is a virtual method table indexed by surface type integer, avoiding any switch/if chain in the hot draw loop.

**Tradeoff — two texture units only.** `GL_SelectTexture` hard-errors on unit > 1. This was a deliberate constraint matching the minimum multitexture spec; more units would require a more complex combiner abstraction than Q3 needed.

---

## Data Flow Through This File

```
Front end (tr_scene.c / tr_world.c)
  → backEndData[n]->commands[]          (RC_DRAW_SURFS written by tr_cmds.c)
    → RB_ExecuteRenderCommands()        (dispatches by command ID)
      → RB_DrawSurfs()
        → RB_BeginDrawingView()         (projection matrix, viewport, clear, portal clip plane)
        → RB_RenderDrawSurfList()       (iterate sorted drawSurf_t array)
          → R_DecomposeSort()           (unpack shader/fog/entity/dlight from 64-bit key)
          → R_RotateForEntity()         (compute per-entity model-view matrix)
          → qglLoadMatrixf()            (upload to GL)
          → rb_surfaceTable[type]()     (tessellate into tess buffer)
          → RB_EndSurface()             (flush tess → GL draw call)
        → RB_ShadowFinish()
        → RB_RenderFlares()
      → RB_SwapBuffers()
        → GLimp_EndFrame()              (OS buffer swap)
```

Cinematic path: `cl_cin.c` → `RE_StretchRaw` / `RE_UploadCinematic` → `qglTexImage2D`/`qglTexSubImage2D` on `tr.scratchImage[client]` → later consumed via a `RC_STRETCH_PIC` command.

**Key state transitions:**
- `backEnd.projection2D`: false after `RB_BeginDrawingView`, true after `RB_SetGL2D`. Controls whether matrices are perspective or orthographic.
- `glState.faceCulling = -1`: force-dirtied at the start of each 3D view to guarantee first-surface cull state is applied even if it matches the last frame's cached value.
- `backEnd.skyRenderedThisView`: cleared per view; set by sky surface render; checked by sun/flare code.

---

## Learning Notes

- **Coordinate flip matrix.** Q3's world space looks down +X; OpenGL's NDC looks down −Z. `s_flipMatrix` performs this rotation. It is applied during portal clip-plane setup (plane transformed into OpenGL eye space). Modern engines either adopt GL conventions directly or handle this in the shader.

- **Depth range hack.** View-model (weapon) entities render with `qglDepthRange(0, 0.3)` to prevent z-fighting against the world. The depth buffer is effectively partitioned: [0, 0.3] for weapons, [0, 1] for everything else. This is still common today.

- **Alpha test via fixed-function pipeline.** `GL_State` calls `qglAlphaFunc`. Modern engines discard in the fragment shader or use coverage-to-alpha. Q3 predates programmable pipelines on mainstream hardware.

- **No scene graph or ECS.** Entities are flat `trRefEntity_t` arrays indexed by integer. The back end just iterates, no hierarchy traversal. Modern engines use ECS or hierarchical scene graphs for culling and transform inheritance.

- **`r_nobind` for GPU profiling.** Redirecting all texture binds to a single 1×1 dlight image eliminates texture-bandwidth variance, isolating geometry throughput — a technique still used in GPU profiling today, though now handled by GPU profiler tools.

- **SMP was ahead of its time.** Quake III's optional render thread predates most game engines' adoption of multithreaded rendering by nearly a decade. The design is limited (2-frame ring, only two threads), but architecturally sound as a producer/consumer separation.

---

## Potential Issues

- **`GL_Bind` accesses `image->frameUsed` even when `image` is NULL (falls back to `tr.defaultImage`) but the NULL guard returns early before the `frameUsed` update** — the `frameUsed` line executes on the non-NULL branch only; however, if `glState.currenttextures[glState.currenttmu] == texnum`, the bind is skipped and `frameUsed` is never updated, meaning LRU eviction (`image->frameUsed`) can mark an actively displayed image as stale if it happens to have an already-cached texnum.

- **`GL_SelectTexture` is hardcoded to 2 units.** Any future multitexture extension beyond TMU 1 silently errors. Not a bug in Q3's context but a porting pitfall.

- **`renderThreadActive` is `volatile` but not protected by a memory barrier.** On weakly-ordered architectures (e.g., ARM), the write on the render thread may not be visible to the front-end thread without explicit synchronization. Fine for x86 in 1999; would require `atomic_bool` under C11.
