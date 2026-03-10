# code/macosx/macosx_glimp.h

## File Purpose
A minimal platform-specific header that sets up the OpenGL framework includes for the macOS renderer backend. It conditionally enables a CGL macro optimization path that bypasses per-call context lookups.

## Core Responsibilities
- Include the macOS OpenGL framework headers (`OpenGL/gl.h`, `OpenGL/glu.h`, `OpenGL/OpenGL.h`)
- Conditionally include `glext.h` if `GL_EXT_abgr` is not already defined
- Optionally enable `CGLMacro.h` mode to eliminate redundant CGL context lookups per GL call
- Expose the `cgl_ctx` alias into translation units that include this header under `USE_CGLMACROS`

## Key Types / Data Structures
None — this is a pure include/configuration header.

## Global / File-Static State
None declared directly. Under `USE_CGLMACROS`, the macro `cgl_ctx` expands to `glw_state._cgl_ctx` (a `CGLContextObj` held in `glwstate_t`), pulled in via `macosx_local.h`.

## Key Functions / Methods
None — this file contains no function declarations or definitions.

## Control Flow Notes
This header is included at the top of macOS renderer/glimp translation units (`.m` or `.c` files). It has no runtime behavior of its own. The `USE_CGLMACROS` path is a compile-time optimization: when enabled, `CGLMacro.h` rewrites every OpenGL call site in the including translation unit to pass the cached `CGLContextObj` directly, skipping the `CGLGetCurrentContext()` lookup that normally occurs on every GL call. This is relevant only during the render loop where GL call frequency is high.

## External Dependencies
- `<OpenGL/OpenGL.h>` — CGL and core GL types (Apple framework)
- `<OpenGL/gl.h>` — Standard OpenGL API (Apple framework)
- `<OpenGL/glu.h>` — OpenGL Utility Library (Apple framework)
- `<OpenGL/glext.h>` — GL extensions (Apple framework), guarded by `GL_EXT_abgr`
- `macosx_local.h` — Pulled in only under `USE_CGLMACROS`; provides `glw_state` (`glwstate_t`) and the `_cgl_ctx` field (`CGLContextObj`)
- `<OpenGL/CGLMacro.h>` — Apple CGL macro rewrite header, only under `USE_CGLMACROS`; defined elsewhere (Apple SDK)
