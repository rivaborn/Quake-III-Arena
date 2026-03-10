# code/game/bg_lib.h — Enhanced Analysis

## Architectural Role

This header is the **VM-exclusive C standard library shim** serving all three QVM modules (game, cgame, ui). It exists because the Quake III virtual machine cannot link against the host OS's C runtime; instead, each VM-compiled module gets lightweight declarations that resolve to `bg_lib.c` implementations at link time. Unlike the native engine, which calls the platform C library directly, VM code goes through this custom layer, ensuring deterministic behavior across platforms and maintaining the sandbox boundary enforced by `qcommon/vm.c`'s `dataMask` memory protection.

## Key Cross-References

### Incoming (who depends on this)
- **game VM** (`code/game/`) — ai_*.c, g_*.c modules use `strlen`, `strcmp`, `strcpy`, `qsort`, `atoi`, `sin`, `cos`, `sqrt` for physics, combat, pathfinding, and entity management
- **cgame VM** (`code/cgame/`) — client-side prediction and rendering code uses `atof`, `vsprintf`, `memcpy` for snapshot interpolation and HUD formatting
- **ui VMs** (`code/q3_ui/`, `code/ui/`) — menu code uses string and memory functions for widget state and text rendering
- **Compiled alongside:** `bg_pmove.c`, `bg_misc.c`, `bg_lib.c` are replicated into each VM's object files to guarantee identical bytecode across platforms
- Implementations live in `code/game/bg_lib.c`, compiled into each QVM's bytecode image via `lcc`

### Outgoing (what this file depends on)
- **Zero external includes** — this is intentionally the bottom of the VM dependency chain
- Implementations in `bg_lib.c` may call `trap_*` syscalls (e.g., `trap_Printf` for debug output) to reach engine services
- No direct link to `qcommon` — all engine access is sandboxed via the VM→host syscall protocol

## Design Patterns & Rationale

**VM Sandbox + Determinism:** Q3A's game logic runs in a restricted bytecode VM to enable mod distribution without code injection risk and to guarantee identical simulation across all clients. This header enforces that boundary by providing **no access to OS system calls** — file I/O, network, process creation, etc. are unavailable. The engine only gives VMs what they need: memory ops, string ops, math, and sorting.

**32-bit everywhere:** The hardcoded `INT_MIN`, `INT_MAX`, and `size_t = int` reflect the 2005-era assumption that 32-bit is universal. This ensures VMs produce identical bytecode on x86, PPC, and other 32-bit targets. (Not portable to x86-64, but Q3A's VM layer never intended that.)

**`va_list` unsafety:** The cdecl-style `va_list` macros assume a specific stack layout (`ap = (va_list)&v + _INTSIZEOF(v)`). This works on x86/x86-64 cdecl but would break on ARM, MIPS, or x86-64 System V ABIs with different argument-passing conventions. The comment "NOT included on native builds" is crucial — native code must never use these macros.

**Non-standard `_atoi` / `_atof`:** The underscore variants accept `const char **` and advance the pointer in-place. This enables streaming parsers (e.g., reading BSP entity strings in real-time) without buffering.

## Data Flow Through This File

**Compile-time (lcc → bytecode):**
1. Game/cgame/ui source includes `bg_lib.h`
2. `lcc` compiler resolves `strlen()` call to a VM instruction referencing the exported symbol
3. Bytecode blob contains QVM `CALL` instructions for each library function

**Runtime (VM interpreter → implementation):**
1. QVM bytecode executes `CALL strlen`
2. `vm_interpreted.c` (or `vm_x86.c` JIT) dispatches to the engine's `bg_lib.c` implementation
3. `bg_lib.c` function (e.g., `strlen`) runs in native code, then returns a result to the VM
4. If the implementation needs engine services, it calls `trap_Printf` → syscall → engine

**Example: Game AI calling `qsort` on weapon scores:**
- `ai_weap.c` → `qsort(scores, count, sizeof(int), weaponCompareFn)` (QVM bytecode)
- Interpreter finds `bg_lib.c`'s `qsort` native implementation
- Native `qsort` repeatedly calls the comparator function pointer (which is a QVM bytecode address)
- Comparator re-enters VM bytecode
- Result copied back to VM's hunk memory

## Learning Notes

**Early 2000s VM design:** This file exemplifies why engines began using VMs for game logic. Before VMs, modders had full OS access; by sandboxing, Quake III prevented rootkits. The tradeoff: mods cannot use std::string, threads, or any advanced C++ — just bare C with a minimal stdlib.

**Precursor to modern sandboxing:** Modern engines (Unreal, Unity) use managed runtimes (C#, Lua, AngelScript). Q3A's approach — minimal function whitelist + bytecode interpreter + memory bounds checking — is cruder but more portable than JIT and safer than native code injection.

**Shared physics determinism:** The presence of `bg_pmove.c` and `bg_misc.c` compiled identically into both game and cgame ensures that a client predicting `usercmd_t` forward and a server running `GAME_RUN_FRAME` stay in sync. This header supplies the math/string primitives needed for that convergence.

**No file I/O in VMs:** Notice `fopen`, `fread`, `fprintf` are absent. Game VMs cannot touch the filesystem — they must ask the engine via `trap_FS_*` syscalls. This prevents mod-based save-game manipulation.

## Potential Issues

- **`va_list` fragility:** The macros assume cdecl. If a platform's `int` is not 4 bytes, or stack alignment differs, the `va_start/va_arg` logic breaks silently. Modern code would use `va_copy` and standard-conforming macros.
- **No bounds checking:** Callers to `strlen`, `strcpy`, `strcat` are trusted to pass valid memory. A malicious script could overflow; the VM's `dataMask` is the only safety net.
- **No errno support:** Standard C functions set `errno` on failure. These stubs do not; callers cannot distinguish "atoi(\"abc\") = 0" from "atoi(\"0\") = 0".
- **Single-threaded:** No atomic ops or thread-safe functions. The 2005 codebase assumed single-threaded VMs, though the engine can SMP its renderer.
