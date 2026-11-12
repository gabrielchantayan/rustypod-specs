<!-- Trusted provenance: F-e1da3f6fb9ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e1da3f6fb9ed

## Purpose

F-e1da3f6fb9ed coordinates a byte-counted operation against a mutable, sector-oriented destination. It resolves destination state, selects existing or additional storage runs, invokes either a sector-oriented dependency or a buffered sequence, advances working cursors, and conditionally updates logical length.

A sector is 512 bytes. The apparent intent is a file-like write with optional append positioning. Actual data transfer, copying, synchronization, allocation, and persistence semantics are **inferred**, not established by the unavailable dependency implementations.

Checked failure is not transactional. The caller restores three saved fields on processing failure, but does not visibly reverse earlier dependency effects. Dependencies invoked after those restorations may change the fields again.

## Interface

### Arguments and return value

The function accepts three 32-bit arguments:

| Argument | Established use |
|---|---|
| Destination token | Opaque value passed to F-667717e3fb3e with selector 3. |
| Source address | Address-like value passed to transfer-related dependencies and advanced as processing proceeds. Its representation is signed 32-bit, but there is no negative-value rejection. |
| Requested byte count | Unsigned 32-bit count. |

The unsigned 32-bit return value is:

- Zero if the count or source address is zero.
- The original requested count after the successful completion path.
- A value loaded from an **unresolved 32-bit global** on destination-resolution failure or checked processing failure.

The failure global is not established as immutable. Its numeric value, initialization, and possible equality with successful return values are unknown.

Its read timing is part of the contract:

- Resolution failure: read the global **before** F-038b46411c0c receives status 9.
- Processing failure: read the global **after** F-591e5dd605b9 returns and **before** F-038b46411c0c receives status 28.

The captured value is returned even if the subsequent status call changes the global. There is no visible partial-count return.

### Observed layouts

The resolved state has these accessed fields:

| Byte offset | Width | Observed role |
|---|---:|---|
| 0 | 32 bits | Descriptor address. |
| 4 | 16 bits | Flags; bit value 8 selects append-related processing. |
| 8 | 32 bits | Byte position. |
| 12 | 32 bits | Allocation cursor. |
| 16 | 32 bits | Mapped sector base. |
| 20 | 32 bits | Assigned 1 when completion extends logical length. |
| 28 | 32 bits | Assigned a final boundary-comparison result. |

The descriptor supplies a 32-bit volume address at offset 0 and a 32-bit metadata address at offset 4. Metadata offset 28 contains the 32-bit logical length.

The volume fields accessed are:

| Byte offset | Width | Use |
|---|---:|---|
| 104 | 32 bits | Mask for extent alignment and position-derived offsets. |
| 120 | 16 bits | Identifier passed to synchronization-like and direct-transfer dependencies. |
| 460 | 16 bits | Unsigned sector-base increment. |
| 462 | 16 bits | Low eight bits used as a shift count. |

These are accessed layouts, not complete object declarations. Byte order and broader object validity are not supplied.

### Preconditions and ownership

Nontrivial processing requires all dereferenced addresses to support the accesses performed. For the inferred write interpretation, source storage must be readable for the portions selected by the caller. No address-range validation, overlap check, general overflow check, or validation of mask and run invariants is visible.

The caller does not explicitly free source storage, state, descriptor, or metadata. An object obtained from F-04b7df4e1a09 is passed to F-2b1ba6d8de9d after its update-like operation, including when that operation reports failure. This does not independently establish ownership transfer or release semantics.

## Behavior

### Entry, resolution, and saved state

A zero count or zero source address returns zero without invoking any dependency.

Otherwise, F-667717e3fb3e receives the destination token and selector 3. A null result selects resolution failure: capture the failure global, report status 9 through F-038b46411c0c, and return the captured value. Neither synchronization-like dependency is invoked on that path.

After successful resolution, the caller caches the volume address obtained through the descriptor. It invokes F-6443787f9616 with the volume’s 16-bit identifier; no result is checked.

It then reads logical length through the current descriptor and metadata pointers. If length is zero, it assigns zero to state offset 12. F-e657f684a3c8 is called next.

