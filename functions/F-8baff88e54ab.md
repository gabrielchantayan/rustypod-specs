<!-- Trusted provenance: F-8baff88e54ab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8baff88e54ab

## Purpose

F-8baff88e54ab resolves a record through two keyed dependency calls, initializes a mutable context, obtains an object through indirect dispatch, and processes a sequence of nonzero unsigned 32-bit values.

Each decoded value is decomposed into a bitmap-word index and a bit position. A dependency receives the object address and those coordinates. A zero result causes the function to increment a 32-bit object field and perform a bitmap assignment. Any nonzero result skips those direct updates. Decoding ends on zero.

The inferred intent is to add previously unrepresented positions to a membership bitmap and maintain a corresponding count. That interpretation depends on contracts not supplied here: the test dependency must identify existing membership correctly, and the bitmap addresses must remain consistent during each update. Neither condition is independently enforced.

There is no explicit returned result or error-reporting channel.

## Interface

### Arguments

| Argument | Visible representation | Established use |
|---|---|---|
| Record selector | Opaque 32-bit value | Third argument to the first keyed lookup. |
| Initial cursor | Character-pointer representation | Copied into mutable local storage, then replaced with an embedded-data address before decoding. |
| Initial context | Unsigned 32-bit value | Copied into writable local storage passed to initialization; subsequently used as a tagged address. |
| Auxiliary value | Opaque 32-bit value | Copied into separate local storage; no subsequent explicit use is visible. |

The initial cursor is not explicitly dereferenced before replacement. The auxiliary value is not explicitly supplied to a dependency. These observations do not establish that initialization cannot access surrounding storage: the extent of access through its context pointer is unknown. Conversely, proximity of local storage does not establish an aggregate argument or permission to access neighboring slots.

### Results and completion

The function returns no explicit value. Normal completion occurs after a decoder call returns zero.

Object and bitmap changes may already have occurred before completion. There is no final consistency check, rollback, explicit release, or conversion of dependency failures into a distinct result.

### Required accessible layouts

Successful execution requires valid memory and callable targets for all accesses actually reached:

- **Resolved record:** a 32-bit quantity at byte offset 0, another at offset 4, and an 8-bit quantity at offset 8.
- **Context target:** the post-initialization context, with bit 0 cleared, identifies a location containing a 32-bit dispatch-table address.
- **Dispatch table:** a callable entry is accessible at byte offset 40.
- **Returned object:** a 32-bit field at offset 4 and a 32-bit bitmap-base field at offset 8 are accessible on update paths.
- **Bitmap storage:** each source-word address must support a 32-bit read, and each destination-word address must support a 32-bit write. These addresses arise from separate evaluations and are not unconditionally guaranteed to coincide.
- **Decoder input:** the mutable cursor slot and whatever input the decoder accesses must satisfy its unavailable contract.

No corresponding validity or bounds checks appear in this function. An immediately terminating decoder avoids the loop’s object-field accesses, but not the preceding initialization and indirect dispatch.

## Behavior

### 1. Preserve argument values locally

The function copies the initial cursor, initial context, and auxiliary value into local storage.

Initialization receives the address of the context slot. Decoding later receives the address of the distinct cursor slot. No larger storage-layout contract is established.

### 2. Resolve the record and initialization input

The function first calls F-9d0ecc5f6adf with no explicit arguments and retains its 32-bit result.

It then calls F-5710a8628cba with:

1. The retained result.
2. A 32-bit key obtained from embedded-data offset 0.
3. The record selector.

The returned value is treated as the resolved-record address.

The function calls F-5710a8628cba a second time with:

1. The same retained result from F-9d0ecc5f6adf.
2. A 32-bit key obtained from embedded-data offset 4.
3. The 32-bit quantity read from resolved-record offset 0.

The second lookup result is retained for initialization. This body does not directly dereference that result.

The exact key values, embedded bytes, and byte order are not established by the supplied material. A descriptive label associated with embedded data is not evidence of its contents.

### 3. Initialize the context

After the second lookup returns, the function calls F-59c1430f85c2 with:

1. The address of the mutable 32-bit context slot.
2. The second lookup result.
3. The record quantity at offset 4, masked to its low 16 bits.
4. The byte at record offset 8.

The third argument has a numerical range of 0 through 65,535. Masking a 32-bit quantity establishes that range; it does not independently establish a 16-bit calling-convention parameter.

The metadata reads occur after the second lookup. They must not be described as values captured before that call, because the dependency is not known to preserve the record. The relative evaluation order of the offset-4 and offset-8 reads is not established.

No initialization return value is used. The subsequent context-slot value supplies the address used for dispatch.

### 4. Obtain the processing object

After initialization returns, the function clears only bit 0 of the 32-bit context value, using mask `0xFFFFFFFE`.

The resulting address is used to read a dispatch-table address. The function obtains the callable entry at table byte offset 40, invokes it, and retains its result as the object address used throughout decoding.

