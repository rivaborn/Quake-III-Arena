# code/game/g_svcmds.c — Enhanced Analysis

## Architectural Role

This file provides the **server-console administrative interface** for the Game VM, isolating all operator-only commands (bans, entity listing, team forcing, bot management) from the network-accessible command pipeline. It is the critical bridge between human server administrators and the game engine's player management, handling both real-time admin commands and persistent ban state. The file is not exposed to network clients; all commands here execute only from the dedicated server's local console or RCON with full authorization, making it a security boundary for privilege escalation.

## Key Cross-References

### Incoming (who depends on this file)

- **Server subsystem** (`code/server/sv_ccmds.c`): Calls `ConsoleCommand()` for every server console command, making this the single entry point for admin-level operations
- **Game module init** (`code/game/g_main.c`): Calls `G_ProcessIPBans()` during startup to restore persisted bans from the `g_banIPs` cvar
- **Client connection pipeline** (`code/game/g_client.c` → `ClientConnect`): Calls `G_FilterPacket()` during `ClientConnect` to gate banned IPs before entity allocation
- **qcommon command handler**: The engine's `Cmd_ExecuteString()` chain eventually routes server console commands to `ConsoleCommand()` via the VM syscall boundary

### Outgoing (what this file depends on)

- **Other game modules** (forward dispatching): `SetTeam()` from `g_cmds.c`, `Svcmd_GameMem_f()` from `g_mem.c`, `Svcmd_AddBot_f()` / `Svcmd_BotList_f()` from `g_bot.c`, `Svcmd_AbortPodium_f()` from `g_arenas.c`
- **Cvar/command engine** via `trap_*` syscalls: `trap_Argv`, `trap_Argc`, `trap_Cvar_Set`, `trap_SendConsoleCommand`, `trap_SendServerCommand`
- **Global game state**: `level` (player count, entity count), `g_entities[]` (entity list), `g_filterBan` / `g_banIPs` / `g_dedicated` (persistent config)
- **Standard library**: `atoi`, `strlen`, `strchr`, `Q_strcat`, `Q_stricmp`, `va()` (string formatting)

## Design Patterns & Rationale

1. **Sentinel-based free-list allocation**: IP filters use `0xffffffff` as a sentinel to mark deleted/reusable slots, avoiding dynamic allocation and heap fragmentation. This is memory-efficient for the small, bounded set of bans (max 1024).

2. **Bidirectional cvar persistence**: The `ipFilters[]` in-memory array is kept synchronized with the `g_banIPs` cvar string via `UpdateIPBans()`. This allows bans to survive server restarts while keeping the runtime representation compact (32-bit mask/compare pairs instead of strings).

3. **Bitwise IP address matching**: IP octets are packed into a single `unsigned int` with a parallel mask, enabling efficient wildcard matching (e.g., `192.168.1.*`) via bitwise AND without string parsing on every packet.

4. **Centralized command dispatch**: `ConsoleCommand()` uses a string-match dispatcher pattern to route console commands. This centralizes authorization (server-console-only) and makes it easy to audit what commands are exposed at the console vs. network level.

5. **Lazy forwarding to subsystem handlers**: Most commands simply forward to handlers in other game modules (`g_bot.c`, `g_mem.c`, etc.), reducing coupling and allowing those modules to own their own command logic.

## Data Flow Through This File

