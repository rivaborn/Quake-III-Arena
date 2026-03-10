# Architecture Overview

## Repository Shape

```
/
├── code/                        # Runtime engine source tree
│   ├── botlib/                  # Bot navigation and AI library (AAS, pathfinding, AI pipeline)
│   ├── bspc/                    # Offline BSP→AAS compiler (standalone tool, no runtime role)
│   ├── cgame/                   # Client-side game logic VM (QVM/DLL)
│   ├── client/                  # Client engine layer (connection, input, sound, cinematics)
│   ├── game/                    # Server-side game logic VM (QVM/DLL)
│   ├── jpeg-6/                  # Vendored IJG libjpeg-6 (texture loading only)
│   ├── macosx/                  # macOS platform layer (AppKit, CoreGraphics, GLX)
│   ├── null/                    # Headless/stub platform layer (porting aid, dedicated server)
│   ├── q3_ui/                   # Legacy base-Q3A UI VM (QVM/DLL)
│   ├── qcommon/                 # Shared engine core (collision, VM host, filesystem, networking)
│   ├── renderer/                # OpenGL renderer DLL (front-end/back-end, shader system)
│   ├── server/                  # Authoritative server (frame loop, client lifecycle, snapshots)
│   ├── splines/                 # Spline math utilities
│   ├── ui/                      # MissionPack (Team Arena) UI VM (QVM/DLL)
│   ├── unix/                    # Linux/Unix platform layer (X11, OSS, pthreads)
│   └── win32/                   # Win32 platform layer (WGL, DirectInput, DirectSound, Winsock)
├── common/                      # Offline tool shared foundation (no runtime engine role)
├── lcc/                         # LCC C compiler used to compile QVM bytecode
├── libs/                        # External library stubs (cmdlib, jpeg6, pak) for tools
├── q3asm/                       # QVM assembler
├── q3map/                       # BSP map compiler tool
├── q3radiant/                   # Level editor
└── ui/                          # Menu/HUD script data files (menudef.h, *.txt)
```

---

## Major Subsystems

### qcommon (Shared Engine Core)
- **Purpose:** Integration layer and service backbone for the entire engine. Provides all subsystem infrastructure required by both client and server: collision detection, command/cvar execution, virtual filesystem, network messaging and channel management, VM hosting, and foundational memory and error infrastructure.
- **Key directories / files:** `code/qcommon/common.c`, `cmd.c`, `cvar.c`, `files.c`, `cm_load.c`, `cm_trace.c`, `cm_test.c`, `cm_patch.c`, `cm_public.h`, `msg.c`, `net_chan.c`, `huffman.c`, `vm.c`, `vm_interpreted.c`, `vm_x86.c`, `vm_ppc.c`, `unzip.c`, `md4.c`, `qcommon.h`
- **Key responsibilities:**
  - Zone and hunk memory allocation with dual-ended hunk buffer; `longjmp`-based `Com_Error`
  - Console variable lifecycle (`CVAR_ROM`, `CVAR_LATCH`, `CVAR_CHEAT`), config-file serialization
  - Buffered text-based command FIFO, tokenization, handler dispatch
  - Transparent virtual filesystem merging directory trees and `.pk3` ZIP archives with priority ordering and pure-server validation
  - Full BSP collision world: map load, AABB/capsule sweep traces, point content tests, PVS cluster queries, area portal connectivity, Bezier patch collision
  - Bit-level network message serialization, Huffman-compressed bitstreams, delta-compression of `usercmd_t`/`entityState_t`/`playerState_t`
  - Reliable sequenced UDP channel with fragmentation/reassembly
  - VM lifecycle management for up to three QVM instances (cgame, game, ui); dispatch to native DLL, x86/PPC JIT, or software interpreter; sandbox enforcement via `dataMask`
  - MD4 checksums for map and pak verification
- **Key dependencies (other subsystems):**
  - Platform layer (`win32/` or `unix/`) for `Sys_*`, `NET_*` socket primitives, `Sys_LoadDll`
  - `code/client` and `code/server` entry points called from `Com_Frame`
  - `q_shared.c` / `q_math.c` math utilities used throughout CM and VM code

---