There is no broader alignment correction or address validation.

The indirect invocation has no explicit arguments in the supplied representation. That does not establish an ordinary no-argument callback interface: the calling convention, implicit receiver, and register-based inputs remain unknown.

### 5. Replace the decoding cursor

After indirect dispatch returns, the function assigns the address at embedded-data offset 8 to its local cursor.

Consequently, the first decoder call receives that embedded-data cursor rather than the original cursor argument in the visible sequential execution.

The source bytes, extent, encoding, and expected decoded values are unavailable.

### 6. Decode and conditionally update

The function repeatedly calls F-d3922c7fe0ec with the address of the mutable cursor slot. Its result is interpreted as an unsigned 32-bit value.

Zero terminates the loop immediately. Zero is never submitted to the membership dependency.

For every nonzero value:

- The word index is the value shifted right by five bits, equivalent to division by 32 with the remainder discarded.
- The bit position is the low five bits, equivalent to the remainder modulo 32.
- F-88658862e573 receives the retained object address, word index, and bit position.

The word index ranges from 0 through 134,217,727. The bit position ranges from 0 through 31.

A nonzero dependency result skips all direct count and bitmap writes for that iteration. No distinction is made between positive and negative nonzero results.

A zero result causes two update statements, with the count update completed before the bitmap-update statement begins.

**Count update.** The function reads the 32-bit field at object offset 4, adds one, and writes the result back to that field. The representation uses signed 32-bit arithmetic for this increment. Behavior outside its defined arithmetic range is not established.

**Bitmap assignment.** The bitmap-base field at object offset 8 occurs in two separate address evaluations:

- One supplies the base for the source-word address.
- The other supplies the base for the destination-word address.

Each address adds four times the word index to its evaluated base. The source word is read as an unsigned 32-bit quantity. Its value is combined by bitwise OR with a mask selecting the decoded bit position, and the result is stored at the destination address.

The following ordering constraints are established:

- The count update completes before either address evaluation belonging to the bitmap statement.
- The source-word value is obtained before the final destination store.
- No relative evaluation order is established between the source-address and destination-address computations.

These are expression-level evaluations, not a guarantee of two physical machine loads from the bitmap-base field.

If both address evaluations resolve to the same word, the assignment sets the selected bit in that word and preserves its other bits as observed by the source read. If they resolve to different words, the destination receives the source word’s value ORed with the mask. The destination’s previous unrelated bits are not necessarily preserved.

The relative byte offset is at most `0x1FFFFFFC`, or 536,870,908, which fits within 32 bits. This does not establish that adding the base yields a valid address or that suitable storage exists.

### 7. Complete without explicit release

When decoding returns zero, the function returns without explicitly releasing the record, context, returned object, or bitmap storage.

This establishes only the absence of release operations in this body. It does not establish ownership transfer, allocation, reference counting, or whether dependencies acquire or release resources.

## Side effects

### Direct storage changes

The visible body directly writes:

- Local copies of the cursor, context, and auxiliary argument.
- The replacement cursor pointing into embedded data.
- The 32-bit object field at offset 4 on every zero membership result.
- A 32-bit bitmap destination on every such result.

Initialization may modify the context through the pointer passed to it. Decoding may modify the cursor through its pointer. Their full permitted access extents are unknown.

The count write precedes the bitmap assignment. A later failure does not trigger visible rollback. If reachable storage aliases, a later bitmap write may also overwrite storage affected by an earlier operation; no non-aliasing guarantee is supplied.

### Stability and concurrency

No dependency is established to preserve global state, records, object fields, bitmap storage, or ownership.

The object address is retained across iterations, but its contents are read after intervening dependency calls. In particular, count access and both bitmap-base address evaluations follow the membership call.

The bitmap statement is not established as an atomic read-modify-write operation. Under stable source and destination addressing, an intervening modification after the source read can be lost at the final store. Without stable addressing, the operation may copy a modified source value into a different destination word instead.

The count and bitmap are not updated as one transaction. An observer capable of interleaving with execution may observe the count change before the bitmap store.

### Errors

The function contains no explicit handling for missing records, invalid contexts, absent dispatch entries, unusable returned objects, bitmap bounds violations, arithmetic overflow, or distinct decoder and membership errors.

A decoder result of zero ends processing regardless of what zero means internally to that dependency. Every nonzero membership result skips updates, even if that result represents an error.

Dependency exceptions, termination behavior, failure sentinels, and resource effects are undocumented. Normal return establishes completion of the visible control flow, not an independently validated success condition.

## Dependencies

