<!-- Trusted provenance: F-7499e6caeab2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7499e6caeab2

## Purpose

F-7499e6caeab2 conditionally fills a shared halfword buffer with one selected 16-bit value, then invokes two dependencies in sequence.

The operation is gated by a pointer held in shared context state. A null pointer suppresses the fill and both dependency calls. A non-null pointer enables the operation even when the associated element count is zero.

The apparent intent is to prepare a uniform buffer and submit it with accompanying metadata. That interpretation is inferred; the established behavior consists of memory reads, halfword writes, and the two calls described below. No display, transfer, allocation, or synchronization semantics are established for those calls.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Visible representation | Use |
|---|---|---|
| Value selector | 32-bit unsigned integer | Supplies either its low 16 bits or an index into a four-entry halfword table. |
| Mode | 32-bit signed integer | Selects the table-based value only when exactly equal to 2; also forwarded to the first dependency. |
| Auxiliary input A | Opaque 32-bit value | Forwarded unchanged to the first dependency. |
| Auxiliary input B | Opaque 32-bit value | Forwarded unchanged to the first dependency. |

The visible interface has no return value. There is no explicit success code, error code, or output argument.

The widths above follow the supplied recovered type annotations. Original source-language declarations and calling-convention details are not independently available.

### Shared state

The contract uses these descriptive names, which identify roles rather than source symbols:

- **Context-address slot:** shared state containing a context address.
- **Buffer-address slot:** shared state containing a destination-buffer address.
- **Selection-table address:** shared state identifying a table used in mode 2.

The relevant memory layouts are:

| Object | Offset | Access |
|---|---:|---|
| Context | 4 bytes | A pointer to a record |
| Record | 0 bytes | Unsigned 16-bit count during the fill; first 16-bit metadata value after the first dependency |
| Record | 2 bytes | Second 16-bit metadata value after the first dependency |
| Selection table | 0, 2, 4, or 6 bytes | Selected 16-bit fill value |
| Destination buffer | Twice the element index | 16-bit fill write |

The record pointer used during the fill is captured before the null check. Later metadata accesses reload the pointer from context rather than relying on that captured pointer.

The supplied material does not establish the complete layout or extent of any object. In particular, a record whose initial count is zero still needs a readable second halfword if execution reaches the second dependency call.

### Preconditions

For ordinary, defined completion:

- The context pointer field at byte offset 4 must be readable, even when its contents are null.
- If the initial record pointer is non-null, its first halfword must remain readable wherever the function reads the count.
- Every destination halfword actually written must be writable.
- Mode 2 requires the selected table halfword to be readable.
- After the first dependency returns, the context used for metadata must contain pointers supporting the two metadata reads.
- Address calculations must identify valid storage without invalid address wraparound.

No explicit checks enforce these conditions. Alignment requirements, invalid-access behavior, and the target’s fault mechanism are not specified.

## Behavior

### 1. Capture shared addresses and test the record pointer

Before testing whether work is enabled, the function captures the destination address from the buffer-address slot and captures the context address from the context-address slot.

It then obtains the initial record pointer by reading the pointer field at offset 4 through the context address supplied by shared state. This lookup is distinct from the earlier context-address capture.

In an ordinary execution without intervening mutation, both context-address reads identify the same context. The visible accesses do not establish an atomic snapshot.

If the initial record pointer is null:

- No selection-table entry is read.
- No record count is read through that null pointer.
- No destination halfword is written.
- Neither dependency is called.
- The function returns without a value.

The initial shared-address reads and context-field lookup still occur on this path.

### 2. Select one fill value

For a non-null initial record pointer, the default fill value is the low 16 bits of the value-selector argument. Higher bits are discarded, not saturated.

When the mode is exactly 2, the default is replaced by a halfword read from the selection table. The table index is the low two bits of the original 32-bit value selector, giving an index from 0 through 3. The corresponding byte displacement is twice that index.

Consequences:

- Modes other than 2 do not read the selection table.
- A negative mode uses the default low-16-bit rule.
- In mode 2, selector bits above bit 1 do not affect the table index.
- The selected halfword is read once and reused for every fill write. It is not reread per element.

### 3. Fill against a repeatedly read count

The function writes the selected value to successive halfwords beginning at the captured destination address.