### Renderer
- **Purpose:** Complete OpenGL-based rendering module implementing a two-phase front-end/back-end pipeline. Loaded as a swappable DLL; sole public entry point is `GetRefAPI`, which returns a `refexport_t` vtable.
- **Key directories / files:** `code/renderer/tr_local.h`, `tr_public.h`, `tr_init.c`, `tr_main.c`, `tr_backend.c`, `tr_shade.c`, `tr_shader.c`, `tr_bsp.c`, `tr_world.c`, `tr_image.c`, `tr_model.c`, `tr_cmds.c`, `tr_surface.c`, `tr_scene.c`, `tr_curve.c`, `tr_animation.c`, `tr_mesh.c`, `tr_light.c`, `tr_shade_calc.c`, `tr_sky.c`, `tr_shadows.c`, `tr_flares.c`, `tr_marks.c`, `tr_noise.c`, `tr_font.c`, `qgl.h`, `qgl_linked.h`
- **Key responsibilities:**
  - BSP tree traversal using PVS cluster data and frustum planes to mark visible leaves and cull surfaces (`tr_world.c`, `tr_main.c`)
  - Draw-surface collection into a unified sort list keyed by shader, fog, entity, and dlight bits; flush sorted list to back-end command queue
  - Parse, cache, and optimize multi-pass `.shader` definitions; collapse two-pass combos into single multitexture passes; synthesize implicit fallback shaders
  - Execute a double-buffered render command queue against OpenGL with a stateful cache minimizing redundant state changes
  - Convert all surface types (BSP faces, Bézier grids, MD3/MD4 meshes, sprites, rails, beams) into interleaved vertex/index data in the global `tess` (`shaderCommands_t`) buffer
  - Load, resample, gamma-correct, mipmap, and upload all textures; manage image hash-table cache and skins
  - Trilinearly sample per-world light grid for entity ambient/directional lighting; distribute dynamic lights through BSP tree
  - Provide `qgl*`-prefixed wrappers over every OpenGL 1.x entry point — dynamic function pointers on Windows/Linux, compile-time `#define` aliases on statically-linked platforms
  - Optional SMP: front-end (scene traversal, sort) and back-end (GL command execution) run on separate threads synchronized via `GLimp_FrontEndSleep`/`GLimp_WakeRenderer`
- **Key dependencies (other subsystems):**
  - Platform GL layer (`GLimp_*` from `win32/win_glimp.c`, `unix/linux_glimp.c`, or `macosx/macosx_glimp.m`) for window creation, context management, swap buffers, gamma
  - `code/jpeg-6` (`jload.c`) for JPEG texture loading
  - `qcommon` (`refimport_t ri`) for `ri.Hunk_Alloc`, `ri.FS_ReadFile`, `ri.CM_ClusterPVS`, cvar/cmd access
  - `q_shared.c` / `q_math.c` for math utilities

---

### Client
- **Purpose:** Complete client-side engine layer. Manages the connection state machine, drives the per-frame loop (input → network → render → audio), and bridges the core engine to the cgame VM, UI VM, and sound system.
- **Key directories / files:** `code/client/client.h`, `cl_main.c`, `cl_cgame.c`, `cl_ui.c`, `cl_parse.c`, `cl_input.c`, `cl_keys.c`, `cl_console.c`, `cl_scrn.c`, `cl_net_chan.c`, `cl_cin.c`, `keys.h`, `snd_dma.c`, `snd_mix.c`, `snd_mem.c`, `snd_adpcm.c`, `snd_wavelet.c`, `snd_local.h`, `snd_public.h`
- **Key responsibilities:**
  - Connection state machine from `connect` through challenge/authorize handshake to `active`
  - Per-frame loop: input processing, packet send/receive, server message parsing, screen update, audio mixing
  - VM syscall dispatch for cgame (`CL_CgameSystemCalls`) and UI (`CL_UISystemCalls`)
  - `usercmd_t` assembly from `kbutton_t` state with delta compression and rate limiting
  - Inbound `svc_*` server message parsing including delta-compressed snapshots and configstrings
  - `clSnapshot_t` ring buffer management and server-time drift correction
  - XOR obfuscation of outgoing/incoming game packets using challenge-derived rolling key (`cl_net_chan.c`)
  - Software-mixed audio pipeline: asset loading, resampling, ADPCM/wavelet/mu-law compression, per-frame DMA mixing
  - RoQ cinematic player: VQ video decode, YUV→RGB, RLL audio, up to 16 simultaneous handles
  - Demo recording and playback; server browser with `servercache.dat` persistence
- **Key dependencies (other subsystems):**
  - Renderer (`refexport_t re`) for all 2D/3D draw calls
  - `qcommon` for `MSG_*`, `NET_*`, `FS_*`, `VM_Create/Call/Free`, `Netchan_*`
  - cgame VM (`cg_public.h`) and UI VM (`ui_public.h`) hosted via `qcommon/vm.c`
  - Platform DMA layer (`SNDDMA_*` from `win32/win_snd.c` or `unix/linux_snd.c`)
  - Platform streaming I/O (`Sys_BeginStreamedFile`, etc.)
  - `code/server` entry points (`SV_Frame`, `SV_Shutdown`) called directly in listen-server mode

---

