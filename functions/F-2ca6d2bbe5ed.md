<!-- Trusted provenance: F-2ca6d2bbe5ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ca6d2bbe5ed

## Purpose

F-2ca6d2bbe5ed operates on a mutable context and returns a 64-bit value assembled from two 32-bit fields of an associated object.

Two paths are visible:

- A distinguished context state selects an immediate path that resets that state, invokes context-processing dependencies, and returns fields from the object currently referenced by the context.
- Other states select an iterative path that examines indexed entries, attempts to obtain an acceptable object, and performs additional processing between scans. It returns the accepted object’s fields, or zero when its continuation condition becomes false without an accepted object.

The interpretation as selection or dispatch is inferred intent. The established behavior is the field access, dependency-call, branching, and return-value contract below. Nothing supplied establishes the application-level meaning of an entry, object, status, or returned field pair.

## Interface

### Input

One 32-bit address-like argument identifies a mutable context.

- Zero is accepted and immediately returns zero.
- A nonzero argument is used directly as a context address. The function performs no visible validity, size, alignment, or lifetime checks.

The following context-relative locations are used. Offsets are in bytes.

| Offset | Access | Established use |
|---|---|---|
| +4 | Read, 32 bits | First argument to several processing dependencies. |
| +8 | Read, 32 bits | Object reference used by the distinguished-state path. |
| +24 | Read/write, 32 bits | Compared with `0x7fffffff`; reset to zero on equality. |
| +28 | Address passed; contents later read as a 32-bit reference | Collection/count interface and base reference for eight-byte records. |
| +40 | Address passed | Indexed-entry interface. |
| +80 | Address passed | Shared dependency argument; internal representation unknown. |
| +92 | Address passed | Additional dependency argument; internal representation unknown. |
| +96 | Read, one byte | Controls whether an auxiliary dependency is invoked. |
| +98 | Read, one byte | Controls one entry-clearing operation after acceptance. |

These locations describe observed access, not a complete context layout. Regions passed by address may be larger than the spacing suggested by the table. Their extent and permitted mutation require dependency contracts.

### Output

The result is a 64-bit bit pattern:

- Bits 0–31 come from the selected object’s 32-bit field at offset +8.
- Bits 32–63 come from its 32-bit field at offset +12.

This is logical concatenation of two words, not an established in-memory 64-bit load. No signed numerical meaning is established.

A zero result can mean a null input, termination without selection, or a selected object whose two returned fields are both zero. The return value alone does not distinguish those cases.

### Preconditions

For defined completion, all context fields and referenced objects actually accessed must remain valid for those accesses. In particular:

- An object whose result is returned must support 32-bit reads at offsets +8 and +12.
- Indexed records accessed by the scan must support two 32-bit reads at offsets +0 and +4 within an eight-byte record.
- Dependencies must accept the supplied context addresses, subobject addresses, indices, and temporary-storage addresses.
- Dependencies must return normally for the subsequent behavior described here to occur.

The function does not enforce these preconditions or provide a visible error return for their violation.

## Behavior

### Null input

A zero input returns the 64-bit value zero. No context dereference or dependency call occurs.

### Distinguished-state path

For a nonzero input, the function first reads the 32-bit field at context offset +24.

If that field equals `0x7fffffff`, the following sequence applies:

1. Write zero to the same 32-bit field.
2. Read the object reference at context offset +8 and invoke F-d6ae0f30f5b4 with the context and that reference.
3. After that call, read the byte at context offset +96.
4. If the byte is zero, read the object reference at +8 again and invoke F-32ecbbf59f8c with the context and that reference.
5. Otherwise, invoke F-9918a047bc22 with the context. Its return value does not control this path.
6. Read the context’s object reference at +8 and obtain the low return word from that object’s offset +8.
7. Read the context’s object reference at +8 again and obtain the high return word from that object’s offset +12.
8. Return the assembled 64-bit result.

The reference is not captured once for the whole path. Changes made by dependencies can affect later arguments and the final object reads. In particular, the object supplied to the first dependency need not be the object from which the result is ultimately read.

The reset of the field at +24 precedes every dependency call on this path. This function does not visibly reset it again afterward.

### Iterative-path state

When the initial field at +24 differs from `0x7fffffff`, the function enters a loop.

Two retained 32-bit status words start at one:

- A wait-status word, subsequently updated by F-126347deb976.
- An auxiliary-status word, subsequently set to zero or updated by F-9918a047bc22.

These statuses are not automatically reset at the beginning of each pass. In particular, an auxiliary result that skips the wait operation leaves the previous wait-status word intact.

### Indexed scan

Each pass begins by invoking F-6e45b75056e5 with the address at context offset +28. Its result is interpreted as an unsigned 32-bit entry count.

The function scans indices from zero upward, stopping before that count unless an object is accepted earlier. The count is obtained once per pass; it is not refreshed after processing individual entries.

For each index:

