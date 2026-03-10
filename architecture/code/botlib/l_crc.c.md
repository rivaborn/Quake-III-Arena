# code/botlib/l_crc.c

## File Purpose
Implements a 16-bit CCITT CRC (XMODEM variant) using polynomial 0x1021 for data integrity verification within the botlib subsystem. It provides both stateful (incremental) and stateless (one-shot) CRC computation over byte sequences.

## Core Responsibilities
- Initialize a CRC accumulator to the standard CCITT seed value (`0xffff`)
- Process individual bytes into a running CRC value via table lookup
- Process a complete byte string in one call, returning a finalized CRC
- Support incremental/continuation CRC computation across multiple string segments
- Finalize a CRC by XOR-ing with `CRC_XOR_VALUE` (0x0000, effectively a no-op here)

## Key Types / Data Structures
None (no structs or typedefs defined in this file).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `crctable` | `unsigned short[257]` | global | Precomputed 256-entry CCITT CRC lookup table (257th entry is unused padding) |

## Key Functions / Methods

### CRC_Init
- **Signature:** `void CRC_Init(unsigned short *crcvalue)`
- **Purpose:** Seeds a CRC accumulator with the CCITT initial value `0xffff`.
- **Inputs:** `crcvalue` — pointer to the CRC state variable.
- **Outputs/Return:** None (modifies `*crcvalue` in place).
- **Side effects:** Writes to caller-owned memory.
- **Calls:** None.
- **Notes:** Must be called before any `CRC_ProcessByte` or `CRC_ContinueProcessString` call.

### CRC_ProcessByte
- **Signature:** `void CRC_ProcessByte(unsigned short *crcvalue, byte data)`
- **Purpose:** Folds one byte into the running CRC using table lookup.
- **Inputs:** `crcvalue` — current CRC state; `data` — byte to process.
- **Outputs/Return:** None (updates `*crcvalue` in place).
- **Side effects:** Reads `crctable`; writes to caller-owned memory.
- **Calls:** None.
- **Notes:** No bounds check on table index; the shift/XOR operation guarantees the index stays within `[0, 255]` by construction (`*crcvalue >> 8` on a 16-bit value is always 0–255).

### CRC_Value
- **Signature:** `unsigned short CRC_Value(unsigned short crcvalue)`
- **Purpose:** Finalizes a CRC by applying the terminal XOR (`CRC_XOR_VALUE = 0x0000`), which is a no-op in this configuration.
- **Inputs:** `crcvalue` — accumulated CRC.
- **Outputs/Return:** Finalized `unsigned short` CRC.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Preserved for CCITT API completeness; the XOR value could be changed to `0xffff` for full CCITT compliance.

### CRC_ProcessString
- **Signature:** `unsigned short CRC_ProcessString(unsigned char *data, int length)`
- **Purpose:** Computes a complete CRC over a byte buffer in one call.
- **Inputs:** `data` — input buffer; `length` — number of bytes.
- **Outputs/Return:** Finalized `unsigned short` CRC.
- **Side effects:** None beyond reading `crctable`.
- **Calls:** `CRC_Init`, `CRC_Value`.
- **Notes:** Contains a redundant bounds check (`ind < 0 || ind > 256`) that clamps to 0 on invalid index — this can never trigger given the shift semantics, and the comment `FIXME: byte swap?` suggests endian-correctness was a concern. The table index here is computed manually (duplicating `CRC_ProcessByte` logic) rather than calling that function.

### CRC_ContinueProcessString
- **Signature:** `void CRC_ContinueProcessString(unsigned short *crc, char *data, int length)`
- **Purpose:** Appends additional data to an already-initialized CRC accumulator, enabling multi-segment CRC computation.
- **Inputs:** `crc` — existing CRC state (caller must have called `CRC_Init`); `data` — next segment; `length` — byte count.
- **Outputs/Return:** None (updates `*crc` in place).
- **Side effects:** Reads `crctable`; writes to caller-owned memory.
- **Calls:** None.
- **Notes:** Accepts `char *` (signed) while `CRC_ProcessString` takes `unsigned char *`; sign-extension could corrupt the table index for bytes > 127 on platforms where `char` is signed — potential latent bug.

## Control Flow Notes
This file is a pure utility module with no engine lifecycle hooks (no init/frame/shutdown integration of its own). It is called on demand by other botlib modules (e.g., for AAS file integrity checks). It does not interact with the frame loop.

## External Dependencies
- `<stdlib.h>`, `<stdio.h>`, `<string.h>` — standard C library (included but not directly used in function bodies).
- `../game/q_shared.h` — provides the `byte` typedef.
- `../game/botlib.h` — botlib API types.
- `be_interface.h` — provides `botimport` (referenced in comment only; not actually called in this file's functions).