### Server
- **Purpose:** Authoritative game simulation host. Owns the server frame loop, all connected client lifecycles, game VM hosting, and routing of all UDP network traffic.
- **Key directories / files:** `code/server/server.h`, `sv_main.c`, `sv_init.c`, `sv_client.c`, `sv_game.c`, `sv_snapshot.c`, `sv_world.c`, `sv_bot.c`, `sv_net_chan.c`, `sv_ccmds.c`, `sv_rankings.c`
- **Key responsibilities:**
  - Authoritative frame simulation via `VM_Call(gvm, GAME_RUN_FRAME, ...)`
  - Client state machine: `CS_FREE → CS_CONNECTED → CS_PRIMED → CS_ACTIVE → CS_ZOMBIE`
  - Game VM hosting: serves all VM→engine system calls (collision, entity queries, configstrings, filesystem, cvar, bot calls) through `SV_GameSystemCalls`
  - Per-client snapshot building: PVS/area culling, delta-encoded entity and playerstate transmission, rate throttling
  - Sector-tree spatial partitioning for entity link/unlink, area queries, swept-box traces, point-contents tests
  - UDP packet routing: connectionless packets (status, info, challenge, connect, rcon) and sequenced in-game packets
  - Bot AI integration: exposes engine services to botlib via `botlib_import_t` vtable; drives per-frame bot AI ticks
  - Operator administration commands; optional GRank async stat reporting
- **Key dependencies (other subsystems):**
  - `qcommon` for `Netchan_*`, `MSG_*`, `CM_*`, `VM_Create/Call/Free`, `FS_*`, `Cvar_*`, `Hunk_*`
  - `code/game` VM (`gvm`) for all game logic
  - `code/botlib` via `botlib_export_t` vtable obtained through `GetBotLibAPI`

---

### Game VM (Server-Side)
- **Purpose:** Server-side game logic module executing as QVM bytecode or native DLL. Owns all authoritative game logic: entity simulation, player physics, combat, item management, team/CTF rules, and the full bot AI stack.
- **Key directories / files:** `code/game/g_local.h`, `g_main.c`, `g_active.c`, `g_client.c`, `g_combat.c`, `g_weapon.c`, `g_missile.c`, `g_items.c`, `g_team.c`, `g_spawn.c`, `g_mover.c`, `g_trigger.c`, `g_target.c`, `g_bot.c`, `g_utils.c`, `g_mem.c`, `g_syscalls.c`, `ai_main.c`, `ai_dmq3.c`, `ai_dmnet.c`, `ai_team.c`, `ai_chat.c`, `bg_pmove.c`, `bg_slidemove.c`, `bg_misc.c`, `bg_lib.c`, `q_shared.c`, `q_math.c`
- **Key responsibilities:**
  - Entity simulation: spawn, think, move, free all server-side entities each frame
  - Player physics: run `Pmove` authoritatively, apply environmental damage, synchronize `playerState_t` → `entityState_t`
  - Combat pipeline: `G_Damage`, `G_RadiusDamage`, death sequencing, scoring, item drops
  - Team and CTF rules: flag lifecycle, obelisk/harvester objectives, team bonuses
  - Bot AI stack: per-bot FSM (`ai_dmnet.c`), `usercmd_t` synthesis via botlib EA layer, bot lifecycle and interbreeding
  - Level loading: parse BSP entity strings, dispatch class-specific spawn functions
  - Client lifecycle: connect, spawn, userinfo, session persistence, disconnect
  - VM/engine boundary: expose `vmMain` as sole engine entry; bridge engine services through `trap_*` wrappers; share physics via `bg_*` layer
- **Key dependencies (other subsystems):**
  - Engine via `trap_*` syscalls: `trap_Trace`, `trap_LinkEntity`, `trap_SetConfigstring`, `trap_BotLib*`
  - `code/botlib` via `trap_BotLib*` syscall range (opcodes 200–599); never linked directly
  - `bg_pmove.c` / `bg_misc.c` compiled identically into both game and cgame VMs for deterministic prediction
  - `q_shared.c` / `q_math.c` for foundational types and math

---

### cgame VM (Client-Side Game)
- **Purpose:** Client-side game logic VM module. Consumes server-delivered snapshots and user input to produce all client-visible output: 3D scene population, 2D HUD rendering, local entity simulation, and client-side movement prediction.
- **Key directories / files:** `code/cgame/cg_main.c`, `cg_local.h`, `cg_public.h`, `cg_syscalls.c`, `cg_snapshot.c`, `cg_predict.c`, `cg_view.c`, `cg_draw.c`, `cg_drawtools.c`, `cg_ents.c`, `cg_players.c`, `cg_playerstate.c`, `cg_event.c`, `cg_weapons.c`, `cg_effects.c`, `cg_localents.c`, `cg_marks.c`, `cg_particles.c`, `cg_servercmds.c`, `cg_consolecmds.c`, `cg_scoreboard.c`, `cg_info.c`, `cg_newdraw.c`, `tr_types.h`
- **Key responsibilities:**
  - Snapshot consumption: advance double-buffered `snapshot_t` pipeline, detect teleports, fire entity/playerstate events
  - Client-side `Pmove` prediction on unacknowledged commands; server-vs-client divergence decay
  - Per-frame packet entity interpolation/extrapolation and type-specific rendering dispatch
  - Player model/skin/animation loading; per-frame skeletal lerp; powerup/flag/effect attachment
  - `EV_*` entity event translation into audio, visual, and HUD feedback
  - 2D HUD composition: status bar, crosshair, team overlays, center prints, scoreboards, lagometer
  - Fixed-size pools for local entities (512), mark decals, and particles (8192)
  - Asset precaching during level init
