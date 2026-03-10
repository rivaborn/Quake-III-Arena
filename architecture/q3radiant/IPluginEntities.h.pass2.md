# q3radiant/IPluginEntities.h — Enhanced Analysis

## Architectural Role

This header defines COM-style plugin interfaces for **Q3Radiant's entity extension system**—allowing third-party DLLs to register custom entity classes and provide rendering callbacks. It sits at the editor↔plugin boundary, enabling the editor core to discover and instantiate plugin-defined entity types without compile-time coupling. This is part of Radiant's broader plugin architecture (see `PlugInManager.cpp`, `IEpairs.h`, `ISurfacePlugin.h`) but specialized for entity lifecycle and visualization.

## Key Cross-References

### Incoming (who depends on this file)

- **Q3Radiant plugin DLLs** (external): implement `IPluginEntity` to define custom entity classes; export factory functions conforming to `PFN_CREATEENTITY`
- **PlugInManager** (`q3radiant/PlugInManager.cpp`): queries plugins for `QERPlugEntitiesFactory_GUID` and `QERPluginEntitiesTable_GUID`; calls `m_pfnCreateEntity` to instantiate plugin entities
- **Entity editor UI**: calls `GetBounds()` for selection/movement gizmos and `CamRender()` for viewport visualization

### Outgoing (what this file depends on)

- **Windows platform types only**: `GUID`, `HMODULE`, `WINAPI` calling convention, basic C++ (virtual methods)
- **eclass_t** (from editor core): entity class metadata (name, color, model, epair schema)
- **IEpair** (from `IEpairs.h`): entity property key-value pairs, passed to factory on instantiation

## Design Patterns & Rationale

**COM-style interface discovery via GUID:** Each interface defines a unique `static const GUID` so plugins can query the manager for specific capabilities by ID. This decouples plugin registration from vtable layout changes.

**Vtable-based polymorphism:** `_QERPluginEntitiesTable` and `_QERPlugEntitiesFactory` use function pointers rather than C++ vtables, making them callable from C-based plugins and serializable across DLL boundaries without name mangling.

**Reference counting (IncRef/DecRef):** Allows the editor to track lifetime and safely unload plugins; plugins remain responsible for heap allocation and cleanup (manual memory management, era-typical).

**Factory pattern:** `PFN_CREATEENTITY` gives plugins control over instantiation logic (initialization, resource allocation) rather than editor-imposed construction.

**Why this structure:** Level editors of this era (Q3A release, 2000) needed plugin extensibility for mod-specific entity types (custom triggers, model entities, dynamic geometry). This interface lets plugins hook entity bounds queries (for selection) and camera rendering (for viewport display) without exposing internal editor data structures.

## Data Flow Through This File

1. **Initialization (editor startup):**
   - Plugin DLL is loaded via `Sys_LoadDll`
   - Editor calls plugin's exported factory function (`PFN_ECLASSSCANDIR`) to register entity classes
   - Factory populates editor's entity class registry with plugin eclass entries

2. **Entity instantiation (map loading):**
   - Editor parses BSP/map entity data, encounters plugin entity class
   - Editor calls `PFN_CREATEENTITY(eclass_t*, IEpair*)` → returns `IPluginEntity*`
   - Plugin allocates and initializes custom entity object

3. **Viewport interaction:**
   - Editor calls `GetBounds()` for intersection tests (selection, movement gizmos)
   - Editor calls `CamRender()` each frame for viewport visualization
   - Plugin renders custom geometry or placeholder via `trap_R_*` syscalls (if game-side) or direct GL calls (if Radiant-side)

4. **Cleanup:**
   - Editor calls `DecRef()` on entity; plugin deallocates when refcount reaches 0

## Learning Notes

**Idiomatic to mid-2000s Windows game tools:** Use of GUIDs for interface discovery, `HMODULE` for runtime DLL management, and manual reference counting were standard before COM+ and .NET. This pattern appears in other Radiant plugins (`ISurfacePlugin`, `IShaders`).

**Contrast with modern engines:** Today's editors (Unreal, Unity) use either:
  - In-process C++ plugin APIs with direct vtable access (no GUIDs)
  - Standalone editor processes communicating via RPC/JSON (no DLL coupling)
  - Scripting layer (Python, C#) for extensibility

**Relationship to game engine:** `IPluginEntity` is *editor-only* and never called by the runtime engine. The entity distinction (plugin vs. built-in) exists only in Radiant; at game time, plugin-defined entity classes are indistinguishable from game-defined ones (both stored as BSP entity strings, parsed by `G_SpawnEntitiesFromString`).

**Reference counting pattern:** Manual refcounting (IncRef/DecRef) without operator overloading suggests this was designed to be language-agnostic and callable from C as well as C++. Modern C++ would use `shared_ptr<IPluginEntity>` to automate this.

## Potential Issues

- **No virtual destructor in IPluginEntity:** If derived classes allocate resources in constructors, the destructor may not be called when deleting via base pointer, causing leaks. Should add `virtual ~IPluginEntity() {}`.
- **Platform-specific:** GUIDs, `HMODULE`, and `WINAPI` tie this to Windows. Cross-platform plugin systems would use opaque handles or abstract factory interfaces.
- **Manual memory management:** No smart pointers; plugins must correctly implement IncRef/DecRef. A single refcount leak breaks the system.
- **No thread safety:** Assumes all calls from single editor thread; concurrent DLL access could race on refcount.
