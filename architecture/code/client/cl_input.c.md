# code/client/cl_input.c

## File Purpose
Translates raw input events (keyboard, mouse, joystick) into `usercmd_t` structures and transmits them to the server each frame. It manages continuous button state tracking and builds the outgoing command packet.

## Core Responsibilities
- Track key/button press and release state via `kbutton_t`, supporting two simultaneous keys per logical button
- Convert key states into fractional movement values scaled by frame time
- Adjust view angles from keyboard and joystick inputs
- Accumulate mouse delta into view angle or movement changes
- Assemble `usercmd_t` per frame from all input sources
- Rate-limit outgoing packets via `cl_maxpackets` and `CL_ReadyToSendPacket`
- Serialize and transmit the command packet with delta-compressed usercmds

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `kbutton_t` | typedef struct (defined in `client.h`) | Tracks which physical keys hold a logical button down, downtime timestamp, accumulated msec, active state, and wasPressed flag |
| `usercmd_t` | typedef struct (defined externally) | Final per-frame command sent to server: angles, movement axes, buttons, weapon, serverTime |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `frame_msec` | `unsigned` | global | Duration of the current frame in milliseconds; used to normalize key fractions |
| `old_com_frameTime` | `int` | global | Previous frame's `com_frameTime`; used to compute `frame_msec` |
| `in_left`, `in_right`, `in_forward`, `in_back`, etc. | `kbutton_t` | global | Logical movement/look buttons |
| `in_buttons[16]` | `kbutton_t[16]` | global | Generic action buttons (attack, use, etc.) |
| `in_mlooking` | `qboolean` | global | Whether mouse-look is currently active |
| `cl_upspeed`, `cl_forwardspeed`, `cl_sidespeed`, `cl_yawspeed`, `cl_pitchspeed`, `cl_run`, `cl_anglespeedkey` | `cvar_t *` | global | Movement/turn speed cvars |

## Key Functions / Methods

### IN_KeyDown
- **Signature:** `void IN_KeyDown( kbutton_t *b )`
- **Purpose:** Records a key press into a `kbutton_t`, supporting up to two simultaneous physical keys per logical button.
- **Inputs:** Pointer to the logical button; `argv(1)` = key number string; `argv(2)` = event timestamp string.
- **Outputs/Return:** None. Mutates `*b`.
- **Side effects:** Sets `b->down[0/1]`, `b->active`, `b->wasPressed`, `b->downtime`.
- **Calls:** `Cmd_Argv`, `atoi`, `Com_Printf`.
- **Notes:** Key number `-1` means typed from console (treated as continuous down). Silently ignores repeat events. Warns if a third key tries to press the same button.

### IN_KeyUp
- **Signature:** `void IN_KeyUp( kbutton_t *b )`
- **Purpose:** Releases a physical key from a `kbutton_t`; accumulates held milliseconds into `b->msec`.
- **Inputs:** Pointer to the logical button; `argv(1)` = key number; `argv(2)` = release timestamp.
- **Outputs/Return:** None. Mutates `*b`.
- **Side effects:** Clears `b->down[0/1]`, sets `b->active = qfalse`, accumulates `b->msec`.
- **Calls:** `Cmd_Argv`, `atoi`.
- **Notes:** If uptime is unavailable, falls back to `frame_msec / 2`. Ignores key-up with no matching key-down (menu pass-through).

### CL_KeyState
- **Signature:** `float CL_KeyState( kbutton_t *key )`
- **Purpose:** Returns the fraction [0, 1] of the current frame the button was held.
- **Inputs:** Pointer to `kbutton_t`.
- **Outputs/Return:** `float` fraction of `frame_msec` the key was active.
- **Side effects:** Resets `key->msec = 0`; updates `key->downtime` if still active.
- **Calls:** None.
- **Notes:** If `key->downtime == 0` while active, assumes the key has been down the entire frame.

### CL_AdjustAngles
- **Signature:** `void CL_AdjustAngles( void )`
- **Purpose:** Applies keyboard yaw/pitch adjustments to `cl.viewangles` based on held keys and frame time.
- **Side effects:** Modifies `cl.viewangles[YAW]` and `cl.viewangles[PITCH]`.
- **Calls:** `CL_KeyState`.

### CL_KeyMove
- **Signature:** `void CL_KeyMove( usercmd_t *cmd )`
- **Purpose:** Sets `cmd->forwardmove`, `cmd->rightmove`, `cmd->upmove` from keyboard state.
- **Side effects:** Reads `in_speed`, `cl_run`, movement `kbutton_t`s; sets `BUTTON_WALKING` in `cmd->buttons`.
- **Calls:** `CL_KeyState`, `ClampChar`.

### CL_MouseMove
- **Signature:** `void CL_MouseMove( usercmd_t *cmd )`
- **Purpose:** Applies accumulated mouse deltas to view angles or strafe movement, with optional smoothing and acceleration.
- **Side effects:** Reads/clears `cl.mouseDx/Dy`, toggles `cl.mouseIndex`, modifies `cl.viewangles` or `cmd->rightmove/forwardmove`.
- **Calls:** `sqrt`, `Com_Printf`, `ClampChar`.

