# code/cgame/cg_draw.c — Enhanced Analysis

## Architectural Role

`cg_draw.c` is the HUD compositor layer sitting at the boundary between the cgame VM's game-state model and the renderer's 2D drawing API. It occupies a terminal position in the cgame data pipeline: all upstream systems (snapshot consumption in `cg_snapshot.c`, prediction in `cg_predict.c`, event processing in `cg_event.c`) exist to produce state that this file visualizes. The file is the sole orchestrator of the full 2D draw pass and the top-level entry point (`CG_DrawActive`) for the entire per-frame render sequence, making it the "main loop" of the cgame visual layer. It straddles two compile-time personalities — base Q3A and MissionPack (Team Arena) — which diverge substantially in text rendering, scoreboard, and team overlay logic, controlled entirely by `#ifdef MISSIONPACK`.

## Key Cross-References

### Incoming (who depends on this file)

- **`cg_view.c` → `CG_DrawActive`**: The per-frame render driver `CG_DrawActiveFrame` in `cg_view.c` is the sole caller of `CG_DrawActive`, making it the top-level entry to this entire file. The stereo-frame enum value passed controls eye offset computation.
- **`cg_snapshot.c` → `CG_AddLagometerSnapshotInfo`**: Called each time a snapshot is processed (or NULL on packet drop) to feed the lagometer ring buffer; this is a push model where snapshot processing drives the network diagnostic display.
- **`cg_view.c` → `CG_AddLagometerFrameInfo`**: Called once per rendered frame to record interpolation offset into the lagometer's frame sample ring. The split between frame and snapshot feeds allows the lagometer to independently visualize both prediction jitter and network latency.
- **`cg_servercmds.c` → `CG_CenterPrint`**: Server command parsing calls this to install center-screen messages; the state written (`cg.centerPrintTime`, `cg.centerPrintLines`) is then consumed each frame by `CG_DrawCenterString`.
- **`cg_scoreboard.c`**: Calls `CG_DrawHead`, `CG_DrawFlagModel`, and `CG_Draw3DModel` for scoreboard rendering; these functions are public precisely because scoreboards need 3D model/icon rendering identical to status bar rendering.
- **`cg_players.c` / `cg_ents.c`**: Read globals `sortedTeamPlayers` / `numSortedTeamPlayers` for team overlay display order. These are populated by sorting logic in this file.
- **`systemChat`, `teamChat1`, `teamChat2`**: Written by `cg_servercmds.c`; read here for HUD display. Simple global char arrays used as single-slot message slots.

### Outgoing (what this file depends on)

- **Renderer (`trap_R_*`)**: All visual output is via `trap_R_DrawStretchPic`, `trap_R_SetColor`, `trap_R_RenderScene`, `trap_R_ClearScene`, `trap_R_AddRefEntityToScene`, `trap_R_ModelBounds`. This file is the primary consumer of 2D renderer calls in the cgame.
- **`cg_drawtools.c`**: `CG_AdjustFrom640` (virtual 640×480 → screen coordinate transform), `CG_DrawPic`, `CG_DrawStringExt`, `CG_FadeColor`, `CG_ColorForHealth`, `CG_DrawBigString`. These are the "standard library" of drawing primitives.
- **`bg_misc.c`**: `BG_FindItemForPowerup` — shared game/cgame code for resolving powerup flag items by type; used in `CG_DrawFlagModel`.
- **`cg_scoreboard.c`**: `CG_DrawOldScoreboard`, `CG_DrawOldTourneyScoreboard` — the scoreboard rendering delegation under non-MISSIONPACK builds.
- **`cg_weapons.c`**: `CG_DrawWeaponSelect` called from `CG_Draw2D`.
- **`ui/ui_shared.h` (MISSIONPACK)**: `displayContextDef_t`, `menuDef_t`, `Menu_Paint`, `Menus_FindByName` — the MissionPack build delegates scoreboard rendering to the data-driven UI widget system, importing the same widget framework used by the UI VM. This is an unusual cross-module dependency where the cgame VM embeds UI widget parsing/rendering.
- **Global `cg` / `cgs` structs**: Consumes nearly all fields — `cg.snap`, `cg.time`, `cg.damageTime`, `cg.headStartYaw/Pitch/EndYaw/Pitch`, `cgs.clientinfo[]`, `cgs.media.*`, `cgs.localServer`, `cg.latestSnapshotTime`.

## Design Patterns & Rationale

