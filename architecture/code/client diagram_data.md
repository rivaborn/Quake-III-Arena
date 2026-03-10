# code/client/cl_cgame.c
## File Purpose
This file implements the client-side interface layer between the engine and the cgame VM module. It provides the system call dispatch table that the cgame VM invokes to access engine services, and manages cgame VM lifecycle (init, shutdown, per-frame rendering and time updates).

## Core Responsibilities
- Load, initialize, and shut down the cgame VM (`VM_Create`/`VM_Free`)
- Dispatch all cgame system calls (`CL_CgameSystemCalls`) to appropriate engine subsystems
- Expose client state to cgame: snapshots, user commands, game state, GL config, server commands
- Process server commands destined for cgame (`CL_GetServerCommand`) including large config string reassembly (`bcs0/bcs1/bcs2`)
- Manage configstring updates (`CL_ConfigstringModified`) into `cl.gameState`
- Drive server time synchronization and drift correction (`CL_SetCGameTime`, `CL_AdjustTimeDelta`)
- Trigger cgame rendering each frame (`CL_CGameRendering`)

## External Dependencies
- `client.h` → pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `cg_public.h`, `bg_public.h`, `keys.h`, `snd_public.h`
- `botlib.h` — `botlib_export_t *botlib_export` (defined in `be_interface.c`)
- `cgvm` — `vm_t *` defined in `cl_main.c`
- `re` — `refexport_t` renderer interface (defined in `cl_main.c`)
- Camera functions (`loadCamera`, `startCamera`, `getCameraInfo`) — declared extern, all call sites commented out
- `CM_*`, `S_*`, `FS_*`, `Key_*`, `CIN_*`, `Cbuf_*`, `Cvar_*`, `Cmd_*`, `Hunk_*`, `Sys_*`, `Com_*` — all defined elsewhere in engine subsystems

# code/client/cl_cin.c
## File Purpose
Implements RoQ video cinematic playback for Quake III Arena, handling decoding of RoQ-format video frames (VQ-compressed), YUV-to-RGB color conversion, audio decompression (RLL-encoded mono/stereo), and rendering of cinematics to the screen or in-game surfaces.

## Core Responsibilities
- Parse and decode RoQ video file format (header, codebook, VQ frames, audio packets)
- Perform YUV→RGB(16-bit and 32-bit) color space conversion using precomputed lookup tables
- Decode RLL-encoded audio (mono/stereo variants) into PCM samples and feed to the sound system
- Manage up to 16 simultaneous video handles (`cinTable[MAX_VIDEO_HANDLES]`)
- Build and cache the quad-tree blitting structure for VQ frame rendering
- Handle looping, hold-at-end, in-game shader video, and game-state transitions
- Upload decoded frames to the renderer via `re.DrawStretchRaw` / `re.UploadCinematic`

## External Dependencies
- `client.h`: `cls`, `cl`, `uivm`, `re` (renderer), `com_timescale`, `cl_inGameVideo`, `SCR_AdjustFrom640`, `CL_ScaledMilliseconds`
- `snd_local.h`: `s_rawend`, `s_soundtime`, `s_paintedtime`
- Sound: `S_RawSamples`, `S_Update`, `S_StopAllSounds`
- Filesystem: `FS_FOpenFileRead`, `FS_FCloseFile`, `FS_Read`
- Streaming I/O: `Sys_BeginStreamedFile`, `Sys_EndStreamedFile`, `Sys_StreamedRead`
- Renderer: `re.DrawStretchRaw`, `re.UploadCinematic`
- Memory: `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`
- `glConfig.hardwareType`, `glConfig.maxTextureSize` — hardware capability checks

# code/client/cl_console.c
## File Purpose
Implements the in-game developer console for Quake III Arena, handling text buffering, scrollback, notify overlays, animated slide-in/out drawing, and chat message input modes.

