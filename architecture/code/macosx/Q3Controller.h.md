# code/macosx/Q3Controller.h

## File Purpose
Declares the `Q3Controller` Objective-C class, which serves as the macOS application controller (NSObject subclass) for Quake III Arena. It acts as the AppKit-facing entry point that bridges the macOS application lifecycle into the engine's main loop.

## Core Responsibilities
- Declares the main application controller class for the macOS platform
- Exposes an Interface Builder outlet for a splash/banner panel
- Provides IBActions for clipboard paste and application termination requests
- Declares `quakeMain` as the engine entry point invoked from the macOS app

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `Q3Controller` | Objective-C class (NSObject subclass) | macOS application controller; owns the banner panel outlet and drives engine startup |

## Global / File-Static State

None.

## Key Functions / Methods

### paste:
- **Signature:** `- (IBAction)paste:(id)sender`
- **Purpose:** Handles a paste action from the macOS menu or keyboard shortcut, forwarding clipboard text into the engine console or input system.
- **Inputs:** `sender` — the UI object that triggered the action
- **Outputs/Return:** `void` (IBAction)
- **Side effects:** Not inferable from this file; implementation in `Q3Controller.m`
- **Calls:** Not inferable from this file
- **Notes:** Conditionally compiled out for `DEDICATED` server builds

### requestTerminate:
- **Signature:** `- (IBAction)requestTerminate:(id)sender`
- **Purpose:** Handles a user-initiated quit action from the macOS menu/UI, triggering a clean engine shutdown.
- **Inputs:** `sender` — the UI object that triggered the action
- **Outputs/Return:** `void` (IBAction)
- **Side effects:** Expected to initiate engine/app shutdown; details in `Q3Controller.m`
- **Calls:** Not inferable from this file
- **Notes:** Conditionally compiled out for `DEDICATED` server builds

### showBanner
- **Signature:** `- (void)showBanner`
- **Purpose:** Displays the `bannerPanel` (likely a splash/loading screen) during startup.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Shows an `NSPanel`; modifies UI state
- **Calls:** Not inferable from this file
- **Notes:** Conditionally compiled out for `DEDICATED` server builds

### quakeMain
- **Signature:** `- (void)quakeMain`
- **Purpose:** The primary engine entry point called from the macOS application delegate or main thread; kicks off the Quake III main loop.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Runs the entire game engine; does not return during normal play
- **Calls:** Not inferable from this file; implementation in `Q3Controller.m`
- **Notes:** Available in both client and `DEDICATED` builds

## Control Flow Notes
This controller is instantiated by AppKit at launch (likely via a `.nib`/`Quake3.nib`). `showBanner` is called early during init to display a splash panel, followed by `quakeMain` which enters the engine loop. `paste:` and `requestTerminate:` respond to AppKit menu events during the session.

## External Dependencies
- `<AppKit/AppKit.h>` — AppKit framework (NSObject, NSPanel, IBOutlet, IBAction)
- `DEDICATED` — preprocessor macro defined externally to strip client-only UI code
- `Q3Controller.m` — implementation file (defined elsewhere)
- `Quake3.nib` — Interface Builder nib file that instantiates this controller and wires `bannerPanel` (defined elsewhere)