- **Key dependencies (other subsystems):**
  - Engine via `trap_*` syscalls: renderer, sound, collision model, snapshot/game state, cvar/console
  - `bg_pmove.c` / `bg_misc.c` (shared with game VM, must remain deterministically identical)
  - `q_shared.c` / `q_math.c` for math and string utilities
  - `ui/menudef.h` for owner-draw IDs and `CG_SHOW_*` visibility flags

---

### UI VMs (q3_ui and ui)
- **Purpose:** QVM-hosted menu and user interface modules. `code/q3_ui` is the legacy base-Q3A UI; `code/ui` is the MissionPack (Team Arena) data-driven UI using a script-parsed widget framework. Both communicate with the engine exclusively through indexed `trap_*` syscall ABIs.
- **Key directories / files:** `code/q3_ui/ui_main.c`, `ui_atoms.c`, `ui_qmenu.c`, `ui_local.h`, `ui_syscalls.c`, `ui_gameinfo.c`, `ui_players.c`, `ui_servers2.c`; `code/ui/ui_main.c`, `ui_shared.c`, `ui_shared.h`, `ui_local.h`, `ui_syscalls.c`, `ui_atoms.c`, `ui_players.c`, `ui_gameinfo.c`, `keycodes.h`, `ui_public.h`
- **Key responsibilities:**
  - Expose `vmMain` as sole engine-facing entry point; route all UI commands (init, shutdown, key/mouse events, frame refresh, menu activation)
  - Menu stack management (`UI_PushMenu`/`UI_PopMenu`) with per-frame input dispatch to topmost menu
  - 2D rendering in virtual 640×480 coordinate space via `trap_R_*` syscalls
  - Widget framework (buttons, sliders, spin controls, list boxes, text fields, bitmaps, radio buttons); `code/ui`'s version is script-parsed at runtime from `.menu` files
  - Animated 3D player model previews with skeletal animation and tag attachment
  - Server browser: LAN/internet query, filtering, sorting, favorites persistence
  - Single-player campaign flow: level selection, difficulty, postgame scoring, award tracking
  - `vmCvar_t` batch-registration and per-frame sync; GRank online rankings integration
  - `ui/menudef.h` provides the shared constant vocabulary (widget types, feeder IDs, owner-draw IDs, `CG_SHOW_*`/`UI_SHOW_*` flags, voice-chat strings) consumed by both UI VMs and cgame
- **Key dependencies (other subsystems):**
  - Engine via `trap_*` syscalls: renderer, sound, cvar, filesystem, input, LAN browser
  - `bg_public.h` types from game module; `tr_types.h` from cgame/renderer
  - `q_shared.h` base utilities
  - `ui/menudef.h` (no runtime dependencies; compile-time constants only)

---

### botlib
- **Purpose:** Self-contained bot library implementing the full navigation (AAS), pathfinding, movement, AI decision-making, and elementary action pipeline for bot clients. Exposed to the engine via a versioned `botlib_export_t` function-pointer table; consumes engine services exclusively through `botlib_import_t`.
- **Key directories / files:** `code/botlib/be_interface.c`, `be_aas_def.h`, `be_aas_main.c`, `be_aas_file.c`, `be_aas_bspq3.c`, `be_aas_sample.c`, `be_aas_reach.c`, `be_aas_route.c`, `be_aas_cluster.c`, `be_aas_move.c`, `be_aas_entity.c`, `be_aas_optimize.c`, `be_aas_routealt.c`, `be_ea.c`, `be_ai_move.c`, `be_ai_goal.c`, `be_ai_weap.c`, `be_ai_char.c`, `be_ai_chat.c`, `be_ai_weight.c`, `be_ai_gen.c`, `l_memory.c`, `l_script.c`, `l_precomp.c`, `l_libvar.c`, `l_struct.c`, `l_log.c`, `l_crc.c`
- **Key responsibilities:**
  - Load, validate, and hold the AAS binary world in the global `aasworld` singleton; write it back and serialize routing caches
  - Point-to-area mapping, bounding-box sweep traces, entity-to-area linking, PVS/PHS tests
  - Compute and cache all inter-area reachability links (14+ travel types); Dijkstra-like routing across cluster/portal hierarchy with LRU eviction
  - Simulate client movement (gravity, friction, acceleration, stepping, liquid) for jump arc and reachability validation
  - Per-bot goal selection (fuzzy LTG/NBG scoring), movement FSM (travel-type execution), weapon selection (fuzzy inventory scoring), chat (template matching), and personality (characteristic interpolation)
  - Accumulate per-frame `bot_input_t` and expose it via `EA_GetInput`; `EA_ResetInput` clears for next frame
  - Internal utility stack: memory (`l_memory.c`), logging, libvar config, lexer, preprocessor, struct serialization, CRC
  - Post-load geometry compaction (`be_aas_optimize.c`): strips all non-ladder geometry to minimize footprint
