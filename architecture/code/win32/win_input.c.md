# code/win32/win_input.c

## File Purpose
Win32-specific input handling for Quake III Arena, managing mouse (both Win32 raw and DirectInput), joystick, and MIDI controller input. It translates hardware input events into engine-queued system events via `Sys_QueEvent`.

## Core Responsibilities
- Initialize, activate, deactivate, and shut down Win32 mouse and DirectInput mouse
- Poll DirectInput buffered mouse data and queue button/wheel/motion events
- Initialize and poll Win32 Multimedia joystick API, mapping axes and buttons to key events
- Initialize and receive MIDI input, mapping MIDI notes to aux key events
- Per-frame input dispatch (`IN_Frame`), including delayed DirectInput init fallback
- Register input-related cvars (`in_mouse`, `in_joystick`, `in_midi`, etc.)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `WinMouseVars_t` | struct | Tracks mouse active/initialized state and old button bitmask |
| `MidiInfo_t` | struct | Holds MIDI device count, capabilities array, and open MIDI handle |
| `joystickInfo_t` | struct | Joystick availability, ID, capabilities, button/POV state, and last sample |
| `MYDATA` | struct | Custom DirectInput data format layout (X/Y/Z axes + 4 buttons) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_wmv` | `WinMouseVars_t` | static | Mouse subsystem state |
| `window_center_x/y` | `int` | static | Screen-center coords for Win32 cursor recentering |
| `s_midiInfo` | `MidiInfo_t` | static | MIDI device enumeration and open handle |
| `joy` | `joystickInfo_t` | static | Single joystick state |
| `g_pdi` | `LPDIRECTINPUT` | static | DirectInput interface pointer |
| `g_pMouse` | `LPDIRECTINPUTDEVICE` | static | DirectInput mouse device |
| `hInstDI` | `HINSTANCE` | static | Loaded `dinput.dll` module handle |
| `in_appactive` | `qboolean` | global | Tracks whether the application window is active |
| `in_mouse`, `in_joystick`, `in_midi`, etc. | `cvar_t *` | global | User-facing input configuration cvars |
| `joyDirectionKeys[16]` | `int[]` | global | Maps joystick POV/axis bit positions to Q3 key codes |

## Key Functions / Methods

### IN_Init
- Signature: `void IN_Init(void)`
- Purpose: Registers all input cvars and calls `IN_Startup`.
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates cvars, registers `midiinfo` console command, calls `IN_Startup`.
- Calls: `Cvar_Get`, `Cmd_AddCommand`, `IN_Startup`
- Notes: Entry point called once at client startup.

### IN_Startup
- Signature: `void IN_Startup(void)`
- Purpose: Initializes all three input subsystems (mouse, joystick, MIDI).
- Inputs: None
- Outputs/Return: None
- Side effects: Sets `in_mouse->modified` and `in_joystick->modified` to false after init.
- Calls: `IN_StartupMouse`, `IN_StartupJoystick`, `IN_StartupMIDI`

### IN_StartupMouse
- Signature: `void IN_StartupMouse(void)`
- Purpose: Detects OS version, attempts DirectInput init, falls back to Win32 mouse.
- Inputs: None
- Outputs/Return: None
- Side effects: Sets `s_wmv.mouseInitialized`/`mouseStartupDelayed`; may set `in_mouse` cvar to `-1`.
- Calls: `IN_InitDIMouse`, `IN_InitWin32Mouse`, `Cvar_Set`
- Notes: Disables DI on NT 4.0. Sets `mouseStartupDelayed` if `g_wv.hWnd` is not yet available.

### IN_InitDIMouse
- Signature: `qboolean IN_InitDIMouse(void)`
- Purpose: Loads `dinput.dll`, creates DI interface and mouse device, sets data format, cooperative level, and buffer size.
- Inputs: None
- Outputs/Return: `qtrue` on success, `qfalse` on any failure.
- Side effects: Allocates `g_pdi`, `g_pMouse`; calls `IN_DIMouse` twice to flush pending samples.
- Calls: `LoadLibrary`, `GetProcAddress`, `IDirectInput_CreateDevice`, `IDirectInputDevice_SetDataFormat/SetCooperativeLevel/SetProperty`, `IN_DIMouse`

### IN_DIMouse
- Signature: `void IN_DIMouse(int *mx, int *my)`
- Purpose: Drains DirectInput buffered event queue for button/wheel events; reads raw delta state for motion.
- Inputs: Pointers for X and Y delta output.
- Outputs/Return: `*mx`, `*my` set to raw axis deltas.
- Side effects: Calls `Sys_QueEvent` for each button press/release and mouse wheel tick; attempts re-acquire on lost device.
- Calls: `IDirectInputDevice_GetDeviceData`, `IDirectInputDevice_Acquire`, `IDirectInputDevice_GetDeviceState`, `Sys_QueEvent`
- Notes: Uses `DIDEVICEOBJECTDATA` buffered reads for buttons/wheel; `DIMOUSESTATE` snapshot for axes.

### IN_Frame
- Signature: `void IN_Frame(void)`
- Purpose: Per-frame input pump — handles delayed mouse init, deactivates mouse in console/unfocused states, posts motion events.
- Inputs: None
- Outputs/Return: None
- Side effects: May call `IN_StartupMouse`, `IN_DeactivateMouse`, `IN_ActivateMouse`, `IN_MouseMove`, `IN_JoyMove`.
- Calls: `IN_JoyMove`, `IN_StartupMouse`, `IN_DeactivateMouse`, `IN_ActivateMouse`, `IN_MouseMove`, `Cvar_VariableValue`, `Cvar_VariableString`
- Notes: Checks `cls.keyCatchers & KEYCATCH_CONSOLE` and `r_fullscreen` to suppress mouse capture in windowed console mode.

### IN_JoyMove
- Signature: `void IN_JoyMove(void)`
- Purpose: Polls joystick state, maps axis/button/POV changes to queued key events and optional mouse-ball SE_MOUSE events.
- Inputs: None
- Outputs/Return: None
- Side effects: `Sys_QueEvent` calls for button and direction changes; updates `joy.oldbuttonstate`, `joy.oldpovstate`.
- Calls: `joyGetPosEx`, `Sys_QueEvent`, `JoyToF`, `JoyToI`

### MidiInProc
- Signature: `static void CALLBACK MidiInProc(HMIDIIN, UINT uMsg, DWORD, DWORD dwParam1, DWORD dwParam2)`
- Purpose: MIDI input callback; decodes note-on/off messages on the configured channel and queues key events.
- Inputs: Standard Win32 MIDI callback parameters.
- Side effects: Calls `MIDI_NoteOn` / `MIDI_NoteOff` → `Sys_QueEvent`.
- Notes: Runs on a system MIDI thread; only `MIM_DATA` messages are acted upon.

## Control Flow Notes
- **Init**: `IN_Init` → `IN_Startup` called once during client startup.
- **Frame**: `IN_Frame` is called every client frame from the Win32 main loop; it drives joystick polling and conditionally activates/deactivates mouse and posts mouse-move events.
- **Window messages**: `IN_MouseEvent` is called from the Win32 window procedure (`MainWndProc`) for WM_MOUSEMOVE/button messages when DI is not active.
- **Shutdown**: `IN_Shutdown` deactivates mouse, releases DI objects, and closes MIDI.

## External Dependencies
- `../client/client.h` — `cls` (keyCatchers), `Cvar_*`, `Cmd_*`, `Com_Printf`, `Sys_QueEvent`, key constants
- `win_local.h` — `g_wv` (hWnd, hInstance, osversion, sysMsgTime), Win32/DInput/DSound headers
- Win32 APIs: `dinput.dll` (loaded dynamically), `winmm` (joystick/MIDI via `joyGetPosEx`, `midiInOpen`)
- Defined elsewhere: `Sys_QueEvent`, `Cvar_Set/Get/VariableValue`, `Com_Printf`, `Com_Memset`, `g_wv`, `cls`
