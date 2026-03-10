# code/botlib/l_crc.h

## File Purpose
Declares the public interface for a CRC (Cyclic Redundancy Check) checksum utility used within the botlib. Provides functions for computing and incrementally updating 16-bit CRC values over byte sequences.

## Core Responsibilities
- Define the `crc_t` type alias for 16-bit CRC values
- Expose CRC initialization, incremental byte/string processing, and value extraction functions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `crc_t` | typedef (`unsigned short`) | 16-bit CRC accumulator type |

## Global / File-Static State
None.

## Key Functions / Methods

### CRC_Init
- **Signature:** `void CRC_Init(unsigned short *crcvalue)`
- **Purpose:** Initializes a CRC accumulator to its starting seed value.
- **Inputs:** `crcvalue` — pointer to the CRC state to initialize.
- **Outputs/Return:** None (writes through pointer).
- **Side effects:** Mutates `*crcvalue`.
- **Calls:** Not inferable from this file.
- **Notes:** Must be called before any `CRC_ProcessByte` or `CRC_ContinueProcessString` calls on a given accumulator.

### CRC_ProcessByte
- **Signature:** `void CRC_ProcessByte(unsigned short *crcvalue, byte data)`
- **Purpose:** Folds a single byte into an in-progress CRC accumulator.
- **Inputs:** `crcvalue` — current CRC state; `data` — byte to fold in.
- **Outputs/Return:** None (updates `*crcvalue` in place).
- **Side effects:** Mutates `*crcvalue`.
- **Calls:** Not inferable from this file.
- **Notes:** Used for streaming/incremental CRC computation.

### CRC_Value
- **Signature:** `unsigned short CRC_Value(unsigned short crcvalue)`
- **Purpose:** Finalizes and returns the CRC checksum from the accumulator.
- **Inputs:** `crcvalue` — completed CRC accumulator value (passed by value).
- **Outputs/Return:** Final 16-bit CRC checksum.
- **Side effects:** None.
- **Calls:** Not inferable from this file.
- **Notes:** Called after all data bytes have been processed.

### CRC_ProcessString
- **Signature:** `unsigned short CRC_ProcessString(unsigned char *data, int length)`
- **Purpose:** Convenience function that computes a CRC over an entire byte buffer in one call.
- **Inputs:** `data` — pointer to byte buffer; `length` — number of bytes to process.
- **Outputs/Return:** Final 16-bit CRC of the entire buffer.
- **Side effects:** None (self-contained; initializes internally).
- **Calls:** Likely calls `CRC_Init`, `CRC_ProcessByte`, `CRC_Value` internally.
- **Notes:** Non-incremental; suitable for one-shot checksumming.

### CRC_ContinueProcessString
- **Signature:** `void CRC_ContinueProcessString(unsigned short *crc, char *data, int length)`
- **Purpose:** Feeds a multi-byte string into an existing CRC accumulator, enabling multi-segment streaming.
- **Inputs:** `crc` — in-progress CRC state; `data` — byte buffer; `length` — byte count.
- **Outputs/Return:** None (updates `*crc` in place).
- **Side effects:** Mutates `*crc`.
- **Calls:** Likely calls `CRC_ProcessByte` in a loop internally.
- **Notes:** Counterpart to `CRC_ProcessString` for incremental multi-chunk use.

## Control Flow Notes
Header only — no control flow. The implementation lives in `l_crc.c`. Functions are utility-level and called on demand, not tied to any specific engine frame phase.

## External Dependencies
- `byte` type — defined elsewhere (expected from `q_shared.h` or equivalent botlib common header).
- No standard library includes are visible in this header.
