# lcc/src/event.c — Enhanced Analysis

## Architectural Role

This file implements a simple **observer pattern** event dispatch system for the LCC C compiler toolchain. It serves as an *extensibility hook* within the compiler pipeline—allowing compile-phase subscribers (callbacks registered from other modules) to respond to key compiler events without hard-coupling those modules. Since LCC is itself repurposed as the QVM bytecode compiler for Quake III Arena's server-side game logic, UI VMs, and client-side cgame VM, this event system enables modular compiler extensions during the VM code generation phase.

## Key Cross-References

### Incoming (who depends on this file)
- **lcc/src/main.c**, **lcc/src/*.c** (other compiler phases): Register `Apply` callbacks via `attach()` during compiler initialization to hook key events (likely: parse completion, code generation, optimization phases)
- **lcc/src/output.c**, **lcc/src/pass2.c** (code emission): Call `apply()` to broadcast when compilation milestones occur, triggering registered handlers

### Outgoing (what this file depends on)
- **lcc/src/list.c** (inferred): The `List` type and `append()` function (circular linked list primitive)
- Memory allocator `NEW()` macro (likely `lcc/src/alloc.c` via `#include "c.h"`): Allocates `struct entry` nodes with `PERM` lifetime to persist for the entire compilation unit

## Design Patterns & Rationale

**Observer Pattern**: Decouples *event publishers* (compiler phases: parser, codegen) from *event subscribers* (optional extensions, analysis tools, code-generation passes). This design is typical of mid-1990s compiler architecture before widespread use of visitor patterns or AST walkers.

**Circular Linked List**: The `while (lp != event)` loop in `apply()` signals that multiple handlers are chained in a ring. This avoids needing a sentinel node and allows handlers to be dynamically registered/removed during iteration (though no removal function is visible here).

**PERM Allocation**: Events are allocated with permanent lifetime (`NEW(p, PERM)`), meaning they survive until compiler shutdown. This is appropriate since event handlers are registered at startup and remain active throughout compilation.

## Data Flow Through This File

1. **Attachment Phase** (compiler startup): Other modules call `attach(func, client_data, list_ptr)` to register a callback and opaque context pointer. The entry is appended to the circular list.
2. **Trigger Phase** (during compilation): When a compiler phase completes a milestone, it calls `apply(event_list, arg1, arg2)` passing the registered event list and two dispatch arguments.
3. **Dispatch**: Each handler in the ring is invoked sequentially: `(*func)(client_context, arg1, arg2)`. The `void *cl` closure allows each handler to maintain its own state.

## Learning Notes

- **Era-appropriate abstraction**: This is a classic 1990s compiler technique—simpler and more transparent than modern event buses or dependency injection frameworks, but effective for a modular C codebase.
- **No handler chaining semantics**: Handlers run to completion; there's no short-circuit, error propagation, or return-value handling. All handlers are fire-and-forget notifications.
- **Implicit initialization**: The global `Events` variable is declared but never initialized in this file; callers must ensure it's `NULL` or initialized elsewhere (likely in `lcc/src/main.c`).
- **Connection to QVM pipeline**: Although LCC is a general-purpose C compiler, when used to build Quake III's QVMs (`code/game/`, `code/cgame/`), this event system allows the compiler to trigger game-specific code-generation passes (e.g., syscall trampolines injected via `code/game/g_syscalls.c` → lcc integration).

## Potential Issues

- **No guard against NULL list**: `apply()` checks `if (event)`, but if a list pointer is dangling or corrupted, the ring traversal could infinite-loop or crash at `lp->link`.
- **No deregistration**: There is no mechanism to remove a registered event handler; handlers must exist for the lifetime of compilation or be managed externally.
