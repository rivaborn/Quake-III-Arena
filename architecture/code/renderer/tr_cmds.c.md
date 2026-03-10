# code/renderer/tr_cmds.c

## File Purpose
This file implements the renderer's command buffer system, acting as the bridge between the front-end (scene submission) and back-end (GPU execution) render threads. It manages double-buffered render command lists and supports optional SMP (symmetric multiprocessing) via a dedicated render thread.

## Core Responsibilities
- Initialize and shut down the SMP render thread
- Provide a command buffer allocation mechanism (`R_GetCommandBuffer`)
- Enqueue typed render commands (draw surfaces, set color, stretch pic, draw buffer, swap buffers)
- Issue buffered commands to the back end (single-threaded or SMP wake)
- Synchronize front and back end threads before mutating shared GL state
- Display per-frame performance counters at various verbosity levels

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `renderCommandList_t` | struct (defined in tr_local.h) | Fixed-size byte buffer (`cmds[MAX_RENDER_COMMANDS]`) plus `used` offset; holds serialized render commands for one frame |
| `drawSurfsCommand_t` | struct | Command payload for submitting a batch of draw surfaces with refdef and viewparms |
| `setColorCommand_t` | struct | Command payload for a 2D color modulate |
| `stretchPicCommand_t` | struct | Command payload for a 2D textured quad |
| `drawBufferCommand_t` | struct | Command payload selecting GL draw buffer (front/back/stereo) |
| `swapBuffersCommand_t` | struct | Command payload triggering buffer swap at end of frame |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `renderCommandList` | `volatile renderCommandList_t *` | global | Pointer passed to the render thread; set by `GLimp_WakeRenderer` |
| `renderThreadActive` | `volatile qboolean` | global | Flag indicating the render thread is currently processing commands |
| `c_blockedOnRender` | `int` | global | Perf counter: main thread stalled waiting for render thread |
| `c_blockedOnMain` | `int` | global | Perf counter: render thread was idle (main was ahead) |

## Key Functions / Methods

### R_PerformanceCounters
- **Signature:** `void R_PerformanceCounters(void)`
- **Purpose:** Prints renderer statistics to console at the verbosity level set by `r_speeds`, then clears both front-end and back-end perf counters.
- **Inputs:** None (reads `r_speeds->integer`, `tr.pc`, `backEnd.pc`, `glConfig`)
- **Outputs/Return:** void; prints to console via `ri.Printf`
- **Side effects:** Zeroes `tr.pc` and `backEnd.pc` on every call
- **Calls:** `R_SumOfUsedImages`, `ri.Printf`, `Com_Memset`
- **Notes:** Called only after the back-end has finished (safe to read `backEnd.pc`)

### R_InitCommandBuffers
- **Signature:** `void R_InitCommandBuffers(void)`
- **Purpose:** Optionally spawns the SMP render thread if `r_smp` is set.
- **Inputs:** None
- **Outputs/Return:** void; sets `glConfig.smpActive`
- **Side effects:** May spawn OS thread via `GLimp_SpawnRenderThread`
- **Calls:** `GLimp_SpawnRenderThread`, `ri.Printf`

### R_ShutdownCommandBuffers
- **Signature:** `void R_ShutdownCommandBuffers(void)`
- **Purpose:** Signals the render thread to exit by waking it with a NULL payload.
- **Side effects:** Sets `glConfig.smpActive = qfalse`
- **Calls:** `GLimp_WakeRenderer`

### R_IssueRenderCommands
- **Signature:** `void R_IssueRenderCommands(qboolean runPerformanceCounters)`
- **Purpose:** Terminates the current command list, waits for the render thread if needed, then dispatches the list to the back end (directly or via SMP wake).
- **Inputs:** `runPerformanceCounters` — whether to call `R_PerformanceCounters` after sync
- **Side effects:** Resets `cmdList->used = 0`; increments `c_blockedOnRender` or `c_blockedOnMain`; calls `RB_ExecuteRenderCommands` or `GLimp_WakeRenderer`
- **Calls:** `GLimp_FrontEndSleep`, `R_PerformanceCounters`, `RB_ExecuteRenderCommands`, `GLimp_WakeRenderer`
- **Notes:** The `RC_END_OF_LIST` sentinel is written directly into the buffer before dispatch.

