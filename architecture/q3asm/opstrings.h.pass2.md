# q3asm/opstrings.h — Enhanced Analysis

## Architectural Role

This file defines the **canonical opcode enumeration** for the QVM bytecode format, serving as the bridge between the LCC C compiler's intermediate representation (IR) and the QVM instruction set. It functions as a **lookup table for the q3asm assembler**, translating human-readable operation names (produced by the `lcc` compiler's output stage) into `OP_*` constants that get encoded into final QVM bytecode. This enumeration is simultaneously the **source of truth for the VM interpreter/JIT layers** (`code/qcommon/vm_interpreted.c`, `vm_x86.c`, `vm_ppc.c`), which must dispatch on these exact opcode values at runtime.

## Key Cross-References

### Incoming (who depends on this file)
- **`q3asm/q3asm.c`** (assembler main): parses this opcode table during initialization; uses it to tokenize assembly-like input from the compiler
- **`code/qcommon/vm_*.c`** (VM interpreters/JITs): the runtime `OP_*` constants defined here must match the bytecode produced by q3asm; any mismatch breaks VM execution
- **`lcc/src/gen.c`** (LCC code generator): implicitly produces operation names (e.g., "ADDF4", "CALL") that **must match** the left-hand strings in this table or assembly generation fails
- **`code/game/`, `code/cgame/`, `code/ui/`** (compiled QVM modules): indirectly depend on this enumeration because their bytecode was assembled using these mappings

### Outgoing (what this file depends on)
- **`q3asm/qfiles.h`** (bytecode format constants): defines the `OP_*` enum that this file references
- **`lcc/` compiler suite**: the operation names in the left-hand column are hardcoded in LCC's code generator output

## Design Patterns & Rationale

**1. Two-Phase Compilation Pipeline:**
- LCC (external C compiler) generates intermediate assembly-like text with operation names (`ADDF4`, `CNSTI4`, etc.) that embed both **type** (F=float, I=signed int, U=unsigned, P=pointer) and **bit-width** (4=32-bit, 2=16-bit, 1=8-bit) information
- q3asm reads this text and uses `opstrings.h` to **compress** these verbose, type-qualified names into compact opcodes
- This allows LCC to remain type-aware during IR emission without modifying the external compiler

**2. Type Coercion Mapping:**
- Multiple source operations map to the **same target opcode**. For example, all int/unsigned 32-bit arithmetic (`ADDI4`, `ADDU4`) map to `OP_ADD`; the VM treats them identically at runtime
- Some operations like `CVII*` (int-to-int conversion) or type-widening conversions map to `OP_IGNORE`, indicating they're no-ops in a flat bytecode model without typed stack frames

**3. Commented-Out Entries:**
- Entries like `//{ "CALLB", OP_UNDEF }` suggest **architectural iteration**: these operations were present in the LCC IR but either not yet implemented, superseded, or deemed unnecessary in the QVM design
- This is a sign of incremental toolchain integration (LCC wasn't written for Quake; it's being co-opted)

**4. Address & Parm Operations:**
- `ADDRGP4` → `OP_CONST`: global/procedure address become immediate constants
- Commented-out `ADDRFP`, `ADDRLP`: LCC can emit frame/local addressing, but the QVM likely flattens these into direct stack offsets elsewhere

## Data Flow Through This File

1. **Source:** LCC compiler (`lcc/src/gen.c`) emits assembly text with operation names
2. **Ingestion:** q3asm reads text, tokenizes, and **searches this table** for each opcode name
3. **Transformation:** Operation name → opcode integer (e.g., `"ADDF4"` → `0x03`)
4. **Output:** Bytecode written to `.qvm` file with numeric opcodes
5. **Consumption:** VM interpreter/JIT (`vm_interpreted.c`, `vm_x86.c`) reads bytecode and **dispatches on opcode** to execute operation

## Learning Notes

- **External Compiler Adoption Pattern:** This file exemplifies how Quake III integrated an off-the-shelf C compiler (LCC) without modifying it. The operation namespace (ADDF4, CNSTI4, etc.) is **LCC's native IR vocabulary**, and q3asm acts as a **semantic adapter**.
- **Flat Bytecode Architecture:** Unlike stack VMs with typed frames, the QVM treats all operations uniformly; type information from the compiler is **discarded** after assembly (operations like `CVII2` → `OP_IGNORE`), indicating a single flat memory model.
- **Compiler-VM Contract:** This file is the **executable spec** of the compiler-to-VM ABI. Any drift between LCC's generated names and this table causes silent or catastrophic failures—there's no version negotiation at runtime.

## Potential Issues

- **Brittleness:** If LCC is upgraded and new operation types are introduced (e.g., `ADDI8` for 64-bit), the assembler will fail with an unknown opcode error. There's no forward-compatibility mechanism.
- **Silent Type Loss:** Coercing `CVII1`, `CVII2`, `CVIU4` to `OP_IGNORE` means precision-changing casts are **not validated** at VM execution time. A buggy compiler could emit nonsensical casts that pass through undetected.