- **Key dependencies (other subsystems):**
  - `botlib_import_t botimport`: file system, memory, BSP traces, PVS, entity state, debug visualization — all provided by the server at runtime
  - `code/game` drives per-bot AI calls each frame through `trap_BotLib*` syscall range; never linked directly to botlib symbols
  - `code/bspc` reuses botlib AAS pipeline code (cluster, reach, optimize) via a stub adapter during offline compilation

---

### bspc (Offline BSP→AAS Compiler)
- **Purpose:** Standalone offline tool. Converts compiled BSP map files from multiple Quake-engine formats (Q1, Q2, Q3, Half-Life, Sin) into AAS binary navigation files consumed by botlib at runtime. Has no runtime role in the game engine.
- **Key directories / files:** `code/bspc/bspc.c`, `qbsp.h`, `aas_create.c`, `aas_store.c`, `aas_file.c`, `be_aas_bspc.c`, `brushbsp.c`, `csg.c`, `map.c`, `map_q3.c`, `aas_areamerging.c`, `aas_facemerging.c`, `aas_edgemelting.c`, `aas_gsubdiv.c`, `aas_prunenodes.c`, `portals.c`, `cfgq3.c`, `l_cmd.c`, `l_mem.c`, `l_poly.c`, `l_threads.c`, `l_bsp_q3.c` (and `_q2`, `_q1`, `_hl`, `_sin`)
- **Key responsibilities:**
  - Multi-format BSP ingestion and normalization into unified `mapbrush_t`/`entity_t` representation
  - CSG and BSP tree construction with optional multithreading
  - BSP→AAS conversion pipeline: face classification → area assignment → edge melting → face merging → area merging → gravitational/ladder subdivision → node pruning
  - AAS geometry packing into final `aas_t` world structure; AAS binary file I/O with endian-swapping and header XOR obfuscation
  - Reachability and cluster computation delegated to botlib via `be_aas_bspc.c` stub adapter
  - Portal generation and leak detection
- **Key dependencies (other subsystems):**
  - `code/botlib`: reuses `be_aas_cluster.c`, `be_aas_optimize.c`, `be_aas_reach.c` directly; `be_aas_bspc.c` provides stub `botlib_import_t` to satisfy botlib's engine abstraction
  - `common/`: `cmdlib`, `mathlib`, `polylib`, `scriplib` from the offline tool foundation
  - Produces `.aas` files consumed at runtime by `code/botlib/be_aas_file.c`

---

### jpeg-6
- **Purpose:** Vendored, Quake III Arena-adapted build of IJG libjpeg-6 (dated August 1995). Provides JPEG decompression for texture loading; integrates with the renderer via `ri.Error`, `ri.Printf`, `ri.Malloc`, `ri.Free`.
- **Key directories / files:** `code/jpeg-6/jload.c`, `jerror.c`, `jmemnobs.c`, `jdatasrc.c`, `jpeglib.h`, `jmorecfg.h`, `jconfig.h`; full pipeline: `jdmarker.c`, `jdinput.c`, `jdhuff.c`, `jdcoefct.c`, `jddctmgr.c`, `jidct*.c`, `jdmainct.c`, `jdsample.c`, `jdmerge.c`, `jdcolor.c`, `jdpostct.c`, `jmemmgr.c`
- **Key responsibilities:**
  - `LoadJPG`: open via `FS_FOpenFileRead`, decode into `Z_Malloc`-allocated RGBA buffer
  - Full JPEG decompression pipeline: marker parsing → entropy decode → IDCT → upsampling → colorspace conversion → pixel output
  - Engine-integrated error (`ri.Error` on fatal) and memory management (`ri.Malloc`/`ri.Free`); no backing store (`jmemnobs.c`)
  - Full JPEG compression pipeline present but used primarily for offline/tool purposes
- **Key dependencies (other subsystems):**
  - `code/renderer` (`refimport_t ri`): error, print, malloc, free
  - `qcommon/files.c`: `FS_FOpenFileRead`, `FS_FCloseFile`
  - Zone allocator (`Z_Malloc`) for output pixel buffers

---

### Platform Layers (win32 / unix / macosx / null)
- **Purpose:** OS-specific implementations of the abstract `Sys_*`, `GLimp_*`, `IN_*`, and `SNDDMA_*` interface contracts. Exactly one platform layer is linked per build.
- **Key directories / files:**
  - `code/win32/`: `win_main.c`, `win_glimp.c`, `win_qgl.c`, `win_gamma.c`, `win_input.c`, `win_wndproc.c`, `win_snd.c`, `win_net.c`, `win_shared.c`, `win_syscon.c`, `glw_win.h`, `win_local.h`
  - `code/unix/`: `unix_main.c`, `linux_glimp.c`, `linux_qgl.c`, `linux_snd.c`, `unix_net.c`, `unix_shared.c`, `linux_joystick.c`, `linux_signals.c`, `linux_common.c`, `linux_local.h`, `unix_glw.h`, `vm_x86.c` (no-op stub)
  - `code/macosx/`: `Q3Controller.m`, `macosx_glimp.m`, `macosx_display.m`, `macosx_input.m`, `macosx_qgl.h`, `CGMouseDeltaFix.m`, `CGPrivateAPI.h`, `macosx_local.h`, `macosx_timers.h`
  - `code/null/`: `null_main.c`, `null_client.c`, `null_glimp.c`, `null_input.c`, `null_net.c`, `null_snddma.c`, `mac_net.c`
