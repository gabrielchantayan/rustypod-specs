<!-- Trusted provenance: F-7dee43db055e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7dee43db055e

## Purpose

F-7dee43db055e coordinates an ordered, synchronous operation over a caller-supplied record, two opaque arguments, resources identified by 5 and 6, and externally selected shared state.

Established behavior consists of checked dependency calls, progress-field updates, interface acquisition, conditional object writes, cleanup, a mandatory waiting loop, and completion publication. Each checked main-stage result is interpreted as zero for continuation or nonzero for failure. Failure skips the remaining main stages but does not skip waiting or completion publication.

Resource preparation, cross-resource transfer, attachment, and finalization are inferred descriptions of call roles, not established domain semantics. Dependency ownership rules and progress arithmetic remain unresolved.

All memory-state descriptions below distinguish this function’s explicit accesses from possible dependency effects. No dependency is assumed to preserve globals, records, interfaces, objects, or previously written fields.

## Interface

### Arguments and return

The function receives:

1. A record address, used as a base for 32-bit reads and embedded-region addresses.
2. Argument A, an opaque 32-bit value.
3. Argument B, an opaque 32-bit value.

There is no explicit return value and no visible argument validation. Internal failure status is reduced to completion byte 2 rather than returned.

The supplied representation uses 32-bit words and pointer-valued fields. Offsets below are byte offsets; a passed embedded-region address does not establish that region’s extent.

### Record accesses

| Offset | Established use |
|---|---|
| 0 | Read as a 32-bit argument to each resource-operation call. |
| 4 | Address passed to F-280a6058ff6e. |
| 24 | Address passed to F-2defcd314a78 and, on the trailing success path, F-aef4bc85d65b. |
| 36 | Read as a 32-bit argument to each resource-operation call. |
| 40 | Read as a 32-bit argument to each resource-operation call. |
| 44 | Read as a 32-bit argument only for resource 5. |
| 92 | Address written into an object field and later passed to F-578012c4f121. |

These accesses do not establish a complete record layout. Record values used by separate calls are not entry snapshots; dependencies may change them between calls. The evaluation order among argument reads for an individual call is not established.

### Shared and external state

An external pointer selects shared storage. Its entry access is significant:

- The function first reads and captures the external pointer.
- It reads that external pointer again to select the destination for the initial completion-byte clear.
- It uses the first, captured address for the initial progress clear and all subsequent explicit shared-state accesses.

The two reads need not produce the same address. No concurrency or pointer-stability guarantee is assumed.

Relative to the captured shared address:

| Offset | Width | Established use |
|---|---:|---|
| 1 | 8 bits | Completion publication: 1 for success, 2 for failure. |
| 4 | 32 bits | Read for an indirect operation’s argument. |
| 8 | 32 bits | Progress-like field, repeatedly read and written. |

The initial byte clear targets offset 1 relative to the **second** external-pointer reading, not necessarily the captured address.

Three additional external 32-bit values are accessed: two milestone values copied into progress, and a threshold captured at specified points for the final loop. These are reads at those points, not immutable configuration snapshots.

### Preconditions and outputs

Normal execution requires valid storage for each access actually reached, valid interface and dispatch-table pointers, and dependencies that return normally. The two interface pointers are used without explicit null checks. Only the object returned by the first indirect operation is checked for zero.

Observable outputs include call ordering, skipped stages, explicit shared-state and object writes, and dependency effects. Invalid addresses, invalid dispatch entries, faults, and non-returning dependencies have no visible recovery path.

## Behavior

### Entry and progress updates

Entry performs, in order:

1. Capture the external shared-state pointer.
2. Read that external pointer again and clear the completion byte at its offset 1.
3. Clear the 32-bit progress field at offset 8 of the captured address.
4. Call F-bfade2068003 with 31 and 0, without checking a result.

A normal **progress update** means reading the current 32-bit progress field, calling F-a83966d8599e with those bits and `0x3f800000`, then writing the returned 32-bit bits to progress.

Although `0x3f800000` encodes binary32 1.0, the helper’s arithmetic is unknown. Neither integer increment nor floating-point addition is established. Each ordinary update reads progress afresh; dependency modifications can therefore affect subsequent inputs.

