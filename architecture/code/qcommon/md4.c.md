# code/qcommon/md4.c

## File Purpose
Implements the RSA Data Security MD4 message-digest algorithm, adapted for use in Quake III Arena's common layer. It provides two engine-facing checksum utilities built on top of the standard MD4 hash primitives.

## Core Responsibilities
- Define MD4 context type and initialize hash state
- Process arbitrary-length byte buffers through the MD4 compression function in 64-byte blocks
- Finalize a hash operation into a 16-byte digest with proper padding and bit-length encoding
- Expose `Com_BlockChecksum` and `Com_BlockChecksumKey` for engine-wide data integrity checks
- Encode/decode between little-endian byte arrays and 32-bit word arrays

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MD4_CTX` | struct | Holds the running hash state: 4×32-bit accumulators, 64-bit bit count, and a 64-byte input buffer |
| `POINTER` | typedef | `unsigned char *` — generic byte pointer used in mem operations |
| `UINT2` | typedef | `unsigned short int` — 16-bit word (defined but unused here) |
| `UINT4` | typedef | `unsigned long int` — 32-bit word used throughout the algorithm |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PADDING` | `static unsigned char[64]` | static | Standard MD4/MD5-style padding block: `0x80` followed by zeros |

## Key Functions / Methods

### MD4Init
- **Signature:** `void MD4Init(MD4_CTX *context)`
- **Purpose:** Resets an MD4 context to initial state.
- **Inputs:** Pointer to an `MD4_CTX` to initialize.
- **Outputs/Return:** void; writes magic constants into `context->state[0..3]` and zeroes `count`.
- **Side effects:** Writes to caller-supplied context.
- **Calls:** None.
- **Notes:** Magic constants `0x67452301`, `0xefcdab89`, `0x98badcfe`, `0x10325476` are the standard MD4 IV.

---

### MD4Update
- **Signature:** `void MD4Update(MD4_CTX *context, const unsigned char *input, unsigned int inputLen)`
- **Purpose:** Ingests an arbitrary-length chunk of data into the running MD4 state, processing complete 64-byte blocks immediately.
- **Inputs:** Active context, input byte buffer, byte count.
- **Outputs/Return:** void; updates `context->state`, `context->count`, `context->buffer`.
- **Side effects:** Modifies context in place; calls `Com_Memcpy` for buffering.
- **Calls:** `Com_Memcpy`, `MD4Transform`.
- **Notes:** Handles buffer carry-over across calls; bit count overflow increments the high word.

---

### MD4Final
- **Signature:** `void MD4Final(unsigned char digest[16], MD4_CTX *context)`
- **Purpose:** Completes the hash, appending standard padding and the 64-bit message length, then writes the 16-byte digest.
- **Inputs:** Output digest buffer (16 bytes), active context.
- **Outputs/Return:** void; writes 16-byte MD4 digest to `digest`.
- **Side effects:** Calls `MD4Update` twice (padding + length), then zeroes the context via `Com_Memset`.
- **Calls:** `Encode`, `MD4Update`, `Com_Memset`.
- **Notes:** Context is zeroized after use to clear sensitive state.

---

### MD4Transform *(static)*
- **Signature:** `static void MD4Transform(UINT4 state[4], const unsigned char block[64])`
- **Purpose:** Core 48-step MD4 compression function; mixes one 64-byte block into the 4-word state using three rounds of 16 operations each.
- **Inputs:** Current 4-word state, one 512-bit (64-byte) message block.
- **Outputs/Return:** void; updates `state[0..3]` in place.
- **Side effects:** Allocates local 64-byte `x[16]` word array; zeroes it via `Com_Memset` before return.
- **Calls:** `Decode`, `Com_Memset`.
- **Notes:** Rounds use macros `FF`/`GG`/`HH` with per-round constants; `x` is zeroized to prevent leaking plaintext on the stack.

---

### Com_BlockChecksum
- **Signature:** `unsigned Com_BlockChecksum(void *buffer, int length)`
- **Purpose:** Computes a 32-bit checksum of a buffer by XOR-folding the 128-bit MD4 digest into a single `unsigned int`.
- **Inputs:** Raw buffer pointer, byte length.
- **Outputs/Return:** `unsigned` — XOR of all four 32-bit digest words.
- **Side effects:** None beyond stack allocation of `MD4_CTX`.
- **Calls:** `MD4Init`, `MD4Update`, `MD4Final`.
- **Notes:** Used for file/data integrity verification in the engine (e.g., pure-server pak checking).

---

### Com_BlockChecksumKey
- **Signature:** `unsigned Com_BlockChecksumKey(void *buffer, int length, int key)`
- **Purpose:** Like `Com_BlockChecksum` but prepends a 4-byte key before the data, producing a keyed checksum.
- **Inputs:** Buffer, byte length, 32-bit key.
- **Outputs/Return:** `unsigned` — XOR-folded keyed digest.
- **Side effects:** None beyond stack allocation.
- **Calls:** `MD4Init`, `MD4Update` (×2), `MD4Final`.
- **Notes:** Used for challenge-response or anti-cheat keyed verification scenarios.

---

### Encode / Decode *(static)*
- **Notes:** `Encode` serializes `UINT4[]` → `unsigned char[]` little-endian; `Decode` does the reverse. Both assume `len` is a multiple of 4.

## Control Flow Notes
This file is a pure utility module — it has no init/frame/shutdown hooks. `Com_BlockChecksum` and `Com_BlockChecksumKey` are called on demand from elsewhere in `qcommon` (e.g., `files.c`, network integrity checks). The MD4 primitives follow a strict Init → Update* → Final sequence.

## External Dependencies
- `<string.h>` — included at top (likely for `memset`/`memcpy` fallbacks).
- `Com_Memset`, `Com_Memcpy` — **defined elsewhere** (`qcommon/common.c`); used in place of `memset`/`memcpy` throughout. Under `__VECTORC` they alias to the standard functions directly.
- `#pragma warning(disable : 4711)` — MSVC-specific; suppresses inline expansion warnings on Windows builds.
