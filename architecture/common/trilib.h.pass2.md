# common/trilib.h — Enhanced Analysis

## Architectural Role

This header is part of the **offline tool asset-import library** housed in `common/`, which provides loaders for multiple 3D geometry formats (ASE, 3DS, TRI, etc.) used by build-time tools. `TRI_LoadPolysets` specifically handles the Alias `.tri` ASCII triangle format—a legacy 1990s modeling tool output. The file serves as the boundary interface between higher-level tools (map compiler, level editor) and the low-level triangle/polyset geometry parsing in `common/trilib.c`.

## Key Cross-References

### Incoming (who depends on this file)
- **Tools in `q3map/` and `q3radiant/`** likely consume this directly for model/geometry import during level design and map compilation
- Comparable sibling loaders exist (`aselib.h`, `l3dslib.h`) in the same `common/` directory, suggesting a pluggable asset-import architecture
- Not used by runtime engine (`code/`) — this is strictly offline, compile-time tooling

### Outgoing (what this file depends on)
- **`common/polyset.h`** — defines the `polyset_t` struct (output data structure)
- **Standard C file I/O** — reads disk files (likely via `common/` wrapper functions like `fopen`/`fread` or qcommon filesystem layer)
- **Memory allocation** — typically from `common/` utilities (possibly malloc or a tool-specific heap)

## Design Patterns & Rationale

**Double-pointer output pattern:** `polyset_t **ppPSET` indicates memory is allocated internally by `TRI_LoadPolysets` and ownership passes to the caller. This was idiomatic pre-RAII C practice—cleaner than returning a handle or requiring pre-allocated buffers, but requires the caller to remember to free.

**Single function, single responsibility:** The minimal interface (one exported function) suggests this is a self-contained, standalone module consumed as a library service, not a framework with complex initialization/teardown.

## Data Flow Through This File

1. **Input:** Filename string (path to `.tri` file on disk)
2. **Processing:** (in `trilib.c`) Parse Alias ASCII triangle/polyset definitions, allocate contiguous array
3. **Output:** Pointer-to-array and count passed back via out-parameters; caller gains ownership

This is a **one-shot load pattern**—no persistent state, no per-frame updates, pure file→memory transformation.

## Learning Notes

**Idiomatic to the era:** The Alias `.tri` format reflects late-1990s game development when Alias/Wavefront was a standard 3D modeling tool alongside 3DS Max. Modern engines (Unity, Unreal, Godot) have standardized on FBX or glTF and don't expose per-format loaders; this is a snapshot of tool infrastructure from when importers were format-specific.

**Comparison to siblings:** The `common/` directory houses a *format zoo*—ASE, 3DS, TRI, polygon libraries. This suggests tools were built to interoperate with multiple art pipelines rather than enforce a single asset format. Contemporary Q3A development allowed mappers and modelers to use their preferred DCC (Digital Content Creation) tool and then convert.

**No error handling in signature:** The `void` return type means failure modes (bad file, parse error) are not communicated via return value—likely logged to stdout or stderr, or program exits via `Com_Error`. This is typical of offline tool code where failure aborts the entire build.

## Potential Issues

- **No null-pointer validation:** If `filename` is NULL or `ppPSET`/`numpsets` are invalid, `TRI_LoadPolysets` will likely segfault or invoke undefined behavior. Pre-modern C defensive programming.
- **Memory ownership ambiguity in caller context:** Without inline documentation, a caller might assume the result should not be freed, or might free it with the wrong allocator.
- **Dead code risk:** If tools have migrated away from `.tri` format (e.g., to ASE/3DS exclusively), this loader may no longer be called, making it a maintenance liability.

---

**Sources:** Architecture overview showing `common/` as offline-tool-only subsystem; sibling format loaders (`aselib.h`, `l3dslib.h`) in same directory; Alias/Wavefront history as early CAD/DCC tool.
