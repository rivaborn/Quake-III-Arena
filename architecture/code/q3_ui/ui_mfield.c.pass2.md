# code/q3_ui/ui_mfield.c — Enhanced Analysis

## Architectural Role

This file implements the foundational text field widget for Q3's QVM-hosted UI layer. Within the broader engine, it occupies a unique niche: **immediate-mode rendering + event-driven input handling within a sandboxed VM**. Unlike the game (server) or cgame (client) VMs which drive physics simulation, the UI VM is purely reactive—it responds to keypresses and timer ticks to update menu display. This file's core `mfield_t` model is deliberately simple to remain VM-efficient; all rendering and input dispatch happen through indexed `trap_*` syscalls back to the engine.

## Key Cross-References

### Incoming (who depends on this file)
- **Menu framework** (`ui_qmenu.c` / `ui_atoms.c`): Calls `MenuField_Init` during menu tree construction, and `MenuField_Draw`/`MenuField_Key` each frame during menu display. The file depends on `Menu_ItemAtCursor` to detect focus state.
- **Menu system entry points** (`ui_main.c`): Routes user input through `MenuField_Key` when a menufield has focus.
- **Anywhere a text field is needed** within the UI (login, CD key entry, player name, server address, etc.).

### Outgoing (what this file depends on)
- **Trap syscalls** (engine-side implementations):
  - `trap_GetClipboardData`, `trap_Key_GetOverstrikeMode`, `trap_Key_SetOverstrikeMode`, `trap_Key_IsDown` — **Input/OS integration** (sandboxed queries)
  - `trap_Error` — **VM→engine error reporting** (triggers engine longjmp)
  - `UI_DrawString`, `UI_DrawChar`, `UI_FillRect` — **Renderer syscalls** (likely `trap_R_*` at the ABI level)
- **Menu framework functions** (in-VM): `Menu_ItemAtCursor` (focus detection), implied `menuframework_s` global state
- **Standard C** (linked into QVM): `strlen`, `memmove`, `memcpy`, `tolower` — **Safe VM library functions**
- **Color/constant externs** (defined elsewhere in UI): `text_color_*`, `listbar_color`, key constants (`K_*`), style flags (`UI_*`)

## Design Patterns & Rationale

1. **Two-layer widget abstraction:** `mfield_t` (reusable field model) + `menufield_s` (menu-system wrapper). Allows console code or in-game chat to reuse `mfield_t` independently from the menu framework.

2. **Global overstrike mode via trap**: `trap_Key_GetOverstrikeMode()` and `trap_Key_SetOverstrikeMode()` imply the insert/overstrike state is **engine-global**, not per-field. This is era-appropriate (Quake series tradition) but differs from modern UI frameworks where mode is typically field-local or application-scoped.

3. **Scroll-to-cursor guarantee:** `MField_Draw` mutates `edit->scroll` before rendering to ensure the cursor is always visible. This is an invariant-maintenance pattern—the model enforces visual correctness on every draw.

4. **Immediate-mode rendering:** No retained geometry; every frame recomputes visible substring and draws it fresh. Efficient in the QVM context but would be prohibitive in a high-frequency engine loop.

## Data Flow Through This File

```
User Input (keystroke)
   ↓
MenuField_Key()  [routes key → character or special]
   ↓
   ├─→ MField_KeyDownEvent()  [non-printables: arrows, del, home/end, insert toggle]
   │      ↓ mutates: edit→{buffer, cursor, scroll}
   │
   └─→ MField_CharEvent()  [printables + Ctrl combos]
      ↓ mutates: edit→{buffer, cursor, scroll}
      ↓ may call: MField_Paste(), MField_Clear()

Render Loop (each frame)
   ↓
MenuField_Draw()
   ├─→ computes focus state via Menu_ItemAtCursor()
   ├─→ draws highlight rect + label + cursor glyph
   └─→ calls MField_Draw()
      ↓
      MField_Draw()  [also mutates edit→scroll to guarantee cursor visibility]
      ├─→ adjusts x origin for CENTER/RIGHT alignment
      └─→ calls trap_R_{DrawString,DrawChar}()

Paste (Shift+Insert or Ctrl+V)
   ↓
MField_Paste()
   ├→ trap_GetClipboardData(buffer, 64)
   └→ for each character: MField_CharEvent()  [respects insert/overstrike/maxchars]
```

## Learning Notes

**Idiomatic to Q3 / 2000s era:**
- **Overstrike as global mode** (not per-field) — reflects original game console conventions.
- **Fixed 64-char clipboard buffer** — appropriate for typical Q3 use (server address, player name) but would be too small for pasting config files.
- **Control-character combos encoded in low bytes** (`'h' - 'a' + 1` = 0x08 = Ctrl+H). Unusual but avoids the need for separate modifier tracking in `MField_CharEvent`.
- **Immediate-mode 2D rendering** — every UI element is drawn fresh each frame; no scene graph, no dirty flags.

**Modern engines do differently:**
- Store overstrike/cursor blink state **per-widget** or in a global **input context**, not queried via trap.
- Support **multi-line text** or at least UTF-8; Q3 is ASCII-only.
- Use **retained draw lists** with caching to avoid recomputing layout every frame.
- **Batch clipboard operations** rather than char-by-char pasting.
- **Separate input handling from rendering** more cleanly (MVC, reactive architecture).

**Game engine concepts:**
- This is a classic **widget toolkit** — every commercial engine (UE, Unity, Godot) includes similar menu/text-field machinery.
- The `mfield_t` is a minimal **data model**; rendering and input handling are the **view** and **controller**.
- Scroll management demonstrates **viewport constraints** — a common pattern in bounded-display rendering.

## Potential Issues

1. **Cursor positioning in CENTER/RIGHT alignment edge case**: Lines 93–105 adjust the x origin for alignment *before* drawing the cursor. The logic is correct, but a developer unfamiliar with how x is mutated could accidentally double-apply the offset.

2. **Duplicate Ctrl+A/E handling:** `MField_KeyDownEvent` catches these via `tolower(key) == 'a' && trap_Key_IsDown(K_CTRL)`, while `MField_CharEvent` catches the control-character encoding. Both paths work, but it's unclear which takes precedence (depends on whether raw keycodes or character codes are delivered first). Not a bug, but adds maintenance burden.

3. **Maxchars interaction with null terminator:** In line 238, cursor is only incremented if `edit->cursor < edit->maxchars - 1`, reserving the last slot for the null terminator. However, overstrike mode (line 234) only checks `len >= maxchars`, not `edit->cursor`. This asymmetry could allow the buffer to be overwritten if overstrike inserts beyond intended bounds — though in practice it's safeguarded by the earlier length check.

4. **No bounds validation on `prestep` in MField_Draw:** Line 62 ensures `drawLen ≤ len`, but if a caller manually corrupts `edit->scroll`, line 65's `memcpy( str, edit->buffer + prestep, drawLen )` could read past the buffer. VM sandbox protects memory safety, but logic safety is caller's responsibility.
