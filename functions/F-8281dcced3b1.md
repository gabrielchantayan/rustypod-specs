<!-- Trusted provenance: F-8281dcced3b1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8281dcced3b1

## Purpose

F-8281dcced3b1 operates on a three-field storage descriptor and a supplied insertion address. It either shifts four-byte words within existing storage or requests replacement storage, invokes two transfer operations, and publishes replacement descriptor fields.

**Established static behavior:** branch selection depends on unsigned comparisons of quantities derived from the descriptor’s address differences. The existing-storage branch updates the logical end before moving words. The replacement branch reads descriptor fields at several distinct times, including a fresh storage-start read after the inserted-word store.

**Inferred intent:** insertion of one four-byte element into a vector-like sequence, with geometric capacity growth when existing capacity is exhausted.

This intent is conditional, not an unrestricted insertion guarantee. Existing-storage insertion has an end-position limitation, and ordinary sequence semantics require descriptor/storage non-overlap. Replacement-storage sequence preservation depends on an unidentified transfer dependency.

## Interface

The routine accepts three arguments and has no explicit return value:

| Input | Representation | Use |
|---|---|---|
| Descriptor address | Address of three consecutive four-byte fields | Supplies and receives storage metadata |
| Insertion address | Address represented in 32 bits | Determines where the inserted word belongs |
| Value address | Address of a readable four-byte word | Supplies the word through a late load |

The descriptor layout is:

| Byte offset | Field | Intended meaning |
|---|---|---|
| 0 | Storage start | First element address |
| 4 | Logical end | Address immediately beyond used elements |
| 8 | Allocation end | Address immediately beyond available storage |

The visible operations use four-byte words, 32-bit address representations, and 32-bit integer calculations. Words are opaque: no element comparison, construction, or destruction is established.

For ordinary sequence semantics and safe memory access, callers need:

- A writable descriptor and readable descriptor fields.
- Coherent, suitably aligned storage extents.
- An insertion address on a four-byte element boundary.
- Accessible source and destination locations for all selected operations.
- A value address readable when the actual value load occurs.
- Address differences, capacities, and allocation sizes within supported arithmetic limits.
- **Descriptor fields that do not overlap element storage being read or written.**
- Any further requirements imposed by the dependencies.

The routine does not enforce these conditions. In particular, coherent extents do not establish descriptor/storage non-overlap.

The value address need not be separate from element storage for the visible operations to occur, but aliasing can change which value is eventually inserted. If it aliases descriptor fields, metadata writes can likewise change the value before it is loaded.

No status value, explicit allocation-failure result, validation failure, or rollback mechanism is present.

## Behavior

### Initial measurements and branch selection

The routine reads and retains the logical-end address. It derives a used-length quantity from logical end minus storage start and a capacity quantity from allocation end minus storage start.

Each address difference is treated as a signed 32-bit quantity and shifted right by two bits. The resulting quantities are then compared as unsigned 32-bit values.

For aligned, nonnegative differences within the signed range, these quantities equal used-word count and capacity in words. That interpretation does not extend automatically to malformed extents, negative differences, signed overflow, or wrapped addresses.

The replacement branch is selected when unsigned capacity is less than or equal to unsigned used length. Otherwise, the existing-storage branch is selected.

Thus a conventional full descriptor selects replacement. A descriptor with capacity below its used length also selects replacement; this is not a validation or repair guarantee.

### Replacement capacity and allocation request

Let \(n\) be the initially derived unsigned used-length quantity.

The routine calculates two candidates using 32-bit unsigned arithmetic:

- \(n + 32\).
- \(n + \lfloor n/2 \rfloor + \lfloor n/8 \rfloor\).

It selects the larger candidate by unsigned comparison. Candidate additions can wrap. The selected capacity is shifted left by two bits to obtain the byte request, which can also lose high bits. No overflow check is visible.

Without overflow, the policy provides at least 32 additional slots and otherwise approximately 1.625-times growth. The integer expressions, rather than that approximation, determine the request.

F-871e64ddc6de receives the byte request and zero. Its result is retained as the replacement base address. There is no direct test that this result is nonzero.

### Replacement transfer and insertion ordering

After the allocation-like call, the following order is established:

1. The unidentified transfer dependency receives the descriptor’s storage start as read for this call, the insertion address, the replacement base, and the descriptor address.

2. After that call returns, the routine reads storage start again. It subtracts that value from the insertion address, clears the difference’s lowest two bits, and adds the replacement base. This produces the inserted-word destination.

3. If that destination is nonzero, the routine reads one four-byte word from the value address and writes it to the destination. If it is zero, both that load and that store are skipped.

4. For the second transfer call, the routine **reads storage start afresh**, recomputes insertion address minus that start with the lowest two bits cleared, and adds the replacement base and four bytes. This recomputed address is the second transfer destination. The other arguments are the insertion address, the descriptor’s logical end as read for this call, and the descriptor address.

The second destination is not necessarily the first destination plus four. Equality requires that the two storage-start reads produce differences with the same masked offset. An unchanged storage-start field is sufficient.

