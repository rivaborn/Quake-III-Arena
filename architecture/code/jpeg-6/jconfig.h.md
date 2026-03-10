# code/jpeg-6/jconfig.h

## File Purpose
Platform-specific configuration header for the JPEG-6 library, targeting Watcom C/C++ on MS-DOS or OS/2. It defines compiler/platform capability macros consumed by the rest of the libjpeg source tree.

## Core Responsibilities
- Advertises C language feature availability (prototypes, unsigned types, stddef/stdlib headers)
- Configures pointer model and string library preferences for the target platform
- Selects the default and fastest DCT (Discrete Cosine Transform) algorithm variant
- Conditionally enables supported image file formats for the standalone cjpeg/djpeg tools
- Guards internal-only settings (shift behavior) behind `JPEG_INTERNALS`

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure preprocessor configuration header.

## Control Flow Notes
This file has no runtime control flow. It is `#include`d indirectly via `jinclude.h` or directly at the top of libjpeg translation units. Three conditional compilation blocks structure its content:

| Guard | Scope |
|---|---|
| (unconditional) | Platform feature flags, DCT selection |
| `JPEG_INTERNALS` | Internal arithmetic behavior (`RIGHT_SHIFT_IS_UNSIGNED`) |
| `JPEG_CJPEG_DJPEG` | Standalone tool image-format support flags |

## External Dependencies
- No includes. Intended to be the first platform-adaptation header consumed by `jinclude.h`.
- `JDCT_FLOAT` — enum value defined in `jpeglib.h`; referenced here before that header is included, so order of inclusion matters.

## Notes
- `NEED_FAR_POINTERS` is explicitly `#undef`'d with an inline comment explaining Watcom uses flat 32-bit addressing — relevant context for anyone porting to segmented-memory targets.
- `JDCT_DEFAULT` and `JDCT_FASTEST` are both set to `JDCT_FLOAT`, favouring floating-point DCT over the integer variants; this is atypical (most embedded configs prefer `JDCT_ISLOW` or `JDCT_IFAST`).
- The file header comment says `jconfig.wat`, indicating this is the Watcom-specific variant copied/renamed to `jconfig.h` at build time, consistent with libjpeg's multi-platform distribution model.
