# common/surfaceflags.h

## File Purpose
Defines all content and surface flag bitmask constants shared between the game engine, tools (q3map, bspc), and the botlib. It serves as a single authoritative source for brush content types and surface property flags used across the entire Quake III Arena toolchain.

## Core Responsibilities
- Define bitmask constants for brush **content types** (`CONTENTS_*`)
- Define bitmask constants for **surface properties** (`SURF_*`)
- Act as a shared header synchronized across `common/`, `code/game/`, and tool directories
- Annotate bot-specific content types for AAS/botlib consumption
- Mark BSP-compiler-specific flags (hints, skips, lightmap behavior)

## Key Types / Data Structures
None. This file contains only preprocessor `#define` constants.

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure constants header with no executable code.

## Control Flow Notes
This file has no runtime control flow. It is `#include`d at compile time by:
- The game module (`code/game/`) — for collision, physics, and trigger logic
- The renderer (`code/renderer/`) — for surface draw/skip/lightmap decisions
- Map compiler tools (`q3map/`, `code/bspc/`) — for BSP splitting, AAS generation, and light tracing
- The botlib — for navigation-relevant content classification (`CONTENTS_TELEPORTER`, `CONTENTS_JUMPPAD`, `CONTENTS_DONOTENTER`, `CONTENTS_CLUSTERPORTAL`)

The comment at the top of the file ("This file must be identical in the quake and utils directories") confirms it is manually mirrored rather than symlinked, making divergence a known maintenance risk.

## External Dependencies
None. No includes. No external symbols.

---

### Flag Group Summary

| Group | Range | Consumer |
|---|---|---|
| `CONTENTS_SOLID` / liquids / fog | bits 0–6 | Engine collision, game logic |
| `CONTENTS_AREAPORTAL` | `0x8000` | BSP vis system |
| `CONTENTS_PLAYERCLIP` / `MONSTERCLIP` | `0x10000–0x20000` | Collision |
| Bot contents (`TELEPORTER`–`DONOTENTER`) | `0x40000–0x200000` | AAS/botlib |
| `CONTENTS_ORIGIN` | `0x1000000` | BSP pre-processing only; stripped before compile |
| Game-only body/corpse/trigger/nodrop | `0x2000000–0x80000000` | Server game logic |
| `SURF_*` physics/audio flags | bits 0–`0x2000` | Game physics, audio |
| `SURF_*` BSP/compiler flags | `HINT`, `SKIP`, `NODRAW`, `NOLIGHTMAP`, `LIGHTFILTER`, `ALPHASHADOW` | q3map compiler only |
| `SURF_*` render flags | `NODLIGHT`, `POINTLIGHT`, `SKY` | Renderer |
