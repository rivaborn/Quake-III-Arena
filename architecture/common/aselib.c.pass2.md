# common/aselib.c — Enhanced Analysis

## Architectural Role

This file is a **format translator** in the Q3A offline build toolchain, bridging Autodesk 3DS Max (via its ASCII Scene Export format) and the engine's downstream compilation tools. It consumes `.ase` text files and yields `polyset_t` / `triangle_t` structures consumed by `q3map` (map compiler), `q3radiant` (editor), and model compilers. Located in `common/` (not `code/`), it has **zero runtime engine dependency** — purely a build-time utility linked into tools.

## Key Cross-References

### Incoming (who depends on this file)
- **`q3map`** (BSP/lightmap compiler) — uses `ASE_Load` to import model geometry for misc_model placement and LOD computation
- **`q3radiant`** (level editor) — uses ASE import for 3D asset preview and scene construction  
- **Model compilers** (referenced in first-pass but not detailed here) — convert ASE frames into MD3/MD4 vertex animations
- **`common/bspfile.c`, `common/cmdlib.c`** — `aselib.h` transitively includes these for `Error`, `Q_filelength`, `gamedir` global, and memory utilities

### Outgoing (what this file depends on)
- **`cmdlib.h` globals**: `gamedir` (string used in path resolution at `ASE_KeyMAP_DIFFUSE`), `Error` (fatal exit handler)
- **`cmdlib` functions**: `Q_filelength` (file size probe), `malloc` / `free` (heap allocation)
- **`polyset.h`** (via `aselib.h`): `polyset_t`, `triangle_t` output structures
- **`mathlib.h`**: `qboolean` type, `strlwr` (case conversion for path normalization)
- Standard C: `<stdio.h>` (file I/O), `<stdlib.h>` (memory), `<string.h>` (strcpy, strstr, strlwr)

## Design Patterns & Rationale

### 1. **Singleton Global Parser State** (`static ase_t ase`)
All parsing context (materials, objects, buffer pointers, frame cursors) lives in one global struct. Why: Tools are single-threaded and load one ASE per invocation. Eliminates parameter passing through deep callback chains. Trade-off: thread-unsafe, prevents streaming.

### 2. **Token-Based Recursive Descent** (`ASE_GetToken` → `ASE_ParseBracedBlock` → callbacks)
Reusable tokenizer feeds a brace-matching dispatcher, which routes tokens to type-specific callbacks (`ASE_KeyMATERIAL_LIST`, `ASE_KeyGEOMOBJECT`, etc.). Why: ASE format is human-readable, context-free; callbacks decouple grammar from action. Similar to Q3A's shader/entity parser architecture.

### 3. **Lazy Path Resolution** (`ASE_KeyMAP_DIFFUSE` path normalization)
Rather than resolve material paths at load time, this function defers resolution until a material is encountered. It normalizes Windows drive-letter paths (e.g., `C:/path`) against Unix UNC paths (`//server/path`), then searches for a suffix match in `gamedir`. Why: ASE files are portable but may originate from Windows 3D tools; must work on heterogeneous build farms (Windows + Linux + Mac). Edge case handling suggests real-world pain points (mixed path separators, drive letters on Unix).

### 4. **Frame Filtering by Name** (`ASE_Process` skips "Bip01", "Bip", "ignore_*")
Selectively discards skeleton/helper bones and user-marked objects. Why: 3DS Max rigs always export character skeletons (Biped armatures) alongside geometry; these aren't renderable. Naming convention filtering is simpler than semantic tagging in the ASE format.

### 5. **Fatal Error Model** (all `Error()` calls are `[NORETURN]`)
No graceful degradation; parse errors call `Error()` (which longjmps in qcommon). Why: ASE files are generated offline by trusted tools (Max + export plugin). Failures indicate tool bugs or corrupted files, not user data issues. Keeps code simple (no error state threading).

## Data Flow Through This File

