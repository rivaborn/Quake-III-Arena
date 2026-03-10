Looking at the codebase context and this file's position within Q3Radiant (the level editor), I can now provide a second-pass architectural analysis that reveals cross-cutting insights.

# q3radiant/BrushScript.cpp — Enhanced Analysis

## Architectural Role

BrushScript.cpp implements a **procedural scripting/macro system for the Q3Radiant level editor**, enabling batch brush manipulation through a command dispatch and variable environment. Unlike the runtime engine (which prioritizes per-frame performance), this editor subsystem prioritizes **user workflow automation** — allowing level designers to parameterize and replay complex transformations (copy, move, rotate, link to map) through a minimal procedural language. It bridges the editor's brush manipulation primitives (`Brush_Clone`, `Brush_Move`, `Select_Rotate*`) with a user-facing UI for interactive scripting.

## Key Cross-References

### Incoming (callers)
- **Q3Radiant editor UI**: Menu commands and script execution triggers (implied by global `g_pParentWnd->MessageBox()` calls and `DoModal()` on `CBSInput` dialog)
- **Script parser/interpreter**: Some parent routine parses textual script and calls individual `CopySelected()`, `MoveSelected()`, `Input()`, etc. functions via function-pointer dispatch (`PFNScript` typedef)

### Outgoing (dependencies)
- **Brush primitives** (`code/bspc/` or internal editor Brush API): `Brush_Clone`, `Brush_Build`, `Brush_Move`, `Brush_AddToList`, `Brush_Free` (some calls commented out, suggesting lifecycle ambiguity)
- **Selection/transform primitives**: `Select_Move()`, `Select_Deselect()`, `Select_Brush()`, `Select_GetTrueMid()`, `Select_RotateAxis()`
- **Entity linking**: `Entity_LinkBrush()` — links brushes to world entity
- **UI/Windowing**: `g_pParentWnd->MessageBox()`, `g_pParentWnd->ActiveXY()->RotateOrigin()`, dialog boxes (`CBSInput`, `DialogInfo`)
- **System**: `Sys_UpdateWindows(W_ALL)` — forces screen redraw after transformations
- **Global state**: Direct manipulation of `selected_brushes` (a linked-list anchor), `active_brushes`, `world_entity`; holds persistent brush pointers in `g_pHold1/2/3`

## Design Patterns & Rationale

**1. Command Dispatch + Parsing**
- `GetParam()` is a **hand-written lexer** (not a real parser) that tokenizes input, respecting strings (quoted), parentheses, commas, and spaces
- Variable substitution happens *during parsing* (if token starts with `$`, resolve to float and format as string) — a form of **eager macro expansion**
- No operator precedence or expression evaluation; all parameters are flat, positional

**2. Symbol Table (Variable Environment)**
- Separate arrays for scalar (`g_Variables[]`) and vector (`g_VecVariables[]`) variables
- Case-insensitive name lookup (all names lowercased on insert and query)
- **Missing feature**: No unset/delete operations; variables can only be added or overwritten
- Rationale: Simplicity for a tool; the fixed `MAX_VARIABLES=64` limit is acceptable for scripts

**3. Brush Lifecycle Ambiguity**
- Some `CopyBrush()` calls have commented-out `Brush_Free()` cleanup (lines 279–280, 285–286, 291–292)
- This suggests **memory management was incomplete** or intentionally deferred (perhaps brushes are freed in batch elsewhere)
- Pattern appears to be: hold brushes in global pointers; map operations clone them into the active list; editor's undo/cleanup handles lifecycle

**4. State Management (InitForScriptRun)**
- Resets all globals before each script execution (hold pointers, counters, loop state, keep-going flag)
- Suggests scripts **do not persist state** across invocations; each run is fresh
- `g_bKeepGoing` flag allows **early termination** if user cancels a dialog

**5. Interactive Input Pattern**
- `_3DPointInput()` uses a **blocking event loop** (lines 455–462) that pumps Windows messages while waiting for user to pick a 3D point
- `AcquirePath()` is likely a camera-controller that listens for user clicks; callback `_3DPointDone()` sets `g_bWaiting = false`
- Shows **tight coupling to the UI event loop** — scripts can pause and wait for interactive input

## Data Flow Through This File

```
Script Execution:
  InitForScriptRun()  [reset environment]
    ↓
  AddVariable() x N   [register user-provided or default parameters]
    ↓
  GetParam() → tokenize script string, expand $variables
    ↓
  Dispatch command function (CopySelected, MoveSelected, RotateSelected, etc.)
    ↓
  [Each command reads params via GetParam(), calls brush/selection primitives]
    ↓
  Input()  [if needed: popup dialog to gather user values]
    ↓
  _3DPointInput()  [if needed: wait for user to pick 3D point]
    ↓
  Sys_UpdateWindows()  [screen refresh after each major op]
```

**State artifacts:** Brushes are **held** in `g_pHold1/2/3` (not in the active map until explicitly copied); the `g_bRotateAroundSelection` flag controls rotation pivot; `RotateOrigin()` is set and consulted by the XY window.

## Learning Notes

**What this reveals about Quake III editor architecture:**
1. **Tool vs. Runtime distinction**: The editor has its own procedural layer (scripts, dialogs, interactive input) completely separate from the runtime game. The runtime engine's architecture (subsystems, VMs, network) doesn't apply here.
2. **Brush-centric design**: Q3Radiant treats brushes as first-class mutable objects with in-place transformation primitives; there's no immutable AST or undo tree visible here (undo likely lives elsewhere).
3. **Global state + imperative updates**: Unlike modern editors (which use functional/immutable patterns), Q3Radiant uses global pointers and direct mutation — typical of late-1990s C++ GUI tools.
4. **Hand-rolled parsing**: No lexer generator or expression evaluator; parameters are position-sensitive and variables expand eagerly. This limits script expressiveness but keeps the tool lightweight.
5. **Interactive + Batch fusion**: Scripts can interleave batch operations (copy, rotate) with **blocking interactive input** (3D point picking, dialog boxes). This is a bridge between macro automation and real-time user control.

## Potential Issues

1. **Memory leak risk**: Commented-out `Brush_Free()` calls suggest brushes may not be freed properly. If `CopyBrush()` is called many times without corresponding deletions, held brushes accumulate.
2. **No error handling for bad variable references**: `VariableValue()` returns `g_fDefault = 9999.9` if a variable is not found (or reports a message box once per run for vector vars). This can silently produce nonsensical values.
3. **String-based variable substitution is fragile**: If a script has `$undefined_var` and it's not defined, the substitution expands to `9999.9`, which is indistinguishable from an intentional value. No type checking or validation.
4. **Blocking event loop in _3DPointInput**: The `while(g_bWaiting) { PeekMessage... }` spin-loop blocks the entire editor UI until the user completes the interaction. Modern UIs would use async callbacks.
5. **No recursion or nesting**: Scripts are flat sequences of commands; no conditionals, loops, or function definitions (beyond the monolithic command dispatch).
6. **Global pointers to brushes can outlive their referents**: If the editor's undo or level-clear mechanism deletes a brush, `g_pHold1/2/3` pointers become dangling.
