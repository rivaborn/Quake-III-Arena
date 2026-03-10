# code/game/be_ai_gen.h

## File Purpose
Public header exposing the genetic selection interface used by the bot AI system. It declares a single utility function for selecting parent and child candidates based on a ranked fitness array, supporting evolutionary/genetic algorithm techniques in bot decision-making.

## Core Responsibilities
- Declare the `GeneticParentsAndChildSelection` interface for use by bot AI modules
- Expose genetic selection logic as a callable contract across translation units

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### GeneticParentsAndChildSelection
- **Signature:** `int GeneticParentsAndChildSelection(int numranks, float *ranks, int *parent1, int *parent2, int *child)`
- **Purpose:** Selects two parent indices and one child index from a ranked population using a genetic/fitness-proportional selection algorithm. Used to evolve bot behavior weights or strategy parameters.
- **Inputs:**
  - `numranks` — size of the population / ranks array
  - `ranks` — array of `float` fitness scores, one per candidate
  - `parent1`, `parent2` — output pointers for the two selected parent indices
  - `child` — output pointer for the selected child index
- **Outputs/Return:** `int` — return value likely indicates success (non-zero) or failure (0); exact semantics defined in the corresponding `.c` implementation (`be_ai_gen.c` in `botlib/`)
- **Side effects:** Writes selected indices through `parent1`, `parent2`, and `child` output pointers. No I/O or global state visible from this header.
- **Calls:** Not inferable from this file (header only).
- **Notes:** The three output indices are expected to be distinct. Behavior when `numranks < 3` is not inferable from this file.

## Control Flow Notes
Not directly part of a frame loop. Called on-demand by bot AI goal/weight evaluation code (likely from `be_ai_goal.c` or `be_ai_weight.c`) when evolving bot personality weights between matches or skill assessments.

## External Dependencies
- No includes in this header.
- Implementation defined elsewhere: `code/botlib/be_ai_gen.c`
- Consumed by: `code/game/` bot AI modules and potentially `code/botlib/` internals
