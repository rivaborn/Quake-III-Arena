# q3radiant/iscriplib.h — Enhanced Analysis

## Architectural Role

This file defines the plugin vtable interface for Q3Radiant's script parsing library. It is the **contract** by which the level editor exports its tokenizer to plugins—enabling tools like the Surface Properties plugin or shader editors to parse the same `.shader`, `.map`, and script formats as the editor itself uses. This exemplifies Q3Radiant's plugin architecture: late binding of capability vtables discovered by GUID, allowing plugins to reuse the editor's parsing without static linking.

## Key Cross-References

### Incoming (who depends on this file)
- Q3Radiant **plugins** (Surface Properties plugin, shader editors, etc.) that need to parse script files
- Plugin loader code in Q3Radiant that discovers/instantiates interfaces by GUID
- Any editor tool that needs to tokenize Q3 script syntax

### Outgoing (what this file depends on)
- Implicit dependency on **parse.h** (mentioned in comment) — the actual parser implementation in the editor
- No direct engine dependencies; isolated to the editor tool layer
- The actual implementation (`botlib/l_script.c` or editor equivalent) provides the function bodies

## Design Patterns & Rationale

**COM-style Plugin Interface**: The GUID-based interface discovery pattern mirrors Windows COM, enabling:
- Runtime interface negotiation without header-file coupling
- Version-safe vtable layout (size field for struct growth)
- Plugin compilation against a stable ABI, not the editor's source

**Three-Token Operations**: The interface is minimal—just three function pointers—mirroring a **push-back lexer** common in 1990s compilers:
- `GetToken(crossline)`: Fetch next token; `crossline=true` skips newlines (continue on next line)
- `UnGetToken()`: Single-token lookahead/pushback
- `Token()`: Retrieve the text of the current token without advancing

This matches the botlib's own `l_script.c` lexer design, suggesting code reuse or parallel implementation.

## Data Flow Through This File

1. **Plugin Load Time**: Editor's plugin loader queries for `QERScripLibTable_GUID`
2. **Runtime**: Plugin receives a populated `_QERScripLibTable` vtable with function pointers
3. **Parsing Loop**: Plugin repeatedly calls `GetToken()` → `Token()` → process → `UnGetToken()` (if needed)
4. **Data Outputs**: Tokens flow into the plugin's own semantic analysis (shader compilation, entity parsing, etc.)

## Learning Notes

**Why This Matters**:
- Demonstrates the **late-bound plugin architecture** common in visual tools (Maya, Radiant, Unreal Editor)
- The minimal interface is intentional: only what plugins truly need for lexing, leaving semantic interpretation to the plugin
- Contrast with the engine itself (botlib/l_script.c) which pairs lexing with semantic parsing; the editor decouples them for plugin flexibility

**Idiomatic to This Era**:
- GUID-based interface lookup predates .NET/COM native interop; was industry standard for extensible Windows tools
- Calling convention `WINAPI` (stdcall) is Windows-specific; reflects Q3Radiant's original Win32-first development

**Modern Equivalents**:
- Rust plugin crates with stable ABI via `#[repr(C)]` vtables
- Node.js native modules (N-API)
- Language-agnostic gRPC/protocol-buffer interfaces

## Potential Issues

None directly inferable from this header. However:
- **Single-token pushback limitation**: `UnGetToken()` stores only one token. Multi-token lookahead is unsupported (not usually needed for tokenizers, but constrains parser design).
- **No error reporting in interface**: Callers must infer parse failure from token contents (token == NULL?). Real-world parsers often want structured error callbacks.
- **Coupling to WINAPI**: The `WINAPI` calling convention makes this Windows-only; cross-platform plugins would need conditional compilation.

---

**File Role Summary**: Minimal, stable **lexer vtable** enabling Q3Radiant plugins to reuse the editor's tokenizer without static or dynamic linking fragility.
