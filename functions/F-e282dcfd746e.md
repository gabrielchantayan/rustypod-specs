<!-- Trusted provenance: F-e282dcfd746e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e282dcfd746e

## Purpose

F-e282dcfd746e writes two caller-supplied 32-bit values into memory obtained through a dependency, then invokes a target selected from a global function-pointer slot.

The established sequence is:

1. Call F-d33f6d597e5d with 24 and retain its returned address-bearing value.
2. Call F-unresolved-01 with that value and 24.
3. Store the first input at byte offset 16 from the retained value.
4. Store the second input at byte offset 20.
5. Read the indirect target from a global function-pointer slot.
6. Invoke the selected target with the ordered arguments 2, the retained value, 5, and 0.

The global target read occurs **after both input stores**. Target selection is therefore affected by any earlier changes to the slot, including changes made by dependencies or by the two stores if their destinations alias the slot. The evidence supplies no non-aliasing guarantee.

The repeated argument 24 and the last written byte at offset 23 are consistent with preparation of a 24-byte record. Allocation, initialization, submission semantics, and ownership transfer are inferred possibilities, not established dependency contracts.

## Interface

### Inputs

The function accepts two four-byte values.

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | Stored at byte offset 16 from the returned base |
| Second input | 32 bits | Stored at byte offset 20 from the returned base |

The inputs are treated as uninterpreted 32-bit values. Neither is validated, masked, narrowed, compared, or used to select a local branch.

All 32 bits of each input participate in its corresponding store. The evidence does not establish whether callers interpret these values as signed integers, unsigned integers, flags, encoded addresses, or another representation.

### Output

The visible interface has no explicit return value. Results from F-unresolved-01 and the final indirect invocation are unused.

Normal completion requires the dependencies to return and the intervening memory accesses to complete. There is no local success or failure status.

### Retained base and required storage

Let **B** denote the value returned by F-d33f6d597e5d. This is a specification label, not an additional input or memory field.

The supplied representation carries B in a 32-bit integer-shaped intermediate. That establishes the visible carrier width, not a portable pointer type, a signed-address interpretation, or an accepted address range.

For the two stores to complete normally, the effective addresses corresponding to B + 16 and B + 20 must each permit a four-byte write at the time that write occurs. Alignment requirements, address-overflow behavior, and invalid-address behavior are not established.

A contiguous writable 24-byte region beginning at B would contain both stores in the ordinary non-wrapping case. The function does not establish or check that such a region exists.

### Global dispatch slot

A global function-pointer slot supplies the final invocation target. The source establishes this selection mechanism and the position of its read after the stores.

The slot’s contents, pointer representation, atomicity, and valid target set are not supplied. The selected target’s signature is represented at this call site by four ordered arguments; exact ABI-level argument widths are not independently established.

## Behavior

### Initial dependency call

The first visible operation is a call to F-d33f6d597e5d with the single argument 24.

Its returned value becomes B. There is no visible retry, alternative source, null check, failure-sentinel check, or bounds check.

The argument is numerically 24. Interpreting it as an allocation size requires a dependency contract that is not supplied.

If B is zero or otherwise unsuitable for the later memory accesses, the function does not handle that condition locally before continuing.

### Preparation dependency

Next, F-unresolved-01 is called with these ordered arguments:

1. B.
2. 24.

This call precedes both input stores. Its result is not used.

No supplied implementation establishes what this dependency does. It may affect the addressed storage, the global dispatch slot, unrelated globals, or resources. The contract does not assume that it clears memory, preserves memory, preserves globals, or leaves B usable.

The function retains the same B after this call; it does not obtain a replacement base from the call’s return value. If the dependency invalidates storage associated with B, the subsequent stores still use the retained base.

### Ordered input stores

After F-unresolved-01 returns, the function performs these stores in order:

| Order | Starting byte offset from B | Covered offsets | Stored value |
|---:|---:|---|---|
| 1 | 16 | 16–19 | First input, all 32 bits |
| 2 | 20 | 20–23 | Second input, all 32 bits |

The first store precedes the second. Both precede the global dispatch-slot read.

In ordinary non-wrapping address arithmetic, the two ranges are adjacent, non-overlapping, and collectively eight bytes wide. The largest explicitly written byte offset is 23.

