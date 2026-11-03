<!-- Trusted provenance: F-b4957f9fffe6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b4957f9fffe6

## Purpose

F-b4957f9fffe6 marks an eligible record and conditionally propagates processing through linked records. Each completed traversal step decrements an 8-bit counter in the destination record. Processing continues at that destination only when the decremented counter is zero.

The observable behavior is a combination of:

- Early termination for a null current record, an already-set flag, or a context-dependent eligibility failure.
- Setting bit `0x04` in a record’s flag byte before checking whether that record has a successor.
- Selecting a successor through a base address and a signed 16-bit index with a 40-byte stride.
- Decrementing the successor’s counter before deciding whether to examine its eligibility or flags.

Calling the flag a “processed” marker and the counter an outstanding-dependency count is **inferred intent**, not an established application-level meaning.

## Interface

The function has two inputs and no explicit return value:

| Input | Role |
|---|---|
| Context address | Address of a structure with a 32-bit condition field at byte offset `12`. |
| Initial record address | Address of the first record, or null. |

The recovered representation uses 32-bit address-bearing fields, 32-bit integer slots, 16-bit index and eligibility accesses, and 8-bit flag and counter accesses. This contract describes that recovered memory model; it does not establish portability to a different pointer width, byte order, or alignment regime.

### Relevant memory layouts

Only the following fields are evidenced. Names describe their use here, not an externally established type.

**Context**

| Byte offset | Access width | Use |
|---|---:|---|
| `12` | 32 bits | A zero value disables the additional eligibility test; any nonzero value enables it. |

**Record**

| Byte offset | Access width | Use |
|---|---:|---|
| `0` | 32 bits | Address of an eligibility-bearing object. Used only when the context condition is nonzero. |
| `4` | 32-bit slot, interpreted through its signed low 16 bits | Successor index. A negative interpreted value terminates processing after marking the current record. |
| `12` | 8 bits | Flag byte; bit `0x04` is tested and set. |
| `13` | 8 bits | Counter, decremented when this record is reached as a successor. |
| `16` | 32 bits | Address of a link descriptor used to locate the successor-record array. |

**Eligibility-bearing object**

| Byte offset | Access width | Use |
|---|---:|---|
| `2` | 16 bits | Bit `0x0001` determines eligibility when the context condition is nonzero. |

**Link descriptor**

| Byte offset | Access width | Use |
|---|---:|---|
| `16` | 32 bits | Base address used for successor selection. |

The successor address is the selected base plus 40 times the nonnegative, signed-16-bit index.

### Preconditions

A null initial record is accepted without accessing the context.

Otherwise, every address reached by the executed path must support the corresponding access. The function provides no explicit checks for null subordinate addresses, address-range validity, alignment, array bounds, writability, or address arithmetic overflow.

Writable storage is required for any flag byte that is marked and any successor counter that is decremented. Other fields need only be readable unless aliasing causes those writes to affect them.

The function receives addresses but establishes no allocation, ownership-transfer, or lifetime-management contract.

## Behavior

### Current-record checks

Processing starts with the initial record and may repeat with a successor.

At each iteration, the following checks occur in order:

1. **Null current record:** terminate immediately. No record or context memory is accessed on this path.
2. **Existing flag:** read the current record’s flag byte at offset `12`. If bit `0x04` is set, terminate without examining the context condition.
3. **Context condition:** read the 32-bit field at context offset `12`.
4. **Conditional eligibility:** if that field is nonzero, obtain the eligibility-object address from the current record and inspect the 16-bit field at object offset `2`. If bit `0x0001` is clear, terminate.

When the context condition is zero, the eligibility-object address and its 16-bit field are not needed by this path. An otherwise invalid eligibility-object address therefore does not itself prevent this path from proceeding.

The context condition is evaluated again on each iteration that passes the flag check. Its initial value is not cached for the entire operation.

### Marking the current record

After passing the checks, the function sets bit `0x04` in the current record’s flag byte.

This write preserves the other seven bits of the byte as read for the update. The eligibility test, when enabled, precedes this write.

The flag update is separate from the earlier flag test. The supplied representation does not establish an atomic test-and-set operation, synchronization, or safety under concurrent mutation.

### Interpreting the successor index

After marking, the function reads the index-bearing slot at record offset `4` and interprets its low 16 bits as a signed value.

- Values with low 16 bits from `0x0000` through `0x7FFF` select indices `0` through `32767`.
- Values with low 16 bits from `0x8000` through `0xFFFF` are negative and terminate processing.
- The upper 16 bits of the slot do not affect this signed-index decision.

For a negative index, termination occurs **after** the current record has been marked. No link descriptor or successor counter is required on that path.

For a nonnegative index, the function obtains the link-descriptor address from current-record offset `16`, reads the array base from descriptor offset `16`, and selects the successor with a 40-byte stride.

The index-times-stride product ranges from zero through `1,310,680`, inclusive. That multiplication fits in a signed 32-bit integer. Whether adding it to the selected base produces a valid address is not checked.

### Updating the successor counter