## Core Responsibilities
- Maintain a circular text buffer (`con.text`) for scrollback history
- Handle line wrapping, word wrapping, and color-coded character storage
- Animate console slide open/close via `displayFrac`/`finalFrac` interpolation
- Render the solid console panel, scrollback arrows, version string, and input prompt
- Render transparent notify lines (recent messages) over the game view
- Manage chat input modes (global, team, crosshair target, last attacker)
- Register console-related commands (`toggleconsole`, `clear`, `condump`, etc.)

## External Dependencies
- `client.h` → pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `keys.h`, `cg_public.h`, `bg_public.h`
- **Defined elsewhere:** `cls` (`clientStatic_t`), `cl` (`clientActive_t`), `cgvm`, `re` (renderer exports), `g_consoleField`, `chatField`, `chat_playerNum`, `chat_team`, `historyEditLines`, `g_color_table`, `cl_noprint`, `cl_conXOffset`, `com_cl_running`; renderer entry points `SCR_DrawSmallChar`, `SCR_DrawPic`, `SCR_FillRect`, `Field_Draw`, `Field_BigDraw`, `Field_Clear`, `VM_Call`

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

## External Dependencies
- `client.h` — brings in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- **Defined elsewhere:** `cl`, `clc`, `cls` (global client state structs); `com_frameTime`; `anykeydown`; `cl_sensitivity`, `cl_mouseAccel`, `cl_freelook`, `cl_showMouseRate`, `m_pitch`, `m_yaw`, `m_forward`, `m_side`, `m_filter`, `cl_maxpackets`, `cl_packetdup`, `cl_showSend`, `cl_nodelta`, `sv_paused`, `cl_paused`, `com_sv_running`; `VM_Call`; `uivm`, `cgvm`; `Cmd_Argv`, `Cmd_AddCommand`; `Cvar_Get`, `Cvar_Set`; `MSG_*` family; `CL_Netchan_Transmit`; `SCR_DebugGraph`; `ClampChar`, `VectorCopy`, `SHORT2ANGLE`, `ANGLE2SHORT`; `Sys_IsLANAddress`; `Com_HashKey`; `IN_CenterView`.

# code/client/cl_keys.c
## File Purpose
Implements the client-side keyboard input system for Quake III Arena, managing key bindings, key state tracking, text field editing (console/chat), and dispatching input events to the appropriate subsystem (console, UI VM, cgame VM, or game commands).

## Core Responsibilities
- Maintain the `keys[]` array of key states (down, repeats, binding)
- Translate between key name strings and key numbers (bidirectionally)
- Handle console field and chat field line editing (cursor, scrolling, history)
- Dispatch key-down/key-up events to the correct handler based on `cls.keyCatchers`
- Execute bound commands (immediate and `+button` style with up/down pairing)
- Register `bind`, `unbind`, `unbindall`, `bindlist` console commands
- Write key bindings to config files

