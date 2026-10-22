<!-- Trusted provenance: F-c5cc0d64a43b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c5cc0d64a43b

## Purpose

F-c5cc0d64a43b conditionally updates a record’s 16-bit count after examining 32-bit entries in descending position order.

For a positive count, stable storage, and supported integer conversions, the observable effect is to exclude trailing zero entries from the counted region. A nonzero final entry causes an immediate return without modifying the count. Otherwise, the resulting count includes all positions through the last nonzero entry, or becomes zero if all counted entries are zero.

This is a boundary adjustment, not compaction. Interior zeros remain inside the resulting region. No table entry is cleared or moved.

An initial count of zero has distinct behavior: the function examines a word immediately before the table base. A nonzero word leaves the count unchanged; a zero word causes a count write of 65,535.

Trailing-zero removal is an inferred purpose. The established behavior concerns numeric values and memory accesses; it does not establish what the entries represent.

## Interface

### Input and result

The function accepts one address identifying a mutable record. It returns no explicit value and exposes no separate success or error status.

The supplied representation supports this record layout:

| Byte offset | Width | Established use |
|---|---:|---|
| 0 | 16 bits | Unsigned count, read once and conditionally overwritten |
| 2–3 | Not directly accessed as a field | Unspecified |
| 4 | 32 bits | Address-bearing value used to select table entries |

The table consists of four-byte units. Each examined unit is read as a 32-bit integer and distinguished only as zero or nonzero. A nonzero value with its high bit set has the same stopping effect as any other nonzero value.

The address-bearing field is accessed through a 32-bit integer representation. This does not establish compatibility with wider pointers or a portable address encoding.

### Access and lifetime requirements

Normal completion requires the record’s count field to be readable, each selected address-field access to be valid, and every selected table word to be readable. A path that updates the count additionally requires a valid 16-bit write to the count field.

The function does not validate:

- The argument address.
- The table address or alignment.
- The available table extent.
- Whether the count fits that extent.
- Whether record and table storage overlap.
- The validity of integer-based address construction.

The required readable table region depends on the values encountered. A positive count does not necessarily cause the entire counted region to be examined. A zero count nevertheless requires a readable word immediately before the table base.

No allocation, ownership transfer, retention, or release is expressed. Required storage must remain available for the accesses actually performed. Any broader ownership or lifetime contract is unknown.

## Behavior

### Observable access contract

At the level expressed by the supplied body, the count is read once before any table access. That captured value determines the initial position and the extent of the descending examination. Later changes to the stored count do not provide a new search boundary.

The first selected position is the position immediately below the captured count boundary:

- A positive count selects its last counted entry.
- A zero count selects the word four bytes before the table base.

The address-bearing field is read before the selected word. The first word is examined without a preceding lower-bound check.

A nonzero first word terminates the function without a count write and without examining another table word.

A zero first word places execution on the update path. Further table reads, when required, visit consecutive lower positions. Each such read is preceded by another access to the address-bearing field. Examination ends on a nonzero entry or upon exhausting the permitted positions. The count is then written exactly once.

For positive counts, exhaustion does not cause a read before position zero. The exceptional pre-base access belongs to the initial selection for a zero count.

These statements describe the memory expressions and their ordering in the supplied body. They do not promise distinct physical bus transactions for ordinary memory, or establish volatile, device-memory, or concurrent-access semantics.

### Positive counts and stable storage

For stable, valid storage, the final nonzero entry within the initially counted region determines the retained boundary.

If the final counted entry is nonzero, the count is not written. Otherwise, the function writes a boundary immediately after the last nonzero entry it encounters. If no counted entry is nonzero, it writes zero.

The resulting region may contain zeros. For example, a nonzero entry followed by an interior zero and then another nonzero entry remains a region containing that interior zero. The function neither counts nonzero entries nor rearranges them.

No position above the initial boundary is examined. The number of table-word reads is at least one and at most the initial positive count. Earlier positions below a discovered nonzero entry are not examined.

The numerical boundary interpretation for large retained regions is subject to the signed-conversion qualification below.

### Zero count

A zero count is not recognized as an empty region before accessing table-relative storage.

Exactly one table word is examined: the word four bytes before the selected base.

- If that word is nonzero, the function returns without writing the count.
- If that word is zero, the function writes 65,535.

Neither outcome involves a read eight bytes before the base. The second outcome is not a scan farther into preceding storage.

The body does not establish whether callers exclude zero counts, whether this behavior is intentional, or whether it represents an invalid caller state.

### Count conversion and width boundaries

The update uses a signed 16-bit conversion of the stopping position before producing the value assigned to the unsigned 16-bit count.

When the stopping position is within the signed 16-bit range, the resulting boundary is established without a high-range signed-conversion ambiguity:

- Exhausting a positive counted region produces count zero.
- Stopping at a nonzero entry in positions zero through 32,767 produces a count from one through 32,768.
- The zero-count path whose preceding word is zero produces 65,535.

For stopping positions from 32,768 upward, the conversion to signed 16 bits is not portably determined by the supplied text alone. Under low-16-bit, two’s-complement narrowing, the eventual unsigned count is the ordinary boundary immediately after the discovered nonzero entry.

That target convention must remain an explicit condition for high-range update examples. A target-specific conversion result or implementation-defined signal is not excluded by the available evidence.

A large initial count does not itself require this ambiguous conversion. Immediate return performs no count conversion or write. A search starting high may also descend to a representable stopping position before updating.

### Address arithmetic

The initial position ranges from minus one through 65,534. Its four-byte displacement ranges from minus four through 262,136. Those displacements fit within signed 32-bit arithmetic.

This does not establish that combining a displacement with the stored address-bearing value is valid. The address addition can fall outside the supported signed arithmetic range or produce an unusable address. The body provides no overflow, wraparound, accessibility, or alignment guarantee.

