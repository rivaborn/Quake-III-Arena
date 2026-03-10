# q3radiant/PlugIn.h — Enhanced Analysis

## Architectural Role

This file defines the **plugin management abstraction for Q3Radiant**, the offline level editor. `CPlugIn` acts as a runtime wrapper around dynamically loaded editor extensions (surface properties, BSP compilers, texture loaders, custom entity editors). It mirrors the engine's own DLL-loading patterns (renderer, game VM) but operates entirely offline; plugins here extend *editor* capabilities, not the game engine itself.

## Key Cross-References

### Incoming (who depends on this)
- **q3radiant main UI** (`Win_main.cpp`, `MainFrm.cpp`, plugin manager code): instantiates `CPlugIn` objects, loads/unloads plugin DLLs, dispatches editor commands to plugins
- **q3radiant PluginManager** (`PlugInManager.h/cpp`): owns a collection of `CPlugIn` instances; routes UI events and commands to the appropriate plugin

### Outgoing (what this depends on)
- **Windows DLL loader** (`HMODULE`, `GetProcAddress`): loads plugin DLLs dynamically; uses MSVC runtime
- **Plugin DLL interface** (implied by function pointers): plugins must export symbols matching the `PFN_QERPLUG_*` function signatures
- **Engine texture/shader system** (implied by `getTextureInfo()`, `loadTexture()`, `getSurfaceFlags()`): plugins can query/load textures and surface properties, suggesting callbacks into the editor's asset pipeline
- **Editor entity system** (via `_QERPlugEntitiesFactory`, `CreatePluginEntity`): allows plugins to register custom entity types for the editor

## Design Patterns & Rationale

**Function-Pointer Vtable Adapter**
- Stores raw C-style function pointers (`PFN_QERPLUG_*`) loaded from DLL exports
- Wraps them in a C++ class interface (`load()`, `getCommandCount()`, `dispatchCommand()`, etc.)
- Rationale: Maintains C-compatible plugin ABI while providing C++ convenience; identical pattern to how the engine abstracts renderer, VM, botlib via `*_export_t` structs

**Two-Phase Initialization**
- `load(const char *p)` opens DLL and resolves symbols
- Plugin-specific init methods (`InitBSPFrontendPlugin()`, `InitSurfacePlugin()`, `RegisterPluginEntities()`) called afterward
- Rationale: Allows querying basic metadata before full initialization; mirrors engine's lazy-loading of subsystems

**Command Dispatch Pattern**
- Plugins provide indexed commands (ID → string mapping via `addMenuID()`, `ownsCommandID()`)
- `dispatchCommand()` routes commands by name with bounding-box context
- Rationale: Decouples editor UI from plugin implementation; plugins don't need to know editor's menu structure

## Data Flow Through This File

1. **Load Phase**: Editor calls `load("plugin.dll")` → resolves all `PFN_*` function pointers via `GetProcAddress` → calls plugin's `Init` function via `m_pfnInit`
2. **Metadata Phase**: `getMenuName()`, `getVersionStr()`, `getCommandCount()`, `getCommand(n)` queried to populate editor UI
3. **Runtime Phase**: Editor UI dispatches commands → `dispatchCommand()` invokes plugin's `m_pfnDispatch` with bounding box context (for spatial-dependent operations)
4. **Shutdown Phase**: `free()` releases DLL via `FreeLibrary()`

## Learning Notes

**Idiomatic to Quake III's era (pre-2005):**
- Explicit function-pointer dispatch rather than COM interfaces (which would be more OOP but Windows-specific)
- Raw `HMODULE` + `GetProcAddress` over modern DLL binding; no linker-time dependency, full runtime flexibility
- MFC (`CObject`, `CString`, `CWordArray`) for UI plumbing—Q3A used MFC for Win32 UI pre-dating .NET

**How modern engines differ:**
- Modern editors (Unreal, Unity) use plugin SDKs with stable ABI versioning, reflection metadata, and package managers
- Q3's plugin ABI is hand-rolled: breaking changes require plugin rebuild
- No dependency injection; plugins get context only via function parameters (e.g., bounding box in `dispatchCommand`)

**Connections to engine architecture:**
- This IS the editor's equivalent of the engine's `refexport_t` (renderer), `vmexport_t` (game), `botlib_export_t` (AI)
- The shared `.shader`, `.surf`, and BSP entity patterns flow through plugins: plugins can load textures, define surface flags, instantiate custom entities—all feeding back into the engine's asset pipeline

## Potential Issues

- **No versioning/ABI stability**: If plugin function signatures change, all plugins must rebuild; no forward/backward compatibility
- **Implicit string-based dispatch**: Commands are routed by name string; no compile-time safety or type checking on plugin-editor contract
- **No error propagation**: Load failures (missing DLL, missing symbol) likely crash or leave editor in inconsistent state; no try/catch or error codes
- **Hard-coded MFC dependency**: Q3Radiant is Windows-only; plugins inherit this limitation
