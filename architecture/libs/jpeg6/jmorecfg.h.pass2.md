# libs/jpeg6/jmorecfg.h — Enhanced Analysis

## Architectural Role

This configuration header defines the compile-time ABI and feature set for the vendored IJG libjpeg-6 library, which the renderer's texture subsystem relies on for JPEG decoding during map and model loading. Rather than core engine code, it's an external dependency interface tuned for Q3's specific requirements: 8-bit lossless JPEG decoding with floating-point DCT acceleration, RGB scanline output, and a relatively small maximum component count (10 channels). The presence of disabled decoder features and commented-out definitions suggests this configuration reflects hard constraints from the original Q3 release: memory footprint, performance targets, and platform compatibility circa 2001.

## Key Cross-References

### Incoming (who depends on this file)
- **code/renderer/tr_image.c**: Calls JPEG texture loader functions whose types are defined here
- **code/jpeg-6/jload.c**: Implements texture-to-image conversion; uses `JSAMPLE`, `JOCTET`, and related macros throughout
- All JPEG library compilation units transitively depend on `BITS_IN_JSAMPLE`, type definitions, and feature flags

### Outgoing (what this file depends on)
- **Compiler feature tests**: Conditional `#ifdef` blocks depend on `HAVE_UNSIGNED_CHAR`, `CHAR_IS_UNSIGNED`, `HAVE_PROTOTYPES`, `__GNUC__`
- **Platform layer (implicit)**: Machine word sizes and architecture assumptions baked into type choices
- No explicit outgoing calls; purely declarative configuration

## Design Patterns & Rationale

**Portable C library configuration pattern**: The file embodies pre-C99 portability practices—explicit conditional compilation for:
- Unsigned char availability (critical on some RISC machines)
- Type widths (short vs. int vs. long for 16-bit and 32-bit values)
- Function calling conventions (`FAR` pointers for 80x86 real mode, now obsolete)

**Precision/Performance tradeoff**: 
- `BITS_IN_JSAMPLE = 8` (not 12) chooses smaller memory footprint and better cache locality; 12-bit would triple sample arrays
- `DCT_FLOAT_SUPPORTED` enabled, `DCT_ISLOW_SUPPORTED` and `DCT_IFAST_SUPPORTED` disabled suggest: floating-point DCT was faster on Q3's target platform (likely Pentium II-era x86)
- `RGB_PIXELSIZE = 4` (not 3) hints at alignment padding—likely 32-bit word boundaries for SIMD or DMA efficiency

**Decoder asymmetry**: All decoder multiscan/progressive options are `#undef`'d, while encoder options are `#define`'d. This reflects Q3's use case: *loading* precompiled baseline JPEGs, not *encoding* progressive ones. Unused decoder code is compiled out, shrinking the binary and DLL footprint.

## Data Flow Through This File

1. **Texture load request** → Renderer's `R_LoadImage` (tr_image.c)
2. **JPEG header parsing** → libjpeg decoder reads bytes into `JOCTET` arrays using macros from this header
3. **Decompression pipeline** → DCT coefficients (`JCOEF`) processed via floating-point, samples reconstructed as `JSAMPLE` (8-bit unsigned)
4. **Upsampling & color conversion** → YCbCr→RGB using `RGB_RED`/`RGB_GREEN`/`RGB_BLUE` offsets; output scanlines with `RGB_PIXELSIZE=4`
5. **Texture upload** → Final image data flows to GL via renderer backend

## Learning Notes

- **Pre-C99 era**: Hand-tweaked type definitions and conditional compilation were essential when `int` could be 16-bit on some architectures and 32-bit on others
- **Sandboxed configuration**: Unlike modern libraries, this vendored copy is locked in Q3's codebase; changes to jmorecfg.h would require recompiling the entire JPEG library
- **Performance archaeology**: The choice of float DCT over integer methods reveals assumptions about CPU cost models that held true in 2001 but may be inverted on modern hardware
- **Disabled features**: Unused decoder features are compiled out, reflecting a philosophy of binary size minimization—especially important for network downloads and CD distribution in that era
- **Modern contrast**: Modern engines either use system-provided JPEG libraries (via dynamic linking) or include only the minimal decode path; full encoder capability is rarely bundled

## Potential Issues

- **INT32 definition is commented out** (lines ~170–172) with a note about XMD_H conflict. This was likely resolved at the platform layer (e.g., qcommon.h or q_shared.h includes xmd.h on some builds), but the commented code suggests a historical compatibility issue
- **RGB_PIXELSIZE = 4 breaks quantization** (note in header): Color quantization is disabled because the library doesn't understand the dummy fourth byte. Modern code would use a separate format enum rather than overloading pixel size
- **Decoder-only configuration limits flexibility**: If a tool (e.g., map editor or asset converter) ever needed to write JPEG textures, reconfiguring for encoder support would require rebuilding the entire JPEG library
- **No 12-bit precision path**: Commenting suggests this was considered but rejected; modern systems might benefit from 10-bit or 12-bit HDR JPEG variants
