<!-- Trusted provenance: F-ca4f8eb7b449; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca4f8eb7b449

## Purpose

F-ca4f8eb7b449 conditionally delegates two payload values, validates metadata preceding the payload, updates context-relative accounting fields, and delegates the metadata address to another dependency.

A null input returns immediately. A non-null input triggers three dependency calls before metadata validation. If validation succeeds and subsequent operations complete, the function performs seven ordered accounting writes and two final dependency calls.

The metadata layout, masked quantity, and accounting reductions suggest an allocation-release operation. This is inferred intent. The supplied representation does not establish that any dependency releases memory, transfers ownership, or implements synchronization.

## Interface

**Input:** one pointer, designated **P**.

**Output:** no explicit return value.

The supplied representation uses 32-bit payload words, metadata words, context addresses, and accounting fields. Two metadata selectors are unsigned 16-bit values. Address calculations below describe the represented 32-bit address model; portability to another address width is not established.

### Payload and preceding metadata

| Address | Width | Use |
|---|---:|---|
| P | 32 bits | Argument to the first dependency |
| P+64 | 32 bits | Argument to F-cde83863856d |
| P−8 | 32 bits | Validation word and source of masked quantities |
| P−4 | 16 bits, unsigned | First accounting selector |
| P−2 | 16 bits, unsigned | Second accounting selector |

The address P−8 is also passed to F-cc6c9b4a0f68.

The selectors are defined by their byte addresses. Their interpretation does not require assuming how a 32-bit word containing both selectors would be numerically decoded.

### Captured context

After the initial three dependency calls, the function loads a 32-bit context address from a global pointer-bearing location. This captured address is designated **C**.

| Address | Width | Direct update |
|---|---:|---|
| C+0x224+4×first selector | 32 bits | Subtract first masked quantity |
| C+0x30C | 32 bits | Subtract first masked quantity |
| C+0xE0+4×second selector | 32 bits | Subtract first masked quantity |
| C+0x220 | 32 bits | Subtract first masked quantity |
| C+0x310+4×dependency result | 32 bits | Decrement by one |
| C+0x390 | 32 bits | Decrement by one |
| C+8 | 32 bits | Subtract a newly read masked quantity |

These formulas do not establish separate arrays or disjoint storage. Context-relative addresses may overlap each other, payload storage, preceding metadata, or other accessible storage. No relationship between C and P is checked.

### Preconditions for normal completion

The visible function does not establish memory safety. Normal completion requires:

- Readable storage for each payload and metadata access when that access occurs.
- A usable captured context address.
- Readable and writable storage at every computed accounting address.
- Selector values and a dependency-derived index that produce valid addresses.
- Continued validity of required storage after intervening dependency calls and accounting writes.
- Returning dependencies on the continuation paths described below.

There are no visible context-null checks, selector bounds checks, dependency-index bounds checks, or accounting-underflow checks. Alignment requirements, byte order, and invalid-memory behavior remain documented_only.

## Behavior

### Null input

If P is null, the function returns without reading payload or metadata, loading C, writing accounting fields, or calling any dependency.

A later repeated non-null condition does not create another reachable null-input outcome: the function has not reassigned P.

### Initial dependency calls

For non-null P, the function performs these operations in order:

1. Reads the 32-bit value at P and passes it to D-01, the document-local opaque identifier for the unmapped external dependency.
2. After D-01 returns, reads the 32-bit value at P+64 and passes it to F-cde83863856d.
3. After that dependency returns, invokes F-2fd8826557a4 without explicit arguments.
4. After that dependency returns, loads C from the global pointer-bearing location.

The payload values are not captured together. D-01 may affect the value subsequently read at P+64 or the validity of that storage. The available evidence supplies no preservation guarantee.

These calls precede metadata validation. A subsequently rejected metadata word does not prevent their effects, and no local rollback is shown.

### Metadata validation

The function first reads the 32-bit word at P−8 and tests whether it is zero.

- If this read yields zero, it invokes F-a817d8d58c10.
- Otherwise, it reads the word again and invokes F-a817d8d58c10 if bit 0 of that second read is set.

F-a817d8d58c10 is represented as non-returning. No bookkeeping or final delegation follows on that represented path. Its termination mechanism is unknown.

Validation therefore rejects zero on the first read or an odd value on the second read. It does not establish that both reads observe the same value. For example, a nonzero first read followed by a zero second read passes the displayed checks.

No other bits are validated. A stable, nonzero word with bit 0 clear can pass even when its masked accounting quantity is zero.

### Preparation and first four accounting writes

After validation, the function invokes F-c0fc6d033caa with C. It does not reload C afterward.

Once that call returns, the function:

1. Reads the unsigned 16-bit selector at P−4 and forms the first selected accounting address.
2. Reads the 32-bit word at P−8 and captures its value masked with **0x3FFFFFFC**.
3. Reads, subtracts the captured quantity from, and writes the field at C+0x224+4×first selector.
4. Performs the same subtraction at C+0x30C.
5. Reads the unsigned 16-bit selector at P−2 and forms the second selected accounting address.
6. Performs the same subtraction at C+0xE0+4×second selector.
7. Performs the same subtraction at C+0x220.

The mask clears bits 0, 1, 30, and 31. The resulting quantity ranges from zero through 0x3FFFFFFC, in multiples of four. The meanings of the excluded bits are not established.

Each selector is zero-extended before multiplication by four. Its scaled contribution ranges from zero through 262,140 bytes. No smaller admissible range is enforced.

The first masked quantity is retained for all four writes. Changes to the metadata word during those writes do not change that captured quantity. However, the second selector is read only after the first two writes. If either write aliases its storage, the second selector can reflect that direct modification.

Every accounting read-modify-write operates on the value present at its own read. Overlapping targets therefore accumulate effects in execution order; the function does not snapshot all accounting fields before writing.

### Count updates and final quantity subtraction

After the first four writes, the function invokes F-09dcb55e8e0a without explicit arguments. Its result is used as an index in the represented 32-bit address calculation.

The function then:

1. Decrements the 32-bit field at C+0x310+4×index by one.
2. Decrements the 32-bit field at C+0x390 by one.
3. Reads the metadata word at P−8 again, masks it with 0x3FFFFFFC, and subtracts this newly derived quantity from the field at C+8.

The final write follows both count writes. The supplied expression requires the current field value and the fresh metadata quantity before that write, but does not independently establish an order between those two reads.

The final quantity need not equal the quantity used for the first four subtractions. It can differ because of dependency effects or because an earlier accounting write aliases the metadata word.

C remains the captured address throughout. A dependency or an aliased accounting write may change the global location from which C was loaded, but this function does not consult that location again.

The quantity subtractions use a 32-bit unsigned masked operand and store 32-bit results. The count decrements are represented with signed 32-bit integer operations. No checked or saturating arithmetic is visible. The supplied representation does not establish a portable execution rule for every overflow, underflow, or address-wraparound case.

### Final delegation

After all seven accounting writes, the function calls:

1. F-cc6c9b4a0f68 with C and P−8.
2. F-e586bff64fb7 with C, if the preceding call returns.

If both return, the function returns without an explicit result.

There are no explicit P-relative stores. Nevertheless, context-relative accounting writes may directly modify payload or metadata storage through aliasing. In particular:

- If C equals P−8, the final write at C+8 targets P.
- If C equals P−16, the final write at C+8 targets the metadata word at P−8.

The function does not revalidate metadata after these writes. The address passed to F-cc6c9b4a0f68 remains P−8 even if the contents at that address have changed.

## Side effects

### Direct effects

On the fully completed non-null path, the function performs seven ordered 32-bit read-modify-write updates:

- Four subtractions using one captured masked quantity.
- Two decrements by one.
- One subtraction using a freshly read masked quantity.

These writes can affect any accessible storage reached by the stated context-relative formulas. There is no established separation from P-relative storage. A write may therefore modify or clear a payload field, metadata word, or selector without any dependency performing that mutation.

The function also reads payload values, preceding metadata, the global context address, and accounting fields. Later reads can observe earlier direct writes when their addresses overlap.

### Dependency-mediated effects

Dependency effects are unspecified beyond their visible arguments, ordering, and use of the index result. The evidence does not establish that dependencies preserve:

- Global state or the captured context’s contents.
- Payload or metadata contents.
- Allocation validity.
- Ownership of referenced resources.

The apparent preparation/finalization pairing of F-c0fc6d033caa and F-e586bff64fb7 does not prove locking or unlocking.

### Ownership, failure, and concurrency

Passing payload values and a metadata address does not by itself establish ownership transfer. No explicit ownership marker or dedicated P-relative clearing operation is shown. This does not imply those locations remain unchanged: accounting writes may clear or alter them through aliasing, and dependencies may also mutate them.

There is no visible transaction or recovery path. Metadata rejection occurs after initial calls. A later failure may occur after some accounting writes have completed, without restoring earlier state.

The updates are represented as ordinary read-modify-write operations. Atomicity, exclusion, and a concurrency model are not established. Observation-dependent examples below do not assert that concurrent mutation is safe or supported.

## Dependencies