The intervening inserted-word store can itself overwrite storage start if its destination overlaps that descriptor field. Consequently, a dependency need not mutate the descriptor for the two offsets to differ.

The source range-like arguments suggest prefix transfer followed by suffix transfer around the inserted word. Actual copying, range conventions, overlap support, and descriptor effects are unknown. The order of individual argument-field reads within a call’s argument preparation is not independently established here.

### Replacement retirement and descriptor publication

After the second transfer call returns, the routine:

1. Snapshots the descriptor’s then-current storage start and logical end.
2. Traverses from the snapshotted start toward the snapshotted end, advancing by four bytes until equality.
3. Reads allocation end and calls F-ff1bdc376dcd with the snapshotted start, the signed allocation-end-minus-start difference shifted right by two, and zero.
4. Writes descriptor fields in this order:
   - Storage start receives the replacement base.
   - Logical end receives the replacement base plus four times the word-count-like difference derived from the two snapshots, plus four bytes.
   - Allocation end receives the replacement base plus four times the selected capacity.

The empty traversal has no visible element reads, writes, or destructor calls. It therefore does not establish element destruction. It requires endpoints reachable by repeated four-byte advancement to terminate; the supplied evidence does not provide a general termination guarantee for malformed endpoints.

The final logical-end calculation uses the snapshots taken **after the second transfer**, not necessarily the original used length. Dependency effects and earlier aliased writes can therefore affect the published length.

The release-like call precedes publication. Although it can have unknown effects, the retained replacement base, selected capacity, and start/end snapshots supply the subsequent field values.

### Existing-storage ordering

Using the logical end retained before branch selection, the routine:

1. Writes old logical end plus four bytes to the descriptor’s logical-end field.
2. If old logical end is nonzero, reads the word immediately before it and writes that word at old logical end.
3. Starts a backward source position one word before old logical end.
4. Until that position equals the insertion address, moves source and destination backward by one word, then reads the source word and writes the destination word.
5. Reads the value-address word and writes it at the insertion address.

Under coherent, accessible storage, a nonempty sequence, an insertion address identifying an existing element, and **descriptor/storage non-overlap**, these operations shift the addressed element and following elements one slot upward.

The ordinary “insert the supplied value” result also requires that earlier writes do not change the value-address contents. Otherwise the final insertion uses the contents present at the late load, not necessarily the value present on entry.

The logical-end field is advanced before any element copying. An abnormal exit can leave that field advanced with movement incomplete.

### Descriptor/storage overlap

Without non-overlap, neither ordinary sequence contents nor preservation of descriptor fields follows from the branch structure.

The initial logical-end write can change an element before the first element read. Later tail, shift, or insertion stores can overwrite any descriptor field if their addresses coincide with those fields.

Accordingly:

- Logical end is the only descriptor field explicitly targeted as metadata in the existing-storage branch.
- It is **not** necessarily the only descriptor field changed.
- Storage start and allocation end remain unchanged only when no element destination aliases them.
- Even the final logical end need not equal the initially published old-end-plus-four value if later element stores overwrite it.

### End insertion and empty sequences

The backward loop initially compares the insertion address with old logical end minus four, not with old logical end.

For insertion at old logical end, it therefore begins moving downward instead of immediately completing an append. A correct ordinary append is not guaranteed. Invalid memory accesses can occur before any eventual equality.

For an empty sequence with spare capacity and a nonzero end address, the initial tail copy reads before the sequence start. If the end address is zero, that copy is skipped, but the backward-loop problem remains.

The replacement branch has no corresponding backward loop. Empty insertion or append there still depends on the transfer dependency accepting the relevant ranges.

## Side effects

### Memory and state

The existing-storage branch explicitly writes the logical-end field, tail slot, backward-shift destinations, and insertion address. With descriptor/storage non-overlap, storage start and allocation end are not changed by this branch. Without it, element writes can modify metadata.

The replacement branch directly publishes all three descriptor fields after the release-like call. Before publication:

- Both transfer calls receive the descriptor address.
- The inserted-word destination can overlap the descriptor.
- Dependencies may affect descriptor fields or other aliased memory.

The surrounding replacement words are not directly established by this routine; their contents depend on the transfer dependency.

No claim is made that dependencies preserve globals, avoid callbacks, preserve input storage, or refrain from changing the value-address contents.

### Ownership

F-871e64ddc6de appears allocation-like, and F-ff1bdc376dcd appears release-like. These roles are inferred from usage.

Established facts are limited to their arguments, the first result’s use as an address, the release-like call’s position before publication, and the final descriptor assignments.

Allocation ownership, allocator compatibility, successful release, and ownership transfer are **documented_only**. Unknown callees are not assumed to acquire or release memory correctly.

### Errors and partial completion

No rollback is visible.

A zero allocation result does not by itself suppress transfer or release-like calls. Only the calculated inserted-word destination is checked for zero; a zero base combined with a nonzero offset can still produce a nonzero destination.

Memory faults, dependency failures, and nontermination are not converted into an explicit error result. Earlier writes remain potentially observable after abnormal completion.

## Dependencies

