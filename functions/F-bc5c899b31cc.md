<!-- Trusted provenance: F-bc5c899b31cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bc5c899b31cc

## Purpose

F-bc5c899b31cc replaces two 32-bit values in an object, obtains a classification result from a dependency, and conditionally links the object into an indexed table entry.

Established behavior comprises ordered dependency calls, object-field reads and writes, two evaluations of an external pointer, a later read through the initially captured pointer, and conditional table and link updates. Interpreting the table entries as list heads and the object fields as membership links is inferred intent.

The function provides no explicit result or error status. It performs no visible table-capacity check, address validation, or rollback.

## Interface

### Arguments and result

| Input | Width | Established role |
|---|---:|---|
| Object address representation | 32 bits | Used to address the object and stored into table and link fields. |
| First replacement value | 32 bits | Passed to a dependency, then stored at object offset +28. |
| Second replacement value | 32 bits | Passed to a dependency, then stored at object offset +24. |

The replacement values are opaque bit patterns. Their interpretation as references, identifiers, or handles is not established.

The object address is represented by a signed 32-bit quantity in the supplied representation. Conversion of computed integer addresses into valid memory accesses depends on the address model.

There is no explicit return value.

### Object layout

Offsets are measured in bytes from the supplied object address.

| Offset | Access width | Established use |
|---|---:|---|
| +8 | 32 bits | Stores the index-producing dependency’s result incremented by one; subsequently read to derive a candidate index. |
| +12 | 32 bits | Written with zero during insertion; the same offset in another object may receive the supplied object address. |
| +16 | 32 bits | Receives the previous table-entry value; subsequently read as a possible object address. |
| +20 | 8 bits | Bit 0 gates insertion and is set during insertion. |
| +24 | 32 bits | Tested before replacement, then receives the second replacement value. |
| +28 | 32 bits | Tested before replacement, then receives the first replacement value. |

Access to the replacement fields requires valid storage through byte +31. No broader object layout is established.

### External pointer and table

An external pointer identifies storage containing a 32-bit integer. The function evaluates that pointer twice initially:

1. It captures the pointer’s value, retaining the resulting cell address.
2. It evaluates the external pointer again and reads the 32-bit value at the address obtained by this second evaluation.

The second evaluation supplies the first argument to F-03b9616af95f. It does not explicitly use the captured pointer.

Later, the function reads a 32-bit value through the captured pointer and uses that value as the table-base integer.

The two initial pointer evaluations identify the same cell only under a pointer-stability assumption. The later read uses the captured address even if the external pointer subsequently designates a different cell.

Table entries are four bytes wide. An entry value is treated as either zero or another object’s address.

### Preconditions

For completion along the selected path:

- Every accessed object field must remain accessible.
- The cell selected by the second external-pointer evaluation must support the initial 32-bit read.
- The captured cell must support the later 32-bit read.
- Each invoked dependency must return normally for subsequent operations to occur.
- An insertion requires a valid computed table-entry access.
- A nonzero value used for the final link update must identify storage writable at offset +12.
- Integer-address calculations and conversions must produce valid accesses under the execution environment.

These are environmental obligations, not checks performed by this function. The function establishes neither storage lifetime nor dependency preservation guarantees.

## Behavior

### Initial pointer evaluations and dependency calls

The function first captures the current external pointer value.

It then evaluates the external pointer again, reads the 32-bit contents of the cell selected by that evaluation, and calls F-03b9616af95f with:

1. That 32-bit cell value.
2. The supplied object address representation.

The captured address is retained for a later read. The initially supplied cell value is not retained as the table base for insertion.

After F-03b9616af95f returns, the function calls F-2609ef4ae406 twice:

1. With the first replacement value.
2. With the second replacement value.

Both calls are unconditional. Zero arguments, equal replacement arguments, and equality with existing object fields do not suppress them.

These three calls occur before the visible tests of the existing associated fields.

### Ordered tests of existing fields

The function reads the 32-bit field at +28. If it is nonzero, F-029930293dcb is called with no explicit arguments.

After that conditional call completes, the function reads the 32-bit field at +24. If it is nonzero, F-029930293dcb is called again, also with no explicit arguments.

Consequences of this ordering:

- The tests observe field values after the preceding dependencies have returned, not necessarily values from function entry.
- The first conditional call may affect the value observed by the second test.
- Neither old field value nor the object address is explicitly passed to F-029930293dcb.
- The dependency’s visible placement does not establish that it releases either old value.

There are zero, one, or two such calls.

### Replacement writes and index derivation

The function writes:

1. The first replacement value to +28.
2. The second replacement value to +24.

It then calls F-fb6e215aa4fd with the second replacement value followed by the first replacement value.

The returned signed 32-bit result is incremented by one and stored at +8. The function next:

1. Reads the 32-bit table-base integer through the previously captured cell address.
2. Reads the signed 32-bit field at +8.
3. Subtracts one.
4. Assigns the subtraction result to an unsigned 32-bit candidate index.