There are no explicit local stores to offsets 0–15. This does not establish that those bytes remain unchanged: dependencies may modify them, and the final target may modify them after invocation.

The stores use the target’s four-byte value representation. Byte order is not independently established, so this contract specifies the stored values rather than an ordered sequence of individual bytes.

There is no explicit readback of either field to verify the stores. However, if a store destination overlaps the global dispatch slot, the subsequent slot read may observe bytes affected by that store.

### Global target selection

After the second input store, the function reads the target from the global function-pointer slot.

This is an observable read with a specified position in the sequence. Target identity is unknown, but the target-selection mechanism is not: the function selects through that global slot at this point.

The evidence does not justify treating the slot as immutable or independent of earlier operations:

- F-d33f6d597e5d may modify the slot.
- F-unresolved-01 may modify the slot.
- Either input store may affect the slot if its destination overlaps it.
- No supplied guarantee excludes those aliases or global effects.

Consequently, selecting or caching the target before the preceding calls or stores is not an equivalent behavioral contract. Earlier selection could choose a different target from the one obtained by the established late read.

This ordering does not establish synchronization with concurrent writers. The behavior of concurrent slot updates, torn reads, or invalid pointer representations remains undocumented.

### Indirect invocation

The function invokes the target obtained from the slot with these ordered arguments:

| Position | Argument |
|---:|---|
| 1 | 2 |
| 2 | B |
| 3 | 5 |
| 4 | 0 |

The second argument is the retained base B, not the address of either input field.

The meanings of 2, 5, and 0 are not supplied. They must not be assigned command, event, size, priority, channel, ownership, or status semantics without further evidence.

When the storage operations complete normally, both input stores precede target selection and invocation. This establishes ordering, not a guarantee that the selected target reads, preserves, copies, or retains those values. In particular, dependencies or external actors may affect memory, and the target itself may modify it.

The supplied representation expresses the final transfer as an indirect call followed by return, while also reporting uncertainty in recovery of the underlying transfer. The global-slot selection, its position after the stores, and the represented argument tuple are established at this evidence level. The precise machine-level call-versus-jump mechanics remain unresolved.

### Completion and failures

If the final invocation returns normally, the function returns without an explicit result.

There is no visible local error handler, retry, rollback, cleanup call, or release operation. No dependency result selects an alternative local path.

If the preparation dependency does not return, the stores and target selection are not reached through the visible sequential path. If the first store completes and the second fails, there is no visible operation that reverses the first. If the final invocation fails or does not return, no local recovery or normal completion is established.

These observations do not define platform fault behavior or dependency-internal recovery.

## Side effects

The established local effects, in order, include:

- Two four-byte writes through B.
- A read of the global function-pointer slot after those writes.
- Invocation of the target obtained from that read.

The preceding dependency calls may have additional memory, global-state, and resource effects. Their implementations are not supplied, so these effects cannot be restricted to the apparent record.

The global-slot read is not independent of the writes. If the written ranges overlap the slot, writing the inputs may change the eventual dispatch target or make its representation invalid. No separate protection, restoration, or validation of the slot is visible.

No synchronization, locking, memory barrier, or atomic-publication protocol is established. Sequential ordering within this function is not proof of safe communication with another thread, interrupt handler, or device.

Ownership and lifetime are **documented_only**:

- Returning B does not establish that F-d33f6d597e5d allocates storage or transfers ownership.
- Receiving B does not establish that F-unresolved-01 preserves or owns the storage.
- Passing B to the final target does not establish borrowing, copying, retention, consumption, or release.
- Absence of a visible local release proves neither a leak nor an ownership transfer.

No post-return guarantee is established for the contents or lifetime of the addressed memory.

## Dependencies

| Opaque function ID | Established call-site behavior | Unresolved properties |
|---|---|---|
| F-d33f6d597e5d | Called first with 24; its result supplies B | Operation, failure convention, extent and alignment of storage, ownership, global effects |
| F-unresolved-01 | Called next with B and 24; result unused | Operation, memory and resource effects, slot changes, validity of storage afterward |
| F-unresolved-02 | Target obtained from the global slot after both stores; invoked with 2, B, 5, 0; result unused | Actual identity, target contract, ownership, failure behavior, precise transfer mechanics |

