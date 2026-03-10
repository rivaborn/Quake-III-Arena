# code/bspc/glfile.c

## File Purpose
Exports a BSP tree's portal geometry to a `.gl` text file for external GL-based visualization tools. It traverses the BSP tree recursively and writes visible portal windings with per-face shading data.

## Core Responsibilities
- Determine which sides of a portal are visible based on node contents
- Serialize winding point data (XYZ + greyscale lighting) to a `.gl` text file
- Traverse the BSP tree recursively, visiting only leaf nodes to emit portals
- Reverse winding order for back-facing portals
- Count and report the total number of GL faces written

## Key Types / Data Structures
None defined in this file; all types are from `qbsp.h`.

| Name | Kind | Purpose |
|------|------|---------|
| `portal_t` | struct (extern) | BSP portal with winding, plane, and adjacent nodes |
| `node_t` | struct (extern) | BSP tree node/leaf; holds portal list and plane number |
| `tree_t` | struct (extern) | BSP tree root with head node |
| `winding_t` | struct (extern) | Polygon defined by an array of 3D points |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `c_glfaces` | `int` | global | Counter for total portal faces written to the GL file |

## Key Functions / Methods

### PortalVisibleSides
- **Signature:** `int PortalVisibleSides(portal_t *p)`
- **Purpose:** Determines which side(s) of a portal should be rendered, based on the contents of the two adjacent nodes.
- **Inputs:** `p` — portal to test
- **Outputs/Return:** `0` = not visible, `1` = front side visible, `2` = back side visible
- **Side effects:** None
- **Calls:** None
- **Notes:** Portals on NULL `onnode` (outside the map box) always return 0. Same-contents pairs return 0. A FIXME comment suggests the zero-content logic may not be fully correct for all map types.

### OutputWinding
- **Signature:** `void OutputWinding(winding_t *w, FILE *glview)`
- **Purpose:** Writes a single winding polygon to the GL view file with point count header, XYZ coordinates, and a cycling grey shading value.
- **Inputs:** `w` — winding to write; `glview` — open file handle
- **Outputs/Return:** void
- **Side effects:** Writes to `glview`; mutates file-static `level` (declared `static int level = 128`) to vary greyscale per winding.
- **Calls:** `fprintf`
- **Notes:** `level` is a persistent static that increments by 28 each call and wraps via `& 255`, producing varied but deterministic shading across faces.

### OutputPortal
- **Signature:** `void OutputPortal(portal_t *p, FILE *glview)`
- **Purpose:** Determines portal visibility, increments `c_glfaces`, and writes the portal's winding (reversed if back-facing) to the GL file.
- **Inputs:** `p` — portal; `glview` — open file handle
- **Outputs/Return:** void
- **Side effects:** Increments `c_glfaces`; may allocate/free a reversed winding; writes to `glview`.
- **Calls:** `PortalVisibleSides`, `ReverseWinding`, `OutputWinding`, `FreeWinding`

### WriteGLView_r
- **Signature:** `void WriteGLView_r(node_t *node, FILE *glview)`
- **Purpose:** Recursively traverses the BSP tree; at leaf nodes, iterates the portal linked list and emits portals owned by that leaf.
- **Inputs:** `node` — current BSP node; `glview` — open file handle
- **Outputs/Return:** void
- **Side effects:** Writes to `glview` via `OutputPortal`.
- **Calls:** `WriteGLView_r` (recursive), `OutputPortal`
- **Notes:** Portals are only output when `p->nodes[0] == node` to avoid double-emission (each portal is shared by two nodes).

### WriteGLView
- **Signature:** `void WriteGLView(tree_t *tree, char *source)`
- **Purpose:** Top-level entry point; constructs the output filename, opens the `.gl` file, drives the recursive traversal, and reports the face count.
- **Inputs:** `tree` — BSP tree to export; `source` — base filename string
- **Outputs/Return:** void
- **Side effects:** Resets `c_glfaces`; creates/writes a file at `<outbase><source>.gl`; prints progress to stdout; calls `Error` on file open failure.
- **Calls:** `sprintf`, `printf`, `fopen`, `WriteGLView_r`, `fclose`, `Error`

## Control Flow Notes
This file is a one-shot tool-stage utility invoked during BSP compilation (not at game runtime). `WriteGLView` is called after the BSP tree and portals are fully built, as a debug/visualization export step. It has no role in the game's init/frame/render/shutdown loop.

## External Dependencies
- **`qbsp.h`** — pulls in all BSP types (`portal_t`, `node_t`, `tree_t`, `winding_t`, `plane_t`), `outbase` (global char array for output path prefix), and utility function declarations.
- **`ReverseWinding`** — defined in `l_poly.c` (via `qbsp.h` chain)
- **`FreeWinding`** — defined in `l_poly.c`
- **`Error`** — defined in `l_cmd.c`
- **`outbase`** — global `char[32]`, defined in `bspc.c`
