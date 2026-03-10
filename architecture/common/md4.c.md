# common/md4.c

## File Purpose
Implements the RSA Data Security MD4 message-digest algorithm, adapted for use in the Quake III engine. It exposes a single engine-facing utility function (`Com_BlockChecksum`) that produces a 32-bit checksum over an arbitrary memory buffer using MD4 as the underlying hash.

## Core Responsibilities
- Initialize, update, and finalize MD4 hash contexts
- Process 64-byte message blocks through three rounds of bitwise transforms
- Encode/decode between little-endian byte arrays and 32-bit word arrays
- Produce a 128-bit digest, then XOR-fold it into a single 32-bit checksum for engine use

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `MD4_CTX` | struct | Holds running MD4 state: 4×32-bit state words, 64-bit bit count, 64-byte input buffer |
| `POINTER` | typedef (`unsigned char *`) | Generic byte pointer used by memory helpers |
| `UINT2` | typedef (`unsigned short int`) | 16-bit word (defined but unused in this file) |
| `UINT4` | typedef (`unsigned long int`) | 32-bit word used throughout MD4 arithmetic |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `PADDING` | `static unsigned char[64]` | static | Standard MD4/MD5 padding block; first byte `0x80`, rest zeros |

## Key Functions / Methods

### MD4Init
- **Signature:** `void MD4Init(MD4_CTX *context)`
- **Purpose:** Resets a context to the standard MD4 IV before hashing begins.
- **Inputs:** `context` — pointer to caller-allocated `MD4_CTX`
- **Outputs/Return:** void; writes into `*context`
- **Side effects:** None beyond writing to `*context`
- **Calls:** None
- **Notes:** Magic constants `0x67452301`, `0xefcdab89`, `0x98badcfe`, `0x10325476` are the MD4 specification IVs.

---

### MD4Update
- **Signature:** `void MD4Update(MD4_CTX *context, unsigned char *input, unsigned int inputLen)`
- **Purpose:** Absorbs an arbitrary-length input chunk into the running MD4 state, processing complete 64-byte blocks immediately and buffering the remainder.
- **Inputs:** `context` — current hash state; `input` — data bytes; `inputLen` — byte count
- **Outputs/Return:** void; mutates `*context`
- **Side effects:** Calls `memcpy` (CRT); calls `MD4Transform` for each complete 64-byte block
- **Calls:** `memcpy`, `MD4Transform`
- **Notes:** Handles the carry into the high 32 bits of the bit count for inputs > 512 MB.

---

### MD4Final
- **Signature:** `void MD4Final(unsigned char digest[16], MD4_CTX *context)`
- **Purpose:** Pads the message to a 56-mod-64 boundary, appends the original bit length, and outputs the 16-byte digest; zeroes the context afterward.
- **Inputs:** `digest` — 16-byte output buffer; `context` — hash context to finalize
- **Outputs/Return:** 16-byte digest written to `digest`
- **Side effects:** Calls `MD4Update` twice (padding + length); calls `memset` to zero context
- **Calls:** `Encode`, `MD4Update`, `memset`
- **Notes:** Context is zeroized on exit to prevent sensitive data leaking on the stack.

---

### MD4Transform *(static)*
- **Signature:** `static void MD4Transform(UINT4 state[4], unsigned char block[64])`
- **Purpose:** Core compression function; applies 48 operations across three rounds (FF/GG/HH) to mix one 64-byte block into the 4-word state.
- **Inputs:** `state` — 4-word running hash; `block` — 64 raw bytes
- **Outputs/Return:** void; `state[]` updated in place
- **Side effects:** Stack-allocates `x[16]` and zeroes it via `memset` before returning
- **Calls:** `Decode`, `memset`
- **Notes:** Round constants: `0x5a827999` (√2 × 2³⁰), `0x6ed9eba1` (√3 × 2³⁰).

---

### Com_BlockChecksum
- **Signature:** `unsigned Com_BlockChecksum(void *buffer, int length)`
- **Purpose:** Engine-facing API; hashes an arbitrary buffer with MD4 and folds the 128-bit digest to a single unsigned 32-bit value via XOR.
- **Inputs:** `buffer` — data to hash; `length` — byte count
- **Outputs/Return:** XOR of all four 32-bit digest words
- **Side effects:** None visible outside function (stack-only MD4_CTX)
- **Calls:** `MD4Init`, `MD4Update`, `MD4Final`
- **Notes:** Used elsewhere in the engine for BSP/pak integrity checks and demo validation; the XOR fold means collisions are more likely than with the full digest.

---

### Encode / Decode *(static)*
- **Notes:** `Encode` serializes `UINT4[]` → `unsigned char[]` little-endian; `Decode` does the reverse. Both assume `len` is a multiple of 4. Trivial byte-shuffle loops.

## Control Flow Notes
This file is a standalone utility with no frame/tick involvement. `Com_BlockChecksum` is called on demand (e.g., at load time for file verification). There is no init/shutdown registration; callers manage `MD4_CTX` lifetime on the stack.

## External Dependencies
- `<string.h>` — for `memcpy` and `memset` (the private `MD4_memcpy`/`MD4_memset` wrappers are declared but **not defined or called** in this file; the implementation uses CRT directly)
- `Com_BlockChecksum` — defined here, declared/used elsewhere in `qcommon` (defined elsewhere: callers in `common/`, `qcommon/files.c`, etc.)
