# code/bspc/_files.c

## File Purpose
This is a pure documentation/manifest file for the BSPC (BSP Compiler) tool. It contains no executable code — its sole purpose is to enumerate and briefly describe every source file belonging to the BSPC module.

## Core Responsibilities
- Serves as a human-readable index of all `.c` source files in the BSPC tool
- Groups files by subsystem: AAS generation, map loading, BSP processing, BSP I/O libraries, GL utilities, and platform-specific Win32 modules
- Documents which files are ported/adapted from an external BSP compiler (marked with `(*)`)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. The file defines no functions, variables, or macros.

## Control Flow Notes
Not applicable. This file participates in no init/frame/update/render/shutdown pipeline. It is not compiled into any translation unit that affects runtime behavior.

## External Dependencies
None. The file contains only a block comment; there are no `#include` directives or symbol references.

---

**Notes on file groupings documented here:**

| Group | Files | Role |
|---|---|---|
| AAS generation | `aas_areamerging.c` … `aas_store.c` | Build the Area Awareness System navigation graph |
| Map loading | `map.c`, `map_hl/q1/q2/q3/sin.c` | Parse source map formats from multiple games |
| BSP processing `(*)` | `tree.c`, `brushbsp.c`, `portals.c`, `csg.c`, `leakfile.c`, `textures.c` | Core BSP compiler stages adapted from id's tools |
| BSP I/O libraries | `l_bsp_*.c`, `l_cmd/log/math/mem/poly/script/threads/utils/qfiles.c` | Low-level parsing, math, memory, and file I/O |
| GL debug output `(*)` | `gldraw.c`, `glfile.c`, `nodraw.c` | Optional OpenGL visualisation and draw-suppression |
| Win32 platform | `bspc.c`, `winbspc.c`, `win32_*.c` | Console/GUI entry points and OS-specific services |
