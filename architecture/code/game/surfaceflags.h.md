# code/game/surfaceflags.h

## File Purpose
Defines bitmask constants for brush content types and surface properties shared across the game engine, tools (BSP compiler, bot library), and utilities. The comment explicitly states it must be kept identical in both the quake and utils directories.

## Core Responsibilities
- Define `CONTENTS_*` flags describing what a brush volume contains (solid, liquid, clip, portal, etc.)
- Define `SURF_*` flags describing per-surface rendering and gameplay properties
- Serve as a shared contract between the game module, renderer, collision system, bot library, and map compiler tools

## Key Types / Data Structures
None. Header is purely `#define` constants.

## Global / File-Static State
None.

## Key Functions / Methods
None. Header defines no functions.

## Control Flow Notes
Passive data — no runtime control flow. These constants are used at multiple engine phases:
- **BSP compile time**: `SURF_HINT`, `SURF_SKIP`, `SURF_NODRAW`, `SURF_LIGHTFILTER`, `SURF_ALPHASHADOW`, `CONTENTS_DETAIL`, `CONTENTS_STRUCTURAL`, `CONTENTS_ORIGIN` are consumed by q3map during level compilation.
- **Load/init**: Collision model (`cm_load.c`) reads brush contents from the BSP and stores them using these constants.
- **Frame/update**: Physics and movement code (`bg_pmove.c`, `bg_slidemove.c`) checks `CONTENTS_*` and `SURF_*` flags against trace results each frame to drive gameplay logic (ladder climbing, liquid movement, damage zones).
- **Bot AI**: `CONTENTS_BOTCLIP`, `CONTENTS_NOBOTCLIP`, `CONTENTS_TELEPORTER`, `CONTENTS_JUMPPAD`, `CONTENTS_DONOTENTER` are used by the bot library's AAS pathfinding system.
- **Render**: `SURF_SKY`, `SURF_NODRAW`, `SURF_NOLIGHTMAP`, `SURF_POINTLIGHT`, `SURF_NODLIGHT` are consumed by the renderer and lightmap baker.

## External Dependencies
- Mirrored (must stay in sync) in `code/game/q_shared.h` — the comment warns these definitions also need to be there.
- Referenced by: `code/qcommon/cm_load.c`, `code/game/bg_pmove.c`, `code/renderer/tr_*.c`, `code/botlib/be_aas_*.c`, `q3map/` compiler sources, `code/bspc/` sources.
- No includes — this file is a pure constant-definition leaf with no dependencies of its own.

## Notable Flag Groups

| Group | Flags | Consumer |
|---|---|---|
| Volumetric content | `SOLID`, `LAVA`, `SLIME`, `WATER`, `FOG` | Physics, damage |
| Clip volumes | `PLAYERCLIP`, `MONSTERCLIP`, `BOTCLIP`, `NOBOTCLIP` | Movement, bot AI |
| BSP/compiler | `AREAPORTAL`, `CLUSTERPORTAL`, `DETAIL`, `STRUCTURAL`, `ORIGIN` | q3map, vis |
| Bot navigation | `TELEPORTER`, `JUMPPAD`, `DONOTENTER`, `MOVER` | AAS pathfinding |
| Surface render | `NODRAW`, `SKY`, `NOLIGHTMAP`, `POINTLIGHT`, `NODLIGHT` | Renderer, q3map -light |
| Surface gameplay | `NODAMAGE`, `SLICK`, `LADDER`, `FLESH`, `METALSTEPS`, `DUST` | Physics, audio, FX |
| Compiler hints | `HINT`, `SKIP`, `LIGHTFILTER`, `ALPHASHADOW` | BSP splitter, shadow casting |