The element index begins at zero. Before each potential write, the function reads the unsigned 16-bit count from the first halfword of the captured initial record. It writes an element only while the index is less than that count. After each write, the index increases by one.

The count is therefore a live loop bound, not a count captured once before filling.

For a stable count of \(N\):

- Exactly \(N\) halfword writes occur.
- Writes proceed in ascending address order.
- Written byte offsets are \(0, 2, \ldots, 2(N-1)\).
- A count of zero produces no destination writes.

If another agent, an aliasing destination write, or an externally visible memory effect changes the record’s first halfword during filling, later decisions use the newly read value. The initial count alone does not determine the number of writes.

The count ranges from 0 through 65,535. The index is a 32-bit unsigned quantity. Under these visible widths, the loop cannot perform more than 65,535 writes: once the index reaches 65,535, no possible 16-bit count is larger. The loop’s index increment therefore does not approach 32-bit wraparound.

For the largest stable count, the final write begins at byte offset 131,068 and covers offsets 131,068–131,069. These bounds do not establish that the destination allocation is large enough.

### 4. Invoke F-a460a9009a02

After filling stops, the function calls F-a460a9009a02 with seven arguments:

| Position | Argument |
|---:|---|
| 1 | Destination address read from the buffer-address slot for this call |
| 2 | A fresh unsigned 16-bit count read from the captured initial record, multiplied by 2 |
| 3 | The captured initial record pointer |
| 4 | The destination address captured before filling |
| 5 | Original mode |
| 6 | Original auxiliary input A |
| 7 | Original auxiliary input B |

The byte count is formed by widening the 16-bit count to an unsigned 32-bit value and shifting left by one bit. Its range is 0 through 131,070, so this calculation does not overflow 32 bits.

This fresh count need not equal either the initial count or the number of completed fill writes. With mutable or aliased memory, the first dependency can receive a byte count that differs from twice the number of elements written.

The first and fourth arguments also need not match: one uses the shared buffer-address slot at call preparation, while the other uses the earlier captured address.

This call occurs even when the fill performs zero writes. Its return value, if any, is not used by the visible function.

### 5. Reload metadata and invoke F-639865e28f2b

If F-a460a9009a02 returns normally, the function calls F-639865e28f2b with seven arguments:

| Position | Argument |
|---:|---|
| 1 | Destination address read from the buffer-address slot after the first dependency returns |
| 2 | Zero |
| 3 | First 16-bit metadata value, read through the pointer at offset 4 of the previously captured context |
| 4 | Second 16-bit metadata value, at byte offset 2 through a pointer loaded from that same context field |
| 5 | Zero |
| 6 | Zero |
| 7 | 52, hexadecimal 0x34 |

These metadata reads occur after the first dependency. They are not a snapshot preserved from before that call.

The context-field pointer is separately loaded for the two metadata expressions. The supplied material does not establish an evaluation order among these argument expressions. If the pointer field can change during their evaluation, the two metadata values are not guaranteed to come from the same record.

There is no second null check before these metadata accesses. If the first dependency changes the context’s record pointer to null or another invalid address, the wrapper does not guard the subsequent reads.

Changes by the first dependency to the shared context-address slot do not redirect these accesses: they use the context address captured earlier. Changes to the pointer field inside that captured context can affect them.

After the second dependency returns normally, the function returns without a value.

## Side effects

### Direct effects

The function directly performs:

- Shared-address and pointer-field reads.
- One selected table read in mode 2 on the non-null path.
- Repeated 16-bit count reads.
- Ascending 16-bit writes to the captured destination.
- Additional count, destination-address, and metadata reads for dependency arguments.
- Two dependency calls on the non-null path.

No direct allocation, release, pointer-field assignment, shared-address-slot assignment, rollback, or explicit error reporting is visible.

### Aliasing and ownership

No non-aliasing guarantee is established. The destination may overlap the initial record, selection table, context, or shared state unless a caller-side contract excludes such overlap.

This matters because a destination write can change a later count or metadata read. A table overlap alone does not change the fill value after selection: the value has already been read.

The function passes addresses to F-a460a9009a02. The supplied material does not establish whether that dependency borrows, retains, modifies, or releases the referenced storage. It must not be assumed to preserve shared globals or memory.

