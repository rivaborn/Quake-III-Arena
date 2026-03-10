# code/qcommon/huffman.c

## File Purpose
Implements an Adaptive Huffman compression/decompression codec used for network message encoding in Quake III Arena. Based on the algorithm from Sayood's *Data Compression* textbook, with node ranks implicitly encoded via doubly-linked list position rather than stored explicitly.

## Core Responsibilities
- Maintain and update an adaptive Huffman tree as symbols are transmitted/received
- Encode symbols to bit-stream output using prefix codes derived from tree position
- Decode symbols from bit-stream input by traversing the tree
- Compress/decompress full `msg_t` network message buffers
- Initialize persistent `huffman_t` state for use by the network channel layer

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `node_t` | struct (`nodetype`) | Huffman tree node; participates in both binary tree (left/right/parent) and ranked doubly-linked list (next/prev/head/weight/symbol) |
| `huff_t` | struct | Complete adaptive Huffman coder state: tree root, list head/tail, symbol→node lookup table (`loc[257]`), node pool, pointer freelist |
| `huffman_t` | struct | Paired compressor + decompressor `huff_t` instances for a network channel |
| `msg_t` | struct | Network message buffer (defined in `qcommon.h`); holds `data`, `cursize`, `maxsize` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `bloc` | `static int` | file-static | Current bit-offset cursor shared across internal bit I/O helpers; set from caller-supplied `*offset` on entry to public functions |

## Key Functions / Methods

### Huff_putBit / Huff_getBit
- **Signature:** `void Huff_putBit(int bit, byte *fout, int *offset)` / `int Huff_getBit(byte *fin, int *offset)`
- **Purpose:** Public bit-level I/O with caller-managed offset; used by `msg.c` for per-bit network reads/writes.
- **Inputs:** Single bit value or buffer pointer; `*offset` is bit index.
- **Outputs/Return:** `Huff_getBit` returns the extracted bit (0/1); both update `*offset`.
- **Side effects:** Reads/writes the file-static `bloc`.

### add_bit / get_bit *(static helpers)*
- **Purpose:** Internal variants that use the module-global `bloc` directly, without an explicit offset parameter. Called only within compression/decompression loops.

### swap
- **Signature:** `static void swap(huff_t *huff, node_t *node1, node_t *node2)`
- **Purpose:** Exchange two nodes' positions in the Huffman binary tree, updating parent pointers. Updates `huff->tree` if either node is the root.
- **Side effects:** Mutates `parent->left`/`right` and `node->parent` fields; may update `huff->tree`.

### swaplist
- **Signature:** `static void swaplist(node_t *node1, node_t *node2)`
- **Purpose:** Exchange two nodes in the rank-ordered doubly-linked list, maintaining `next`/`prev` linkage. Handles self-reference edge cases from in-place swap.

### increment
- **Signature:** `static void increment(huff_t *huff, node_t *node)`
- **Purpose:** Recursive post-transmission weight increment; enforces the Huffman sibling property by swapping the node with the highest-ranked node of equal weight before incrementing, then recurses on the parent.
- **Side effects:** Calls `swap`, `swaplist`, `get_ppnode`, `free_ppnode`; mutates tree and list structure throughout.
- **Notes:** Base case is `node == NULL`. Maintains the invariant that `*node->head` points to the highest-ranked node in each weight class.

### Huff_addRef
- **Signature:** `void Huff_addRef(huff_t *huff, byte ch)`
- **Purpose:** Record a new symbol occurrence. If the symbol has never been seen (NYT path), expands the tree by replacing the NYT leaf with an internal node whose children are the new symbol leaf and the new NYT. Then calls `increment` to update weights.
- **Side effects:** Allocates from `huff->nodeList` pool; updates `huff->loc[ch]`, `huff->lhead`, tree structure.

### Huff_Receive / Huff_offsetReceive
- **Signature:** `int Huff_Receive(node_t *node, int *ch, byte *fin)` / `void Huff_offsetReceive(..., int *offset)`
- **Purpose:** Decode one symbol by traversing the tree from a given node, reading bits (right=1, left=0). `offsetReceive` also manages the `bloc`/`*offset` handoff.
- **Outputs/Return:** `Huff_Receive` returns the decoded symbol (0 on error); stores symbol in `*ch`.

### Huff_transmit / Huff_offsetTransmit
- **Signature:** `void Huff_transmit(huff_t *huff, int ch, byte *fout)` / `void Huff_offsetTransmit(..., int *offset)`
- **Purpose:** Encode symbol `ch` by emitting its prefix code. If unseen, emits the NYT code followed by 8 raw bits (literal escape). Recursively traces the path to root via `send`.
- **Side effects:** Writes bits to `fout` via `add_bit`.

### Huff_Decompress
- **Signature:** `void Huff_Decompress(msg_t *mbuf, int offset)`
- **Purpose:** Decompress the Huffman-coded portion of a `msg_t` in-place starting at `offset`. Reads a 2-byte uncompressed-size header, then decodes symbols one by one, calling `Huff_addRef` after each to keep the tree adaptive.
- **Side effects:** Overwrites `mbuf->data+offset` with decompressed bytes; updates `mbuf->cursize`. Uses a local 64 KB scratch buffer `seq[65536]`.
- **Notes:** Clamps output length to `mbuf->maxsize` to avoid overflow. Contains a `// FIXME` noting missing overflow guard in `get_bit`.

### Huff_Compress
- **Signature:** `void Huff_Compress(msg_t *mbuf, int offset)`
- **Purpose:** Compress the raw portion of a `msg_t` in-place. Writes 2-byte original-size header, then encodes each byte with `Huff_transmit` + `Huff_addRef`.
- **Side effects:** Overwrites `mbuf->data+offset` with compressed bits; updates `mbuf->cursize`. Note: source has a typo `mbuf->data+ +offset` (double `+`), harmless.

### Huff_Init
- **Signature:** `void Huff_Init(huffman_t *huff)`
- **Purpose:** Zero-initialize and seed both compressor and decompressor `huff_t` instances with a single NYT root node each.
- **Side effects:** Writes to `*huff` in full.

## Control Flow Notes
- `Huff_Init` is called during network channel setup (once per connection).
- `Huff_Compress` / `Huff_Decompress` are called from `msg.c` or `net_chan.c` per-packet, operating on `msg_t` buffers.
- The `bloc` global is set from `*offset` at the start of public API calls and written back on exit, making the public API re-entrant with respect to caller state but **not thread-safe** due to the shared static.

## External Dependencies
- `../game/q_shared.h` — `byte`, `qboolean`, `Com_Memset`, `Com_Memcpy`
- `qcommon.h` — `msg_t`, `node_t`, `huff_t`, `huffman_t`, `NYT`, `INTERNAL_NODE`, `HMAX` constants, all public `Huff_*` prototypes
- `oldsize` — declared `extern int`; defined elsewhere (likely `msg.c`); referenced but not used in this file's visible code paths