Invalid arithmetic and invalid accesses are outside the normal-completion contract. There is no explicit recovery or error-reporting path.

## Side effects

The only explicit write is a 16-bit store to the count field. It occurs only after the first examined word was zero and after all table reads for that execution have finished.

The function does not explicitly:

- Modify table entries.
- Clear entries outside the resulting count.
- Replace the address-bearing field.
- Allocate or release storage.
- Acquire or relinquish ownership.
- Call another function.
- Publish an error status.

These are access-role statements, not a non-aliasing guarantee. If table-relative addresses overlap record storage, a table read can observe record bytes. If the count field lies within storage also treated as table data, the final count store changes those shared bytes. There is no direct field access to bytes 2–3, but an overlapping table read may include them.

The function’s final write cannot change an earlier search decision in the same invocation because it follows all search reads.

The record and table are not captured as an atomic snapshot. The initial count is captured once, whereas the address-bearing field is expressed as a fresh access for each examined table word. If an execution environment permits defined, observable changes to that field between accesses, later reads can use a different base.

This does not make unsynchronized concurrent mutation valid. No synchronization or atomicity guarantee is established.

## Dependencies

There are no called-function dependencies in F-c5cc0d64a43b and therefore no additional opaque function IDs to list.

Normal execution instead depends on environmental properties:

- The represented 16-bit and 32-bit integer widths.
- Signed narrowing and unsigned assignment rules.
- Interpretation of the 32-bit address-bearing value.
- Validity of integer-based address construction.
- Storage extent, alignment, accessibility, and lifetime.
- Aliasing and any supported external mutation semantics.

No unknown callee can be credited with validating addresses, preserving state, or managing memory: no call is present.

The evidence does not identify whether the record or table resides in stack, heap, global, shared, mapped, or device storage.

## Acceptance examples

These are conditional expectations from static evidence, not executed tests.

Unless otherwise specified, assume stable, valid, nonoverlapping storage, supported address arithmetic, and zero-based table positions. Each table-word read is preceded by an access to the address-bearing field.

| Initial state | Required behavior |
|---|---|
| Count 4; position 3 is nonzero | Examine position 3 only. Return without writing the count. |
| Count 4; position 3 is zero; position 2 is nonzero | Examine positions 3 and 2, in that order. Write count 3. |
| Count 4; positions 3 through 1 are zero; position 0 is nonzero | Examine positions 3 through 0 in descending order. Write count 1. |
| Count 4; all counted entries are zero | Examine positions 3 through 0. Write count zero. Do not read before the base. |
| Count 1; position 0 is nonzero | Examine position 0 and return without a count write. |
| Count 1; position 0 is zero | Examine position 0 and write count zero. Do not read before the base. |

Additional boundary cases:

- **Interior zeros do not trigger compaction.** With count 5 and a nonzero value at position 4, earlier positions are not examined, regardless of any zeros there.
- **Any nonzero bit pattern stops examination.** A word whose signed interpretation is negative is still nonzero and has the same stopping effect.
- **Entries above the boundary are irrelevant.** With count 2, a zero at position 1 and a nonzero value at position 0 produce count 1. Position 2 is not read.
- **Zero count with nonzero preceding word.** Read the word four bytes before the base and return without a count write.
- **Zero count with zero preceding word.** Read that same word and write 65,535. Perform no additional table-word read.
- **Maximum count with immediate return.** With count 65,535 and a nonzero value at position 65,534, examine that position and return without a count write. No high-range signed narrowing occurs.
- **Maximum count with a one-entry reduction.** With count 65,535, a zero at position 65,534, and a nonzero value at position 65,533, examine those two positions. Under low-16-bit, two’s-complement narrowing, write 65,534. Without an established target conversion rule, that numerical update remains conditional.
- **High initial boundary, representable retained boundary.** With count 32,769, a zero at position 32,768, and a nonzero value at position 32,767, examine those positions and write 32,768. The stopping position is representable as signed 16 bits.
- **Changing base under a defined observation model.** If the environment supports an observable base-field change between table accesses, later accesses use the later field value. Requiring one captured base for the entire invocation would add an unsupported guarantee.

## Uncertainties

### Inferred intent

For positive counts and stable storage, excluding a zero-valued suffix is the strongest supported functional interpretation. The meanings of zero, nonzero entries, and the count are not established.

The zero-count behavior may reflect an excluded caller state, an intentional convention, or a defect. The body alone does not distinguish these possibilities.

### documented_only

The following remain unresolved:

- Caller restrictions on zero counts and table capacity.
- Exact target address encoding, byte order, and alignment requirements.
- The target’s signed 16-bit conversion behavior for large positive positions.
- Address-addition behavior outside the valid arithmetic range.
- Whether overlapping record and table storage is supported.
- Whether external mutation is supported under a defined memory model.
- Failure behavior for inaccessible storage or exceptional conversions.
- Ownership and lifetime obligations beyond the accesses required here.

These uncertainties must not be replaced with assumptions about ordinary pointers, address wraparound, automatic bounds validation, or resource management.

## Evidence

Static evidence establishes the initial count read, the address-bearing field at byte offset 4, four-byte table spacing, zero/nonzero decisions, descending examination, conditional 16-bit count store, and absence of calls.

The first table access precedes any lower-bound check. This establishes both zero-count outcomes and distinguishes them from positive-count exhaustion.

The expressed repeated accesses to the address-bearing field prevent describing the body as capturing a single base. The signed 16-bit conversion on the update path requires a target qualification for high retained boundaries.

The table’s semantic purpose, caller guarantees, and exceptional platform behavior are not established. Trailing-zero removal is inferred intent; unresolved environmental and interface properties remain documented_only.

No runtime tests have been performed.
