<!-- Trusted provenance: F-fe1c70369baf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe1c70369baf

## Purpose

F-fe1c70369baf initializes selected fields in a sequence of destination records using corresponding pairs of 32-bit source words.

For each processed record, it clears two fields and assigns two further fields from a source pair. A zero selector retains the pair’s order; any nonzero selector reverses it. This describes the resulting values when source and destination do not interfere. With overlapping storage, the ordered reads and writes specified below determine the result; the operation is not an overlap-safe pair swap.

**Established static evidence:** the supplied body contains descriptor reads, a count-controlled loop, 32-bit memory transfers, and fixed-stride address advancement. It contains no calls and returns no value.

**Inferred intent:** the operation prepares a collection of records while selecting the orientation of a pair of values. The meanings of those records, the cleared fields, and the pair are not established.

## Interface

### Inputs

The function accepts two arguments:

1. **Descriptor address.** An address used to obtain a record count, destination base address, and an indirect source base address.
2. **Selector.** A 32-bit integer tested only for equality with zero. Zero selects original order; every nonzero value selects reversed order. Negative selector values are not rejected or treated separately.

The visible representation uses four-byte words and four-byte descriptor slots. The following offsets are byte offsets:

| Location relative to descriptor | Width | Interpretation |
|---|---:|---|
| +0 | 32 bits | Loop count |
| +8 | 32 bits | Destination base address |
| +20 | 32 bits | Address of an auxiliary structure |

At byte offset +4 within the auxiliary structure is a 32-bit source base address.

The descriptor fields at other offsets are not accessed directly by this body. No descriptor length, source length, destination capacity, or validity flag is consulted.

### Source layout

Each logical source entry consists of two consecutive 32-bit words:

| Offset within source entry | Width | Role |
|---|---:|---|
| +0 | 32 bits | First word |
| +4 | 32 bits | Second word |

Successive source entries begin eight bytes apart. The words are transferred as opaque 32-bit values. There is no visible conversion, arithmetic on their contents, comparison, or interpretation as pointers.

### Destination layout

Successive destination records begin 48 bytes apart. Within each record, the function directly writes these fields:

| Offset within destination record | Width | Assigned value |
|---|---:|---|
| +16 | 32 bits | Zero |
| +32 | 32 bits | Zero |
| +36 | 32 bits | Selected first output word |
| +40 | 32 bits | Selected second output word |

For a zero selector, the outputs at +36 and +40 come from the first and second source words, respectively. For a nonzero selector, they ordinarily come from the second and first source words, respectively.

There is no visible whole-record clearing. Bytes outside these four fields receive no direct destination-field write from this function. Aliasing can nevertheless cause one of these writes to affect storage serving another role.

### Outputs and ownership

The function has no return value or explicit error result. Its output consists of memory writes.

No allocation, deallocation, ownership transfer, reference-count update, or resource release appears in the body. The caller must provide storage suitable for all accesses actually reached. This does not establish the lifetime or ownership rules imposed by the surrounding application.

## Behavior

### Initial memory reads

Before checking whether the count is zero, the function obtains its working addresses and count in this order:

1. Reads the auxiliary-structure address from descriptor offset +20.
2. Reads the source base address from auxiliary-structure offset +4.
3. Reads the destination base address from descriptor offset +8.
4. Reads the count from descriptor offset +0.

Consequently, a zero count does not avoid the auxiliary-structure dereference. The descriptor and the auxiliary location containing the source address must still be readable.

The resulting source address, destination address, and count are retained as working values. The body does not reread the descriptor or auxiliary structure during the loop. A later write that overlaps one of those structures therefore does not, by itself, replace a working address or reload the count.

This read order is the order expressed by the supplied body. The evidence does not establish additional volatile, atomic, or concurrent-observation guarantees.

### Iteration and field-access order

The loop proceeds while the working count is nonzero. Each iteration handles one 48-byte destination record and one eight-byte source entry.

