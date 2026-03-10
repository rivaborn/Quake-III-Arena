# code/cgame/cg_public.h — Enhanced Analysis

## Architectural Role

This file defines the **runtime syscall ABI contract** between the engine's client layer (`code/client/`) and the cgame QVM module. It enumerates all engine services (renderer, sound, collision, filesystem, input) that cgame can invoke, plus all cgame entry points (init, draw frame, input events) that the engine calls. Together with `snapshot_t`, it forms the complete public interface for client-side game logic—the engine's sole dependency on cgame's implementation details.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/client/cl_cgame.c`** — Implements `CL_CgameSystemCalls()`, the dispatch handler that converts `cgameImport_t` syscall IDs into engine function calls (renderer via `re.*`, sound via `S_*`, filesystem via `FS_*`, etc.)
- **`code/cgame/cg_*.c` files** — Each includes this header via `cg_syscalls.c` to access the `cgameImport_t` enum IDs; uses them indirectly through `trap_*` macro wrappers (e.g., `trap_R_DrawStretchPic` → `CG_DRAWSTRETCHPIC`)
- **`code/cgame/cg_main.c`** — Implements `vmMain()` entry point; dispatcher handles all `cgameExport_t` function IDs (CG_INIT, CG_DRAW_ACTIVE_FRAME, CG_KEY_EVENT, etc.)
- **`code/client/cl_parse.c`** — Builds `snapshot_t` structures from inbound server packets; stores them in the client's snapshot ringbuffer for cgame to query
- **`code/qcommon/vm.c`** — The VM host infrastructure; knows about API_VERSION field for compatibility checking when loading cgame DLL/QVM

### Outgoing (what this file depends on)
- **`game/q_shared.h`** — Supplies `playerState_t`, `entityState_t`, byte, qboolean
- **`game/bg_public.h`** — Supplies player state and shared game type definitions
- **`qcommon/qfiles.h`** — Supplies `MAX_MAP_AREA_BYTES` constant for PVS bitmask size
- **Implicit globals** — `SNAPFLAG_*` constants (likely in `qcommon.h`) and trap ID dispatch logic in engine

## Design Patterns & Rationale

**1. Integer Dispatch ABI (Syscall Model)**
- The `cgameImport_t` enum values are **not C function pointers**; they're integer IDs used by the QVM interpreter to trap into the engine. This design:
  - Allows the QVM to run on CPU architectures without stable ABI (PPC, interpreter mode)
  - Provides coarse-grained sandboxing: cgame cannot call arbitrary engine code, only registered trap IDs
  - Avoids dynamic relocation issues when cgame is a DLL loaded at runtime
  - Predates modern plugin ABIs (COM, standard C library ABI)

**2. API Versioning** 
- `CGAME_IMPORT_API_VERSION = 4` is checked by `VM_Create()` at load time to reject incompatible versions. Rationale:
  - Syscall IDs are **order-dependent**: inserting a new trap ID in the middle breaks binary compatibility
  - The comment `// 1.32` shows late-stage API extensions; versioning prevents silent breakage when engine and cgame are built separately

**3. Double-Buffered Snapshot Pipeline**
- `snapshot_t` contains a **complete immutable world view** at one server timestamp, plus `serverCommandSequence` for reliable command delivery
- Cgame receives two snapshots per frame (`CG_GETSNAPSHOT`, `CG_GETCURRENTSNAPSHOTNUMBER`) to enable smooth interpolation and change detection
- The `areamask[MAX_MAP_AREA_BYTES]` PVS bitmask is **server-authoritative visibility**—cgame respects it for culling (though the server already culled entities)

**4. Event-Driven UI Overlay Modes**
- `CGAME_EVENT_NONE`, `CGAME_EVENT_SCOREBOARD`, etc. allow the engine to notify cgame of UI state changes without poll loops
- `CG_EVENT_HANDLING` syscall lets cgame reconfigure input routing (key capture for scoreboard vs. normal gameplay)

## Data Flow Through This File

