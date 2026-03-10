# code/game/be_ai_gen.h — Enhanced Analysis

## Architectural Role

This header exposes the genetic algorithm selection mechanism used by the bot AI subsystem to breed improved bot behavior parameters across generations. It bridges the game VM's AI evolution logic (in `code/game/`) with the botlib library's genetic operations, enabling bots to adapt their personality weights and skill parameters based on match performance rankings.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/game/be_ai_weight.c`** — Likely primary consumer; weight interpolation and personality evolution relies on parent/child selection during bot breeding cycles
- **`code/game/g_bot.c`** — Bot lifecycle management (spawn, free, interbreeding) probably calls into genetic selection when evolving bot population
- **`code/game/ai_dmnet.c`** or **`code/game/ai_main.c`** — May invoke genetic selection to rank and select bots for tournament-style play or improvement cycles

### Outgoing (what this file depends on)
- **`code/botlib/be_ai_gen.c`** — Implementation; called via the function pointer or directly linked when `botlib` is built into the game DLL
- No engine syscalls visible; this is purely within the botlib/game AI boundary
- No global state dependencies inferable from the header alone

## Design Patterns & Rationale

**Fitness-Proportional / Rank-Based Selection:**
The function signature takes a `ranks` array and outputs two parent indices plus one child index. This is a classic evolutionary algorithm pattern where:
- Higher-ranked candidates are more likely to be selected as parents
- The child index selects whose weights/parameters will be mutated for the next generation
- **Rationale:** Allows bots to evolve more competitive strategies over repeated matches without explicit skill coding—a common approach in game AI circa the early 2000s

**Three-Index Output Design:**
Rather than returning a struct, the interface uses output pointers for symmetry with the input interface. This is idiomatic to Q3A's C89-era codebase (no standard library containers, minimal abstraction).

**Population Ranking as Input:**
The caller is responsible for computing `ranks[]` (likely win rates, fraglimit ratios, or skill deltas). This decouples genetic logic from game-specific fitness metrics—botlib doesn't know or care what "good" means.

## Data Flow Through This File

**Caller workflow:**
1. Collect a population of N bots with their match results
2. Compute a `float ranks[N]` fitness metric for each bot (e.g., kill/death ratio, wins)
3. Call `GeneticParentsAndChildSelection(N, ranks, &p1, &p2, &child)`
4. Use `p1`, `p2` to identify bots whose parameters will be averaged/crossed over
5. Use `child` to mark which bot's genome gets overwritten with the mutated offspring

**Internal algorithm (inferred):**
- Likely uses a weighted random selection or tournament selection based on rank values
- `parent1` and `parent2` are distinct candidates with probability proportional to their fitness
- `child` may be the lowest-ranked individual (common in generational replacement) or another stochastic pick

## Learning Notes

**Idiomatic to Q3A Era:**
- Botlib was designed as a self-contained, portable library; genetic selection is a pure utility with no I/O or platform dependencies
- The use of floating-point ranks suggests Q3A bots could track skill evolution in real time, unlike scripted or hand-tuned NPC AI
- This is similar to techniques used in RTS games (StarCraft bots, etc.) but simpler—no neural networks, just weight interpolation

**Game AI Evolution Model:**
Modern engines (Unreal, Unity) often use behavior trees or hierarchical FSMs with hard-coded transitions; Q3A's approach of breeding bot weights is fundamentally **neuroevolution-lite**—evolving scalar parameters rather than network topology or code itself.

**Connection to broader botlib pipeline:**
- Genetic selection feeds the **personality interpolation** layer in `be_ai_char.c` and **weapon/goal weighting** in `be_ai_weap.c` and `be_ai_goal.c`
- Bots selected as `parent1/parent2` have their personality traits, weapon preferences, and goal weights blended into the `child` bot
- This creates a believable population where some bots play aggressively, others camp, others roam—and these tendencies can shift across tournament rounds

## Potential Issues

- **Population size assumption:** Return value semantics (success/failure) and behavior when `numranks < 3` are opaque; caller must ensure population is large enough
- **Determinism:** If selection uses random number generation, bot interbreeding may be non-deterministic across replays—relevant for demo recording integrity
- **Rank normalization:** Caller is responsible for ensuring `ranks[]` is well-conditioned (no negative values, reasonable scale); if ranks have extreme outliers, one bot may dominate selection unfairly