The later captured-cell read occurs before the insertion decision, including on paths that skip insertion.

When both signed arithmetic operations are representable and there is no intervening mutation of +8, the candidate index represents the original dependency result after conversion to unsigned 32 bits. Signed boundary cases remain unresolved; the unsigned storage of the candidate does not retroactively make the preceding signed increment or subtraction unsigned.

### Insertion eligibility

Eligibility uses a signed 32-bit interpretation of the unsigned candidate index.

- If that interpretation is negative, insertion is skipped without reading the flag byte for eligibility.
- Otherwise, the function reads the byte at +20 and tests bit 0.
- Insertion proceeds only if that bit is clear.

The negative-candidate path is excluded because the unsigned candidate itself remains the nonzero value controlling the final decision. Thus the exclusion covers every candidate interpreted as negative, not only the representation of −1.

There is no visible upper-bound comparison for a nonnegative candidate. An already-set flag bit suppresses insertion without establishing that any existing membership is correct.

Skipping insertion leaves earlier dependency effects, replacement writes, and the +8 write in place.

### Table-entry address arithmetic

The selected entry’s integer address is computed from:

- The later 32-bit table-base integer.
- Four times the unsigned 32-bit candidate index.

Multiplication is unsigned 32-bit arithmetic. The table-base operand is converted to unsigned 32 bits for the addition, which is also unsigned 32-bit arithmetic. Both operations therefore reduce modulo \(2^{32}\).

This is established intermediate arithmetic, not an unresolved overflow policy. A sufficiently large nonnegative candidate can wrap its byte displacement and select the same integer address as a smaller candidate.

Whether the resulting integer can be converted into a valid, suitably accessible address is a separate, unresolved address-model question. No capacity check or overflow-rejection branch is visible.

### Ordered insertion accesses

When eligible, the function performs these accesses in order:

1. Writes a 32-bit zero to the object at +12.
2. Reads the selected 32-bit table entry.
3. Writes the entry value to the object at +16.
4. Writes the supplied object address into the selected table entry.
5. Reads the byte at +20, sets bit 0, and writes the resulting byte back.
6. Reads the 32-bit field at +16 and tests whether it is zero.
7. If nonzero, reads +16 again to obtain an address and writes the supplied object address at offset +12 from that address.

The flag update uses a fresh byte read after the table-entry write. It preserves the other seven bits from that read, not necessarily from the earlier eligibility read or from function entry.

The +16 read used for the zero test and the subsequent +16 read used to form the final destination are distinct visible accesses. They must not be described as one immutable snapshot when considering aliasing or external mutation.

Under valid, nonoverlapping list-storage assumptions, this sequence makes the supplied object the new head, sets its predecessor to zero, retains the former head as its successor, and writes the supplied object into the former head’s predecessor field. These structural labels are inferred from the access pattern.

### Completion and errors

The function returns after insertion or after skipping insertion. No explicit status distinguishes a negative candidate, a set membership bit, an empty entry, or an existing head.

There is no visible recovery path for invalid memory access or abnormal dependency termination. Earlier effects are not compensated if execution stops before completion.

## Side effects

### Direct writes

A path reaching the eligibility decision has performed explicit writes to:

- +28: first replacement value.
- +24: second replacement value.
- +8: incremented dependency result.

Insertion additionally writes:

- Zero to +12.
- The previous entry value to +16.
- The object address to the selected table entry.
- The flag byte with bit 0 set.
- The object address to another object’s +12 when the conditional link update is taken.

On a skipped-insertion path, this function performs no explicit insertion writes. Dependencies may nevertheless modify these locations.

### Ownership and lifetime

No explicit allocation or deallocation is visible. The function copies 32-bit values and address representations.

The call order may suggest acquiring replacement references before handling previous references, but that interpretation is not established:

- F-2609ef4ae406 is not known to retain anything.
- F-029930293dcb is not known to release anything.
- F-03b9616af95f is not known to unlink the object.
- No dependency is assumed to preserve globals, object contents, or allocation lifetime.

No special handling exists for replacement values equal to each other or equal to existing values.

### Aliasing and synchronization

No nonaliasing check or synchronization mechanism is visible.

If the object, pointer cells, table entry, or linked object overlap, an earlier write can affect a later read. The ordered accesses define the local behavior; ordinary list postconditions require additional nonaliasing assumptions.

The table entry is published before the flag update and before the final back-link update. Atomic insertion, thread safety, and a consistent intermediate list view are not established.

## Dependencies

| Opaque ID | Invocation | Established result use |
|---|---|---|
| F-03b9616af95f | Once, with the cell value obtained through the second external-pointer evaluation and the object address. | No result consumed. |
| F-2609ef4ae406 | Twice, with the first replacement and then the second replacement. | No result consumed. |
| F-029930293dcb | Zero to two times, following the ordered nonzero tests at +28 and +24; no explicit arguments. | No result consumed. |
| F-fb6e215aa4fd | Once after replacement writes, with the second replacement followed by the first. | Signed 32-bit result feeds the +8 write and subsequent candidate derivation. |