- **Key responsibilities:**
  - Process entry point (`WinMain`/`main`/AppKit `quakeMain`), OS event dispatch, game loop
  - Window creation, OpenGL context (WGL/GLX/CGL) lifecycle, fullscreen/windowed switching, gamma ramp management
  - Runtime OpenGL DLL loading and `qgl*` function pointer table population; optional per-call GL trace logging
  - Keyboard, mouse (raw/DGA/DirectInput), joystick input; event queue injection via `Sys_QueEvent`
  - DMA audio output (DirectSound/OSS) driving the portable mixer in `code/client/snd_dma.c`
  - UDP sockets for IP (and IPX on Win32); optional SOCKS5 proxy (Win32)
  - `Sys_*` platform services: millisecond timer, CPU detection, DLL load, path queries, user name
  - `code/null` stubs all interfaces to no-ops for headless/dedicated-server builds and serves as a porting starting point
- **Key dependencies (other subsystems):**
  - `qcommon` for `Com_Init`, `Com_Frame`, `Sys_QueEvent`, `NET_*` declarations
  - `code/renderer` (`tr_local.h`): `glConfig`, `glState`, `ri`, renderer cvars consumed by GL window modules
  - `code/client/snd_dma.c`: `dma_t dma` global written directly by platform DMA drivers

---

### common (Offline Tool Foundation)
- **Purpose:** Shared utility foundation for all offline build tools (q3map, bspc, q3radiant, q3asm). No runtime engine role except `md4.c`, which is also called from `qcommon/files.c`.
- **Key directories / files:** `common/cmdlib.c`, `mathlib.c`, `bspfile.c`, `polylib.c`, `scriplib.c`, `imagelib.c`, `aselib.c`, `trilib.c`, `l3dslib.c`, `threads.c`, `mutex.c`, `md4.c`, `qfiles.h`, `surfaceflags.h`, `polyset.h`
- **Key responsibilities:**
  - Portable tool utility foundation: file I/O, path resolution, error handling, endian conversion, string operations
  - Global BSP lump arrays; BSP file load/write with byte-swap; entity key-value parse/serialize
  - Convex polygon (`winding_t`) library underpinning BSP plane-splitting, portal generation, and CSG
  - 3D mesh import from ASE, Alias `.tri`, and 3DS binary formats into common `polyset_t` representation
  - Image I/O for offline tools: LBM, PCX, BMP, TGA
  - Platform-abstracted multi-threaded work dispatch and mutual exclusion (Win32, OSF1, IRIX, single-threaded fallback)
  - Single authoritative source of `CONTENTS_*`/`SURF_*` flag definitions (`surfaceflags.h`)
- **Key dependencies (other subsystems):**
  - `qfiles.h` BSP lump structs; `surfaceflags.h` brush classification constants
  - Tool `main()` entry points set `gamedir`/`qdir` globals consumed by asset path resolution

---

## Key Runtime Flows

### Initialization

1. **Platform entry point** (`WinMain` in `win32/win_main.c` or `main` in `unix/unix_main.c`) initializes timing and calls `Com_Init` in `qcommon/common.c`.
2. **`Com_Init`** sequentially initializes: `Cmd`, `Cvar`, `FS` (filesystem including `.pk3` indexing), `NET`/`Netchan`, and VM infrastructure.
3. **`CL_Init`** (`code/client/cl_main.c`) registers client cvars and commands; defers renderer, sound, and cgame/UI loading to `CL_StartHunkUsers`, called when a connection or map load begins.
4. **`SV_Init`** (`code/server/sv_main.c`) registers server cvars and operator commands.
5. On **map load** (`SV_SpawnServer`): `CM_LoadMap` deserializes the BSP collision world from `qcommon`; `SV_InitGameProgs` calls `VM_Create(gvm, ...)` to load the game VM; the game VM receives `GAME_INIT` → `G_InitGame` which parses BSP entity strings, initializes botlib (`BotAISetup`), and loads AAS data (`BotAILoadMap` → `BotLibLoadMap` → `AAS_LoadAASFile`).
6. **Renderer init** (`GetRefAPI` → `TR_Init`): `GLimp_Init` creates the GL window and context; platform `QGL_Init` resolves `qgl*` function pointers; `R_InitImages`, `R_InitShaders`, `R_InitSkins`, `R_ModelInit`, `R_InitFreeType` execute in order.
7. **cgame VM init** (`CL_InitCGame` → `VM_Call(cgvm, CG_INIT, ...)`): `cg_main.c` clears global state, registers cvars, parses game state config strings, precaches sounds/graphics/models, and drives the loading screen via `trap_UpdateScreen`.
8. **UI VM init** (`VM_Call(uivm, UI_INIT, ...)`): registers cvars, populates the `displayContextDef_t` vtable, loads arena/bot data files, preloads shared widget assets, and pushes the initial menu screen.

