<!-- Trusted provenance: F-816ea8e29abb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-816ea8e29abb

## Purpose

F-816ea8e29abb constructs or updates a small descriptor from a caller-supplied byte sequence and advances a caller-owned input cursor on success.

The established transformation distinguishes inputs by the high bit of their first byte. Inputs with that bit clear are copied after removal of at most one leading zero byte. Inputs with that bit set are converted using a right-to-left complement-and-carry transformation, after removal of at most one leading all-ones byte.

**Inferred intent:** the descriptor represents an integer using a magnitude byte sequence and a separate sign or type field. The positive field value is known; the negative field value comes from an unresolved global. Neither the semantic meaning of that global nor the complete descriptor format is established.

This contract describes the supplied static evidence. No runtime tests have been performed.

## Interface

### Inputs

The function accepts three arguments:

1. **Optional descriptor-holder address.**  
   A null address requests a new descriptor without publishing it through a holder. A nonnull address identifies a pointer-sized slot:
   - A null slot value requests a new descriptor.
   - A nonnull slot value supplies the descriptor to update.

2. **Required input-cursor address.**  
   This identifies a slot containing the address of the input bytes. The function reads that slot after obtaining a descriptor and, on success, replaces it with the end address of the original input range.

3. **Input byte count.**  
   The supplied representation uses a signed integer count. Ordinary supported operation requires a nonnegative count, sufficient readable input storage, representable address arithmetic, and a representable count-plus-one allocation request.

The visible representation uses four-byte words and stores pointer values in those words. The descriptor’s accessed prefix is:

| Byte offset | Width | Observed role |
|---|---:|---|
| 0 | 4 bytes | Result byte count |
| 4 | 4 bytes | Sign or type field |
| 8 | 4 bytes | Result-buffer address |

The supplied source is consistent with a 32-bit pointer and integer model. An independently verified ABI, alignment specification, and complete descriptor size are not available.

### Return value

On success, the function returns the selected descriptor address. If a holder was supplied, the function also stores this address in that holder.

On either visible allocation-failure path, it returns null. There is no separate status result.

The function does not return the advanced input address directly.

### Preconditions and storage requirements

For defined ordinary use:

- A supplied holder must be readable and, on success, writable.
- A reused descriptor must have a readable and writable prefix covering all three fields.
- The cursor slot must be readable and writable, including when the input count is zero.
- Positive-length input must be readable for the requested byte count.
- The count-plus-one calculation and input-end calculation must be valid in the actual execution model.
- Storage and pointer values returned by the dependencies must satisfy the accesses described below.

No explicit argument validation enforces these requirements.

The examples below assume the cursor slot, holder, descriptor, and input do not overlap in ways that change their contents during execution. The implementation does not enforce such separation.

## Behavior

### 1. Select or obtain a descriptor

If a holder address is present, its value is read first.

If that value is nonnull, it becomes the selected descriptor. Otherwise, F-f9450f0778b9 is called with the numeric argument **2**.

If that dependency returns null, the function returns null immediately. In particular, the visible body has not yet read the input cursor, allocated a result buffer, or invoked the later error-reporting dependency.

If a new descriptor is returned, its field at offset 4 is immediately written with **2**. The other two fields are not initialized explicitly at this point. Later behavior therefore depends on initialization supplied by F-f9450f0778b9, particularly for the field at offset 8.

### 2. Capture the input range and obtain result storage

After descriptor selection, the function reads the input-cursor slot.

It calculates an end address using that captured start address and the **original** input count. This saved end address is not reduced when a leading input byte is subsequently omitted.

F-43da6b23bd44 is then called with:

- The original input count plus one.
- Zero.
- Zero.

The returned address is used as the output byte buffer. The extra byte accommodates one expansion case in the negative-input transformation. No general terminator behavior follows from the extra capacity.

If this allocation fails:

1. F-4fc89ec41a6d is called with the five numeric arguments **13, 146, 65, 0, 0**, in that order.
2. After that call, the function determines whether the selected descriptor is still referenced by the supplied holder:
   - If no holder was supplied, F-6c81b0b12feb is called with the selected descriptor.
   - If a holder was supplied, the holder is read again. F-6c81b0b12feb is called only if its current value differs from the selected descriptor.
3. The function returns null.

That second holder read is significant. The cleanup decision uses the holder’s value **after** the error-reporting call, not necessarily its entry value.

The visible body does not advance the cursor or publish a new result-buffer address on this failure path. Dependency effects remain unknown.

### 3. Transform the bytes

#### Empty input

For an original count of zero:

- The descriptor field at offset 4 is written with **2**.
- No input byte is read by the transformation.
- No result-buffer byte is explicitly initialized by this branch.
- The result count remains zero.

Result storage is still requested before taking this branch.

#### First-byte high bit clear

For nonempty input whose first byte has its high bit clear:

1. The descriptor field at offset 4 is written with **2**.
2. The first byte is examined again.
3. If that byte is zero and the input count is not one, exactly one leading byte is omitted and the working count is reduced by one.
4. The unmapped external byte-copy dependency is invoked with the output address, the retained input address, and the working count.

There is no loop removing multiple leading zero bytes. For example, a three-byte sequence beginning with two zero bytes loses only its first byte.

The source establishes the copy call and its arguments. The precise external operation, including any overlap guarantees, is not independently supplied.

#### First-byte high bit set

For nonempty input whose first byte has its high bit set:

1. The descriptor field at offset 4 is written with a value read from an unresolved global.
2. The first byte is examined again.
3. If that byte is all ones and the count is not one, exactly one leading byte is omitted and the working count is reduced by one.
4. The retained bytes are transformed from right to left.

The transformation preserves low-order zero bytes in their corresponding output positions. It then writes the first encountered nonzero byte as its eight-bit complement plus one, retaining only the resulting byte, and writes the eight-bit complement of each remaining more-significant byte.

Thus the output is consistent with obtaining an unsigned magnitude from a big-endian two’s-complement negative encoding. This numerical interpretation is inferred; the byte operations themselves are established.

The writes proceed from the end of the retained output range toward its beginning. No further output normalization removes a leading zero produced by the conversion.

#### Retained bytes all zero

A special path handles an all-zero retained sequence.

Under ordinary nonnegative-length, stable-input conditions, this can occur in the negative branch only after a leading all-ones byte has been omitted. For example, the retained sequence from an input of `FF 00 00` is two zero bytes.

The implementation:

- Writes the retained zero bytes while scanning backward.
- Reads the byte immediately before the retained sequence before recognizing that the retained count has been exhausted.
- Writes **1** to the first output byte.
- Writes **0** at the output index equal to the retained count.
- Increases the result count by one.

For a valid input entering this case through prefix omission, that extra read reaches the omitted all-ones byte within the original input range. It is nevertheless an observable read beyond the retained subrange.

The result for `FF 00 00` is therefore `01 00 00`, not a two-byte result.

### 4. Replace the descriptor’s result and publish success

After the transformation:

1. The descriptor field at offset 8 is read.
2. If it is nonzero, F-5990a26eaaa3 is called with **no explicit arguments**.
3. The result count is written at offset 0.
4. The newly allocated result-buffer address is written at offset 8.
5. If a holder address was supplied, the selected descriptor address is written there.
6. The saved original input-end address is written to the cursor slot.
7. The selected descriptor is returned.

The no-argument call must not be described as definitely freeing the previous buffer. Its trigger is known, but its memory effects and use of implicit state are not.

The sign or type field is written before this call. The function does not explicitly restore that field afterward. If the dependency changes it through shared state, the supplied evidence does not establish that the earlier value survives.

## Side effects

### Descriptor state

A successful ordinary call replaces the descriptor’s count and result-buffer address. The sign or type field is set during transformation:

- **2** for empty input and the high-bit-clear branch.
- An unresolved global value for the high-bit-set branch.

The count records the number of output bytes, not necessarily the number consumed. It can be one less than the input count after prefix omission, or increase again in the all-zero-retained negative case.

### Input consumption

On success, the cursor is set to the captured start address plus the original count. Omitted prefix bytes are therefore still consumed.

For zero length, the cursor slot is still written, even though its address value ordinarily remains unchanged.

The visible body does not write input bytes directly. An external copy operation or other dependency could have additional effects not specified here.

### Allocation and ownership

The function may obtain a new descriptor and always requests new result storage once a descriptor has been selected.

On success, the new buffer address is installed in the descriptor. This establishes the association, but not a complete ownership or deallocation policy.

On result-buffer allocation failure, F-6c81b0b12feb may receive the selected descriptor. The supplied evidence does not establish whether that dependency releases memory, destroys nested state, or performs another action.

When a descriptor already has a nonzero buffer field, the function invokes F-5990a26eaaa3 before overwriting it. There is no explicit old-buffer argument and no proven release guarantee.

### Failure atomicity and aliasing

The function is not established to be transactional. Dependencies may modify caller-visible state, and a newly obtained descriptor receives an early field write.

For an existing descriptor, the visible body does not change its fields before result-buffer allocation succeeds. This is not a guarantee against dependency-induced changes.

