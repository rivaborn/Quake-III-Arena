# code/game/g_rankings.h

## File Purpose
Defines a comprehensive set of numeric key constants used to report per-player and per-session statistics to a global online rankings/scoring backend. Each key encodes metadata about the stat's type, aggregation method, and category directly within its numeric value.

## Core Responsibilities
- Define all `QGR_KEY_*` constants for the rankings reporting system
- Encode stat semantics (report type, stat type, data type, calculation method, category) into each key's decimal digits
- Provide per-weapon stat keys for all 10 base weapons (Gauntlet through Grapple) plus unknowns
- Conditionally define `MISSIONPACK`-exclusive keys for Team Arena weapons, ammo, powerups, and holdables
- Provide keys for session metadata (hostname, map, gametype, limits)
- Provide keys for hazards, rewards, CTF events, and teammate interaction

## Key Types / Data Structures

None (header-only; all definitions are `#define` preprocessor constants).

## Global / File-Static State

None.

## Key Functions / Methods

None (no functions declared or defined).

## Control Flow Notes

This is a pure constants header with no executable code. It is `#include`-d by the game module (primarily `g_rankings.c`) which calls into a rankings API, passing these key IDs alongside stat values. The keys are used at end-of-match or during gameplay to submit structured stat records. The `MISSIONPACK` preprocessor guard gates Team Arena–specific keys, so the same header serves both base Q3A and Team Arena builds.

## External Dependencies

- No includes.
- The key encoding scheme implies an external global rankings server/API (not defined here) that interprets the numeric key structure.
- `MISSIONPACK` macro defined externally (build system / project settings) to enable Team Arena extensions.

---

**Key encoding schema** (decoded from the header comment):

| Digit position | Meaning | Notable values |
|---|---|---|
| 10⁹ | Report type | 1=normal, 2=dev-only |
| 10⁸ | Stat type | 0=match, 1=single-player, 2=duel |
| 10⁷ | Data type | 0=string, 1=uint32 |
| 10⁶ | Calculation | 0=raw, 1=add, 2=avg, 3=max, 4=min |
| 10⁴–10⁵ | Category | 00=general, 02=weapon, 09=reward, 11=CTF, etc. |
| 10²–10³ | Sub-category | weapon index (×100) or item tier |
| 10⁰–10¹ | Ordinal | stat variant within category |