---

### Per-frame / Main Loop

1. **Platform loop** calls `Com_Frame` each tick (driven by `WinMain`/`main`).
2. **`Com_Frame`** (`qcommon/common.c`): drains the OS/network event queue via `Sys_SendKeyEvents`; executes the command buffer (`Cbuf_Execute`); calls `SV_Frame` then `CL_Frame`.
3. **`SV_Frame`** (`code/server/sv_main.c`):
   - Detects client timeouts and zombie cleanup.
   - Calls `VM_Call(gvm, GAME_RUN_FRAME, ...)` → `G_RunFrame`: processes client commands, thinks for all entities, runs `Pmove` per client, advances missiles/movers/items/triggers, drives per-bot FSM (`BotAIStartFrame` → `BotDeathmatchAI` → EA input → `usercmd_t` submission).
   - Calls `SV_BotFrame` for bot AI ticks.
   - Calls `SV_SendClientMessages` → `SV_SendClientSnapshot` per active client (PVS/area cull → delta encode `entityState_t`/`playerState_t` → transmit via `sv_net_chan.c`).
4. **`CL_Frame`** (`code/client/cl_main.c`):
   - `CL_SendCmd` / `CL_WritePacket`: assembles `usercmd_t` from input state; transmits via `cl_net_chan.c`.
   - Network receive loop: `CL_netchan_Process` → `CL_ParseServerMessage` decodes inbound `svc_*` messages, updates snapshot ring buffers.
   - `CL_SetCGameTime`: advances server time with drift correction; triggers cgame snapshot processing.
   - `SCR_UpdateScreen` → `CL_CGameRendering` → `VM_Call(cgvm, CG_DRAW_ACTIVE_FRAME, ...)`:
     - `cg_snapshot.c`: advances snapshot pipeline, fires entity/playerstate events.
     - `cg_predict.c`: runs `Pmove` prediction on unacknowledged commands.
     - `cg_view.c`: computes view origin/angles/FOV; calls `CG_AddPacketEntities`, `CG_AddViewWeapon`, local entity/mark/particle add passes; submits `refdef_t` via `trap_R_RenderScene`.
     - `cg_draw.c` (`CG_DrawActive`): composites all 2D HUD elements.
   - `S_Update` (`code/client/snd_dma.c`): spatializes channels and drives `S_PaintChannels` to fill and submit the DMA buffer.

---

### Shutdown

1. `CL_Shutdown` (`code/client/cl_main.c`): calls `CL_Disconnect` to close any active connection; calls `VM_Free` for cgame and UI VMs; calls `S_Shutdown` and `re.Shutdown` (renderer).
2. `SV_Shutdown` (`code/server/sv_main.c`): transmits `disconnect` to all connected clients; calls `SV_ShutdownGameProgs` → `VM_Free(gvm)` which triggers `GAME_SHUTDOWN` → `G_ShutdownGame` (serializes session data, shuts down botlib via `BotAIShutdown`, submits final ranking stats); notifies master servers; frees client array.
3. `Com_Shutdown` (`qcommon/common.c`): calls `CM_ClearMap` to free BSP collision data; tears down filesystem search paths, network channels, and zone/hunk memory.
4. Platform shutdown (`GLimp_Shutdown`, `QGL_Shutdown`, `SNDDMA_Shutdown`, `WSACleanup`/socket close, gamma restore) executes in the platform layer.

---

## Data & Control Boundaries

