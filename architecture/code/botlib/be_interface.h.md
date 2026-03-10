# code/botlib/be_interface.h

## File Purpose
Declares the global state and external symbols for the botlib interface layer. It defines the central `botlib_globals_t` structure that tracks top-level botlib runtime state, and exposes key extern declarations used across the botlib subsystem.

## Core Responsibilities
- Define the `botlib_globals_t` struct holding library-wide runtime state
- Expose the `botlibglobals` singleton and `botimport` interface externals
- Expose the `bot_developer` flag for conditional debug/verbose behavior
- Declare the `Sys_MilliSeconds` platform timing function
- Gate optional debug fields (`debug`, `goalareanum`, `goalorigin`, `runai`) behind `#ifdef DEBUG`
- Enable `RANDOMIZE` macro to vary bot decision-making behavior

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `botlib_globals_t` | struct (typedef) | Central singleton holding botlib setup state, entity/client limits, global time, and optional debug fields |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botlibglobals` | `botlib_globals_t` | global (extern) | Singleton holding botlib runtime state; defined in `be_interface.c` |
| `botimport` | `botlib_import_t` | global (extern) | Engine-provided import function table (trap functions); defined in `be_interface.c` |
| `bot_developer` | `int` | global (extern) | Non-zero when developer mode is active; gates verbose bot logging |

## Key Functions / Methods

### Sys_MilliSeconds
- Signature: `int Sys_MilliSeconds(void)`
- Purpose: Returns the current system time in milliseconds; used for timing within the botlib.
- Inputs: None
- Outputs/Return: `int` — millisecond timestamp
- Side effects: None (read-only platform call)
- Calls: Not inferable from this file (platform-specific implementation)
- Notes: Declaration only; implementation is platform-specific, likely in `unix_main.c` or `win_main.c`

## Control Flow Notes
This header is included by botlib implementation files (`be_interface.c`, and other `be_*.c` files). The `botlibglobals.botlibsetup` field acts as an initialization guard checked before any botlib operation proceeds. The `float time` field is updated each server frame to give all botlib subsystems a consistent global clock. The `RANDOMIZE` macro is unconditionally defined here, meaning randomized bot behavior is always compiled in; `DEBUG` is commented out, suppressing debug fields in shipping builds.

## External Dependencies
- `botlib_import_t` — defined in `botlib.h` (the engine-to-botlib import function table); used here by extern declaration only
- `vec3_t`, `qboolean` — defined in `q_shared.h`; used only under `#ifdef DEBUG`
- `Sys_MilliSeconds` — implemented in platform-specific system files (not in botlib)
