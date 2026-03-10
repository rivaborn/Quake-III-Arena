# code/client/cl_cgame.c — Enhanced Analysis

## Architectural Role

This file is the **VM boundary adapter** for the cgame module — the symmetric counterpart to `sv_game.c`'s `SV_GameSystemCalls`. It sits at the intersection of five subsystems: the qcommon VM host (`vm.c`), the client connection state machine (`cl_main.c`/`cl_parse.c`), the renderer (`re` vtable from `cl_main.c`), the collision system (`CM_*`), and the sound system (`S_*`). Every service the cgame VM needs from the engine passes through `CL_CgameSystemCalls`'s ~60-case dispatch. No other file in the client layer aggregates this many subsystem dependencies simultaneously — it is intentionally a hub, not a layered module.

The file also owns the **client-time synchronization pipeline**: `CL_AdjustTimeDelta` and `CL_SetCGameTime` manage the offset between wall-clock and server time, which governs all interpolation and prediction in cgame. This logic belongs here because it directly drives what the cgame VM sees when it calls `trap_GetCurrentSnapshotNumber`.

## Key Cross-References

### Incoming (who depends on this file)

- **`cl_main.c`**: Calls `CL_InitCGame`, `CL_ShutdownCGame`, `CL_CGameRendering`, `CL_SetCGameTime`, `CL_GameCommand`, `CL_FirstSnapshot`. This is the only file that orchestrates the cgame VM lifecycle.
- **`cl_parse.c`**: Calls `CL_FirstSnapshot` after the first valid snapshot arrives; drives `CL_GetServerCommand` indirectly via the cgame VM's `trap_GetServerCommand` syscall path.
- **`qcommon/vm.c`**: The `cgvm` handle created here is used by `VM_Call` and `VM_Free` throughout the client; `CL_CgameSystemCalls` is registered as the callback function pointer when `VM_Create` is called, making it the sole re-entry point from the VM sandbox.
- **cgame VM itself**: Every `trap_*` call in `code/cgame/cg_syscalls.c` routes to `CL_CgameSystemCalls` at runtime.

### Outgoing (what this file depends on)

- **`cl_main.c` globals**: `cgvm` (`vm_t*`), `re` (`refexport_t`), `cl_connectedToPureServer`, and the three client state globals `cl`, `clc`, `cls` are all defined in `cl_main.c` and read/written here.
- **Renderer** (`re.*`): All `CG_R_*` cases delegate to `re.RegisterModel`, `re.AddPolyToScene`, `re.RenderScene`, etc. — the entire renderer API surface is exposed to cgame through this file.
- **Collision** (`CM_*` from `qcommon/cm_*.c`): Map load, inline models, box/capsule traces, point contents.
- **Sound** (`S_*` from `client/snd_dma.c`): Register sounds, start/stop sounds, add loop sounds, update listener.
- **botlib** (`botlib_export` from `be_interface.c`): `CG_PC_*` cases (script parser) forward to `botlib_export->PC_*`. This is the only place cgame touches botlib — solely for parsing `.bot` personality/character files at menu time.
- **`cl_cin.c`**: `CIN_*` cinematic calls for in-game videos (briefings, end screens).
- **Camera functions** (`loadCamera`, `startCamera`, `getCameraInfo`): Declared `extern` from an unknown translation unit; all their dispatch cases in `CL_CgameSystemCalls` are compiled dead — the actual call sites are commented out.

## Design Patterns & Rationale

**Indexed syscall dispatch (trap table)**: The large `switch(args[0])` in `CL_CgameSystemCalls` is the engine's VM ABI. The integer trap numbers defined in `cg_public.h` form a versioned contract — adding a new syscall requires adding a case here and a matching `trap_*` wrapper in `cg_syscalls.c`. This avoids linking the cgame module against engine code directly, enabling QVM bytecode portability. The same pattern appears in `SV_GameSystemCalls` and `CL_UISystemCalls`.

**`VMA`/`VMF` boundary macros**: `VM_ArgPtr` translates VM-sandbox-relative integer addresses into host pointers (applying `dataMask` bounds enforcement). This is the sandboxing mechanism — cgame cannot pass an arbitrary host pointer; it passes an integer offset into its data segment. `VMF` reinterprets the integer as IEEE-754 float without a cast, exploiting type-punning to pass float arguments through the integer `args[]` array.