F-unresolved-01 and F-unresolved-02 are document-local opaque identifiers. They are not recovered canonical identities. F-unresolved-02 denotes the selected target for an invocation, not a guarantee that every execution selects the same function.

No dependency is assumed to preserve globals or to own or release memory.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Ordinary record preparation

Assume B supports the required writes, the dependencies return, and no external modification interferes.

For inputs 0x11223344 and 0xAABBCCDD, the required order is:

1. F-d33f6d597e5d receives 24 and returns B.
2. F-unresolved-01 receives B and 24.
3. The first value is stored at B + 16.
4. The second value is stored at B + 20.
5. The global target slot is read.
6. The selected target receives 2, B, 5, and 0.

No particular individual-byte ordering is required by this evidence.

### Full-width input preservation

Under valid storage conditions, inputs 0xFFFFFFFF and 0x80000000 retain all their bits in the respective four-byte stores.

Narrowing to 16 bits, clearing a high bit, swapping the inputs, or converting them to Boolean values would violate the established field behavior.

### Preparation changes the dispatch target

Assume the global slot initially selects one target, and F-unresolved-01 replaces it with another valid target. Assume the input stores do not subsequently affect the slot and no other actor changes it.

The final invocation must select the replacement through the post-store slot read. Using a target cached before the preparation call would not satisfy this behavior.

The same ordering principle applies if the initial dependency changes the slot.

### A store aliases the dispatch slot

Assume a written range overlaps the global slot and the completed stores leave it containing a valid target representation different from its earlier contents. Assume no later external modification.

The subsequent slot read must observe the resulting slot contents under the target’s applicable memory semantics. The invocation must not use an earlier cached target.

This example does not assert a particular slot width or alignment. If the stores instead leave an invalid target representation, successful invocation is not guaranteed and no local validation is established.

### Preparation writes the apparent record

Assume F-unresolved-01 writes recognizable values throughout a writable 24-byte region beginning at B.

The two local stores subsequently replace the values at offsets 16–19 and 20–23. There is no corresponding local overwrite at offsets 0–15.

This criterion establishes order without requiring that the dependency actually initialize such a region.

### Zero base or non-returning dependency

If F-d33f6d597e5d returns zero, F-unresolved-01 still receives zero and 24. If it returns, the function proceeds toward the fixed-offset writes without a visible local failure check. Whether those writes complete is undocumented.

If F-unresolved-01 does not return, neither store nor the global-slot read is reached through the visible sequential path.

## Uncertainties

The following remain **documented_only**:

- Whether the initial dependency allocates storage or interprets 24 as a byte count.
- The preparation dependency’s operation and effects.
- Target byte order, alignment constraints, address-overflow semantics, and invalid-access behavior.
- Failure sentinels and valid address ranges.
- Global-slot contents, pointer width and representation, atomicity, and valid targets.
- Whether any particular execution aliases the slot with a store destination.
- Exact ABI-level widths and meanings of the final literal arguments.
- Dependency effects on memory, globals, ownership, and lifetime.
- Precise machine-level mechanics of the final transfer.
- Concurrency behavior, fault handling, and normal-return guarantees.

The global selection mechanism and late read position are established, not uncertainties. What remains unknown is the value obtained and its interpretation under unsupplied target and dependency contracts.

## Evidence

**Established static evidence:** the supplied representation shows two four-byte inputs; a call to F-d33f6d597e5d with 24; a second dependency call with its retained result and 24; ordered four-byte stores at byte offsets 16 and 20; and then target selection through a global function-pointer slot followed by an invocation represented with arguments 2, B, 5, and 0. No local validation, conditional error path, cleanup, or explicit result is shown.

**Inferred intent:** the repeated value 24 and stores ending at offset 23 suggest preparing a 24-byte record for further processing. This inference does not establish allocation, clearing, submission semantics, or ownership transfer.

**Evidence limits:** dependency implementations, global-slot contents, and runtime behavior were not supplied. The final indirect transfer carries an explicit recovery limitation. No runtime tests have been performed.
