# code/renderer/tr_public.h — Enhanced Analysis

## Architectural Role

This file defines the **DLL boundary** between the engine's client/core layer and the swappable OpenGL renderer module. It enables the renderer to be loaded, initialized, and orchestrated entirely through versioned vtable indirection (`GetRefAPI`), decoupling rendering from the engine's core event loop and memory infrastructure. The renderer consumes low-level services from `qcommon` (filesystem, memory, collision PVS, cvars) and produces all 3D/2D visual output and query results consumed by the client (`cl_main.c`, `cl_scrn.c`) and cgame VM (`cg_syscalls.c`).

## Key Cross-References

### Incoming (who depends on this)
- **`code/client/cl_main.c`** — Loads the renderer DLL via `GetRefAPI` at startup; calls `re.BeginFrame`, scene building (`re.AddRefEntityToScene`, `re.AddPolyToScene`, `re.AddLightToScene`), and `re.RenderScene` each frame
- **`code/client/cl_scrn.c`** — Calls 2D draw functions: `re.SetColor`, `re.DrawStretchPic`, `re.DrawStretchRaw`, `re.UploadCinematic` for HUD and cinematics
- **`code/client/cl_cgame.c`** — Launches cgame VM, which uses `trap_R_*` (syscalls dispatched back to renderer functions)
- **`code/cgame/cg_syscalls.c`** — VM syscall dispatcher routing cgame `trap_R_*` calls to renderer `refexport_t` functions
- **`code/client/cl_cin.c`** — Cinematic playback uses `re.UploadCinematic` and `re.DrawStretchRaw`
- **Client initialization loop** — Each map load triggers `re.BeginRegistration` → `re.Register*` → `re.LoadWorld` → `re.SetWorldVisData` → `re.EndRegistration`

### Outgoing (what this file depends on)
- **`code/qcommon/files.c`** — Via `ri.FS_ReadFile`, `ri.FS_FileIsInPAK`, `ri.FS_ListFiles` — all texture/shader/model asset I/O
- **`code/qcommon/cm_*.c`** — Via `ri.CM_DrawDebugSurface` — collision world queries for visibility and bounding-box tests
- **`code/renderer/tr_init.c`** — Implements `GetRefAPI` and fills `refexport_t`; stores `refimport_t` in global `ri`
- **`code/renderer/tr_public.h` → `code/cgame/tr_types.h`** — Shared type definitions (`refEntity_t`, `refdef_t`, `glconfig_t`, `stereoFrame_t`, etc.)
- **Platform layer** — `GLimp_*` functions (not in this header but called by renderer internals) for OpenGL context, swaps, gamma
- **`code/jpeg-6/`** — Texture loading; renderer calls `ri.FS_ReadFile`, which may decompress JPEGs

## Design Patterns & Rationale

