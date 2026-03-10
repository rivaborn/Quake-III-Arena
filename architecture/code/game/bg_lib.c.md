# code/game/bg_lib.c

## File Purpose
A self-contained replacement for the standard C library, compiled exclusively for use in Quake III's virtual machine (Q3_VM) target. It provides `qsort`, string functions, math functions, printf-family functions, and numeric parsing so that VM-compiled game modules (game, cgame, ui) do not depend on the host platform's libc.

## Core Responsibilities
- Provide a portable `qsort` (Bentley-McIlroy) usable in both VM and native builds
- Supply string functions (`strlen`, `strcpy`, `strcat`, `strcmp`, `strchr`, `strstr`) for VM builds
- Supply character-classification helpers (`tolower`, `toupper`) for VM builds
- Provide table-driven trigonometry (`sin`, `cos`, `acos`, `atan2`) and `tan` for VM builds
- Implement numeric conversion (`atoi`, `atof`, `_atoi`, `_atof`) with pointer-advance variants
- Implement a minimal `vsprintf`/`sscanf` for formatted I/O inside the VM
- Provide `memmove`, `rand`/`srand`, `abs`, `fabs`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cmp_t` | typedef (function pointer) | Comparator signature `int(const void*, const void*)` used by `qsort` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `randSeed` | `static int` | file-static | PRNG state for `rand()`/`srand()` |
| `sintable` | `float[1024]` | file-static (inside `#if 0` block — disabled) | Pre-computed quarter-period sine values for table-driven trig |
| `acostable` | `float[]` | file-static (inside `#if 0` block — disabled) | Pre-computed arc-cosine values over [−1, 1] |

> **Note:** `sintable` and `acostable`, along with `sin`, `cos`, `acos`, `atan2`, `floor`, `memset`, `memcpy`, `strncpy`, `sqrt` are all inside `#if 0` — compiled out even in VM builds. Only `tan` (under `#ifdef Q3_VM`) is live trig code.

## Key Functions / Methods

### qsort
- **Signature:** `void qsort(void *a, size_t n, size_t es, cmp_t *cmp)`
- **Purpose:** In-place generic sort using Bentley-McIlroy 3-way quicksort with median-of-3 (and ninther for n>40) pivot selection; falls back to insertion sort for small/nearly-sorted input.
- **Inputs:** Array base `a`, element count `n`, element size `es`, comparator `cmp`
- **Outputs/Return:** void; array sorted in-place
- **Side effects:** None beyond mutating `a`
- **Calls:** `med3`, `swapfunc` (via macros `swap`, `vecswap`); recurses/gotos `loop`
- **Notes:** Uses `goto loop` to iterate the larger partition rather than recurse, limiting stack depth. `swaptype` selects word-at-a-time, multi-word, or byte-at-a-time swapping based on alignment.

### memmove
- **Signature:** `void *memmove(void *dest, const void *src, size_t count)`
- **Purpose:** Overlap-safe byte copy; direction depends on `dest > src`.
- **Inputs:** `dest`, `src`, `count`
- **Outputs/Return:** `dest`
- **Side effects:** Writes to `dest`
- **Calls:** None

### vsprintf
- **Signature:** `int vsprintf(char *buffer, const char *fmt, va_list argptr)`
- **Purpose:** Minimal formatted string writer supporting `%d`, `%i`, `%f`, `%s`, `%c`, `%%`; no `*`/`$` support.
- **Inputs:** Output buffer, format string, variadic args cast as `int *`
- **Outputs/Return:** Number of characters written
- **Side effects:** Writes to `buffer`; does **not** bounds-check the output buffer
- **Calls:** `AddInt`, `AddFloat`, `AddString`, `strlen`
- **Notes:** Uses raw `int *` pointer arithmetic on `va_list` — relies on LCC/Q3VM calling convention. `%f` advances arg pointer by 1 under `__LCC__`, 2 otherwise.

### sscanf
- **Signature:** `int sscanf(const char *buffer, const char *fmt, ...)`
- **Purpose:** Minimal scanner; handles `%i`, `%d`, `%u` (integer) and `%f` (float) only.
- **Inputs:** Input buffer, format string, output pointer arguments
- **Outputs/Return:** Always returns 0 (count not incremented — acknowledged as "really crappy")
- **Side effects:** Writes parsed values through variadic pointer arguments
- **Calls:** `_atoi`, `_atof`

### rand / srand
- **Signature:** `int rand(void)` / `void srand(unsigned seed)`
- **Purpose:** LCG PRNG with multiplier 69069, increment 1; returns low 15 bits.
- **Side effects:** Mutates `randSeed`

### atof / _atof
- **Notes:** `atof` parses a `const char*`; `_atof` advances `*stringPtr` past consumed characters. Neither handles scientific notation (`10e10`).

### AddInt / AddFloat / AddString
- **Notes:** Internal helpers for `vsprintf`. Write into a `char **buf_p` cursor. `AddFloat` does not produce correct output for the fractional part when integer digits overflow `text[32]`.

## Control Flow Notes
This file has no engine lifecycle hooks (no init/frame/shutdown). It is a pure utility library. Under `Q3_VM`, it is linked into every VM module (game, cgame, ui) to satisfy libc symbol references that the LCC compiler emits. Under native builds, only `qsort` and `memmove` are compiled (plus `rand`/`srand`/`atof`/`_atof`); the rest are guarded by `#if defined(Q3_VM)`.

## External Dependencies
- **Includes:** `q_shared.h` (provides `qtrue`, `M_PI`, `size_t`, `va_list`, and the `Q3_VM` macro)
- **Defined elsewhere:** `cmp_t` is conditionally typedef'd here only when `Q3_VM` is not defined; under `Q3_VM` it is assumed provided by `bg_lib.h` (included via `q_shared.h → bg_lib.h`)
- **No heap allocation:** All functions operate on caller-supplied buffers or static/stack storage
