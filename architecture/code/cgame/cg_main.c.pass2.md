# code/cgame/cg_main.c — Enhanced Analysis

## Architectural Role

`cg_main.c` is the **ABI root and global state owner** of the cgame VM module. It sits at the boundary between `code/client` (which hosts the VM via `qcommon/vm.c`) and the rest of the cgame subsystem. Every engine-to-cgame call transits `vmMain`, which makes this file the single point of trust for the engine's `VM_Call` dispatch. It also owns the only copies of `cg`, `cgs`, `cg_entities`, `cg_weapons`, and `cg_items` — the entire memory substrate that all 20+ other cgame files read from. Because the cgame VM may run as a QVM bytecode image, a JIT-compiled x86 native, or a hard-linked DLL, `vmMain` must literally be the first compiled symbol in the `.q3vm` link order — a constraint enforced by comment and build script rather than language.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/client/cl_cgame.c`** is the primary caller. It invokes `VM_Call(cgvm, CG_INIT, ...)`, `VM_Call(cgvm, CG_DRAW_ACTIVE_FRAME, ...)`, and the input event commands — all of which land in `vmMain` here. `cl_cgame.c` also reads `cgs.cursorX/Y` (set in cgame) and queries the cgame VM for crosshair/attacker player numbers.
- **All other `cg_*.c` files** read `cg` and `cgs` globals declared here. `cg_snapshot.c` writes into `cg.snap`; `cg_predict.c` reads `cg.predictedPlayerState`; `cg_draw.c` reads dozens of `vmCvar_t` globals declared here; `cg_players.c` reads `cg_entities[]` and `cg_weapons[]`.
- **`bg_pmove.c` and `bg_misc.c`** (compiled identically into both game and cgame VMs) depend on the `Com_Error`/`Com_Printf` stubs defined here under `#ifndef CGAME_HARD_LINKED`. Without these shims the shared physics code cannot link, because it calls `Com_Error` rather than any cgame-specific error path.

### Outgoing (what this file depends on)

- **Renderer (`trap_R_*`):** `CG_RegisterGraphics` calls `trap_R_LoadWorldMap`, `trap_R_RegisterModel`, `trap_R_RegisterShader`, `trap_R_RegisterSkin` — mapping all visual assets into `cgs.media`.
- **Sound (`trap_S_*`):** `CG_RegisterSounds` registers all sound assets. `CG_Init` calls `trap_S_ClearLoopingSounds` and `CG_StartMusic`.
- **Collision model (`trap_CM_LoadMap`, `trap_CM_InlineModel`):** loads the BSP for client-side traces (used by prediction and mark decals).
- **Cvar system (`trap_Cvar_Register`, `trap_Cvar_Update`, `trap_Cvar_Set`):** all 70+ cgame cvars flow through the table here.
- **`code/cgame/cg_view.c`** (`CG_DrawActiveFrame`), **`cg_consolecmds.c`** (`CG_ConsoleCommand`, `CG_KeyEvent`, `CG_MouseEvent`, `CG_EventHandling`), **`cg_players.c`** (`CG_NewClientInfo`), **`cg_snapshot.c`** (`CG_SetConfigValues`) are all dispatched from `vmMain`/`CG_Init`.
- **`ui/ui_shared.h` (MISSIONPACK):** `CG_LoadHudMenu` wires `cgDC` as a function-pointer vtable into the shared menu framework — making cgame a consumer of the same widget system used by the Team Arena UI VM.

## Design Patterns & Rationale

**Data-driven cvar registration (`cvarTable`):** All ~70 cvars are registered in a single loop from a static struct array. This avoids a 70-call waterfall of `trap_Cvar_Register` scattered across the file and makes it trivial to audit defaults and flags in one place. The pattern is shared with `g_main.c` in the game VM and the UI VMs.

**Modification-count polling for change detection:** Rather than callbacks or notifications, `CG_UpdateCvars` compares `cg_forceModel.modificationCount` against a saved value each frame. This is idiomatic for Q3's engine — the cvar system is purely pull-based. The `forceModelModificationCount` global (misleadingly file-scope rather than true `static`) is the simplest possible change detector.

