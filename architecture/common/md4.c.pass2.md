# common/md4.c — Enhanced Analysis

## Architectural Role

This file bridges offline build tools and the runtime engine through a shared MD4 implementation. While located in `common/` (the offline tool foundation), `Com_BlockChecksum` is actively consumed by the **qcommon subsystem** for BSP and pak file integrity verification. The algorithm appears in two execution contexts: compile-time toolchain verification (bspc, q3map) and runtime snapshot/network validation, making this a critical integrity anchor across the build-to-runtime pipeline.

## Key Cross-References

### Incoming (who depends on this file)
- **qcommon integrity pipeline** (`code/qcommon/files.c`, server pak validation) — calls `Com_BlockChecksum` to verify `.pk3` archive and BSP checksums during load and runtime pure-server enforcement
- **Build tools** (bspc, q3map) — statically linked for map preprocessing validation; ensures deterministic checksums across compilation boundaries
- **Demo/snapshot layer** — inferred use in network validation and demo playback consistency checks

### Outgoing (what this file depends on)
- **Standard C library** (`<string.h>`) — `memcpy`, `memset` only; notably, the declared `MD4_memcpy`/`MD4_memset` function pointers are **never implemented or called**, suggesting incomplete refactoring from an abstraction layer attempt
- **No upward dependencies** — entirely self-contained; does not call back into qcommon or engine services

## Design Patterns & Rationale

**Streaming Hash API**: The `Init`–`Update`–`Final` pattern is the canonical design for processing variable-length inputs without materializing the entire dataset in memory. Mandatory for network protocol and streaming file I/O contexts where buffering entire maps is infeasible.

**RSA Reference Implementation**: This is the unmodified 1991–1992 RSA Data Security MD4 spec, indicating Q3A adopted the authoritative algorithm wholesale rather than custom-tuning. The conservative choice ensured inter-platform compatibility and avoided implementation bugs—critical for deterministic network play and demo compatibility.

**Unused Abstractions**: The `MD4_memcpy`/`MD4_memset` function pointers declared but never defined suggest an incomplete attempt to sandbox memory operations (perhaps for embedded platforms or sandboxed VMs). They were abandoned in favor of direct CRT calls, leaving dead declarations.

## Data Flow Through This File

```
Arbitrary-length buffer
  ↓
Com_BlockChecksum
  ├→ MD4Init (context reset to IV)
  ├→ MD4Update (streaming incremental absorb, 64-byte block processing)
  │   ├→ MD4Transform (48-round compression)
  │   │   ├→ Decode (byte array → 32-bit LE words)
  │   │   └→ memset (sensitive state wipe)
  │   └→ buffer remainder for next chunk
  ├→ MD4Final (padding, length append, finalization)
  │   ├→ Encode (state → byte array)
  │   └→ memset (context zeroize on exit)
  └→ XOR-fold digest[0..3] → single 32-bit unsigned

Output: collision-prone but fast file fingerprint
```

**Sensitivity**: The `memset` calls in `MD4Transform` and `MD4Final` zeroize the working state to prevent leakage of intermediate values on the stack. Appropriate for a cryptographic algorithm, even though MD4 itself is cryptanalytically broken.

## Learning Notes

**Cryptographic Archaeology**: MD4 (1990) was superseded by MD5 (1992), which was superseded by SHA-1 (1995), which is now deprecated in favor of SHA-2/SHA-3. This file preserves a snapshot of pre-collision-attack cryptography era. Developers studying this engine learn how to preserve algorithm fidelity across decades: the Q3A source remains byte-for-byte compatible with original 1999 binaries.

**Determinism Across Heterogeneous Platforms**: The explicit little-endian encoding (`Encode`/`Decode`) ensures identical digests on big-endian (PPC) and little-endian (x86) targets—critical for network consistency and cross-platform demo playback validation.

**XOR Folding Tradeoff**: Reducing 128 bits to 32 via XOR discards 75% of the information. This is pragmatic (lower cache footprint, faster comparison) but statistically increases collision risk. Acceptable here because:
  - Integrity goal is catching **accidental** corruption, not **adversarial** tampering
  - Pak files use MD4 only as a fast fingerprint, not security boundary

**Modern Engines**: Contemporary engines typically use SHA-256 or hardware CRC32C for file integrity, plus cryptographic signatures for anti-cheat/DRM. Q3A's approach is representative of late-1990s pragmatism: fast, deterministic, sufficient.

## Potential Issues

- **MD4 is cryptographically broken** (collision attacks since 1995). Not an issue for integrity verification (intended use), but this code should never be repurposed for security-critical functions like authentication.
- **Unused function pointers** (`MD4_memcpy`/`MD4_memset` declared, never defined) create confusion and increase maintenance burden; should be removed.
- **Stack-allocated `x[16]` in `MD4Transform`** (256 bytes) could contribute to stack fragmentation in deeply nested collision/pathfinding queries; not critical but worth noting for resource-constrained platforms.