Only after that call does the caller save, in order:

1. Byte position.
2. Mapped sector base.
3. Allocation cursor.

These values are the processing-failure restoration point, not necessarily the entry values.

### Append-related processing

The caller reads the 16-bit flags and tests bit value 8. When that bit is set, it reads logical length again. A nonzero length causes F-5fd7b5f528b4 to receive the state address, an unresolved auxiliary argument, two zero arguments, and selector 2.

A signed 64-bit result equal to negative one selects processing failure. Every other result permits continuation and is followed by F-e657f684a3c8.

“Seek to end” is inferred intent. The auxiliary argument, positioning semantics, and effects on state remain unknown. The saved restoration point is not refreshed after this sequence.

### Boundary and requested-run arithmetic

Before the main loop, the caller computes a 32-bit boundary by adding logical length and the volume mask, then clearing mask-selected bits. This uses unsigned arithmetic, including possible wrap in the addition. It denotes conventional upward alignment only under an appropriate mask invariant.

Each main iteration computes a requested allocation-unit count from:

- Remaining bytes plus 511, shifted right by nine.
- The current position masked by both the volume mask and 65535, then shifted right by nine.
- The sum of those quantities, minus one.
- A right shift by the low eight bits of the shift field.
- A final addition of one for the dependency argument.

This requested-run calculation is unsigned 32-bit arithmetic. In particular, the addition of 511 can wrap; it must not be replaced unconditionally with unbounded ceiling division. Shift counts of 32 or greater have unresolved behavior.

The cached volume address is reused, but its fields are read again at their respective uses. Their values are not established as constant across dependency calls.

### Existing and additional runs

The caller compares current position and the maintained boundary as unsigned values.

Below the boundary, F-5a8a1a11a291 receives the state, cached volume address, current allocation cursor, address of an output-cursor slot, and requested allocation-unit count.

At or beyond the boundary, F-241dff0e9fd7 receives the cached volume address, the address of state offset 12, and the requested count.

Both results are represented as **signed 32-bit run values**. Only zero is rejected. A negative nonzero result proceeds; no positive-run validation is visible.

After a nonzero additional-run result:

1. F-2267038adc6d receives the volume, state, current allocation cursor, and run value.
2. An output cursor is formed from the then-current allocation cursor plus the run value minus one.
3. F-40e4fc419124 receives the descriptor’s two 32-bit contents.
4. If its result is zero, F-27e49f66c358 receives those contents, freshly read.
5. F-08426f0ea83d receives the volume and current allocation cursor; its result is assigned to state offset 16.
6. The maintained boundary is increased by the run value shifted by the configured count and multiplied by 512.

No failure result is checked for these subsequent calls. In particular, zero from F-40e4fc419124 selects an additional operation, not immediate failure.

The output-cursor addition, run-value left shift, and multiplication contain signed intermediates. Signed overflow and invalid shifts are unresolved; this sequence is not uniformly modulo-32-bit arithmetic.

### Direct-sector path

A sector-aligned position with at least 512 remaining bytes selects the direct path.

The caller derives a sector offset from the current position, volume mask, and low-16-bit restriction. Available sectors are formed from the signed run-value left shift followed by subtraction of the unsigned sector offset. The resulting available-sector quantity is unsigned.

The selected count is the smaller of available sectors and remaining whole sectors. If remaining whole sectors are **strictly fewer**, the output cursor is recalculated from the selected endpoint and current allocation cursor. Equality does not select that recalculation.

F-44c2d260d7a4 receives:

- The volume’s currently read 16-bit identifier.
- Current mapped sector base plus sector offset.
- Working source address.
- Selected sector count.
- A final zero argument.

These arguments establish a sector-transfer-shaped call, not that bytes are actually written. Zero return selects processing failure; every nonzero result is accepted.

After a nonzero return, the caller advances the working source by the selected count times 512, subtracts that amount from remaining bytes, and assigns the current position plus that amount to state offset 8. The position used by this assignment is read **after** the transfer dependency returns.

It then assigns the output cursor to state offset 12, calls F-08426f0ea83d, and assigns that result to offset 16. This call site explicitly presents only the volume argument; its complete contract is unresolved.

