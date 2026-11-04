<!-- Trusted provenance: F-f687f2eff3da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f687f2eff3da

## Purpose

F-f687f2eff3da updates a caller-supplied mutable record, invokes two external dependencies in sequence, and returns two retained byte samples in separate halves of a 64-bit result.

The function:

- Stores a character-sized conversion of an input at record offset 13.
- Snapshots two unsigned 32-bit fields and limits each snapshot to 28.
- Calls a first dependency with record-derived arguments and the original inputs.
- Writes reset values to offsets 15 and 16.
- Samples unsigned bytes at offsets 14 and 13.
- Calls a second dependency with the retained values and two interior record addresses.
- Returns the retained samples without rereading the record.

The record’s application-level meaning is not established. This contract identifies its fields by byte offset rather than assigning unsupported meanings.

## Interface

### Inputs

The supplied representation exposes four inputs:

| Input | Representation | Established use |
|---|---|---|
| Record address | Pointer in an apparent 32-bit address model | Used for direct record accesses and passed to the first dependency. Two interior addresses are passed to the second dependency. |
| Value input | Opaque 32-bit quantity | Converted to a character-sized value for the initial store; its complete 32-bit representation is forwarded to the first dependency. |
| Auxiliary input A | Opaque 32-bit quantity | Forwarded twice to the first dependency. |
| Auxiliary input B | Opaque 32-bit quantity | Forwarded twice to the first dependency. |

No integer, address, handle, or ownership interpretation is established for the three non-pointer inputs beyond these uses.

### Record layout and access widths

Offsets are relative to the record address.

| Offset or range | Source-level access | Established use |
|---|---|---|
| 0–3 | Unsigned 32-bit read | First field snapshot, subsequently limited to 28. |
| 4–7 | Unsigned 32-bit read | Second field snapshot, subsequently limited to 28. |
| 13 | Character-sized write; later unsigned 8-bit read | Initial converted value, then a post-first-call sample. |
| 14 | Unsigned 8-bit read | Post-first-call sample. |
| 15 | One-byte read; later one-byte write | First argument to the first dependency; subsequently reset to zero. Its address is passed to the second dependency. |
| 16–19 | Unsigned 32-bit read | Converted to a character-sized value and then to an integer for the first dependency. |
| 16 | One-byte write | Reset to 255 after the first dependency returns. Its address is passed to the second dependency. |

The read covering offsets 16–19 and the later write at offset 16 are different operations with different widths.

In particular, narrowing the unsigned word read at offsets 16–19 does not establish that the resulting character corresponds to the byte at offset 16. That association depends on byte order and conversion semantics.

### Storage and execution preconditions

The direct source-level operations require storage extending through offset 19 inclusive: a minimum 20-byte extent from the record address. A 17-byte extent is insufficient for the unsigned 32-bit read beginning at offset 16.

More precisely:

- Offsets 0–7 and 16–19 must support the indicated unsigned 32-bit reads.
- Offsets 13–15 must support their indicated byte reads.
- Offsets 13, 15, and 16 must support byte writes.
- Alignment and object-access requirements must permit the three unsigned 32-bit reads.
- Address calculations must designate the intended locations without invalid address arithmetic.
- Storage must remain valid for every subsequent direct access.

The dependencies may impose stronger requirements. Twenty accessible bytes cover the highest direct source-level access but do not establish either dependency’s memory footprint.

The actual machine-load width used to realize the read at offsets 16–19 is unknown. That uncertainty does not reduce the source-level storage requirement.

No explicit null, bounds, alignment, or lifetime checks are present. Violations have no specified error result in this contract.

### Result

On normal completion, the result is 64 bits.

Let \(a\) be the retained unsigned byte from offset 14 and \(b\) the retained unsigned byte from offset 13. Then:

\[
R=(b\times 2^{32})+a
\]

The multiplication and addition describe the 64-bit result, not a 32-bit intermediate calculation.

| Result bits | Value |
|---|---|
| 0–7 | Sample from offset 14 |
| 8–31 | Zero |
| 32–39 | Sample from offset 13 |
| 40–63 | Zero |

This is not a contiguous 16-bit packing.

## Behavior

### Initial update

The first direct record operation writes a character-sized conversion of the value input to offset 13.

Under conventional eight-bit truncating character conversion, the stored representation is the input’s low eight bits. Exact behavior for values outside the target character type’s representable range depends on target conversion rules and remains documented_only.

The conversion does not replace the original input. The first dependency receives the full 32-bit value separately.

### Unsigned snapshots and limiting

After the initial write, the function reads the unsigned 32-bit field at offsets 0–3. A value at most 27 is retained; any greater value is replaced by 28.

It then reads the unsigned 32-bit field at offsets 4–7 and applies the same rule independently.

