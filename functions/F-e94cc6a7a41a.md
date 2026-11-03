<!-- Trusted provenance: F-e94cc6a7a41a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e94cc6a7a41a

## Purpose

F-e94cc6a7a41a either delegates an operation using its five original arguments or executes an ordered sequence of dependency calls and memory updates.

An initial dependency selects the path by returning zero or nonzero. On the nonzero path, a replaceable local pointer slot supplies the address for three separate record accesses: one read and two writes. **Each access uses the slot’s value at that access. The three accesses are not guaranteed to address the same record.**

The function returns no value. Its observable behavior consists of dependency calls, memory accesses, and effects produced by those dependencies.

Replacing a resource and updating associated metadata is a possible interpretation, but remains inferred intent. The supplied evidence does not establish allocation, release, copying, locking, or the semantic meaning of either accessed record word.

## Interface

The function accepts five arguments:

| Argument | Established use |
|---|---|
| A1 | An integer-valued address base. A four-byte word at byte offset 24 is read before the initial dependency and, on the nonzero path, again before the final dependency. |
| A2 | An opaque 32-bit value forwarded to dependencies. |
| A3 | An opaque 32-bit value passed twice to the initial dependency and once to the final nonzero-path dependency. |
| A4 | An original pointer value used to initialize a local pointer slot. The original value is also passed independently to other dependencies. |
| A5 | An opaque 32-bit value passed to dependencies and written through the local pointer slot on the nonzero path. |

Four-byte data quantities in the supplied representation are treated here as 32-bit words. A1 and the initial dependency’s result have integer representations; their exact machine widths, the pointer width, and address-overflow behavior are not independently established by the excerpt.

The local pointer slot holds a pointer to four-byte words. Its address is passed to the initial dependency. The slot is distinct from any pointer variable in the caller merely used to supply A4.

For each direct record access, the current pointer designates these locations:

| Byte offset from the current pointer | Width | Use |
|---|---:|---|
| 4 | 32 bits | Read before the first nonzero-path dependency. |
| 4 | 32 bits | Written after the external dependency returns. |
| 0 | 32 bits | Written after the offset-4 write. |

These are separate access descriptions, not a guarantee of a single record identity. They also do not establish a complete structure layout, allocation size, alignment requirement, or ownership convention.

### Preconditions

For completion without an invalid access:

- A1 must support the required four-byte reads at byte offset 24.
- On the nonzero path, the pointer slot’s value at the offset-4 read must support that read.
- Its value at the offset-4 write must support that write.
- Its value at the offset-0 write must support that write.
- Dependencies must accept their arguments and return normally wherever subsequent behavior is required.
- Any additional dependency-specific validity requirements must hold.

Validity of the pointer immediately after the initial dependency is not sufficient to establish validity of the later writes. Intervening dependencies may modify the slot through retained access. The offset-4 write may itself affect the slot through aliasing.

The function performs no explicit null, bounds, alignment, ownership, or argument checks. On the zero path, it does not directly dereference A4 or the replacement pointer, although dependencies may do so.

## Behavior

### Initial decision

The function initializes a local pointer slot with A4, then reads the four-byte word at A1 plus 24 bytes.

It invokes F-148a31e02bc5 with these arguments, in order:

1. The word just read.
2. A2.
3. A3.
4. The address of the local pointer slot.
5. A3 again.

The fourth argument permits replacement of the pointer used for later direct record accesses. The evidence does not establish that access to this slot ends when F-148a31e02bc5 returns. A dependency may retain its address and make it available to subsequent dependencies.

The return value is tested only for zero versus nonzero. No additional distinction among nonzero values is made.

### Zero-result path

If F-148a31e02bc5 returns zero, the function invokes F-3f05bb325e5c with A1, A2, A3, A4, and A5, in their original order.

It then returns without a value.

On this path:

- A replacement value in the local pointer slot is not substituted for A4 in the delegation call.
- No direct record read or write follows the decision.
- The second read at A1 plus 24 bytes does not occur.
- None of the nonzero-path dependencies is invoked.

These restrictions describe this function’s direct behavior. They do not limit side effects of either dependency executed on this path.

### Nonzero-result path

If F-148a31e02bc5 returns nonzero, the function performs the following ordered behavior.

1. **Read through the current slot.** It uses the pointer slot’s value at this access to read the four-byte word at offset 4. It invokes F-8349c252c168 with that word and the integer constant 26.

2. **Obtain a replacement value.** After F-8349c252c168 returns, it invokes F-638beded7699 with A5 and 26. The resulting 32-bit value is retained for later use.