### CL_JoystickMove
- **Signature:** `void CL_JoystickMove( usercmd_t *cmd )`
- **Purpose:** Applies joystick axis values to view angles or movement.
- **Side effects:** Reads `cl.joystickAxis`, modifies `cl.viewangles` and cmd movement fields.
- **Calls:** `ClampChar`.

### CL_CmdButtons
- **Signature:** `void CL_CmdButtons( usercmd_t *cmd )`
- **Purpose:** Packs `in_buttons[]` states into `cmd->buttons` bitmask; also sets `BUTTON_TALK` and `BUTTON_ANY`.
- **Side effects:** Clears `in_buttons[i].wasPressed`; reads `cls.keyCatchers`, `anykeydown`.

### CL_FinishMove
- **Signature:** `void CL_FinishMove( usercmd_t *cmd )`
- **Purpose:** Stamps `cmd` with current weapon, serverTime, and quantized view angles.
- **Side effects:** Reads `cl.cgameUserCmdValue`, `cl.serverTime`, `cl.viewangles`.
- **Calls:** `ANGLE2SHORT`.

### CL_CreateCmd
- **Signature:** `usercmd_t CL_CreateCmd( void )`
- **Purpose:** Orchestrates all input sources into a complete `usercmd_t` for this frame.
- **Calls:** `CL_AdjustAngles`, `CL_CmdButtons`, `CL_KeyMove`, `CL_MouseMove`, `CL_JoystickMove`, `CL_FinishMove`, `SCR_DebugGraph`.

### CL_CreateNewCommands
- **Signature:** `void CL_CreateNewCommands( void )`
- **Purpose:** Computes `frame_msec`, increments `cl.cmdNumber`, and stores the new command into `cl.cmds[]`.
- **Side effects:** Updates `frame_msec`, `old_com_frameTime`, `cl.cmdNumber`, `cl.cmds[cmdNum]`.
- **Notes:** Clamps `frame_msec` to 200ms to avoid runaway movement after hitches. No-ops if `cls.state < CA_PRIMED`.

### CL_ReadyToSendPacket
- **Signature:** `qboolean CL_ReadyToSendPacket( void )`
- **Purpose:** Rate-limits outgoing packets to `cl_maxpackets`; also suppresses sending during demos, cinematics, and downloads.
- **Calls:** `Cvar_Set`, `Sys_IsLANAddress`.

### CL_WritePacket
- **Signature:** `void CL_WritePacket( void )`
- **Purpose:** Serializes reliable commands and delta-compressed `usercmd_t` history into a network message and transmits it.
- **Side effects:** Writes to `cl.outPackets[]`, `clc.lastPacketSentTime`; calls `CL_Netchan_Transmit`.
- **Calls:** `MSG_Init`, `MSG_WriteLong`, `MSG_WriteByte`, `MSG_WriteString`, `MSG_WriteDeltaUsercmdKey`, `CL_Netchan_Transmit`, `CL_Netchan_TransmitNextFragment`, `Com_HashKey`.

### CL_SendCmd
- **Signature:** `void CL_SendCmd( void )`
- **Purpose:** Per-frame entry point: generates new commands then conditionally transmits.
- **Calls:** `CL_CreateNewCommands`, `CL_ReadyToSendPacket`, `CL_WritePacket`.

### CL_InitInput
- **Signature:** `void CL_InitInput( void )`
- **Purpose:** Registers all `+`/`-` console commands for movement and button bindings; initializes `cl_nodelta` and `cl_debugMove` cvars.
- **Calls:** `Cmd_AddCommand`, `Cvar_Get`.

## Control Flow Notes
- Called once at startup: `CL_InitInput` (registers commands/cvars).
- Called every client frame: `CL_SendCmd` → `CL_CreateNewCommands` → `CL_CreateCmd` (aggregates all inputs) → `CL_WritePacket` (if rate allows).
- Input events (key, mouse, joystick) arrive asynchronously via `IN_KeyDown`/`IN_KeyUp`, `CL_MouseEvent`, `CL_JoystickEvent`, which mutate global `kbutton_t` state and `cl.mouseDx/Dy/joystickAxis`.

## External Dependencies
- `client.h` — brings in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- **Defined elsewhere:** `cl`, `clc`, `cls` (global client state structs); `com_frameTime`; `anykeydown`; `cl_sensitivity`, `cl_mouseAccel`, `cl_freelook`, `cl_showMouseRate`, `m_pitch`, `m_yaw`, `m_forward`, `m_side`, `m_filter`, `cl_maxpackets`, `cl_packetdup`, `cl_showSend`, `cl_nodelta`, `sv_paused`, `cl_paused`, `com_sv_running`; `VM_Call`; `uivm`, `cgvm`; `Cmd_Argv`, `Cmd_AddCommand`; `Cvar_Get`, `Cvar_Set`; `MSG_*` family; `CL_Netchan_Transmit`; `SCR_DebugGraph`; `ClampChar`, `VectorCopy`, `SHORT2ANGLE`, `ANGLE2SHORT`; `Sys_IsLANAddress`; `Com_HashKey`; `IN_CenterView`.
