# code/qcommon/msg.c

## File Purpose
Implements the network message serialization layer for Quake III Arena, providing bit-level read/write primitives over a `msg_t` buffer. It handles both raw out-of-band (OOB) byte-aligned I/O and Huffman-compressed bitstream I/O, and provides delta-compression for `usercmd_t`, `entityState_t`, and `playerState_t` structures.

## Core Responsibilities
- Initialize and manage `msg_t` buffers (normal and OOB modes)
- Write/read individual bits, bytes, shorts, longs, floats, strings, and angles
- Perform Huffman-compressed bit I/O via `msgHuff` global
- Delta-encode/decode `usercmd_t` (with optional XOR key obfuscation)
- Delta-encode/decode `entityState_t` using a static field descriptor table
- Delta-encode/decode `playerState_t` including fixed-size stat/ammo/powerup arrays
- Initialize the Huffman codec from a hardcoded byte-frequency table (`msg_hData`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `msg_t` | struct (defined in qcommon.h) | Network message buffer with bit cursor, size tracking, OOB flag |
| `netField_t` | struct | Descriptor for a single delta-compressed network field: name, struct offset, bit-width (0 = float) |
| `huffman_t` | struct (defined in qcommon.h) | Paired compressor/decompressor Huffman trees |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `msgHuff` | `huffman_t` | static (file) | Single shared Huffman codec instance used for all bitstream messages |
| `msgInit` | `qboolean` | static (file) | One-time initialization guard for `msgHuff` |
| `pcount[256]` | `int[256]` | global | Per-field change counters (partially commented out; used by `MSG_ReportChangeVectors_f`) |
| `oldsize` | `int` | global | Running tally of uncompressed bit cost, for bandwidth analysis |
| `overflows` | `int` | global | Counter of value overflow events in `MSG_WriteBits` |
| `kbitmask[32]` | `int[32]` | global | Precomputed bitmasks for 1–32 bit widths, used by keyed delta reads |
| `entityStateFields[]` | `netField_t[]` | global (file-local) | Ordered field descriptors for `entityState_t` delta coding |
| `playerStateFields[]` | `netField_t[]` | global (file-local) | Ordered field descriptors for `playerState_t` delta coding |
| `msg_hData[256]` | `int[256]` | global | Hardcoded byte-frequency table used to pre-train the Huffman codec |

## Key Functions / Methods

### MSG_Init / MSG_InitOOB
- **Signature:** `void MSG_Init(msg_t *buf, byte *data, int length)` / `...OOB(...)`
- **Purpose:** Zero-initialize a `msg_t` and attach a backing data buffer. OOB variant sets `buf->oob = qtrue` for byte-aligned I/O.
- **Side effects:** Lazily calls `MSG_initHuffman()` on first use.

### MSG_WriteBits
- **Signature:** `void MSG_WriteBits(msg_t *msg, int value, int bits)`
- **Purpose:** Core write primitive. Dispatches to either byte-aligned OOB writes (8/16/32 bits only, little-endian) or Huffman-compressed bit-by-bit output.
- **Inputs:** `bits` can be negative (signed value); `bits == 0` is illegal.
- **Side effects:** Updates `msg->cursize`, `msg->bit`, `msg->overflowed`; increments `oldsize` and `overflows`.
- **Notes:** OOB path for 32-bit has a bug: advances `msg->bit` by 8 instead of 32.

### MSG_ReadBits
- **Signature:** `int MSG_ReadBits(msg_t *msg, int bits)`
- **Purpose:** Core read primitive. Mirrors `MSG_WriteBits`; sign-extends result when `bits` is negative.
- **Side effects:** Advances `msg->readcount` and `msg->bit`.

### MSG_WriteDeltaEntity
- **Signature:** `void MSG_WriteDeltaEntity(msg_t *msg, entityState_s *from, entityState_s *to, qboolean force)`
- **Purpose:** Writes a packetentities delta record. Encodes only changed fields up to the last-changed field index (`lc`). NULL `to` signals entity removal.
- **Calls:** `MSG_WriteBits`, `MSG_WriteByte`
- **Notes:** Asserts that `entityStateFields` count + 1 matches `sizeof(entityState_t)/4`; floats use a compact integer encoding when representable within `FLOAT_INT_BITS` (13 bits).

### MSG_ReadDeltaEntity
- **Signature:** `void MSG_ReadDeltaEntity(msg_t *msg, entityState_t *from, entityState_t *to, int number)`
- **Purpose:** Reads and applies an entity delta; sets `to->number = MAX_GENTITIES-1` on removal.
- **Side effects:** Optionally prints field names/values to console when `cl_shownet >= 2 || == -1`.

### MSG_WriteDeltaPlayerstate / MSG_ReadDeltaPlayerstate
- **Purpose:** Delta-encode/decode `playerState_t`. Struct fields use the same compact float encoding as entities. The four 16-element arrays (`stats`, `persistant`, `ammo`, `powerups`) are encoded with a 16-bit presence bitmask each.
- **Calls:** `MSG_WriteBits`, `MSG_WriteByte`, `MSG_WriteShort`, `MSG_WriteLong`

### MSG_WriteDeltaUsercmdKey / MSG_ReadDeltaUsercmdKey
- **Purpose:** XOR-obfuscated usercmd delta. Key is XOR'd with `to->serverTime` before encoding each field to resist packet replay/spoofing.
- **Notes:** If no field changed, writes a single 0 bit and returns early.

### MSG_initHuffman
- **Signature:** `void MSG_initHuffman(void)`
- **Purpose:** Pre-trains `msgHuff` compressor and decompressor by feeding each byte value `msg_hData[i]` times, establishing static Huffman frequencies derived from real Q3 network traffic.
- **Side effects:** Sets `msgInit = qtrue`; calls `Huff_Init`, `Huff_addRef`.

## Control Flow Notes
- Not part of the per-frame update loop directly; called by the client (`cl_parse.c`) and server (`sv_snapshot.c`, `sv_client.c`) whenever packets are assembled or decoded.
- `MSG_initHuffman` runs once at the first `MSG_Init`/`MSG_InitOOB` call (lazy init).
- The `cl_shownet` cvar reference (`extern`) enables debug printing in read paths without a dependency on client-only headers.

## External Dependencies
- **Includes:** `../game/q_shared.h`, `qcommon.h`
- **Defined elsewhere:** `Huff_Init`, `Huff_addRef`, `Huff_putBit`, `Huff_getBit`, `Huff_offsetTransmit`, `Huff_offsetReceive` (implemented in `huffman.c`); `cl_shownet` cvar (client module); `Com_Error`, `Com_Printf`, `Com_Memset`, `Com_Memcpy` (common); `LittleShort`, `LittleLong` (platform endian macros)
