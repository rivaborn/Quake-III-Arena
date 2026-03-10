# Subsystem Overview

## Purpose
The `code/client` subsystem is the complete client-side engine layer for Quake III Arena. It manages the connection state machine, drives the per-frame loop (input → network → render → audio), and bridges the core engine to three VM modules: cgame, UI, and the sound system. It owns all functionality between the player's hardware (keyboard, mouse, audio hardware) and the game/renderer VMs.

## Key Files

| File | Role |
|---|---|
| `client.h` | Master header; defines `clientActive_t`, `clientConnection_t`, `clientStatic_t`, `clSnapshot_t`; declares all inter-module prototypes and extern globals |
| `cl_main.c` | Central manager; owns `CL_Frame`, connection state machine, renderer/sound/UI/cgame lifecycle, demo recording/playback, file downloads |
| `cl_cgame.c` | cgame VM bridge; dispatches cgame system calls, drives cgame init/shutdown/rendering, manages time synchronization |
| `cl_ui.c` | UI VM bridge; dispatches UI system calls, manages LAN server browser cache (`servercache.dat`), UI VM lifecycle |
| `cl_parse.c` | Incoming server message parser; decodes `svc_*` opcodes, snapshots, entity delta decompression, configstrings, downloads |
| `cl_input.c` | Input→`usercmd_t` pipeline; tracks `kbutton_t` state, assembles and rate-limits outgoing command packets |
| `cl_keys.c` | Keyboard subsystem; `keys[]` state table, key binding dispatch, console/chat field editing, history ring buffer |
| `cl_console.c` | Developer console; circular text buffer, color-coded rendering, notify overlays, slide animation, chat input modes |
| `cl_scrn.c` | Screen pipeline orchestrator; 640×480 virtual-coordinate utilities, per-frame 2D dispatch, debug graph overlay |
| `cl_net_chan.c` | Client network channel wrapper; XOR obfuscation of outgoing/incoming game packets using challenge-derived rolling key |
| `cl_cin.c` | RoQ cinematic player; VQ video decode, YUV→RGB conversion, RLL audio decode, up to 16 simultaneous handles |
| `keys.h` | Key input interface header; `qkey_t`, `keys[MAX_KEYS]`, `field_t` text input, history buffer, binding API |
| `snd_local.h` | Private sound subsystem header; `sfx_t`, `channel_t`, `dma_t`, `sndBuffer`, `portable_samplepair_t`, all internal prototypes |
| `snd_public.h` | Public sound API; lifecycle, one-shot/looping 3D playback, raw PCM injection, music streaming declarations |
| `snd_dma.c` | Sound system main module; channel allocation, spatialization, looping sounds, music streaming, drives `S_PaintChannels` |
| `snd_mix.c` | Audio mixing pipeline; fills stereo `paintbuffer`, mixes PCM/ADPCM/Wavelet/Mu-Law channels, writes to DMA buffer |
| `snd_mem.c` | Sound memory manager; slab free-list allocator for `sndBuffer` chunks, WAV loader, PCM resampler |
| `snd_adpcm.c` | Intel/DVI ADPCM codec; encodes/decodes 4-bit ADPCM, stores compressed audio in `sndBuffer` linked lists |
| `snd_wavelet.c` | Wavelet/mu-law codec; Daubechies-4 transform, mu-law quantization, `mulawToShort[256]` lookup table |

## Core Responsibilities

- **Connection state machine**: Manages the full lifecycle from `connect` through challenge/authorize handshake to `active`, including disconnect, reconnect, and timeout handling (`cl_main.c`).
- **Per-frame loop**: Drives `CL_Frame` — reads input, sends packets, receives and parses server messages, updates screen and audio, and advances cinematics.
- **VM bridging**: Provides system call dispatch tables for both the cgame VM (`CL_CgameSystemCalls`) and the UI VM (`CL_UISystemCalls`), translating VM requests into engine subsystem calls.
- **Input processing**: Translates keyboard/mouse/joystick events into `usercmd_t` structures via `kbutton_t` state tracking, then serializes and transmits them to the server with delta compression and configurable rate limiting.
- **Server message decoding**: Parses all `svc_*` server-to-client opcodes including delta-compressed snapshots, entity states, game state, configstrings, file downloads, and server commands (`cl_parse.c`).
- **Snapshot and time management**: Maintains `clSnapshot_t` ring buffers, interpolates server time with drift correction (`CL_AdjustTimeDelta`), and exposes current snapshot to cgame.
- **Packet obfuscation**: Applies XOR encoding/decoding to outgoing and incoming game packets using a rolling key derived from `challenge`, `serverCommands`, and `reliableCommands` (`cl_net_chan.c`).
- **Software audio pipeline**: Manages a complete software-mixed sound system — asset loading, resampling, channel spatialization, ADPCM/wavelet/mu-law compression, per-frame paint buffer mixing, and DMA output (`snd_*.c`).
- **Cinematic playback**: Decodes RoQ VQ video and RLL audio into displayable frames, supporting up to 16 simultaneous handles for both full-screen and in-game surface cinematics (`cl_cin.c`).
- **Demo recording/playback**: Writes and replays network message streams to/from demo files, including full gamestate snapshots at recording start.
- **Server browser**: Manages LAN, global, MPlayer, and favorites server lists with persistent cache in `servercache.dat`; drives ping infrastructure (`cl_ui.c`, `cl_main.c`).

## Key Interfaces & Data Flow

