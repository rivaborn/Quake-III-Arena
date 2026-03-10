# code/renderer/qgl_linked.h

## File Purpose
This header provides a compile-time macro mapping layer that aliases all `qgl*` OpenGL wrapper function names directly to their native `gl*` counterparts. It is used on platforms where OpenGL is statically linked (e.g., macOS), eliminating the need for runtime function pointer indirection.

## Core Responsibilities
- Maps every `qgl*` call used in the renderer to the corresponding standard `gl*` OpenGL 1.x function via `#define`
- Provides a zero-overhead, compile-time alternative to the dynamic dispatch path used in `qgl.h` / `linux_qgl.c` / `win_qgl.c`
- Covers the full OpenGL 1.1 core API surface including geometry, texturing, state management, display lists, feedback, evaluators, and pixel operations
- Enables the rest of the renderer to use `qgl*` names uniformly regardless of platform linking strategy

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. This file contains only preprocessor `#define` directives; no functions or data are declared.

## Control Flow Notes
This file is included instead of (or in place of) the function-pointer-based QGL dispatch header. On platforms with static OpenGL linkage, including this header at the top of renderer translation units causes all `qgl*` calls to resolve directly to `gl*` symbols at compile time, bypassing the pointer table populated during `QGL_Init()` on dynamic platforms. It has no runtime presence.

## External Dependencies
- Implicitly depends on a system OpenGL header (e.g., `<GL/gl.h>` or `<OpenGL/gl.h>`) being included before or alongside this file to supply the `gl*` symbol declarations.
- All `gl*` symbols referenced are **defined elsewhere** — provided by the platform OpenGL library (e.g., `libGL.so`, `OpenGL.framework`, `opengl32.dll`).
