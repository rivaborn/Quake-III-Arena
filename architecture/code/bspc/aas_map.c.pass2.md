# code/bspc/aas_map.c — Enhanced Analysis

## Architectural Role

This file implements the **brush normalization and expansion stage** of the offline BSPC (BSP→AAS compiler) pipeline. It bridges raw BSP geometry (loaded as `mapbrush_t` arrays) to AAS-ready geometry by (1) validating which entities contribute navigation data, (2) applying entity origin/rotation transforms, (3) expanding brushes outward by player bounding boxes using Minkowski sum, and (4) duplicating geometry per configured presence type (normal vs. crouch). The output feeds directly into the AAS tree construction phase (`aas_create.c`). Without correct expansion here, runtime bot pathfinding would produce collisions or deadlock areas.

## Key Cross-References

### Incoming
- **`AAS_CreateMapBrushes`** — Main entry point; called by BSPC map loader (likely `code/bspc/map_q3.c` or equivalent) once per brush during entity/brush enumeration.
- **Function discovery path:** From `aas_map.h` public API; dispatcher likely in `code/bspc/bspc.c` or `code/bspc/map.c` during the "process entity brushes" stage.

### Outgoing
- **Geometric utilities** (from `qbsp.h`): `FindFloatPlane` (registers new planes into global `mapplanes[]`), `BaseWindingForPlane`, `ChopWindingInPlace`, `AddBrushBevels`, `WindingsNonConvex`.
- **Math** (from `q_shared.c` / `q_math.c`): `CreateRotationMatrix`, `RotatePoint`, `DotProduct`, `VectorMA`, `VectorCopy`, `VectorInverse`, `ClearBounds`, `AddPointToBounds`.
- **Entity introspection** (from BSP entity parser): `ValueForKey`, `FloatForKey`, `GetVectorForKey`, `atoi`, `strlen`.
- **Global brush arrays** (from `map.c`): Mutates `mapbrushes[]`, `brushsides[]`, `nummapbrushes`, `nummapbrushsides`, `mapplanes[]`.
- **AAS configuration** (from `aas_cfg.h`): Reads `cfg` (player bounding boxes and presence types) to drive per-bbox duplication.

## Design Patterns & Rationale

### Minkowski Sum Expansion for Collision Volumes
`BoxOriginDistanceFromPlane()` and `CapsuleOriginDistanceFromPlane()` compute the **support point** of a bounding box relative to a plane normal. When applied to every brush plane in `AAS_ExpandMapBrush()`, this implements a **Minkowski sum**—the geometric effect is that the brush "grows" outward so its plane is just beyond the player's furthest corner. This is essential for correct bot pathfinding: if the brush planes matched the player exactly, bots would clip geometry; expansion ensures clearance.

**Why two methods?** The `capsule_collision` global switch allows selection between:
- **AABB expansion:** Simpler, square-ish player envelope (support point = corner max/min per axis)
- **Capsule expansion:** More realistic (player modeled as cylinder + 2 spheres); uses ball radius and vertical component

### Per-Presence-Type Duplication
Most game engines use a single collision hull (e.g., standing player). Q3's `AAS_CreateMapBrushes()` creates **N copies of each solid/ladder brush**, one per `cfg.numbboxes` (typically 2: `PRESENCE_NORMAL` and `PRESENCE_CROUCH`). Each copy is expanded by its respective bounding box. This enables:
- **Sophisticated routing:** The AAS tree can explicitly represent crouch-only crawlspaces or standing-only doorways.
- **Presence-aware pathfinding:** Queries like "can a crouching bot reach point X?" use the crouch-expanded geometry.
- **No runtime overhead:** Duplication is offline; the AAS file bakes all possibilities.

### Entity Classification and Special-Case Handling
`AAS_ValidEntity()` and `AAS_PositionBrush()` classify entities into semantic categories (`func_wall`, `func_door`, `trigger_hurt`, etc.). Each category gets specialized handling:
- **`func_door_rotating`** with trigger_always: Positioned to the open angle, so AAS assumes the door is passable.
- **`trigger_hurt`/`trigger_push`**: Content flags set to `CONTENTS_HINT`, signaling that areas inside these volumes have special gameplay semantics.
- **Liquid brushes** (CONTENTS_WATER, CONTENTS_LAVA): Single expanded copy (no duplication) because swimming disables presence types.

**Rationale:** AAS geometry must encode static gameplay facts: "this trigger is always active," "bots can swim here." Bots later use these hints to decide routing strategies (avoid lava, expect fall damage in trigger_hurt).

### Recursive Trigger Chain Analysis
`AAS_AlwaysTriggered_r()` statically walks the entity target graph to determine if a mover (e.g., `func_door_rotating`) is permanently triggered by a `trigger_always` entity. Uses a recursion guard (`mark_entities[]` + `memset`). This allows the compiler to **decide once** which door positions to bake into the AAS tree, rather than requiring runtime simulation.

## Data Flow Through This File