The first two memory operations within every iteration are writes:

1. Write a 32-bit zero at destination offset +16.
2. Write a 32-bit zero at destination offset +32.

The remaining source reads and destination writes depend on the selector.

**Zero selector:**

1. Read the first source word.
2. Write that word to destination offset +36.
3. Read the second source word.
4. Write that word to destination offset +40.

**Nonzero selector:**

1. Read the second source word.
2. Write that word to destination offset +36.
3. Read the first source word.
4. Write that word to destination offset +40.

The count is decremented once per iteration. In the supplied expression order, the decrement occurs before the second source-word read: after the +36 write on the zero-selector path, and before the first source read on the nonzero-selector path. The count change is local; it is not written back to the descriptor.

The source working address advances by eight bytes per iteration. Its advancement is expressed before the final destination-field write, but does not fetch the next entry. The destination working address advances by 48 bytes after the +40 write.

For a positive count of N, with all required addresses valid and no arithmetic overflow, the function processes exactly N entries.

### Aliasing and read-after-write effects

The function does not first capture both source words. It also clears destination fields before reading either source word. These details are part of the behavior.

If a clearing write overlaps a source word, the later source read can observe zero rather than the word’s earlier value. If the write at destination offset +36 overlaps a source word that has not yet been read, the later read can observe the value just written.

The same principle applies across iterations: writes for an earlier destination record can alter source data consumed by a later iteration.

Accordingly, the simple pair-order description requires a noninterference condition: preceding writes must not change source words before their reads. Without that condition, the ordered accesses above define the result, subject to valid memory and applicable execution semantics.

### Count and address arithmetic

The count is represented as a 32-bit integer and tested for nonzero, not for positivity. There is no explicit rejection of a negative count.

For ordinary positive counts, decrementing reaches zero after the stated number of iterations. A negative count does not receive zero-count treatment and enters the loop.

The supplied body alone does not establish safe termination for negative counts. Under modulo-2³² decrement semantics, a starting count bit pattern would reach zero after its unsigned numerical value in iterations. For example, an all-ones count would require 4,294,967,295 iterations. That is a conditional arithmetic interpretation, not a guarantee that the memory accesses or signed arithmetic remain valid.

Address progression uses strides of eight and 48 bytes. No bounds or overflow checks are visible. For positive N, the last record’s highest directly accessed destination byte is at offset 48 × (N − 1) + 43 from the initial destination base. The source entries occupy offsets 0 through 8 × N − 1.

These expressions describe the required spans mathematically. They are not evidence that the implementation performs checked multiplication or can safely traverse a span that exceeds its address representation.

## Side effects

For each completed iteration, the body performs four 32-bit destination writes:

- Zero at +16.
- Zero at +32.
- A selected source value at +36.
- The other selected source value at +40.

It also reads each source word once per iteration, with selector-dependent order.

The descriptor count and stored base addresses are not explicitly updated. The auxiliary structure is not explicitly updated. Either may nevertheless be modified if destination writes alias their storage.

There are no visible direct accesses to named global state and no visible calls through which additional behavior could occur. This does not rule out memory-mapped effects or other behavior attached to the supplied addresses.

There is no visible synchronization or transaction mechanism. If an access fails after earlier writes have occurred, the body provides no rollback. The exact failure mechanism and resulting externally observable state are not established by the supplied source alone.

## Dependencies

No dependency calls appear in the supplied body. There are therefore no separate opaque function IDs to invoke or describe for this operation.

Its execution depends on caller-provided memory and the execution environment:

- Readable descriptor fields at the stated offsets.
- A readable source-address field in the auxiliary structure.
- Readable source words for every reached iteration.
- Writable destination fields for every reached iteration.
- Valid addressing and alignment under the target’s memory-access rules.
- Arithmetic behavior sufficient for the intended count and address progression.

No assumption about an unknown callee preserving globals or releasing memory is needed: no such callee is visible.