| Opaque function ID | Visible role | Unresolved boundary |
|---|---|---|
| F-9d0ecc5f6adf | Produces the retained first argument for both lookups. | Meaning, lifetime, ownership, failure behavior, and other effects. |
| F-5710a8628cba | Performs two sequential keyed lookups. | Lookup semantics, result validity, mutation, and ownership. |
| F-59c1430f85c2 | Receives writable context storage, the second lookup result, masked metadata, and a byte. | Initialization semantics, access extent, hidden effects, and resource handling. |
| F-d3922c7fe0ec | Produces unsigned 32-bit values through a mutable cursor interface. | Encoding, cursor movement, input bounds, and error conventions. |
| F-88658862e573 | Receives the object and decoded word/bit coordinates. | Whether it actually tests membership, whether it mutates storage, and what nonzero results mean. |

The indirect dispatch target is an additional dependency. No opaque function ID identifies it in the supplied mapping, and its ABI and resource contract are unknown.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, they assume valid accessible memory, completing dependencies, arithmetic within the established range, distinct count and bitmap storage, and no interfering changes. Examples describing a same-word update additionally assume both bitmap-base evaluations resolve to the same base.

### Immediate termination

If the first decoder result is zero, both lookups, initialization, indirect dispatch, and cursor replacement have already occurred. F-88658862e573 is not called, and the loop makes no direct count or bitmap writes.

This does not imply that the invocation has no side effects.

### First bit of a later word

If decoding produces 32 and then zero, the membership dependency receives word index 1 and bit position 0.

If it returns zero, a count of 7 becomes 8 before the bitmap statement. With stable addressing, source word 1 equal to `0x00000004` becomes `0x00000005` at that same word.

### Different source and destination addresses

Suppose decoding produces 32, membership returns zero, and the two bitmap-base evaluations yield different bases through behavior outside the stable-address assumption.

If the source word at relative offset 4 contains `0x00000004`, the destination at relative offset 4 receives `0x00000005`, assuming the mask operation has its ordinary bit interpretation. A previous destination value of `0x00000080` need not be preserved.

This case does not prescribe which address is evaluated first or assert that this function causes the base to change.

### Nonzero membership result

For decoded value 33, the coordinates are word 1, bit 1. If membership returns −1, the iteration performs no direct count or bitmap writes. Dependency-internal effects remain possible.

### Repeated values

For decoded values 33, 33, then zero:

- If membership returns zero first and nonzero second, only the first iteration performs direct updates.
- If membership returns zero twice, the count is incremented twice. With stable addressing and no other changes, the same bit is ORed into the same word twice.

Deduplication depends on membership behavior, not on the bitmap assignment alone.

### Already-set bit

If membership returns zero despite the selected source bit already being set, the count still increments. With stable addressing, the bitmap word may retain its numerical value.

The function does not independently reconcile the membership result with the bitmap.

### Metadata and boundary values

A record quantity of `0xABCD1234` at offset 4 contributes numerical value `0x1234` to initialization. The offset-8 byte supplies a separate argument.

Decoded value 31 selects word 0, bit 31. Under ordinary 32-bit machine-mask behavior, the mask is `0x80000000`; the supplied representation alone does not establish defined signed-shift behavior for this case.

Decoded value `0xFFFFFFFF` selects word 134,217,727, bit 31, with relative byte offset `0x1FFFFFFC`. This arithmetic does not establish valid storage or resolve the high-bit-mask uncertainty.

## Uncertainties

- **documented_only — Embedded data:** Exact lookup keys, sequence bytes, encoding, and decoded values are unavailable.
- **documented_only — Semantic interpretation:** “Count,” “bitmap,” and “membership” describe inferred roles, not independently verified dependency contracts.
- **documented_only — Initialization extent:** The writable context pointer does not establish a four-byte access limit or permission to access neighboring storage.
- **documented_only — Indirect-call ABI:** Hidden arguments, receiver passing, and target identity are unresolved.
- **documented_only — Bitmap evaluation:** Both address expressions are present, but their relative evaluation order and physical load count are not established. Equal source and destination addresses require stability assumptions.
- **documented_only — Arithmetic:** Signed count overflow, signed address calculations outside their defined range, and a mask shift selecting bit 31 lack an independently supplied machine-level resolution.
- **documented_only — Termination:** There is no iteration limit. Normal termination depends on the decoder returning zero.
- **documented_only — Ownership and synchronization:** Allocation, release, aliasing, lifetimes, concurrency guarantees, and dependency preservation are unavailable.

## Evidence

Established behavior derives from static inspection of the supplied function body and opaque dependency mapping: argument copies; sequential lookup calls; record offsets; metadata masking; tagged-context masking; dispatch offset; cursor replacement; zero-terminated decoding; index decomposition; zero-result branching; and the count statement preceding the bitmap statement.

The bitmap assignment contains separate source-address and destination-address evaluations of the bitmap-base field. It establishes a source-value dependency before the final store, but not an order between the two address computations or a required number of physical loads.

Membership semantics and count consistency are inferred intent. Dependency internals, embedded bytes, machine-level arithmetic details, ownership, and memory validity remain outside the established evidence.

No runtime tests have been performed.
