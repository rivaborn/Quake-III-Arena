# code/jpeg-6/jfdctint.c — Enhanced Analysis

## Architectural Role

This file is part of a **vendored IJG libjpeg-6 library** integrated into the renderer's offline texture loading pipeline. Unlike the engine's core subsystems (qcommon, renderer, server, game), it is a stateless utility with no coupling to game engine data structures or runtime state. It executes once per JPEG texture load during initialization or dynamic asset streaming, consuming raw 8×8 image blocks and emitting scaled DCT coefficients for the downstream quantization and color-space conversion stages within libjpeg.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer texture pipeline** (`code/renderer/tr_image.c` → `LoadJPG` via `jload.c`)
  - Invoked during `R_LoadImage` when a `.jpg` file is encountered
  - Upstream: BSP/shader textures, HUD artwork, map lightmaps
  - **Execution context**: Engine initialization (`R_Init`) and dynamic material loading; **not per-frame**

### Outgoing (what this file depends on)
- **libjpeg internal only**: `jinclude.h`, `jpeglib.h`, `jdct.h`
  - Zero coupling to qcommon, server, game, or cgame
  - Uses only pre-compiled constant tables (`FIX_*` macros); no dynamic engine state
  - `MULTIPLY16C16`, `DESCALE`, `RIGHT_SHIFT` — platform-tunable, compile-time macros (not function calls)

## Design Patterns & Rationale

### 1. **Vendored Library Independence**
The file is part of a complete, self-contained JPEG decoder snapshot. Q3A ships libjpeg-6 unchanged (1994 era code) rather than integrating it or using a system library. This ensures:
- **Reproducibility**: Same JPEG behavior across all platforms (Win32, Linux, macOS)
- **Control**: No risk of system libjpeg API/ABI drift breaking texture loading
- **Simplicity**: Avoids OS dependency fragmentation for a non-critical subsystem

### 2. **Scaled Fixed-Point Arithmetic (1989 Algorithm)**
The file implements Loeffler–Ligtenberg–Moschytz (LLM) with 12 multiplies and 32 adds per 1-D pass, using **scaled fixed-point instead of floating-point**. Rationale circa 1994:
- **FPU cost**: Floating-point multiply/accumulate was expensive on 1990s CPUs (Pentium, PowerPC); integer ALU was faster
- **Precision control**: By scaling constants by `CONST_SCALE` (2^13) and deferring division, intermediate rounding is minimized
- **Minimal bit-width**: All intermediate values fit in 32-bit signed integers without overflow (validated by `BITS_IN_JSAMPLE + CONST_BITS + PASS1_BITS <= 26`)

**Modern tradeoff**: Contemporary engines use SIMD or GPU JPEG decode; scaling FPU instructions would be irrelevant on modern CPUs with pipelined FPUs.

### 3. **Intentional Output Scaling (Deferred Quantization)**
Pass 1 scales results by `sqrt(8) * 2^PASS1_BITS`; Pass 2 removes `PASS1_BITS` but leaves **net factor-of-8 scaling**. This is **not a bug** — it's a deliberate contract:
- **Reason**: Saves two multiplies per 1-D DCT (y0, y4 coefficients skip sqrt(N) division)
- **Consequence**: Quantization step in `jcdctmgr.c` (not in this file) absorbs the factor-of-8
- **Coordination**: Tight coupling between this DCT and IJG's quantization module; changing either breaks the invariant

### 4. **Compile-Time Configuration Over Runtime Detection**
- `BITS_IN_JSAMPLE` (8 or 12) determines `CONST_BITS`, `PASS1_BITS`, and macro selection
- `MULTIPLY` is `MULTIPLY16C16` for 8-bit samples (16×16→32 multiply optimization), full 32×32 for 12-bit
- `DCTSIZE == 8` is compile-time assertion; dynamic size is not supported
- **Rationale**: Avoids runtime branches in hot loop; compiler generates optimal code for target sample width

## Data Flow Through This File

```
Input (per 8×8 block):
  64 × DCTELEM (signed int/INT32), ±CENTERJSAMPLE range
       ↓
Pass 1 (row iteration, ctr=7→0):
  - Load 8 samples from each row
  - Even/odd parts (LL&M figure 1)
  - 12 multiplies × FIX_* constants, scaled by CONST_BITS
  - Store results with LEFT_SHIFT(PASS1_BITS)
  - Advance dataptr by DCTSIZE
       ↓
Pass 2 (column iteration, ctr=7→0):
  - Load 8 samples from each column via DCTSIZE-strided access
  - Repeat even/odd decomposition
  - Multiply by FIX_* constants (same as pass 1)
  - DESCALE by (CONST_BITS + PASS1_BITS) to undo pass-1 scaling
  - Leave residual factor-of-8 scaling (netted with quantization)
  - Advance dataptr by 1
       ↓
Output:
  64 × DCTELEM, each scaled by factor-of-8 relative to true DCT
  Ready for downstream quantization, huffman, and serialization
```

**Key state transitions**:
- Input: Unsigned 8-bit or signed JSAMPLE values (e.g., centered at 128 for unsigned 8-bit)
- Intermediate: INT32, ±27-bit precision after multiplies, before DESCALE
- Output: Signed DCTELEM, ±15-bit range post-scaling

## Learning Notes

### Idiomatic Patterns of the Era (1990s)
1. **Fixed-point DSP arithmetic** was standard for portable, predictable performance before SIMD and pipelined FPUs
2. **Compile-time polymorphism** (conditional macros, not C++ templates) for per-configuration code generation
3. **Separable 2-D transforms** (row-then-column) instead of direct algorithms, despite higher code complexity, were faster in practice
4. **In-place algorithms** (overwrite input buffer) to minimize memory bandwidth

### Contrast with Modern Practice
- Modern JPEG decoders use **SIMD** (NEON, SSE, AVX) or **GPU decode** (H.264/H.265 codec blocks)
- **WebAssembly/asm.js** projects often port this exact code to JavaScript because it's portable and predictable
- **Streaming/mobile** hardware (JPEG IP cores) offload decode entirely
- **Precision/overflow analysis** is now verified by fuzzing or formal methods, not hand-calculated bit budgets

### Connection to Game Engine Concepts
- **Asset streaming pipeline**: This is the computational core of texture loading; in modern engines, it's a VFS → decoder → GPU upload chain
- **Offline preprocessing**: Like the `bspc` offline BSP→AAS compiler, JPEG decode is a one-time-per-asset operation, not per-frame
- **Platform abstraction**: The `MULTIPLY16C16` macro parallels the renderer's `qgl*` dynamic function pointers — compile-time vs. runtime platform specialization tradeoffs

## Potential Issues

**None inferable from code + context**. The implementation is:
- **Mathematically correct**: Implements published LLM algorithm with documented precision budget
- **Well-integrated**: Scaling contract with downstream quantization is explicit (comments reference `jcdctmgr.c`)
- **Self-contained**: No external state, no runtime errors, no memory allocation
- **Guarded**: `#ifdef DCT_ISLOW_SUPPORTED` and `DCTSIZE != 8` assertion prevent misconfiguration

The only "issue" is **obsolescence**: Modern hardware has JPEG decode engines; CPU-based scaled fixed-point DCT is a 1994 optimization that adds unnecessary code complexity on 2020s CPUs. But for a vendored library shipped unchanged, this is by design.
