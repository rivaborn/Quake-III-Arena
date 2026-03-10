# code/client/cl_input.c — Enhanced Analysis

## Architectural Role

`cl_input.c` is the **client-side input aggregation hub** sitting between two distinct engine layers: the asynchronous hardware event delivery layer (platform-specific `IN_*` callbacks and `CL_MouseEvent`/`CL_JoystickEvent`) and the outbound network pipeline (`CL_Netchan_Transmit` via `cl_net_chan.c`). It is the sole producer of `usercmd_t` records, which flow in two directions simultaneously: to the server over UDP as the authoritative command stream, and locally into `cl.cmds[]` where `cg_predict.c` reads them to run client-side `Pmove` prediction against `bg_pmove.c`. The file therefore directly couples the Client subsystem to both the network layer (qcommon `MSG_*`, `Netchan_*`) and the shared gameplay physics layer (`bg_public.h`, `q_shared.h`).

## Key Cross-References

### Incoming (who depends on this file)

- **`cl_main.c`**: Calls `CL_InitInput` at startup and `CL_SendCmd` each client frame — this is the sole entry point driving the entire pipeline.
- **`cl_keys.c`**: Calls `IN_KeyDown`/`IN_KeyUp` wrappers (e.g. `IN_ForwardDown`) when key events are dispatched from the key-event system; also directly calls `IN_CenterView`.
- **Platform input layers** (`win32/win_input.c`, `unix/linux_joystick.c`): Deliver raw mouse deltas via `CL_MouseEvent` and joystick axis values via `CL_JoystickEvent`.
- **`cg_predict.c`** (cgame VM): Reads `cl.cmds[cl.cmdNumber & CMD_MASK]` — the ring buffer this file fills — to replay unacknowledged commands for prediction.
- **`cl_cgame.c`**: Reads `cl.cgameSensitivity` (written by cgame via `CG_MOUSE_EVENT`) and `cl.cgameUserCmdValue` (weapon selection) in `CL_FinishMove`.

### Outgoing (what this file depends on)

- **qcommon networking** (`msg.c`, `net_chan.c`): `MSG_Init`, `MSG_WriteLong`, `MSG_WriteByte`, `MSG_WriteString`, `MSG_WriteDeltaUsercmdKey`, and `CL_Netchan_Transmit`/`CL_Netchan_TransmitNextFragment` — the full serialization and transmission chain.
- **VM dispatch** (`vm.c`): `VM_Call(uivm, UI_MOUSE_EVENT, ...)` and `VM_Call(cgvm, CG_MOUSE_EVENT, ...)` — mouse events are routed to hosted VMs when key catchers are set.
- **Client state globals** (`cl_main.c`/`client.h`): `cl`, `clc`, `cls` structs are read and mutated throughout; `com_frameTime`, `anykeydown`, `cl_sensitivity`, `m_pitch`, `m_yaw`, `cl_maxpackets`, etc.
- **`cl_net_chan.c`**: `Com_HashKey` and the XOR rolling key derived from `clc.challenge` — packet obfuscation is computed here before transmission.

## Design Patterns & Rationale

**Dual-slot key tracking** (`b->down[0]`, `b->down[1]`): A forward-move could be bound to both `W` and `↑`; the engine must not release the logical button until *both* physical keys are released. This is a deliberate Quake-era idiom to handle key-rebinding correctly. The limit of two is hardcoded and silently drops a third key — adequate for typical bindings but a real limitation.

**Partial-frame time summing** (`b->msec`, `b->downtime`): When both key-down and key-up events arrive in the same frame (a sub-frame press), `CL_KeyState` would otherwise return 0. The downtime/msec accumulation ensures the correct fractional contribution. This was sophisticated for 1999 and is still sound.

**`wasPressed` latch**: The boolean persists across the frame boundary so a tap shorter than one frame still sets the button bit in `cmd->buttons`. This matters most for fire buttons on high-ping connections.

**Double-buffered mouse accumulation** (`cl.mouseIndex ^= 1`): Implements a two-element sliding window for `m_filter` smoothing. An XOR-toggle index is cheaper than a queue and sufficient for a two-tap average.