- **Immediate-mode 2D rendering**: No retained widget tree. Each frame redraws every visible HUD element unconditionally. This matches the renderer's command-buffer model and avoids dirty-flag complexity, at the cost of per-frame CPU work.
- **Ring-buffer telemetry** (`lagometer_t`): The lagometer uses two independent power-of-two ring buffers (frame samples and snapshot samples) with separate counters. The widths are deliberately mismatched — frame samples are written every frame, snapshot samples only when a packet arrives — allowing the graph to independently visualize client-side prediction jitter vs. server round-trip latency on the same pixel column.
- **Hermite spline for head animation**: `CG_DrawStatusBarHead` uses `frac = frac*frac*(3-2*frac)` — a classic smooth-step (cubic Hermite basis) — to interpolate idle head yaw/pitch between randomly chosen targets. This is a simple procedural animation pattern requiring no keyframe data.
- **`#ifdef MISSIONPACK` compile-time split**: Rather than runtime branching, the two UI generations are resolved at compile time. The MissionPack path re-uses the `ui_shared.c` widget system directly in cgame (embedding it), while the base Q3A path uses bespoke hand-coded HUD drawing. This avoids shipping the widget interpreter overhead in the base game.
- **`RF_NOSHADOW | RDF_NOWORLDMODEL`**: `CG_Draw3DModel` uses these flags to create a mini-scene rendered into a subrectangle without touching the world BSP — a lightweight "picture-in-picture" pattern used consistently for 3D icon rendering.

## Data Flow Through This File

```
Server snapshots ──► cg_snapshot.c ──► cg.snap (playerState_t, entityState_t[])
                                              │
                    ┌─────────────────────────┘
                    ▼
           CG_DrawActive (per frame, called from cg_view.c)
                    │
                    ├── trap_R_RenderScene(cg.refdef)          [3D world scene]
                    │
                    └── CG_Draw2D()
                              │
                              ├── reads cg.snap->ps.*          [health, armor, ammo]
                              ├── reads cg.time                [fade/pulse timing]
                              ├── reads cgs.clientinfo[]       [player models, icons]
                              ├── reads lagometer ring buffers [network graph data]
                              │
                              └── trap_R_DrawStretchPic / trap_R_SetColor
                                         [all 2D HUD pixels]

Per-snapshot:  CG_AddLagometerSnapshotInfo → lagometer.snapshotSamples[]
Per-frame:     CG_AddLagometerFrameInfo    → lagometer.frameSamples[]
Server cmds:   CG_CenterPrint             → cg.centerPrint / cg.centerPrintTime
```

State enters primarily from `cg` and `cgs` globals (written by upstream snapshot/prediction code). Transformation is purely presentational — timing, coordinate adjustment, color computation. Output is exclusively through `trap_R_*` renderer syscalls; no state is written back to game-logic globals.

## Learning Notes

- **Virtual 640×480 coordinate space**: All positions are authored assuming 640×480 and remapped by `CG_AdjustFrom640`. This was the standard resolution-independence technique of the era; modern engines use resolution-independent unit systems or anchor points relative to safe areas.
- **No scene graph or layout engine**: HUD element positions are hard-coded pixel offsets from screen edges. The MissionPack partially addresses this by loading `.menu` scripts that declare widget positions declaratively — an early step toward data-driven UI that modern engines carry much further.
- **Stereo rendering via view-origin offset**: Stereo 3D is achieved in `CG_DrawActive` by shifting `cg.refdef.vieworg` laterally along the right axis by `cg_stereoSeparation / 2` before each eye's `trap_R_RenderScene` call. This is geometry-level stereo (not post-process), which correctly produces depth parallax but requires two full scene renders.
- **Lagometer as debugging artifact shipped to players**: The lagometer visualizes frame interpolation state and packet timing in a way that exposes the client-side prediction model. Modern games hide this behind net graphs accessible only via developer consoles.
- **cgame VM embeds UI widget system (MISSIONPACK)**: The MissionPack build of cgame includes compiled-in copies of `ui_shared.c` and related widget code. This means the scoreboard is rendered by the same widget interpreter used by the full UI VM — a form of code reuse across VM boundaries that requires maintaining two synchronized compilations of the shared code.
- **`deferred` client model cross-out**: `CG_DrawHead` checks `ci->deferred` and overlays `cgs.media.deferShader` (a red X sprite). This is the visual indicator for the late-download player model fallback system, where an unknown player skin is replaced with the default until the asset arrives.

## Potential Issues

- **`lagometer.frameSamples` fixed-size array with unbounded counter**: The ring index is `lagometer.frameCount & (LAG_SAMPLES-1)`. If `frameCount` is `int` and wraps at `INT_MAX` after ~2 billion frames (impossible in practice but technically undefined behavior in C), the mask behavior still holds since power-of-two modulo is safe with unsigned semantics. No practical concern.
- **`CG_ScanForCrosshairEntity` ray-cast every frame**: A full `trap_CM_BoxTrace` is issued every rendered frame for crosshair name display. On slow hardware this adds measurable cost; modern engines typically amortize such traces over multiple frames.
- **`systemChat`/`teamChat1`/`teamChat2` are unguarded global char arrays**: Written by server command parsing and read by draw code with no length checking beyond the fixed 256-byte size. If server commands exceed this, silent truncation occurs at the write site in `cg_servercmds.c`.
