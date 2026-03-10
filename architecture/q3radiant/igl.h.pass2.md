# q3radiant/igl.h — Enhanced Analysis

## Architectural Role

This header defines the **OpenGL plugin interface contract** for Q3Radiant (the level editor). It provides two complementary mechanisms: an `IGLWindow` interface allowing plugins to draw within editor viewports, and a `_QERQglTable` dispatch table exposing ~50 GL/WGL function pointers that plugins invoke at runtime. This file is purely **tooling infrastructure** — it has no role in the runtime engine and is never linked into the shipped game DLLs. The dispatch table pattern decouples plugins from hard-coded GL library dependencies, enabling dynamic linking and platform-specific GL loading.

## Key Cross-References

### Incoming (who depends on this file)
- Q3Radiant main application (`q3radiant/*.cpp`) implements the `_QERQglTable` vtable by querying OpenGL function addresses and providing it to plugins at init time
- Q3Radiant plugins query for the `QERQglTable_GUID` interface and receive the populated dispatch table to call GL functions
- `IGLWindow`-derived plugins (in-editor viewport renderers) register themselves via `PFN_QERAPP_HOOKXYGLWINDOW` / `PFN_QERAPP_UNHOOKGLWINDOW` hooks

### Outgoing (what this file depends on)
- `<gl/gl.h>`, `<gl/glu.h>` for OpenGL/GLU type definitions (`GLenum`, `GLfloat`, `GLdouble`, `GLuint`, etc.)
- Windows headers (implicit in typedefs like `HGLRC`, `HDC`, `APIENTRY`, `WINAPI`) for WGL context and device handles
- No internal engine dependencies — this is a pure interface boundary file

## Design Patterns & Rationale

**Virtual Dispatch Table (vtable) Pattern:** The `_QERQglTable` struct is a classic pre-C++ function-pointer vtable. Every GL call in a plugin becomes an indirect call through the table. This allows:
- Runtime GL loading without plugins knowing where GL comes from
- Potential mocking/interception in debug builds
- Platform-specific GL implementations (Windows WGL vs. Linux GLX) without recompiling plugins

**Plugin Interface Discovery via GUID:** The hardcoded GUID `{0xf237620, 0x854b, 0x11d3, ...}` allows plugins to query "does the editor support OpenGL drawing?" and negotiate API versions — though notably, **no version field exists** in the table, suggesting a fixed-format assumption (brittle).

**Reference-Counted Drawing Interface:** `IGLWindow::IncRef()`/`DecRef()` manages plugin lifetime. When a plugin registers itself to draw in a viewport, the editor increments the ref count; on unload or viewport close, it decrements. This prevents use-after-free when plugins are unloaded mid-session (critical for long-running editor sessions).

**Monolithic Function Pointer Collection:** All 50+ functions are bundled into one struct rather than smaller, focused interfaces (e.g., `IGL_MatrixOps`, `IGL_TextureOps`). This simplifies initial dispatch table initialization but makes it harder to extend without breaking plugins.

## Data Flow Through This File

1. **Initialization Phase:** Q3Radiant's `GetRefAPI`-style loading loads the GL DLL (e.g., `opengl32.dll`), queries each function address (e.g., via `wglGetProcAddress`), and **fills the `_QERQglTable` structure** — this populates all 50+ function pointers.
2. **Plugin Discovery:** A plugin DLL loads, queries the editor for `QERQglTable_GUID`, receives the populated table, and stashes it in a local variable.
3. **Drawing Phase:** Plugin code calls (e.g.) `g_qglTable->m_pfn_qglColor3f(1.0f, 0.0f, 0.0f)` to invoke `glColor3f` through the dispatch table.
4. **Viewport Registration:** Plugin derives from `IGLWindow`, implements `Draw(VIEWTYPE)`, and registers via `m_pfnHookXYGLWindow`. The editor calls `Draw()` at refresh time; plugin calls GL functions via the dispatch table.

## Learning Notes

- **Era-specific architecture:** This file reflects late-1990s/early-2000s Windows game engine tooling patterns, pre-COM but using COM-like GUIDs for interface discovery.
- **No compile-time polymorphism:** Plugins see only function pointers, not C++ virtual methods. Minimal ABI coupling and easy to version in future (theoretical; version field not yet added).
- **Windows-centric:** WGL functions (`wglCreateContext`, `wglSwapBuffers`, `wglUseFontBitmaps`) dominate. Cross-platform editors would need separate `#ifdef` blocks or a separate abstraction.
- **Stateless GL:** The dispatch table has no per-context state — all 50+ functions are global entry points. Plugins assume a single active GL context at any time (editor guarantees this via `m_pfnGetQeglobalsHGLRC`).
- **No error checking:** Function pointers are never validated as non-NULL. If GL is unavailable at runtime, calling through the table will segfault.

## Potential Issues

1. **Null Pointer Dereference:** No checks that any `m_pfn_*` field is non-NULL before dereferencing. If GL loading fails, plugins crash silently.
2. **No Versioning:** GUID is fixed; if future Radiant versions add more GL functions (e.g., `glBindBufferARB` for VBO support), old plugins won't know about them, and new plugins linked against the extended table will fail on old Radiant builds.
3. **Monolithic Table Brittleness:** Adding or reordering fields in `_QERQglTable` breaks all plugins; no forward/backward compatibility strategy visible.
4. **No Context Affinity:** The dispatch table assumes a single global GL context. Multi-threaded plugin scenarios (e.g., background viewport rendering) could invoke GL calls on the wrong thread.
5. **Incomplete Coverage:** Missing modern GL extensions (`glBindBuffer`, `glMapBuffer`, etc.); plugins relying on newer GL features must fall back to unsafe direct DLL linking or go unsupported.

---

**Summary:** q3radiant/igl.h is a classic editor plugin dispatch interface. It solves the immediate problem (plugins need to call GL without linking GL directly) but lacks versioning and defensive programming. The architecture would benefit from a version field in `_QERQglTable` and NULL-checks in plugins.
