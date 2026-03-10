# code/unix/linux_common.c

## File Purpose
Provides Linux/GAS-syntax x86 assembly implementations of `Com_Memcpy` and `Com_Memset` as drop-in replacements for the MSVC inline-asm versions in `qcommon/common.c`. The active code path (guarded by `#if 1`) simply delegates to libc `memcpy`/`memset`, while the disabled `#else` branch contains hand-optimized MMX/x86 assembly routines.

## Core Responsibilities
- Supply `Com_Memcpy` and `Com_Memset` as Linux platform overrides
- (Disabled) Implement a 32-byte-unrolled scalar x86 `memcpy` with alignment handling
- (Disabled) Implement an MMX-accelerated `memset` via `_copyDWord` for blocks ≥ 8 bytes
- (Disabled) Implement a software prefetch routine `Com_Prefetch` for read/read-write access patterns
- Convert MSVC `__asm` syntax to GAS `__asm__ __volatile__` with local labels and input/output constraints

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `e_prefetch` | enum | Hint type for `Com_Prefetch`: `PRE_READ`, `PRE_WRITE`, `PRE_READ_WRITE` (disabled branch only) |

## Global / File-Static State
None.

## Key Functions / Methods

### Com_Memcpy
- **Signature:** `void Com_Memcpy(void* dest, const void* src, const size_t count)`
- **Purpose:** Engine-wide memory copy; active path wraps libc `memcpy`.
- **Inputs:** `dest` — destination buffer; `src` — source buffer; `count` — byte count.
- **Outputs/Return:** void; writes to `dest`.
- **Side effects:** Writes to destination memory.
- **Calls:** `memcpy` (active path); `Com_Prefetch` (disabled path).
- **Notes:** Disabled path uses a 32-byte-unrolled x86 scalar loop with separate fallthrough cases for 16-, 8-, 4-, 2-, 1-byte tails. GAS local labels (0:–6:) allow inlining.

### Com_Memset
- **Signature:** `void Com_Memset(void* dest, const int val, const size_t count)`
- **Purpose:** Engine-wide memory fill; active path wraps libc `memset`.
- **Inputs:** `dest` — destination; `val` — fill byte value; `count` — byte count.
- **Outputs/Return:** void; fills `dest`.
- **Side effects:** Writes to destination memory.
- **Calls:** `memset` (active path); `_copyDWord` (disabled path).
- **Notes:** Disabled path fast-paths small fills (< 8 bytes) with scalar asm, then broadcasts the byte value to a dword pattern and calls `_copyDWord` for the bulk, finishing with a 0–3 byte scalar tail.

### _copyDWord *(disabled)*
- **Signature:** `void _copyDWord(unsigned int* dest, const unsigned int constant, const unsigned int count)`
- **Purpose:** MMX-accelerated dword fill; packs `constant` into `mm0` and writes 64 bytes per iteration.
- **Inputs:** `dest` — qword-aligned destination; `constant` — 32-bit fill value; `count` — number of dwords.
- **Outputs/Return:** void.
- **Side effects:** Writes to `dest`; issues `emms` to clear MMX state.
- **Calls:** Nothing (pure inline asm).
- **Notes:** Aligns destination to qword boundary before the main loop; handles 8-, 4-, 2-, 1-dword tail cases; `emms` is mandatory before any FPU use after this call.

### Com_Prefetch *(disabled)*
- **Signature:** `void Com_Prefetch(const void *s, const unsigned int bytes, e_prefetch type)`
- **Purpose:** Software prefetch up to 4 KB into L1 cache by touching each 32-byte cache line.
- **Inputs:** `s` — start address; `bytes` — byte count (clamped to 4096); `type` — hint enum.
- **Outputs/Return:** void.
- **Side effects:** Read-touches cache lines; `PRE_WRITE` is a no-op (no benefit assumed).
- **Notes:** Uses `testb` reads rather than a true `prefetch` instruction, so it works on all Pentium-class CPUs.

## Control Flow Notes
This file is compiled only on Linux/Unix platforms. `Com_Memcpy` and `Com_Memset` are called throughout the engine (renderer, sound, network, game logic) as performance-critical bulk memory primitives. No init/shutdown lifecycle of its own; functions are stateless and called on demand.

## External Dependencies
- `<unistd.h>` — for `size_t`
- `<string.h>` — for `memcpy`, `memset` (active path)
- `Com_Prefetch` declared locally (disabled path); defined in same `#else` block
- `_copyDWord` defined locally (disabled path only)
- `Com_Memcpy` / `Com_Memset` symbols consumed by `qcommon/common.c` and the rest of the engine ("defined here, used everywhere")
