# code/q3_ui/keycodes.h

## File Purpose
Defines the `keyNum_t` enumeration mapping all recognized input sources (keyboard, mouse, joystick, aux) to integer key codes for use by the input and UI systems. It serves as the shared vocabulary for key event dispatch throughout the Q3 UI module.

## Core Responsibilities
- Enumerate all virtual key codes for keyboard special keys, function keys, numpad keys, mouse buttons, scroll wheel, joystick buttons, and auxiliary inputs
- Anchor low-ASCII printable/control keys at their ASCII values (TAB=9, ENTER=13, ESC=27, SPACE=32)
- Provide `K_LAST_KEY` as a sentinel/bounds-check value (must remain < 256)
- Define `K_CHAR_FLAG` bitmask to multiplex character events over the same key-event path

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `keyNum_t` | enum (typedef) | Canonical integer identifiers for all input events passed to `KeyEvent` |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header defining types and macros only.

## Control Flow Notes
This header is consumed by the UI module (`q3_ui/`) and the client input system. Key event handlers receive a `keyNum_t` value; the system or'ing in `K_CHAR_FLAG` allows the menu subsystem to receive both raw key events and translated character events on a single dispatch path without separate code routes.

Normal printable ASCII characters are passed as their lowercase ASCII values directly (below 128); special keys start at 128 (`K_COMMAND`) and extend upward. The constraint `K_LAST_KEY < 256` implies key arrays are sized at 256 entries elsewhere in the engine.

## External Dependencies
- No includes.
- `keyNum_t` values are consumed by: `KeyEvent` (defined elsewhere in the client/input layer), menu/UI event handlers (defined elsewhere in `q3_ui/`).
- `K_CHAR_FLAG` (value `1024`) is used by the menu code to distinguish char vs. key events — the or'ing logic lives outside this file.
