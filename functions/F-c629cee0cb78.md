<!-- Trusted provenance: F-c629cee0cb78; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c629cee0cb78

## Purpose

F-c629cee0cb78 selects one of two construction variants, invokes F-793a94891814, and writes a fixed set of fields into the storage identified by that dependency’s result.

The established behavior is a conditional dependency call followed by ordered initialization writes. The function does not expose the resulting address as a return value. Whether the initialized storage becomes accessible through the first input, a registry, or another mechanism depends on F-793a94891814 and is not established here.

The apparent intent is to construct or configure a record with several value-and-byte fields. That interpretation is inferred, not a proven object model. The contract below specifies byte offsets and access widths without assigning unsupported semantic meanings to the fields.

## Interface

### Inputs

| Input | Established role | Width and interpretation |
|---|---|---|
| First input | Passed unchanged as the first argument to F-793a94891814. | An opaque 32-bit value. Its interpretation as a handle, address, identifier, or other value is unknown. |
| Second input | Compared with zero, controls two dependency arguments, and is also passed unchanged as the dependency’s sixth argument. | Presented as an integer. Its exact ABI width and signedness are not independently established by the supplied evidence. |

The second input is not normalized to a Boolean before forwarding. All nonzero values choose the same construction variant, but the dependency receives the original value. If the input representation admits negative values, those values take the nonzero branch and remain unchanged in the forwarded argument.

### Result

There is no explicit return value. On normal completion, the function has performed the writes specified below.

The address returned by F-793a94891814 is consumed internally. No separate output parameter or direct publication of that address is visible in this function.

### External value sources

The function obtains ten opaque 32-bit values from external storage. Their contents and semantic types are unavailable. This contract assigns them neutral labels:

| Label | Role |
|---|---|
| S-zero | Third dependency argument when the second input equals zero. |
| S-nonzero | Third dependency argument when the second input is nonzero. |
| V0 | Value written at destination offset 0. |
| V24 | Value written at destination offset 24. |
| V32 | Value written at destination offset 32. |
| V60 | Value written at destination offset 60. |
| V120 | Value written at destination offset 120. |
| V132 | Value written at destination offset 132. |
| V144 | Value written at destination offset 144. |
| V156 | Value written at destination offset 156. |

These labels identify value sources, not known constants. The evidence does not justify treating the sources as immutable, mutually distinct, or independent of the destination storage.

### Preconditions for defined completion

The dependency must return an address permitting every specified write. The highest addressed byte is at offset 160, so a single contiguous destination region would need to provide at least 161 accessible bytes beginning at that address.

The destination must also satisfy the execution environment’s requirements for 32-bit stores at the listed offsets. The function supplies no alignment check.

External source storage must support the listed 32-bit reads. The function does not validate source availability, destination capacity, or the dependency result.

## Behavior

### 1. Select the dependency arguments

Exactly one of the two selection sources is read:

- If the second input is zero, the second dependency argument is 19 and the third argument is the value read from S-zero.
- Otherwise, the second dependency argument is 20 and the third argument is the value read from S-nonzero.

The unselected source has no explicit read in this function.

The selected value is obtained before the dependency invocation. A later change to its source does not change the already supplied argument value.

### 2. Invoke the dependency

F-793a94891814 is invoked once with six arguments in this logical order:

| Position | Argument |
|---|---|
| 1 | Original first input |
| 2 | 19 for a zero second input; otherwise 20 |
| 3 | Selected 32-bit source value |
| 4 | 9 |
| 5 | 44 |
| 6 | Original second input |

The meanings of 9 and 44 are not established. In particular, 44 must not be assumed to be the destination allocation size: the visible writes reach offset 160.

The dependency result is interpreted as an address for subsequent writes. No null check, status check, retry, or alternative path appears between the invocation and those writes.

If the dependency does not return normally, the post-call writes described below are not reached. Any effects already performed by the dependency remain outside this function’s established contract.

### 3. Initialize destination fields

After the dependency returns, the function first reads V24 and retains that 32-bit value. It then reads V0 and performs the following destination writes in order.

All offsets are byte offsets relative to the returned address.

| Write order | Offset | Width | Value |
|---:|---:|---:|---|
| 1 | 0 | 32 bits | V0 |
| 2 | 24 | 32 bits | Previously captured V24 |
| 3 | 28 | 8 bits | 9 |
| 4 | 32 | 32 bits | V32 |
| 5 | 60 | 32 bits | V60 |
| 6 | 64 | 8 bits | 12 |
| 7 | 120 | 32 bits | V120 |
| 8 | 124 | 8 bits | 17 |
| 9 | 132 | 32 bits | V132 |
| 10 | 136 | 8 bits | 18 |
| 11 | 144 | 32 bits | V144 |
| 12 | 148 | 8 bits | 19 |
| 13 | 156 | 32 bits | V156 |
| 14 | 160 | 8 bits | 20 |

V32, V60, V120, V132, V144, and V156 are each read when their corresponding word assignment is reached, after the preceding listed writes. They are not all captured before initialization begins.

This order matters if the destination aliases external source storage:

- The first destination write cannot change the V24 value used by the second write; V24 has already been captured.
- An earlier destination write may affect a source read for a later field if those locations alias.
- Values changed by F-793a94891814 before returning are visible to the post-call reads. The function does not preserve a pre-call snapshot of the initialization sources.

The function performs eight 32-bit writes and six byte writes. Their destination byte ranges do not overlap: 38 distinct bytes are explicitly written across the span from offset 0 through offset 160.

No destination contents are explicitly read by this function. No clearing of gaps, padding, or surrounding storage is shown.

### 4. Complete

After the byte write at offset 160, the function returns without an explicit result.