1. Invoke F-a8416e4277b0 with an eight-byte temporary-storage address, the address at context offset +40, and the index.
2. Pass that temporary-storage address to F-d9f8aea71acf.
3. Interpret the returned 64-bit value as two 32-bit words.

If the low word is zero, processing of that index ends. The high word does not affect this decision, and no record lookup, candidate construction, candidate evaluation, or entry-clearing call follows for that index.

If the low word is nonzero:

1. Read the 32-bit record-base reference at context offset +28.
2. Locate the eight-byte record for the current index.
3. Read its first and second 32-bit words.
4. Invoke F-389707b2b41e with the context, the high word returned by F-d9f8aea71acf, and the two record words.

A zero result from F-389707b2b41e causes the indexed entry to be cleared through the operation described below. Scanning then continues.

A nonzero result is treated as a candidate object reference. The function reads the 32-bit field at context offset +4 and invokes F-9b7345170807 with that value, the candidate, and the address at context offset +80.

The evaluation result has three locally distinguished cases:

| Evaluation result | Local behavior |
|---|---|
| Zero | Accept the candidate and end the scan early. |
| Two | Clear the indexed entry, then continue scanning. |
| Any other nonzero 32-bit value | Continue scanning without a local clearing call for that entry. |

These distinctions are exact bit-pattern comparisons. No further interpretation of the evaluation result is established.

### Entry clearing

When clearing is required, the function:

1. Invokes F-a8416e4277b0 again for the same index, using a second eight-byte temporary.
2. Passes that temporary’s address and the value zero to F-bcb6380f47f7.

The second lookup is a new dependency call. The function does not simply reuse the first temporary.

Calling this sequence “clearing” describes the visible zero argument and its role in control flow. The actual mutation performed by F-bcb6380f47f7 remains dependent on its contract.

### Accepted candidate

When evaluation returns zero:

1. Invoke F-d6ae0f30f5b4 with the context and accepted candidate.
2. Read the byte at context offset +98 after that call.
3. If the byte is zero, perform the entry-clearing sequence for the accepted index.
4. Read the context’s 32-bit field at +4 and invoke F-14d88f6b12b8 with that value and the address at +80.
5. Read the context’s 32-bit field at +4 again and invoke F-b64dd98038aa.
6. Exit the loop regardless of that dependency’s result or the retained status words.
7. Read the accepted candidate’s 32-bit field at +8, then its 32-bit field at +12, and return their concatenation.

The result words are read after all these calls, not when the candidate is first produced or accepted. This function performs no visible validity check before those final reads.

### Pass without acceptance

If scanning finishes without an accepted candidate, the function does not return the last candidate that happened to be produced. It proceeds as a no-selection pass.

First, it reads the byte at context offset +96:

- If zero, set the auxiliary-status word to zero.
- If nonzero, invoke F-9918a047bc22 with the context and retain its 32-bit result as the auxiliary-status word.

If F-9918a047bc22 returns nonzero, skip the wait-status update for this pass. Otherwise—including when the flag byte was zero—invoke F-126347deb976 with:

- The freshly read 32-bit field at context offset +4.
- The address at context offset +80.
- The address at context offset +92.

Its 32-bit result replaces the wait-status word.

Next, invoke F-b64dd98038aa using a fresh read of the context field at +4. Its zero/nonzero result selects the continuation calculation:

- A zero result selects bitwise OR of the retained wait and auxiliary statuses.
- A nonzero result selects bitwise AND of those statuses.

A nonzero combined word begins another pass, including a new count query and a scan starting at index zero. A zero combined word ends the loop and returns the 64-bit value zero.

These are bitwise operations, not logical combinations of normalized Boolean values. Two nonzero statuses can have a zero AND result.

## Side effects

### Direct writes

The only explicit context-field write visible in this function is the 32-bit zero written at offset +24 on the distinguished-state path.

The function also provides two eight-byte temporary-storage areas to dependencies. Their contents and ownership semantics are not established.

### Dependency-mediated effects

Dependencies receive context references, subobject addresses, candidate references, and temporary-storage addresses. Their effects may influence subsequent reads and branch decisions.

The function does not establish that dependencies:

- Preserve context fields, referenced objects, or global state.
- Preserve collection size or record storage.
- Retain, transfer, allocate, or release object ownership.
- Leave an accepted candidate alive until its final field reads.
- Restrict mutations to the addresses explicitly passed.

No explicit allocation or release operation is visible in the supplied body. That does not prove the call chain performs none.

### Ordering and reentrancy

The dependency-call order and direct-access order described above are significant. Reads made after a call must not be replaced conceptually by snapshots from before that call.

No synchronization or reentrancy protection is visible. Safe concurrent mutation, callback reentry, and signal-handler interaction are not established.

## Dependencies

Only the caller-visible use of each dependency is established here.

