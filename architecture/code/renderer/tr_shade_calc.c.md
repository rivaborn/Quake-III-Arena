# code/renderer/tr_shade_calc.c

## File Purpose
Implements the shader calculation support functions for the Quake III renderer back end, providing vertex deformation, color generation, alpha generation, and texture coordinate generation. All functions operate on the global tessellator buffer (`tess`) and are called during shader stage evaluation before geometry is submitted to OpenGL.

## Core Responsibilities
- Evaluate waveform functions (sin, triangle, square, sawtooth, noise) against precomputed lookup tables
- Deform tessellated vertex positions and normals (wave, bulge, move, autosprite, text)
- Generate per-vertex colors from entity properties, waveforms, and diffuse lighting
- Generate per-vertex alpha values from entity properties, waveforms, and specular calculation
- Generate and transform texture coordinates (environment mapping, fog, turbulence, scroll, scale, rotate, stretch)
- Apply fog density modulation to per-vertex color and alpha channels

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `waveForm_t` | struct (defined in tr_local.h) | Describes a periodic waveform: func, base, amplitude, phase, frequency |
| `deformStage_t` | struct (defined in tr_local.h) | Parameters for a single vertex deformation stage |
| `texModInfo_t` | struct (defined in tr_local.h) | 2×2 matrix + translate for texture coordinate transformation |
| `genFunc_t` | enum (defined in tr_local.h) | Waveform function type selector (GF_SIN, GF_TRIANGLE, etc.) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `lightOrigin` | `vec3_t` | global | Hardcoded specular light position; marked FIXME for dynamic tracking |
| `edgeVerts` | `int[6][2]` | global (file) | Edge vertex index pairs for all 6 edges of a quad, used by Autosprite2 |

## Key Functions / Methods

### TableForFunc
- **Signature:** `static float *TableForFunc( genFunc_t func )`
- **Purpose:** Returns the appropriate precomputed waveform table from `tr` globals.
- **Inputs:** `func` — waveform type enum
- **Outputs/Return:** Pointer into one of `tr.sinTable`, `tr.triangleTable`, `tr.squareTable`, `tr.sawToothTable`, `tr.inverseSawToothTable`
- **Side effects:** Calls `ri.Error(ERR_DROP)` on invalid func, terminating the frame.
- **Calls:** `ri.Error`
- **Notes:** `GF_NOISE` is not handled here; callers that need noise call `R_NoiseGet4f` directly.

### EvalWaveForm / EvalWaveFormClamped
- **Signature:** `static float EvalWaveForm( const waveForm_t *wf )` / `static float EvalWaveFormClamped( const waveForm_t *wf )`
- **Purpose:** Evaluates a waveform at the current shader time using the `WAVEVALUE` macro; clamped variant clips result to [0,1].
- **Inputs:** `wf` — waveform descriptor
- **Outputs/Return:** Float scalar wave value
- **Side effects:** Reads `tess.shaderTime` (global state).
- **Calls:** `TableForFunc`, `myftol`

### RB_DeformTessGeometry
- **Signature:** `void RB_DeformTessGeometry( void )`
- **Purpose:** Dispatcher — iterates over all deform stages in `tess.shader` and calls the appropriate deform function.
- **Inputs:** None (reads `tess.shader->deforms[]`)
- **Outputs/Return:** void; modifies `tess.xyz`, `tess.normal`, `tess.numVertexes`, `tess.numIndexes` indirectly
- **Side effects:** Mutates global `tess` vertex data.
- **Calls:** `RB_CalcDeformNormals`, `RB_CalcDeformVertexes`, `RB_CalcBulgeVertexes`, `RB_CalcMoveVertexes`, `RB_ProjectionShadowDeform`, `AutospriteDeform`, `Autosprite2Deform`, `DeformText`

### RB_CalcDeformVertexes
- **Signature:** `void RB_CalcDeformVertexes( deformStage_t *ds )`
- **Purpose:** Displaces each vertex along its normal by a wave amplitude; if frequency≠0, per-vertex positional phase offset creates a ripple effect.
- **Inputs:** `ds` — deform stage parameters
- **Side effects:** Mutates `tess.xyz` in-place.
- **Calls:** `EvalWaveForm`, `TableForFunc`, `VectorScale`