```
┌─ Initialization ─────────────────────────────────────────────────┐
│  g_banIPs cvar (persisted, e.g., "192.168.1.* 10.0.0.5")       │
│           │                                                      │
│           ↓ (G_ProcessIPBans called from g_main.c)              │
│  Parse space-delimited IP masks, call AddIP for each           │
│           │                                                      │
│           ↓                                                      │
│  ipFilters[] array populated (up to 1024 entries)              │
└──────────────────────────────────────────────────────────────────┘

┌─ Runtime Ban Management ─────────────────────────────────────────┐
│  Server console: "addip 192.168.5.*"                            │
│           │                                                      │
│           ↓ (handled by ConsoleCommand / Svcmd_AddIP_f)        │
│  StringToFilter parses dotted IP → mask/compare pair          │
│           │                                                      │
│           ↓ (AddIP stores in free slot)                         │
│  ipFilters[i] = {mask, compare}                                │
│           │                                                      │
│           ↓ (UpdateIPBans reconstructs cvar)                    │
│  g_banIPs = "192.168.1.* ... 192.168.5.* ..."                 │
└──────────────────────────────────────────────────────────────────┘

┌─ Connection Filtering ──────────────────────────────────────────┐
│  Incoming client connects (IP "203.0.113.42:27960")            │
│           │                                                      │
│           ↓ (G_FilterPacket called from ClientConnect)         │
│  Parse IP string → unsigned int 203.0.113.42                   │
│           │                                                      │
│           ↓ (linear search ipFilters[0..numIPFilters-1])       │
│  For each filter: if (in & mask) == compare → MATCH            │
│           │                                                      │
│           ↓                                                      │
│  Return: (g_filterBan ? DENY : ALLOW) on match,               │
│          inverse if no match                                    │
│           │                                                      │
│           ↓ (qtrue = block packet, qfalse = allow)             │
│  Connection accepted or rejected at ClientConnect stage        │
└──────────────────────────────────────────────────────────────────┘
```

## Learning Notes

- **Era-specific design**: This file exemplifies early 2000s Q3A administrative patterns—centralized console dispatch, cvar-based persistence, and bounded fixed-size arrays. Modern engines use database backends or separate admin systems, but this demonstrates the "no external dependencies" philosophy of id Tech 3.

- **Privilege isolation**: The segregation of server-console-only commands in a separate dispatch function (`ConsoleCommand` vs. network commands) is a security practice—it ensures sensitive operations like IP banning cannot be triggered by malicious clients exploiting network protocols.

- **Bitwise network thinking**: The IP filtering code shows practical bitwise manipulation for networking (packing octets, masking, wildcard matching), a common idiom in systems code of that era before higher-level abstractions became standard.

- **Cvar as quasi-database**: The pattern of serializing state to a cvar string (e.g., `g_banIPs`) as the persistent layer is unique to Q3A's design. It's simple but limited (size constraints) and demonstrates how engine bootstrapping prioritized simplicity over scale.

- **Stateless command dispatch**: Console commands take no explicit context—they read global state (`level`, `g_entities`) and syscall results. This is typical of 1990s/2000s game engines where explicit dependency injection was not a design principle.

## Potential Issues

- **Cvar string overflow**: The `MAX_CVAR_VALUE_STRING` limit (~1024 bytes) caps the ban list to roughly 20 masks. `UpdateIPBans()` silently truncates if it overflows (line 140–143), potentially losing bans without user notice.

- **Endianness assumption**: `StringToFilter()` casts `byte[4]` to `unsigned*` (lines 117–118). This assumes little-endian layout; the code would fail to match IPs on big-endian architectures (though Q3A primarily targeted x86/x86-64).

- **Linear filter search**: `G_FilterPacket()` iterates all `numIPFilters` entries; with up to 1024 filters, this is O(n) per connection. Hash-table lookup would be O(1), though 1024 entries is small enough that the bottleneck is elsewhere.

- **Sentinel collision risk**: The value `0xffffffff` marks a free slot. If an administrator explicitly bans all IPs (mask=`0x00000000`, compare=`0xffffffff`), the code would misinterpret it as a free slot and skip it. Extremely unlikely in practice, but technically unsound.

- **No input validation after cvar load**: `G_ProcessIPBans()` parses the `g_banIPs` cvar directly. If the cvar is manually corrupted (e.g., malformed tokens), the parsing could silently fail or produce unexpected filter entries without warning.