## External Dependencies
- **Includes:** `client.h` → `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- **Defined elsewhere:** `Field_Clear`, `Field_CompleteCommand` (likely `cl_console.c`); `Con_PageUp/Down/Top/Bottom/ToggleConsole_f` (`cl_console.c`); `VM_Call` (`vm.c`); `Cbuf_AddText`, `Cmd_AddCommand`, `Cmd_Argc/Argv` (`cmd.c`); `Cvar_Set/VariableValue` (`cvar.c`); `Z_Free`, `CopyString` (memory); `FS_Printf` (`files.c`); `Sys_GetClipboardData` (platform); `SCR_Draw*` (`cl_scrn.c`); `CL_AddReliableCommand` (`cl_main.c`); `cvar_modifiedFlags` (`cvar.c`); `cls`, `clc`, `cgvm`, `uivm` (client globals).

# code/client/cl_main.c
## File Purpose
This is the central client subsystem manager for Quake III Arena, responsible for initializing, running, and shutting down all client-side systems. It drives the per-frame client loop, manages the connection state machine (connecting → challenging → connected → active), and owns demo recording/playback, server discovery, and reliable command queuing.

## Core Responsibilities
- Register and manage all client-side cvars and console commands
- Drive the per-frame `CL_Frame` loop: input, timeout, packet send, screen/audio/cinematic update
- Manage connection lifecycle: connect, disconnect, reconnect, challenge/authorize handshake
- Record and play back demo files (write gamestate snapshot + replayed net messages)
- Handle out-of-band connectionless packets (challenge, MOTD, server status, server list)
- Manage file downloads from server (download queue, temp files, FS restart on completion)
- Initialize and teardown renderer, sound, UI, and cgame subsystems via hunk lifecycle
- Provide server browser ping infrastructure (`cl_pinglist`, `CL_UpdateVisiblePings_f`)

## External Dependencies
- **Includes:** `client.h` (aggregates `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`); `<limits.h>`
- **Defined elsewhere:**
  - `GetRefAPI` — renderer module export
  - `SV_BotFrame`, `SV_Shutdown`, `SV_Frame` — server module
  - `CL_ParseServerMessage` — `cl_parse.c`
  - `CL_SendCmd`, `CL_WritePacket`, `CL_InitInput` — `cl_input.c`
  - `CL_InitCGame`, `CL_ShutdownCGame`, `CL_SetCGameTime` — `cl_cgame.c`
  - `CL_InitUI`, `CL_ShutdownUI` — `cl_ui.c`
  - `CL_Netchan_Process` — `cl_net_chan.c`
  - `S_Init`, `S_Shutdown`, `S_Update`, `S_DisableSounds`, etc. — sound subsystem
  - `Hunk_*`, `CM_*`, `FS_*`, `NET_*`, `Cvar_*`, `Cmd_*`, `MSG_*` — `qcommon`

# code/client/cl_net_chan.c
## File Purpose
Provides the client-side network channel layer, wrapping the core `Netchan_*` functions with client-specific XOR obfuscation for outgoing and incoming game packets. It encodes transmitted messages and decodes received messages using a rolling key derived from the client challenge, server/sequence IDs, and acknowledged command strings.

## Core Responsibilities
- Encode outgoing client messages (bytes after `CL_ENCODE_START`) before transmission
- Decode incoming server messages (bytes after `CL_DECODE_START`) after reception
- Append `clc_EOF` marker before encoding and transmitting
- Delegate fragment transmission to the base `Netchan_TransmitNextFragment`
- Accumulate decoded byte counts in `newsize` for diagnostics/comparison with `oldsize`

## External Dependencies
- `../game/q_shared.h` — base types (`byte`, `qboolean`, `msg_t` fields)
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `Netchan_Transmit`, `Netchan_TransmitNextFragment`, `Netchan_Process`, `MSG_ReadLong`, `MSG_WriteByte`, `LittleLong`, `CL_ENCODE_START`, `CL_DECODE_START`, `MAX_RELIABLE_COMMANDS`, `clc_EOF`
- `client.h` — `clc` (`clientConnection_t`: `challenge`, `serverCommands`, `reliableCommands`)
- `oldsize` — `extern int` defined elsewhere (likely `cl_parse.c`) used for bandwidth comparison

# code/client/cl_parse.c
## File Purpose
Parses incoming server-to-client network messages for Quake III Arena. It decodes the server message stream into snapshots, entity states, game state, downloads, and server commands that the client uses to update its local world representation.

## Core Responsibilities
- Dispatch incoming server messages by opcode (`svc_*`)
- Parse full game state on level load/connection (configstrings + entity baselines)
- Parse delta-compressed snapshots (player state + packet entities)
- Reconstruct entity states via delta decompression from prior frames or baselines
- Handle file download protocol (block-based chunked transfer)
- Store server command strings for deferred cgame execution
- Sync client-side cvars from server `systeminfo` configstring

## External Dependencies
- **Includes:** `client.h` (pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`)
- **Defined elsewhere:** `cl` (`clientActive_t`), `clc` (`clientConnection_t`), `cls` (`clientStatic_t`), `cl_shownet` (cvar), `MSG_Read*` family (msg.c), `FS_*` (files.c), `Cvar_*` (cvar.c), `CL_AddReliableCommand` / `CL_WritePacket` / `CL_NextDownload` / `CL_ClearState` / `CL_InitDownloads` (cl_main.c), `Con_Close` (console), `Info_*` (q_shared.c)