### RB_CalcBulgeVertexes
- **Signature:** `void RB_CalcBulgeVertexes( deformStage_t *ds )`
- **Purpose:** Animates vertex bulge along normals using U texture coordinate phase offset into the sin table.
- **Side effects:** Mutates `tess.xyz`.
- **Calls:** `tr.sinTable` lookup directly via index arithmetic; reads `backEnd.refdef.time`.

### AutospriteDeform / Autosprite2Deform
- **Signature:** `static void AutospriteDeform( void )` / `static void Autosprite2Deform( void )`
- **Purpose:** Rebuilds quad geometry as view-facing billboards. Autosprite replaces each quad with a camera-aligned stamp; Autosprite2 pivots a rectangular quad along its long axis toward the view.
- **Side effects:** Resets `tess.numVertexes`/`tess.numIndexes` and repopulates via `RB_AddQuadStamp`.
- **Calls:** `GlobalVectorToLocal`, `VectorLength`, `VectorScale`, `CrossProduct`, `VectorNormalize`, `RB_AddQuadStamp`, `VectorMA`, `sqrt`
- **Notes:** Autosprite validates quad-multiple vertex/index counts and warns on mismatch.

### RB_CalcDiffuseColor
- **Signature:** `void RB_CalcDiffuseColor( unsigned char *colors )`
- **Purpose:** Computes per-vertex Lambert diffuse color from entity ambient + directed light and light direction; includes AltiVec (PPC) SIMD fast path.
- **Inputs:** `colors` — output RGBA byte array for `tess.numVertexes` vertices
- **Side effects:** Writes to `colors[]`; reads `backEnd.currentEntity` lighting vectors.
- **Calls:** `DotProduct`, `myftol`, `VectorCopy`
- **Notes:** AltiVec path uses unaligned vector loads; non-AltiVec path clamps per channel to [0,255].

### RB_CalcFogTexCoords
- **Signature:** `void RB_CalcFogTexCoords( float *st )`
- **Purpose:** Computes per-vertex fog texture coordinates encoding fog depth and distance for the fog lookup texture approach.
- **Side effects:** Writes to `st[]`; reads `tr.world->fogs`, `backEnd.or`, `backEnd.viewParms`.
- **Calls:** `VectorSubtract`, `DotProduct`
- **Notes:** Handles eye-inside vs eye-outside fog plane clipping via the T coordinate.

### RB_CalcSpecularAlpha
- **Signature:** `void RB_CalcSpecularAlpha( unsigned char *alphas )`
- **Purpose:** Computes Phong specular highlight and stores it in the alpha channel (used with AGEN_LIGHTING_SPECULAR).
- **Side effects:** Writes to `alphas[3]` (stride 4); reads hardcoded `lightOrigin`.
- **Calls:** `VectorSubtract`, `VectorNormalizeFast`, `DotProduct`, `Q_rsqrt`
- **Notes:** `lightOrigin` is a global with a FIXME comment; specular exponent is effectively 4 (l⁴).

### RB_CalcScrollTexCoords
- **Signature:** `void RB_CalcScrollTexCoords( const float scrollSpeed[2], float *st )`
- **Purpose:** Adds time-based scroll offset to texture coordinates, clamped via `floor()` to avoid precision loss.
- **Side effects:** Modifies `st[]` in-place; reads `tess.shaderTime`.

### RB_CalcRotateTexCoords
- **Signature:** `void RB_CalcRotateTexCoords( float degsPerSecond, float *st )`
- **Purpose:** Rotates texture coordinates around (0.5, 0.5) using sin/cos from the precomputed table.
- **Calls:** `RB_CalcTransformTexCoords`

## Control Flow Notes
All `RB_Calc*` functions are called from `tr_shade.c` during the back-end shader stage iteration loop, after surface geometry has been assembled into `tess` but before OpenGL draw calls. `RB_DeformTessGeometry` is called once per surface before stage iteration begins. No init or shutdown logic resides here.

## External Dependencies
- **`tr_local.h`** — all renderer types, `tess` (shaderCommands_t), `backEnd`, `tr`, `ri`
- **Defined elsewhere:** `R_NoiseGet4f`, `RB_AddQuadStamp`, `RB_AddQuadStampExt`, `RB_CalcFogTexCoords` (self-referential within file), `RB_CalcTransformTexCoords`, `RB_ProjectionShadowDeform`, `Q_rsqrt`, `VectorNormalizeFast`, `VectorNormalize`, `myftol` (x86 inline asm or macro fallback)
- **`WAVEVALUE` macro** — inline table lookup combining phase, time, and frequency into a table index