### R_SyncRenderThread
- **Signature:** `void R_SyncRenderThread(void)`
- **Purpose:** Flushes pending commands and blocks until the render thread is idle; allows the main thread to issue OpenGL calls safely.
- **Calls:** `R_IssueRenderCommands`, `GLimp_FrontEndSleep`
- **Notes:** No-op if `!tr.registered`

### R_GetCommandBuffer
- **Signature:** `void *R_GetCommandBuffer(int bytes)`
- **Purpose:** Reserves `bytes` of space in the active frame's command list.
- **Inputs:** `bytes` — size of the command struct to allocate
- **Outputs/Return:** Pointer into `cmdList->cmds`, or `NULL` if the buffer is full
- **Side effects:** Advances `cmdList->used`
- **Notes:** Leaves 4 bytes headroom for the `RC_END_OF_LIST` sentinel; fatal error if a single command exceeds buffer capacity.

### RE_BeginFrame
- **Signature:** `void RE_BeginFrame(stereoFrame_t stereoFrame)`
- **Purpose:** Per-frame setup: increments frame counters, handles overdraw measurement toggle, texture mode and gamma changes, GL error checking, and enqueues an `RC_DRAW_BUFFER` command.
- **Side effects:** May call `R_SyncRenderThread` multiple times; may enable/disable GL stencil test; writes `RC_DRAW_BUFFER` command
- **Calls:** `R_SyncRenderThread`, `GL_TextureMode`, `R_SetColorMappings`, `qglGetError`, `R_GetCommandBuffer`, various `qgl*`

### RE_EndFrame
- **Signature:** `void RE_EndFrame(int *frontEndMsec, int *backEndMsec)`
- **Purpose:** Enqueues `RC_SWAP_BUFFERS`, dispatches all accumulated commands, toggles the SMP double-buffer frame, and returns timing data.
- **Side effects:** Calls `R_IssueRenderCommands(qtrue)`, `R_ToggleSmpFrame`, resets `tr.frontEndMsec` and `backEnd.pc.msec`
- **Notes:** This is the canonical end of the render pipeline for a frame.

### R_AddDrawSurfCmd / RE_SetColor / RE_StretchPic
- These are thin command-enqueue wrappers: each calls `R_GetCommandBuffer`, fills in the command struct, and returns. No logic beyond filling fields.

## Control Flow Notes
- **Init:** `R_InitCommandBuffers` is called during renderer init; may spawn SMP thread.
- **Frame:** `RE_BeginFrame` → scene submission (`R_AddDrawSurfCmd`, `RE_SetColor`, `RE_StretchPic`, etc.) → `RE_EndFrame`.
- `RE_EndFrame` is the only place `R_IssueRenderCommands(qtrue)` (with perf counters) is called; all mid-frame syncs use `R_SyncRenderThread` which passes `qfalse`.
- **Shutdown:** `R_ShutdownCommandBuffers` wakes the render thread with NULL to signal exit.

## External Dependencies
- `tr_local.h` — all renderer types, globals (`tr`, `backEnd`, `glConfig`, `glState`), cvars, SMP platform functions
- `GLimp_SpawnRenderThread`, `GLimp_FrontEndSleep`, `GLimp_WakeRenderer` — platform-specific SMP primitives (defined in `win_glimp.c` / `linux_glimp.c`)
- `RB_ExecuteRenderCommands`, `RB_RenderThread` — defined in `tr_backend.c`
- `R_ToggleSmpFrame`, `R_SumOfUsedImages`, `R_SetColorMappings`, `GL_TextureMode` — defined elsewhere in the renderer