```
Server → Network → Client
                      ↓
                  CL_ParseServerMessage()
                      ↓
                  Build snapshot_t (delta decode entities, player state, area mask)
                      ↓
            Ring buffer: snapshots[PACKET_BACKUP]
                      ↓
           CG_DrawActiveFrame (each engine frame)
                      ↓
        CG_GETSNAPSHOT (cgame trap_id)
                      ↓
        CL_CgameSystemCalls() dispatches
                      ↓
        cgame reads snapshot_t, fires events, calls:
        - CG_R_* (render scene/items/text)
        - CG_S_* (play sounds)
        - CG_CM_* (collision traces for prediction)
        - CG_GETSERVERCOMMAND (text commands from server)
                      ↓
         Scene submitted to renderer
         Audio queued to mixer
         HUD drawn as 2D overlay
```

Reverse direction:
```
User input (keyboard, mouse)
    ↓
CL_KeyEvent() / CL_MouseEvent()
    ↓
CG_KEY_EVENT / CG_MOUSE_EVENT syscalls
    ↓
cgame updates local usercmd_t
    ↓
CG_SETUSERCMDVALUE communicates to engine
    ↓
cl_input.c assembles usercmd_t for network
```

## Learning Notes

**Idiomatic to Quake III / Late 2000s Engine Design:**
1. **QVM-only interface**: No cgame code runs natively; all engine calls marshal through integer traps. Contrasts with modern engines (Unreal, Unity) where game code is compiled native and calls into engine via C++ interfaces.
2. **Snapshot is god**: The `snapshot_t` is the primary unit of truth; everything cgame does is a response to the latest snapshot. No persistent server-side player object accessible from client.
3. **Event system instead of callbacks**: Cgame discovers game events (deaths, item pickups) by reading event IDs in `playerState_t`/`entityState_t` deltas and calling event handlers. Modern engines use observer/delegate patterns.
4. **Fixed-size snapshot entity limit**: `MAX_ENTITIES_IN_SNAPSHOT = 256` was reasonable for 2001 hardware; now engines use dynamic allocation or streaming.
5. **Syscall parity between import and export**: Both directions use integer dispatch IDs. The engine calls `vmMain(export_id, ...)` with export IDs; cgame calls engine via trap IDs (import IDs).

**Modern Contrast:**
- Modern engines (UE4/5) compile game code as native DLLs with C++ virtual method dispatch (vtables)
- ECS-based engines (Bevy, DOTS) decouple entity data (snapshots) from behavior (systems); no monolithic "draw frame" call
- Network stacks often use event queues instead of snapshot ringbuffers

## Potential Issues

1. **Fragile API extensibility**: The `cgameImport_t` enum is **append-only**. Adding a new syscall in the middle breaks binary compatibility. The comment `// 1.32` and `CG_MEMSET = 100` (offset jump) suggest workarounds, but are error-prone. A versioned dispatch table would be more robust.

2. **Fixed entity cap**: `MAX_ENTITIES_IN_SNAPSHOT = 256` is a hard limit. Maps with >256 visible entities will drop entities silently. On high-end hardware or dense scenes, this is a bottleneck. (Modern engines use dynamic allocation or LOD culling.)

3. **Undefined constants**: `SNAPFLAG_RATE_DELAYED` and similar are referenced but not defined in this header. Cgame must `#include` them from elsewhere (likely `qcommon.h` or `game/q_shared.h`). This is a hidden dependency that can cause link-time surprises.

4. **No input validation in syscall dispatch**: The engine's `CL_CgameSystemCalls()` trusts cgame to pass valid trap IDs and argument counts. A malformed QVM could corrupt the engine or cause crashes. Modern VMs add bounds checking.

5. **Implicit ordering dependency**: The cgame QVM must be built **after** the engine is finalized, so trap IDs match. Any manual reordering of the `cgameImport_t` enum without recompiling cgame will silently call the wrong engine functions (e.g., `CG_PRINT` becomes `CG_CVAR_REGISTER`).
