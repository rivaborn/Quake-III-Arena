# q3map/prtfile.c — Enhanced Analysis

## Architectural Role

This file is a critical stage in the **offline BSP map compilation pipeline**. It serializes the spatially-partitioned BSP tree and its portal/cluster information into a binary-friendly `.prt` (portal file) format that the **qvis visibility compiler** consumes as input. This file sits at the boundary between geometric BSP creation and visibility/PVS preprocessing, making it a bridge between two major compilation phases.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/bsp.c** — Likely calls `NumberClusters()` and `WritePortalFile()` as part of the main BSP build sequence
- **q3map/main compilation loop** — Invokes portal file generation after BSP tree construction is complete

### Outgoing (what this file depends on)
- **q3map/tree_t structure** — Reads `tree->headnode` (BSP tree root), traverses via `node->children[0/1]`, `node->planenum`, `node->cluster`, `node->opaque`
- **Portal structures** (node-local) — Each node holds a linked list of `portal_t` (with `winding_t`, `plane_t`, `hint` flag, cluster connections)
- **Winding structures** — Reads `winding_t::numpoints`, `p[i][]` vertex positions
- **Helper functions** (likely from q3map/portals.c or similar) — Calls `Portal_Passable()` to classify portals vs. solid surfaces
- **Math utilities** — `WindingPlane()`, `DotProduct()` for plane/normal arithmetic
- **Standard I/O** — `fprintf()`, `fopen()`, `fclose()` for file writing
- **Global state** — `source` filename, `qprintf()` logging

## Design Patterns & Rationale

**Recursive Tree Traversal:** Three parallel recursive passes (`NumberLeafs_r`, `WritePortalFile_r`, `WriteFaceFile_r`) descend the BSP tree. This is idiomatic for BSP tools — decision nodes branch down, leaves are processed.

**Two-Phase Numbering:** `NumberClusters()` is a required **initialization pass** before `WritePortalFile()`. This assigns cluster IDs to all leaf nodes, enabling later backreferences in the portal file format. The global counters (`num_visclusters`, `num_visportals`, `num_solidfaces`) are accumulated and printed for build verification.

**Plane Orientation Tolerance:** The check `if (DotProduct(p->plane.normal, normal) < 0.99)` handles floating-point precision near axis-aligned plane transitions. Rather than trusting the pre-computed plane, it recomputes from the winding and flips cluster order if needed — defensive against numerical errors in BSP construction.

**Global FILE Pointer:** `FILE *pf` is a module-global rather than stack-local parameter, reflecting early-90s C idiom and simplifying the deeply-nested recursion.

## Data Flow Through This File

**Input Phase:**
1. `NumberClusters()` → `NumberLeafs_r()` traverses tree, assigns `node->cluster` ID to each non-opaque leaf (opaque leaves get -1), counts portals/faces in each leaf's portal list.

**Output Phase:**
2. `WritePortalFile()` opens `.prt` file, writes header (magic "PRT1", cluster/portal/face counts).
3. `WritePortalFile_r()` → emits **portals** (passable edges between clusters): numpoints, cluster indices, hint flag, then vertex coordinates.
4. `WriteFaceFile_r()` → emits **solid faces** (non-passable portals): numpoints, cluster index, vertices (reversed if traversed from second node to maintain CCW winding).

**Key Transitions:**
- Opaque nodes are skipped entirely (solid walls, brushes).
- Each portal is written once (only from `p->nodes[0]`, avoiding duplicates).
- Vertex order is critical: solid faces reverse if emitted from `nodes[1]` to maintain consistent face orientation for qvis.

## Learning Notes

**BSP Compiler Idiom:** This file exemplifies the separation of concerns in offline tools — parsing/creation (earlier stages) vs. serialization (this stage) vs. analysis (qvis). Modern engines often skip this intermediate format and compute PVS inline.

**Portal Definition:** A "portal" here is a half-edge in the BSP tree's cluster graph — it connects two adjacent leaves. Passability is determined by some logic (likely leaf material flags) outside this file.

**Winding vs. Mesh:** Uses `winding_t` (ordered vertex list) rather than indexed triangles — typical for BSP face representation, efficient for convex planar surfaces.

**Floating-Point Pragmatism:** The plane-flip heuristic is a pragmatic workaround: rather than trusting pre-computed normals, it recomputes from data and compares. This is typical of legacy geometry code where strict invariants were sometimes relaxed.

## Potential Issues

- **No validation of tree structure:** If `node->planenum == PLANENUM_LEAF` assertion fails unexpectedly, the code silently treats non-leaves as leaves. No error handling for corrupted trees.
- **Portal_Passable() coupling:** The exact definition of "passable" is opaque here. If that function's semantics change upstream, portal counts and qvis input can silently diverge from expectations.
- **Floating-point tolerance:** The 0.99 threshold in plane comparison is magic-number-tuned; different maps with different scales might hit edge cases.
- **No bounds checking on cluster IDs:** If a node's cluster is uninitialized or corrupted, the write is unchecked (fprintf doesn't validate).