**VM entry-point first-symbol requirement:** The `.q3asm` link file lists `cg_main.c` first, ensuring `vmMain` is at bytecode offset 0. The engine calls `VM_Call(vm, 0, ...)` by index into the export table. This is a static ABI contract imposed by the QVM format, not C linkage.

**`Com_Error`/`Com_Printf` shims:** The shared `bg_*` / `q_shared.c` code calls `Com_Error` and `Com_Printf`. Since those functions live in `qcommon/common.c` (outside the VM sandbox), the cgame VM provides its own wrapper implementations that forward to `trap_Error`/`trap_Print`. This is a deliberate seam allowing the same physics source to compile into three different execution contexts (engine, game VM, cgame VM) without modification.

## Data Flow Through This File

```
Engine (cl_cgame.c)
    │  VM_Call(CG_INIT, serverMsgNum, cmdSeq, clientNum)
    ▼
vmMain → CG_Init
    ├─ memset(cg, cgs, cg_entities, cg_weapons, cg_items) ← zero all state
    ├─ CG_RegisterCvars  → trap_Cvar_Register (cvars → cvarTable globals)
    ├─ trap_CM_LoadMap   → collision world loaded into engine
    ├─ CG_RegisterSounds → sound handles → cgs.media
    ├─ CG_RegisterGraphics → model/shader handles → cgs.media + trap_R_LoadWorldMap
    └─ CG_RegisterClients → per-player models → cg_entities[].currentState

Per-frame:
Engine (cl_cgame.c)
    │  VM_Call(CG_DRAW_ACTIVE_FRAME, time, stereoView, isActive)
    ▼
vmMain → CG_DrawActiveFrame (cg_view.c)
              reads cg, cgs, cg_entities, vmCvar_t globals ← all owned here

Per-frame cvar sync:
CG_UpdateCvars (called from cg_view.c each frame)
    ├─ trap_Cvar_Update all entries in cvarTable
    ├─ detect cg_drawTeamOverlay change → trap_Cvar_Set("teamoverlay", ...)
    └─ detect cg_forceModel change → CG_ForceModelChange → CG_NewClientInfo × MAX_CLIENTS
```

## Learning Notes

- **QVM module entry point convention:** The engine's VM hosting layer (`vm.c`) does not use dynamic symbol lookup. It calls offset 0 in the bytecode image. `vmMain` must be at that offset — a constraint invisible from C code alone. Modern engines use explicit DLL export tables or plugin registration callbacks instead.
- **Global state as the entire module's heap:** There is no heap allocation for the primary game state — `cg`, `cgs`, `cg_entities[]`, `cg_weapons[]`, `cg_items[]` are all statically allocated globals. This is deliberate for QVM: the interpreter's `dataMask` enforces that the VM cannot access memory outside its statically allocated data segment. Modern engines use component/ECS architectures with dynamic allocation.
- **Separation of transient (`cg_t`) vs. persistent (`cgs_t`) state:** `cg` is zeroed on every `CG_Init` call; `cgs` holds data that survives tournament restarts (like configstrings and media handles). This two-tier model maps to the modern concept of scene state vs. resource state.
- **The E3 HACK comment** in `CG_UpdateCvars` forces `teamoverlay` always on regardless of user preference. This is a live shipping regression shipped in the released source — a window into how demo/trade show deadlines leave permanent marks in production code.
- **`cg_pmove_msec` vs. `pmove_msec`:** Two separate cvars exist for the same concept (physics timestep), with the cgame-prefixed one commented out. This reflects the ongoing tension between client-side and server-side authority over physics parameters, resolved in this era by using server-communicated `systeminfo` cvars.

## Potential Issues

- **`vsprintf` without bounds checking** in `CG_Printf` and `CG_Error` — fixed-size 1024-byte buffers with unbounded format expansion. A format string or argument producing >1024 bytes silently overflows the stack. Modern code uses `vsnprintf`.
- **`forceModelModificationCount` is declared at file scope without `static`**, making it a genuine external-linkage global despite being an implementation detail. Any other TU in a hard-linked build could accidentally alias it.
- **The teamoverlay FIXME/E3 HACK** (`trap_Cvar_Set("teamoverlay", "1")` unconditionally after the conditional set) means `cg_drawTeamOverlay 0` never actually disables team overlay userinfo reporting to the server — a behavioral bug shipped in the release sources.
