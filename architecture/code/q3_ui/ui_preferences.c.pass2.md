# code/q3_ui/ui_preferences.c — Enhanced Analysis

## Architectural Role

This file implements a **cvar-to-UI bridge** for gameplay and rendering preferences. It sits in the `q3_ui` VM and translates between two domains: the player-facing menu system (widgets, events, rendering) and the engine's cvar subsystem (persistent game settings). The menu is **state-read-only on init**, **event-driven on interaction**, and **entirely decoupled from per-frame updates**—changes propagate directly through the cvar system and take effect next frame or on recompile (e.g., shaders).

## Key Cross-References

### Incoming (who calls this file's functions)

- **`UI_PreferencesMenu`** is exposed as a public entry point; called from the main menu flow in `code/q3_ui/ui_main.c` (inferred—exact call site not in cross-ref excerpt but follows Q3A menu frame pattern)
- **`Preferences_Cache`** is called during asset precaching phase (likely from a global `UI_Cache()` or `UI_Init()` dispatcher in `ui_atoms.c`)
- **`Preferences_MenuInit`** is called exclusively from `UI_PreferencesMenu`, tightly coupled

### Outgoing (what this file depends on)

- **Cvar layer** (`trap_Cvar_VariableValue`, `trap_Cvar_SetValue`, `trap_Cvar_Reset`): Each widget read/write maps directly to an engine cvar. The file implements a **bidirectional sync**: reads on init, writes on user activation
- **Renderer syscalls** (`trap_R_RegisterShaderNoMip`): Fills `crosshairShader[]` handles; **no validation** of success (handle could be invalid if shader load fails)
- **Menu framework** (`Menu_AddItem`, `UI_PopMenu`, `UI_PushMenu`): Called from `Preferences_MenuInit` and `Preferences_Event` (ID_BACK case); provides the container for this menu's widgets
- **UI rendering** (`UI_FillRect`, `UI_DrawChar`, `UI_DrawString`, `UI_DrawHandlePic`): Only in `Crosshair_Draw` for owner-draw; demonstrates that complex widgets can hook the draw pipeline
- **Utility** (`Com_Clamp`, `va`): Used for clamping team overlay indices and formatting shader names

## Design Patterns & Rationale

### 1. **Cvar-as-Model**
The file treats cvars as the single source of truth for game state. Rather than maintaining a duplicate internal state cache, it reads from/writes to cvars in real-time. **Why**: In Q3A's architecture, cvars are global, persistent, and inherently serialized to `q3config.cfg`; duplicating state in the UI VM risks desynchronization.

### 2. **Owner-Draw for Complex Widgets**
The crosshair selector uses `QMF_OWNERDRAW` + custom `Crosshair_Draw` callback instead of a simple list. **Why**: It needs to render a live 24×24 graphical preview of the selected shader, not just a text label. This pattern is idiomatic in Q3A's menu framework for any widget that needs custom rendering logic.

### 3. **Lazy Asset Registration**
`Preferences_Cache` is called before menu init, registering all 10 crosshair shaders + UI art in advance. **Why**: Reduces hitches during menu interaction; shader compilation/caching happens off the critical path.

### 4. **Inverted Semantics: r_fastsky**
The cvar `r_fastsky` is stored **inverted** in the widget: `highqualitysky.curvalue = (r_fastsky == 0)`. **Why**: The player sees "High Quality Sky: On/Off" but the engine uses "fast sky: On/Off"; this inversion hides implementation details from the player. The event handler must invert back: `trap_Cvar_SetValue("r_fastsky", !s_preferences.highqualitysky.curvalue)`.

### 5. **Deferred Reset for Special Cases**
`ID_EJECTINGBRASS` calls `trap_Cvar_Reset("cg_brassTime")` when enabled, rather than setting a hardcoded value. **Why**: Respects the default cvar definition in the engine; if the default changes, the menu adapts automatically. However, this is **fragile** if the default is 0 (disabled).

## Data Flow Through This File

