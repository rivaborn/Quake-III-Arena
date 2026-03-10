# code/ui/keycodes.h

## File Purpose
Defines the canonical enumeration of all input key codes used by the Quake III Arena UI and input systems. It provides a hardware-agnostic numeric identity for every bindable input, including keyboard keys, mouse buttons, scroll wheel, joystick buttons, and auxiliary inputs.

## Core Responsibilities
- Define `keyNum_t`, the master enum of all recognized input identifiers
- Anchor ASCII-compatible keys at their literal ASCII values (Tab=9, Enter=13, Escape=27, Space=32)
- Enumerate extended keys (function keys, numpad, arrows, modifiers) starting at 128
- Enumerate mouse, scroll wheel, joystick (32 buttons), and auxiliary (16) inputs
- Define `K_CHAR_FLAG` bitmask to distinguish character events from key events in the menu system
- Assert via comment that `K_LAST_KEY` must remain below 256

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `keyNum_t` | enum (typedef) | Numeric identity for every bindable input; used as the canonical key token throughout the input and UI pipelines |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header defining constants and types only.

## Control Flow Notes
This header is a passive data definition. It is included by:
- The **input layer** (`cl_keys.c`, platform input files) to translate raw OS/hardware events into `keyNum_t` values before dispatching via `KeyEvent`.
- The **UI/menu system** (`ui_*.c`, `q3_ui/`) to interpret bound keys and handle menu navigation.
- `K_CHAR_FLAG` (value `1024`) is OR'd into a key number at the call site when a text character event (rather than a raw key press) is being delivered to the menu, allowing a single `KeyEvent`-style dispatch path to carry both event types.

## External Dependencies
- No includes. Self-contained.
- `keyNum_t` values are consumed by:
  - `KeyEvent()` — defined elsewhere in the client/input layer
  - Key-binding tables in `cl_keys.c`
  - Menu input handlers in `ui_main.c` / `ui_atoms.c`