The function next reads the successor’s byte at offset `13`, subtracts one, and stores the byte-sized result back to the same location.

Under the recovered 8-bit storage model, this is a modulo-256 decrement:

- `1` becomes `0`.
- `2` becomes `1`.
- `0` becomes `255`.

The function then examines the decremented result:

- If it is nonzero, processing terminates.
- If it is zero, processing repeats with that successor as the current record.

The successor counter update happens **before** the successor’s null check, flag check, context check, or eligibility check in the next iteration. Consequently, the counter can be changed even when the successor is already marked or would fail eligibility.

The initial record’s counter is not decremented merely because it is the initial input. It is decremented only if some traversal selects it as a successor.

## Side effects

The only explicit writes are:

1. Setting bit `0x04` at offset `12` of each current record that passes its checks.
2. Decrementing the byte at offset `13` of each selected successor.

The function does not explicitly allocate or release storage, call another function, produce output, or return an error status.

There is no rollback. Earlier writes remain when a later check terminates processing. For example, a marked current record remains marked when its successor’s counter becomes nonzero, and a decremented successor counter remains changed when that successor subsequently fails eligibility.

No disjointness requirement is enforced. If records, descriptors, the context, or eligibility objects overlap in memory, writes can affect later reads. The established behavior follows the ordering above; independent logical objects must not be assumed in an aliased layout.

In particular, the context condition is not explicitly written, but an overlapping flag or counter write could change bytes later read as part of that condition.

Cycles do not imply unbounded propagation in valid, stable memory. Each record that passes the checks acquires the flag before traversal, and a later iteration encountering that flag stops. A counter decrement may nevertheless occur on an already-marked record before that record’s flag is checked.

Invalid memory accesses have no evidenced local recovery path. Their external consequences are outside the established contract.

## Dependencies

No explicit callee appears in F-b4957f9fffe6. There are therefore no opaque dependency function IDs to list.

The function depends directly on caller-provided memory and the recovered layout described above. No dependency contract establishes:

- Storage ownership or release obligations.
- Valid array extents.
- Concurrency control.
- Preservation of overlapping state by external actors.
- Platform behavior for invalid, misaligned, or overflowing address accesses.

## Acceptance examples

These examples are conditional on the required storage being valid and on no external mutation during execution.

### Null initial record

Given a null initial record, the function returns without reading the context. The context address need not be dereferenceable for this particular path.

### Initial record already marked

Given a non-null initial record whose flag byte contains `0x04`, the function returns without reading the context condition, eligibility object, index, or link descriptor. It performs no write.

### Eligibility rejected

Given an unmarked initial record, a nonzero context condition, and an eligibility field whose bit `0x0001` is clear, the function returns without marking the record or following its successor.

### Eligibility bypassed

Given an unmarked initial record and a zero context condition, the eligibility object is not inspected. If the index is negative, the record is marked and processing stops.

### Negative index after marking

Given an eligible record with flag byte `0x80` and an index whose low 16 bits are `0xFFFF`, the flag becomes `0x84`. No successor is selected.

The same negative-index result applies regardless of the index slot’s upper 16 bits.

### Counter does not reach zero

Given a current record that passes all checks and selects a valid successor whose counter is `2`, the current record is marked and the successor counter becomes `1`. The function stops without checking the successor’s flag or eligibility.

### Counter reaches zero

Given a valid selected successor whose counter is `1`, that counter becomes `0`, and the function begins the current-record checks for the successor.

If the successor is already marked, the function stops at its flag check. The counter decrement is retained.

### Counter wraps

Given a selected successor whose counter is `0`, the stored result becomes `255` under the recovered byte model. Because the result is nonzero, processing stops; this is not a saturating decrement.

### Successor rejected after decrement

Given a successor whose counter changes from `1` to `0`, whose flag is clear, and whose eligibility test then fails, the successor remains unmarked with counter `0`. The preceding record remains marked.

## Uncertainties

The following remain **documented_only**:

- The application meaning of the flag, counter, context condition, and eligibility bit.
- Whether records are formally allocated as 40-byte objects or merely selected using that stride.
- The complete structure layouts and valid array bounds.
- Byte order, alignment guarantees, and the exact machine-level read sequence produced by compilation.
- The language-level signedness of plain character storage. The zero-result branch is unaffected by that signedness under the recovered byte model.
- Behavior when address arithmetic exceeds the addressable range.
- Behavior under concurrent mutation or invalid memory access.
- Any caller-side ownership, lifetime, or synchronization requirements beyond the accesses shown.

The function’s name and a possible dependency-propagation interpretation do not establish additional guarantees.

## Evidence

**Established static evidence:** the supplied function representation contains the ordered termination checks, conditional 16-bit eligibility test, flag-byte update, signed-16-bit index interpretation, 40-byte successor stride, successor-byte decrement, and repetition only when the decremented byte is zero. It contains no explicit calls, allocation, release, or error reporting.

**Inferred intent:** marking a record and propagating when a successor counter reaches zero resembles completion or dependency processing. This interpretation is not required for conformance.

**Runtime evidence:** none. No runtime tests have been performed.
