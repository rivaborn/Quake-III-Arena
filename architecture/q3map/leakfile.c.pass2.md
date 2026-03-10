# q3map/leakfile.c — Enhanced Analysis

## Architectural Role

This file is a **diagnostic utility** for the q3map BSP compiler tool, used exclusively during offline map compilation. It generates a `.lin` ("leak") visualization file that helps level designers identify and fix BSP sealing violations—areas where the outside void is reachable from supposedly enclosed map regions. The output feeds directly into QE3 (the level editor) for interactive debugging.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/bspc.c**: Main q3map entry point calls `LeakFile()` when BSP tree construction detects the `outside_node` is occupied (indicating a leak)
- **Not part of runtime engine**: This file is **never linked into the shipped game engine**—it's build-tool-only, compiled only when shipping `q3map` executable

### Outgoing (what this file depends on)
- **q3map/qbsp.h**: Header providing `tree_t`, `node_t`, `portal_t` type definitions and BSP-specific state
- **q3map local functions**: 
  - `WindingCenter()` — computes centroid of a portal's polygon winding
  - `GetVectorForKey()` — extracts entity property (here, `"origin"`)  
  - `qprintf()`, `Error()` — q3map diagnostic output macros
- **Standard C I/O**: `fopen`, `fprintf`, `fclose`, `sprintf`
- **Global `source`**: q3map global holding the input map filename (used to construct `.lin` output path)

## Design Patterns & Rationale

**Greedy shortest-path trace**: Rather than exhaustively searching all portal chains, the algorithm greedily follows portals toward decreasing `occupied` values (the distance metric from the outside node). This is efficient O(n) traversal assuming portals form a tree-like structure during BSP construction.

**Entity-agnostic debugging output**: The code outputs raw 3D coordinates rather than entity names, making the `.lin` file format independent of game content—it's pure geometry visualization that any editor can parse.

**Why this design exists**: Map leaks are a notoriously hard-to-debug problem in Quake-era mappers. A visual trace from outside to the leak point is far more actionable than error messages alone. This matches Q1/Q2 BSP compiler conventions (e.g., `qbsp.exe` generates `.lin` files).

## Data Flow Through This File

1. **Input**: `tree_t *tree` — fully compiled BSP tree with portals and node occupancy flags set
2. **Entry condition**: Check `tree->outside_node.occupied > 1` (non-zero occupation means leak exists)
3. **Trace phase**: Starting from outside node, iterate through connected portals, greedily moving to the neighbor with smallest `occupied` value (closest to the leak)
4. **Centroid extraction**: For each portal crossed, compute its winding polygon center and write to file
5. **Terminal point**: When reaching the sealed area containing the leak (occupancy ≤ 1), write the entity origin
6. **Output**: `.lin` text file with one 3D coordinate per line, read by QE3 to draw the leak path as a line in 3D space

## Learning Notes

**Offline tool architecture**: This exemplifies how Quake III separates **build-time tools** (q3map, bspc, q3asm, q3radiant) from **runtime engine**. Diagnostic features like leak files have zero runtime cost—they're pure development aids compiled into separate binaries.

**Portal/node terminology**: BSP terminology here differs from modern renderers:
- **Node**: Leaf or internal BSP tree node (not a portal entity)
- **Portal**: Connects two BSP leaves, created during tree subdivision
- **Winding**: Polygon boundary of a portal face (distinct from surface winding in renderer)

**Greedy vs. exhaustive search**: Modern leak-finding tools sometimes use BFS/Dijkstra to find *guaranteed* shortest path. This greedy approach relies on BSP subdivision properties—it's a clever optimization for the constraints of the era.

## Potential Issues

- **Buffer overflow risk**: `sprintf(filename, "%s.lin", source)` assumes `source` string + 4 bytes fits in `filename[1024]`. Safe in practice but not bounds-checked.
- **Uninitialized pointer**: If `node->nextnode` is never set in the portal loop (malformed BSP tree), `node` becomes garbage and the loop continues indefinitely. No safety checks exist.
- **File I/O error handling**: If `fopen` fails, code calls `Error()` (which likely `longjmp`s), but intermediate file state isn't validated. If the map directory is read-only, the tool aborts silently to the user.
- **Portal cycle assumption**: Algorithm assumes portal graph is acyclic when starting from outside. A BSP tree should guarantee this, but no assertion validates it.