There is no additional dependency call, cleanup action, or final validation in the visible behavior.

## Side effects

### Destination mutation

The direct destination effects are limited to the ranges in the write table. For example, the byte write at offset 28 does not establish any value for offsets 29 through 31. Likewise, the byte write at offset 160 does not establish a wider trailing field.

Where destination storage is valid and non-aliased, bytes outside the listed ranges retain their prior values with respect to this function’s explicit writes. This statement does not constrain changes performed by F-793a94891814.

The byte representation of each 32-bit value follows the execution environment’s byte order. No byte swapping or serialization conversion is visible.

### External reads and dependency effects

On an ordinary complete path, the function explicitly reads one selected source before the dependency call and eight initialization sources after it. These are logical accesses established by the supplied representation, not a guarantee about physical bus transactions or concurrent visibility.

F-793a94891814 may have additional effects. Its implementation and contract are unavailable. It must not be assumed to preserve global storage, allocate fresh memory, retain the first input, or avoid publishing the returned address.

### Ownership and lifetime

This function does not explicitly free or release the destination, either input, or any source value. That does not prove that the dependency performs no release or ownership transfer.

The destination’s owner, lifetime, and relationship to the first input remain unknown. The visible function neither returns the destination nor establishes a caller-visible ownership transfer.

### Errors and partial completion

No explicit error result, diagnostic, exception handler, or rollback mechanism is present.

If a destination access fails after earlier writes have completed, the function contains no logic to undo those writes. The precise outcome of an invalid address, insufficient capacity, or unsuitable alignment depends on the execution environment and is not established as a supported failure mode.

No synchronization or atomic group update is shown. The write sequence must not be described as a transaction.

## Dependencies

### F-793a94891814

**Established interface usage:** one invocation with six arguments, followed by use of the result as writable storage.

**Required condition for normal completion:** the result supports the specified writes through byte offset 160.

**Unknown behavior:**

- Whether it allocates, retrieves, reuses, or embeds the destination.
- Whether it initializes any destination bytes.
- Whether it modifies the external value sources.
- Whether it publishes the destination before this function finishes initialization.
- Whether it consumes or retains either forwarded input.
- Whether it can fail, return an invalid address, or exit without returning.
- What the numerical arguments mean.
- What ownership obligations, if any, accompany its result.

No other function dependency is visible.

## Acceptance examples

These are conditional contract examples, not executed tests. They assume valid storage and normal dependency return unless stated otherwise.

### Zero selection

Given a second input of zero and a selected source value A:

- The dependency receives the original first input, 19, A, 9, 44, and zero.
- S-nonzero is not explicitly read by this function.
- The returned destination receives the fourteen writes in the specified order.

### Nonzero selection preserves the original input

Given a second input of 7 and a selected source value B:

- The dependency receives 20 as its second argument.
- Its third argument is B.
- Its sixth argument is 7, not 1.

For another nonzero input, the branch-selected arguments remain the same except where the selected source itself has changed. The sixth argument still carries that original input.

### Dependency changes an initialization source

Suppose V32 contains A before the dependency call, and the dependency changes it to B before returning.

Provided no intervening aliasing write changes that source again, the word written at offset 32 is B. The function does not use a pre-call copy of V32.

### Captured V24 survives an earlier destination write

Suppose the first destination write would overwrite the storage serving as V24.

The write at offset 24 still uses the V24 value read before that first write. It does not reread V24 after writing offset 0.

### Later sources can observe earlier writes

Suppose the storage serving as V60 aliases a destination range modified before the offset-60 assignment.

The value used at offset 60 is obtained after those earlier writes. An implementation that snapshots every source immediately after the dependency returns would not preserve the established behavior in this case.

### Narrow writes preserve adjacent bytes

Given valid destination storage whose bytes at offsets 29 through 31 initially contain distinct values, the explicit write of 9 at offset 28 leaves those neighboring bytes unchanged.

This example assumes the dependency and aliased source effects do not separately modify those bytes.

### Capacity boundary

A contiguous region covering offsets 0 through 160 provides the byte extent needed for the listed stores, subject to alignment and access permissions.

A region covering only offsets 0 through 159 is insufficient. No capacity check or graceful rejection is established.

## Uncertainties

The following are **documented_only** rather than guessed:

- The numeric contents and semantic roles of all ten external sources.
- The exact ABI width and signedness of the second input.
- Pointer width, target byte order, and alignment requirements.
- Whether the observed destination layout belongs to one object, an embedded region, or another storage arrangement.
- The meanings of the word fields and their neighboring byte values.
- The reasons for the gaps between fields.
- The meanings of dependency arguments 19, 20, 9, and 44.
- Destination ownership, publication, lifetime, and capacity guarantees.
- Dependency failure behavior and global-state effects.
- Concurrency rules and whether any accesses have stronger visibility guarantees than ordinary memory operations.

**Inferred intent:** the repeated word-and-byte pattern is consistent with installing field values plus small tags or attributes. Neither that interpretation nor any particular type system is established.

The normal-path behavior requires valid memory, but that requirement is not evidence of internal validation.

## Evidence

The supplied static function representation establishes:

- A zero-versus-nonzero branch.
- Selection between two external 32-bit values.
- One six-argument invocation of F-793a94891814.
- Use of the dependency result as a destination address.
- A post-call V24 read preceding the first destination write.
- Fourteen ordered stores at the widths and offsets listed above.
- No visible validation, cleanup, rollback, or explicit return value.

The byte offsets follow from four-byte word indexing, including the final byte write at word index 40, which corresponds to byte offset 160. All listed offsets are small fixed values; no input-dependent destination arithmetic is visible.

This contract uses only the supplied static evidence. No implementation access, dependency inspection, runtime tests, or dynamic observations have been performed.
