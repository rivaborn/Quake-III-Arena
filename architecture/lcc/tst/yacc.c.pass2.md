# lcc/tst/yacc.c — Enhanced Analysis

## Architectural Role

This test file validates the LCC compiler's **yacc-based expression parser** infrastructure. While located in the offline LCC compiler toolchain (not the runtime engine), it's critical to the **QVM bytecode compilation pipeline**: the lexer/parser mechanisms demonstrated here enable LCC to parse C source code into an AST for eventual compilation to QVM bytecode executed by the game and cgame VMs. The file exemplifies how yacc-generated parsers integrate with the LCC pipeline to support the three runtime VM subsystems (`code/game`, `code/cgame`, `code/ui`).

## Key Cross-References

### Incoming (who depends on this file)
- **LCC compiler build system** (`lcc/lburg/`, `lcc/src/`, `lcc/etc/`) — this test validates the yacc infrastructure that LCC's own parser modules depend on
- **Q3 VM bytecode compilation** — indirectly: LCC uses yacc-generated parsers to tokenize and parse all C code destined for `code/game`, `code/cgame`, `code/ui` VMs
- Not directly called from runtime engine; it's an **offline/build-time test**

### Outgoing (what this file depends on)
- **C standard library** (`stdio.h` only) — minimal dependencies
- **yacc runtime infrastructure** — the lexer/parser tables are self-contained; no external grammar or symbol tables
- **Not a runtime subsystem** — does not call into `qcommon`, renderer, or any other engine module

## Design Patterns & Rationale

**Yacc-generated dual-layer parsing:**
- **Lexer** (`yylex()`, `yylook()`) — finite automaton recognizing tokens via state transitions (`yycrank[]` transition table, `yysvec[]` state vector)
- **Parser** (`yyparse()`, shift/reduce LR table via `yyact[]`/`yypact[]`) — deterministic bottom-up parsing with grammar rules in `yyr1[]/yyr2[]`
- **Separation of concerns:** The lexer handles character-stream tokenization; the parser handles syntax structure (operator precedence, parentheses, unary minus via `UNARYMINUS` token distinct from binary minus)

**Why this structure?** LCC must parse full C syntax, not just this toy grammar. This test demonstrates the core pattern that LCC replicates for its real parser (`lcc/src/lex.c`, `lcc/src/expr.c`, etc.), allowing expressions with operator precedence, function calls, and complex type declarations to be correctly parsed into a form suitable for compilation to QVM bytecode.

**Tradeoff:** The yacc-generated tables (hundreds of lines of static arrays) are verbose but enable fast, predictable parsing. This is essential for an offline compiler.

## Data Flow Through This File

1. **Input:** ASCII character stream (via `input()` macro, which calls `getc(yyin)` from stdin)
2. **Lexer phase:** Character stream → token stream (via `yylex()` state machine)
   - Recognition rules: decimal digits → `CON` (constant), alphabetic/`_` → `ID` (identifier), operators and parentheses pass through
3. **Parser phase:** Token stream → parse tree reduction (via `yyparse()` shift-reduce automaton)
   - Reduces tokens according to grammar (`yyr1[]` LHS, `yyr2[]` RHS lengths)
   - Example: `ID '+' ID` reduces to expression, then `expression '+' expression` reduces to a new expression
4. **Output:** Parse success/failure; no semantic action code (stub `case` bodies just `break`)
5. **Error handling:** `yyerror()` prints to stdout; no recovery strategy in this test

## Learning Notes

**Idiomatic to LCC-era C compiler design (vs. modern engines):**
- Uses **yacc/lex** (1970s–90s standard tools) rather than hand-rolled recursive descent or parser-combinator libraries
- No AST construction in this test (just syntax validation); real LCC builds trees for optimization passes
- **Global state** (`yychar`, `yyval`, `yystate`, `yytext[]`) rather than parser-context objects
- **No error recovery** — parse fails completely on first syntax error; modern compilers collect multiple errors per pass

**Connection to game engine:**
- The expression grammar here (arithmetic, precedence, unary/binary operators) is a **subset** of C
- Real LCC parser handles declarations, function definitions, control flow — complexity needed for `code/game/`, `code/cgame/` source
- This test is essentially a "hello world" for the yacc infrastructure; production game code C→QVM compilation reuses these same lexer/parser mechanisms at a much larger scale

**Key architectural insight:** The **LCC toolchain is a compile-time artifact** — it's not linked into any runtime subsystem. The `.qvm` files it produces are loaded and executed by `code/qcommon/vm.c` and the platform-specific VM interpreters (`vm_interpreted.c`, `vm_x86.c`, `vm_ppc.c`), but LCC itself is never invoked at runtime. This test file validates that the parsing layer works before attempting to compile real game code.

## Potential Issues

None inferable from the code alone; this is a test harness with minimal error handling. The stub action code (empty `case` bodies) would fail silently on real grammar in a production compiler, but that's intentional for a parser test.
