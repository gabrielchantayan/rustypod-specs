<!-- Trusted provenance: F-cbee4e26b4d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cbee4e26b4d1

## Purpose

F-cbee4e26b4d1 performs a dispatch-selected operation on a shared destination, obtains a final 32-bit value from that destination, and stores that value through a caller-supplied output pointer. On every visible path that reaches normal completion, it returns the 32-bit value 1.

Four selector values receive specialized handling: `0x6D54`, `0x6D55`, `0x6D56`, and `0x6D59`. All other values use a generic preparation, consumption, and cleanup sequence.

Before dispatch, the function may perform a gated preliminary operation. This preliminary operation is independent of the selector and may affect the state subsequently observed by dispatch.

The apparent role of the shared destination is inferred from its repeated use across dependencies. Whether it represents text, a stream, a structured result, or another resource is not established. This contract therefore describes calls and observable data movement without assigning undocumented meanings to dependency results.

## Interface

### Inputs

The visible interface has four arguments:

| Argument | Visible representation | Use |
|---|---|---|
| Object reference | Address carried in a 32-bit integer-shaped argument | Supplies fields at byte offsets `0x5C`, `0x84`, and `0x8C` on selected paths; also passed unchanged to the generic preparation dependency |
| Selector | 32-bit integer | Compared with the four specialized selector values |
| Output pointer | Pointer to a writable 32-bit word | Copied into temporary state on entry; receives the final result after dispatch |
| Auxiliary value | Opaque 32-bit word | Copied into temporary state on entry; available to the generic path through that state |

The interface is shown in a 32-bit representation. The object reference is used as an address despite its integer-shaped declaration. No wider-pointer interface is established.

The output pointer identifies a location that must be writable when the final store occurs. The function does not inspect its previous contents directly.

### Object layout accessed directly

Offsets are measured in bytes from the object reference.

| Offset | Width | Direct use |
|---|---:|---|
| `0x5C` | 32 bits | Zero/nonzero choice for `0x6D54`; opaque third call argument for `0x6D55`; captured payload for `0x6D56`; integer-shaped argument for `0x6D59` |
| `0x84` | 32 bits | Payload for `0x6D59`, read after its lookup completes |
| `0x8C` | 32 bits | Integer-shaped value increased by one for `0x6D56` |

No object-field read is visible in this function for a selector outside the four specialized values. The generic dependency may still inspect the object.

### External state and fixed operands

The function accesses these logically distinct external operands:

- A control-record reference, with a 32-bit flag word at byte offset `0x0C`.
- A preliminary gate operand.
- A shared destination reference.
- Two fixed operands used by the preliminary operation.
- Two distinct fixed lookup keys for the zero and nonzero cases of `0x6D54`.
- A fixed lookup key for `0x6D56`.
- A fixed lookup key for `0x6D59`.
- An address identifying fixed data for `0x6D55`.

These labels identify visible roles, not inferred data types. The underlying contents, encodings, storage lifetimes, and mutability are not supplied.

### Result

After dispatch, the function calls F-ecd1af0519f9 with the then-used shared destination reference and stores its 32-bit return value through the original output pointer.

The function then returns 1 as a 32-bit result.

That return value does not distinguish among selectors, report dependency failure, or certify the semantic validity of the stored output.

## Behavior

### Entry state

Before the preliminary operation, the function initializes temporary local state with:

1. A copy of the output pointer.
2. A copy of the auxiliary 32-bit argument.

The generic path receives the address of the first temporary slot. It does not receive the caller’s output location as that first argument: it receives a local slot containing a copy of the output pointer.

The visible representation includes a neighboring auxiliary slot. The complete layout and interpretation expected by dependencies are not established merely by the address of the first slot.

Changing the temporary pointer slot does not, by itself, redirect the function’s final output store. That store uses the original output-pointer argument.

### Preliminary gate

Before examining the selector, the function reads the 32-bit control word at control-record offset `0x0C` and tests bit 0.

- If bit 0 is set, it skips the entire preliminary dependency sequence.
- If bit 0 is clear, it calls F-b255b26284e6 with the preliminary gate operand.
- If that call returns zero, no further preliminary call occurs.
- If that call returns any nonzero value, the function performs the following sequence:
  1. Calls F-5b24edfb22c0 with the shared destination reference.
  2. Passes that call’s 32-bit result, followed by the two fixed preliminary operands in their visible argument order, to F-804adc29bbfb.
  3. Calls F-d08d3ca17b81 with the preliminary gate operand.

The returns from F-804adc29bbfb and F-d08d3ca17b81 are not used by this function.

Dispatch follows this sequence. Consequently, object fields read during dispatch may reflect changes made by preliminary dependencies. The function does not snapshot the object before the preliminary gate.