Likewise, no preservation or ownership guarantees are available for F-639865e28f2b. The wrapper itself shows no explicit ownership transfer, but that does not establish the dependencies’ contracts.

### Errors and interruption

There is no explicit validation or recovery path. If a memory access fails, or a dependency faults, does not return, or otherwise interrupts execution, the visible function supplies no rollback.

Writes completed before such an interruption are not visibly undone. The second dependency is reached only after the first returns normally.

## Dependencies

### F-a460a9009a02

Called once on every non-null initial-record path, after filling.

Established at this boundary:

- Seven arguments are supplied in the order listed above.
- Two destination-related arguments may differ.
- The length is derived from a fresh count read.
- The initial record pointer is passed.
- Mode and both auxiliary inputs are forwarded unchanged.

Its behavior, return convention beyond the unused result, memory access extent, ownership rules, and effects on shared state are **documented_only**.

### F-639865e28f2b

Called once after F-a460a9009a02 returns normally.

Established at this boundary:

- Seven arguments are supplied in the order listed above.
- The destination and metadata are obtained after the first dependency.
- Three arguments are zero.
- The final argument is 52.
- No dependency result is used by the wrapper.

The meanings of the metadata and constants, and all dependency-internal effects, are **documented_only**.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests.

### Null initial record

Given readable shared state and context, with a null pointer at context offset 4:

- The function performs no fill writes.
- Neither dependency is invoked.
- Mode 2 does not cause a table read.

This does not eliminate the prerequisite that the context pointer field itself be readable.

### Direct-value fill

Given stable, valid, non-overlapping state, mode 0, selector 0x12345678, and count 3:

- Three halfwords become 0x5678, in ascending destination order.
- F-a460a9009a02 receives byte count 6.
- Its record argument is the initial record pointer.
- Assuming that dependency preserves the relevant state, F-639865e28f2b receives the record’s first two halfwords as metadata.

### Table-selected fill

Given mode 2, selector 0xFFFFFFFE, count 2, and a selected table entry of 0xA135:

- The table index is 2.
- Two destination halfwords become 0xA135.
- The selector’s low 16-bit value is not used as the fill value.

### Zero count with a non-null record

Given a non-null record whose count remains zero:

- No destination halfword is written.
- F-a460a9009a02 is still invoked with byte count zero.
- If it returns normally and the metadata addresses remain valid, F-639865e28f2b is also invoked.

### Count changed by an overlapping write

Suppose the destination begins at the initial record’s first halfword, the initial count is 3, and the selected fill value is zero. Assume ordinary memory behavior and no other mutation.

- The first loop check admits one write.
- That write changes the count to zero.
- The next loop check stops filling.
- F-a460a9009a02 receives byte count zero despite one completed halfword write.

### State changed by the first dependency

Suppose the first dependency returns after changing the buffer-address slot and replacing the record pointer inside the captured context with another valid record.

- F-639865e28f2b receives the new destination address.
- Its metadata comes through the replaced pointer, subject to the separate-read qualification above.
- The wrapper does not repeat the fill.

## Uncertainties

The following remain **documented_only**:

- Original source types, target byte order, pointer width, alignment rules, and calling convention.
- Storage capacities and complete context, record, and table layouts.
- Whether the relevant memory is ordinary RAM, volatile storage, or memory-mapped hardware.
- Synchronization rules and whether concurrent mutation is permitted.
- Evaluation order among dependency argument expressions.
- Dependency ownership, lifetime, mutation, failure, and non-return behavior.
- The semantic meanings of the record metadata, mode values other than the selection rule, auxiliary inputs, and constant 52.
- Whether callers enforce non-aliasing or stable-count preconditions.

Buffer preparation followed by submission is inferred intent, not an established external effect.

## Evidence

The supplied static representation establishes the null-pointer gate, exact mode-2 selection rule, 16-bit value truncation, four-entry table indexing, repeated unsigned count reads, ascending halfword writes, dependency order, argument construction, and post-dependency metadata reloads.

It also establishes the absence of visible allocation, release, explicit validation, rollback, and returned status in this wrapper. It does not establish their absence inside dependencies.

No implementation access, dependency bodies, caller contracts, or runtime observations were supplied. No runtime tests have been performed.