These assignments do not establish a net final position increase: dependencies before, during, and after them may modify state.

### Buffered path

A nonaligned position or fewer than 512 remaining bytes selects the buffered path.

Using the position captured at path selection, the caller obtains:

- A within-sector byte offset from the low nine bits.
- A selected byte count equal to the smaller of remaining bytes and bytes to the next sector boundary.
- A sector address from the current mapped sector base and masked position-derived sector offset.

F-04b7df4e1a09 receives the volume and sector address. Null selects processing failure.

For a nonnull returned address, the caller invokes:

1. F-2eceb10e9abc with returned address plus 24 plus within-sector offset, working source address, and selected byte count.
2. F-c7bbf7146083 with the returned address.
3. F-2b1ba6d8de9d with the returned address and argument 1.

The third call occurs whether the second returns zero or nonzero. The first call’s return is not checked.

A byte copy into a payload beginning at offset 24 is inferred from the first call’s arguments. Actual copying, buffer modification, overlap behavior, and payload validity are not established.

If F-c7bbf7146083 returns zero, processing fails after F-2b1ba6d8de9d. The caller has not yet performed this chunk’s source, remaining-count, or position advances, although dependencies may already have changed memory or state.

On nonzero update result, after F-2b1ba6d8de9d returns, the caller rereads position. It advances source and assigns that reread position plus the selected byte count to offset 8, then reduces remaining bytes.

If the newly computed position has no bits selected by the currently read volume mask, the signed run value is decremented:

- Nonzero remainder: increment sector base by the unsigned 16-bit stride, then increment allocation cursor by one.
- Zero remainder: install the output cursor and perform the refresh sequence through F-08426f0ea83d.

The run decrement and allocation-cursor increment contain signed arithmetic with unresolved overflow behavior.

Where the current run remains active, more than 511 remaining bytes lead directly to the direct-sector path without another run lookup. Exhausting the run takes the cursor-installation and refresh path, then returns to the loop condition. Boundary handling and refresh can occur even when the chunk consumed the final requested byte.

### Completion and processing failure

After remaining bytes reach zero, logical length is read through the then-current descriptor and metadata pointers. If its unsigned value is below current position, the caller writes 1 to state offset 20, then stores current position as logical length. Otherwise, neither write occurs; offset 20 is not cleared.

The caller next assigns state offset 28:

- Zero if current position is unsigned-less-than the maintained boundary.
- One otherwise, including equality.

It invokes F-591e5dd605b9 with the identifier reread from the cached volume, reports status 0 through F-038b46411c0c, and returns the original requested count.

Checked processing failure instead:

1. Restores saved position.
2. Restores saved sector base.
3. Restores saved allocation cursor.
4. Invokes F-591e5dd605b9 with the volume identifier.
5. Captures the failure global.
6. Reports status 28 through F-038b46411c0c.
7. Returns the captured value.

The caller omits successful length-extension and final boundary-marker assignments on this path. Neither restoration nor completion assignments imply that those values remain unchanged after subsequent dependencies.

## Side effects

Established caller writes include cursor and sector-base assignments, the conditional length-extension marker and logical length, and the completion boundary marker.

There is no direct source-memory write shown. The caller passes source-related addresses to dependencies whose behavior is unavailable.

Dependency effects may include storage changes, buffer changes, metadata updates, global changes, or synchronization effects. None is independently guaranteed. Consequently:

- Cursor restoration does not establish data rollback.
- Allocation-related calls precede later checked failures.
- The buffer sequence may have effects even when its update result is zero.
- Successful return does not establish persistence or durability.
- Descriptor relationships, volume fields, and state fields cannot be assumed preserved across calls.
- The failure global may change between failure occurrences or during synchronization exit.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-667717e3fb3e | Resolve token with selector 3; null is checked. |
| F-6443787f9616 | Entry-like call with a 16-bit identifier; result unchecked. |
| F-591e5dd605b9 | Exit-like call before success status or processing-failure capture. |
| F-e657f684a3c8 | State preparation before snapshot and after accepted append-related call. |
| F-5fd7b5f528b4 | Positioning-shaped call; signed 64-bit negative one is checked. |
| F-5a8a1a11a291 | Existing-run lookup with an output-cursor address; signed result checked only for zero. |
| F-241dff0e9fd7 | Additional-run request with allocation cursor by address; signed result checked only for zero. |
| F-2267038adc6d | Post-allocation call; result unchecked. |
| F-40e4fc419124 | Zero result selects F-27e49f66c358. |
| F-27e49f66c358 | Conditional descriptor-related operation. |
| F-08426f0ea83d | Result assigned as sector base; argument presentation differs by site. |
| F-44c2d260d7a4 | Sector-transfer-shaped call; zero is checked. |
| F-04b7df4e1a09 | Obtain an address associated with a sector; null is checked. |
| F-2eceb10e9abc | Three-argument copy-shaped call; result unchecked. |
| F-c7bbf7146083 | Buffer update-shaped call; zero is checked. |
| F-2b1ba6d8de9d | Post-update call with argument 1 on both update outcomes. |
| F-038b46411c0c | Receives status 9, 28, or 0. |

No dependency is assumed to preserve globals or state, own memory, or release memory.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Trivial request:** Zero count or zero source address returns zero without resolving the token or calling any dependency.

2. **Resolution-failure capture:** If resolution returns null and the failure global contains value A when read, status 9 is then reported. The return is A even if status reporting changes the global to B.

3. **Processing-failure capture:** On a checked processing failure, restoration assignments precede synchronization exit. If that exit changes the failure global to C, the subsequent capture returns C, regardless of a later status-28 change.

4. **Single aligned sector:** Assume no append positioning, a usable existing run covering at least one sector, valid arithmetic, and preservation of relevant state by dependencies. For 512 requested bytes at an aligned position, the direct call receives sector count 1 and final argument zero. After a nonzero return, the caller assigns position read after that call plus 512. A net increase of 512 additionally requires all relevant dependencies, including refresh, exit, and status reporting, to preserve position.

5. **Small buffered operation:** Given a selected within-sector offset of 100 and remaining count 20, a nonnull acquired address produces a call to F-2eceb10e9abc with destination address plus 124, working source address, and count 20. This does not prove that 20 bytes were copied.

6. **Buffered update failure:** If F-c7bbf7146083 returns zero, F-2b1ba6d8de9d still receives the acquired address and 1 before restoration. No buffer rollback is shown, and no actual prior buffer modification is established.

7. **Negative run result:** A run result of negative one passes the zero check. Subsequent outcome is documented_only where signed shifts or arithmetic are invalid; rejection as an allocation failure is not supported.

8. **Completion equality:** If position equals the maintained boundary at the final comparison, the caller assigns 1 to offset 28. Later dependency effects may change its final value.

9. **Length extension ordering:** If the completion length comparison succeeds, offset 20 receives 1 before logical length receives current position. If it fails, the caller performs neither assignment.

## Uncertainties

The following remain **documented_only**:

- Failure-global value, mutability rules, and collisions with successful counts.
- Actual transfer, copying, allocation, synchronization, status, ownership, and persistence semantics.
- Complete arguments for append-related positioning and the one-explicit-argument refresh.
- Required mask, stride, shift, cursor, and positive-run invariants.
- Initialization of the existing-run output cursor when later consumed.
- Signed overflow, negative signed left shifts, and shift counts of 32 or greater.
- Progress and termination when selected sector count is zero or runs are unusable.
- Address validity, aliasing, overlap, concurrent mutation, reentrancy, and interrupted operations.

Unsigned arithmetic observations do not resolve signed intermediate behavior.

## Evidence

This contract derives solely from the supplied static representation and opaque dependency mapping.

Established evidence comprises widths, offsets, explicit accesses and assignments, argument values, branch conditions, call ordering, result checks, arithmetic signedness, and return-value capture timing.

Write, append, copy, allocation, synchronization, and release terminology describes inferred intent only. No dependency implementations, platform execution rules resolving the arithmetic gaps, or runtime traces were supplied. No runtime tests have been performed.