**Exposed to other subsystems:**
- `CL_Frame(int msec)` — called by `Com_Frame` (qcommon) each engine tick; entry point for the entire client loop
- `CL_Init()` / `CL_Shutdown()` — called by qcommon at startup and teardown
- `CL_PacketEvent` / `CL_KeyEvent` / `CL_MouseEvent` — event injection points from platform/OS layer
- `S_*` functions (via `snd_public.h`) — sound API consumed by cgame, UI, and cinematic layers
- `SCR_UpdateScreen()` — callable from UI VM and other subsystems to force a frame redraw
- `Key_*` functions (via `keys.h`) — key binding and state query API for console and UI
- `CIN_*` functions — cinematic handle API consumed by cgame and UI VMs

**Consumed from other subsystems:**
- **Renderer** (`re`, `refexport_t` from `tr_public.h`): All 2D/3D draw calls, shader registration, cinematic upload (`re.DrawStretchRaw`, `re.UploadCinematic`), `glConfig` hardware caps
- **qcommon** (`qcommon.h`): `MSG_*` message encoding/decoding, `NET_*` networking, `FS_*` filesystem, `Cvar_*`, `Cmd_*`, `Hunk_*`, `Z_*` memory, `VM_Create/Call/Free`
- **cgame VM** (`cg_public.h`): Receives system calls; called each frame via `VM_Call(cgvm, CG_DRAW_ACTIVE_FRAME, ...)`
- **UI VM** (`ui_public.h`): Receives system calls; called for menu rendering and server browser queries
- **Server** (`sv_*`): `SV_Frame`, `SV_BotFrame`, `SV_Shutdown` called directly when running a local server (listen server mode)
- **botlib** (`botlib.h`, `botlib_export`): Parse context functions bridged to the UI VM for script reading
- **Platform DMA layer** (`SNDDMA_*`): `SNDDMA_Init`, `SNDDMA_GetDMAPos`, `SNDDMA_BeginPainting`, `SNDDMA_Submit` — implemented in `win_snd.c` / `linux_snd.c`
- **Platform streaming I/O** (`Sys_BeginStreamedFile`, `Sys_StreamedRead`, `Sys_EndStreamedFile`): Used by cinematic and music streaming

## Runtime Role

**Init (`CL_Init`):**
- Registers all client cvars and console commands
- Initializes key bindings, console, screen, and cinematic subsystems
- Does not immediately load the renderer or sound; those are deferred to `CL_StartHunkUsers` which is called when a connection or map load begins

**Frame (`CL_Frame`):**
1. `CL_SendCmd` / `CL_WritePacket` — processes input and sends outgoing command packets
2. Network receive loop — calls `CL_netchan_Process` → `CL_ParseServerMessage` to decode inbound messages and update snapshot ring buffers
3. `CL_SetCGameTime` — advances server time with drift correction; triggers cgame snapshot processing
4. `SCR_UpdateScreen` — orchestrates the full 2D+3D render pass:
   - Dispatches to `CL_CGameRendering` (active game), `SCR_DrawCinematic`, or UI VM depending on connection state
   - Draws console overlay, debug graphs, demo recording indicator
5. `S_Update` — spatializes channels and drives `S_PaintChannels` to fill and submit the DMA buffer

**Shutdown (`CL_Shutdown`):**
- Calls `CL_Disconnect` to cleanly close any active connection
- Shuts down cgame and UI VMs (`VM_Free`)
- Calls `S_Shutdown` and renderer shutdown via `re.Shutdown`
- Frees hunk memory allocated for client systems

## Notable Implementation Details

- **Three-tier client state hierarchy**: `clientStatic_t cls` (persists across map loads; owns connection state, renderer handle), `clientConnection_t clc` (per-connection; owns netchan, challenge, download state), and `clientActive_t cl` (per-session; owns snapshot buffers, game state, usercmd history).
- **Large configstring reassembly**: Server commands `bcs0`/`bcs1`/`bcs2` are reassembled from multiple packets before being applied as a single configstring update (`cl_cgame.c:CL_GetServerCommand`).
- **Snapshot ring buffer**: `cl.snapshots[PACKET_BACKUP]` stores delta-compressed server snapshots; cgame interpolates between them using `cl.snap` (latest acknowledged) and `cl.nextSnap`.
- **Packet XOR obfuscation**: The rolling XOR key in `cl_net_chan.c` is seeded from `clc.challenge` combined with server/sequence IDs and reliable command strings — this is obfuscation, not cryptographic security.
- **Sound codec selection**: `snd_mem.c` selects between raw PCM, ADPCM (`snd_adpcm.c`), wavelet (`snd_wavelet.c`), and mu-law based on asset characteristics and cvars; `snd_mix.c` dispatches to the matching decode path at mix time.
- **Virtual 640×480 coordinate system**: `cl_scrn.c` provides `SCR_AdjustFrom640` to map all 2D UI drawing from a fixed virtual resolution to the actual display resolution, used consistently across console, HUD, and cinematic drawing.
- **Platform fast-path mixing**: `snd_mix.c` contains `#ifdef id386` inline x86 assembly and `#ifdef idppc_altivec` AltiVec SIMD paths for the inner mixing loop, with a portable C fallback.
- **cgame camera functions stubbed**: In `cl_cgame.c`, `loadCamera`, `startCamera`, and `getCameraInfo` call sites are present but commented out, indicating an unfinished or removed camera scripting feature.
