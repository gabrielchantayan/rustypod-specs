<!-- Trusted provenance: F-aa47c0abc40b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa47c0abc40b

## Purpose

F-aa47c0abc40b conditionally decrements a 32-bit field associated with a caller-held address and selects further actions according to whether the decremented value is zero. Every established normal-return path leaves the caller-held address slot equal to zero.

The structure suggests a release operation involving reference-counted state. That interpretation is **inferred intent**, not an established ownership contract. The supplied representation establishes field accesses, calls, branching, and slot clearing; it does not establish that any dependency frees memory, invokes a destructor, balances synchronization, or preserves global state.

Two distinctions are essential:

- The address slot is reread at multiple points. The function does not consistently operate on one captured object address.
- Two calls are marked as non-returning in the supplied representation. Paths reaching those calls do not have an established normal return or a guaranteed final slot clear.

## Interface

### Input

The function accepts one argument: an address of a writable slot containing a 32-bit address value.

The argument is the address of the slot, not the address stored in that slot. A null value **inside** a valid slot is supported. A null or otherwise inaccessible slot address is not checked before access.

The supplied representation uses a 32-bit address and integer model. Within that model:

| Location | Width | Observed use |
|---|---:|---|
| Caller-held slot | 32 bits | Address value; repeatedly read and eventually cleared on normal completion |
| Selected object address + 0 | 32 bits | Address used to begin an indirect callback lookup |
| Selected object address + 4 | 32 bits | Integer field decremented by one |
| Retained object address + 8 | 32 bits | Value tested and potentially passed to a terminal dependency |
| Callback descriptor address + 0 | 32 bits | Address of a further table |
| Further table address + 4 | 32 bits | Indirect callable address |

These are accessed locations, not a complete object declaration. The representation does not establish the total allocation size, required alignment, ownership of the referenced regions, or meaning of unaccessed bytes.

### Output

There is no explicit return value.

On established normal completion, the caller-held slot contains zero. This is an externally visible output even when no other operation occurs.

No error code, exception contract, or recovery result is provided.

### Preconditions

For a normally executing path, every location actually accessed on that path must be valid for its respective read or write. In particular:

- The caller-held slot must be readable and writable.
- A nonzero address selected for the decrement must permit a 32-bit access at offset 4.
- The zero-result branch may require a readable word at object offset 0 and additional readable callback lookup locations.
- A retained nonzero address used later must permit a 32-bit read at offset 8.
- An indirect callback target, when selected, must be callable under the applicable calling convention.

These conditions apply to the values observed **at each access**. Satisfying them only for the entry-time address is insufficient if a dependency, alias, or concurrent participant changes the slot.

## Behavior

### Entry and empty-slot handling

The first observable data access is a read of the caller-held slot.

If that read yields zero, the function returns immediately:

- No dependency is invoked.
- No object-relative field is accessed.
- No explicit clearing write is performed.

The slot therefore remains zero in an execution without an external intervening write. The stronger statement that the function itself writes zero on this path would be incorrect.

If the initial value is nonzero, the function calls F-51fc13af409a with no explicit arguments in the supplied representation. It does not capture and retain the initial object address before this call.

### Decrement and branch selection

After F-51fc13af409a returns, the function rereads the caller-held slot and reads the 32-bit integer at offset 4 from that newly observed address. It computes that integer minus one.

The function then rereads the caller-held slot for the destination address and stores the computed value at offset 4 from that address.

Consequently, the read and write are not established to target the same object if the slot changes between them. The branch decision uses the computed result, not a later reread of the stored field.

For values whose signed 32-bit subtraction is representable:

- An old value of 1 produces zero and selects the zero-result branch.
- An old value of 2 produces 1 and selects the nonzero-result branch.
- An old value of 0 produces −1 and also selects the nonzero-result branch.
- An old value of −1 produces −2 and selects the nonzero-result branch.

There is no check that the old value is positive. There is no saturation at zero or explicit underflow error handling. The minimum signed 32-bit value requires separate treatment under **Uncertainties**.

### Nonzero-result branch

If the computed value is not zero, the function invokes F-0923d65e0f4b with no explicit arguments in the supplied representation.

If that call returns normally, the function writes a 32-bit zero to the caller-held slot and returns.

This branch does not perform the callback lookup and does not itself read an object field at offset 8. Its dependency may have additional effects, which are unknown.

### Zero-result branch: optional indirect callback

If the computed value is zero, the function rereads the caller-held slot and reads the word at offset 0 of the selected object.

If that word is zero, the callback is skipped.

