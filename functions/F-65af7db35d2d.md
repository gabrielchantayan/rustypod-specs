<!-- Trusted provenance: F-65af7db35d2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-65af7db35d2d

## Purpose

F-65af7db35d2d conditionally appends one nonzero scalar value to an inline, eight-entry collection associated with a caller-supplied storage address.

The decision depends on two facts: whether the supplied value is zero and, for a nonzero value, whether the collection’s existing occupancy byte is less than eight. An accepted insertion increments that byte and stores the value in the slot selected by the byte’s original value. The occupancy update precedes the slot write.

This describes the behavior established by the supplied static evidence. Interpreting the storage as a collection is inferred intent; the observed behavior does not require the entries to be pointers, handles, or any other particular kind of value.

## Interface

### Inputs

The function receives two integer-shaped arguments:

- **Storage address:** used as the base address for fixed-offset memory accesses.
- **Candidate value:** tested against zero and, if accepted, written into an indexed slot.

There is no separate capacity argument, output buffer, allocator, error destination, or callback argument.

The source representation uses integer types for both inputs and for the slot write. Its four-byte slot spacing supports a 32-bit address/value model. This contract describes the resulting layout using 32-bit slots and an 8-bit occupancy byte. The excerpt does not independently establish the target ABI, integer widths, address-conversion rules, or byte order; those platform details remain documented_only uncertainties.

Under that model, every nonzero 32-bit candidate is eligible for insertion. No distinction is made between positive and negative signed interpretations. A candidate with its highest bit set is not rejected merely because it could represent a negative integer.

### Storage layout

Offsets are relative to the storage address.

| Region | Offset | Width | Observed role |
|---|---:|---:|---|
| Occupancy | `0x15` | 1 byte | Read as unsigned; updated on accepted insertion |
| Slot 0 | `0x18` | 4 bytes | Destination when original occupancy is 0 |
| Slot 1 | `0x1C` | 4 bytes | Destination when original occupancy is 1 |
| Slot 2 | `0x20` | 4 bytes | Destination when original occupancy is 2 |
| Slot 3 | `0x24` | 4 bytes | Destination when original occupancy is 3 |
| Slot 4 | `0x28` | 4 bytes | Destination when original occupancy is 4 |
| Slot 5 | `0x2C` | 4 bytes | Destination when original occupancy is 5 |
| Slot 6 | `0x30` | 4 bytes | Destination when original occupancy is 6 |
| Slot 7 | `0x34` | 4 bytes | Destination when original occupancy is 7 |

The final slot occupies offsets `0x34` through `0x37` under the stated width model. Bytes outside the occupancy byte and the selected slot are not explicitly accessed by the function. In particular, it does not inspect the two bytes between the occupancy byte and the first slot.

### Output and ownership

The function has no return value and provides no explicit success or rejection status. Its defined result is any change to the supplied storage.

No allocation, release, ownership transfer, reference-count adjustment, or dereference of the candidate value is visible. The function stores the candidate as a scalar. Any ownership meaning attached to that stored value belongs to an external contract not present here.

## Behavior

### Zero candidate

A zero candidate causes an immediate no-op with respect to the supplied storage:

- The occupancy byte is not read.
- No slot is read or written.
- The occupancy byte is not written.

Consequently, this branch does not require the supplied address to designate accessible collection storage. This is a statement about the function’s explicit accesses, not a general guarantee that an arbitrary integer address is valid under every platform ABI.

Zero is not a value that this function can append.

### Nonzero candidate

For a nonzero candidate, the function reads exactly the occupancy byte at offset `0x15` for its decision. It interprets that byte as unsigned.

An occupancy value below eight permits insertion. An occupancy value of eight or greater rejects insertion without any explicit write.

The unsigned interpretation matters for bytes with the high bit set: values from `0x80` through `0xFF` are outside the accepted range. They are not treated as negative counts and do not select slots before the slot area.

### Accepted insertion

When the original occupancy is between zero and seven inclusive, the function performs two source-ordered writes:

1. It replaces the occupancy byte with the original occupancy plus one.
2. It stores the candidate in the four-byte slot indexed by the original occupancy.

The slot selection uses the original occupancy, not the incremented byte and not a reread of that byte.

For example, original occupancy seven selects slot seven, then leaves occupancy eight. It does not select a ninth slot. The increment cannot overflow an 8-bit byte on this branch because the largest accepted original value is seven.

The indexed displacement ranges from zero through 28 bytes. With the initial slot offset included, the selected write starts between offsets `0x18` and `0x34`. This bounded arithmetic does not itself overflow a 32-bit integer. Whether adding those offsets to the supplied base address is valid depends on the address, platform arithmetic, and actual storage extent.

### Rejected insertion

For a nonzero candidate and an occupancy byte of at least eight:

- The occupancy byte has been read.
- The occupancy byte is left unchanged.
- No slot is accessed.
- No repair, clamping, clearing, or replacement occurs.

Values above eight are therefore treated like a full collection for this operation, even if they represent corrupted state. The function does not distinguish “full” from “invalid occupancy.”