| Opaque ID | Visible call-site contract |
|---|---|
| D-01 | Receives the 32-bit value read at P; first call for non-null input. This is a document-local label, not a recovered identity. |
| F-cde83863856d | Receives the subsequently read 32-bit value at P+64. |
| F-2fd8826557a4 | No explicit arguments; returns before C is loaded on the continuing path. |
| F-a817d8d58c10 | No explicit arguments; invoked for rejected metadata and represented as non-returning. |
| F-c0fc6d033caa | Receives C after validation and before selectors and quantities are read for accounting. |
| F-09dcb55e8e0a | No explicit arguments; called after the first four writes; supplies the count index. |
| F-cc6c9b4a0f68 | Receives C and P−8 after all seven accounting writes. |
| F-e586bff64fb7 | Receives C after F-cc6c9b4a0f68 returns. |

Other effects, actual ownership contracts, and ordinary-return guarantees remain documented_only.

## Acceptance examples

These examples are conditional consequences of static evidence, not executed tests. Unless stated otherwise, all required addresses are valid, dependencies return on continuation paths without modifying relevant storage, address arithmetic is valid, and arithmetic results remain representable without disputed overflow behavior.

### Null input

For null P, no dependency is called and no payload, metadata, or context access occurs.

### Rejection occurs after initial effects

For non-null P whose first validation read yields zero, D-01, F-cde83863856d, and F-2fd8826557a4 have already been called, and C has been loaded, before F-a817d8d58c10 is invoked. No accounting write follows on the represented non-returning path.

If the first validation read is nonzero and the second has bit 0 set, rejection likewise occurs before F-c0fc6d033caa.

### Ordinary non-overlapping accounting

With stable metadata 0x00000100 and mutually non-overlapping accounting targets that are also disjoint from payload and metadata storage:

- Each of the five quantity fields decreases by 256.
- Each of the two count fields decreases by one.
- The final dependencies receive the stated arguments in the stated order.

### Accepted zero quantity

With stable metadata 0x00000002, validation passes and the masked quantity is zero. If the accounting targets are disjoint, the five quantity writes preserve their respective read values, while both count decrements still occur.

This example does not imply quantity fields remain unchanged if another update aliases them.

### Dependency changes the final quantity

If the first captured quantity is 256 and F-09dcb55e8e0a changes the metadata so its final masked value is 512, the first four subtractions use 256 and the final subtraction uses 512, provided intervening writes do not alter that metadata again.

### Overlapping count fields

If F-09dcb55e8e0a returns 32, the indexed count address equals C+0x390. That field is decremented twice in sequence. With an initial value of 10 at the start of those decrements, it becomes 8.

### Final write changes the payload directly

Let C equal P−8, let the metadata remain 0x00000100 until the final subtraction, and let P contain 0x00000900 at that time. Assume earlier accounting targets do not overlap either word.

The final accounting write targets P and changes it to 0x00000800. D-01 has already received the value read at P during the initial call sequence. This payload mutation is a direct accounting effect, not necessarily a dependency effect.

### Final write clears the metadata directly

Let C equal P−16, with metadata 0x00000100 and earlier targets disjoint from it.

The final accounting target is P−8. Its current value is 256 and its masked quantity is 256, so that write changes the metadata word to zero. No validation is repeated. F-cc6c9b4a0f68 is subsequently called with the address of that now-zero word.

## Uncertainties

The following remain **documented_only**:

- Business meanings of payload fields, metadata bits, and accounting categories.
- D-01’s identity and implementation.
- Allocation extents and any intended separation between context, payload, metadata, and global storage.
- Valid selector ranges, table extents, and intended alias restrictions.
- The actual interface and permitted result range of F-09dcb55e8e0a. Its result is used as a signed 32-bit integer in the supplied representation; no nonnegative guarantee is shown.
- Whether dependencies free, retain, invalidate, or otherwise change referenced storage.
- Ownership-transfer and synchronization contracts.
- Alignment, byte order, concurrency guarantees, memory-fault behavior, and target-specific arithmetic exceptional behavior.
- The termination mechanism of F-a817d8d58c10.

Unknown separation cannot be treated as proof of non-aliasing. Conversely, the possible alias examples do not establish that such layouts are intended or valid in the surrounding system.

## Evidence

**Established static evidence:** the supplied representation shows the null return, dependency ordering, payload and metadata offsets, context capture, two-stage metadata check, mask, seven ordered accounting writes, fresh final metadata read, and final dependency arguments. The write-address formulas contain no disjointness checks against P-relative storage.

**Inferred intent:** the preceding metadata, masked quantity, counter reductions, and final delegation are consistent with allocator bookkeeping and resource release. They do not prove release, ownership transfer, or synchronization.

**Verification status:** no runtime tests have been performed. Acceptance examples express conditional static behavior, not observed execution results.
