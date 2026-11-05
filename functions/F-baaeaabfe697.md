<!-- Trusted provenance: F-baaeaabfe697; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-baaeaabfe697

## Purpose

F-baaeaabfe697 conditionally transfers selected bytes from an object’s staging fields into a three-byte-stride region, advances a 32-bit index unless that index equals 16, and conditionally dispatches an indirect call.

The established behavior includes an unconditional call to F-338d16f9ce72 before the object’s index, mode, and staged bytes are examined. Consequently, the operation is defined in terms of the state available **after that dependency returns**, not necessarily the state supplied at entry.

A likely intent is to accumulate up to sixteen three-byte entries using separate updates for different components. That interpretation is inferred, not established: the function neither validates an index range nor writes every component of an entry on every invocation.

## Interface

### Input and result

The function receives one argument, interpreted as the base address of mutable storage. This contract calls that address **B**. The supplied representation uses a 32-bit integer argument for the address, 32-bit integer fields, and 32-bit pointer fields.

There is no explicit return value. Normal completion occurs either after the final dispatch condition fails or after the indirect call returns. No result from either called dependency is used.

There is no null check or general validity check for B.

### Relevant storage layout

Offsets are byte offsets relative to B.

| Offset | Width | Established use |
|---|---:|---|
| +0x28 | Address passed to a dependency | F-338d16f9ce72 receives B + 0x28. |
| +0x28 | 4 bytes | Later interpreted as a nullable pointer to another 32-bit value. |
| +0x54 | 1 byte | First staged component, used when the mode is zero. |
| +0x55 | 1 byte | Second staged component, used when the mode is zero. |
| +0x56 | 1 byte | Third staged component, used when the mode is one. |
| +0x5c | 4 bytes | Index used for the equality guard, destination selection, and increment. |
| +0x60 onward | Selected one-byte locations | Destination region addressed with a stride of three bytes. |
| +0x90 | 1 byte | Mode selector. |

For a particular index value **n**, the three candidate destination offsets are:

- First component: 0x60 + 3n.
- Second component: 0x61 + 3n.
- Third component: 0x62 + 3n.

These formulas describe addressing behavior, not a validated array boundary.

The sixteen nominal entries at indices 0 through 15 occupy offsets +0x60 through +0x8f. The mode byte at +0x90 immediately follows them. This supports, but does not prove, the inferred sixteen-entry interpretation.

### Preconditions for defined, successful execution

The caller must supply storage valid for all accesses that actually occur, including accesses performed by dependencies.

For the ordinary interpretation, B identifies a live, writable object; the index is between 0 and 16 inclusive; and any nonnull pointer loaded from +0x28 points to readable storage containing at least one 32-bit value. Those are safe-use assumptions, not conditions enforced by this function.

Out-of-range indices, address arithmetic overflow, concurrent mutation, and invalid indirect-call targets have no established recovery behavior.

## Behavior

### 1. Invoke the preparation dependency

The first observable operation in the supplied body is a call to F-338d16f9ce72 with the address B + 0x28.

No index, mode, staging-byte, or final-dispatch decision is established before this call. The dependency may alter relevant object storage or external state; its implementation is unavailable. This contract does not assume that it preserves fields, globals, pointers, ownership, or object lifetime.

The remaining behavior is conditional on this call returning and leaving the subsequent accesses valid.

### 2. Read the index and apply an exact equality guard

After the dependency returns, the function reads the 32-bit field at +0x5c.

If that value is exactly 16:

- No mode byte is examined by the guarded update.
- No staged component is copied by the guarded update.
- The index is not incremented.
- Execution still proceeds to the final pointer inspection and possible indirect dispatch.

The guard is **equality with 16**, not a less-than-capacity check. Values greater than 16 and negative values enter the update path.

### 3. Perform mode-dependent component writes

When the initially read index is not 16, the function examines the byte at +0x90.

#### Mode zero

If that byte is zero:

1. The byte at +0x54 is copied to the first-component destination selected using the index obtained for the initial guard.
2. The index field at +0x5c is read again.
3. The byte at +0x55 is copied to the second-component destination selected using that reread index.

The first write precedes the second write.

The second destination is not unconditionally tied to the originally captured index. In normal, non-overlapping storage with no intervening mutation, both writes address the same entry. If the first write changes bytes of the index field through an out-of-range destination, the second destination can differ.

No third-component write is performed in this mode.

#### Mode one

If the first mode examination did not find zero, the function examines +0x90 again to determine whether it equals one.

If that examination yields one, the byte at +0x56 is copied to the third-component destination selected using the index obtained for the initial guard.

No first- or second-component write is performed in this branch.

#### Other modes

If the mode is nonzero on the first examination and is not one on the second examination, no staged component is copied.

This does not suppress the index increment.

For stable ordinary memory, these branches reduce to mode zero, mode one, and every other byte value. The repeated mode examination matters when describing the actual access structure; the supplied evidence does not establish support for concurrent or device-driven changes between reads.

### 4. Increment the current index field

For every path whose initially read index was not 16, the function reads the 32-bit field at +0x5c again, adds one, and stores the result back to +0x5c.

This is an increment of the field’s value at that later point. It is not necessarily an increment of the initial index snapshot.

The increment follows any component writes. It also occurs for modes that caused no component write.

For an ordinary stable index n in the range 0 through 15, the resulting index is n + 1. In particular, an index of 15 advances to 16, after which a later invocation skips the guarded update unless the preparation dependency changes the state.

The recovered representation uses signed 32-bit integer arithmetic. Behavior at signed overflow and during overflowing address calculations is not established by this evidence.

### 5. Inspect the final dispatch state

After the guarded update, or immediately after skipping it, the function reads the 32-bit pointer field at +0x28.

- If the pointer is null, it does not dereference it and performs no indirect dispatch.
- If the pointer is nonnull, it reads one 32-bit value from the pointed-to address.
- If that value is zero, it performs no indirect dispatch.
- If the pointer is nonnull and the pointed-to value is nonzero, it invokes an indirect target obtained through a separate external pointer location.

The pointed-to value is used only as a zero/nonzero condition in the supplied body. It is not shown as the indirect call target.

No explicit arguments are shown for the indirect call. Its complete calling convention and any implicit register-based inputs are unknown.

The final pointer read occurs after the local update. Therefore, where invalid or overlapping layouts cause writes to affect +0x28 or its referenced storage, the dispatch decision can reflect those writes.

## Side effects

### Directly established effects

Subject to the branches above, the function may:

- Invoke F-338d16f9ce72.
- Write one byte to each of two selected destinations in mode zero.
- Write one byte to a selected destination in mode one.
- Write the 32-bit index field whenever the initial index differs from 16.
- Read through the pointer stored at +0x28.
- Invoke an indirect external target.

The local body does not explicitly allocate, free, transfer, retain, or release storage. That absence does not establish that either called dependency lacks such effects.

### Preservation limits

For valid indices 0 through 15 and stable ordinary storage, the local component writes affect only the selected components. Components not selected by the mode retain their previous contents unless changed by a dependency or another actor.

No blanket preservation guarantee can be made for the complete function because both dependencies are opaque.

### Failure and interruption

There is no explicit error result, rollback, or recovery path.

If a memory access or a dependency does not complete normally, subsequent operations are not guaranteed to occur. Earlier writes are not undone by any mechanism visible here.

For example, a failure during the second mode-zero component access may occur after the first component was already written. A failure during final pointer inspection may occur after the index has already advanced.

## Dependencies

### F-338d16f9ce72

- Called unconditionally before local update decisions.
- Receives B + 0x28.
- Its result is not used.
- Its mutation, ownership, error, and lifetime behavior are unknown.

The shared offset between this argument and the later pointer field suggests a related substructure, but does not establish the dependency’s purpose.