3. **Invoke the no-argument dependency.** It invokes F-14590d82970d with no explicit arguments. Any return value is unused.

4. **Invoke the external dependency.** It invokes D-EXT-01 with the retained 32-bit value, original A4, and A5. Any return value is unused. D-EXT-01 is a document-local opaque dependency ID because no published opaque ID was supplied for this dependency.

5. **Write through the current slot at offset 4.** After D-EXT-01 returns, the function uses the pointer slot’s value at this write to store the retained 32-bit value at offset 4. This pointer need not equal the pointer used for the earlier read.

6. **Write through the current slot at offset 0.** After the offset-4 write, the function uses the slot’s value at this next access to store A5 at offset 0. This pointer need not equal either earlier pointer.

7. **Reread the base field and notify.** It reads the four-byte word at A1 plus 24 bytes again, then invokes F-1488711eb836 with the newly read word, A2, and A3.

The function then returns without a value.

### Pointer-slot changes and aliasing

The slot’s address escapes to F-148a31e02bc5. Normal dependency returns and valid memory do not imply that subsequent dependencies preserve its contents.

Consequently:

- F-8349c252c168 can receive a word from one record while the later offset-4 write targets another.
- Any intervening dependency with access to the slot may redirect the later writes.
- D-EXT-01 still receives original A4, regardless of changes to the slot.
- The offset-4 write can itself change the slot if its destination overlaps the slot’s storage.
- The offset-0 write uses the resulting slot value, not a guaranteed cached copy of the pointer used for the offset-4 write.

For overlapping storage, the resulting pointer depends on the pointer representation and the bytes written. If that produces an unusable pointer, the subsequent access is not guaranteed to complete. The contract does not impose a disjointness condition that the supplied evidence lacks.

### Ordering requirements

The following ordering is established:

- The offset-4 read precedes F-8349c252c168.
- F-8349c252c168 precedes F-638beded7699.
- F-638beded7699 precedes F-14590d82970d, which precedes D-EXT-01.
- D-EXT-01 returns before either direct record write.
- The offset-4 write precedes the offset-0 write.
- Both writes precede the second base-field read and F-1488711eb836.

The second base-field read is a fresh observation. Dependencies or either direct write may have changed its storage through aliasing.

No atomic two-word update or explicit synchronization is established.

## Side effects

The function directly:

- Initializes a local pointer slot with A4.
- Exposes the slot’s address to F-148a31e02bc5.
- Reads four bytes at A1 plus 24 bytes before the initial dependency.
- On the nonzero path, reads four bytes at offset 4 through the then-current slot value.
- Later writes four bytes at offset 4 through the slot value current at that write.
- Subsequently writes four bytes at offset 0 through the slot value current at that write.
- Rereads four bytes at A1 plus 24 bytes before the final dependency.

These accesses may overlap one another or the pointer slot where the execution environment permits such aliasing. Describing them as record accesses does not imply disjoint storage.

Dependency effects are not bounded by this excerpt. In particular, no dependency is assumed to preserve globals or an escaped pointer slot merely because its explicit argument list omits them.

### Ownership and lifetime

No ownership transfer is established.

- Passing the read word to F-8349c252c168 does not prove release.
- Receiving a value from F-638beded7699 does not prove allocation.
- Passing original A4 to D-EXT-01 does not establish copying, consumption, retention, or release.
- Storing a value through the pointer slot does not establish ownership by the destination.

The pointer slot is local to this invocation. Retained access during the invocation can affect later accesses. Whether a dependency retains the address beyond return, and whether any such use is permitted, remains unresolved.

### Errors and interrupted execution

There is no explicit error return, cleanup branch, rollback, or exception handling.

Zero from F-148a31e02bc5 means delegation for this function; it is not established to mean failure.

The result of F-638beded7699 is not checked. Zero is passed to D-EXT-01 and stored by the offset-4 write if execution reaches those operations.

If a dependency does not return normally, or an access cannot complete, later behavior is not guaranteed. Earlier effects are not explicitly undone. In particular, a completed offset-4 write is not rolled back if the subsequent offset-0 write fails.

## Dependencies

