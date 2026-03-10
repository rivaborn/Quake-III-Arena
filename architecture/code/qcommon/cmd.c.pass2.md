# code/qcommon/cmd.c — Enhanced Analysis

## Architectural Role

`cmd.c` is the **central command bus** of the qcommon layer — everything that executes a text command flows through it. It sits directly below `common.c`'s frame loop and above every game subsystem: client, server, cgame VM, game VM, and UI VM all deposit work here and receive dispatch here. The file implements two orthogonal concerns that are tightly coupled by design: a **priority-aware text FIFO** (`cmd_text`) that smooths command delivery across frames, and a **tokenize-then-dispatch pipeline** (`Cmd_ExecuteString`) that routes individual commands to one of six possible sinks. This dual role — buffer management plus dispatch — makes it the single choke point for all console, bind, and scripted command traffic.

## Key Cross-References

### Incoming (who depends on this file)

- **`cl_keys.c`** — key bind execution calls `Cbuf_AddText` to enqueue bound command strings each frame.
- **`cl_console.c`** — console line submission calls `Cbuf_AddText` or `Cmd_ExecuteString` directly for typed commands.
- **`cl_cgame.c` / `sv_game.c`** — VM syscall ABIs expose `trap_SendConsoleCommand` / `trap_Cmd_*` which funnel into `Cbuf_InsertText` or `Cmd_AddCommand` / `Cmd_RemoveCommand`.
- **`common.c` (`Com_Frame`)** — calls `Cbuf_Execute` once per frame to drain the buffer; calls `Cbuf_AddText` / `Cbuf_InsertText` during `Com_Init` to queue startup configs.
- **`cl_main.c` / `sv_main.c`** — call `Cmd_AddCommand` at init to register subsystem commands; call `Cmd_RemoveCommand` at shutdown.
- **`sv_ccmds.c`** — registers server operator commands via `Cmd_AddCommand`; these become queryable via `Cmd_Argc`/`Cmd_Argv` inside handler bodies.
- **The renderer DLL** (`tr_init.c`) — calls `Cmd_AddCommand` through `ri.Cmd_AddCommand` (the `refimport_t` vtable) to register renderer-specific commands like `imagelist`, `shaderlist`.

### Outgoing (what this file depends on)

- **`cvar.c`** — `Cmd_ExecuteString` calls `Cvar_Command()` as the second dispatch fallback; `Cmd_Vstr_f` calls `Cvar_VariableString`.
- **`cl_cgame.c`** — `CL_GameCommand()` called when no registered handler or cvar matches; routes to the cgame VM.
- **`sv_game.c`** — `SV_GameCommand()` similarly called for server-side game VM dispatch.
- **`cl_ui.c`** — `UI_GameCommand()` as a third VM fallback.
- **`cl_main.c`** — `CL_ForwardCommandToServer()` as the final fallback, sending the raw token string over the network; reads `com_cl_running` / `com_sv_running` globals to gate this.
- **`files.c`** — `Cmd_Exec_f` calls `FS_ReadFile` / `FS_FreeFile` to load `.cfg` scripts into the buffer.
- **`common.c`** — `S_Malloc` / `Z_Free` / `CopyString` for linked-list node allocation in `Cmd_AddCommand` / `Cmd_RemoveCommand`.

## Design Patterns & Rationale

- **Priority-insert FIFO with head-injection.** The buffer is normally append-only (`Cbuf_AddText`) but `Cbuf_InsertText` shifts all existing content right and injects at the head. This is necessary for `exec` and `vstr`: when a config file calls `exec another.cfg`, the inner file's commands must run before the remaining commands in the outer file. The O(n) memmove cost is acceptable because buffer operations are infrequent relative to frame time.

- **MRU promotion on command lookup.** `Cmd_ExecuteString` walks the `cmd_functions` linked list and, on a hit, unlinks the node and re-inserts it at the head. This amortizes lookup cost under the (valid) assumption that the same commands recur in temporal clusters (e.g., `+attack`/`-attack` pairs, movement binds).