**Rate limiting via `cl_maxpackets`**: Rather than sending every frame, `CL_ReadyToSendPacket` enforces a maximum packet rate (clamped to `[1, 125]` Hz on LANs). This separates render framerate from network framerate — a foundational netcode decision.

## Data Flow Through This File

```
Platform events (async)
  IN_KeyDown/Up        → kbutton_t globals (active, msec, downtime, wasPressed)
  CL_MouseEvent        → cl.mouseDx[]/cl.mouseDy[]  (or VM_Call to UI/cgame)
  CL_JoystickEvent     → cl.joystickAxis[]

Per-frame (CL_SendCmd):
  CL_CreateNewCommands
    → frame_msec = com_frameTime - old_com_frameTime  (clamped to 200ms)
    → CL_CreateCmd:
        CL_AdjustAngles  → cl.viewangles (keyboard yaw/pitch)
        CL_CmdButtons    → cmd.buttons bitmask + wasPressed clear
        CL_KeyMove       → cmd.forwardmove, rightmove, upmove (keyboard)
        CL_MouseMove     → cl.viewangles or cmd strafe (mouse w/ accel+FOV scale)
        CL_JoystickMove  → cl.viewangles or cmd axes
        CL_FinishMove    → cmd.weapon, cmd.serverTime, cmd.angles (ANGLE2SHORT)
    → cl.cmds[cmdNum & CMD_MASK] = cmd   (ring buffer for prediction + retransmit)

  CL_WritePacket (if rate-limited check passes):
    → reliable cmds flushed as strings
    → last cl_packetdup+1 usercmds delta-encoded via MSG_WriteDeltaUsercmdKey
    → CL_Netchan_Transmit (XOR-obfuscated, Huffman-compressed)
```

Key state transition: view angles are **local client state** updated every frame, never directly networked. The server reconstructs orientation from `cmd.angles` (quantized to 16-bit shorts via `ANGLE2SHORT`) — lossy quantization that limits angular precision to ~0.0055°.

## Learning Notes

- **No input abstraction layer**: All device types (keyboard, mouse, joystick) are handled with separate, ad-hoc code paths in one file. Modern engines (Unity Input System, Unreal Enhanced Input) abstract this behind device-agnostic action mappings. The Q3 approach is direct but requires touching this file for any new device support.
- **View angles as client-local state**: Q3 separates "where the client is looking" (local, updated every render frame) from "what the server authorizes" (usercmds, rate-limited). This is still the standard client-server FPS architecture.
- **`ClampChar` as wire format constraint**: Movement values are clamped to `[-128, 127]` to fit a signed byte in the `usercmd_t`. This means there are exactly 256 discrete movement speeds per axis — fine for the physics model but would be limiting for analog controls.
- **`frame_msec` 200ms cap**: Protects against physics runaway after a hitch (e.g. alt-tab). This same pattern appears in essentially every engine derived from Quake.
- **Redundant command transmission** (`cl_packetdup`): Sending the last N commands again on each packet is the standard UDP reliability technique for input — cheaper than ACK+retransmit for time-sensitive data. Modern FPS games still use this.
- **`cl_maxpackets` separation**: Decoupling render rate from network send rate was novel in 1999 and is now universal. It anticipates modern tick-rate architectures (CSGO at 64/128 Hz, Valorant at 128 Hz).

## Potential Issues

- **Three-key limit on logical buttons**: The `Com_Printf("Three keys down for a button!\n")` and silent return means a legitimate third binding silently fails. Not a bug per se but a documented design limit.
- **Mouse smoothing only averages two frames**: `m_filter` computes `(frame[0] + frame[1]) * 0.5` — a rudimentary filter that adds exactly one frame of latency with no configurable kernel size.
- **`frame_msec` global mutated in `CL_CreateNewCommands`**: `CL_KeyState` reads `frame_msec` during `CL_CreateCmd` — these execute sequentially so there's no race, but the global coupling means `CL_KeyState` cannot be called outside the command-creation context without stale values.
