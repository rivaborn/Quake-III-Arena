# code/renderer/tr_noise.c — Enhanced Analysis

## Architectural Role

This file implements a deterministic 4D value noise generator serving as a foundational utility for the renderer's shader effects subsystem. It bridges shader animation requirements (via `tess.shaderTime` from the rendering pipeline) with surface deformation calculations in `tr_shade_calc.c`. The noise field is seeded once at renderer initialization (`R_Init` → `R_NoiseInit`) and then consumed per-frame during the front-end surface culling and tesselation phases when shaders specify waveform deformations (e.g., water ripples, energy distortions).

## Key Cross-References

### Incoming (who depends on this file)
- **`tr_init.c:R_Init`** — Calls `R_NoiseInit()` once at startup to seed the noise tables before any frame loop
- **`tr_shade_calc.c:RB_CalcTurbulentTexCoords`** — Samples `R_NoiseGet4f()` to perturb texture coordinates in turbulent shader stages
- **`tr_shade_calc.c:RB_DeformTessGeometry`** — Calls `R_NoiseGet4f()` to deform surface vertices based on time-varying noise (e.g., `deform wave` shader directives)
- **`tr_local.h`** — Declares `R_NoiseInit` and `R_NoiseGet4f` in the renderer's public/private interface

### Outgoing (what this file depends on)
- **Standard C library** — `floor()` (via math.h), `srand()`, `rand()` (via stdlib.h)
- **`tr_local.h`** — Includes the renderer's merged header chain, pulling in `q_shared.h`, `qcommon.h`, platform headers, and OpenGL bindings
- **No dynamic engine calls** — Operates entirely within static memory (no `ri.Hunk_Alloc`, cvar, or command dispatch)

## Design Patterns & Rationale

**Deterministic Seeding via Hard-Coded Seed (1001)**
The fixed seed ensures all clients generate identical noise across the network, critical for multiplayer games where visual deformations must be synchronized. This is preferable to random seeding for gameplay-visible effects.

**Power-of-2 Lookup Tables with Bitmasking**
The `NOISE_MASK (0xFF)` bitmask trades table size (256 entries) for O(1) wrapped indexing. This was essential in 1999–2005 when CPU cache efficiency and avoiding modulo operations were critical. Modern engines often use texture-based noise or Simplex noise for better quality/speed tradeoffs.

**Macro-based Permutation Hashing (INDEX/VAL)**
The nested macro `INDEX(x,y,z,t) → VAL(x+VAL(y+VAL(z+VAL(t))))` chains permutation lookups, effectively performing a hash of four integer coordinates into a single index. This avoids explicit hash functions and loop unrolling overheads.

**Linear LERP vs. Smoothstep**
The shader uses the simple `LERP(a, b, w) = a*(1−w) + b*w` macro, not Perlin's smoothstep curve. This sacrifices gradient continuity at lattice boundaries but gains determinism and speed — acceptable for shader effects where visual artifacts are masked by other rendering.

## Data Flow Through This File

1. **Init Phase** (`R_Init` → `R_NoiseInit`):
   - `srand(1001)` — Deterministic seed
   - Fills `s_noise_table[0..255]` with random floats in `[−1, 1]`
   - Fills `s_noise_perm[0..255]` with random bytes `[0, 255]`

2. **Per-Frame Shader Evaluation** (frame loop → `RB_CalcTurbulentTexCoords` / `RB_DeformTessGeometry`):
   - Caller supplies `(x, y, z, tess.shaderTime)` to `R_NoiseGet4f()`
   - Function discretizes floats into integer grid `(ix, iy, iz, it)` and fractional offsets `(fx, fy, fz, ft)`
   - Fetches 8 corner values per time-slice via `GetNoiseValue()` (16 samples total across `it` and `it+1`)
   - Trilinear interpolates in `(x, y, z)` for both time slices, then linearly interpolates between slices using `ft`
   - Returns single float value `[−1, 1]` to caller for texture coordinate or vertex perturbation

## Learning Notes

**Era-Specific Design**
This noise implementation reflects late-1990s optimization priorities: fixed tables, power-of-2 sizes, macro inlining, and deterministic seeding. Modern engines (Unreal, Unity) use GPU texture-based Perlin or Simplex noise for better scaling and visual quality, but this approach was pragmatic for software-rendered shader stages on mid-range hardware circa 2005.

**Multiplayer-First Architecture**
The hard-coded seed and deterministic lookup pattern emphasize **reproducibility across clients** over aesthetic quality. Every player sees identical water ripples and energy deformations, essential for fair competitive gameplay.

**Shader Deformation Pipeline Integration**
The `t` dimension is explicitly `tess.shaderTime`, the shader stage's accumulated time. This design allows shader authors to write time-based deformation directives (e.g., `deform wave sin 10 10 0 0.1`) that are baked into vertex/texture perturbation during the front-end tesselation phase.

## Potential Issues

- **Derivative Discontinuities at Lattice Boundaries**: Linear interpolation creates visible "grid" artifacts in normal maps or high-frequency effects; modern engines avoid this via higher-order interpolants (Hermite, smoothstep).
- **No Input Validation**: Float coordinates wrap silently via the modulo mask; out-of-bounds time values wrap at 256 units, potentially causing visible seams if shaders assume unbounded time.
- **Limited Spatial Resolution**: 256-entry table means noise repeats every 256 units in each spatial dimension. High-frequency details beyond this period are aliased.
- **srand/rand Portability**: Different C standard library implementations may produce different random sequences, potentially causing cross-platform desyncs in network play (though mitigated by the fixed seed).
