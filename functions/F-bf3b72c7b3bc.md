<!-- Trusted provenance: F-bf3b72c7b3bc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bf3b72c7b3bc

## Purpose

F-bf3b72c7b3bc conditionally contributes 32-bit values to an output sequence according to an 8-bit discriminator in an input object. Some discriminator values cause a direct append; others first invoke F-1cedbadc4b50 and then conditionally append an additional value.

The function returns the resulting output cursor. Its local append operation is guarded only by cursor equality with the supplied boundary. It does not establish that the cursor lies within a valid buffer, that the boundary is reachable, or that the dependency respects that boundary.

The interpretation of the input as a tagged object and of the output as a bounded sequence is inferred intent. The field accesses, dispatch decisions, calls, writes, and returned cursor relationships below are established by the supplied static representation.

## Interface

### Inputs

The function accepts three arguments:

1. **Input-object address.** An address used to access the discriminator and selected 32-bit fields. The supplied representation carries this address in an integer-shaped argument; its declaration does not establish a portable pointer interface.
2. **Output cursor.** A pointer to the next candidate 32-bit output location.
3. **Output boundary.** A pointer compared with the current output cursor before a local append.

The boundary is treated as an equality sentinel, consistent with—but not sufficient to prove—a one-past-the-end buffer convention.

### Relevant input layout

Offsets are in bytes from the input-object address.

| Offset | Width | Role |
|---|---:|---|
| +8 | 8 bits | Discriminator |
| +12 | 32 bits | Direct output value or first argument to F-1cedbadc4b50 |
| +16 | 32 bits | Optional value read after the dependency returns |
| +20 | 32 bits | Optional value for discriminator 10 |

The discriminator is interpreted as an unsigned byte, with values from 0 through 255. No structure-wide read is shown. Only fields selected by the executed path need to be read.

The output element width is 32 bits. A successful local append advances the output cursor by one such element, corresponding to four bytes in the supplied representation. The function performs no arithmetic on the value being appended.

Byte order, alignment requirements, and the full object layout are not established.

### Return value

The return value is an output cursor, not a status code:

- For a path with no local append and no dependency call, it is the original output cursor.
- Following a dependency call, it is based on the cursor returned by that dependency.
- For a successful local append, it is the applicable cursor advanced by one 32-bit element.
- If the applicable cursor equals the boundary, the local append is suppressed and that cursor is returned unchanged.

The function does not return an appended-element count or distinguish “nothing selected” from “no local space available.”

### Preconditions

For defined execution, all accesses actually taken must be valid:

- The discriminator byte must be readable.
- Each selected 32-bit input field must be readable when accessed.
- Whenever a local append occurs, the current output cursor must designate writable storage for one 32-bit element, and advancing it must be valid.
- Dependency-path inputs and storage must additionally satisfy F-1cedbadc4b50’s requirements, which are not supplied.

Pointer inequality is not a validity or capacity check. A cursor beyond the boundary, an unrelated pointer, or a null pointer unequal to the boundary is not rejected.

## Behavior

### Dispatch overview

For a discriminator that remains stable during the invocation, the behavior is:

| Discriminator values, decimal | Action |
|---|---|
| 1, 2, 12, 13, 14, 15, 16, 17 | Invoke F-1cedbadc4b50 using the +12 field, then consider the +16 field |
| 3, 4, 6, 9, 18, 19, 20, 21, 22 | Consider the +12 field as an unconditional direct value |
| 10 | Consider the +20 field, omitting it when zero |
| 0, 5, 7, 8, 11, 23–255 | No dependency call and no local output write |

“Unconditional direct value” means zero is a valid value to append. It does not override the cursor-equals-boundary guard.

### Discriminator read order

The function first reads the discriminator at +8.

Values 1, 2, and 12–17 immediately select the dependency path. Values 3 and 18–20 immediately select the direct +12 path.

All other initial values reach a second discriminator read at +8. This second read selects:

- Direct +12 handling for 4, 6, 9, 21, or 22.
- Optional +20 handling for 10.
- No further action for every other value.

The two reads are part of the supplied access sequence; the second is not merely a restatement of the first. There is no dependency call or explicit write between them. The stable-discriminator table therefore describes ordinary execution without concurrent or externally induced changes. The supplied material does not establish a concurrency or volatile-memory contract.

### Direct +12 handling

The function reads the 32-bit field at +12 before checking output capacity.

If the output cursor equals the boundary, it returns the cursor without writing. Otherwise, it writes the field’s 32-bit value at the cursor and returns the cursor advanced by one element.

There is no zero-value filter on this path. A zero field is appended if the cursor differs from the boundary.

### Optional +20 handling

The function reads the 32-bit field at +20.

- If the field is zero, it immediately returns the original output cursor.
- If the field is nonzero, it applies the local append rule: suppress the write when cursor and boundary are equal; otherwise store the value and advance one element.

The field read and zero test precede the equality guard. A full output buffer does not prevent the selected input-field read.

### Dependency handling and optional +16 value

The function reads the 32-bit field at +12 and invokes F-1cedbadc4b50 with three arguments:

1. The value read from +12.
2. The original output cursor.
3. The original output boundary.

The dependency’s return value becomes the current output cursor.

Only after the dependency returns does F-bf3b72c7b3bc read the 32-bit field at +16. Consequently, a dependency-induced change to that field can affect the optional value. The function does not snapshot +16 before the call.

If the newly read +16 value is zero, the function returns the dependency’s cursor without a local write.

If it is nonzero:

- When the dependency’s cursor equals the original boundary, no local write occurs.
- Otherwise, the value is written at the dependency’s cursor, and the returned cursor advances by one element.

The dependency is called even when the original output cursor equals the boundary. The local boundary guard does not protect or suppress the dependency’s own behavior.

### Local append rule

Every local append follows the same behavior:

1. The candidate 32-bit value has already been read.
2. The applicable cursor is compared for equality with the boundary.
3. Equality suppresses the write and cursor advancement.
4. Inequality causes one 32-bit store at the cursor and returns the next element position.

There is no less-than comparison, remaining-capacity calculation, truncation count, or overflow report. The function does not deliberately wrap a cursor or recover from invalid pointer arithmetic.

## Side effects

### Direct effects

The function itself performs:

- One or two discriminator-byte reads.
- Selected 32-bit input-field reads.
- At most one local 32-bit output store.
- At most one call to F-1cedbadc4b50.

No explicit allocation, deallocation, logging, global-variable access, or input-field assignment appears in the supplied body.

### Dependency effects

Total invocation effects cannot be bounded to one output store because F-1cedbadc4b50 may itself write output or affect other state. Its implementation and behavioral contract are not supplied.

The dependency must not be assumed to preserve input memory, globals, allocation ownership, or output contents. In particular, the +16 read occurs after that call and depends on the input object remaining readable.

### Aliasing

No non-aliasing check is present.

An output store may overlap the input object if the caller supplies overlapping addresses. The locally selected value is read before that store. No further input-field read follows the local store.

On a dependency path, effects through aliases during the dependency call can affect the subsequent +16 read. The original boundary argument remains the equality sentinel used by this function; changes to memory at that address do not redefine the pointer value.

### Ownership and errors

No ownership transfer is explicitly established by this function’s direct operations. Returning an advanced cursor does not establish ownership of the underlying storage.

There is no explicit error channel. A suppressed append is not reported as an error. Invalid input addresses, invalid output storage, dependency failures, and unreadable post-call input memory have no local recovery behavior shown.

## Dependencies

### F-1cedbadc4b50

This is the only explicit function dependency.

It is invoked exactly once on discriminator values 1, 2, and 12–17, assuming the initial discriminator read has one of those values. It is not invoked on other paths.

