# code/jpeg-6/jversion.h

## File Purpose
Defines version and copyright identification macros for the Independent JPEG Group's (IJG) JPEG library version 6. It serves as the single authoritative source of version metadata for the library build.

## Core Responsibilities
- Declares the library version string (`JVERSION`)
- Declares the copyright notice string (`JCOPYRIGHT`)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This is a pure header file containing only preprocessor macro definitions. It has no runtime behavior and is not directly tied to any init/frame/update/render/shutdown lifecycle. It is included by other IJG library files (typically `jinclude.h` or version-reporting utilities) to embed version information into the compiled library or into output strings.

## External Dependencies
- No includes.
- No external symbols.

---

**Notes:**
- `JVERSION` value is `"6  2-Aug-95"`, indicating JPEG library release 6, dated August 2, 1995.
- `JCOPYRIGHT` credits Thomas G. Lane and the Independent JPEG Group.
- This file is vendored into the Quake III Arena source tree as part of the embedded `jpeg-6` library used for JPEG texture loading (see `code/jpeg-6/jload.c`).