- **VM isolation boundary:** All cgame, game, and UI module calls cross an integer-opcode `VM_Call` dispatch point in `qcommon/vm.c`. The QVM interpreter and JIT backends enforce a `dataMask` on all VM memory accesses, preventing bytecode from reading or writing outside the VM's own data segment. Game modules never hold direct pointers into engine-internal structures.
- **Renderer DLL boundary:** The renderer is loaded as a swappable shared library. The engine and renderer exchange data exclusively through the `refexport_t`/`refimport_t` vtables established at `GetRefAPI` time. The renderer's internal types (`tr_local.h`) are not visible to engine-level code.
- **botlib vtable boundary:** The engine never links directly to botlib object code. All botlib calls from the game VM flow through `trap_BotLib*` syscalls (opcodes 200–599) in `code/server/sv_game.c`. Botlib itself consumes engine services exclusively through `botlib_import_t botimport`, established at `GetBotLibAPI` time.
- **`bg_*` shared physics boundary:** `bg_pmove.c`, `bg_slidemove.c`, and `bg_misc.c` are compiled identically into both the game VM and the cgame VM. Changes to these files affect both the authoritative server simulation and client-side prediction simultaneously; a mismatch produces prediction errors.
- **Snapshot delta boundary:** The server and client share no memory at runtime. Game state crosses the network as delta-compressed `entityState_t` and `playerState_t` structures, with the server maintaining per-client `clientSnapshot_t` rings and the client maintaining `clSnapshot_t` rings. No live pointers cross this boundary.
- **AAS file boundary:** botlib's runtime navigation data originates from `.aas` files produced offline by the `bspc` tool. The BSPC tool reuses botlib cluster/reachability code directly via a stub `botlib_import_t`; the runtime engine loads precomputed `.aas` files and never re-runs the offline pipeline.
- **QVM/DLL dual-compile boundary:** cgame, game, and UI modules produce two binary forms from the same source: QVM bytecode (via the lcc/q3asm toolchain) and native shared libraries. The integer-opcode syscall ABI is identical in both; `PASSFLOAT` macro rewrites float arguments to cross the integer-only syscall path.
- **Global state singletons:** `aasworld` (`be_aas_def.h`) is the botlib navigation world singleton; `cm` (`qcommon/cm_local.h`) is the collision model singleton; `tr` and `backEnd` and `tess` (`renderer/tr_local.h`) are the renderer frame-state singletons; `svs`/`sv` (`server.h`) are the server static/per-map singletons; `cls`/`clc`/`cl` (`client.h`) are the three-tier client state singletons. All are global C structs with no locking except where the SMP render thread is active.
- **Hunk allocator ownership:** Collision model lump data and QVM code images are allocated from the permanent low end of the hunk. Per-map game data and routing caches are freed on map change via `CM_ClearMap` and `BotLibLoadMap`. Renderer image/shader data is allocated from the renderer's own hunk slice, freed on `RE_Shutdown`.
- **`CONTENTS_*` / `SURF_*` flag definitions:** Defined authoritatively in `common/surfaceflags.h` and consumed by the engine, all VMs, botlib, and offline tools. Any change requires recompilation of all consumers.

---

## Notable Risks / Hotspots

- **`bg_itemlist` / `inv.h` manual synchronization:** `MODELINDEX_*` constants in `code/game/inv.h` must be kept manually in sync with the ordering of `bg_itemlist[]` in `bg_misc.c`. A mismatch silently corrupts bot item recognition with no compile-time diagnostic.
- **`qasm.h` struct offset coupling:** Byte-offset constants in `code/unix/qasm.h` must be manually synchronized with their C struct counterparts in model, sound, and renderer headers. No compile-time enforcement exists; divergence produces incorrect assembly code at runtime.
- **AAS file header XOR obfuscation:** `be_aas_file.c` applies a lightweight XOR cipher to the AAS file header. This is not cryptographic protection; it is a version-fingerprint mechanism. Forged or corrupted AAS files pass this check trivially.
- **JPEG error handling is a hard crash:** `code/jpeg-6/jerror.c` routes all fatal JPEG decode errors to `ri.Error(ERR_FATAL, ...)`. A corrupt or malformed JPEG texture file in a `.pk3` archive will crash the engine unconditionally with no recovery path.
- **`jmemnobs.c` no-backing-store constraint:** All JPEG working memory must fit in `ri.Malloc`-managed heap. Very large progressive JPEG images with multi-pass quantization could trigger an unconditional fatal error if working memory is insufficient.
- **Deferred reachability computation stalls:** `AAS_ContinueInitReachability` spreads the O(N²) area-pair reachability scan across multiple server frames, but maps with very large AAS worlds may still exhibit perceptible frame stalls during the first seconds after map load.
- **LRU routing cache memory pressure eviction:** `be_aas_route.c` evicts oldest routing cache entries when `AvailableMemory` falls below a threshold. On memory-constrained configurations this can cause thrashing where repeatedly needed routes are evicted and recomputed.
- **Fixed-size global BSP lump arrays in `common/bspfile.c`:** All BSP lump data is loaded into flat global C arrays at `MAX_MAP_*` compile-time limits. Very large maps may silently truncate data or assert at these limits with no runtime-resizing mechanism.
- **`cgame` fixed-size pool exhaustion evicts oldest entries:** When the 512-slot local entity pool is exhausted, `cg_localents.c` evicts the oldest active entry rather than dropping the new allocation. Under heavy effect load this can cause premature disappearance of in-flight visual effects.
- **Private macOS CGS API:** `code/macosx/CGPrivateAPI.h` uses Apple's undocumented `CGSRegisterNotifyProc` for global mouse input. This API is resolved at runtime and carries no forward-compatibility guarantee; it has broken across macOS releases historically.
- **`botlib_export_t` ABI version enforcement:** `GetBotLibAPI` rejects calls with a mismatched `BOTLIB_API_VERSION`. When the game module and botlib are out of sync (e.g., partial rebuild), the mismatch produces a hard shutdown with no graceful fallback.
- **`bg_pmove.c` prediction determinism:** Any floating-point or logic divergence between the game VM's server-side `Pmove` execution and the cgame VM's client-side prediction (e.g., compiler optimization differences between DLL and QVM paths) manifests as visible prediction errors and correction snaps for all players.
- **Vendored libjpeg-6 (1995):** `code/jpeg-6` is IJG release 6 from August 1995 — predating libjpeg-6b and all subsequent security patches. Known decoder vulnerabilities fixed in later libjpeg releases are not present here.