```
Input (per entity/brush pair):
  mapbrush_t brush (raw BSP geometry)
  entity_t mapent (origin, rotation, spawnflags, targetname, target, classname, etc.)
  
Validation → AAS_ValidEntity()
  Filter: Keep only world, func_wall, func_door, triggers, func_static, func_door_rotating (if always-triggered)
  
Positioning → AAS_PositionBrush() / AAS_PositionFuncRotatingBrush()
  Translate brush planes by entity origin
  For func_door_rotating: Compute final angle, rotate all planes, check AAS_AlwaysTriggered()
  Register new planes via FindFloatPlane()
  
Content Classification → brush->contents normalization
  Map trigger types → CONTENTS_HINT, CONTENTS_TELEPORTER, CONTENTS_JUMPPAD, etc.
  
Geometry Expansion → AAS_ExpandMapBrush() × cfg.numbboxes
  For each configured bounding box (PRESENCE_NORMAL, PRESENCE_CROUCH):
    AAS_CopyMapBrush()
    Call AAS_ExpandMapBrush() with bbox mins/maxs
    Call AAS_MakeBrushWindings() to compute face windings
    Call AddBrushBevels() to add axis-aligned faces
  
Output (modified global state):
  mapbrushes[nummapbrushes]     (N copies per input, N = cfg.numbboxes)
  brushsides[nummapbrushsides]  (expanded copies have more sides after beveling)
  mapplanes[]                   (new planes registered)
  nummapbrushes, nummapbrushsides (incremented)
  
Next stage: AAS_Create() in aas_create.c reads these expanded brushes to build BSP tree
```

## Learning Notes

### 1. Minkowski Sum as Foundational Algorithm
Game engines rarely expose Minkowski sum to developers. This file shows the **offline compiler strategy**: compute it once per map as part of preprocessing. Runtime engines can then assume all geometry is pre-expanded and skip expensive convolution. This is why BSPC runs slowly but the engine runs fast.

### 2. Entity Metadata as Game Logic Encoding
Map editors (Q3 Radiant) encode game rules as entity properties:
- `classname="func_door_rotating" angle="90"` = "this door rotates 90 degrees"
- `target="door1"` on `trigger_always` = "activate door1 immediately"
- `contents="CONTENTS_LAVA"` on trigger brush = "gameplay rule: this is damage"

Compilers like BSPC **consume and validate** this metadata, sometimes making static decisions (e.g., "can bots pathfind through this door if it's locked?"). Runtime engines execute the rules. This separation is idiomatic to Q3-era game engines.

### 3. Presence-Type Aware Navigation
Most games model the player as a single capsule/cylinder. Q3 goes further: **same map, different navigation geometry per player pose.** This enabled sophisticated bots that could navigate crawlspaces. Modern engines (e.g., Unreal, Unity) often skip per-pose variants and use a single "largest" hull, accepting some unrealistic pathfinding. Q3's approach is more accurate but requires offline duplication.

### 4. Offline Compilation Enables Correctness Guarantees
Functions like `AAS_FixMapBrush()` add clamping planes for out-of-bounds brushes—operations that would be too expensive at runtime. Offline compilation allows expensive validation and correction that ensures bot pathfinding never sees malformed geometry.

### 5. Geometric Epsilon and Robustness
`BBOX_NORMAL_EPSILON = 0.0001` guards against near-zero plane normals (degenerate planes). The careful handling of windings (via `BaseWindingForPlane`, `ChopWindingInPlace`) reflects Q3's era: floating-point geometry was notoriously fragile. Modern engines use higher precision or integer geometry; this code shows workarounds for 32-bit float brittleness.

## Potential Issues

### 1. Global State Coupling
- **`capsule_collision` flag**: Read during expansion (`AAS_ExpandMapBrush()`), but set outside this file (likely in `bspc.c`). If set incorrectly or changed mid-run, brush expansion would be silently wrong.
- **Mitigation:** Flag should be const-like or validated at start of pipeline.

### 2. Out-of-Bounds Brushes Logged, Not Failed
- `AAS_FixMapBrush()` adds clamping planes and logs warnings. `AAS_MakeBrushWindings()` calls `Log_Print()` for out-of-range but *doesn't fail*—it silently zeroes the brush (`ob->numsides = 0`).
- **Risk:** Bad maps produce silently dropped geometry; users see no error, bots pathfind incorrectly.
- **Mitigation:** Could elevate to `Error()` for unrecoverable cases.

### 3. Recursion Guard Relies on Entity Ordering
- `AAS_AlwaysTriggered_r()` uses `mark_entities[]` to detect cycles. Works fine for acyclic target graphs, but if entities are reordered (or if the algorithm is tweaked), cycle detection could fail silently.
- **Mitigation:** The `Warning()` call on cycle detection is a safeguard, but silent failure is possible.

### 4. Configuration Assumption
- `cfg.numbboxes` and `cfg.bboxes[]` are assumed valid; no defensive checks.
- **Risk:** Misconfigured bounding boxes (e.g., zero-sized, NaN) would produce malformed AAS geometry without error messages.
- **Mitigation:** Validate `cfg` at BSPC startup.

---

**Key Takeaway:** This file is a masterclass in **offline geometric preprocessing**. Every function is designed to be expensive (recursive searches, Minkowski sums, winding computations) because they run once at compile time. The result is a baked, validated geometry database that enables fast, correct runtime pathfinding. The per-presence-type duplication is particularly sophisticated—a design choice that distinguishes Q3's bot AI from simpler game engines.