### Seven preliminary stages

The stages below execute in order. **Immediately after each checked call returns, the function reads the external threshold, before testing the result and before any success-path progress update.** This read occurs on both zero and nonzero results.

A nonzero result leaves the main sequence using that just-read threshold. A zero result performs one progress update and continues. Successful-stage threshold readings still occur even when later readings replace them.

| Stage | Checked call and arguments |
|---|---|
| 1 | F-917058744226: record address, Argument A. |
| 2 | F-2defcd314a78: record address plus 24, Argument B. |
| 3 | F-26dffdd2d0c4: record words at offsets 0, 36, 40, and 44, together with Argument A and resource identifier 5. Argument order is offset-0 word, Argument A, offset-36 word, offset-40 word, offset-44 word, 5. |
| 4 | F-7ee4ad4490fb: record address. |
| 5 | F-a2c7a33ab4bc: record address, 5, address of a 32-bit output slot. |
| 6 | F-3b23b10763c1: record address, Argument B, address of another 32-bit output slot. |
| 7 | F-26dffdd2d0c4: offset-0 word, Argument B, offset-36 word, offset-40 word, literal 0, 6. |

The stage-5 output is later passed to the cross-resource call; the stage-6 output is later passed to preparation. Neither output slot has a visible prior initialization. Zero results do not independently prove that the dependencies initialized their outputs.

Before stage 3, two local 32-bit slots are assigned 0 and 1. Those assignments are repeated before stage 7. Their addresses are not explicit arguments to either resource-operation call in the supplied representation. One slot later supplies preparation output storage; the other is reset and used for acquisition cleanup.

### Acquisition and object preparation

After all seven stages succeed:

1. Reset the acquisition/cleanup slot to 0.
2. Call F-061d5c1fbc1d with 6 and that slot’s address; retain the returned first interface pointer.
3. Perform a progress update without checking the acquisition return.
4. Call F-1ec05f1e044b with the first interface and 0; retain the second interface pointer.
5. Invoke the first interface’s dispatch entry at byte offset 16, passing the first interface itself.

Dispatch offsets refer to the table reached through the interface’s first pointer-sized field. Interface and table validity are unchecked.

If the indirect result is zero, set internal failure status to 2 and proceed to acquisition cleanup.

Otherwise, treat the result as an object address and call F-696b25f8f663 with the record address, the saved stage-6 output, and the preparation-output slot’s address. That slot was explicitly set to 0 before stage 7; it is not explicitly reset immediately before this preparation call.

A nonzero preparation result goes to acquisition cleanup. On zero, call F-280a6058ff6e with record address plus 4, resource identifier 6, and the preparation output. Retain its opaque 32-bit result.

The following sequence then occurs:

1. Read progress and call F-a83966d8599e with it and `0x3f800000`; retain the result without writing progress.
2. Write byte 1 at object offset 88.
3. Write the 32-bit record-plus-92 address at object offset 92.
4. Call F-a83966d8599e with the retained result and `0x3f800000`.
5. Write the second result to progress.

There is no explicit progress write between these two helper calls. The second call uses the first result, not a new progress read. Either helper may have other effects under its unknown contract.

Next, invoke the second interface’s dispatch entry at byte offset 148 with:

- The second interface itself.
- The current 32-bit value at captured shared-state offset 4.
- The retained result from F-280a6058ff6e.

A nonzero result goes to acquisition cleanup. The object writes precede this invocation, but their persistence is not guaranteed: intervening or subsequent dependencies may alter the object. This function performs no explicit rollback.

### Cross-resource stages and cleanup

After the second indirect operation returns zero:

1. Perform a progress update.
2. Call F-935ebf2e6435 with 6.
3. On zero, read the first external milestone and copy its 32 bits to progress.
4. Call F-546092ee8f58 with the record address, 5, 6, and the saved stage-5 output.
5. On zero, read the second external milestone and copy its 32 bits to progress.
6. Call F-d2c3f7647481 with 5.
7. On zero, perform a progress update, then call F-a16e3cb9c0ab with 5.
8. On zero, perform a progress update, then call F-578012c4f121 with record address plus 92 and the acquired object address.
9. On zero, perform a progress update, then call F-3e371ea4ac82 with the acquisition/cleanup slot’s address.