| Dependency | Established use | Unresolved contract |
|---|---|---|
| F-871e64ddc6de | Receives a 32-bit byte request and zero; result is used as replacement base | Allocation, alignment, failure signaling, ownership, global and aliased-memory effects |
| F-ff1bdc376dcd | Receives snapshotted storage start, a signed shifted allocation-extent quantity, and zero | Release semantics, required units, ownership requirements, failure behavior and other effects |
| Unidentified transfer dependency; opaque ID unavailable | Called twice with two range-like addresses, a destination, and the descriptor address | Identity, transfer semantics, overlap support, range conventions, descriptor mutation, allocation/release behavior and errors |

No dependency implementation was supplied. Assigning a mapped function ID to the unidentified transfer dependency would be unsupported.

## Acceptance examples

These are conditional predictions from static evidence, not runtime observations.

### Ordinary existing-storage insertion

Given separate descriptor storage, words \([10,20,30]\), capacity for at least four words, insertion at the second word, and an unaffected external value of 99:

- Resulting words are \([10,99,20,30]\).
- Logical end advances four bytes.
- Storage start and allocation end remain unchanged.
- No dependency is called.

These expectations require accessible storage, coherent arithmetic, and normal completion.

### Value alias changed by shifting

With the same separate descriptor and sequence, let the value address identify the original third-word location.

The tail copy preserves 30 in the new tail slot. The shift then replaces the original third-word location with 20. The late value load reads 20.

Expected words are \([10,20,20,30]\), not \([10,30,20,30]\).

### Element storage overlapping the descriptor

Assume writable storage begins at address `0x1000`, and the descriptor occupies its first three words:

- Storage start: `0x1000`.
- Logical end: `0x100c`.
- Allocation end: `0x1010`.

The extents describe three used words and four slots, but the elements are also the descriptor fields. Insert at `0x1000` using a separate value word containing 99.

The initial logical-end update changes the second element to `0x1010` before shifting. Subsequent movement copies descriptor values as element contents. The final insertion overwrites storage start with 99; the backward shift has overwritten logical end with `0x1000`.

Under the stated accessible-memory conditions, the final descriptor fields are 99, `0x1000`, and `0x1010`. Ordinary sequence insertion and preservation of storage start are therefore not acceptance expectations for overlapping descriptor/storage layouts.

### Small full sequence

For eight used words and eight slots, without overflow:

- Additive capacity candidate: 40.
- Geometric candidate: 13.
- Allocation request: 160 bytes, with second argument zero.

If dependencies preserve the descriptor until publication and implement the intended transfers, normal completion publishes nine used words and capacity for 40 words.

### Geometric growth

For 64 used words in full storage:

- Additive candidate: 96.
- Geometric candidate: 104.
- Allocation request: 416 bytes.

Sequence preservation remains conditional on dependency behavior and suitable memory separation.

### Inserted-word store changes the second transfer destination

Suppose the replacement branch is selected, insertion address and storage start are both `0x1000`, and the first transfer leaves storage start unchanged.

Conditionally let the allocation result equal the descriptor address, `0x3000`. The first inserted-word destination is then `0x3000`. If the value word contains `0x0ffc`, that store changes the descriptor’s storage-start field to `0x0ffc`.

The second transfer destination is recomputed as:

\[
\texttt{0x3000}
+ ((\texttt{0x1000}-\texttt{0x0ffc}) \mathbin{\&} \texttt{0xfffffffc})
+4
= \texttt{0x3008}.
\]

It is not `0x3004`, the first destination plus four. This example requires the deliberately overlapping allocation result and accessible descriptor memory; it does not assert that F-871e64ddc6de normally returns such storage.

### End insertion with spare capacity

For a nonempty sequence with spare capacity and insertion at logical end, ordinary append is not an acceptance expectation. The backward source begins below the insertion address and moves farther downward.

## Uncertainties

The following remain **documented_only**:

- The unidentified transfer dependency’s identity and complete behavior.
- Allocation and release guarantees, ownership, alignment, and failure signaling.
- Caller guarantees concerning descriptor/storage separation and value aliasing.
- Caller restrictions excluding end insertion on the existing-storage branch.
- Effects of dependencies on globals, descriptor fields, source storage, and the supplied value.
- Platform-level consequences of signed overflow, negative shifts, wrapped address calculations, misalignment, and inaccessible memory.
- Whether malformed empty-traversal endpoints eventually meet under the execution environment’s address arithmetic.

Vector insertion is inferred intent. The fresh storage-start read before the second transfer, late value loads, descriptor-write ordering, and exposure to descriptor/storage overlap are established static behavior.

## Evidence

This contract uses only the supplied routine body and dependency mapping. Prior prose and feedback are not independent behavioral evidence.

The body establishes three four-byte descriptor fields, unsigned growth selection, two transfer calls surrounding a conditional inserted-word store, independently computed transfer offsets, post-transfer snapshots, a release-like call before descriptor publication, and backward movement after an early logical-end update.

No callee implementation was supplied. No runtime tests, allocation experiments, fault injection, or interactive observations have been performed.