### Selector `0x6D54`

The function reads the 32-bit field at object offset `0x5C` and chooses a branch based solely on whether that value equals zero.

It then:

1. Calls F-9d0ecc5f6adf with no visible arguments.
2. Obtains the fixed lookup key corresponding to the previously chosen zero or nonzero branch.
3. Calls F-c0b020cd3882 with the first call’s result and that key.
4. Calls F-151ab4bfe553 with the shared destination reference and the lookup result.

Every nonzero field value chooses the same nonzero branch. There is no distinct handling for negative values or individual nonzero bit patterns.

The zero/nonzero decision occurs before F-9d0ecc5f6adf. A later change to the object field does not cause a new branch decision within this invocation.

### Selector `0x6D55`

The function calls F-1f62160dfd52 with three visible arguments:

1. The shared destination reference.
2. The address of the selector-specific fixed data.
3. The 32-bit value read from object offset `0x5C`.

There is no context-acquisition or lookup call on this branch.

The object field is forwarded as a 32-bit value without visible conversion or validation. Its interpretation depends on F-1f62160dfd52 and the fixed data, neither of which is defined here.

### Selector `0x6D56`

The function performs these reads and calls in order:

1. Captures the 32-bit value at object offset `0x5C`.
2. Reads the integer-shaped 32-bit value at offset `0x8C` and forms that value plus one.
3. Calls F-9d0ecc5f6adf.
4. Calls F-c0b020cd3882 with that result and the fixed key for this selector.
5. Calls F-1f62160dfd52 with four visible arguments:
   - The shared destination reference.
   - The lookup result.
   - The incremented value derived from offset `0x8C`.
   - The previously captured value from offset `0x5C`.

Both object-derived arguments are captured before the context and lookup calls. Later changes to those fields by either dependency do not replace the captured arguments.

For values whose increment is representable as a signed 32-bit integer, the visible arithmetic is addition by one. Behavior at signed overflow is not established by the supplied representation; wraparound, saturation, and rejection must not be assumed.

### Selector `0x6D59`

The function:

1. Captures the integer-shaped 32-bit field at object offset `0x5C`.
2. Calls F-9d0ecc5f6adf.
3. Calls F-c0b020cd3882 with that result and the fixed key for this selector.
4. Reads the 32-bit field at object offset `0x84`.
5. Calls F-1f62160dfd52 with four visible arguments:
   - The shared destination reference.
   - The lookup result.
   - The earlier value from offset `0x5C`.
   - The later value from offset `0x84`.

The timing difference between these field reads is observable if either intervening dependency changes the object. The first field is captured before those calls; the second is read after them.

Unlike `0x6D56`, this branch does not add one to an object-derived argument.

### All other selectors

Every other selector, including nearby values such as `0x6D57` and `0x6D58`, follows the same generic path:

1. F-0dcd1529f252 receives the address of the temporary pointer slot, the original object reference, and the original selector.
2. F-46f0b3cb316e receives the shared destination reference and the address of that same temporary slot.
3. F-289ec4311c64 receives the address of that same temporary slot.

These calls are ordered. The latter calls can observe changes made to temporary state by earlier calls.

There is no visible check of their return values. F-289ec4311c64 is invoked only on this generic path, before final extraction. Calling it “cleanup” describes its position and apparent role, not an established guarantee that it releases memory or invalidates the temporary state.

### Common completion

After any dispatch branch completes normally:

1. F-ecd1af0519f9 is called with the shared destination reference.
2. Its 32-bit result is written through the original output pointer.
3. The function returns 1.

The final output store occurs after all visible dispatch calls, including the generic path’s F-289ec4311c64 call.

## Side effects

### Direct memory effects

The function directly:

- Initializes temporary local state.
- Reads the preliminary control word.
- Reads the selector-dependent object fields described above.
- Stores one 32-bit result through the original output pointer on normal completion.

No direct object-field write or direct control-word update is visible.

### Dependency-mediated effects

Dependencies may change shared state, mutate objects, retain references, allocate memory, release memory, or fail to return. Their implementations are unavailable. None of those effects may be ruled out merely because their return values are ignored.

The function does not establish that the shared destination remains identical across calls. Where the external destination reference is accessed again after a dependency call, its later value must not be replaced contractually by an assumed entry-time snapshot. The same caution applies to other external operands.

The supplied evidence establishes dependency-call ordering and explicitly separated field reads. It does not establish an ordering among all independent argument reads within a single call expression.

### Ownership and lifetime

No ownership transfer is established for the output pointer, object, auxiliary word, lookup results, or final result.

In particular:

