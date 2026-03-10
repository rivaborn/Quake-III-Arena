# lcc/include/alpha/osf/signal.h — Enhanced Analysis

## Architectural Role

This is a platform-specific signal abstraction header for the LCC C compiler's Alpha OSF (DEC Alpha / OSF/1) target. While LCC itself is a standalone offline tool (not part of the runtime engine), it serves the critical build-time role of compiling `cgame`, `game`, and `ui` module source code into QVM bytecode. This header enables LCC to provide POSIX signal semantics to user code compiled for the Alpha OSF platform—a legacy architecture that the Q3 engine supported in the early 2000s.

## Key Cross-References

### Incoming
- Included by LCC compiler passes and C standard library headers when targeting Alpha OSF
- Consumed by code compiled **through** LCC, not by the runtime engine directly
- Part of the platform-specific include tree (`lcc/include/alpha/osf/`) used to simulate target-platform libc

### Outgoing
- Defines the interface contract for signal handling on Alpha OSF
- No outgoing dependencies within this file; it is purely declarative
- At runtime, the game engine's native platform layers (`code/win32/`, `code/unix/`, `code/macosx/`) provide their own signal handling—not via QVM

## Design Patterns & Rationale

**Minimal platform abstraction**: This header follows LCC's strategy of providing lightweight per-platform header variants in `lcc/include/{arch}/{os}/` rather than a unified conditional-compilation header. This isolates platform-specific quirks and keeps the compiler's include search path simple.

**Signal handler typedef**: The function-pointer cast `void (*)(int)` is explicit and verbose—typical of pre-C99 practice. It allows signal handlers to accept a signal number and return void.

**Macro-based constants**: Signal numbers (2, 4, 6, 8, 11, 15) and special handler values (0, -1, 1) are hardcoded per Alpha OSF's ABI, not derived from includes. This ensures the compiler produces correct code regardless of host platform.

## Data Flow Through This File

1. **Compile-time**: When LCC compiles user code for Alpha OSF target, `#include <signal.h>` resolves to this file.
2. **Code generation**: User code calling `signal()` or `raise()` generates Alpha OSF-compatible QVM bytecode referencing these symbols.
3. **Runtime**: The QVM—executed in `qcommon/vm.c`—cannot directly invoke OS signals; any `trap_*` calls for system functionality route through the game or cgame module's syscall layer, not through this header.

## Learning Notes

**Era-specific artifact**: This header reflects Q3A's multi-platform compiler toolchain from the mid-2000s (Alpha, MIPS, Sparc, x86). Modern game engines target fewer architectures and often use LLVM or platform-provided SDKs instead of self-hosted compiler trees.

**Layered abstraction boundary**: The clear separation between LCC's platform-specific headers and the runtime engine's platform layers (`unix/`, `win32/`, `macosx/`) shows a clean division: *LCC provides target-platform simulation for compilation; the engine provides host-platform integration for execution.* This is lost in modern single-platform workflows.

**Sandbox isolation**: QVM bytecode compiled from code using `signal.h` cannot escape the VM to invoke actual OS signals—the engine's `dataMask`-based sandbox enforces this. The declarations are semantically present but functionally inert at runtime.

## Potential Issues

**None detected**. This is a stable, minimal platform-specific header. It correctly maps the Alpha OSF ABI. However, its presence as a historical artifact (with no modern use) indicates the codebase retains significant legacy cross-compilation infrastructure unlikely to be exercised.