Any nonzero checked result in this sequence skips its remaining success stages and invokes the same cleanup dependency. The zero-object and preparation-failure paths also invoke it.

For these failure paths, the external threshold is read **after cleanup returns**, replacing the preliminary threshold capture, and execution enters waiting.

Cleanup’s result is unchecked. Its memory effects, ownership effects, and interpretation of the slot are unknown.

### Trailing stages after successful attachment

After F-578012c4f121 returns zero, its subsequent progress update and cleanup occur before these trailing stages:

1. Call F-d2c3f7647481 with 6; immediately read the external threshold, then test the result.
2. On zero, perform a progress update, call F-a16e3cb9c0ab with 6, immediately read the threshold, then test the result.
3. On zero, perform a progress update, call F-aef4bc85d65b with record address plus 24, immediately read the threshold, then test the result.
4. On zero, perform one more progress update, then read the threshold again.

A nonzero result skips later stages and enters waiting with the threshold captured immediately after that failed call. Complete success uses the final reading after the final progress update.

There is no second cleanup after these trailing stages. Along normal control flow reaching acquisition, cleanup is called once before waiting.

### Mandatory waiting loop

Every normal path reaches waiting with an internal status and a captured 32-bit threshold. The threshold is not reread within the loop.

Each iteration:

- Calls the delay dependency with 500.
- Reads progress as a signed 32-bit value.
- Calls F-a83966d8599e with those same progress bits and `0x3f800000`.
- Stores the returned bits to progress.
- Repeats if the signed value read **before** the helper call is less than the captured threshold interpreted as signed 32-bit.

At least one iteration occurs. The comparison is signed integer comparison, not unsigned or floating-point comparison, and does not use the helper’s newly returned value.

The delay occurs before the progress read, so delay-side changes can influence that iteration. Helper-side changes do not replace the saved comparison operand. Termination is not guaranteed; no explicit timeout or iteration bound exists. The delay argument’s units are unknown.

### Completion publication

After waiting terminates, the function explicitly publishes:

- For zero internal status: completion byte 1, followed by progress equal to the captured threshold bits.
- For nonzero status: completion byte 2, followed by progress bits `0x3f800000`.

Both writes use the originally captured shared-state address. The byte write precedes the 32-bit write; atomic publication is not established.

The function then calls F-85722f62e20a with 31 and 1000, ignoring its result. These are publication writes **before the final dependency call**, not guarantees about memory at return. That dependency may change the published fields or other state.

## Side effects

Direct writes cover the initial clears, progress updates, milestone copies, completion publication, and the two conditional object-field writes. Dependencies receive caller-region addresses, local output addresses, interfaces, and an object address; their possible mutations are not bounded by this function.

No direct allocation or release is established. Acquisition and cleanup are descriptive roles, not proof of ownership transfer, reference counting, destruction, or memory release.

Writing record-plus-92 into the object creates a pointer relationship at that write. Neither continued retention nor the required record lifetime is established. Cleanup may affect the object or interfaces; subsequent use does not prove preservation by cleanup.

No explicit rollback restores earlier record, resource, object, or shared-state effects. This absence does not imply that dependencies leave those effects intact.

