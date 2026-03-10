# code/game/g_rankings.h — Enhanced Analysis

## Architectural Role

This file defines the **telemetry protocol contract** between the local game server and a remote global rankings backend (GRank). Rather than being used for in-game mechanics, it serves as a **self-documenting schema**: each `QGR_KEY_*` constant's numeric value encodes the stat's category, calculation method, and reporting tier directly into its decimal digits. The file is conditionally compiled to support both base Quake III Arena and MissionPack (Team Arena) stat inventories, allowing a single remote server to aggregate player progression data across both game variants.

## Key Cross-References

### Incoming (who depends on this file)
- **code/server/sv_rankings.c** — Server-side rankings reporter; uses these keys when submitting match statistics to the remote GRank service
- **code/game/g_rankings.c** (implied) — Collects per-player combat stats during gameplay (frags, damage, deaths, item pickups) and associates them with these keys
- **code/game/g_combat.c, g_weapon.c, g_items.c, g_team.c** (indirectly) — Game logic that generates raw statistics (kills, healing, CTF captures) eventually mapped to these keys by the rankings system

### Outgoing (what this file depends on)
- None. This is a pure constants header with no external dependencies. The encoding scheme and category enumeration are self-contained.

## Design Patterns & Rationale

**Numeric-Key Semantic Encoding Pattern**: This uses a **self-describing identifier** where metadata is encoded as decimal digit positions:
- Each digit position (10^0 through 10^9) carries fixed meaning (ordinal, sub-category, calculation method, etc.)
- No separate lookup tables or configuration files needed
- Keys can be decoded/audited manually (e.g., `1211020500` → "report=1(normal), stat=2(SP/duel?), data=1(uint32), calc=1(add), category=02(weapon), sub=05(rocket), ord=00")

**Why this design?**
- **Bandwidth efficiency**: If keys are transmitted to a remote server, numeric encoding is more compact than string-based alternatives
- **Type safety at call-site**: Different key types cannot be confused or misassigned
- **Protocol flexibility**: New categories/stat types can be added by using previously-unused digit combinations without breaking existing deployed code
- **Deterministic, no runtime overhead**: No data structures or initialization required; keys are compile-time constants

**Tradeoffs made:**
- **Readability sacrifice**: `QGR_KEY_FRAG_ROCKET` (1211020500) is harder to read than a semantic name, requiring documentation cross-reference
- **Fixed capacity**: The 10-digit format limits extensibility; if the stat vocabulary grows beyond ~10 billion unique combinations per dimension, the scheme breaks
- **No versioning mechanism**: The protocol has no version field, so incompatible protocol changes require careful server-side rollout coordination
- **Rigid structure**: Digit positions cannot be repurposed or extended without a major protocol revision

## Data Flow Through This File

**Incoming data sources:**
- Per-frame combat events during gameplay: `g_combat.c` tracks damage given/taken, `g_weapon.c` records shots fired and hits, `g_items.c` logs item pickups, `g_team.c` fires CTF events (flag capture, teammate damage)
- Entity lifecycle events: entity deaths, suicides, environmental kills
- Session metadata: game type, map, limits, max clients, ping ranges (from spawn/init code)

**Transformation:**
- Raw stat values are accumulated in server-side player/session structures during the match frame loop
- At end-of-match (or periodic intervals), collected stats are paired with the appropriate `QGR_KEY_*` constant
- The key's "calculation" field (encoded in 10^6 position) determines aggregation: raw value, sum total, average, max, or min

**Outgoing to external system:**
- `sv_rankings.c` batches (key, value) pairs into a network message
- Submits asynchronously to the remote GRank API (not defined in this codebase; external HTTP/UDP service)
- Remote server decodes keys using the same digit-position scheme, indexes stats by player/session, computes ELO/rating deltas

## Design Patterns & Learning Insights

**What a developer learns from this file:**
1. **Self-describing identifiers** — Keys embed their own metadata; no separate schema registry needed. Modern systems (GraphQL, Protocol Buffers, JSON Schema) separate schema from data, but this 1990s design chose embedding for simplicity and compactness.
2. **Schema versioning via conditional compilation** — The `#ifdef MISSIONPACK` pattern allows the same codebase to export two distinct stat inventories without code duplication. Applied broadly, this is cumbersome, but for a stable protocol (Q3A released 2000, Team Arena 2001), it worked.
3. **Telemetry architecture** — Game servers rarely own player progression; delegating to an external service allows cross-server (cross-game) progression aggregation. This is a precursor to modern matchmaking backends.

**How modern engines differ:**
- **Unreal/Unity** use structured formats (JSON/msgpack) and external analytics platforms (Mixpanel, Sentry, DataDog), not custom numeric encodings
- **Cloud backends** (AWS GameLift, Azure PlayFab) provide versioned schema APIs and automatic schema evolution
- **gRPC/protobuf** define message schemas explicitly, not embedded in message IDs

**Game engine concepts:**
- This is **player telemetry** and **progression infrastructure** — the foundation for matchmaking, ELO ratings, and seasonal progression
- Related to **session persistence** (`g_session.c`) and **bot skill rating** (botlib weights driven by player stats)

## Potential Issues

1. **No payload validation**: Nothing in the codebase checks that a key is valid before using it. A typo or accidental key misuse goes undetected at compile time.

2. **Semantic ambiguity in categories**: The header comments describe digit meanings (e.g., category 08 = "hazard", category 09 = "reward"), but don't document what each category *means* semantically or how rewards/hazards differ from weapons/ammo. A developer must reverse-engineer from usage.

3. **Apparent typo in MISSIONPACK section**: `QGR_KEY_FRAG_NAILGIN` (line ~340) appears to be `NAILGUN` misspelled. This is present in the original released Q3 source, suggesting it persisted through both base and Team Arena releases.

4. **No protocol version field**: If the remote GRank server evolves (e.g., adds new calculation modes), the local server has no way to communicate its capability level. Breaking changes require coordinated deployment across all game servers.

5. **Fixed-capacity design**: The 10-digit format is theoretically exhausted if new weapons, items, or game modes push stat variants beyond the planned ranges. Extending to 11+ digits would require all clients and servers to update simultaneously.

6. **Dead data**: This file defines keys for base-Q3A weapons (Grapple, BFG) that may not exist in all game variants or mods. Unused keys still occupy the namespace and must be accounted for in protocol discussions.