Dependency internals, hidden inputs, ownership rules, failure conventions, and mutation effects are **documented_only** unknowns.

In particular, dependency effects can change the second old-field test, the contents of the captured cell, the later flag value, and the external pointer’s current value. Changing the external pointer does not change the address already captured.

## Acceptance examples

These examples are conditional consequences of static evidence, not executed tests. Unless specified otherwise, storage is valid and nonoverlapping, dependencies return normally without unmentioned mutations, the external pointer is initially stable, and signed arithmetic is representable.

### Eligible insertion into an empty entry

If F-fb6e215aa4fd returns 3, flag bit 0 is clear, and the selected entry contains zero:

- +8 receives 4.
- +28 and +24 receive the first and second replacements.
- +12 and +16 receive zero.
- Entry 3 receives the object address.
- Flag bit 0 becomes set; the other bits retain their values from the update-time byte read.
- No final back-link write occurs.

### Insertion before an existing head

If the result is 2, flag bit 0 is clear, and entry 2 contains a distinct valid object address, the supplied object replaces that entry. Its +16 receives the former head address, its +12 becomes zero, and the former head’s +12 receives the supplied object address.

### Set membership bit

If the candidate is nonnegative and bit 0 is set when eligibility is checked, insertion is skipped. The replacement fields and +8 have already been written. There is no explicit insertion write, regardless of whether the object is actually linked into the selected entry.

### Negative result

If the result is −1, +8 receives zero and the derived unsigned candidate represents −1 under the signed eligibility interpretation. Insertion is skipped without an eligibility read of +20.

If the result is −2, +8 receives −1 and insertion is likewise skipped. Neither example crosses a signed arithmetic boundary.

### Ordered old-field observations

If +28 is nonzero at its test, the first F-029930293dcb call occurs. If that call changes +24 to zero, the second call is skipped even when +24 was nonzero at entry.

Zero replacement arguments do not suppress the unconditional F-2609ef4ae406 calls.

### Distinct initial pointer evaluations

Conditionally, if the first external-pointer evaluation selects cell A and the second selects cell B, F-03b9616af95f receives B’s contents. The later table-base read uses A.

This example specifies which access supplies each value. It does not establish a synchronized mechanism for changing the pointer between those evaluations.

### Pointer redirection versus captured-cell mutation

If a dependency redirects the external pointer from cell A to cell B after A has been captured, the later read still uses A. If a dependency changes A’s contents, insertion uses A’s updated contents. Redirection alone does not substitute B’s contents.

### Unsigned displacement wrap

If the dependency returns 1,073,741,825, both the increment and subsequent subtraction are representable as signed 32-bit values. The candidate is nonnegative, but multiplying its unsigned value by four reduces to 4 modulo \(2^{32}\).

With a table-base integer of 4096, the computed entry address is therefore 4100. If the flag permits insertion and that address is valid, the function accesses it without rejecting the large candidate. A logical table-capacity interpretation does not override the established arithmetic.

## Uncertainties

- **documented_only — Signed arithmetic boundaries:** The result increment and stored-field decrement are signed operations. Boundary overflow behavior is not established. Integer expressions used for object-relative addressing also require a compatible address model.
- **Established — Unsigned entry arithmetic:** Candidate multiplication by four and addition to the table-base integer are unsigned 32-bit operations modulo \(2^{32}\). This is distinct from signed-overflow uncertainty.
- **documented_only — Address interpretation:** Validity of integer-to-address conversion, alignment, byte order, and accessibility are not established.
- **documented_only — Pointer stability:** The source evaluates the external pointer twice initially. Equality of their selected cells requires a stability assumption; safe concurrent mutation is not established.
- **documented_only — Dependency contracts:** Preservation, ownership, lifetime, hidden state, and failure behavior remain unknown.
- **documented_only — Capacity and structure:** No table size, cycle check, self-link check, duplicate-membership check, or stale-pointer check is supplied.
- **documented_only — Flag lifecycle:** Only local testing and setting of bit 0 are established. Clearing and wider membership invariants are unknown.
- **Inferred intent:** +12 and +16 resemble predecessor and successor links; +8 resembles a one-based classification encoding. These interpretations do not establish external invariants.
- **documented_only — Concurrency:** No locking, atomicity, or memory-ordering contract is available.

## Evidence

This contract is based solely on static inspection of the supplied function representation and opaque dependency mapping.

Established evidence includes the two initial external-pointer evaluations, later dereference of the captured address, dependency call order and explicit arguments, field widths and offsets, signed increment and decrement, unsigned candidate storage, modulo-\(2^{32}\) table-address arithmetic, insertion gating, and ordered memory accesses.

No runtime tests have been performed. No dependency implementations, caller contracts, table declaration, or independent machine-level boundary evidence were supplied. Acceptance examples are conditional specifications, not observed executions.