| Opaque ID | Established call-site contract |
|---|---|
| F-148a31e02bc5 | Receives the initial base-field word, A2, A3, the local pointer-slot address, and A3 again. Its zero/nonzero result selects the path. It can modify the slot; retention and later access are not ruled out. |
| F-3f05bb325e5c | Receives all five original arguments only on the zero path. |
| F-8349c252c168 | Receives the word read at offset 4 through the slot’s value at that read, followed by 26. Any return value is unused. |
| F-638beded7699 | Receives A5 and 26. Its 32-bit result is passed to D-EXT-01 and later stored through the then-current pointer slot. |
| F-14590d82970d | Receives no explicit arguments. Runs between F-638beded7699 and D-EXT-01. Any return value is unused. |
| D-EXT-01 | Receives the retained 32-bit result, original A4, and A5. Its published identity and internal behavior are unknown. |
| F-1488711eb836 | Receives the freshly reread base-field word, A2, and A3 after both direct writes. Any return value is unused. |

These descriptions establish invocation behavior, not resource-management or state-preservation guarantees.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. Each assumes normal dependency returns and valid accesses unless stated otherwise. Those assumptions alone do not imply pointer-slot preservation.

### Delegation after replacement

F-148a31e02bc5 changes the local slot and returns zero.

F-3f05bb325e5c must receive original A4. No direct record access follows the decision, and the final nonzero-path dependency is not invoked.

### Stable slot selecting a different record

F-148a31e02bc5 selects record R, distinct from the storage originally designated by A4, and returns nonzero. **Assume the slot continues to designate R through all three direct record accesses, including across dependencies and the first write.**

If the offset-4 read yields X and F-638beded7699 returns Y:

- F-8349c252c168 receives X and 26.
- D-EXT-01 receives Y, original A4, and A5.
- After D-EXT-01 returns, R’s offset-4 word is written with Y.
- R’s offset-0 word is then written with A5.

Without the explicit slot-preservation assumption, the two destination claims do not follow.

### Dependency redirects the write

The slot designates R at the offset-4 read. An intervening dependency changes it to S, and it continues to designate S through both writes.

F-8349c252c168 receives the word read from R. Both writes target S, in offset-4 then offset-0 order. D-EXT-01 still receives original A4.

### First write redirects the second write

At the offset-4 write, the slot designates an address whose offset-4 destination overlaps the slot itself.

If storing the retained result changes the slot to a valid pointer T, the following offset-0 write stores A5 through T. The function does not guarantee that this second write uses the pre-write pointer.

If the overlap instead produces an invalid pointer, successful completion of the second write is not guaranteed.

### Changed base-field value

The initial base-field read yields U. The same location contains V when the final read occurs.

F-148a31e02bc5 receives U. On the nonzero path, F-1488711eb836 receives V, not a cached U.

### Zero replacement value

F-638beded7699 returns zero.

F-14590d82970d and D-EXT-01 are still invoked, with zero as D-EXT-01’s first argument. If execution continues, zero is stored at offset 4 through the slot value current at that write. A5 is subsequently stored through the slot value current at the offset-0 write.

### Zero A5

A5 is zero and the initial result is nonzero.

Zero is passed to F-638beded7699 and as D-EXT-01’s third argument. The offset-0 write stores zero. There is no direct zero-value shortcut.

### Final write aliases the base field

At the offset-0 write, the slot designates the storage at A1 plus 24 bytes. Assume ordinary sequential execution and no intervening modification before the final read.

The write stores A5 there, so the final dependency receives A5 as its first argument. This scenario constrains the pointer at the actual offset-0 access, not merely after the initial dependency.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of A2, A3, A5, constant 26, and the record words.
- Whether any word represents a pointer, size, count, or resource.
- Complete layouts, allocation sizes, alignment rules, exact integer and pointer widths, and address-arithmetic overflow behavior.
- Dependency ownership, retention, resource-management, and global-state effects.
- Which dependencies can access the escaped slot, and whether they actually modify it.
- Whether any actual execution aliases a record destination with the slot.
- Pointer-representation consequences of a partially overlapping write.
- D-EXT-01’s published identity and operation.
- Concurrency, reentrancy, synchronization, fault handling, and exceptional control flow.
- Permitted use of the slot’s address after this invocation ends.

Resource replacement remains inferred intent, not a basis for strengthening these guarantees.

## Evidence

This contract derives from static inspection of the supplied function representation and dependency-ID mapping.

Established evidence includes the zero/nonzero branch, argument order, repeated A3 argument, constant 26, separate base-field reads, original-A4 forwarding, escaped local pointer slot, separate pointer-slot-based record accesses, dependency order, and ordered four-byte writes.

The supplied representation supports no guarantee that the pointer selected by the initial dependency remains unchanged. In particular, intervening dependency effects and aliasing by the first write must not be excluded without an explicit assumption.

No runtime tests have been performed. Dependency implementations, broader type declarations, caller behavior, and runtime memory layouts were not supplied.
