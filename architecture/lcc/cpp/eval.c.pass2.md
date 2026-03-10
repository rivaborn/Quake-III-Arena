# lcc/cpp/eval.c — Enhanced Analysis

## Architectural Role

This file implements the expression evaluator for the LCC C preprocessor, enabling conditional compilation directives (`#if`, `#elif`, `#ifdef`, `#ifndef`) to be computed at compile-time. As part of the offline **lcc compiler toolchain** (not the runtime engine), it processes Quake III's QVM bytecode source files during compilation. The evaluator is deterministic and independent of runtime state, making it a critical gating mechanism for build-time feature selection.

## Key Cross-References

### Incoming (who depends on this file)
- **`lcc/cpp/cpp.c`** — The main preprocessor driver calls `eval()` when encountering `#if` and `#elif` directives to decide whether to include/exclude subsequent lines
- **Preprocessor token stream** — Input flows from `lex.c` (tokenizer) → `eval()`, which consumes tokens from a `Tokenrow` struct

### Outgoing (what this file depends on)
- **`lcc/cpp/cpp.h`** — Type definitions (`Token`, `Tokenrow`, `Nlist`, `enum toktype`)
- **`expandrow()`** — Called to expand macros in the conditional expression before evaluation (`kwdefined` activation/deactivation signals this)
- **`lookup()`** — Symbol table query to check if a name is defined (for `#ifdef` and `DEFINED` operator)
- **`error()` macro** — Reports syntax and semantic errors during evaluation

## Design Patterns & Rationale

**Operator precedence climbing** — The `priority[]` table encodes C's operator precedence, arity, and type conversion rules. `evalop()` implements a bottom-up shunting yard–style precedence-climbing parser that respects parentheses and operator associativity. This avoids the overhead of a full recursive descent parser while handling complex nested expressions like `(1 << 3) && defined(FOO)`.

**Stack-based evaluation** — Two parallel stacks (`vals[]` for operands, `ops[]` for operators) track computation state. This allows the evaluator to defer operator application until precedence rules dictate, mimicking a calculator's behavior.

**Type tracking during evaluation** — Each value carries a `type` field (SGN / UNS / UND) to track signedness and undefinedness. This enables correct handling of unsigned-safe comparisons (marked with `UNSMARK` bit) and short-circuiting logic operators when values are undefined.

**Lazy macro expansion** — The global `kwdefined` token is toggled before/after `expandrow()` to activate special handling of the `defined()` operator, allowing it to work with undefined names without triggering errors.

## Data Flow Through This File

1. **Input**: `Tokenrow *trp` pointing to tokens after `#if`/`#elif` keyword
2. **Preprocessing phase**: 
   - For `#ifdef`/`#ifndef`: direct symbol lookup, return boolean
   - For `#if`/`#elif`: expand macros in the expression (with `defined()` special-cased)
3. **Tokenization & validation**:
   - Iterate through tokens, classify as nilary (NAME, NUMBER, CCON), unary (TILDE, NOT, DEFINED), or binary (EQ, PLUS, etc.)
   - Enforce proper operand/operator ordering (`rand` flag alternates)
4. **Operator evaluation**:
   - `evalop()` processes operators in precedence order, popping operands and applying operations
   - Type conversions (SGN → UNS → UND) propagate through the computation
   - Division/modulo by zero marks result as UND (undefined)
5. **Output**: Single `long` value (0 or 1 for conditionals) or raw expression value

## Learning Notes

**Idiomatic to 1990s C preprocessor**: This code uses a **flat operator table** indexed by token type rather than a modern AST-building approach. This reflects the simplicity requirements of preprocessor evaluation: expressions are always constant-foldable and must remain deterministic and portable across compilers.

**Contrast with modern practice**: A contemporary preprocessor might use recursive descent parsing or an expression grammar; this one uses a homegrown stack machine, which is both more compact and transparent about operator precedence.

**Undefined value semantics**: The `UND` type is crucial—it propagates undefined-ness through short-circuit operators (`&&`, `||`) correctly, so `#if defined(X) && X` doesn't fail even if `X` is undefined.

**Connection to game engine**: While LCC is offline-only, conditional expressions it evaluates shape the QVM bytecode delivered to the runtime. Bot AI, UI, and game logic can use `#ifdef` guards to compile feature variants.

## Potential Issues

- **No overflow detection**: Integer arithmetic silently wraps; no semantic warning for `#if (1 << 40)` even on 32-bit targets
- **Character constant assumptions**: `tokval()` assumes ASCII escape sequences (`\n`, `\t`, etc.); non-ASCII locales could behave unexpectedly
- **Stack bounds not checked**: A very deeply nested expression (>32 operators or operands) will overflow `vals[]` and `ops[]` arrays without bounds checking