### Existing contents and repeated calls

The function does not inspect existing slot contents, check for duplicates, search for an empty slot, or verify that entries before the occupancy boundary are initialized.

If the occupancy byte points to a slot containing an existing value, an accepted insertion overwrites that slot. The occupancy byte alone determines the destination.

Under serial execution, accessible storage, and no external state changes, successive accepted calls fill slots in ascending order. After occupancy reaches eight, further calls do not modify the collection. Zero-valued candidates do not consume capacity.

## Side effects

The only explicit memory effects are:

- One occupancy-byte read for a nonzero candidate.
- One occupancy-byte write followed by one slot write for an accepted candidate.

No direct global-state access, I/O, logging, allocation, or external call appears in the supplied body.

The update is not transactional. Its source-level order publishes the increased occupancy before storing the new entry. If execution is interrupted or a memory access fails after the occupancy write, the evidence provides no rollback mechanism. In particular, a failure during the slot write may leave an increased occupancy without a completed slot update; the precise result depends on the platform’s fault behavior.

No synchronization or atomic operation is visible. Callers requiring concurrent insertion or consistent observation must provide an external coordination contract. The source-level write order does not establish cross-thread visibility guarantees or hardware store ordering.

The function also supplies no internal validation for:

- A readable occupancy address when the candidate is nonzero.
- A writable occupancy byte when insertion is accepted.
- A writable selected slot.
- Required alignment for the integer-sized slot access.
- The actual allocation size behind the supplied address.

Failure of those preconditions has no function-defined error result.

## Dependencies

No opaque dependency IDs are needed: the supplied body contains no function calls.

Its behavior depends only on its arguments, the occupancy byte read from caller-supplied storage, and the execution platform’s memory and integer semantics.

There is no evidence of a hidden ownership helper or error handler. Equally, the absence of calls does not establish that the supplied storage is exclusively owned by this function or protected from external modification.

## Acceptance examples

These examples are conditional static expectations, not runtime test results. Unless stated otherwise, they assume the 8-bit/32-bit layout described above, valid accesses, serial execution, and no outside modification.

| Initial condition | Candidate | Required result |
|---|---|---|
| Occupancy 0 | Nonzero | Occupancy becomes 1; slot 0 receives the candidate. |
| Occupancy 3 | Nonzero | Occupancy becomes 4; slot 3 receives the candidate; other slots are not explicitly modified. |
| Occupancy 7 | Nonzero | Occupancy becomes 8; slot 7 receives the candidate. |
| Occupancy 8 | Nonzero | Occupancy is read; no write occurs. |
| Occupancy 255 | Nonzero | Occupancy is read as unsigned and remains 255; no slot is accessed. |
| Any occupancy | Zero | No supplied-storage read or write occurs. |

Additional boundary expectations:

- **High-bit candidate:** With occupancy two, a nonzero candidate whose highest bit is set is stored in slot two. Occupancy becomes three.
- **Duplicate candidate:** With available capacity, a candidate equal to an existing entry is still accepted.
- **Occupied destination:** With occupancy one and an arbitrary value already in slot one, that slot is overwritten by an accepted candidate.
- **Unreadable slot area, full occupancy:** If occupancy eight is readable but the slot area is inaccessible, a nonzero candidate causes no explicit slot access.
- **Zero candidate, inaccessible storage:** No access to the supplied storage is attempted by the function’s shown operations.
- **Failure during accepted insertion:** If the occupancy update succeeds but the subsequent slot write fails, the contract does not promise unchanged state, rollback, or a normal return.

For a sequence beginning at occupancy zero, eight nonzero candidates are stored in slots zero through seven in call order. A ninth nonzero candidate is rejected without replacing an earlier entry.

## Uncertainties

### Established static evidence

The supplied body establishes the zero-value gate, the unsigned occupancy read, the exclusive upper bound of eight, the fixed offsets, the original-count slot selection, and the occupancy-before-slot write order.

It also establishes that there are no explicit calls and no explicit error-reporting path.

### Inferred intent

The layout and transitions suggest a bounded append operation over an inline array. That interpretation is useful but does not establish an enclosing object type, a public API name, or the semantic type of the stored values.

### documented_only

The following are not resolved by the excerpt:

- Exact calling convention, pointer width, and integer-width guarantees.
- Target endianness and alignment requirements.
- Whether integer address arithmetic wraps or has other platform-specific behavior.
- The enclosing storage allocation and its lifetime.
- Whether candidates represent addresses, identifiers, or unrelated scalar data.
- Caller-side ownership obligations and synchronization.
- Machine-level visibility, fault handling, and partial-write behavior.
- Whether callers inspect storage afterward to distinguish acceptance from rejection.

None of these unknowns introduces an additional check or guarantee into the established behavior.

## Evidence

This contract is based solely on the supplied static function body. The evidence supports a single-byte unsigned capacity decision and, on acceptance, two ordered stores using a four-byte indexed slot stride.

No runtime tests, execution traces, caller implementations, platform declarations, or external documentation were provided or used. The acceptance examples are deductions from that static evidence rather than observed executions.
