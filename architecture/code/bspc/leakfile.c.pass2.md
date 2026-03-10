# code/bspc/leakfile.c — Enhanced Analysis

## Architectural Role

This file implements a **diagnostic artifact generator** for the offline BSPC (Binary Space Partition Compiler) tool. It is not part of the runtime engine; rather, it serves the **map authoring workflow**: after BSP compilation and flood-fill analysis detect a map leak, `LeakFile` writes a `.lin` trace file for QE3 (the level editor) to visualize the leak path in 3D space. This tight coupling between the compiler output and editor visualization was essential in the Q3A development pipeline, allowing level designers to quickly identify and fix structural problems without manual debugging.

## Key Cross-References

### Incoming (who depends on this file)
- Called from `code/bspc/bspc.c` (main BSPC driver) after BSP tree construction and entity flood-fill are complete
- No ingame/runtime code depends on this; it is purely an offline compilation tool

### Outgoing (what this file depends on)
- **`WindingCenter`** (`code/bspc/l_poly.c`) — computes portal winding centroids for trace points
- **`GetVectorForKey`** (`code/bspc/l_bsp_ent.c`) — retrieves entity data (specifically the final occupant's origin)
- **`qprintf`** (`code/bspc/l_cmd.c`) — diagnostic console output
- **`Error`** (`code/bspc/l_cmd.c`) — fatal error handler
- **Global `source`** (defined in `code/bspc/bspc.c`) — base filename for `.lin` output path construction
- Standard C library (`stdio.h`, `string.h`) — file I/O and string operations

## Design Patterns & Rationale

**Greedy shortest-path traversal:** The algorithm follows portals greedily, always choosing the exit portal whose destination node has the *smallest `occupied` value* less than the current node's. This is valid because `FloodEntities` assigns monotonically increasing `occupied` distances during a breadth-first flood-fill from the outside leaf. The algorithm exploits this invariant to avoid explicit breadth-first search or Dijkstra; a single O(n) greedy pass suffices.

**Early-exit guard:** Returns immediately if `tree->outside_node.occupied == 0`, avoiding file I/O and allocation if no leak exists. This is efficient for valid maps (the common case).

**Q3A idiom `s = (p->nodes[0] == node)`:** Standard way to determine which side of a portal the current node occupies, then walk `p->next[!s]` to iterate portals for that node's side. Compact but requires familiarity with Q3's portal representation.

**No defensive null-checks:** The code assumes that if the loop finds no reachable portal, the flood-fill invariant is violated — a programmer error in the BSP compiler, not a runtime condition. This reflects the offline tool philosophy: strict correctness assumptions, fail fast on inconsistency.

## Data Flow Through This File

1. **Input:** `tree_t *tree` with a fully constructed and flood-filled BSP tree (nodes marked with `occupied` distances; portals linked)
2. **Processing:**
   - Early exit if outside node is not occupied (no leak)
   - Open output file `<source>.lin` for writing
   - Initialize `node` to the outside (exterior) leaf
   - **Loop:** While `node->occupied > 1` (not yet at the entity-containing leaf):
     - Iterate all portals linked to current node
     - Select the portal whose destination has the smallest `occupied` < current
     - Compute centroid of that portal's winding
     - Write centroid to `.lin` file
     - Advance `node` to the destination
   - **Final step:** Append the occupant entity's origin as the last trace point
3. **Output:** `.lin` file with one XYZ coordinate per line; consumed by level editor to draw a line from the map exterior through leak points to the first trapped entity

## Learning Notes

**Mid-1990s offline tool philosophy:** This code exemplifies how Q3A's compiler toolchain was designed as a separate, standalone suite. Unlike modern engines that validate maps at runtime, Q3A compiles offline and assumes correctness. The BSPC tool is purely deterministic (no runtime randomness or platform-dependent I/O), enabling reproducible builds and predictable developer workflows.

**Portal-graph algorithms:** The greedy traversal here is a variant of **graph reachability tracing**, conceptually similar to pathfinding in botlib's AAS system (`code/botlib/be_aas_route.c`), but vastly simpler because it exploits the flood-fill invariant rather than using Dijkstra or A*. Modern engines typically use explicit graph search for such diagnostics; Quake's approach trades flexibility for speed.

**Minimal data structure coupling:** The file depends on `tree_t`, `node_t`, and `portal_t`, but does not construct, modify, or validate them — purely a traversal consumer. This reflects clean separation: BSP builder (elsewhere in BSPC) owns tree construction; this module is a read-only diagnostic.

**Editor-compiler feedback loop:** The `.lin` file is part of a **tight coupling between tool output and authoring workflow**. Modern engines might emit JSON, XML, or networked queries; Q3A's simple text format was sufficient and debuggable by hand. The format (one XYZ per line, floats, no header) mirrors the low-overhead design philosophy of the entire Q3A codebase.

## Potential Issues

**No null-check after portal search loop:** If the inner `for` loop terminates without finding a candidate, `nextportal` and `nextnode` are uninitialized, and the subsequent `node = nextnode` dereferences garbage. In practice, this cannot happen if the flood-fill is correct (an invariant), but the code assumes this invariant without asserting it. A defensive `if (!nextnode) Error("...")` would catch BSP corruption early.

**Fixed 1024-byte filename buffer:** `sprintf (filename, "%s.lin", source)` assumes `source` is short enough that `strlen(source) + 5` ≤ 1024. Modern code would use `snprintf` or dynamic allocation; here it's acceptable only because `source` is constrained by the command-line argument parser in `bspc.c`.