If that word is nonzero, the function performs a callback lookup using a fresh read of the caller-held slot. The lookup has the following structure:

1. The selected object's word at offset 0 supplies a descriptor address.
2. The word at offset 0 of that descriptor supplies a further table address.
3. The word at offset 4 of that further table supplies the callable address.

The callback is invoked with no explicit arguments in the supplied representation.

The nonzero check covers only the object's first word as observed during the check. It does not validate the further table address or the callable address. Because the lookup rereads the slot and object data, it is not established to use the same descriptor value that passed the check.

In particular, the callback address is **not** established to reside directly at offset 4 from the object's first pointer. There is an intervening read of the descriptor's first word.

### Zero-result branch: dependency call and retained address

After the optional callback returns, or immediately if it was skipped, the function rereads the caller-held slot and passes that current 32-bit address value as an explicit argument to F-0923d65e0f4b.

This differs from the nonzero-result branch, where the same dependency is invoked without an explicit argument. The representation does not justify assuming that the dependency has one conventional fixed signature across both sites.

After this call returns, the function rereads the caller-held slot and retains the result for the remaining object-relative work.

If this retained value is zero, no offset-8 field is read. The function proceeds to the final slot-clearing write and returns.

If the retained value is nonzero, the function reads its 32-bit field at offset 8.

### Zero-result branch: terminal calls

If the retained object's offset-8 field is zero, the function passes the retained object address to F-feee9156a257.

If that field is nonzero:

- The function first calls F-a600efb7b035 with no explicit arguments.
- After that call returns, it rereads the field at offset 8 from the **retained object address**.
- It passes this newly read field value to F-feee9156a257.

The second field read is significant. The argument passed to F-feee9156a257 need not equal the value that caused the nonzero branch. F-a600efb7b035 could change that field, including changing it to zero.

Conversely, changing the caller-held slot during F-a600efb7b035 does not redirect this final field read: the object address was already retained.

Both calls to F-feee9156a257 are marked as non-returning in the supplied representation. Under that control-flow model, neither reaches the final slot-clearing write.

### Final slot clearing

The shared normal-completion operation is a 32-bit zero write to the caller-held slot.

This operation follows:

- A normal return from the nonzero-result branch's F-0923d65e0f4b call; or
- A normal return from the zero-result branch's F-0923d65e0f4b call followed by observation of a zero slot.

It is not an unconditional cleanup guarantee. Faults, non-returning dependencies, callback transfers that do not return, or other abnormal exits can prevent it.

## Side effects

### Direct memory effects

The function can directly perform:

- One decrement-result write to an object-relative 32-bit field at offset 4.
- One final 32-bit zero write to the caller-held slot.

The initially empty-slot path performs neither write.

The supplied representation does not provide an atomic read-modify-write operation for the decrement. It establishes a load, arithmetic, and a store, with a separate slot read supplying the store destination. Whether surrounding dependencies provide synchronization is unknown.

The slot and accessed object fields are not shown to be disjoint. Aliasing can therefore affect subsequent observations. The contract must not assume conventional object layout or independent storage beyond the addresses actually supplied.

### Indirect effects

All invoked dependencies and the indirect callback may have side effects not described here. No preservation guarantee is established for:

- The caller-held slot;
- Object fields or descriptor data;
- Other memory or global state;
- Allocations, resource ownership, or synchronization state.

The function's repeated reads remain behaviorally relevant even if a likely implementation of a dependency would normally leave these locations unchanged.

### Ownership and lifetime

Clearing the slot removes the stored address from that location. It does not, by itself, prove that an allocation was released.

The decrement, callback, and terminal calls suggest lifetime management, but no dependency body or independent contract establishes which component owns, releases, or invalidates any referenced storage.

In particular, the function accesses the slot after F-0923d65e0f4b returns and may access object storage after F-a600efb7b035 returns. Valid continuation requires those later accesses to remain permissible; that is a precondition, not a proven dependency guarantee.

## Dependencies

| Opaque function ID | Invocation condition | Explicit arguments shown | Established caller behavior |
|---|---|---|---|
| F-51fc13af409a | Initial slot read is nonzero | None | Called before the decrement; slot is reread afterward |
| F-0923d65e0f4b | Computed decrement result is nonzero | None | Normal return is followed by slot clearing |
| F-0923d65e0f4b | Computed decrement result is zero, after optional callback | Current slot value | Slot is reread after return |
| F-a600efb7b035 | Retained address is nonzero and its offset-8 field initially reads nonzero | None | Offset-8 field is reread after return |
| F-feee9156a257 | Retained address is nonzero and its offset-8 field reads zero | Retained object address | Marked non-returning |
| F-feee9156a257 | After F-a600efb7b035 returns | Fresh offset-8 field value from retained object | Marked non-returning |

