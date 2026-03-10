# code/renderer/tr_noise.c

## File Purpose
Implements a 4-dimensional value noise generator for the Quake III Arena renderer. It provides seeded random noise lookup and trilinear+temporal interpolation used by shader effects such as waveform deformations and turbulence.

## Core Responsibilities
- Initialize a fixed-size noise table and permutation array with a deterministic seed
- Provide a permutation-indexed lookup into the noise table via `INDEX`/`VAL` macros
- Perform 4D trilinear interpolation over the noise lattice (x, y, z, t)
- Expose `R_NoiseInit` and `R_NoiseGet4f` as the renderer's public noise API

## Key Types / Data Structures
None (no structs or typedefs defined in this file).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_noise_table` | `static float[256]` | static | Pre-computed noise values in [-1, 1], indexed by permuted lattice coordinates |
| `s_noise_perm` | `static int[256]` | static | Permutation table used to hash 4D lattice indices into `s_noise_table` |

## Key Functions / Methods

### R_NoiseInit
- **Signature:** `void R_NoiseInit( void )`
- **Purpose:** Seeds `srand(1001)` then fills `s_noise_table` with uniform random floats in [-1, 1] and `s_noise_perm` with random bytes [0, 255].
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Writes to `s_noise_table` and `s_noise_perm`; calls `srand`/`rand`.
- **Calls:** `srand`, `rand`
- **Notes:** Seed is hard-coded to 1001, making the noise field fully deterministic across runs.

### GetNoiseValue
- **Signature:** `static float GetNoiseValue( int x, int y, int z, int t )`
- **Purpose:** Returns the precomputed noise value at integer lattice point (x, y, z, t) by chaining permutation lookups via the `INDEX` macro.
- **Inputs:** Four integer lattice coordinates.
- **Outputs/Return:** `float` from `s_noise_table`.
- **Side effects:** None.
- **Calls:** Macro `INDEX` → `VAL` → `s_noise_perm` array access.
- **Notes:** All coordinate components are masked to `NOISE_MASK` (0xFF), so the lattice wraps at 256.

### R_NoiseGet4f
- **Signature:** `float R_NoiseGet4f( float x, float y, float z, float t )`
- **Purpose:** Evaluates smooth 4D noise at an arbitrary floating-point position using trilinear interpolation in (x, y, z) and linear interpolation over two time slices.
- **Inputs:** Four float coordinates; `t` is the time dimension.
- **Outputs/Return:** Interpolated `float` noise value.
- **Side effects:** None.
- **Calls:** `floor`, `GetNoiseValue`, `LERP` macro.
- **Notes:** Samples 8 corners of a unit cube in (x, y, z) for each of 2 consecutive integer time slices (it + 0, it + 1), then linearly interpolates between the two time results using `ft`. The `LERP` macro is linear (not smoothstep), so there is no gradient continuity at lattice boundaries.

## Control Flow Notes
`R_NoiseInit` is called once during renderer initialization (from `R_Init` in `tr_init.c`). `R_NoiseGet4f` is called per-frame during shader evaluation — specifically from `RB_CalcTurbulentTexCoords` and `RB_DeformTessGeometry` (in `tr_shade_calc.c`) — driven by `tess.shaderTime` passed as the `t` argument.

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Standard library:** `floor` (via math.h transitively), `srand`, `rand`
- **Defined elsewhere:** `R_NoiseInit` and `R_NoiseGet4f` are declared in `tr_local.h` and called by other renderer modules.
