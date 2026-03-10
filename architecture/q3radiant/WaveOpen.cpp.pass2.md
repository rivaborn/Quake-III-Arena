# q3radiant/WaveOpen.cpp — Enhanced Analysis

## Architectural Role

WaveOpen.cpp is a specialized modal dialog component of Q3Radiant's asset browser layer, enabling mappers to audition WAV files during level design. It sits entirely outside the runtime engine subsystems—the editor is a standalone Windows/MFC application used to produce `.bsp` and `.ent` files that the engine consumes. This file represents the editor's minimal audio integration: a single-purpose file dialog wrapper that wraps the OS-level `sndPlaySound` API for real-time preview.

## Key Cross-References

### Incoming (who depends on this file)
- Q3Radiant application framework (unknown parent code; would instantiate `CWaveOpen` when user initiates audio asset selection)
- Likely called from a sound/audio property editor or level entity configuration panel

### Outgoing (what this file depends on)
- **Windows MFC framework**: `CFileDialog` base class, `CWnd`, message map macros (`IMPLEMENT_DYNAMIC`, `BEGIN_MESSAGE_MAP`)
- **Windows multimedia API**: `mmsystem.h` for `sndPlaySound()`, `SND_FILENAME`, `SND_ASYNC`
- **Radiant application headers**: `Radiant.h`, `StdAfx.h` (precompiled headers, MFC initialization)
- **Resource IDs**: `IDD_PLAYWAVE`, `IDC_BTN_PLAY` (defined in `.rc` resource file, not visible in source)

## Design Patterns & Rationale

**MFC Dialog Customization Pattern**: Extends `CFileDialog` rather than building a custom dialog from scratch, leveraging the OS file picker UI while injecting custom logic via message overrides (`OnFileNameChange`, `OnBtnPlay`, `OnInitDialog`).

**Enable/Disable Button Gating**: The play button is disabled until a `.wav` file is selected (checked by simple string suffix match). This is a UX guard preventing errors when invoking `sndPlaySound()` on non-audio files or empty paths.

**Async Audio Playback**: Uses `SND_ASYNC` flag to avoid blocking the UI during preview; a prior `sndPlaySound(NULL, NULL)` stops any playing sound before starting the new one.

**Why this structure?** Radiant was built circa 2000 using MFC (Windows-only tooling). The editor's audio preview was a secondary feature—just enough to let mappers test sound placements in real time without leaving the editor or recompiling the map.

## Data Flow Through This File

1. **User initiates**: Opens the audio file picker via menu/button in parent Radiant window
2. **Dialog construction**: `CWaveOpen` constructor configures the MFC file dialog with template `IDD_PLAYWAVE` and `OFN_EXPLORER` | `OFN_ENABLETEMPLATE` flags
3. **Initialization**: `OnInitDialog()` disables the play button initially (no file selected)
4. **User navigates**: As the user selects files, `OnFileNameChange()` fires; if the path ends in `.wav`, button is enabled; otherwise disabled
5. **User previews**: Click play → `OnBtnPlay()` calls `sndPlaySound(path, SND_ASYNC)`; system's audio driver plays the file in the background
6. **User confirms or cancels**: Standard `CFileDialog` completion; parent receives the selected path (or cancellation)

## Learning Notes

**Idiomatic to this era/engine**:
- **Heavy MFC dependency**: Q3Radiant is a pure Windows MFC application (no cross-platform editor UI). Modern engines (Unreal, Unity) use platform-agnostic frameworks or web-based editors.
- **Synchronous resource loading implied**: The file picker and audio preview are all synchronous blocking operations; no async task queue or threading model visible. Modern editors offload file I/O to worker threads.
- **Simple suffix-based type checking** (`str.Find(".wav")`): No MIME type or binary header validation. Robust systems would inspect file magic bytes or use `GetOpenFileName` file-type filters.

**Game engine concepts**:
- **Tool-vs-engine separation**: The editor is completely decoupled from the runtime engine. It produces artifacts (`.bsp`, `.ent`) that the engine consumes, but shares no code. Modern engines tightly couple the editor and runtime for live editing.
- **Asset preview pattern**: Allowing mappers to audition assets in situ (without leaving the editor) is a foundational UX principle, still used in modern editors (e.g., Unreal's content browser audio preview).

## Potential Issues

- **No error handling**: `sndPlaySound()` can fail silently (invalid file, no audio device, permission denied). No user feedback if preview fails.
- **Hard-coded `.wav` suffix check**: Case-sensitive on case-sensitive filesystems; doesn't validate file headers or MIME type.
- **Resource ID fragility**: Assumes `IDD_PLAYWAVE` and `IDC_BTN_PLAY` resource IDs exist and are correct; mismatch causes runtime failures with no compile-time check.
- **No async cancellation**: If a large WAV is playing and the user closes the dialog, the OS continues playback in the background (minor UX annoyance, not a crash).
