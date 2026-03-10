# code/bspc/aas_cfg.h

## File Purpose
Defines the AAS (Area Awareness System) configuration structure used by the BSPC (BSP Compiler) tool. It encapsulates all physics simulation parameters and reachability scoring constants needed to classify navigation areas and build bot pathfinding data.

## Core Responsibilities
- Declare bounding box presence-type flags for grounded vs. airborne states
- Define `cfg_t`, the central configuration structure holding physics and reachability constants
- Expose a global `cfg` instance accessible across the BSPC tool
- Declare `DefaultCfg` and `LoadCfgFile` as the initialization entry points

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cfg_t` / `cfg_s` | struct | Aggregates all AAS physics parameters and reachability scoring floats used during BSP-to-AAS compilation |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cfg` | `cfg_t` | global (extern) | Singleton config instance read by all BSPC subsystems during AAS generation |

## Key Functions / Methods

### DefaultCfg
- Signature: `void DefaultCfg(void)`
- Purpose: Populate `cfg` with hard-coded default physics and reachability values
- Inputs: None
- Outputs/Return: void; side-effects `cfg` global
- Side effects: Writes to global `cfg`
- Calls: Not inferable from this file
- Notes: Called when no config file is provided or as a baseline before `LoadCfgFile`

### LoadCfgFile
- Signature: `int LoadCfgFile(char *filename)`
- Purpose: Parse a named config file and populate `cfg` with its values
- Inputs: `filename` — path to the AAS config file
- Outputs/Return: `int` (likely non-zero on success, 0 on failure)
- Side effects: Writes to global `cfg`; performs file I/O
- Calls: Not inferable from this file
- Notes: Overrides defaults set by `DefaultCfg`; used to tune per-game or per-map physics

## Control Flow Notes
This header is consumed at BSPC tool startup. `DefaultCfg` is called during init, optionally followed by `LoadCfgFile` to override values. The resulting `cfg` global is then read throughout AAS area classification, reachability analysis (`be_aas_reach.c`), and movement simulation during the offline compilation phase — not at game runtime.

## External Dependencies
- Relies on `aas_bbox_t` and `AAS_MAX_BBOXES` defined elsewhere (likely `aasfile.h` or `be_aas_def.h`)
- `vec3_t` from `q_shared.h` / mathlib
- `BBOXFL_GROUNDED` / `BBOXFL_NOTGROUNDED` flags used by presence-type logic in `aas_create.c` or equivalent