An additional indirect dependency is selected through the descriptor and table lookup. No opaque function ID is available for that target. Its signature, effects, and termination behavior are unknown beyond the call site's lack of explicit arguments.

“No explicit arguments” describes the supplied call expression. It does not prove the absence of relevant register state, an implicit receiver, or another ABI-level input.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Except where a change is specified, assume dependencies return normally, accessed memory remains valid, slot and object-field storage do not overlap, and no external participant changes relevant state.

### Initially empty slot

Given a valid writable slot containing zero:

- The function returns without invoking a dependency.
- It does not access object-relative memory.
- It performs no explicit slot write.

### Decrement remains nonzero

Given a slot containing address A and a value of 2 at A + 4:

- F-51fc13af409a runs first.
- A + 4 receives 1.
- F-0923d65e0f4b is called without an explicit argument.
- The slot is cleared before normal return.

No callback lookup or offset-8 read is performed directly by this function.

### Zero input count is not treated as empty

Given a nonzero slot containing A and a value of 0 at A + 4:

- The stored result is −1.
- The nonzero-result branch is selected.
- There is no direct rejection of the count value.

An empty slot and a zero count are therefore different cases.

### First dependency redirects the operation

Given an initially nonzero slot containing A, if F-51fc13af409a changes it to valid address B before returning:

- The decrement reads from B + 4, not A + 4.
- With no later change, the result is stored at B + 4.

If that dependency instead clears the slot, the function does not repeat the entry guard. Its next object-relative access lacks an established valid target.

### Zero result with no callback and cleared slot

Given a count of 1 and a zero first object word:

- The count becomes zero.
- No indirect callback is invoked.
- F-0923d65e0f4b receives the current slot value.

If that dependency clears the slot before returning, the function performs no offset-8 read and completes with a final zero write to the slot.

### Callback lookup through two address-bearing words

Given a zero decrement result, an object first word pointing to descriptor D, and D's first word pointing to table T:

- The callable address is read from T + 4.
- That callback is invoked before F-0923d65e0f4b.

A target stored merely at D + 4 does not satisfy this lookup.

### Terminal argument changes across the intervening call

Given a retained object A whose offset-8 field initially contains nonzero value X:

- F-a600efb7b035 runs.
- If it changes A + 8 to Y, F-feee9156a257 receives Y, not necessarily X.
- This remains true if Y is zero.

If it instead changes the caller-held slot to B while leaving A + 8 unchanged, the final field read still uses A + 8.

### Retained object with zero offset-8 field

Given a retained nonzero address A and zero at A + 8:

- F-feee9156a257 receives A itself.
- F-a600efb7b035 is not called.
- Under the supplied non-return model, the final slot-clearing write is not reached.

## Uncertainties

The following are **documented_only**:

- **Arithmetic overflow:** The representation uses a signed 32-bit subtraction. It does not independently establish behavior when subtracting one from −2,147,483,648. Modular wraparound must not be promoted to a guaranteed result without lower-level evidence.
- **Address arithmetic:** Valid examples require that the indicated field addresses exist without problematic address-space wrapping. No portable wrapping or fault behavior is established.
- **Dependency contracts:** Names and call placement do not prove locking, unlocking, destruction, deallocation, error reporting, or termination semantics.
- **Non-return annotations:** The representation marks F-feee9156a257 as non-returning, but its body is unavailable. Behavior if it unexpectedly returns is outside the established control-flow contract.
- **Calling conventions:** The inconsistent explicit argument presentation for F-0923d65e0f4b is unresolved. A fixed prototype must not be invented.
- **Concurrency:** No language memory model, atomicity guarantee, or permission for concurrent mutation is supplied. Repeated reads are established; safe concurrent use is not.
- **Invalid accesses:** No defined fault type, exception, recovery action, or diagnostic is established.
- **Resource ownership:** No allocation provenance, ownership transfer, or release guarantee is supplied.

The lifecycle-management interpretation remains inferred intent only.

## Evidence

This contract is based solely on the supplied static representation of F-aa47c0abc40b and the supplied opaque dependency mapping.

Established evidence comprises the visible access widths and offsets, repeated slot reads, decrement and zero comparison, callback lookup depth, explicit call arguments, ordering of calls and field accesses, final slot write, and non-return annotations.

Dependency bodies, ABI details, allocation metadata, synchronization guarantees, and runtime observations were not supplied. No runtime tests have been performed.