- **Six-sink dispatch chain (registered → cvar → cgame → game → UI → forward).** This ordering encodes a policy: engine-registered commands take priority over cvars, which take priority over game-module commands, which take priority over server forwarding. The design avoids any knowledge of VM internals inside cmd.c itself — the VM boundaries are behind opaque function pointers.

- **Zero heap allocation during tokenization.** `cmd_tokenized` and `cmd_argv` are file-static. This is an intentional late-1990s optimization: avoiding `malloc` on the per-command hot path. The tradeoff is that tokenization is not reentrant — a command handler cannot call `Cmd_TokenizeString` without corrupting the in-progress parse.

- **`cmd_cmd` — raw command preservation.** The original untokenized string is copied to `cmd_cmd` before any parsing. This was added specifically for `rcon` (remote console) to allow the server to retransmit the command without recomposing it from tokens, avoiding quote-loss bugs (bugzilla #543 comment in source).

## Data Flow Through This File

```
[Key bind / console / network / startup cfg]
        │  Cbuf_AddText / Cbuf_InsertText
        ▼
  cmd_text[] FIFO (16 KB static)
        │  Cbuf_Execute (once per Com_Frame)
        │  -- extract one line --
        ▼
  Cmd_ExecuteString(line)
        │  Cmd_TokenizeString → cmd_argc, cmd_argv[], cmd_cmd[]
        │
        ├─ cmd_functions list match → xcommand_t handler()
        ├─ Cvar_Command()           → cvar set/get
        ├─ CL_GameCommand()         → cgame VM call
        ├─ SV_GameCommand()         → game VM call
        ├─ UI_GameCommand()         → UI VM call
        └─ CL_ForwardCommandToServer() → UDP packet to server
```

Key state transitions: `cmd_wait > 0` causes `Cbuf_Execute` to return early without consuming lines, decrementing the counter — the buffer accumulates commands across frames until the wait expires.

## Learning Notes

- **`wait` as a scripting primitive.** The `wait` command (and `cmd_wait` counter) is a Quake-era idiom for frame-synchronized scripting. Bind scripts like `+attack ; wait ; -attack` relied on this for timed sequences. Modern engines use event-driven input or animation callbacks instead.

- **The tokenizer's `\"` limitation** is explicitly flagged in a TTimo comment. This means `"foo \"bar\""` would be mishandled — the escape is not interpreted. Any system that needs to pass quoted strings containing embedded quotes (e.g., complex rcon payloads) must use `cmd_cmd` rather than the tokenized `cmd_argv`.

- **No ECS or scene graph here.** This is purely a command-dispatch pattern predating those concepts — closer to the command-object pattern with a global handler registry. The lack of namespacing on command names (all commands share a flat namespace) is a structural limitation that caused real collisions in mods.

- **Static `cmd_args` in `Cmd_Args` / `Cmd_ArgsFrom`** are separate static buffers — calling both in the same expression would be safe, but calling either from a nested command handler would corrupt the in-flight result.

- **VM sandbox boundary.** `Cmd_AddCommand` with a NULL function pointer is used to register forwarding stubs — commands that exist only to be forwarded to the server. This is the mechanism by which client-side command completion works for server-only commands.

## Potential Issues

- **`strcat` in `Cmd_Args` / `Cmd_ArgsFrom`** accumulates into a fixed `MAX_STRING_CHARS` / `BIG_INFO_STRING` static buffer without bounds checking on each append. If individual tokens are large and numerous, silent truncation or overflow is possible.

- **Non-reentrancy of tokenizer state.** Since `cmd_argc`, `cmd_argv`, and `cmd_tokenized` are file-static, any registered command handler that triggers another `Cmd_TokenizeString` call (e.g., via `Cbuf_Execute` recursion through `Cmd_Exec_f`) will silently corrupt the argument array of the outer call. `Cbuf_Execute` partially mitigates this by consuming the buffer before calling the handler, but `EXEC_NOW` in `Cbuf_ExecuteText` can still trigger re-entrance.

- **`cmd_wait` is not frame-rate independent.** A `wait 1` always delays by exactly one `Cbuf_Execute` call, which is tied to `Com_Frame` rate. At high frame rates this is a sub-millisecond delay; scripts that relied on it for timing (common in competitive Quake3 configs) broke with `com_maxfps` changes.