# code/client/cl_scrn.c
## File Purpose
Manages the screen rendering pipeline for the Quake III Arena client, orchestrating the drawing of all 2D screen elements (HUD, console, debug graphs, demo recording indicator) and driving the per-frame refresh cycle. It also provides a set of virtual-resolution drawing utilities used throughout the client and UI code.

## Core Responsibilities
- Initialize screen-related CVars and set the `scr_initialized` flag
- Convert 640×480 virtual coordinates to actual screen resolution
- Draw 2D primitives: filled rectangles, named/handle-based shaders, big/small chars and strings with color codes
- Drive the per-frame screen update, handling stereo rendering and speed profiling
- Dispatch rendering to the appropriate subsystem based on connection state (cinematic, loading, active game, menus)
- Maintain and render the debug/timing graph overlay

## External Dependencies
- **Includes:** `client.h` (transitively pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `cg_public.h`, `bg_public.h`, `keys.h`, `snd_public.h`)
- **Defined elsewhere:** `re` (`refexport_t`), `cls` (`clientStatic_t`), `clc` (`clientConnection_t`), `uivm` (`vm_t *`), `g_color_table`, `com_speeds`, `time_frontend`, `time_backend`, `cl_debugMove`, `VM_Call`, `Con_DrawConsole`, `CL_CGameRendering`, `SCR_DrawCinematic`, `S_StopAllSounds`, `FS_FTell`, `Com_Error`, `Com_DPrintf`, `Com_Memcpy`, `Q_IsColorString`, `ColorIndex`, `Cvar_Get`

# code/client/cl_ui.c
## File Purpose
This file implements the client-side UI virtual machine (VM) bridge layer, providing the system call dispatch table that translates UI module requests into engine function calls. It also manages the UI VM lifecycle (init/shutdown) and maintains the server browser (LAN) data structures with cache persistence.

## Core Responsibilities
- Dispatch all `UI_*` system calls from the UI VM to engine subsystems via `CL_UISystemCalls`
- Initialize and shut down the UI VM (`CL_InitUI`, `CL_ShutdownUI`)
- Provide LAN server list management: add, remove, query, compare, and mark visibility across four server sources (local, mplayer, global, favorites)
- Persist and restore server browser caches to/from `servercache.dat`
- Bridge UI requests to renderer (`re.*`), sound (`S_*`), key system, filesystem, cinematic, and botlib parse contexts
- Expose client/connection state (`GetClientState`, `CL_GetGlconfig`) and config strings to the UI VM