```
ASE_Load (entry)
  ↓ [reads file → malloc buffer]
  ASE_Process (top-level dispatcher)
    ↓ [main loop: ASE_GetToken]
    ├─ "*MATERIAL_LIST" → ASE_ParseBracedBlock(ASE_KeyMATERIAL_LIST)
    │   └─ "*MATERIAL" / "*MAP_DIFFUSE" / "*BITMAP" → path normalization → ase.materials[]
    ├─ "*GEOMOBJECT" → ASE_ParseBracedBlock(ASE_KeyGEOMOBJECT)
    │   └─ "*NODE_NAME" → filter "Bip", "ignore_*"
    │   └─ "*MESH" / "*MESH_ANIMATION" → ASE_ParseBracedBlock(ASE_KeyMESH)
    │       └─ "*MESH_VERTEX_LIST" → fill ase.objects[n].anim.frames[m].vertexes[]
    │       └─ "*MESH_TVERTEXMAP" / "*MESH_TVERTEXLIST" → fill tvertexes[], tfaces[]
    │       └─ "*MESH_FACE_LIST" → fill faces[]
    └─ [CollapseObjects] → compact multi-frame sequences
  ↓
ASE_GetSurfaceAnimation (consumer)
  ↓ [iterate frames, copy vertex/UV data]
  ↓ [allocate polyset_t + triangle_t arrays]
  ↓ [output to caller (q3map, radiant, etc.)]
ASE_Free (cleanup)
  ↓ [deallocate per-frame vertex/face arrays]
```

**Key transformations:**
- **Vertex positions**: raw XYZ → `polyset_t.triangles[].verts[]` (3D point per vertex per triangle)
- **Texture coordinates**: separate UV channel → `texcoords[]` (U,V per vertex per triangle)
- **Normals**: parsed but **discarded** (not copied to `polyset_t`; renderer recomputes)
- **Materials**: filename strings → resolved relative to `gamedir`, normalized path separators
- **Animations**: per-frame mesh arrays → filtered frame range (skipFrameStart/End, maxFrames) → condensed polyset_t sequence

## Learning Notes

### What developers studying Q3A's toolchain would learn:

1. **Format diversity in game pipelines**: Q3A toolchain accepts Autodesk Max (`.ase`) as a primary modeling input, not just proprietary formats. Shows pragmatic reliance on industry-standard DCC tools.

2. **Path complexity in cross-platform builds**: The sophisticated Windows/Unix path normalization in `ASE_KeyMAP_DIFFUSE` reveals a real-world 2000s problem: build farms mixing Windows artists' tools with Linux servers. Modern toolchains (Python, CMake) hide this; here it's explicit C string manipulation.

3. **Skeleton filtering via naming**: The "Bip01" filtering is idiomatic Max workflow (Biped is the built-in character rig). Similar filtering appears in game tools that consume Max files (UE4, Source, etc.).

4. **No-copy parsing philosophy**: ASE parser doesn't validate or normalize early; it trusts downstream tools. Modern parsers might schema-validate and error early. Here, structural errors surface only when consumers try to use the data.

5. **Comparison to modern engines**:
   - **Modern**: Asset pipelines are type-safe (protobuf, UASSET binary blobs), validated at import-time
   - **Q3A** (2005): Text-based, trusts upstream tools, defers validation
   - **Tradeoff**: simpler parser, harder to debug upstream tool failures

### Idiomatic patterns of this era:
- Flat global state (`static ase_t ase`) instead of context objects or handles
- Callback-based recursive descent (predates modern tree-walk visitors)
- Fatal errors via longjmp instead of exception handling or error codes
- Lazy allocation (allocate frames only if referenced) and reusable token buffer (1024-byte scratch)

## Potential Issues

1. **Buffer overrun on very long tokens**: `s_token[1024]` is fixed-size; tokens longer than 1023 chars will overflow silently (though unlikely in practice; ASE files don't have long atoms).

2. **Path resolution ambiguity**: `strstr(buff2, buff1 + 2)` assumes the normalized material path is a substring of `gamedir`. Could produce false positives if multiple `gamedir` paths contain the same suffix. Modern code would use path canonicalization + fuzzy matching.

3. **No bounds checks on object/material counts**: `MAX_ASE_OBJECTS` (64) and `MAX_ASE_MATERIALS` (32) are silently enforced by `Error()` calls, but exceed limits without prewarning. Tools loading complex Max scenes might fail cryptically.

4. **Memory not freed on Error()**: If `ASE_Process` hits `Error()` mid-parse, `ase.buffer` is never freed (longjmp bypasses cleanup). Tolerable for short-lived tools; not production-grade.

5. **Frame count assumption in `ASE_GetSurfaceAnimation`**: Assumes frames are stored sequentially without gaps. Malformed ASE files with missing/reordered frames could produce undefined behavior.
