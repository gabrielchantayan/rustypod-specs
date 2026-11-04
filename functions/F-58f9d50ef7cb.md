<!-- Trusted provenance: F-58f9d50ef7cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-58f9d50ef7cb

## Purpose

F-58f9d50ef7cb selects one of two argument configurations, invokes F-793a94891814 once, and applies a fixed sequence of sparse writes to the storage identified by that dependency’s return value.

The selection depends only on whether the second input is zero. Both configurations lead to the same subsequent write offsets and byte constants. The full second input is also passed to the dependency without normalization.

The established behavior is configuration selection followed by partial initialization or modification of a returned region. Allocation, object construction, record typing, and registration are possible interpretations, not established purposes. In particular, the available evidence does not establish whether the returned storage is new, shared, already initialized, or owned by the caller.

## Interface

### Inputs

The function has two inputs:

| Input | Represented width and interpretation | Use |
|---|---|---|
| First input | An uninterpreted 32-bit value | Forwarded as the first argument to F-793a94891814 |
| Second input | An integer represented as a 32-bit signed value in this contract | Compared with zero, then forwarded unchanged as the sixth argument to F-793a94891814 |

No pointer interpretation, permitted-value enumeration, or ownership meaning is established for the first input.

Only zero versus nonzero affects the local branch. Positive and negative nonzero second inputs take the same branch, but their distinct values remain observable to the dependency.

The listing represents the second input as an integer rather than an explicitly sized integer type. The 32-bit interpretation is consistent with the supplied representation; independent ABI confirmation is unavailable.

### Output

There is no explicit return value. The dependency’s returned address is used internally and is not returned by this function.

Normal completion means the dependency returned and all thirteen specified writes completed. No status code or local failure result is provided.

### Referenced global values

The contract uses the following role-based labels for global 32-bit source cells. These labels are specification notation, not source identifiers.

| Label | Role |
|---|---|
| Selection-zero | Third dependency argument when the second input is zero |
| Selection-nonzero | Third dependency argument when the second input is nonzero |
| Word-0 | Value written at destination byte offset 0 |
| Word-24 | Value written at destination byte offset 24 |
| Word-32 | Value written at destination byte offset 32 |
| Word-36 | Value written at destination byte offset 36 |
| Word-44 | Value written at destination byte offset 44 |
| Word-60 | Value written at destination byte offset 60 |
| Word-120 | Value written at destination byte offset 120 |
| Word-132 | Value written at destination byte offset 132 |

The contents of these cells are not supplied. Their values must not be replaced with guessed constants, addresses, or semantic types.

## Behavior

### 1. Select dependency arguments

If the second input is zero:

- The second dependency argument is 23.
- Selection-zero is read, and its 32-bit value becomes the third dependency argument.
- Selection-nonzero is not read by this function.

If the second input is nonzero:

- The second dependency argument is 24.
- Selection-nonzero is read, and its 32-bit value becomes the third dependency argument.
- Selection-zero is not read by this function.

The selected global value is obtained before the dependency call. It is the value obtained by that read, not a value refreshed after the dependency returns.

### 2. Invoke the dependency

F-793a94891814 is called exactly once, with six arguments in this positional order:

| Position | Argument |
|---|---|
| 1 | First input, unchanged |
| 2 | 23 for a zero second input; otherwise 24 |
| 3 | Value read from the selected global cell |
| 4 | 9 |
| 5 | 55 |
| 6 | Second input, unchanged |

This table specifies argument positions, not a machine-level order of argument evaluation or register assignment.

There is no local retry, alternate dependency call, or fallback branch.

The dependency’s return value is treated as the base address of writable storage. If the dependency does not return normally, the subsequent operations described here are not reached.

### 3. Read and retain the offset-24 value

After the dependency returns, Word-24 is read into a retained 32-bit value.

This read precedes both the read of Word-0 and the first destination write. Consequently:

- Changes to Word-24 made by the dependency before returning can affect the value subsequently written at offset 24.
- Changes to Word-24 after this read do not alter the retained value used for that write.
- If the first destination write overlaps Word-24, that overlap does not replace the already retained value.

No assumption is made that the dependency preserves any referenced global cell.

### 4. Perform the destination writes

The function performs the following writes in the listed order. Offsets are in bytes relative to the dependency’s returned base address.

| Order | Byte offset | Width | Value |
|---:|---:|---:|---|
| 1 | 0 | 32 bits | Value read from Word-0 |
| 2 | 24 | 32 bits | Previously retained value from Word-24 |
| 3 | 28 | 8 bits | 9 |
| 4 | 32 | 32 bits | Value read from Word-32 |
| 5 | 36 | 32 bits | Value read from Word-36 |
| 6 | 40 | 8 bits | 10 |
| 7 | 44 | 32 bits | Value read from Word-44 |
| 8 | 60 | 32 bits | Value read from Word-60 |
| 9 | 64 | 8 bits | 12 |
| 10 | 120 | 32 bits | Value read from Word-120 |
| 11 | 124 | 8 bits | 17 |
| 12 | 132 | 32 bits | Value read from Word-132 |
| 13 | 136 | 8 bits | 18 |

Except for Word-24, each word-valued source is read for its corresponding write at that point in the sequence. The function does not establish a single snapshot of all global source cells.

The one-byte writes affect only the byte at their specified offset. They do not establish the contents of the next three bytes.

The function performs no explicit destination reads. It neither clears nor initializes the entire intervening region.

### Layout and access requirements

The explicit writes occupy these inclusive byte ranges:

- 0–3
- 24–28
- 32–40
- 44–47
- 60–64
- 120–124
- 132–136

These total 37 written bytes: eight four-byte values and five one-byte values.

