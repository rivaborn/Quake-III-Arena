# q3radiant/qerplugin.h — Enhanced Analysis

## Architectural Role
This header defines the plugin ABI contract for QERadiant, the level editor—completely separate from the runtime engine. Whereas the engine (`code/`) uses swappable DLL modules (e.g., renderer) only at top-level subsystems, the editor permits *mid-level* plugin extensibility: third-party texture/model loaders, entity class registerers, surface property customizers, and map handlers can be dynamically loaded to augment editor capabilities without recompilation. The plugin system is Windows-centric and uses COM-style vtables and opaque handle patterns prevalent in 2000s game tool design.

## Key Cross-References
### Incoming (who depends on this file)
- **q3radiant itself**: Core editor loads plugins via `LoadLibrary`/`GetProcAddress` at startup; dispatch to `QERPlug_Init`, `QERPlug_GetCommandList`, `QERPlug_Dispatch` based on loaded module type
- **Plugin DLLs**: Texture format handlers (e.g., TGA, PNG loaders), model importers (ASE, MD3), BSP map variant handlers (Q2, Half-Life), custom surface property dialogs
- **No runtime engine references**: Unlike the renderer DLL or server/client modules, `qerplugin.h` is **never** linked into the game engine itself; editor and game ship independently

### Outgoing (what this file depends on)
- **qertypes.h**: Provides foundational types (`vec3_t`, `qboolean`, `brushprimit_texdef_t`, `epair_t`, `patchMesh_t`) shared with editor core
- **Windows.h**: HWND, HMODULE, WINAPI calling convention (Windows-only, no cross-platform plugin support)
- **Implicit**: The function pointers assume editor binary exports the `_QERFuncTable_1` vtable; plugins receive it via `PFN_QERPLUG_GETFUNCTABLE`

## Design Patterns & Rationale
**ABI versioning**: Six parallel function sets (QERPLUG_* names) + four parallel function pointer table versions (`_QERFuncTable_1` etc.) allow incremental feature addition (v1.0 → v1.7) without breaking old plugins. Plugins declare `m_fVersion` in their returned struct; editor checks version before calling newer methods. This is more forgiving than function-at-a-time versioning but still requires editor to maintain all historical function signatures.

**Opaque handle pattern**: Brushes/entities/patches passed as `LPVOID`; actual structure definitions hidden inside editor. Prevents accidental plugin memory corruption and isolates editor internals. Cost: plugins cannot inspect handles directly, must call editor functions to read/write.

**Vtable dispatch**: Both plugins (expose `QERPlug_*` via `GetProcAddress`) and editor (expose `QERApp_*` via table) use function-pointer tables. Mirrors renderer DLL model; avoids static linking and permits hot reload.

**Stateless commands**: `QERPlug_Dispatch(LPCSTR p, vec3_t vMin, vMax, BOOL bSingleBrush)` is generic command router; plugins decode string commands and act. Simple but fragile (no enum safety).

## Data Flow Through This File
1. **Initialization**: Editor calls `QERPlug_Init(hApp, hwndMain)`, plugin caches vtable pointer (`_QERFuncTable_1`) and returns name/version string.
2. **Capability advertisement**: `QERPlug_GetName()` and `QERPlug_GetCommandList()` tell editor what this plugin can do.
3. **Texture/Model pipeline**: Plugin calls `PFN_QERAPP_LOADTEXTURERGBA` callback; editor returns RGBA data and surface/contents flags. Plugin may register texture format via `QERPLUG_GETTEXTUREINFO`.
4. **Brush manipulation**: Plugin calls brush creation/face-add functions, populates `_QERFaceData` (texture name, UV scroll/rotate/scale, brush primitives), commits to map via `PFN_QERAPP_COMMITBRUSHHANDLETOMAP`.
5. **Entity/epair handling** (v1.7): Plugin can enumerate/create entities and modify key-value pairs via `epair_t` linked lists.
6. **Result signals**: Editor calls `QERPlug_Dispatch("PointDone"/"BrushDone")` to signal completion of user operations.

## Learning Notes
**Era-specific idioms**:
- Pre-STL C: Fixed-size names (`QER_MAX_NAMELEN`), manual linked lists (`epair_t`), memset-based initialization in constructors
- COM-style interfaces: Vtables, `REFGUID` interface requests (`QERPlug_RequestInterface`), `LPVOID` casts
- No C++ exceptions or RTTI; all errors implicit (missing function pointers = silent failure)
- Windows-only; `WINAPI` and `HWND` prevent porting without stubbing entire plugin layer

**Contrast to modern engines**: 
- Modern UE/Unity use scripting languages (C#, Blueprints, Lua) for editor extensions, not binary plugin ABIs
- Modern editors use JSON/protobuf for serialization, not C struct layouts vulnerable to alignment/endianness issues
- No concept of plugin sandboxing or versioning negotiation (plugins directly call editor via vtable; no capability query API)

**Brush primitives** (`brushprimit_texdef_t`) hint at Q3 shader-based material system; not a separate namespace—texdef embedded in face data. This differs from pure BSP editors (Q1/Q2) that use flat surface properties.

## Potential Issues
1. **ABI fragility**: Multiple function table versions (v1.0, 1.5, 1.6, 1.7) and `LPVOID` opaque handles create tight coupling; any change to internal editor structure (brush layout, entity representation) requires ABI bump and plugin recompilation.
2. **No error signaling**: Most callbacks return `void`; no status/error codes. A failed brush commit or entity lookup is silent.
3. **Version sprawl**: Commented-out `QERPLUG_GETSERVICETPE` and unimplemented hook registrations (`QERAPP_REGISTER_MAPLOADFUNC`) suggest abandoned designs; unused cruft in final API.
4. **Thread safety unspecified**: `QERPlug_Dispatch` and editor function calls lack synchronization hints; plugins calling from multiple threads could corrupt editor state.
5. **No 64-bit plan**: `LPVOID` and `int` handles are pointer-width-dependent; file format may not survive architecture changes (though source release in 2005 meant no forward maintenance was expected).