For either unsigned field \(x\), the effective snapshot is:

\[
\min(x,28)
\]

The comparison is unsigned. The maximum 32-bit value, 4,294,967,295, therefore produces 28.

These limited values remain local snapshots. They are not written back to the record. Changes made by either dependency to the original fields do not replace these retained values.

### First dependency

The function next invokes **F-external-01** with eight arguments:

| Position | Argument |
|---:|---|
| 1 | One-byte value read at record offset 15. |
| 2 | Unsigned 32-bit value read at offsets 16–19, converted first to a character-sized value and then to an integer. |
| 3 | Auxiliary input A. |
| 4 | Auxiliary input B. |
| 5 | Record address. |
| 6 | Complete 32-bit value input. |
| 7 | Auxiliary input A again. |
| 8 | Auxiliary input B again. |

Argument 2 involves a source-level unsigned 32-bit read, not merely a byte read. Its character-sized conversion narrows the word’s numerical value. Without an established byte order, the word’s low eight bits cannot be identified with the byte at offset 16.

Under conventional eight-bit conversion:

- An unsigned-character interpretation yields an integer from 0 through 255.
- A signed-character interpretation with low-eight-bit preservation yields an integer from −128 through 127.

For example, a word whose low eight bits are 255 yields 255 in the former model and −1 in the latter. Plain-character signedness and out-of-range conversion behavior are not established independently.

The representation of argument 1 establishes a one-byte read but does not fully establish its extension or the dependency’s parameter interpretation.

Both argument-forming reads occur before entry to F-external-01. Their relative evaluation order is not established. Both occur after the two limited field snapshots have been prepared.

The function does not use the dependency’s return value, if it has one.

### Ordered resets and sampling

If F-external-01 returns normally, the function performs these direct operations in order:

1. Writes zero to offset 15.
2. Writes 255 to offset 16 using a one-byte write.
3. Reads offset 14 as an unsigned byte and retains its value.
4. Reads offset 13 as an unsigned byte and retains its value.

The write at offset 16 does not directly write offsets 17–19. It is not a 32-bit reset.

The samples reflect storage after F-external-01 returns and after both reset writes. In particular, the offset-13 sample need not equal the initial converted input: the first dependency receives the record address and may have changed that location.

### Second dependency

The function invokes **F-external-02** with eight arguments:

| Position | Argument |
|---:|---|
| 1 | Limited snapshot from offsets 0–3, masked to eight bits. |
| 2 | Retained unsigned byte from offset 14. |
| 3 | Retained unsigned byte from offset 13. |
| 4 | Limited snapshot from offsets 4–7, masked to eight bits. |
| 5 | Same retained offset-14 value as argument 2. |
| 6 | Same retained offset-13 value as argument 3. |
| 7 | Address of record offset 15. |
| 8 | Address of record offset 16. |

Each limited snapshot is already between 0 and 28, so its eight-bit mask changes nothing.

Arguments 2 and 5 share one retained sample. Arguments 3 and 6 share another. These four argument positions do not cause four record reads.

The final two arguments identify adjacent record locations. Their construction does not establish the dependency’s pointee types, access widths, or permitted memory extent.

The dependency’s return value, if any, is not used.

### Completion and abnormal paths

After F-external-02 returns normally, the function constructs the result from the two retained samples. It does not reread offsets 13 or 14.

There is no visible branch based on either dependency’s result.

If F-external-01 does not return normally, the subsequent resets, samples, second call, and normal result are not reached. If F-external-02 does not return normally, no normal result is produced. Faults, exceptions, and nonlocal exits have no additional established handling.

## Side effects

The function directly performs three byte writes:

| Order | Location | Written value |
|---:|---|---|
| 1 | Offset 13 | Character-sized conversion of the value input. |
| 2 | Offset 15 | Zero, after the first dependency returns. |
| 3 | Offset 16 | 255, immediately after the offset-15 reset. |

The function does not directly write the limited snapshots back to offsets 0–7. Its reset at offset 16 does not directly modify offsets 17–19.

No explicit allocation, release, ownership transfer, or address retention by this function is visible. This does not establish that its dependencies refrain from those actions.

F-external-01 receives the record address. F-external-02 receives two interior addresses. Their implementations may have additional memory, resource, or global-state effects; none are specified here.

Consequently:

- The reset values at offsets 15 and 16 are not guaranteed final values.
- The initial value written at offset 13 is not guaranteed to survive either dependency.
- The returned samples remain unchanged by later record writes during F-external-02.
- Normal continuation requires valid storage for the remaining direct operations, but the first dependency is not assumed to preserve its lifetime.

No synchronization or atomicity mechanism is visible. Concurrent modification, volatile storage, and device-memory semantics are not established.

## Dependencies

These are document-local opaque identifiers.