```
┌─ Engine Startup
│  └─ trap_R_RegisterShaderNoMip() × 14
│     (preload UI art + 10 crosshairs into renderer memory)
│
├─ Player Opens Preferences Menu
│  └─ UI_PreferencesMenu()
│     └─ Preferences_MenuInit()
│        ├─ Menu widget struct setup + layout
│        ├─ Menu_AddItem() × 14
│        └─ Preferences_SetMenuItems()
│           └─ trap_Cvar_VariableValue() × 11
│              (read: cg_drawCrosshair, cg_simpleItems, cg_brassTime, ...)
│              (populate widget.curvalue fields)
│        └─ UI_PushMenu()
│
├─ Player Activates Widget (e.g., crosshair list)
│  └─ Preferences_Event(ptr, QM_ACTIVATED)
│     ├─ Dispatch on ((menucommon_s*)ptr)->id
│     ├─ Mutate widget.curvalue
│     └─ trap_Cvar_SetValue("cg_drawCrosshair", newvalue)
│        (cvar broadcasts change; cgame + renderer react next frame)
│
└─ Rendering: If Focus
   └─ Crosshair_Draw()
      ├─ Render label + focus highlight
      └─ UI_DrawHandlePic() with s_preferences.crosshairShader[curvalue]
```

**Key insight**: Once a cvar is written, the change propagates **through the engine's cvar observer system** (likely `Cvar_Update` in `qcommon/cvar.c`) and takes effect globally. The UI doesn't manage re-rendering or state invalidation—the engine does.

## Learning Notes

1. **Syscall Boundary Discipline**: Every interaction crosses the VM sandbox via `trap_*`. There is no cheating or shared memory access (except the cvar values themselves).

2. **Widget Framework Vocabulary**: This file is a textbook example of Q3A's menu widget types:
   - `menutext_s` (static banner)
   - `menubitmap_s` (art + back button)
   - `menulist_s` (crosshair selector with owner-draw)
   - `menuradiobutton_s` (on/off toggles)
   - `menuradiobutton_s` with `MTYPE_SPINCONTROL` (team overlay cycler)

3. **Layout by Manual Y Increment**: All widget positioning is done with a simple `y` counter incremented by `BIGCHAR_HEIGHT + [padding]`. This is **not scale-invariant** and would break at different UI scales. Modern engines use layout managers or data-driven positioning; Q3A hardcodes it. This is a teaching moment about late-1990s game UI architecture.

4. **Idempotent Precaching**: `Preferences_Cache` can be called multiple times safely (shader handles are already in the renderer's cache after first call). It doesn't check for redundancy—the renderer does.

5. **No Bidirectional Observation**: Unlike modern engines with property bindings or observers, this file doesn't react to external cvar changes (e.g., if the server changes `cg_drawCrosshair`). Sync is **one-shot on menu init** only. This is acceptable because the UI menu is rarely open while gameplay is active.

## Potential Issues

### 1. **Silent Shader Load Failure**
`Preferences_Cache` doesn't validate that `trap_R_RegisterShaderNoMip` succeeded. If a shader is missing, `crosshairShader[n]` is a handle to a non-existent shader. `Crosshair_Draw` will then call `UI_DrawHandlePic` with an invalid handle. The renderer likely falls back to white or a placeholder, but there's no error message or recovery.

### 2. **Brass Cvar Reset Fragility**
```c
if (s_preferences.brass.curvalue)
    trap_Cvar_Reset("cg_brassTime");  // Assumes default != 0
else
    trap_Cvar_SetValue("cg_brassTime", 0);
```
If someone edits the engine's cvar defaults to make `cg_brassTime` default to 0, toggling brass on will **disable it** (not enable). A safer pattern is `trap_Cvar_SetValue("cg_brassTime", s_preferences.brass.curvalue ? 200 : 0)` (where 200 is the "enabled" milliseconds value).

### 3. **Dual Cvar Sync (Allow Download)**
```c
case ID_ALLOWDOWNLOAD:
    trap_Cvar_SetValue("cl_allowDownload", s_preferences.allowdownload.curvalue);
    trap_Cvar_SetValue("sv_allowDownload", s_preferences.allowdownload.curvalue);  // ← Inconsistent
    break;
```
This is the **only** setting that syncs two cvars. It hints at a design inconsistency: `cl_allowDownload` (client preference) and `sv_allowDownload` (server permission) are conflated in the UI. If `sv_allowDownload` is edited separately (e.g., by an admin), the UI will overwrite it next time the player touches this widget. Better: only write `cl_allowDownload` and let the server enforce its own policy.

### 4. **No Input Validation**
`Preferences_Event` doesn't validate incoming widget IDs before dispatching. A malformed or corrupted menu item could cause undefined behavior. In practice, all IDs are hardcoded, so this is low-risk, but it's a code-smell for robustness.

---

**Summary**: This file is a clean example of **state-machine UI driven by syscalls**. It's straightforward but reveals the limitations of Q3A's late-90s menu architecture: manual layout, hardcoded cvar mappings, and no reactive observers. Modern engines would use data-driven menu definitions and automatic model-view binding. Yet for its era, this design was pragmatic and maintainable.