Aliasing, concurrent mutation, synchronization, and reentrancy guarantees are unavailable. Distinct described fields or roles must not be assumed to imply disjoint underlying storage.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-bfade2068003 | Entry call with 31 and 0. |
| F-917058744226 | First checked record operation. |
| F-a83966d8599e | Produces progress-update bits. |
| F-2defcd314a78 | Checked operation on record-plus-24. |
| F-26dffdd2d0c4 | Checked resource-5 and resource-6 operations. |
| F-7ee4ad4490fb | Checked record-only operation. |
| F-a2c7a33ab4bc | Supplies the saved cross-resource input. |
| F-3b23b10763c1 | Supplies the saved preparation input. |
| F-061d5c1fbc1d | Returns the first interface; receives the cleanup slot. |
| F-1ec05f1e044b | Returns the second interface. |
| F-696b25f8f663 | Checked preparation with output storage. |
| F-280a6058ff6e | Supplies the later indirect operation’s opaque argument. |
| F-935ebf2e6435 | Checked resource-6 operation before the first milestone. |
| F-546092ee8f58 | Checked operation receiving both resource identifiers. |
| F-d2c3f7647481 | Checked operation separately applied to 5 and 6. |
| F-a16e3cb9c0ab | Subsequent checked operation for each resource. |
| F-578012c4f121 | Checked operation receiving record-plus-92 and the object. |
| F-3e371ea4ac82 | Cleanup call receiving the cleanup slot’s address. |
| F-aef4bc85d65b | Last checked main-stage operation. |
| F-85722f62e20a | Final call after publication, with 31 and 1000. |

Opaque IDs were not supplied for the delay dependency or the indirect dependencies at dispatch offsets 16 and 148. Their identities and deeper contracts are documented_only.

## Acceptance examples

These are conditional obligations derived from static evidence, not executed tests. They describe this function’s accesses and calls, not dependency-preserved final memory.

### Different entry-pointer readings

If the first external-pointer read yields one address and the second yields another, the completion clear targets the second address plus 1. The initial progress clear and subsequent explicit shared accesses target the first address. No later explicit completion clear at the first address is implied.

### Successful preliminary stage

If stage 1 returns zero, the external threshold is read before its progress update. If the progress helper changes the external threshold, that does not retroactively change this reading. A later stage performs its own threshold read.

### Immediate failure

If F-917058744226 returns nonzero, the threshold is read immediately afterward. No later main-stage call or acquisition cleanup is issued. If waiting terminates, the function writes completion 2 and progress `0x3f800000`, then invokes the final dependency; return-time contents remain unspecified.

### Missing object

If the preliminary stages succeed but dispatch offset 16 returns zero, internal status becomes 2. Cleanup is called once, then the threshold is read. Preparation, object writes, attachment, and trailing resource-6 stages are skipped.

### Failure after object writes

If preparation succeeds and dispatch offset 148 fails, this function has already issued the byte-1 and record-plus-92 pointer writes. It performs no explicit rollback. Their contents at cleanup, publication, or return are not guaranteed.

### Failure after attachment

If attachment succeeds but F-d2c3f7647481 for resource 6 fails, cleanup has already been called. The threshold is read after that failed call, cleanup is not repeated, and later trailing stages are skipped.

### Complete success

If all checked stages return zero, required accesses are valid, calls return, and waiting terminates, the function writes completion 1 followed by the captured threshold bits to progress before F-85722f62e20a. This does not guarantee those contents at return or equality with the external threshold then.

### Loop boundaries

If the first post-delay progress reading is signed-greater-than-or-equal to the captured threshold, exactly one iteration occurs, including its helper call and write.

If that reading has bits `0x80000000` and the captured threshold is zero, the comparison requires another iteration because the progress operand is negative. The helper’s returned bits do not alter that iteration’s comparison.

## Uncertainties

The following remain documented_only:

- Domain meanings of the arguments, resources, interfaces, and operations.
- Progress-helper arithmetic, including overflow or special-value behavior.
- Milestone and threshold values, intended units, and delay units.
- Complete layouts, storage extents, alignment requirements, and aliasing constraints.
- Output initialization guarantees on dependency success.
- Ownership, retention, release, and required lifetimes.
- Dependency mutation of globals, record fields, dispatch tables, objects, or shared storage.
- Synchronization, atomicity, concurrency, and reentrancy guarantees.
- Recovery from invalid accesses, dependency faults, or nontermination.
- Memory contents at return after the final dependency call.

## Evidence

Established behavior derives from the supplied static representation’s explicit accesses, widths, argument construction, call order, branch conditions, dispatch offsets, and signed loop comparison.

The entry’s two external-pointer reads, threshold reads on both successful and failed preliminary calls, and publication before the final dependency call are observable ordering facts. No preservation contract is supplied for unknown callees.

Domain descriptions are inferred intent, not established semantics. No runtime tests have been performed.