| Identifier | Established interaction | Unresolved contract |
|---|---|---|
| **F-external-01** | Called once after the initial write and two field snapshots; receives eight arguments including the record address and all three non-pointer inputs. | Complete signature, argument interpretation, return type, memory footprint, ownership, address retention, global effects, and failure behavior. |
| **F-external-02** | Called once after resets and sampling; receives eight arguments including retained values and addresses at offsets 15 and 16. | Complete signature, pointee widths, read/write extent, ownership, address retention, global effects, and failure behavior. |

On a normally completing invocation, F-external-01 precedes F-external-02. No dependency implementation is supplied. Neither preservation of globals nor ownership or release of memory may be assumed.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, they require valid storage, applicable conversion semantics, both dependencies returning normally, and no unrelated concurrent mutation.

### Unsigned limiting boundaries

If the initial fields at offsets 0–3 and 4–7 contain 27 and 28, arguments 1 and 4 to F-external-02 are respectively 27 and 28.

If they contain 29 and 4,294,967,295, both arguments are 28.

If F-external-01 later changes either field, these arguments still use the original limited snapshots.

### Full-width forwarding and initial narrowing

For a value input with representation `0x123456AB`, under conventional eight-bit truncating conversion:

- The initial store at offset 13 writes `0xAB`.
- Argument 6 to F-external-01 remains `0x123456AB`.

If the first dependency leaves offset 13 unchanged, its retained sample is 171. The result’s high 32-bit half is then `0x000000AB`.

### Word narrowing is not an offset-16 byte sample

Suppose offsets 16–19 contain, in increasing address order, `0x12`, `0x34`, `0x56`, and `0x78`.

Under little-endian unsigned 32-bit interpretation, the word is `0x78563412`; conventional low-eight-bit narrowing yields `0x12`.

Under big-endian unsigned 32-bit interpretation, the word is `0x12345678`; the same narrowing yields `0x78`.

Thus argument 2 to F-external-01 is conditionally 18 or 120 in these models. The source alone does not select one. In neither case may the source-level operation be reduced to an unconditional byte read at offset 16.

### Reset width and visibility

Suppose F-external-01 returns with offsets 16–19 containing `0x10`, `0x20`, `0x30`, and `0x40`.

Absent other mutation, the direct reset changes those bytes to `0xFF`, `0x20`, `0x30`, and `0x40`. It does not replace the entire word with 255.

Before entering F-external-02, offset 15 has also been written with zero, before the offset-16 write.

### First dependency determines the samples

Suppose F-external-01 changes offsets 14 and 13 to `0x23` and `0x91`.

F-external-02 receives 35 at positions 2 and 5, and 145 at positions 3 and 6. If it returns normally, the result is:

`0x0000009100000023`

The initial offset-14 value and initial offset-13 store do not override these samples.

### Second dependency cannot replace the retained result

Suppose the retained samples are 2 from offset 14 and 3 from offset 13. F-external-02 then changes both record bytes to zero.

The returned value remains `0x0000000300000002`. Final record contents may reflect the dependency’s changes.

### Storage boundary

A record allocation ending at offset 16 does not satisfy the direct source-level requirements, even if offsets 13–16 are writable. The word read beginning at offset 16 requires accessible storage through offset 19.

An extent through offset 19 addresses that direct boundary only. It does not prove adequate storage for the dependencies.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the record and operation.
- Definitive ABI, byte order, and portability beyond the apparent 32-bit address model.
- Actual machine-load width implementing the source-level word read at offsets 16–19.
- Plain-character signedness and exact out-of-range conversion rules.
- Extension and interpretation of the first dependency’s one-byte argument.
- Relative evaluation order of the first dependency’s two record-derived argument reads.
- Complete dependency signatures, memory footprints, and effects.
- Whether dependencies retain addresses, release storage, or invalidate subsequent accesses.
- Abnormal-control-flow behavior beyond the absence of visible local handling.
- Concurrency, volatile-access, and device-memory guarantees.

These unknowns do not weaken the established source-level 32-bit read or its storage requirement.

## Evidence

**Established static evidence:** The supplied body shows an initial character-sized store, two unsigned 32-bit field reads with limiting at 28, a first external call whose second argument includes an unsigned 32-bit read at offsets 16–19, two ordered byte resets, two ordered unsigned-byte samples, a second external call, and a 64-bit result assembled from retained samples. Neither dependency result is used. No explicit validation, allocation, or release is shown.

**Inferred intent:** The sequence is consistent with preparing record state, invoking an operation, resetting two locations, and handing their addresses to a second operation. This interpretation does not establish application semantics, ownership, or dependency guarantees.

**Unknowns:** Dependency implementations, definitive target conversion rules, and runtime behavior are not supplied. No runtime tests have been performed.
