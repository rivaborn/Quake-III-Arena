# code/botlib/l_crc.h — Enhanced Analysis

## Architectural Role
l_crc.h provides low-level checksum utility functions to botlib's internal infrastructure, particularly supporting binary file I/O validation and data integrity for the AAS (Area Awareness System) subsystem. As part of botlib's self-contained utility stack (alongside memory, logging, scripting), it enables offline tools (bspc) and runtime engine (server) to validate serialized bot-world data without external dependencies.

## Key Cross-References

### Incoming (who depends on this file)
- **be_aas_file.c / be_aas_file.h**: Likely uses `CRC_ProcessString` or incremental CRC API to validate AAS binary files during load (`AAS_LoadAASFile`). Both streaming and convenience APIs suggest both header validation (quick checksum) and full-file integrity (incremental processing).
- **bspc (offline tool)**: The aas_file.c in code/bspc mirrors the runtime version, so both toolchain and runtime validation use identical CRC logic.
- **be_aas_route.c**: May use CRC for routing cache serialization/checksumming to detect stale cache invalidation.
- **l_script.c / l_libvar.c / be_interface.c**: Likely use CRC for configuration/state serialization integrity checks within botlib's internal data pipelines.

### Outgoing (what this file depends on)
- **q_shared.h / botlib common headers** (inferred): `byte` type and `unsigned short` — platform-neutral base types shared across engine and botlib.
- No external dependencies; self-contained utility (no calls to other botlib modules visible).

## Design Patterns & Rationale
- **Dual API pattern**: Streaming (Init → ProcessByte loop → Value) + convenience (ProcessString all-in-one) reflects two use cases:
  - Large file validation: read BSP/AAS in buffers, incrementally feed CRC
  - Quick checksumming: one-shot API for small config strings or headers
- **16-bit CRC choice** (`crc_t = unsigned short`) reflects Quake III's era (late 1990s) when:
  - 16-bit CRC-16 sufficient for detecting file corruption (not cryptographic)
  - Native register width on target CPUs; fits in CPU cache
  - Faster than 32-bit CRC on some platforms
- **Pointer-based accumulator mutation** (CRC_Init/ProcessByte take `*crcvalue`) enables:
  - No dynamic allocation; stack-safe
  - Compatible with streaming I/O that doesn't buffer entire file
  - Explicit state threading (no hidden global state)

## Data Flow Through This File
1. **Initialization** → caller invokes `CRC_Init(&crc)`, setting crc to seed
2. **Incremental feed** → per data chunk: `CRC_ProcessByte(&crc, byte_data)` folds bytes into accumulator
3. **Finalization** → `CRC_Value(crc)` returns final 16-bit checksum
4. **Alternative (one-shot)** → `CRC_ProcessString(buffer, len)` wraps init+loop+finalize internally
5. **Multi-segment streaming** → `CRC_Init(&crc); CRC_ContinueProcessString(&crc, ...); CRC_ContinueProcessString(&crc, ...); CRC_Value(crc)` for data arriving in multiple chunks (e.g., network packets, file segments)

## Learning Notes
- **Idiomatic to this era**: CRC-16 for non-cryptographic file integrity was standard before widespread adoption of SHA-1/MD5. Modern engines use cryptographic hashes, but Q3A chose CRC for simplicity and speed.
- **No ECS/modern patterns**: Pure procedural utility with no encapsulation beyond function signatures. No error handling; CRC algorithms are deterministic and cannot fail (by design).
- **Streaming design is elegant**: The Init/ProcessByte/Value triplet is a textbook example of monadic-style incremental computation—caller controls memory, control flow, and buffering. This pattern recurs in botlib elsewhere (e.g., `l_script.c` token streaming).
- **Connection to file validation**: In a game engine, checksums guard against corrupted asset files or network desynchronization. Botlib's reliance on CRC hints at the importance of AAS file integrity to bot pathfinding correctness.

## Potential Issues
- **No overflow/wrap-around semantics documented**: 16-bit CRC will wrap; unclear if that's intentional or if callers expect error detection for files > 64 KB. (Likely not an issue in practice given Q3A's asset sizes, but worth noting for understanding intent.)
- **`byte` type undefined in header**: Header assumes `byte` is pre-defined (likely from `q_shared.h` included earlier). No guard visible here, which could cause build failures if included standalone. (Not a correctness issue in integrated build, but suggests header dependency ordering is fragile.)
- **No CRC algorithm specification in header**: Callers cannot validate algorithm choice without reading `.c` implementation. (Purely a documentation issue, but relevant if botlib is ever reused in different context.)