### Opaque indirect dependency

- Invoked only when the final pointer at +0x28 is nonnull and its referenced 32-bit value is nonzero.
- The call target is obtained separately from that tested pointer and value.
- No opaque function ID is supplied for this target.
- Its arguments beyond the absence of explicit arguments, return behavior, global effects, and ownership behavior are unknown.

Neither dependency may be assumed to preserve global state or to leave the input object usable without an independent contract.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume F-338d16f9ce72 returns normally, the described state holds after it returns, memory is valid and stable, address arithmetic is valid, and any indirect target is callable.

1. **Mode-zero update within range.**  
   With index 2 and mode 0, staged bytes at +0x54 and +0x55 are written to +0x66 and +0x67, respectively. The byte at +0x68 is not written by the local component-update logic. The index becomes 3.

2. **Mode-one update within range.**  
   With index 2 and mode 1, the staged byte at +0x56 is written to +0x68. The local component-update logic does not write +0x66 or +0x67. The index becomes 3.

3. **Unrecognized mode still consumes an index.**  
   With index 2 and stable mode 7, no staged byte is copied, but the index becomes 3. Reject an implementation that leaves the index unchanged merely because the mode is unrecognized.

4. **Exact sentinel skips only the update.**  
   With index 16, the local update neither copies components nor increments the index. The function still examines the pointer at +0x28 and may dispatch.

5. **Last nominal entry.**  
   With index 15 and mode 0, the local writes target +0x8d and +0x8e, and the index becomes 16. Offset +0x8f is not written by that branch.

6. **No saturation above the sentinel.**  
   With index 17 and mode 1, the selected destination is +0x95 and the index becomes 18, provided those accesses and arithmetic are valid. Reject a replacement that silently clamps the index or skips all indices greater than 16.

7. **Null dispatch pointer.**  
   When the final value at +0x28 is null, no pointed-to value is read and no indirect dispatch occurs. Any earlier component update remains effective.

8. **Nonzero dispatch condition.**  
   When the final pointer is valid and its referenced 32-bit value is nonzero, the separate indirect target is invoked. A negative nonzero value satisfies the condition just as a positive nonzero value does.

9. **Preparation changes the decision.**  
   If the entry-state index is 16 but F-338d16f9ce72 changes it to 0 before returning, the local update uses the post-call value 0. Entry-state sentinel detection is not equivalent behavior.

10. **Aliasing can affect the reread index.**  
    A negative initial index is not rejected. For example, index −1 places the mode-zero first destination at +0x5d, inside the index field. If execution is otherwise defined, the second destination and final increment use the resulting reread field, not necessarily −1. No exact resulting value is specified without additional layout and arithmetic facts.

## Uncertainties

The following are **documented_only**:

- The concrete object type and allocated extent.
- The semantic meaning of the staging bytes, mode, and referenced dispatch condition.
- The contracts of both dependencies, including ownership and global mutations.
- The identity and ABI of the indirect target.
- Endianness and the consequences of partially overwriting a 32-bit field.
- Target-specific behavior for integer overflow, address overflow, invalid pointers, and misaligned accesses.
- Whether storage is ordinary memory, volatile storage, or shared with concurrent actors.
- Whether an index-range invariant is established elsewhere.
- The precise machine-level ordering within an individual assignment beyond the inter-statement ordering described above.

No runtime tests have been performed.

## Evidence

The supplied static body establishes the unconditional preparation call, field offsets and access widths, exact equality guard against 16, mode-dependent byte writes, repeated index access, index increment, and final pointer-gated indirect call.

The three-byte stride and the placement of the mode byte immediately after sixteen nominal entries support an inferred bounded-entry interpretation. They do not establish bounds enforcement, initialization, or ownership.

The supplied dependency map identifies F-338d16f9ce72 as the direct dependency and F-baaeaabfe697 as this function. No implementation evidence is available for the direct dependency or the indirect target.
