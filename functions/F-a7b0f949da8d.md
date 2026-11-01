<!-- Trusted provenance: F-a7b0f949da8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a7b0f949da8d

## Purpose

F-a7b0f949da8d updates a forward reference in one memory record and conditionally updates a reverse reference in another. When the forward reference changes, it may also clear two bits in the first record’s flags word.

The established behavior is narrower than “make two records mutually reference each other.” An existing reverse reference is preserved whenever its referenced record already points forward to the proposed record—even if that referenced record is not the record supplied to this invocation. The function also does not remove a reverse reference from the formerly selected record.

**Inferred intent:** maintain a selected-record relationship while invalidating two state bits when the selection changes between different classification values, or when a selection is first established. Neither the record types nor the meanings of the classification and flag fields are established.

## Interface

The function accepts three arguments and has no explicit return value.

| Argument | Representation | Established use |
|---|---|---|
| First argument | Four-byte value | Not used by the supplied body. |
| Destination record | Address represented by a 32-bit integer in the supplied representation | Locates the forward-reference field and, conditionally, the flags word. |
| Proposed record | Address represented by a 32-bit integer in the supplied representation | Becomes the destination’s forward reference and supplies classification and reverse-reference fields. |

No memory pointed to by the first argument is accessed merely because that argument is present. The body does not establish that it is a context, owner, callback, or allocation handle.

### Relevant memory layout

Offsets are byte offsets from the applicable record address. Each accessed field occupies four bytes.

| Record role | Offset | Interpretation in this function |
|---|---:|---|
| Destination record | `0x10` | Forward-reference value |
| Destination record | `0x18` | Unsigned flags word |
| Proposed or previously selected record | `0x04` | Classification value, compared for equality |
| Proposed record | `0x08` | Reverse-reference value |
| Record named by a nonzero reverse reference | `0x10` | Forward-reference value, checked against the proposed record |

These are accessed fields, not complete structure definitions. No size, layout, or meaning is established for intervening bytes.

The forward and reverse references are compared as 32-bit values and used as addresses when dereferenced. Classification values are compared only for equality. The flags transformation operates on a 32-bit unsigned word.

### Preconditions

For ordinary completion, every access reached by the selected control-flow path must address valid memory with the required read or write permission:

- The destination’s field at `0x10` must be readable.
- If the forward reference changes, that field must also be writable.
- The proposed record’s field at `0x08` must be readable on every normally completing path.
- If reverse-reference replacement is required, the proposed record’s field at `0x08` must be writable.
- Classification fields, the destination’s flags word, and an existing reverse target’s forward-reference field must be accessible whenever their respective conditions require them.

There is no explicit null check for either record argument. A zero proposed-record value is not an accepted empty-selection operation established by this body: the function eventually attempts to access its `0x08` field.

Alignment requirements, address-space validity rules, and address-offset overflow behavior are **documented_only**. Validity of the accessed addresses is a precondition; this contract does not prescribe wrapping address arithmetic.

## Behavior

### 1. Read the previous forward reference

The function first reads the destination record’s four-byte field at `0x10`.

That captured value is the previous reference used for both the change decision and any subsequent classification comparison. The body does not reload the previous reference from the destination before comparing classifications.

### 2. Conditionally replace the forward reference

If the captured previous reference equals the proposed-record value:

- The destination’s forward-reference field is not written.
- No classification comparison occurs.
- The flags-update operation is skipped.
- Processing continues to the reverse-reference check.

If the two values differ, the function writes the proposed-record value to the destination’s field at `0x10`.

This write occurs **before** any classification reads used to decide whether to clear flags. It also occurs before all reverse-reference processing.

### 3. Conditionally clear two flag bits

Following a forward-reference change, the function clears flag bits if either:

1. The captured previous reference is zero; or
2. The previous reference is nonzero and its classification at `0x04` differs from the proposed record’s classification at `0x04`.

When the previous reference is zero, neither classification field is read for this decision.

When the previous reference is nonzero, both classification fields are read. The supplied expression does not establish which classification operand is read first. No ordering guarantee between those two reads should be imposed by this contract.

If clearing is required, the function reads the destination’s current flags word at `0x18`, masks it with `0xFFFFFCFF`, and writes the result back to the same field.

This operation:

- Clears bit 8, whose mask is `0x00000100`.
- Clears bit 9, whose mask is `0x00000200`.
- Preserves every other bit of the flags word as read at this step.

The write still occurs if those two bits were already clear. This is a read-modify-write operation, not an established atomic operation.

If the forward reference changes between records with equal classifications, the flags word is neither read nor written by this phase.

### 4. Inspect the proposed record’s reverse reference

After the forward-reference and optional flags operations, the function reads the proposed record’s field at `0x08`.

If that value is zero, the function proceeds directly to reverse-reference replacement. It does not dereference a zero reverse-reference value.

If it is nonzero, the body contains another read of the proposed record’s field at `0x08`, then reads the field at `0x10` of the record named by that second value.

If the resulting forward-reference value equals the proposed-record value, the function returns without writing the proposed record’s reverse reference.

This preservation condition does **not** require the existing reverse-reference value to equal the destination-record argument. A reverse reference to some other record is retained when that other record already points forward to the proposed record.

The two source-level reads of the reverse-reference field must not be treated as a documented coherent snapshot under concurrent mutation. Machine-level load count and synchronization guarantees are not established.

### 5. Otherwise replace the reverse reference

If the reverse reference was zero, or its referenced record did not point forward to the proposed record, the function writes the destination-record value to the proposed record’s field at `0x08`.

The function then returns.

No further validation or repair is shown.