| Opaque ID | Use by F-2ca6d2bbe5ed |
|---|---|
| F-6e45b75056e5 | Obtain the unsigned 32-bit scan count from the address at +28. |
| F-a8416e4277b0 | Resolve an indexed entry into eight-byte temporary storage. |
| F-d9f8aea71acf | Obtain a 64-bit entry value; low word gates processing, high word becomes a construction argument. |
| F-389707b2b41e | Produce a candidate reference or zero from the context and three 32-bit values. |
| F-9b7345170807 | Evaluate a candidate; results zero, two, and other nonzero values select different behavior. |
| F-bcb6380f47f7 | Receive a resolved entry temporary and zero during the clearing sequence. |
| F-d6ae0f30f5b4 | Process the distinguished-path object or an accepted candidate before later decisions and reads. |
| F-32ecbbf59f8c | Additional distinguished-path operation when the byte at +96 is zero. |
| F-9918a047bc22 | Distinguished-path operation or source of the iterative auxiliary status. |
| F-126347deb976 | Update the iterative wait status unless skipped by a nonzero auxiliary result. |
| F-14d88f6b12b8 | Accepted-candidate operation before the final mode query and result reads. |
| F-b64dd98038aa | Select bitwise OR versus AND for continuation; also called after acceptance. |

Dependency failures, exceptions, nonreturning behavior, and ownership contracts are documented_only.

## Acceptance examples

These are conditional acceptance cases derived from static inspection, not executed tests.

1. **Null context.** Input zero returns zero without invoking any dependency.

2. **Distinguished state, zero flag.** If +24 initially contains `0x7fffffff` and the byte at +96 is zero when read, the function writes zero to +24, calls F-d6ae0f30f5b4, then calls F-32ecbbf59f8c. Subject to valid final references, object words `0x11223344` and `0x55667788` yield `0x5566778811223344`.

3. **Dependency changes the distinguished object.** If F-d6ae0f30f5b4 replaces the context reference at +8, later argument selection and final reads use the reference present when each read occurs, not necessarily the original object.

4. **Inactive indexed entry.** A value from F-d9f8aea71acf with a zero low word skips that entry’s record reads and candidate calls, even if its high word is nonzero.

5. **Construction yields zero.** A nonzero low entry word followed by a zero construction result causes a second indexed lookup and a call to F-bcb6380f47f7 with zero. Candidate evaluation is not invoked.

6. **Evaluation returns two.** The entry-clearing sequence occurs, but the candidate is not accepted. Processing continues to later indices.

7. **Evaluation returns another nonzero value.** No local clearing call occurs for that entry, and scanning continues. That candidate is not returned merely because it was the last one examined.

8. **Accepted candidate with clearing disabled.** If evaluation returns zero and the byte at +98 is nonzero after F-d6ae0f30f5b4, the accepted entry is not locally cleared. F-14d88f6b12b8 and F-b64dd98038aa still run before the candidate’s result words are read.

9. **Bitwise continuation.** With no accepted candidate, retained statuses of two and one produce three under OR, so another pass occurs. The same statuses produce zero under AND, so the function returns zero.

10. **Skipped wait preserves its previous value.** On the first pass, a nonzero auxiliary result skips F-126347deb976 and leaves wait status equal to its initial value of one. If auxiliary status is two, OR mode continues, while AND mode terminates.

11. **Empty collection.** A zero count skips entry processing but still performs the no-acceptance processing and continuation decision. An empty collection does not alone guarantee immediate termination.

## Uncertainties

- **documented_only — Domain meanings:** Context fields, object types, status semantics, and the returned word pair have no established application-level names.
- **documented_only — Ownership:** No dependency contract establishes who owns entry temporaries, records, candidates, or the context.
- **documented_only — Lifetime:** Accepted-object survival through subsequent calls is a requirement for valid final reads, not a guarantee established by this function.
- **documented_only — Termination:** No iteration bound or progress guarantee is visible. Repeated nonzero continuation results can keep the function running indefinitely.
- **documented_only — Fault behavior:** Invalid addresses and dependency failures have no visible local recovery mechanism. Specific runtime consequences are not established.
- **documented_only — Arithmetic environment:** Counts and indices are unsigned 32-bit quantities, records are eight bytes apart, and address-like values are represented with 32-bit widths in the supplied material. Safe address formation and sufficient backing storage remain preconditions; no wider overflow check is visible.
- **documented_only — Concurrency:** The supplied material does not establish memory ordering or atomicity guarantees for concurrent access.
- **Inferred intent:** The paired entry interfaces appear to connect indexed eligibility with record data. Their semantic relationship is not independently established.

## Evidence

The evidence is static inspection of the supplied function body and the supplied opaque dependency mapping. It supports the explicit widths, offsets, comparisons, argument relationships, call ordering, status retention, bitwise continuation rule, and result construction described above.

The supplied material is not treated as authority for application intent or dependency behavior. No dependency implementation, independent type layout, runtime trace, or external behavioral specification was supplied. No runtime tests have been performed.
