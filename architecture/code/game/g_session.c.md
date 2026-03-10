# code/game/g_session.c

## File Purpose
Manages persistent client session data in Quake III Arena's server-side game module. Session data survives across level loads and tournament restarts by serializing to and deserializing from cvars at shutdown/reconnect time.

## Core Responsibilities
- Serialize per-client session state to named cvars on game shutdown
- Deserialize per-client session state from cvars on reconnect
- Initialize fresh session data for first-time connecting clients
- Initialize the world session and detect gametype changes across sessions
- Write all connected clients' session data atomically at shutdown

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `clientSession_t` | struct (defined in `g_local.h`) | Persistent per-client data: team, spectator state, wins/losses, team leader flag |
| `gclient_t` | struct | Full client state; contains `sess` (clientSession_t) and `pers` fields |
| `level_locals_t` | struct | Global level state; used here for `level.clients`, `level.maxclients`, `level.newSession`, `level.time` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `level` | `level_locals_t` | global (extern) | Accessed for client array, maxclients, newSession flag, and current time |
| `g_gametype` | `vmCvar_t` | global (extern) | Checked to detect gametype changes between sessions |
| `g_teamAutoJoin` | `vmCvar_t` | global (extern) | Determines auto team assignment on first connect |
| `g_maxGameClients` | `vmCvar_t` | global (extern) | Enforces active player cap in FFA/single-player modes |

## Key Functions / Methods

### G_WriteClientSessionData
- **Signature:** `void G_WriteClientSessionData( gclient_t *client )`
- **Purpose:** Serializes one client's session fields into a cvar named `session<clientIndex>`.
- **Inputs:** Pointer to the client whose session should be saved.
- **Outputs/Return:** void
- **Side effects:** Calls `trap_Cvar_Set` to write a cvar; no memory allocation.
- **Calls:** `va`, `trap_Cvar_Set`
- **Notes:** Uses pointer arithmetic (`client - level.clients`) to compute the client index. Format string encodes 7 integers space-separated.

---

### G_ReadSessionData
- **Signature:** `void G_ReadSessionData( gclient_t *client )`
- **Purpose:** Deserializes session data from the cvar `session<clientIndex>` back into the client's `sess` struct on reconnect.
- **Inputs:** Pointer to the client being reconnected.
- **Outputs/Return:** void
- **Side effects:** Writes fields of `client->sess`; calls `trap_Cvar_VariableStringBuffer`.
- **Calls:** `va`, `trap_Cvar_VariableStringBuffer`, `sscanf`
- **Notes:** Uses intermediate `int` locals for `teamLeader`, `spectatorState`, and `sessionTeam` to safely cast enum/qboolean values post-parse (bk001205/bk010221 bug fixes). If the cvar is absent/empty, `sscanf` will leave fields at indeterminate values.

---

### G_InitSessionData
- **Signature:** `void G_InitSessionData( gclient_t *client, char *userinfo )`
- **Purpose:** Initializes session state for a brand-new client connection (first-time connect, not a reconnect).
- **Inputs:** Client pointer and raw userinfo string from the connecting client.
- **Outputs/Return:** void
- **Side effects:** Writes `client->sess`, calls `BroadcastTeamChange`, calls `G_WriteClientSessionData`.
- **Calls:** `Info_ValueForKey`, `PickTeam`, `BroadcastTeamChange`, `G_WriteClientSessionData`
- **Notes:** Team assignment logic branches on `g_gametype`: team games use auto-join or default to spectator; FFA/SP enforce `g_maxGameClients`; Tournament caps at 2 active players. Sets `spectatorTime = level.time` for queue ordering.

---

### G_InitWorldSession
- **Signature:** `void G_InitWorldSession( void )`
- **Purpose:** Reads the global `session` cvar (which stores the last gametype) and sets `level.newSession = qtrue` if the gametype changed, discarding all old per-client session data.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** May set `level.newSession`; calls `G_Printf`.
- **Calls:** `trap_Cvar_VariableStringBuffer`, `atoi`, `G_Printf`

---

### G_WriteSessionData
- **Signature:** `void G_WriteSessionData( void )`
- **Purpose:** Persists the current gametype to the `session` cvar, then iterates all client slots and writes session data for every connected client.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `trap_Cvar_Set` and `G_WriteClientSessionData` for each connected client.
- **Calls:** `trap_Cvar_Set`, `va`, `G_WriteClientSessionData`
- **Notes:** Only clients with `pers.connected == CON_CONNECTED` are saved; connecting/disconnected clients are skipped.

## Control Flow Notes
- **Init:** `G_InitWorldSession` is called early in game module initialization (before clients connect) to establish whether old session data is valid.
- **Connect:** `G_InitSessionData` is called on first connect; `G_ReadSessionData` is called on reconnect — both are invoked from `g_client.c:ClientConnect`.
- **Shutdown:** `G_WriteSessionData` is called at game shutdown to persist state for the next map/session.
- This file has no per-frame update logic.

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `trap_Cvar_Set`, `trap_Cvar_VariableStringBuffer` — engine syscall stubs (g_syscalls.c)
  - `PickTeam`, `BroadcastTeamChange` — defined in `g_client.c` / `g_cmds.c`
  - `Info_ValueForKey` — defined in `q_shared.c`
  - `va` — defined in `q_shared.c`
  - `level`, `g_gametype`, `g_teamAutoJoin`, `g_maxGameClients` — globals defined in `g_main.c`
