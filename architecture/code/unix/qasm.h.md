# code/unix/qasm.h

## File Purpose
A shared header file for x86 assembly (`.s`/`.nasm`) translation units on Unix/Linux, providing C-to-assembly symbol name mangling, architecture detection macros, `.extern` declarations for all software-renderer and audio globals, and byte-offset constants for key C structs used directly from assembly code.

## Core Responsibilities
- Define the `C(label)` macro to handle ELF vs non-ELF symbol name decoration (`_` prefix)
- Detect x86 architecture and set `id386` accordingly
- Declare `.extern` references to all software-renderer globals (z-buffer, texture, lighting, span, edge, surface state) for use in `.s` assembly files
- Declare `.extern` references to audio mixer globals (`paintbuffer`, `snd_p`, etc.)
- Define byte-offset constants for C structs (`plane_t`, `hull_t`, `channel_t`, `edge_t`, `surf_t`, etc.) so assembly can perform field-access without the C type system
- Mirror C struct layouts precisely; comments throughout warn that offsets must stay in sync with their C counterparts

## Key Types / Data Structures
None — this file only defines preprocessor constants and assembly directives; no new C types are introduced.

## Global / File-Static State
None declared in this file. All symbols listed under `.extern` are defined in other C/assembly translation units; this file only references them.

## Key Functions / Methods
None — this is a pure header/macro file with no function definitions.

## Control Flow Notes
Not directly part of any runtime flow. Included at the top of Unix x86 `.s` assembly source files (e.g., `snd_mixa.s`, `sys_dosa.s`, `matha.s`) to give them access to external C globals and struct offsets. Has no effect at runtime; all content is consumed at assemble time.

## External Dependencies
- No `#include` directives — entirely self-contained preprocessor/assembler definitions.
- Depends implicitly on the following C headers staying in sync (noted in comments):
  - `model.h` — `plane_t`, `hull_t`, `medge_t`, `mvertex_t`, `mtriangle_t`, `dnode_t`
  - `sound.h` — `sfxcache_t`, `channel_t`, `portable_samplepair_t`
  - `r_shared.h` — `espan_t`, `edge_t`, `surf_t`
  - `d_local.h` — `sspan_t`
  - `d_polyset.c` — `spanpackage_t`
  - `r_local.h` — `clipplane_t`, `NEAR_CLIP`, `CYCLE`
  - `render.h` — `refdef_t`
- External symbols used but defined elsewhere (selected significant ones):

| Symbol | Likely Owner |
|---|---|
| `d_pzbuffer`, `d_zistepu`, `d_ziorigin` | Software renderer depth/z subsystem |
| `paintbuffer`, `snd_p`, `snd_out`, `snd_vol` | Audio mixer (`snd_mix.c`) |
| `r_turb_*` | Turbulent surface rasterizer |
| `edge_p`, `surface_p`, `surfaces`, `span_p` | Renderer edge/surface list manager |
| `aliastransform`, `r_avertexnormals` | Alias model renderer |
| `D_PolysetSetEdgeTable`, `D_RasterizeAliasPolySmooth` | Polyset rasterizer (C entry points called from ASM) |
| `vright`, `vup`, `vpn` | View orientation vectors |