Aliasing can alter otherwise straightforward behavior. In particular:

- A descriptor-field write can affect later input reads if storage overlaps.
- The final holder write precedes the final cursor write.
- If the holder and cursor slot overlap, the cursor publication can overwrite the holder publication.
- Dependencies may alter shared data between the visible reads and writes.

No overlap rejection or defensive snapshot of all relevant state is shown.

## Dependencies

| Opaque function ID | Established interaction | Not established |
|---|---|---|
| F-f9450f0778b9 | Called with 2 when no descriptor is available; null means immediate failure. | Full allocation size, initialization, ownership, and global effects. |
| F-43da6b23bd44 | Called with original count plus one, zero, zero; returned storage receives result bytes. | Allocation policy, initialization, argument widths beyond the supplied representation, and global effects. |
| F-4fc89ec41a6d | Called with 13, 146, 65, 0, 0 on result-storage failure. | Error namespace, reporting mechanism, and state effects. |
| F-6c81b0b12feb | Receives the selected descriptor on the conditional allocation-failure cleanup path. | Destruction or release semantics. |
| F-5990a26eaaa3 | Called without explicit arguments when the current buffer field is nonzero. | Whether or how it handles the previous buffer, and whether descriptor fields survive unchanged. |

An additional external byte-copy dependency has no supplied opaque function ID. Its destination, source, and count arguments are established. Its complete contract is **documented_only**.

The negative sign or type value is read from a global whose value and ownership are also **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests. They assume valid 32-bit storage, nonnegative lengths, successful dependencies with conventional allocation and copy behavior, no disruptive aliasing, and no dependency mutation of relevant state.

| Input bytes | Result count | Result bytes | Sign/type field |
|---|---:|---|---|
| Empty | 0 | No byte content specified | 2 |
| `00` | 1 | `00` | 2 |
| `00 7F` | 1 | `7F` | 2 |
| `00 00 7F` | 2 | `00 7F` | 2 |
| `7F FF` | 2 | `7F FF` | 2 |
| `FF` | 1 | `01` | Unresolved negative value |
| `80` | 1 | `80` | Unresolved negative value |
| `FF 7F` | 1 | `81` | Unresolved negative value |
| `FF FF 7F` | 2 | `00 81` | Unresolved negative value |
| `80 00` | 2 | `80 00` | Unresolved negative value |
| `FF 00 00` | 3 | `01 00 00` | Unresolved negative value |

Additional acceptance conditions:

- A successful conversion of `00 7F` consumes both input bytes despite producing one byte.
- Empty input still requires a result-storage request of one byte and publishes its returned address.
- A failed descriptor acquisition returns null before the visible body reads the cursor.
- A failed result-storage request invokes the reporting dependency before the conditional descriptor cleanup.
- For a reused descriptor with a nonzero prior buffer field, the no-argument dependency is called before the new count and buffer address are installed.
- Multiple redundant sign-extension bytes are not fully stripped.

## Uncertainties

- **documented_only — Negative field value:** the global’s numeric value and semantic interpretation are unavailable.
- **documented_only — Complete object contract:** allocation size, initialization, trailing fields, alignment, and destruction rules are unavailable.
- **documented_only — Dependency effects:** no dependency is assumed to preserve globals, descriptor fields, holder values, or memory ownership.
- **documented_only — Exceptional control flow:** dependency exceptions, nonlocal exits, and process termination are not characterized.
- **documented_only — Invalid counts:** negative counts and overflowing count-plus-one calculations are unchecked. No reliable error result or portable transformation is specified for them.
- **documented_only — Pointer arithmetic limits:** address overflow and pointer provenance outside valid input storage are not resolved.
- **documented_only — External copy semantics:** overlap behavior and exact implementation are unavailable.
- **Inferred intent:** signed big-endian integer conversion is consistent with the operations but is not independently confirmed.
- **Established limitation:** the positive path does not append a terminator, and the empty path does not initialize result bytes. Extra allocated capacity is not evidence of zero filling or termination.

## Evidence

The supplied static function body establishes the branch conditions, dependency arguments, field accesses, buffer transformation, failure-path order, and publication order described above.

The supplied dependency map establishes the opaque IDs used here. It does not establish the dependencies’ internal behavior.

The representation exposes four-byte fields, byte-wide transformation operations, and pointer values stored in integer words. The complete target ABI remains unverified.

No runtime execution, dependency inspection, allocation-failure injection, or acceptance testing has been performed. The acceptance examples are consequences of the visible operations under their stated conditions, not observed results.
