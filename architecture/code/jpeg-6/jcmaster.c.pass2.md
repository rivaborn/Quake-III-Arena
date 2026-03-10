# code/jpeg-6/jcmaster.c — Enhanced Analysis

## Architectural Role

This file is part of the vendored **IJG libjpeg-6 library**, integrated into the Renderer subsystem. While `jcmaster.c` implements JPEG **encoder** logic (compression pipeline), the runtime engine uses only the **decoder** path via `code/renderer/tr_image.c` for texture loading. The full compression machinery is vendored for completeness but remains dormant at runtime; it would activate only in off-engine screenshot/save-to-disk paths (which the game does not implement via this interface).

## Key Cross-References

### Incoming (who depends on this file)
- **Indirect/Unused**: The public encoder entry point `jpeg_start_compress()` and pass-driving macros are exposed via `jpeglib.h`, but no in-engine code path calls them
- **Actual texture I/O consumer**: `code/renderer/tr_image.c` loads JPEG textures using the **decoder** (`jpeg_start_decompress`, `jpeg_read_scanlines`), not this module

### Outgoing (what this file depends on)
- **Internal libjpeg modules**: Dispatches `start_pass` calls to `cinfo->cconvert`, `->downsample`, `->prep`, `->fdct`, `->entropy`, `->coef`, `->main`, `->marker` (all initialized elsewhere in libjpeg)
- **Utility functions**: `jdiv_round_up()` from `jutils.c`; error macros (`ERREXIT`) from `jerror.h`
- **Shared definitions**: `jpeglib.h` types, constants (`DCTSIZE`, `MAX_COMPONENTS`, `C_MAX_BLOCKS_IN_MCU`)

## Design Patterns & Rationale

**Modular Pipeline Architecture**: The code exemplifies late-90s image-codec design: a **multi-pass DAG** of independent processing stages (color conversion, downsampling, FDCT, entropy encoding, output) wired via vtable dispatch. Each stage has a `start_pass()` hook to initialize or switch modes. This decouples concerns and allows swapping implementations (e.g., `optimize_coding=TRUE` inserts a Huffman-optimization pass).

**State Machine for Pass Sequencing**: `c_pass_type` enum and `pass_number`/`scan_number` tracking implement a lightweight FSM. Progressive JPEG support requires a different pass ordering than sequential JPEG, and multi-scan support branches into `huff_opt_pass` or `output_pass` modes—all managed here.

**Early Validation**: `initial_setup()` and `validate_script()` validate all input before compression starts, enforcing JPEG standard rules at initialization rather than during streaming. This is defensive but adds startup cost.

**Tradeoff**: The tight coupling of pass sequencing to module initialization calls means that adding a new pass type or reordering stages requires editing this file—low extensibility. Modern codec frameworks use data-driven pass graphs.

## Data Flow Through This File

**Initialization Phase** (`jpeg_start_compress` → `jinit_c_master_control`):
- Validates image geometry, sampling factors, component counts
- Computes MCU dimensions and block layout
- Loads and validates scan script (if progressive/multi-scan)
- Allocates `my_comp_master` state; initializes all submodule vtables

**Per-Frame Encoding Loop** (application calls `jpeg_write_scanlines` repeatedly):
- On first call: `pass_startup()` hook writes JPEG headers
- Application feeds scanlines to `cinfo->main`
- Master advances through passes: `prepare_for_pass()` → process → `finish_pass_master()` → repeat until `is_last_pass`
- Each pass dispatches to a specific submodule subset based on `pass_type`

**Termination** (`jpeg_finish_compress`):
- Final `finish_pass_master()` flushes entropy coder and closes stream

## Learning Notes

**Historical Codec Idioms**:
- **JMETHOD/vtable dispatch** (`(*cinfo->fdct->start_pass)(cinfo)`) was standard pre-OOP C pattern; modern C++ codecs use inheritance
- **Multi-pass architecture** reflects ICT (Integrated Color Transform) and Huffman table optimization being expensive operations that benefited from separate passes on 1990s hardware
- **Stateful module initialization** (`start_pass`) is why the pass control logic is so intricate—each module must be reset

**What Modern Engines Do Differently**:
- Use hardware decoders (NVDEC, MediaFoundation) for JPEG/H.264; pure-software codecs are rare in real-time engines
- Data-driven pass graphs (instead of hard-coded pass types) allow plugin architecture
- Lazy initialization: don't validate everything upfront; stream and validate as you go

**No Local Optimization**: This file does not perform any application-level caching or tiling optimizations—it assumes the application will feed scanlines in order. Modern encoders often auto-tile large images and use parallelism at the block level.

## Potential Issues

- **Unused at Runtime**: The compression pipeline is dead code in the shipped game. Leaving it vendored increases binary size and maintenance burden if libjpeg is ever patched.
- **Tight Coupling**: Adding or modifying pass types requires editing `prepare_for_pass()` and `finish_pass_master()` in sync; no registry or data table.
- **No Progressive Fallback**: If progressive mode is requested but `C_PROGRESSIVE_SUPPORTED` is undefined, the code exits; it doesn't gracefully downgrade to sequential.
- **Restart Interval Limitation**: `restart_interval` is clamped to 16-bit (`65535L`), which may cause suboptimal recovery granularity for very large images.
