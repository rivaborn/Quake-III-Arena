# code/botlib/be_ai_gen.c

## File Purpose
Implements a fitness-proportionate (roulette wheel) genetic selection algorithm for the bot AI system. It provides utilities for selecting individuals from a ranked population, used to evolve bot behavior parameters over time.

## Core Responsibilities
- Perform weighted random selection from a ranked population (higher rank = higher probability)
- Fall back to uniform random selection when all rankings are zero or negative
- Select two parent bots and one child bot for genetic crossover, ensuring the child is selected inversely (lowest-ranked preferred)
- Enforce a hard cap of 256 bots for the parent/child selection function
- Validate minimum population size (at least 3 valid bots) before proceeding

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### GeneticSelection
- **Signature:** `int GeneticSelection(int numranks, float *rankings)`
- **Purpose:** Selects one index from `[0, numranks)` using fitness-proportionate (roulette wheel) selection. Entries with negative rankings are excluded. Falls back to uniform random selection if the total sum is zero.
- **Inputs:** `numranks` — count of candidates; `rankings` — array of float fitness scores (negative = invalid/excluded).
- **Outputs/Return:** Index of the selected candidate (`int`), or `0` as a last-resort fallback.
- **Side effects:** None. Reads `rankings` array in-place; does not modify it.
- **Calls:** `random()` (macro from `q_shared.h` using `rand()`).
- **Notes:** The roulette loop subtracts each rank from the running sum and returns on `sum <= 0`, which correctly implements fitness-proportionate selection. The fallback linear scan handles the degenerate all-zero case.

### GeneticParentsAndChildSelection
- **Signature:** `int GeneticParentsAndChildSelection(int numranks, float *ranks, int *parent1, int *parent2, int *child)`
- **Purpose:** Selects two parents (high-fitness preferred) and one child (low-fitness preferred, via inverted rankings) for a genetic crossover step. Uses a local copy of rankings to allow successive exclusion.
- **Inputs:** `numranks` — total candidates; `ranks` — fitness scores; `parent1`, `parent2`, `child` — output indices.
- **Outputs/Return:** `qtrue` on success, `qfalse` if constraints are violated (too many bots or fewer than 3 valid entries).
- **Side effects:** Prints `PRT_WARNING` via `botimport.Print` on error. Writes `*parent1`, `*parent2`, `*child`.
- **Calls:** `GeneticSelection`, `botimport.Print`, `Com_Memcpy`.
- **Notes:** After selecting both parents, their slots are set to `-1` in the local copy to prevent re-selection. Rankings are then inverted (`max - rank`) so the weakest remaining bot has the highest probability of being selected as the child. Hard limit of 256 bots enforced by stack-allocated `rankings[256]`.

## Control Flow Notes
This file is not part of the per-frame update loop. It is called on-demand by the game-side bot management code (e.g., `g_bot.c`) during bot roster changes or skill evolution events. It has no init or shutdown path of its own.

## External Dependencies
- `../game/q_shared.h` — `random()` macro, `Com_Memcpy`, `qboolean`, `qtrue`/`qfalse`
- `be_interface.h` — `botimport` global (provides `botimport.Print`)
- `../game/botlib.h` — `PRT_WARNING` print type constant
- `l_memory.h`, `l_log.h`, `l_utils.h`, `l_script.h`, `l_precomp.h`, `l_struct.h`, `aasfile.h`, `be_aas_funcs.h`, `../game/be_aas.h`, `../game/be_ai_gen.h` — included but none of their symbols are directly used in this file's two functions; they establish the standard botlib compilation environment.
- `botimport` — defined in `be_interface.c` (global singleton).
