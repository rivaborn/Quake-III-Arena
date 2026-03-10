# q3radiant/cbrushstub.cpp — Enhanced Analysis

## Architectural Role
This stub file serves as a **placeholder implementation layer** within q3radiant (the offline level editor). It defines empty implementations for curve and patch manipulation routines that would handle Bézier surface editing in the map editor. The presence of this stub—rather than full implementations—suggests these advanced geometry features were either incomplete, deferred, or conditionally compiled out during q3radiant development. The file bridges the editor's brush/geometry pipeline and potential curve/patch editing UI without requiring full backend implementations.

## Key Cross-References

### Incoming (who calls these stubs)
- **q3radiant UI/viewport code**: Likely `q3radiant/CamWnd.cpp`, `q3radiant/XYWnd.cpp`, or other viewport/drawing modules call `Curve_CameraDraw()` and `Curve_XYDraw()` during viewport refresh
- **q3radiant brush operations**: File I/O and data persistence likely invokes `Curve_WriteFile()`, `Curve_StripFakePlanes()`, `Curve_AddFakePlanes()`
- **q3radiant patch system**: Similar viewport and persistence calls through the `Patch_*` function family (draw, select, move, scale, etc.)
- **Global toggle handlers**: Code that manages the `g_bShowPatchBounds` and `g_bPatchWireFrame` flags (likely UI prefs or viewport rendering logic)

### Outgoing (what this file depends on)
- **qe3.h**: Declares `brush_t`, function signatures, and possibly defines the curve/patch subsystem interface
- **stdafx.h**: Precompiled header; transitively pulls in Windows/MFC headers and q3radiant's core types
- **No runtime engine dependencies**: Unlike the map compiler (q3map) or BSP processor (bspc), q3radiant stubs do not call into `code/qcommon`, `code/botlib`, or `code/renderer`

## Design Patterns & Rationale

**Stub/Placeholder Pattern**: This file exemplifies the "stub" pattern—function signatures are published (likely in a header), but implementations are deferred or disabled. Two coding styles coexist:
- **Statement-form stubs** (`Curve_*`): Empty function body using `{ }`, suggesting these were planned but never filled in
- **Expression-form stubs** (`Patch_*`): Inline empty bodies using `{}`, more compact syntax for simple no-ops

**Why stubs?** In a large game editor project, curved brushes and patches are non-critical features compared to basic brush/sector manipulation. Rather than delete the declarations and break caller code, the team likely:
1. Kept the signatures for API stability
2. Implemented minimal stubs to satisfy the linker
3. Could later fill these in if needed
4. Or left them as documented "not yet implemented" in the shipped code

**Global visibility flags**: The two `bool` globals (`g_bShowPatchBounds`, `g_bPatchWireFrame`) suggest a simple feature-toggle system—likely checked during viewport rendering to conditionally display patch boundaries or wireframe overlays.

## Data Flow Through This File

**No active data flow**: Because all functions are empty stubs, no data actually flows through this file:
- **Input**: Parameters (brush pointers, geometry data, file names) are accepted but ignored
- **Processing**: None occurs
- **Output**: None is produced; functions return `void` with no side effects

**Intent (if implemented)**:
- `Curve_BuildPoints()` would compute tessellation vertices for Bézier curves
- `Curve_CameraDraw()/Curve_XYDraw()` would push rendering commands to the viewport system
- `Patch_*` family would manage terrain patch geometry: selection state, transformation, I/O
- Global flags would gate visibility of patch/curve UI overlays

## Learning Notes

**Editor development pattern**: This exemplifies how professional editors often ship with incomplete subsystems. Rather than delete code, teams leave stubs in place—a pragmatic choice that:
- Preserves API contracts for future use
- Avoids cascading deletions throughout the codebase
- Documents intent: "Curves and patches were planned but deferred"
- Allows conditional compilation or plugin loading of missing features

**Q3A's priority hierarchy**: The fact that curves and patches are stubbed while basic brush editing is fully implemented reveals q3radiant's design priorities. Bézier surfaces (common in modern engines via ECS or procedural systems) were less critical to Q3A's art pipeline than direct brush geometry.

**Separation of offline tools from runtime**: Unlike the runtime engine (which uses Bézier patches in `code/renderer/tr_curve.c` and `code/qcommon/cm_patch.c`), the editor can safely stub high-level geometry operations because:
- The game engine doesn't import q3radiant code
- The editor is built and shipped separately (qradiant.exe)
- Map files are serialized in BSP format, not q3radiant's internal format

## Potential Issues

**Dead code / incomplete feature**: If callers are not updated, invoking any of these functions will silently do nothing—a subtle silent failure. For example, if a user attempts to "write curve data to file," the operation appears to succeed but no file is written. This could cause data loss or user confusion.

**Global state not initialized**: The two `bool` globals are declared but never initialized. If reading before any explicit assignment, they may hold garbage values, leading to nondeterministic behavior in viewport rendering (patches may randomly appear/disappear).

**Type dependency**: The file relies on `brush_t` from `qe3.h` but does not document whether this type is safe to forward-declare or requires full definition, making refactoring risky.