- The temporary slot initially contains a borrowed copy of the output-pointer value; copying it is not evidence of ownership transfer.
- F-289ec4311c64’s presence does not prove allocation or release.
- A result stored in the output word may be a scalar, handle, or pointer-shaped value; its meaning and disposal requirements are unknown.
- Dependencies’ retention of temporary addresses is unknown. Safe escape of those addresses beyond the invocation is not established.

### Preconditions and errors

No explicit null-pointer, range, alignment, or selector-validity checks are visible.

The output location must support the final 32-bit store. The control record must support the initial control-word read. Specialized branches require their corresponding object fields to be readable at the time of access. Dependencies may impose additional preconditions.

Unrecognized selectors are not rejected; they are delegated to the generic path.

No explicit error status, rollback, or exception-handling mechanism is visible. If a dependency does not return normally, the common output store and return of 1 are not guaranteed to occur.

## Dependencies

| Opaque ID | Visible use |
|---|---|
| F-b255b26284e6 | Conditional preliminary query; zero versus nonzero controls further preliminary calls |
| F-5b24edfb22c0 | Produces the first argument for the preliminary operation |
| F-804adc29bbfb | Receives that result and two fixed preliminary operands |
| F-d08d3ca17b81 | Runs after the preliminary operation |
| F-9d0ecc5f6adf | Produces a lookup input for three specialized selectors |
| F-c0b020cd3882 | Combines that input with a selector-specific key; result forwarded to another dependency |
| F-151ab4bfe553 | Receives destination and lookup result for `0x6D54` |
| F-1f62160dfd52 | Receives three visible arguments for `0x6D55`, four for `0x6D56` and `0x6D59` |
| F-0dcd1529f252 | Prepares or modifies temporary state on the generic path |
| F-46f0b3cb316e | Receives destination and temporary-state address after generic preparation |
| F-289ec4311c64 | Receives temporary-state address after generic consumption |
| F-ecd1af0519f9 | Produces the final 32-bit value stored through the output pointer |

The differing visible argument counts for F-1f62160dfd52 do not establish its complete calling convention or whether it is variadic.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not results of executed tests. Each assumes required memory is valid and invoked dependencies return normally.

1. **Preliminary operation disabled by flag.**  
   With control bit 0 set, none of the four preliminary dependencies is called. Dispatch and common completion still occur.

2. **Preliminary query returns zero.**  
   With control bit 0 clear and F-b255b26284e6 returning zero, the query occurs but F-5b24edfb22c0, F-804adc29bbfb, and F-d08d3ca17b81 do not.

3. **Preliminary query returns a non-unit nonzero value.**  
   A result such as 2 enables the preliminary sequence just as 1 does. That sequence completes before selector-specific object reads.

4. **Zero versus nonzero lookup choice.**  
   For `0x6D54`, field value zero selects the zero-case key. Values 1 and all-bits-set both select the nonzero-case key.

5. **Incremented argument.**  
   For `0x6D56`, with offset `0x8C` equal to 7 and offset `0x5C` containing an opaque value P, F-1f62160dfd52 receives 8 and P as its final two arguments, even if lookup dependencies subsequently change those fields.

6. **Late payload read.**  
   For `0x6D59`, if offset `0x5C` initially contains 4 and a lookup dependency changes offset `0x84` from A to B, the final call receives 4 and B, provided B remains present at the later read.

7. **Generic temporary mutation.**  
   For `0x6D57`, changes F-0dcd1529f252 makes to the temporary pointer slot are available to the next two generic dependencies. Such a slot change alone does not redirect the final store away from the original output pointer.

8. **Common output behavior.**  
   For any normally completing selector path, if F-ecd1af0519f9 returns the 32-bit value R, the original output location receives R and the function returns 1.

## Uncertainties

The following remain **documented_only**:

- Dependency semantics, error behavior, ownership rules, and global-state preservation.
- Fixed-data contents and lookup-key meanings.
- The complete temporary-state layout and how much memory its consumers access.
- The precise machine-level result of overflowing the signed-shaped increment.
- Permitted aliasing between output storage, object fields, and external state.
- Concurrency, synchronization, and reentrancy guarantees.
- Lifetime and disposal requirements for the final 32-bit result.
- Behavior for invalid addresses, unsupported alignment, or dependency contract violations.

The shared destination’s apparent accumulator role and the generic sequence’s apparent preparation/consumption/cleanup roles are inferred intent, not independently established semantics.

## Evidence

The supplied static representation supports the selector comparisons, field offsets, 32-bit accesses, dependency arguments, call order, conditional preliminary sequence, final output store, and constant normal return described above.

It does not supply dependency bodies, fixed-data contents, a complete ABI description, or runtime observations. No runtime tests have been performed. Acceptance examples are therefore conditional specifications rather than verified execution outcomes.
