<!-- Trusted provenance: F-16d0badfba73; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-16d0badfba73

## Purpose

F-16d0badfba73 requests a container header, validates its signature, acquires an associated object, writes metadata into that object, optionally updates accounting, and processes subordinate records. Records whose type equals 3 invoke a dedicated processing dependency; other types use a calculated content skip. Both paths subsequently request a separate trailing-byte skip.

Processing is controlled by a 32-bit unsigned counter compared against the container header’s count field **again at every loop condition**. The count is not established to remain equal to its initially read value.

The operation is nontransactional. Object acquisition, metadata writes, accounting, and record processing can precede a returned error. No local rollback or release operation is visible.

The apparent intent is selective loading from a binary container. That purpose is inferred; the format identity and the meanings of most fields remain unresolved.

## Interface

### Arguments and result

| Item | Established behavior |
|---|---|
| First argument | A 32-bit input-context address, passed to input-related dependencies and used for direct context-relative reads. |
| Second argument | An opaque 32-bit value forwarded as the first argument to F-9e5d6dacabf4; not directly dereferenced here. |
| Result | A 32-bit status. Zero indicates completion of the visible processing sequence. Nonzero read, skip, and content-processing statuses are returned unchanged. |

The function also generates these statuses:

| Status | Local condition |
|---|---|
| −208 | A container or record signature differs from its expected value at comparison time. |
| −108 | F-9e5d6dacabf4 returns zero. |
| −50 | F-3080127849ac returns zero. |

These values do not establish conventional error names. Dependencies could independently return identical values.

### Storage and access requirements

Defined operation requires:

- A context valid for the dependencies and direct reads.
- A readable byte at context offset 13 after each successful fixed-size read.
- A readable 32-bit accounting flag at a context-relative displacement supplied by unresolved external storage.
- A nonzero object result supporting the metadata writes below and, when applicable, access through the object address plus 32.
- If accounting is enabled, a state result supporting 32-bit accesses at offsets 3120 and 3124.
- Valid dependency behavior for supplied destinations, handles, and addresses.

No local null, size, alignment, extent-consistency, or arithmetic-overflow validation protects these accesses. Invalid storage is not translated into a defined local error status.

### Container header

The first read requests 76 bytes into temporary storage. Offsets below are relative to that storage:

| Offset | Width | Local use |
|---|---:|---|
| 0 | 4 bytes | Container signature |
| 4 | 4 bytes | Header extent for the initial skip |
| 12 | 4 bytes | Unsigned loop-bound count, reread at each loop condition |
| 16 | 4 bytes | Second argument to object acquisition |
| 20 | 4 bytes | Metadata for object offset 24 |
| 24 | 4 bytes | Metadata for object offset 28; unsigned accounting amount |
| 28 | 2 bytes | Metadata for object offset 16 |
| 30 | 2 bytes | Metadata for object offset 18 |
| 32 | 2 bytes | Metadata for object offset 20 |
| 34 | 2 bytes | Metadata for object offset 22 |
| 36 | 4 bytes | Third argument to object acquisition |

Remaining bytes have no established direct use here, but normalization receives the header address and may use them.

### Record header

Each attempted record read requests 24 bytes into a separate temporary buffer:

| Offset | Width | Local use |
|---|---:|---|
| 0 | 4 bytes | Record signature |
| 4 | 4 bytes | Record-header extent |
| 8 | 4 bytes | Record extent used on the non-type-3 path |
| 12 | 2 bytes | Type, compared with 3 |
| 15 | 1 byte | Unsigned trailing-byte count |

### Observation-time rule

Header fields are not immutable snapshots. Dependencies receive temporary header addresses, and their contracts do not establish preservation of those buffers or other aliased state.

Each described field use therefore refers to the value observed **when that use occurs**. In particular:

- Acquisition arguments are taken after the initial skip.
- Metadata is read after acquisition returns.
- Accounting uses the amount available after its state-supplying dependency returns.
- The loop count is read after preparation and again following each completed record.
- Record type and content-skip fields are used after the header-extension skip.
- The trailing count is read separately after content handling.

Earlier validation does not cause a later signature recheck or freeze the remaining fields.

## Behavior

### Container recognition

F-c02571304c5c is first called with the input context, the temporary container-header destination, and size 76.

A nonzero status returns immediately. No local normalization, signature comparison, initial skip, or object acquisition follows.

After a zero status, the context byte at offset 13 is read. If nonzero, F-c70776c346eb receives the container-header address. Its return value is ignored.

The 32-bit container signature is then compared with the expected value obtained from unresolved external storage. A mismatch returns −208. Normalization, when enabled, therefore precedes signature validation.

On a match, F-22914e190356 receives the context and the currently observed header extent minus 76. This call occurs even when the difference is zero. A nonzero result returns unchanged.

### Object acquisition and metadata

After a successful initial skip, F-9e5d6dacabf4 receives:

1. The function’s second argument.
2. The container field at offset 16.
3. The container field at offset 36.

A zero result returns −108. A nonzero result is treated as the object address.

The following writes occur in order, using container fields observed after acquisition:

| Order | Container offset | Object offset | Width |
|---:|---:|---:|---:|
| 1 | 20 | 24 | 4 bytes |
| 2 | 24 | 28 | 4 bytes |
| 3 | 28 | 16 | 2 bytes |
| 4 | 30 | 18 | 2 bytes |
| 5 | 32 | 20 | 2 bytes |
| 6 | 34 | 22 | 2 bytes |

No dependency call or local validation intervenes between these writes. Aliasing, if possible under the dependency contracts, is not excluded by local checks.

### Optional accounting

After metadata writes, a 32-bit flag is read from the context plus an unresolved displacement. A zero flag bypasses accounting. Its value is not cached from before object acquisition.

For a nonzero flag, F-a9b747a47f9c is called without arguments. Its result is used as a state address without a local zero check.

The function:

- Reads the unsigned low accounting word at offset 3120.
- Adds the unsigned container amount at offset 24 and writes the low 32 bits back.
- Updates the high word at offset 3124 by adding the carry associated with adding that amount to the saved original low word.

The amount participates in both the low-word calculation and the carry calculation. A single stable amount across those accesses is not guaranteed against aliasing or concurrent mutation.

With a stable amount and no interfering accesses, this is addition of a 32-bit unsigned amount to a two-word 64-bit unsigned counter, wrapping modulo 2⁶⁴. The low-word write precedes the high-word update; the operation is not atomic.

### Processing preparation

F-3080127849ac receives the object address after any accounting update.

A zero result returns −50. Earlier writes and accounting have already occurred.

A nonzero result is retained as an opaque processing value. Its representation as a pointer, handle, or other identifier is unresolved. Preparation is not bypassed merely because a previously observed record count was zero.

### Record-loop admission and completion

The unsigned 32-bit counter starts at zero. Before each attempted record read, it is compared with the **then-current** unsigned count at container offset 12.

If the counter is at least that observed count, the function returns zero without attempting another record read. Consequently, successful completion does not establish that the number of completed records equals the count originally supplied by the initial read.

If the counter is less than the observed count, F-c02571304c5c requests 24 bytes into the record-header buffer. A nonzero status ends processing and is returned unchanged.

The count is not checked again during that admitted iteration. A change during record handling affects the next loop condition, not admission of the current record.

### Record recognition and header extension

After each successful record-header read, the context byte at offset 13 is read anew. If nonzero, F-58734d45a018 receives the record-header address. Its result is ignored.

The current record signature is compared with its expected value from unresolved external storage. A mismatch returns −208.

A matching signature is followed by F-22914e190356 with the context and the currently observed record-header extent minus 24. A nonzero result returns unchanged.

Only after this skip succeeds is the type field evaluated. The header-extension skip is not established to preserve that field or the length fields.

### Content handling and trailing skip

If the type observed at dispatch equals 3, F-ca9ad2c08e4d receives, in order:

1. Input context.
2. Literal 2.
3. Literal 3.
4. The retained nonzero preparation result.
5. The acquired object address plus 32 bytes.

The function does not calculate a general content length for this path. Content consumption is delegated to F-ca9ad2c08e4d.

For every other observed type, F-22914e190356 receives the context and the 32-bit unsigned result of:

**record extent − trailing count − record-header extent**

These fields are evaluated for this call after the successful header-extension skip. The trailing count is zero-extended from eight bits.

A nonzero content-processing or content-skip result returns unchanged.

After either content path succeeds, the trailing-count byte is read again. F-22914e190356 receives that separately observed value. The earlier content-length calculation does not determine this later argument. A zero trailing count still produces a call.

A nonzero trailing-skip result returns unchanged. Only a zero result permits the counter to increase by one and processing to return to the loop condition.

### Arithmetic boundaries

No local rejection occurs for extents smaller than their fixed headers or for inconsistent record lengths.

The non-type-3 content calculation uses unsigned 32-bit arithmetic and wraps modulo 2³². An underflow can therefore produce a large skip argument.

The two fixed-header subtractions are represented with signed 32-bit operands. They are not checked mathematical subtraction; behavior at signed-overflow extremes is not independently established.

The loop counter cannot wrap on a normally completing iteration: admission requires it to be below an unsigned 32-bit bound, so an admitted counter is at most 4,294,967,294. Its increment can reach 4,294,967,295, after which no unsigned 32-bit count can admit another iteration.

## Side effects

### Input and temporary storage

Reads, normalization, skips, and type-3 processing occur in the order described above. Exact input advancement, short-read behavior, and position after failure belong to the dependency contracts.

The context’s normalization byte is reread after every successful fixed-size read. Different headers can therefore take different normalization paths.

Dependencies that have received temporary-header addresses are not established to preserve those buffers across later calls during this invocation. This directly affects loop bounds and successive skip arguments.

### Object and accounting state

Six metadata writes precede preparation. Type-3 processing receives an interior object address and may modify reachable state.

Accounting occurs at most once locally, before preparation, when the accounting flag is nonzero. It is not performed per record. Its amount need not equal the value previously written to object offset 28 if the container field changes between those uses.