## Acceptance examples

The following are conditional acceptance examples derived from static evidence, not runtime test results. Unless specified otherwise, all accesses are valid, source and destination do not overlap, and arithmetic does not overflow.

### Zero selector preserves pair order

Given a count of one and source words 0x11223344 and 0xAABBCCDD:

- Destination +16 becomes zero.
- Destination +32 becomes zero.
- Destination +36 becomes 0x11223344.
- Destination +40 becomes 0xAABBCCDD.

Other destination bytes receive no direct write.

### Any nonzero selector reverses pair order

With the same source pair and count, selectors 1, −1, and any other nonzero 32-bit value select the same branch:

- Destination +16 and +32 become zero.
- Destination +36 becomes 0xAABBCCDD.
- Destination +40 becomes 0x11223344.

There is no distinct behavior based on the magnitude or sign of a nonzero selector.

### Multiple entries use different source and destination strides

Given a count of two and consecutive source pairs (10, 20) and (30, 40), a zero selector produces:

- First destination record: values 10 and 20 at offsets +36 and +40.
- Second destination record, beginning 48 bytes later: values 30 and 40 at its offsets +36 and +40.
- Both records: zeros at their respective +16 and +32 fields.

The second source entry begins eight bytes after the first, not 48 bytes after it.

### Zero count still performs setup reads

Given a zero count and valid descriptor and auxiliary-field reads, the function performs no source-word reads and no destination writes.

The auxiliary structure must nevertheless support the initial source-address read. An invalid auxiliary address is not made harmless by the zero count.

### Overlap prevents an ordinary in-place swap

Suppose the source pair begins exactly at destination offset +36, initially containing distinct values A and B, and the selector is nonzero.

Assuming ordinary coherent memory behavior:

- The second source word, B, is read.
- B is written to destination +36, replacing the first source word.
- The first source word is then read as B.
- B is written to destination +40.

The resulting pair is (B, B), not (B, A). This follows from the read/write ordering.

### Clearing can change source input before it is read

Suppose the first source word occupies the same storage as destination offset +16, and the selector is zero.

The initial clearing write replaces that source word with zero before the source read. Destination +36 therefore receives zero, regardless of the word’s earlier value.

### A negative count is not rejected

Given a negative count and initially valid addresses, the function enters the loop rather than treating the count as empty or returning an error. Safe completion is not established; any stronger acceptance condition depends on arithmetic semantics and the validity of every subsequent access.

## Uncertainties

The following are **documented_only**:

- The business meaning of the descriptor, destination fields, and source pair.
- Whether callers enforce nonnegative counts, nonoverlapping storage, alignment, and adequate capacity.
- The target’s complete pointer, signed-overflow, alignment, and invalid-access semantics.
- Whether any supplied addresses refer to volatile or memory-mapped storage.
- Behavior under concurrent modification or observation.
- Exact failure behavior for inaccessible memory or address overflow.
- Whether external application rules assign ownership or lifetime obligations beyond the accesses visible here.

The four-byte widths and strides are established by the supplied representation. Byte order does not affect opaque word copying, but it would matter when interpreting those words as individual bytes; no particular byte order is required by this contract.

No runtime tests have been performed.

## Evidence

This contract is based only on the supplied static function body.

The body establishes:

- Descriptor reads at byte offsets +20, +8, and +0, with an intervening indirect source-address read at auxiliary offset +4.
- A nonzero-count loop with one decrement per iteration.
- Four 32-bit destination writes at offsets +16, +32, +36, and +40.
- Two 32-bit source reads per iteration.
- Selector-dependent source-read order.
- Eight-byte source progression and 48-byte destination progression.
- No visible calls, allocation, release, validation branch, error result, or descriptor writeback.

The inferred purpose is limited to initializing records and selecting pair orientation. Broader semantics and runtime guarantees remain explicitly unestablished.
