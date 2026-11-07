<!-- Trusted provenance: F-f4ca12fc15d5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f4ca12fc15d5

## Purpose

F-f4ca12fc15d5 performs an insertion-like operation on four-byte words held in contiguous storage described by three 32-bit address fields.

Two behaviors are established by the supplied representation:

- With spare capacity, it advances the logical end, shifts words toward higher addresses, and reads the supplied value only after that movement.
- Without spare capacity, it calculates a replacement capacity, calls an allocation-like dependency, makes two transfer-like calls around a guarded inserted-word write, calls a release-like dependency, and replaces the descriptor fields.

Ordinary sequence insertion is the inferred intent. The direct arithmetic, memory operations, and call ordering are established static evidence. Successful allocation, correct range transfer, ownership transfer, and recovery from failures are not established.

This contract describes the visible behavior independently of those inferred dependency roles. No runtime tests have been performed.

## Interface

### Inputs and layout

The function takes three address-valued arguments:

| Argument | Required interpretation |
|---|---|
| Descriptor address | Address of three consecutive 32-bit fields |
| Insertion address | Address identifying the intended insertion position |
| Value address | Address of a readable four-byte value, read when insertion reaches its direct value-copy operation |

The descriptor layout is:

| Byte offset | Field | Ordinary sequence interpretation |
|---|---|---|
| 0 | Beginning | First storage address |
| 4 | Logical end | Address immediately after the live words |
| 8 | Capacity end | Address immediately after the available storage |

The representation uses 32-bit address-bearing fields and four-byte elements. It does not establish a portable interface for wider addresses.

The value address is not an entry-time value snapshot. Its contents must remain suitable until the actual read if the caller requires insertion of the original value.

### Output

There is no explicit return value or error result. Observable output consists of memory changes and dependency calls.

Under ordinary valid-input conditions and suitable dependency behavior, the intended result is one additional live word at the insertion position.

### Ordinary-use preconditions

Ordinary sequence behavior requires:

- A readable and writable descriptor.
- Valid, suitably aligned storage with four-byte elements.
- Ordered beginning, logical-end, and capacity-end addresses.
- Live and capacity byte distances divisible by four.
- Readable sources and writable destinations for every operation reached.
- Arithmetic that does not encounter signed-overflow ambiguity or address wraparound.
- Dependency contracts compatible with the supplied arguments.
- No unintended aliasing between descriptor fields and element destinations.

These conditions are not checked.

With spare capacity, the insertion address must identify an existing live word for the visible backward movement to terminate within the live range. Neither insertion into an empty sequence nor insertion at the old logical end is implemented as an ordinary append on that path.

The growth path has no corresponding explicit restriction, but its accepted insertion positions depend on the unresolved transfer contract.

## Behavior

### Widths and arithmetic conventions

Counts used for capacity selection are unsigned 32-bit quantities. Address-bearing fields and the operands of several address differences are represented as signed 32-bit values.

A signed right shift by two appears in the live-count, capacity-count, release-count, and final logical-end calculations. For nonnegative distances divisible by four, it gives the ordinary word count. The contract does not replace that operation with unrestricted mathematical division for negative or overflowing differences.

Clearing the low two bits of an insertion displacement is a separate operation. It acts on the difference between the insertion address and a particular observed beginning address; it is not alignment of the insertion address independently.

### Capacity decision

The function first reads the logical end and retains it. It then derives an initial live count from the signed right shift by two of logical end minus beginning, interpreting the shifted result as unsigned.

It separately derives a capacity count from capacity end minus a read of beginning, using the same signed-shift form and unsigned interpretation.

Growth is selected when the capacity count is less than or equal to the live count. Otherwise, the in-place path is selected.

Thus a malformed descriptor whose derived capacity is below its derived length also selects growth. This comparison is not validation or repair of such a descriptor.

### Replacement capacity

Let \(n\) denote the initial unsigned live count. The two candidates, in words, are:

- \(n+32\).
- \(n+\lfloor n/2\rfloor+\lfloor n/8\rfloor\).

The larger unsigned 32-bit result is selected. Each addition has 32-bit unsigned arithmetic behavior; no overflow check is visible.

The allocation request is the selected capacity shifted left by two, also as a 32-bit result. The ordinary interpretation as a byte capacity therefore requires both capacity selection and conversion to bytes to remain in range.

For example, three live words select 35 words and request 140 bytes. Zero live words select 32 words and request 128 bytes.

### Growth-path ordering

The growth path has the following observable sequence:

1. Call F-871e64ddc6de with the calculated byte request and zero. Retain its result as the replacement address.
2. Call D-01 with the descriptor’s current beginning, the insertion address, the replacement address, and the descriptor address.
3. Read beginning again. Subtract it from the insertion address, clear the difference’s low two bits, and add the replacement address. This is the direct inserted-word destination.
4. If that destination is nonzero, read the four-byte value at the value address and write it to the destination.
5. Prepare the second D-01 call using another read of beginning, occurring after the guarded inserted-word operation. Recompute the masked insertion displacement from this fresh beginning. The destination is the replacement address plus that newly computed displacement plus four bytes. The source boundaries are the insertion address and a current read of logical end.
6. Make that second D-01 call.
7. Read and retain beginning and logical end.
8. Advance a local address cursor from the retained beginning toward the retained logical end in four-byte increments, stopping only when equal. The loop body contains no visible memory access or dependency call.
9. Read capacity end and derive the release word count by signed-right-shifting by two the difference between that capacity end and the retained beginning. Call F-ff1bdc376dcd with the retained beginning, that count, and zero.
10. Store the replacement address into the descriptor’s beginning field.
11. Compute the difference between the retained logical end and retained beginning, signed-right-shift it by two, multiply the shifted result by four, then add the replacement address and four. Store that result as logical end.
12. Store the replacement address plus the selected replacement capacity multiplied by four as capacity end.

The final three descriptor stores are ordered beginning, logical end, capacity end.

The source does not establish a relative evaluation order between separate argument expressions within an individual call. The sequence above specifies the dependencies between statements and calls, not an invented total order for all argument reads.

### Fresh reads and retained values

The descriptor is not treated as an immutable snapshot.

In particular, the inserted-word destination and the second transfer destination use distinct beginning reads. The second read occurs after the guarded inserted-word operation. If that operation writes over the descriptor’s beginning field, the second destination uses the newly written beginning value.

For replacement address \(R\), insertion address \(P\), and a beginning observation \(B\), the relevant displacement is the 32-bit difference \(P-B\) with its low two bits cleared. The second destination uses its own later \(B\), not the displacement retained from the direct insertion.

After the second transfer, beginning and logical end are retained before the empty traversal and before the release call. Capacity end is read later for the release count. Consequently:

- The release count uses that later capacity end minus the retained beginning.
- The final logical-end calculation uses the retained beginning and logical end.
- A release dependency that changes descriptor fields does not change those retained operands.
- The selected replacement capacity still comes from the initial live count.
- Normal return from the release dependency is followed by the three explicit descriptor stores.

No preservation of descriptor fields or unrelated global state by a dependency is assumed.

### Final logical-end arithmetic

The final logical end is not unconditionally replacement address plus the retained raw byte distance plus four.

The retained distance is first signed-right-shifted by two and then multiplied by four. For a nonnegative distance, this discards any remainder below four bytes. For ordinary aligned live ranges, it reproduces the original byte distance.

This distinction remains part of the contract even if unknown dependencies introduce unusual descriptor values. Such values can also prevent the preceding traversal from reaching its endpoint; the final store is described only if execution reaches it.

### Guarded insertion and null results

Only the calculated direct inserted-word destination is checked for zero.

A zero destination suppresses that direct value read and write. It does not suppress either transfer call, the release call, or final descriptor replacement if execution otherwise continues.

A zero replacement address does not necessarily produce a zero insertion destination: a nonzero displacement can yield a nonzero address. There is no visible allocation-failure branch, rollback, or error return.

### In-place ordering

The in-place path uses the logical end retained before the capacity decision.

Its operations are:

1. Store that old logical end plus four into the descriptor’s logical-end field.
2. If the old logical end is nonzero, copy the word immediately before it into the word at the old logical end.
3. Start at the last old live-word position. Until that position equals the insertion address, move the source and destination positions backward by four and copy the next word one position higher.
4. Read the supplied four-byte value and write it at the insertion address.

For an ordinary insertion within the live range, movement proceeds from high addresses to low addresses, preserving the suffix despite overlap.

Insertion at the last existing word needs no loop copies beyond the initial last-word duplication. Insertion at the old logical end instead searches backward away from that address. It is not an append case and can access outside valid storage before any possible termination.

An empty sequence with spare capacity has no valid last live word. When the old end is nonzero, the initial copy attempts to read before it. A zero old end suppresses only that initial copy; the later backward traversal is still present.

### Value aliasing

The in-place path reads the inserted value after shifting. If the value address points into a location overwritten by that shift, the function inserts the changed value.

The logical-end update also precedes this read. A value address aliasing that field therefore observes its updated contents, subject to any intervening aliased writes.

On growth, the direct value read occurs after the first transfer and before the second. Its contents and validity can depend on effects of the first dependency. The resulting store can itself change descriptor fields if addresses alias, affecting the fresh reads described above.

## Side effects

### Direct memory effects

The function directly reads descriptor fields, reads and writes four-byte words, and updates descriptor fields.

On the in-place path, only logical end receives an explicit descriptor-field assignment. Beginning and capacity end remain unchanged only when element writes do not alias them.

On growth, the direct inserted-word write precedes the second transfer. All final descriptor assignments follow the release-like call.

The empty traversal contains no visible data access. Nevertheless, its termination condition can affect whether the release and final stores are reached.

### Ownership

The growth path passes an old beginning address to F-ff1bdc376dcd and later publishes the replacement address. This supports an inferred storage-replacement intent, not an established ownership contract.

The available evidence does not establish whether any dependency:

- Allocates or frees storage.
- Constructs, moves, copies, or destroys objects.
- Retains supplied addresses.
- Changes the descriptor or global state.
- Transfers ownership on success or failure.

