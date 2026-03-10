# code/ui/ui_public.h — Enhanced Analysis

## Architectural Role
This header encodes the **VM sandbox boundary** between the engine core and untrusted UI code. It enforces a strict contract: the UI VM (whether native DLL or QVM bytecode) can only call the engine through indexed `trap_*` syscalls (uiImport_t), and the engine can only invoke UI by dispatching through numbered entry points (uiExport_t). This isolation pattern is inherited from the Quake II / Heretic II era, enabling hot-reloadable UI modules without engine recompilation or relinker dependencies.

## Key Cross-References

### Incoming (who depends on this file)
- **code/client/cl_ui.c** — Hosts the UI VM; dispatches `uiExport_t` entry points (UI_INIT, UI_REFRESH, UI_KEY_EVENT, etc.) each frame via `VM_Call(uivm, ...)`
- **code/q3_ui/** and **code/ui/** — Implement the `uiExport_t` vtable as `vmMain()` entry; wrap all `uiImport_t` syscalls via `trap_*` macros (e.g., `trap_R_RegisterModel` calls syscall with opcode `UI_R_REGISTERMODEL`)
- **code/qcommon/vm.c** — Implements the `trap_*` syscall routing layer; maps `uiImport_t` opcode numbers to engine service functions (qcommon, client, renderer, sound)

### Outgoing (what this file depends on)
- **code/q_shared.h** — `connstate_t`, `MAX_STRING_CHARS` types used in `uiClientState_t`
- **code/qcommon/qcommon.h** — Implicit dependency via `connstate_t` definition
- **code/renderer/tr_types.h** — Implicitly required by code consuming UI render syscalls (`UI_R_*`)

## Design Patterns & Rationale

**VM Sandbox via Numbered Syscalls:**
- Unlike modern scripting (Lua, WebAssembly), Q3 uses a C-based VM loaded as a DLL or JIT-compiled bytecode. All engine calls are routed through `syscall(opcode, arg0, arg1, ...)`, making it impossible for UI code to directly call engine functions or dereference arbitrary memory.
- Versioning (`UI_API_VERSION = 6`) allows the engine to reject incompatible UI modules at runtime.

**Sparse Opcode Space:**
- Main syscalls (0–130): file I/O, rendering, sound, input, network, cvar, command execution, cinematics.
- High range (100+): math/memory builtins (`MEMSET`, `SQRT`, `SIN`, etc.) to avoid conflicts; allows adding opcodes mid-range without breaking the builtin range.

**One-Way Coupling:**
- Engine knows about `uiImport_t` / `uiExport_t` ordinals (hardcoded in `cl_ui.c` and `qcommon/vm.c`), but the UI VM only sees symbolic trap wrappers (`trap_R_DrawStretchPic` = `syscall(UI_R_DRAWSTRETCHPIC, ...)`).
- This makes UI VM implementations interchangeable: `code/q3_ui` and `code/ui` can coexist because they implement the same `uiExport_t` contract.

**Stateless Frame-Driven Loop:**
- Engine drives UI via `UI_REFRESH(time)` each frame. UI is pure function of current input/state; no implicit engine callbacks except syscalls. Simplifies debugging and replay.

## Data Flow Through This File

1. **Client Layer** (`cl_ui.c`): Initializes UI VM at startup → calls `VM_Call(uivm, UI_INIT)`.
2. **Per-Frame Loop**:
   - Input events: `UI_KEY_EVENT(key)`, `UI_MOUSE_EVENT(dx, dy)`.
   - Render trigger: `UI_REFRESH(time)`.
   - Menu state changes: `UI_SET_ACTIVE_MENU(uiMenuCommand_t)`.
3. **During UI Refresh**, the UI VM calls back:
   - **Rendering**: `trap_R_RegisterModel`, `trap_R_DrawStretchPic`, `trap_R_SetColor`, etc. → routed to **Renderer** (`code/renderer/tr_*.c`).
   - **Sound**: `trap_S_RegisterSound`, `trap_S_StartLocalSound` → routed to **Client audio** (`code/client/snd_*.c`).
   - **Input/Cvar**: `trap_Key_*`, `trap_Cvar_*` → routed to **Client** input and cvar systems.
   - **Filesystem**: `trap_FS_*` → routed to **qcommon** virtual filesystem.
   - **Server browser**: `trap_LAN_*` → routed to **Client** server-list cache (`servercache.dat`).
   - **Connection state**: `trap_GetClientState()` returns `uiClientState_t` snapshot (server name, connection status, messages).

4. **Shutdown**: `UI_SHUTDOWN()` → VM freed, all UI assets released.

## Learning Notes

**Era-Specific Design:**
- This represents the **early 2000s approach** to dynamic module loading in game engines. Modern engines (Unreal, Unity, Godot) use either:
  - Scripting languages (Lua, Python, C#) with VM-agnostic bridging.
  - ECS architectures where UI is data-driven, not a separate VM.
  - Hot-reloading via DLL reload (not QVM bytecode).

**Strengths of This Design:**
- Bulletproof sandboxing: UI code cannot crash the engine via memory access or recursion (VM enforces stack limits).
- Swap implementations without engine rebuild: `code/q3_ui` vs. `code/ui` are two separate QVMs.
- Deterministic for demos/replays: all I/O is logged via syscall boundaries.

**Weaknesses (in modern hindsight):**
- Opcode enum is brittle: reordering breaks all compiled UI modules. (Addressed in later Q3A patches by appending new opcodes at the end.)
- Overhead: every render call crosses VM → engine boundary (trampoline cost, especially under SMP).
- Hard to debug: UI code operates in a separate VM stack; stack traces don't bridge into engine.

**Connections to Engine Concepts:**
- **VM Hosting**: `qcommon/vm.c` is the sandbox enforcer; this header is its contract.
- **Syscall Dispatch**: `CL_UISystemCalls()` in `code/client/cl_ui.c` maps `uiImport_t` opcodes to function calls; similar pattern used for cgame (`CL_CgameSystemCalls`) and game (`SV_GameSystemCalls`).
- **Data Marshaling**: `uiClientState_t` is a single struct passed to UI at runtime; cgame receives `clSnapshot_t` + `playerState_t`; game receives `entityState_t`. Minimal serialization; all pointers are resolved on the engine side.

## Potential Issues

**Fragility of Opcode Ordering:**
- If a developer inserts a new opcode (e.g., `UI_NEW_FEATURE`) between existing opcalls, all pre-compiled UI VMs break silently (they'll call the wrong engine functions).
- Mitigation in real code: appended new opcodes at the end (post-1.32 patches) rather than inserting.
- A version check in `UI_GETAPIVERSION` would catch major breaks, but not micro-changes.

**Missing Null Checks in Consuming Code:**
- The UI VM can call `trap_GetClientState()` and receive a `uiClientState_t*` pointer. If the client layer corrupts or nulls this before the UI VM reads it, the result is undefined. (Unlikely in practice given the frame-synchronous design, but not explicitly documented.)

**No explicit forward-compatibility mechanism:**
- `UI_API_VERSION = 6` is a simple integer. If the engine is version 7 and a UI module reports version 6, the client layer (`cl_ui.c`) should reject it, but this is not enforced at the header level—it's a runtime check in the engine initialization code.