**`bcs0/bcs1/bcs2` + `goto rescan`**: The three-phase big-configstring protocol splits oversized configstrings across multiple reliable commands (to fit within `MAX_STRING_CHARS`). The `goto rescan` on `bcs2` completion re-tokenizes the assembled string as if it arrived as a normal `cs` command. This is a deliberate layering hack — the command parser is reused as an intermediate format.

**Circular buffer validation trinity**: Every history accessor (`CL_GetSnapshot`, `CL_GetParseEntityState`, `CL_GetUserCmd`) applies the same three-check pattern: future guard → overflow guard → pointer derivation. This prevents cgame from reading stale or uninitialized data when the ring buffers wrap.

## Data Flow Through This File

```
Server network packets
  → cl_parse.c (delta-decode snapshots → cl.snapshots[] ring buffer)
  → CL_SetCGameTime (server-time drift correction → cl.serverTime)
  → cgame VM calls trap_GetSnapshot
  → CL_GetSnapshot (translate clSnapshot_t → snapshot_t, resolve parseEntities[] indices)
  → cgame consumes snapshot, builds render scene
  → cgame calls trap_R_RenderScene
  → CL_CgameSystemCalls (CG_R_RENDERSCENE) → re.RenderScene()
  → renderer draws frame
```

Configstring updates flow orthogonally:
```
Server reliable command "cs <idx> <val>"
  → CL_GetServerCommand (called by cgame trap_GetServerCommand)
  → CL_ConfigstringModified (rebuilds cl.gameState string table)
  → if CS_SYSTEMINFO: CL_SystemInfoChanged (parses serverId cvar)
  → qtrue returned → cgame receives command token and acts on it
```

## Learning Notes

- **Why no direct function pointers to cgame?** The entire file exists because cgame must run as QVM bytecode (for pure-server validation and cross-platform distribution). Direct function-pointer vtables like the renderer's `refexport_t` would require native linking. The integer trap ABI is the QVM-compatible alternative.
- **The symmetry with `sv_game.c`**: A developer studying Q3 quickly sees the pattern — every VM (cgame, game, ui) has a `*_SystemCalls` dispatch in its host layer. The server does this in `sv_game.c`; the client does it here and in `cl_ui.c`.
- **Time delta drift correction** (`CL_AdjustTimeDelta`): The hard/fast/slow three-speed adjustment is an early example of PLL-like clock synchronization in game engines. Modern engines use more sophisticated jitter buffers, but the concept is identical.
- **`FloatAsInt`**: Type-punning float→int through a union or pointer cast was standard late-1990s C practice before `memcpy`-based type punning became the idiom. Technically UB under strict aliasing, but universally "works" on the targeted compilers.
- **No ECS here**: Entity management in Q3 is flat arrays indexed by entity number — no component system, no scene graph. The cgame consumes `entityState_t` arrays directly from snapshots and manually dispatches per-entity rendering by `eType`.

## Potential Issues

- **`CG_R_REGISTERFONT` fall-through**: Missing `return 0` after `re.RegisterFont(...)` causes execution to fall through into `CG_R_CLEARSCENE`. This is a definite latent bug; it only avoids crashing because `CG_R_CLEARSCENE` calls `re.ClearScene()` which is benign, but the return value of `RegisterFont` is lost.
- **`CG_UPDATESCREEN` event loop comment**: The commented-out `Com_EventLoop()` call with its FIXME note reveals a genuine re-entrancy hazard — map download + map change while loading can trigger a server restart inside the event loop, crashing the client. The workaround (`SCR_UpdateScreen` only) means loading screens may miss network events during long loads.
- **`bigConfigString` is `static` local**: The `bcs0/bcs1/bcs2` assembly buffer is a static within `CL_GetServerCommand`. This is not thread-safe and assumes strictly sequential server command processing — valid given Q3's single-threaded client loop, but fragile.
- **Dead camera API**: Three `extern` camera functions are declared and dispatch cases exist in the syscall table, but all call sites are commented out. The declarations reference a translation unit not identifiable from context — possibly a cut feature from the scripted camera system.