The highest accessed byte is offset 136. Thus, the destination has a minimum address span of 137 bytes from its base. This is not evidence that the actual allocation size is 137 bytes, nor that every byte in that span must be accessed. The return value must permit all listed writes, including the target’s alignment and addressing requirements for the four-byte accesses.

The byte sequence of each four-byte value depends on target endianness, which is not supplied. The offsets and one-byte constants do not depend on that uncertainty.

There is no input-dependent offset calculation, size calculation, loop count, or local allocation request established by this function.

## Side effects

### Direct memory effects

The function reads one selected configuration cell before the dependency call. After the dependency returns, it reads eight word-source cells and performs thirteen destination writes.

Outside the listed destination ranges, it performs no explicit writes. This is a statement about this function’s direct operations, not a guarantee that the dependency leaves other memory unchanged.

If a destination range aliases a global source cell that has not yet been read, an earlier destination write can affect a later source value. By contrast, the selected configuration value and the retained Word-24 value have already been captured before the relevant later operations. The supplied evidence does not establish whether such aliasing is possible.

### Ownership and lifetime

The function directly performs no allocation or release operation other than whatever effects may occur inside F-793a94891814.

The returned address is used for writes only. No ownership transfer, reference-count adjustment, retained handle, or release obligation is established. The dependency’s ownership contract remains unknown.

### Failure and partial effects

There is no explicit null check, validity check, bounds check, or error-reporting path before writing through the returned address.

Successful execution therefore requires readable source cells and a return value valid for every specified write. If these requirements fail, the function provides no local recovery behavior.

There is no rollback. If execution stops during the write sequence, earlier completed writes are not explicitly undone. The exact behavior of an invalid or faulting access is outside the supplied evidence.

The specified ordering is the logical order represented by the listing. It is not an independent guarantee of atomicity, synchronization, inter-thread visibility, or memory barriers.

## Dependencies

### F-793a94891814

This is the sole explicit function dependency.

Established interaction:

- One call per invocation that reaches the call site.
- Six arguments, with the positions and values specified above.
- A return value interpreted as a writable destination address.
- Subsequent global reads occur after the dependency returns.

Unknown dependency properties include:

- Whether it allocates, finds, reuses, or initializes storage.
- Whether it mutates any of the referenced global cells.
- Whether it changes unrelated state.
- Whether it can return an invalid address.
- Whether it can fail, terminate execution, or transfer control without returning.
- Who owns or must eventually release the returned storage.

The fixed arguments 9 and 55 have no established meanings such as count, size, type, or capacity.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reported runtime results.

### Zero selects the first configuration

Given a zero second input and readable Selection-zero, the dependency must receive:

- 23 as argument 2.
- The pre-call value read from Selection-zero as argument 3.
- Zero as argument 6.

Selection-nonzero must not be read by this function. If the dependency returns valid storage, the common thirteen-write sequence follows.

### Nonzero values are not normalized

Given a second input of −1, the dependency must receive 24 as argument 2 and −1 as argument 6.

Given a second input of 7, it must receive 24 as argument 2 and 7 as argument 6.

Both cases use Selection-nonzero, but their sixth arguments remain distinct.

### Dependency mutations affect later reads

Suppose the dependency changes Word-24 and Word-32 before returning. With valid writable storage and no intervening changes, offsets 24 and 32 receive those changed values.

If the dependency also changes the selected configuration cell, that change does not retroactively alter argument 3 of the call already made.

### The offset-24 value is captured before the first write

Suppose the dependency returns an address such that the write at offset 0 overlaps Word-24, and the accesses are otherwise valid.

The write at offset 24 still uses the value read from Word-24 before the offset-0 write, not the value created by that earlier write.

### One-byte fields do not clear adjacent bytes

Suppose destination bytes 28–31 initially contain four distinct values, and neither the dependency nor another actor changes them during the subsequent sequence.

The direct write at offset 28 changes byte 28 to 9. Bytes 29–31 remain unchanged. The equivalent condition applies after the one-byte writes at offsets 40, 64, 124, and 136.

### Untargeted bytes remain untouched by the direct sequence

Given valid storage with known contents after the dependency returns, no aliasing effects, and no concurrent modification, byte 4 is unchanged by this function’s direct writes. Offset 0 receives a four-byte value, but the function does not clear the following gap.

### Invalid return values have no defined fallback

If the dependency returns an address unsuitable for the required writes, acceptance must not require a clean failure status or a fallback object. Neither is established. Such execution falls outside the successful-access preconditions.

## Uncertainties

The following matters are **documented_only**:

- Actual values and meanings of all global source cells.
- Exact ABI details, including independent confirmation of integer and pointer representations.
- Target endianness and alignment rules.
- Destination allocation size, object type, and interpretation of its fields or gaps.
- Dependency failure, ownership, lifetime, and global-mutation behavior.
- Whether source cells can alias the returned destination.
- Concurrency, atomicity, and synchronization guarantees.
- Platform-specific consequences of invalid memory accesses.

The sparse pattern may suggest initialization of a structured object, but that is inferred intent rather than established behavior. No semantic names or types are assigned to the destination fields.

## Evidence

The supplied static listing establishes the zero/nonzero branch, selected pre-call global read, single six-argument dependency call, post-call Word-24 read, and ordered sequence of eight four-byte writes and five one-byte writes.

The byte offsets follow from four-byte element addressing in the represented destination accesses. The byte-width accesses affect only their specified bytes. The listing also establishes the absence of an explicit return value, local validation branch, retry, or cleanup call.

All behavioral claims here are bounded by that static representation. No implementation access, runtime tests, memory observations, or independent dependency analysis have been performed.
