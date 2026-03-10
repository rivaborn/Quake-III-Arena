# code/macosx/macosx_timers.h

## File Purpose
Conditional header that exposes macOS-specific OmniTimer profiling instrumentation for Quake III Arena's renderer and collision subsystems. When `OMNI_TIMER` is not defined, all macros collapse to no-ops, making the profiling entirely compile-time optional.

## Core Responsibilities
- Define `OTSTART`/`OTSTOP` macros for push/pop-style hierarchical timer nodes
- Declare extern `OTStackNode*` globals representing named profiling points across the renderer and collision paths
- Declare the `InitializeTimers()` initialization function
- Provide a zero-cost stub path (empty macros) when `OMNI_TIMER` is undefined

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OTStackNode` | typedef/struct (defined in OmniTimer framework) | Represents a named node in a hierarchical profiling stack |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rootNode` | `OTStackNode *` | global (extern) | Root of the profiling tree |
| `markFragmentsNode1/2/4` | `OTStackNode *` | global (extern) | Timer nodes for `R_MarkFragments` sub-phases |
| `markFragmentsGrid` | `OTStackNode *` | global (extern) | Timer node for grid traversal in mark-fragments |
| `addMarkFragmentsNode` | `OTStackNode *` | global (extern) | Timer node for fragment accumulation |
| `chopPolyNode` | `OTStackNode *` | global (extern) | Timer node for polygon clipping |
| `boxTraceNode` | `OTStackNode *` | global (extern) | Timer node for box-trace collision queries |
| `boxOnPlaneSideNode` | `OTStackNode *` | global (extern) | Timer node for plane-side classification |
| `recursiveWorldNode` | `OTStackNode *` | global (extern) | Timer node for recursive BSP world traversal |
| `surfaceAnimNode` | `OTStackNode *` | global (extern) | Timer node for animated surface rendering |
| `surfaceFaceNode` | `OTStackNode *` | global (extern) | Timer node for planar face rendering |
| `surfaceMeshNode` | `OTStackNode *` | global (extern) | Timer node for mesh surface rendering |
| `surfaceEndNode` | `OTStackNode *` | global (extern) | Timer node for surface pipeline end |
| `shadowEndNode` | `OTStackNode *` | global (extern) | Timer node for shadow rendering end |
| `stageIteratorGenericNode` | `OTStackNode *` | global (extern) | Timer node for generic shader stage iteration |
| `computeColorAndTexNode` | `OTStackNode *` | global (extern) | Timer node for per-vertex color/texcoord computation |
| `mp3DecodeNode` | `OTStackNode *` | global (extern) | Timer node for MP3 audio decoding |

## Key Functions / Methods

### OTSTART / OTSTOP
- Signature: `#define OTSTART(node)` / `#define OTSTOP(node)`
- Purpose: Push/pop a named timer node onto the OmniTimer stack, bracketing a code region for hierarchical profiling
- Inputs: `node` — an `OTStackNode *` identifying the profiling region
- Outputs/Return: None (macros with side effects on the OmniTimer internal stack)
- Side effects: Mutates OmniTimer's internal call stack; no-op when `OMNI_TIMER` undefined
- Calls: `OTStackPush(node)` / `OTStackPop()`
- Notes: Must be paired; mismatched calls would corrupt the profiling stack

### InitializeTimers
- Signature: `extern void InitializeTimers()`
- Purpose: Allocates and registers all `OTStackNode` globals with the OmniTimer framework
- Inputs: None
- Outputs/Return: `void`
- Side effects: Writes all extern node pointers; defined in a corresponding `.m` file
- Calls: Not inferable from this file
- Notes: Must be called before any `OTSTART`/`OTSTOP` use; only compiled when `OMNI_TIMER` is defined

## Control Flow Notes
This header is included by macOS-specific renderer and collision source files. `InitializeTimers()` would be called during engine startup (likely from `macosx_sys.m` or the `Q3Controller` init path). `OTSTART`/`OTSTOP` pairs wrap hot paths in the render loop and collision queries, executing every frame when profiling is active.

## External Dependencies
- `<OmniTimer/OmniTimer.h>` — macOS/OmniGroup framework providing `OTStackNode`, `OTStackPush`, `OTStackPop`; not present in the open-source release
- All `OTStackNode*` definitions live in a corresponding `.m` implementation file (not in this header)