## External Dependencies
- **Includes:** `client.h` (pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`), `../game/botlib.h`
- **Defined elsewhere:** `cls` (`clientStatic_t`), `clc` (`clientConnection_t`), `cl` (`clientActive_t`), `re` (`refexport_t`), `cl_connectedToPureServer`, `cl_cdkey`, `cvar_modifiedFlags`, `VM_Create/Call/Free/ArgPtr`, `NET_*`, `FS_*`, `S_*`, `Key_*`, `CIN_*`, `SCR_UpdateScreen`, `Sys_GetClipboardData`, `Sys_Milliseconds`, `Hunk_MemoryRemaining`, `Com_RealTime`, `CL_CDKeyValidate`, `CL_ServerStatus`, `CL_UpdateVisiblePings_f`, `CL_GetPing*`, `Z_Free`

# code/client/client.h
## File Purpose
Primary header for the Quake III Arena client subsystem. Defines the three core client state structures (`clSnapshot_t`, `clientActive_t`, `clientConnection_t`, `clientStatic_t`) and declares all function prototypes for every client-side module: main, input, parsing, console, screen, cinematics, cgame, UI, and network channel.

## Core Responsibilities
- Defines snapshot representation for server-to-client delta-compressed state
- Defines the three-tier client state hierarchy (active/connection/static)
- Declares all inter-module function interfaces for the client subsystem
- Declares the global extern instances (`cl`, `clc`, `cls`) shared across client modules
- Declares VM handles for cgame and UI modules (`cgvm`, `uivm`, `re`)
- Declares all client-facing cvars as extern pointers
- Pulls in all required subsystem headers (renderer, UI, sound, cgame, shared)

## External Dependencies
- `../game/q_shared.h` — shared math, entity/player state types, `qboolean`, `cvar_t`
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `vm_t`, filesystem, `netadr_t`, `connstate_t`
- `../renderer/tr_public.h` — `refexport_t`, `glconfig_t`, `stereoFrame_t`
- `../ui/ui_public.h` — `uiClientState_t`, `uiMenuCommand_t`
- `keys.h` — `qkey_t`, key binding declarations, `field_t` input fields
- `snd_public.h` — sound system public interface (included but not shown)
- `../cgame/cg_public.h` — cgame public interface, `stereoFrame_t`
- `../game/bg_public.h` — `playerState_t`, `usercmd_t`, pmove shared definitions
- **Defined elsewhere:** `vm_t`, `gameState_t`, `entityState_t`, `playerState_t`, `usercmd_t`, `netchan_t`, all `Netchan_*` base functions, `MSG_*` functions

# code/client/keys.h
## File Purpose
Declares the key input subsystem interface for the Quake III Arena client, defining key state storage, text input field operations, and the public API for key binding management.

## Core Responsibilities
- Defines the `qkey_t` struct representing per-key state (down/repeat/binding)
- Declares the global `keys[MAX_KEYS]` array as the central key state table
- Exposes text input field rendering and event functions for console/chat UI
- Declares the command history ring buffer and active console/chat fields
- Provides the public API for reading, writing, and querying key bindings
- Exposes insert/overstrike mode toggle state

## External Dependencies
- `../ui/keycodes.h` — defines `keyNum_t` enum covering all 256 possible key slots
- `field_t` — declared in `qcommon/qcommon.h` (noted inline by TTimo)
- `fileHandle_t` — defined in `q_shared.h` / `qcommon.h`
- `qboolean` — defined in `q_shared.h`

# code/client/snd_adpcm.c
## File Purpose
Implements Intel/DVI ADPCM (Adaptive Differential Pulse-Code Modulation) audio compression and decompression for Quake III Arena's sound system. It encodes raw PCM audio into a 4-bit-per-sample ADPCM format and decodes it back, and provides the glue functions to store/retrieve ADPCM-compressed sound data in the engine's chunked `sndBuffer` system.

## Core Responsibilities
- Encode 16-bit PCM samples into 4-bit ADPCM nibbles (`S_AdpcmEncode`)
- Decode 4-bit ADPCM nibbles back to 16-bit PCM samples (`S_AdpcmDecode`)
- Calculate memory requirements for ADPCM-compressed sound assets (`S_AdpcmMemoryNeeded`)
- Retrieve decoded samples from a single `sndBuffer` chunk (`S_AdpcmGetSamples`)
- Encode an entire `sfx_t` sound asset into a linked list of `sndBuffer` chunks (`S_AdpcmEncodeSound`)

## External Dependencies
- **Includes:** `snd_local.h` → pulls in `q_shared.h`, `qcommon.h`, `snd_public.h`
- **Defined elsewhere:**
  - `dma` (`dma_t`) — global DMA state providing `dma.speed`
  - `SND_malloc()` — sndBuffer allocator (defined in `snd_mem.c`)
  - `PAINTBUFFER_SIZE`, `SND_CHUNK_SIZE_BYTE` — macros from `snd_local.h`
  - `adpcm_state_t`, `sndBuffer`, `sfx_t`, `wavinfo_t` — types from `snd_local.h`

# code/client/snd_dma.c
## File Purpose
Main control module for the Quake III Arena software-mixed sound system. It manages sound channel allocation, spatialization, looping sounds, background music streaming, and drives the DMA mixing pipeline each frame.

## Core Responsibilities
- Initialize and shut down the sound system via `SNDDMA_*` platform layer
- Register, cache, and evict sound assets (`sfx_t`) from memory
- Allocate and manage `channel_t` slots for one-shot and looping sounds
- Spatialize 3D sound channels using listener position and orientation
- Stream background music from WAV files into the raw sample buffer
- Drive the mixing pipeline (`S_PaintChannels`) each frame via `S_Update_`
- Handle Doppler scaling for looping sounds tied to moving entities

## External Dependencies
- `snd_local.h`: `sfx_t`, `channel_t`, `dma_t`, `loopSound_t`, `SNDDMA_*`, `S_PaintChannels`, `SND_malloc/free`, `S_LoadSound`
- `client.h`: `cls.framecount` (Doppler frame tracking)
- **Defined elsewhere**: `SNDDMA_Init/Shutdown/GetDMAPos/BeginPainting/Submit` (platform layer: `win_snd.c` / `linux_snd.c`), `S_PaintChannels` (`snd_mix.c`), `S_LoadSound` (`snd_mem.c`), `Sys_BeginStreamedFile/StreamedRead/EndStreamedFile` (OS layer), `VectorRotate`, `DistanceSquared` (math), `Com_Milliseconds`, `FS_Read/FOpenFileRead/FCloseFile`

# code/client/snd_local.h
## File Purpose
Private internal header for Quake III Arena's software sound mixing system. It defines all core data structures, buffer layouts, global state declarations, and internal function prototypes used across the sound subsystem's mixing, spatialization, ADPCM compression, and wavelet/mu-law encoding modules.

## Core Responsibilities
- Define sample buffer structures (`sndBuffer`, `portable_samplepair_t`) for the mixing pipeline
- Define the `sfx_t` sound effect asset type with optional compression metadata
- Define `channel_t` for active playback channels with spatialization state
- Define `dma_t` describing the platform DMA output buffer
- Declare all cross-module globals (channels, listener orientation, cvars, raw sample buffer)
- Declare internal API for sound loading, mixing, spatialization, ADPCM, and wavelet codec functions
- Declare platform-abstraction stubs (`SNDDMA_*`) that must be implemented per OS

## External Dependencies
- `q_shared.h` — `vec3_t`, `qboolean`, `cvar_t`, `byte`, `MAX_QPATH`
- `qcommon.h` — `Z_Malloc`/`S_Malloc`, `Cvar_Get`, `FS_ReadFile`, `Com_Printf`
- `snd_public.h` — public sound API declarations consumed by client layer
- `SNDDMA_*` functions — defined elsewhere in platform-specific files (`win_snd.c`, `linux_snd.c`, `snd_null.c`)
- `mulawToShort[]` — defined in `snd_adpcm.c` or `snd_wavelet.c`

# code/client/snd_mem.c
## File Purpose
Implements the sound memory manager and WAV file loader for Quake III Arena's audio system. It manages a fixed-size pool of `sndBuffer` chunks via a free-list allocator, parses WAV headers, and resamples raw PCM audio to match the engine's DMA output rate.

## Core Responsibilities
- Initialize and manage a slab-based free-list allocator for `sndBuffer` chunks
- Parse RIFF/WAV file headers to extract format metadata (`wavinfo_t`)
- Resample PCM audio data (8-bit or 16-bit, mono) from source rate to `dma.speed`
- Load and decode sound assets into `sfx_t` structures, optionally applying ADPCM compression
- Report free/used sound memory statistics

## External Dependencies
- **Includes:** `snd_local.h` → `q_shared.h`, `qcommon.h`, `snd_public.h`
- **Defined elsewhere:**
  - `dma` (`dma_t`) — global DMA state; `dma.speed` used for resampling ratio
  - `S_FreeOldestSound` — eviction policy, defined in `snd_dma.c`
  - `S_AdpcmEncodeSound` — ADPCM encoder, defined in `snd_adpcm.c`
  - `LittleShort` — endian swap macro, from `q_shared.h`
  - `FS_ReadFile`, `FS_FreeFile`, `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`, `Com_Milliseconds`, `Cvar_Get`, `Com_Printf`, `Com_DPrintf`, `Com_Memset` — engine common layer

# code/client/snd_mix.c
## File Purpose
Implements the portable audio mixing pipeline for Quake III Arena's DMA sound system. It reads from active sound channels, mixes them into an intermediate paint buffer, and transfers the result into the platform DMA output buffer.

## Core Responsibilities
- Maintain and fill the stereo `paintbuffer` intermediate mix buffer
- Mix one-shot and looping sound channels into the paint buffer per-frame
- Support four audio decompression paths: raw PCM 16-bit, ADPCM, Wavelet, and Mu-Law
- Apply volume scaling and optional Doppler pitch shifting during mixing
- Transfer the paint buffer to the DMA output buffer with bit-depth/channel-count adaptation
- Provide platform-specific fast paths: x86 inline asm (`id386`) and AltiVec SIMD (`idppc_altivec`)

## External Dependencies
- **`snd_local.h`** — all shared types, channel arrays, DMA state, cvars, and scratch buffer globals
- **`s_channels[MAX_CHANNELS]`**, **`loop_channels`**, **`numLoopChannels`** — defined in `snd_dma.c`
- **`s_paintedtime`**, **`s_rawend`**, **`s_rawsamples`**, **`dma`** — defined in `snd_dma.c`
- **`s_volume`**, **`s_testsound`** — cvars registered in `snd_dma.c`
- **`sfxScratchBuffer`**, **`sfxScratchPointer`**, **`sfxScratchIndex`** — defined in `snd_mem.c`
- **`mulawToShort[256]`** — lookup table defined in `snd_adpcm.c`
- **`S_AdpcmGetSamples`**, **`decodeWavelet`** — defined in `snd_adpcm.c` / `snd_wavelet.c`
- **`S_WriteLinearBlastStereo16`** (Linux x86) — implemented in `unix/snd_mixa.s`
- **`Com_Memset`** — defined in `qcommon`

# code/client/snd_public.h
## File Purpose
Public interface header for the Quake III Arena sound system, exposing all externally callable sound functions to other engine subsystems (client, cgame, etc.). It declares the full lifecycle API for sound playback, looping sounds, spatialization, and background music.

## Core Responsibilities
- Declare sound system initialization and shutdown entry points
- Expose one-shot and looping 3D spatialized sound playback functions
- Provide background music track control (intro + loop)
- Declare raw PCM sample injection for cinematics and VoIP
- Define entity-based position update and reverberation/spatialization calls
- Expose sound registration (asset loading) interface
- Provide utility/diagnostic functions (free memory display, buffer clearing)

## External Dependencies
- `vec3_t`, `qboolean`, `byte` — defined in `q_shared.h`
- `sfxHandle_t` — defined in `q_shared.h` or `snd_local.h`
- All function bodies defined in `snd_dma.c`, `snd_mix.c`, `snd_mem.c` (and platform DMA backends)

# code/client/snd_wavelet.c
## File Purpose
Implements wavelet-based and mu-law audio compression/decompression for Quake III's sound system. It encodes PCM audio data into compact `sndBuffer` chunks using either a Daubechies-4 wavelet transform followed by mu-law quantization, or mu-law encoding alone with dithered error feedback.

## Core Responsibilities
- Apply forward/inverse Daubechies-4 (daub4) wavelet transform to float sample arrays
- Drive multi-resolution wavelet decomposition/reconstruction via `wt1`
- Encode 16-bit PCM samples to 8-bit mu-law bytes (`MuLawEncode`)
- Decode 8-bit mu-law bytes back to 16-bit PCM (`MuLawDecode`)
- Build and cache the `mulawToShort[256]` lookup table on first use
- Compress an `sfx_t` sound asset into linked `sndBuffer` chunks (`encodeWavelet`, `encodeMuLaw`)
- Decompress `sndBuffer` chunks back to PCM for mixing (`decodeWavelet`, `decodeMuLaw`)

## External Dependencies
- `snd_local.h` — `sfx_t`, `sndBuffer`, `SND_CHUNK_SIZE`, `SND_malloc`, `NXStream`, `qboolean`, `byte`, `short`
- `SND_malloc` — defined in `snd_mem.c`
- `myftol` — declared but not called in this file; defined elsewhere (platform float-to-long helper)
- `numBits[256]` — file-static lookup table for bit-count of byte values

