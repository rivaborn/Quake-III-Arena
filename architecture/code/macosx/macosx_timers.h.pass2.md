# code/macosx/macosx_timers.h — Enhanced Analysis

## Architectural Role

This file provides **macOS-only hierarchical profiling instrumentation for the renderer and collision subsystems**. It sits at the intersection of the **platform layer** (`code/macosx/`) and two hot-path subsystems: the **renderer** (front-end/back-end scene traversal) and **collision** (BSP traces, area queries). The profiling layer is completely optional at compile-time, allowing shipping builds to have zero profiling overhead while development builds can hierarchically sample execution time across the render loop and spatial queries.

## Key Cross-References

### Incoming (who depends on this file)

- **Renderer (`code/renderer/tr_*.c`)** calls `OTSTART`/`OTSTOP` around:
  - `R_MarkFragments` (and sub-phases 1, 2, 4; grid traversal)
  - `R_AddMarkFragments` (surface/decal accumulation)
  - `R_ChopPolygon` (polygon clipping for lightmaps/decals)
  - Surface rendering pipeline (`R_SurfaceAnim`, `R_SurfaceFace`, `R_SurfaceMesh`, etc.)
  - Shader stage iteration (`StageIteratorGeneric`) and per-vertex color/texture computation
  - Dlight computation paths

- **Collision system (`code/qcommon/cm_*.c` and `code/botlib/be_aas_*.c`)** calls around:
  - Box-trace queries (`CM_BoxTrace` → `boxTraceNode`)
  - Plane-side classification (`CM_BoxOnPlaneSide` → `boxOnPlaneSideNode`)
  - Recursive BSP world traversal (generic recursion hotspot)

- **macOS platform layer (`code/macosx/macosx_sys.m`)** calls `InitializeTimers()` during engine startup to allocate and register all nodes with OmniTimer

- **Renderer main loop** includes this header to bracket every frame's render command execution

### Outgoing (what this file depends on)

- **OmniTimer framework** (`<OmniTimer/OmniTimer.h>`) — Apple/OmniGroup profiling library:
  - Provides `OTStackNode` opaque type
  - `OTStackPush(node)` / `OTStackPop()` push/pop hierarchical call stack
  - Accumulates wall-clock timings in a hierarchical tree
  - Not part of the open-source release (proprietary or system-level framework)

- **Implementation file** (elsewhere, likely `macosx_timers.m`) defines and initializes all extern `OTStackNode*` globals

## Design Patterns & Rationale

**Hierarchical Stack Profiling (OmniTimer Pattern)**
- Each `OTStackNode` represents a named interval in a call-stack hierarchy
- `OTSTART(node)` pushes; `OTSTOP()` pops; nesting is automatic
- Rationale: Rich timing data without manual interval management; ideal for frame-rate-critical profiling

**Zero-Cost Abstraction (Compile-Time Optional)**
- When `OMNI_TIMER` undefined, both macros compile to nothing — zero instructions, zero overhead
- Rationale: Shipping builds incur no profiling cost; development builds get detailed profiling at the cost of a function-call pair per bracketed region

**Conditional Compilation + Extern Globals**
- All node pointers are extern, definition deferred to `.m` file
- No per-file coupling; any source including this header can use the same global nodes
- Rationale: Minimizes recompilation; allows multiple compilation units to share profiling points without duplicating node allocation

**macOS Platform Layer Encapsulation**
- Profiling is entirely confined to `code/macosx/`; other platform layers (`win32/`, `unix/`) have their own profiling (or none)
- This isolates OmniTimer dependency to one platform
- Rationale: Portable to other platforms without dragging in platform-specific profiling APIs

## Data Flow Through This File

1. **Initialization Phase** (engine startup):
   - Platform layer calls `InitializeTimers()` (implementation not visible here)
   - All 17 extern `OTStackNode*` pointers are allocated and registered with OmniTimer

2. **Per-Frame Phase** (render loop and collision queries):
   - Renderer code executes `OTSTART(markFragmentsNode1)` at the start of `R_MarkFragments`
   - Sub-phases push their own nodes (e.g., `OTSTART(markFragmentsGrid)` during grid traversal)
   - `OTSTOP()` pops each node on exit
   - OmniTimer's internal stack tracks call depth and accumulates per-node wall-clock time
   - Collision queries similarly bracket box traces and BSP traversals

3. **Profiling Output** (developer tool / debugger):
   - OmniTimer framework exposes a GUI (Xcode integration, standalone app, or in-game overlay)
   - Developer sees hierarchical flamegraph-style visualization of per-frame time distribution

## Learning Notes

**Era-Specific Profiling Approach (2005 macOS)**
- OmniTimer was state-of-the-art for Xcode/Cocoa app profiling in the early 2000s
- Modern engines use:
  - Built-in OS profilers (Instruments, perf, ETW)
  - GPU profiling (PIX, RenderDoc, Nsight)
  - Custom in-engine timers with per-frame budgeting
  - Frame pacing and latency histograms

**Third-Party Framework Integration**
- The engine directly imports and wraps a proprietary profiling framework
- No abstraction layer to swap profilers; coupling is tight to OmniTimer
- Contrast with modern engines, which hide profiling behind a facade (e.g., `profiler::ScopedTimer`)

**Hierarchical vs. Flat Profiling**
- This approach assumes you want **nested timing context** (which functions called which, in what order?)
- Modern alternatives: flat per-function histograms (gprof), statistical sampling (perf), or GPU hardware counters
- Hierarchical profiling is excellent for understanding frame time distribution but expensive in detail (every push/pop is a function call)

**Idiomatic Patterns**
- The `RAII`-style scoped timing (push on entry, pop on exit) is now standard (C++ `std::scoped_lock` analogue for profiling)
- The zero-cost abstraction via macro guard is still used today (e.g., `assert()`-style macros in C)

## Potential Issues

**Initialization Ordering**
- `InitializeTimers()` must be called **before** any `OTSTART(node)` invocation
- If renderer runs before initialization, null-pointer dereference in OmniTimer
- No guard in the header; caller must respect the contract

**Macro Pairing Requirement**
- Every `OTSTART` **must** have exactly one matching `OTSTOP()`
- Mismatched pairs (e.g., `OTSTART` in one function, `OTSTOP` in another) corrupt the call stack
- No compile-time checking; easy to introduce bugs with early returns or exceptions

**Framework Dependency**
- Cannot build/profile on any platform without OmniTimer installed
- Open-source builds will likely need to stub out or disable profiling
- Cross-platform code that includes this header but runs on Unix/Windows will fail to link

**Node Proliferation**
- 17 hardcoded profiling points is coarse-grained; hot paths like vertex shading or per-entity traces aren't individually profiled
- Adding new profiling points requires:
  1. Allocating a new extern node in this header
  2. Initializing it in the `.m` file
  3. Bracketing the code path with `OTSTART`/`OTSTOP`
  - No dynamic or data-driven profiling; all instrumentation is manual and static

**MP3 Decode Node Incongruity**
- `mp3DecodeNode` stands out: audio decoding is CPU-bound but not part of the render loop critical path
- Suggests profiling was opportunistically added wherever bottlenecks were suspected, rather than systematically applied
- Hints that profiling maintenance was ad-hoc during development