The function does not separately release the insertion address or value address.

### Errors and partial state

No explicit error handling, atomicity, concurrency guarantee, or recovery mechanism is visible.

On the in-place path, the logical end is extended before movement begins. A later fault can leave an extended descriptor with incomplete data movement.

On growth, dependency effects and the direct inserted-word write occur before release and final descriptor replacement. If a dependency does not return normally, later operations are not established to occur.

Invalid memory access, malformed descriptors, arithmetic outside ordinary representable ranges, and traversal failure are not converted into a defined error result.

## Dependencies

### F-871e64ddc6de

Called once on growth with a 32-bit byte request and zero. Its result is used as a replacement address.

Allocation intent is inferred. Alignment, initialization, ownership, failure representation, and other side effects are documented_only.

### D-01

D-01 is a document-local opaque label for the dependency whose publishing ID was not supplied.

Called twice on growth, each time with two apparent source boundaries, a destination address, and the descriptor address.

The calls suggest prefix and suffix transfer around an inserted word. Endpoint inclusion, overlap handling, copying versus moving, ownership, descriptor mutation, failure behavior, and pointer retention are documented_only.

### F-ff1bdc376dcd

Called once on growth after both D-01 calls and the empty traversal.

Arguments are the retained beginning, the signed-shift-derived capacity distance in words, and zero. The capacity endpoint is read for this call after the traversal; the beginning operand was retained earlier.

Release intent is inferred. The count is not the allocation byte request. Actual deallocation, ownership requirements, accepted null values, and failure behavior are documented_only.

## Acceptance examples

These are conditional static expectations, not executed results.

### Ordinary insertion with spare capacity

Given live words `[10, 20, 30]`, room for at least four words, insertion at the second word, a stable external value `99`, and no descriptor/storage aliasing:

- Logical end advances first.
- Movement duplicates `30` into the new final position and moves `20` one position higher.
- The resulting live words are `[10, 99, 20, 30]`.
- Beginning and capacity end remain unchanged.

Insertion at the third existing word instead produces `[10, 20, 99, 30]`.

### Aliased value after shifting

Given `[10, 20, 30]` with spare capacity, insertion at the first word, and the value address pointing to the original second word:

- Shifting overwrites the second word with `10`.
- The final value read obtains `10`, not `20`.
- The resulting words are `[10, 10, 20, 30]`.

### Ordinary growth

Given three live words and capacity three, the request is 140 bytes for 35 words.

If F-871e64ddc6de supplies suitable storage, D-01 performs the inferred transfers, dependencies preserve the descriptor, release returns normally, and the value remains readable, insertion at the second word produces four live words in replacement storage. The final capacity is 35 words.

### Growth-store alias affecting the second destination

Conditionally, let the replacement address be `0x1000`, the insertion address `0x2004`, and beginning after the first transfer be `0x2000`. The direct insertion destination is `0x1004`.

If that address is also the descriptor’s beginning field and the inserted word is `0x1FFC`, the subsequent fresh beginning read observes `0x1FFC`.

The second transfer destination is then `0x100C`: the new difference is eight bytes, followed by the additional four-byte gap. Reusing the previous displacement would incorrectly predict `0x1008`.

This example specifies call preparation only; it does not assume that the resulting descriptor permits successful completion.

### Retained operands across release

If the retained live byte distance is 12 and release returns normally, the final logical end is replacement address plus 16, even if release changes the descriptor’s logical-end field.

For arithmetic illustration, a retained positive distance of seven yields a shifted-and-rescaled distance of four, not seven. This is not an end-to-end success example: a seven-byte separation cannot be reached by the visible four-byte traversal without other execution assumptions.

### Empty growth and unsupported append

An empty equal-field descriptor selects 32 words and requests 128 bytes. Successful insertion additionally requires dependencies to accept the empty ranges and release arguments.

By contrast, insertion at the old logical end with spare capacity is not accepted as ordinary append behavior; the visible traversal proceeds backward rather than immediately completing.

## Uncertainties

The following remain documented_only:

- All unresolved dependency semantics, including ownership and global-state effects.
- D-01’s publishing identity.
- Allocation-failure representation and propagation.
- Support for null storage and empty ranges.
- Object lifetime semantics beyond direct four-byte memory operations.
- Exact consequences of signed overflow, negative signed shifts, wrapped addresses, invalid alignment, or invalid memory access.
- Whether callers enforce the path-dependent insertion-position requirements.
- Whether the empty traversal is retained in executable form and its runtime consequences.

Aliasing does not make visible read timing unknown: the fresh beginning read after the guarded growth store is established. What remains unknown is which aliased configurations dependencies and callers permit.

## Evidence

Established static evidence is limited to the supplied function representation: three 32-bit descriptor fields, four-byte movement, unsigned capacity selection, two growth formulas, three dependency identities or call roles, two transfer calls, guarded direct writes, repeated descriptor reads, retained pre-release operands, and ordered final descriptor stores.

Insertion, allocation, transfer, and release are inferred intent where dependency implementations are unavailable. No dependency implementation, caller contract, external behavioral documentation, or runtime result was supplied. No runtime tests have been performed.