Its first argument is a 32-bit value from input offset +12. The supplied evidence does not establish whether that value denotes an address, handle, scalar, or another object representation.

Its return value is consumed as an output cursor without validation. The function does not check that it:

- Lies within the original output region.
- Is aligned.
- Is non-null.
- Is at or before the boundary.
- Preserves any relationship to the original cursor.

Dependency semantics, ownership effects, failure conventions, and output-capacity guarantees are documented_only unknowns.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Assume readable input fields, valid required output storage, a stable discriminator, and any additional dependency preconditions.

1. **Direct nonzero value.**  
   With discriminator 4, +12 equal to 37, and cursor unequal to boundary, the function writes 37 at the original cursor and returns the next 32-bit element position. It does not call F-1cedbadc4b50.

2. **Direct zero value.**  
   With discriminator 3 and +12 equal to zero, an available output slot receives zero. The zero-omission rule does not apply to this path.

3. **Direct path at the boundary.**  
   With discriminator 22 and cursor equal to boundary, the function still reads +12 but performs no output store and returns the original cursor.

4. **Optional +20 value omitted.**  
   With discriminator 10 and +20 equal to zero, the function returns the original cursor without writing, regardless of whether it equals the boundary.

5. **Optional +20 value appended.**  
   With discriminator 10, a nonzero +20 value, and cursor unequal to boundary, that value is written and the cursor advances one element.

6. **Dependency followed by a local append.**  
   With discriminator 12, suppose F-1cedbadc4b50 returns a valid cursor unequal to the boundary. If +16 is nonzero when read after the call, that value is stored at the returned cursor, and the final result is one element beyond it.

7. **Dependency fills the available output.**  
   With discriminator 1, suppose the dependency returns the boundary. A nonzero +16 field is read but not locally appended. The boundary is returned.

8. **Dependency changes the optional field.**  
   With discriminator 17, suppose +16 initially contains 5, but the dependency changes it to zero while keeping the object readable. The function observes zero after the call and returns the dependency’s cursor without a local append.

9. **Initially full output does not suppress delegation.**  
   With discriminator 2 and original cursor equal to boundary, F-1cedbadc4b50 is still called. Its return value determines the cursor used for subsequent handling.

10. **Unrecognized discriminator.**  
    With discriminator 255, the function reads the discriminator twice, calls no dependency, reads none of the three 32-bit payload fields, and returns the original cursor without a local store.

11. **Cursor beyond the boundary.**  
    On a path selecting a local append, a cursor beyond the boundary is not rejected merely because of its ordering. If unequal, it takes the store path. This is not an accepted safe-buffer usage; valid writable storage and valid pointer advancement remain preconditions.

## Uncertainties

The following are documented_only rather than guessed:

- Semantic names and meanings of discriminator values.
- Full input-object size and layout.
- Byte order, alignment, and a portable calling convention.
- Whether +12 represents a pointer or another kind of value on dependency paths.
- F-1cedbadc4b50’s implementation, side effects, ownership rules, and failure behavior.
- Whether callers guarantee non-overlapping input and output regions.
- Whether callers guarantee that dependency-returned cursors remain within the output region.
- Whether the double discriminator read has significance in the intended memory model.
- Behavior under concurrent mutation, invalid addresses, invalid pointer advancement, or dependency-induced invalidation.

The likely intent is bounded emission from a tagged object, potentially delegating part of that emission. That interpretation does not strengthen the observed equality-only capacity guard into a general bounds guarantee.

## Evidence

Established static evidence consists solely of the supplied function representation and opaque dependency mapping. It supports the byte offsets, discriminator groups, two-stage discriminator access, dependency-call ordering, post-call +16 read, zero filters, equality-only write guard, and cursor advancement described above.

Names such as “input object,” “output cursor,” “boundary,” and “append” are behavioral descriptions, not recovered domain semantics.

No dependency implementation, caller contract, runtime trace, or platform specification was supplied. No runtime tests have been performed.
