# code/bspc/cfgq3.c

## File Purpose
This is a BSPC (BSP Compiler) configuration data file for Quake III Arena, written in a custom domain-specific script format (not standard C despite the `.c` extension). It defines the physical bounding volumes and movement physics parameters used by the AAS (Area Awareness System) generator when compiling bot navigation data from BSP maps.

## Core Responsibilities
- Define player bounding box dimensions for normal-stance and crouch-stance presence types
- Specify gravity, friction, and velocity physics constants for bot movement simulation
- Provide risk/reward cost weights (`rs_*`) for various movement actions (jumping, teleporting, grappling, etc.)
- Parameterize the AAS reachability analysis so the compiler can accurately model what movements are physically possible

## Key Types / Data Structures
None. This is a data/script file with no C type definitions.

## Global / File-Static State
None. This file contains no C declarations; all entries are parsed as configuration data by the BSPC tool at runtime.

## Key Functions / Methods
None. This file defines no functions; it is a pure configuration script consumed by the BSPC parser (`l_precomp.c` / `l_script.c`).

## Control Flow Notes
- This file is **not compiled as C source** in the conventional sense; the `.c` extension is conventional for BSPC configs in the Q3 toolchain.
- It is loaded and parsed during the **AAS compilation phase** of the BSPC tool (offline map processing, not runtime game engine).
- The `bbox` blocks define two presence hulls used during reachability analysis: `PRESENCE_NORMAL` (30×30×56 unit standing box) and `PRESENCE_CROUCH` (30×30×40 unit crouched box).
- The `settings` block feeds physics constants into the AAS reachability builder so it can simulate jump arcs, step heights, water behavior, etc. when determining area connectivity.
- The `rs_*` (reachability/risk score) values weight the travel cost of special movement types, influencing bot pathfinding preference in the compiled AAS file.

## External Dependencies
- Parsed by the BSPC script/precompiler subsystem (`code/bspc/l_precomp.c`, `code/botlib/l_script.c`)
- Presence type macros (`PRESENCE_NONE`, `PRESENCE_NORMAL`, `PRESENCE_CROUCH`) mirror definitions in `code/botlib/be_aas_def.h`
- Physics values correspond to Quake III Arena engine constants (e.g., `g_gravity 800`, `pm_maxspeed 320`) defined in `code/game/bg_public.h` and `code/game/g_local.h`