| Pattern | Rationale |
|---------|-----------|
| **Versioned vtable entry point** (`GetRefAPI(apiVersion, rimp)`) | DLL isolation: single linker-exported symbol. Versionization allows API evolution without breaking old mod code. |
| **Dual-table design** (`refexport_t` + `refimport_t`) | **Dependency injection**: renderer declares what it exports (services) and what it imports (dependencies). Inverts control; engine fills import table. |
| **Deferred scene rendering** (`ClearScene` → `Add*` → `RenderScene`) | Avoids immediate GL state changes; collects all scene data first, then sort & flush in `RenderScene` for optimal batching and SMP. |
| **Registration phase** (`BeginRegistration` → `Register*` → `EndRegistration`) | One-time asset precaching before gameplay prevents stalls from disk I/O during frame loop. |
| **Stereo frame parameter** (`BeginFrame(stereoFrame_t)`) | Era-specific VR/stereoscopic rendering (e.g., QuakeCon '99 had 3Dfx stereo demos). Platform/HMD setup deferred to `GLimp_*`. |
| **Memory callback delegation** (`ri.Hunk_Alloc`, `ri.Malloc`, `ri.Free`) | Renderer has **zero control** over memory lifecycle; all allocations must go through engine. Simplifies lifetime auditing and avoids allocator collisions in Windows DLL model. |

## Data Flow Through This File

### Initialization Path (One-time)
```
Engine: GetRefAPI(REF_API_VERSION, &rimp)
  └─→ Renderer: Validate version, store rimp globally, return &refexport_t
      (implemented in tr_init.c, not visible here)
```

### Per-Level Load (Once per map)
```
Client: BeginRegistration(&glconfig)
  ├─→ Renderer: Invalidate old assets, init cache, return video config
Client: RegisterModel(name), RegisterSkin(name), RegisterShader(name)
  ├─→ Renderer: ri.FS_ReadFile → load asset, cache in global hash tables
Client: LoadWorld(mapname)
  ├─→ Renderer: ri.FS_ReadFile → parse BSP, build RB tree
Client: SetWorldVisData(vis)
  ├─→ Renderer: Store PVS cluster data (from qcommon/cm_load.c)
Client: EndRegistration()
  ├─→ Renderer: Precache all registered textures to VRAM
```

### Per-Frame Render (Every 16ms at 60 FPS)
```
Client: BeginFrame(stereoFrame)
  ├─→ Renderer: GLimp_SetupFrame(), platform-specific setup

Client: ClearScene()
  ├─→ Renderer: Zero scene refEntity array, light list

Client: AddRefEntityToScene(re), AddPolyToScene(shader, verts), 
        AddLightToScene(pos, intensity, rgb)
  ├─→ Renderer: Append to scene arrays (no GL calls yet)

Client: RenderScene(refdef_t)
  ├─→ Renderer:
      ├─ Traverse BSP with PVS + frustum culling
      ├─ Sort all visible surfaces + entities by shader
      ├─ Execute queued GL commands (GLimp_EndFrame → swap)

Client: DrawStretchPic(...), SetColor(...) [HUD/UI]
  ├─→ Renderer: 2D quad rendering (orthographic mode)

Client: EndFrame(frontEndMsec, backEndMsec)
  ├─→ Renderer: Optional return timing for profiling (SMP case)
```

## Learning Notes

- **No scene persistence**: `ClearScene` each frame decouples renderer from game state snapshots; cgame rebuilds the scene every tick from `refdef_t` + entity snapshots.
- **Collision world is read-only**: The renderer only reads `vis` data (via `SetWorldVisData`); it never calls `CM_Trace` or `CM_GetArea`. Collision queries are server-authoritative (game VM owns them).
- **Syscall indirection overhead**: cgame → trap_R_* → client dispatcher → refexport_t is a ~4-step call chain. Q3A chose this to sandbox the VM; modern engines directly link cgame rendering code.
- **A3D hook** (`#ifdef __USEA3D`) is a relic of late-90s *Aureal* 3D audio hardware acceleration. Removed in most modern Q3 ports; illustrates how era-specific optimizations bloat ABIs.
- **No built-in LOD or streaming**: The renderer doesn't declare LOD management functions; the client/cgame VM decides what to render (e.g., distant players as sprites). This is intentional—rendering is **passive**, not adaptive.
- **Vignetting/gamma/palette management**: The header doesn't expose color correction. That's done via `glconfig_t.gammaCorrection` (filled once at init) or `SetColor` RGBA blending (per-draw).

## Potential Issues

- **Vtable initialization vulnerability**: If the engine fails to fill `refimport_t` correctly (e.g., NULL `ri.Malloc`), the renderer will crash with no validation. There's no defensive `ri` checks at module load.
- **Stereo frame lifetime unclear**: `BeginFrame(stereoFrame_t)` doesn't define when stereo mode affects scene vs. per-eye view matrices. SMP back-end threading adds complexity not documented here.
- **No error recovery in queries**: Functions like `LightForPoint`, `LerpTag`, `MarkFragments` return results without error codes. OOB calls may silently fail or return garbage.
- **PVS format tight coupling**: `SetWorldVisData` commits the renderer to a specific BSP PVS structure (cluster bytes). A map compiler update breaks binary compatibility; no versioning.
- **Thread safety of import table**: The renderer stores `refimport_t *rimp` globally and calls it from back-end thread (SMP mode). The header doesn't document synchronization; assumes engine-provided callbacks are thread-safe.
