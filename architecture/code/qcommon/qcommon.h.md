# code/qcommon/qcommon.h

## File Purpose
Central shared header for Quake III Arena's engine, declaring all subsystem interfaces shared between the client and server (but not game VM or renderer modules). It acts as the primary include for engine-level code, aggregating messaging, networking, VM, command, cvar, filesystem, memory, and platform abstraction APIs.

## Core Responsibilities
- Declares the `msg_t` bitstream serialization API (read/write primitives, delta compression)
- Declares the network layer: address types, packet I/O, `netchan_t` reliable sequenced channels
- Defines the protocol version, server/client opcode enums (`svc_ops_e`, `clc_ops_e`)
- Declares the Virtual Machine (`vm_t`) lifecycle and call interface
- Declares command buffer (`Cbuf_*`) and command execution (`Cmd_*`) APIs
- Declares the console variable (`Cvar_*`) system
- Declares the virtual filesystem (`FS_*`) with pk3/PAK abstraction
- Declares zone/hunk memory allocators (`Z_Malloc`, `Hunk_*`)
- Declares Adaptive Huffman compression structures and functions
- Declares platform abstraction (`Sys_*`) and client/server frame-loop entry points

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `msg_t` | struct | Bitstream buffer for network message serialization/deserialization |
| `netadrtype_t` | enum | Network address family (IP, IPX, loopback, broadcast, bot) |
| `netadr_t` | struct | Network address (type + IPv4 + IPX + port) |
| `netchan_t` | struct | Reliable sequenced network channel with fragment reassembly |
| `svc_ops_e` | enum | Server-to-client message opcodes |
| `clc_ops_e` | enum | Client-to-server message opcodes |
| `vm_t` | struct (opaque) | Virtual machine instance handle (defined in `vm_local.h`) |
| `vmInterpret_t` | enum | VM execution mode: native DLL, bytecode, or JIT-compiled |
| `sharedTraps_t` | enum | Shared VM trap (syscall) numbers for math/memory ops |
| `field_t` | struct | Console input field with cursor, scroll, and text buffer |
| `memtag_t` | enum | Zone memory allocation tags for tracking/freeing by subsystem |
| `sysEventType_t` | enum | System event types (key, mouse, joystick, console, packet) |
| `sysEvent_t` | struct | Platform event record passed into the engine event loop |
| `node_t` | struct | Adaptive Huffman tree node (doubly-linked list + tree pointers) |
| `huff_t` | struct | Huffman codec state (tree, free list, node pool) |
| `huffman_t` | struct | Paired compressor/decompressor Huffman state |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cvar_modifiedFlags` | `int` | global (extern) | OR-accumulator of flags from all modified cvars since last check |
| `com_developer` … `com_cameraMode` | `cvar_t *` | global (extern) | Core engine cvars (developer mode, dedicated, speed timers, etc.) |
| `cl_paused`, `sv_paused` | `cvar_t *` | global (extern) | Pause state shared between client and server |
| `time_game`, `time_frontend`, `time_backend` | `int` | global (extern) | Per-frame timing buckets for `com_speeds` profiling |
| `com_frameTime`, `com_frameMsec` | `int` | global (extern) | Current frame timestamp and duration |
| `com_errorEntered` | `qboolean` | global (extern) | Reentrancy guard for `Com_Error` |
| `com_journalFile`, `com_journalDataFile` | `fileHandle_t` | global (extern) | Handles for input/output journaling |
| `cl_cdkey` | `char[34]` | global (extern) | CD key string, centralized per bug #470 |
| `clientHuffTables` | `huffman_t` | global (extern) | Huffman tables used for network message compression |
| `demo_protocols[]` | `int[]` | global (extern) | List of compatible protocol versions for demo playback |

## Key Functions / Methods

### MSG_WriteBits / MSG_ReadBits
- Signature: `void MSG_WriteBits(msg_t *msg, int value, int bits)` / `int MSG_ReadBits(msg_t *msg, int bits)`
- Purpose: Core bitfield I/O primitives underlying all typed read/write helpers
- Inputs: Message buffer pointer, value/bit count
- Outputs/Return: Read variant returns the decoded integer value
- Side effects: Advances `msg->bit` and `msg->cursize`; sets `msg->overflowed` on overflow
- Calls: Defined in `msg.c`
- Notes: All higher-level `MSG_Write*`/`MSG_Read*` functions delegate to these

### MSG_WriteDeltaEntity / MSG_ReadDeltaEntity
- Signature: `void MSG_WriteDeltaEntity(msg_t*, entityState_s *from, entityState_s *to, qboolean force)` / `void MSG_ReadDeltaEntity(msg_t*, entityState_t *from, entityState_t *to, int number)`
- Purpose: Delta-encodes entity state changes for bandwidth-efficient snapshot transmission
- Inputs: Base state, new state, entity number
- Side effects: Writes/reads into `msg_t`; large side effect on network bandwidth
- Notes: `force` bypasses the unchanged-field optimization

### VM_Create / VM_Call / VM_Free
- Signature: `vm_t *VM_Create(const char *module, int (*systemCalls)(int *), vmInterpret_t interpret)` / `int QDECL VM_Call(vm_t *vm, int callNum, ...)` / `void VM_Free(vm_t *vm)`
- Purpose: Lifecycle management and invocation for game/cgame/ui virtual machines
- Inputs: Module name (bare, e.g. `"cgame"`), syscall dispatcher, interpretation mode
- Outputs/Return: `VM_Create` returns opaque `vm_t*`; `VM_Call` returns the VM's return value
- Side effects: Allocates/frees significant memory; DLL loading on `VMI_NATIVE`
- Notes: Module name must be bare — no extension or path prefix

### Com_Init / Com_Frame / Com_Shutdown
- Signature: `void Com_Init(char *commandLine)` / `void Com_Frame(void)` / `void Com_Shutdown(void)`
- Purpose: Top-level engine lifecycle — initialization, per-frame dispatch, teardown
- Side effects: `Com_Init` initializes all subsystems; `Com_Frame` drives client + server frames; `Com_Shutdown` frees all resources
- Calls: `CL_Frame`, `SV_Frame`, `Cbuf_Execute`, event loop, memory systems

### Netchan_Transmit / Netchan_Process
- Signature: `void Netchan_Transmit(netchan_t *chan, int length, const byte *data)` / `qboolean Netchan_Process(netchan_t *chan, msg_t *msg)`
- Purpose: Send/receive on a reliable sequenced channel with large-message fragmentation
- Outputs/Return: `Netchan_Process` returns false on out-of-order or duplicate packets
- Side effects: Manages `unsentFragments` state; calls `NET_SendPacket`

### Z_Malloc / Z_TagMalloc / Z_Free / Hunk_AllocateTempMemory
- Purpose: Zone allocator (tagged, 0-filled) and hunk allocator (stack-like, temp memory)
- Notes: `ZONE_DEBUG` macro redirects to debug variants with file/line tracking; `S_Malloc` is for small short-lived allocations only

## Control Flow Notes
This header is included by virtually every engine `.c` file. `Com_Init` bootstraps all subsystems at startup. `Com_Frame` is the main loop body — it pumps the event loop (`Com_EventLoop`), then calls `SV_Frame` and `CL_Frame` with the elapsed milliseconds. `Com_Shutdown` tears down in reverse order. The VM subsystem is initialized once per map load and destroyed on map unload.

## External Dependencies
- `code/qcommon/cm_public.h` → collision model public API (`CM_LoadMap`, `CM_BoxTrace`, etc.)
- `code/qcommon/qfiles.h` (via `cm_public.h`) → on-disk format structures
- `q_shared.h` (implicitly required) → `qboolean`, `vec3_t`, `cvar_t`, `fileHandle_t`, `vmCvar_t`, `usercmd_t`, `entityState_t`, `playerState_t`, `trace_t`, etc. — all defined elsewhere
- All `MSG_*`, `NET_*`, `Netchan_*`, `VM_*`, `Cmd_*`, `Cvar_*`, `FS_*`, `Com_*`, `Sys_*`, `Huff_*`, `Z_*`, `Hunk_*` function bodies are **defined elsewhere** in their respective `.c` files