## Side effects

### Direct writes

The body can perform these direct writes, in this order:

1. Destination `0x10`: proposed-record value, only when it differs from the captured previous value.
2. Destination `0x18`: masked flags, only when the forward reference changed and the classification condition requires clearing.
3. Proposed record `0x08`: destination-record value, only when the existing reverse relationship does not satisfy the preservation check.

Any of these writes may be absent. Earlier writes are not rolled back if a later access fails.

### Relationship state

The function does not explicitly clear or repair:

- The previously selected record’s reverse reference.
- A forward reference belonging to the record formerly named by the proposed record’s reverse reference.
- Other records participating in the same relationship.

Consequently, normal completion does not guarantee a globally unique or fully reciprocal relationship.

### Aliasing

The body establishes no disjointness requirement between the supplied records or records reached through stored references. Writes must therefore be understood in their stated order.

For example, if the proposed record and destination record are the same record, a changed forward-reference write makes that record’s `0x10` field equal its own address before reverse-reference validation. A reverse reference back to that same record can consequently satisfy the preservation check.

More generally, overlapping fields may cause an earlier write to affect a later read. The function uses the captured previous reference for its change and classification decisions, but later field reads observe memory after the preceding writes. A description that evaluates all conditions from entry-state values would not match this behavior.

### Ownership, allocation, and errors

No allocation, deallocation, ownership transfer, external call, explicit I/O, or explicit global access is shown.

Stored references do not establish ownership or lifetime extension. The function neither acquires nor releases referenced objects.

There is no returned success indicator and no explicit error-reporting path. Invalid or inaccessible memory can prevent completion; the resulting runtime mechanism is not specified by the supplied material.

## Dependencies

No dependency function is called by F-a7b0f949da8d. There are therefore no opaque callee IDs to enumerate.

Execution depends on the surrounding environment providing valid memory and suitable access permissions. No locking, reference counting, callback behavior, or memory-management service is established.

## Acceptance examples

These examples are conditional on valid, accessible, stable memory and non-overlapping fields unless otherwise stated. They describe expected behavior, not executed tests.

### Unchanged forward reference, missing reverse reference

Initially:

- The destination already points forward to the proposed record.
- The proposed record’s reverse reference is zero.

Expected:

- No forward-reference write.
- No classification reads or flags access.
- The proposed record’s reverse reference is set to the destination.

An unchanged forward reference does not skip reverse-reference maintenance.

### First selection

Initially:

- The destination’s forward reference is zero.
- The proposed record’s reverse reference is zero.
- The destination flags are `0xA5A5FFFF`.

Expected:

- The destination’s forward reference is set to the proposed record.
- Neither classification field is read.
- The flags become `0xA5A5FCFF`.
- The proposed record’s reverse reference is set to the destination.

### Replacement with matching classification

Initially:

- The destination points to a different, nonzero record.
- The previous and proposed records have equal classification values.
- The proposed record’s reverse reference is zero.

Expected:

- The destination’s forward reference changes to the proposed record.
- Both classification fields are read.
- The flags word is untouched.
- The proposed record’s reverse reference is set to the destination.
- No explicit write is made to the previous record.

### Replacement with different classification

Under the preceding conditions, but with different classification values:

- The forward reference changes first.
- Flag bits 8 and 9 are cleared.
- The remaining flags are preserved.
- Reverse-reference processing follows.

### Existing reciprocal relationship elsewhere

Initially:

- The proposed record’s reverse reference names another record, distinct from the destination.
- That other record’s forward reference equals the proposed-record value.

Expected:

- The proposed record’s reverse reference remains unchanged.
- Destination forward-reference and flags processing still occurs as applicable.

This case demonstrates that the function does not always make the destination the proposed record’s reverse target.

### Existing nonreciprocal reverse reference

Initially:

- The proposed record’s reverse reference is nonzero.
- Its referenced record’s forward reference does not equal the proposed-record value.

Expected:

- The proposed record’s reverse reference is overwritten with the destination.
- No explicit write repairs the formerly referenced record.

### Zero proposed-record value

A zero proposed-record value is not established as a successful clearing request. Depending on earlier state, the destination’s forward reference and flags may already have been written before an invalid proposed-record access occurs.

No rollback or successful empty-state result is specified.

## Uncertainties

The following remain **documented_only**:

- Semantic names and meanings of the records, classifications, and two cleared flag bits.
- Calling convention and ABI details beyond the supplied argument and return representation.
- Exact machine-level load counts, access widths after compilation, and atomicity.
- Endianness, alignment guarantees, and behavior of invalid address arithmetic.
- Thread-safety and behavior under concurrent mutation.
- Whether callers enforce non-aliasing, valid lifetimes, or stronger relationship invariants.
- The runtime outcome of invalid memory access.
- Whether the preserved reverse relationship is intentional policy or merely the behavior of this implementation.

The relational interpretation is inferred intent. The conditional reads, writes, comparisons, and mask operation are established static evidence.

## Evidence

This contract is based solely on static inspection of the supplied body for F-a7b0f949da8d.

The supplied body establishes:

- One unused four-byte argument and two address-valued arguments.
- Four-byte field accesses at the documented offsets.
- Capture of the previous forward reference before any conditional update.
- A forward-reference write before classification-dependent flag processing.
- Conditional clearing with the 32-bit mask `0xFFFFFCFF`.
- Reverse-reference preservation based on the referenced record’s forward value, not on its identity as the destination.
- No explicit dependency calls, allocation, release, error result, or rollback.

No runtime tests have been performed. The acceptance examples are conditional deductions from this static evidence, not observed execution results.
