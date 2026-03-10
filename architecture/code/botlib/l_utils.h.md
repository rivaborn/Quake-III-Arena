# code/botlib/l_utils.h

## File Purpose
A minimal utility header for the botlib subsystem providing convenience macro aliases. It maps botlib-local names to engine-standard symbols and defines simple arithmetic macros.

## Core Responsibilities
- Aliases `vectoangles` under a more descriptive macro name for botlib use
- Aliases `MAX_QPATH` under the platform-conventional `MAX_PATH` name
- Provides inline `Maximum` and `Minimum` comparison macros

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. All definitions are preprocessor macros; no functions are declared or defined.

**Notes:**
- `Vector2Angles(v, a)` — expands to `vectoangles(v, a)`; converts a direction vector to Euler angles. `vectoangles` is defined elsewhere in the engine/shared code.
- `MAX_PATH` — expands to `MAX_QPATH` (typically 64); provides a Windows/POSIX-familiar name without pulling in platform headers.
- `Maximum(x, y)` / `Minimum(x, y)` — single-expression ternary macros. Not type-safe and evaluate arguments once; no multi-evaluation hazard for simple uses, but unsafe with side-effecting expressions.

## Control Flow Notes
Header-only; no runtime control flow. Included by botlib source files that need these aliases without depending on platform-specific headers or repeating the ternary pattern inline.

## External Dependencies
- `vectoangles` — defined elsewhere (engine shared code / `q_math.c`); not declared here.
- `MAX_QPATH` — defined in `q_shared.h` or equivalent engine header; must be visible at inclusion time for `MAX_PATH` to resolve correctly.