No compensating writes or accounting decrement are visible after subsequent failures.

### Ownership

No explicit release, destructor, or rollback dependency is called on success or failure. The object address is not returned directly.

This does not establish caller ownership, a leak, or retention by another component. Acquisition, preparation, and content processing may have internal registration or memory-management effects that are unknown here.

Dependencies are not assumed to preserve globals, expected-signature storage, context fields, header contents, or aliased memory.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-c02571304c5c | Requests input into a supplied destination; sizes are 76 or 24; nonzero status stops processing. |
| F-c70776c346eb | Optional container-header transformation; receives its address; result ignored. |
| F-58734d45a018 | Optional record-header transformation; receives its address; result ignored. |
| F-22914e190356 | Receives context and a skip/advance argument; nonzero status is propagated. |
| F-9e5d6dacabf4 | Acquires an object from three arguments; zero is rejected. |
| F-a9b747a47f9c | Supplies the address used for accounting; no local validity check follows. |
| F-3080127849ac | Supplies an object-associated processing value; zero is rejected. |
| F-ca9ad2c08e4d | Handles content for type 3; nonzero status is propagated. |

These descriptions establish local usage, not complete callee contracts.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **Initial read failure.** If the 76-byte read returns 17, the function returns 17 without local signature inspection, acquisition, or accounting.

2. **Container mismatch.** If the signature differs from the expected value when compared, after any enabled normalization, the result is −208. No initial skip follows.

3. **Empty at the first loop condition.** If recognition, initial skip, acquisition, metadata writes, enabled accounting, and preparation succeed, and the count is still zero when the first loop condition evaluates it, the result is zero with no record read. A zero count only at initial read time is insufficient.

4. **Changed count before admission.** If the initial header contains count zero but intervening dependencies leave count one at the first loop comparison, the counter is zero and a record read is attempted.

5. **Changed count after a record.** If one record completes successfully and the next comparison observes count one, processing terminates successfully at counter one—even if an earlier comparison observed count two.

6. **Acquisition failure.** If recognition and initial skip succeed but acquisition returns zero, the result is −108. No metadata writes, accounting, or preparation follow.

7. **Preparation failure after accounting.** If enabled accounting completes and preparation returns zero, the result is −50. This function performs no compensating decrement or metadata rollback.

8. **Accounting carry.** With low word 4,294,967,280, amount 32 at both amount uses, and no aliasing or interference affecting the update, the low word becomes 16 and the high word increases by one modulo 2³².

9. **Non-type-3 skips with preserved fields.** Suppose the header-extension argument observes extent 24; after that call, dispatch observes a type other than 3 and the content calculation observes record extent 40, trailing count 4, and header extent 24; after the content skip, the separate trailing read still observes 4. The requested skips are 0, 12, and 4, as three distinct calls. A single earlier header snapshot does not by itself establish this sequence.

10. **Changed trailing count.** If the content-skip calculation observes extent 40, header extent 24, and trailing count 4, its argument is 12. If that successful dependency leaves the trailing byte equal to 7, the subsequent trailing skip requests 7, not 4.

11. **Type-3 dispatch.** If the type observed after a successful header-extension skip is 3, the dedicated processor receives literals 2 and 3, the retained preparation value, and the object address plus 32. Only a zero result permits the separate trailing-count read and skip.

12. **Record mismatch.** A mismatch at the record-signature comparison returns −208 before that record’s header-extension skip. Earlier effects are not locally undone.

13. **Trailing-skip failure.** If content handling succeeds but the trailing skip returns 23, the function returns 23 without incrementing the counter.

14. **Unsigned content-length underflow.** If a non-type-3 content calculation observes record extent 20, trailing count 1, and header extent 24, its argument is 4,294,967,291. No local format rejection precedes that call.

## Uncertainties

The following remain **documented_only**:

- Exact expected signature values and accounting-flag displacement.
- Raw byte order and normalization transformations.
- Dependency preservation or mutation of temporary headers, globals, context fields, and aliased storage.
- Whether retained header access is actually used to change later field observations.
- Field meanings, extent units, and metadata semantics beyond demonstrated uses.
- Complete object layout, allocation size, alignment, registration, and ownership.
- Dependency error meanings, input-position guarantees, validation, and memory management.
- Interpretation of large skip-argument bit patterns by the skip dependency.
- Behavior for invalid addresses, insufficient storage, concurrent mutation, and extreme signed arithmetic.

Changing loop bounds and changing record fields are permitted uncertainties of this local contract, not claims that such mutation necessarily occurs.

## Evidence

The supplied static representation establishes fixed read requests, field widths and offsets, metadata write order, branch conditions, explicit statuses, dependency-call ordering, skip expressions, and accounting accesses.

It also establishes that the count field is evaluated at every loop condition, record-content fields are evaluated after the header-extension call, and the trailing byte is read separately after content handling. No preservation guarantee for those fields is supplied.

The format-loading purpose is inferred. Dependency internals and unresolved external values are unknown.

No runtime tests have been performed. The acceptance examples describe conditional behavior derived from static evidence, not observed execution.
