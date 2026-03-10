# code/unix/unix_glw.h

## File Purpose
Declares the platform-specific OpenGL window state structure for Linux/FreeBSD. It defines a single shared state object used by the Unix OpenGL window and rendering subsystem.

## Core Responsibilities
- Guards inclusion to Linux/FreeBSD platforms only via a compile-time `#error` directive
- Defines the `glwstate_t` struct holding Unix GL window state
- Exposes `glw_state` as an `extern` global for use across the Unix GL subsystem

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `glwstate_t` | struct | Holds the Unix-side OpenGL window state: a handle to the dynamically loaded GL library and an optional log file pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `glw_state` | `glwstate_t` | global (extern) | Singleton instance of the GL window state; defined in the corresponding `.c` implementation file (`linux_glimp.c`) |

## Key Functions / Methods
None. This is a header-only declaration file with no functions.

## Control Flow Notes
This header is included by the Unix GL implementation files (e.g., `linux_glimp.c`, `linux_qgl.c`). The `glw_state.OpenGLLib` handle is populated during GL initialization when the OpenGL shared library is dynamically loaded (`dlopen`), and released on shutdown (`dlclose`). The `log_fp` field supports optional GL call logging during development/debugging.

## External Dependencies
- `<stdio.h>` — implied by `FILE *log_fp` (must be included before this header by consumers)
- `linux_glimp.c` — defines `glw_state` (definition lives elsewhere)
- No Quake-specific headers; this file is intentionally minimal and low-level
