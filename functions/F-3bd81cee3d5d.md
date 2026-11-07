<!-- Trusted provenance: F-3bd81cee3d5d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3bd81cee3d5d

## Purpose

F-3bd81cee3d5d performs ordered cleanup of five values associated with an object, clears the object's first 32-bit word, and then performs metadata-driven accounting and block-management operations against a shared context.

The behavior resembles destruction of an object followed by release through a custom allocator. That interpretation is **inferred intent**, not an established ownership contract: the supplied evidence does not establish what the cleanup dependencies release, whether the block-management dependency releases storage, or whether any dependency changes shared state.

A null object pointer is a supported no-op. A non-null pointer is not validated before its associated values are passed to dependencies. The explicit metadata check occurs only after those calls and the clearing of the first object word. Consequently, invalid metadata does not prevent earlier side effects.

## Interface

**Input:** one pointer to an object whose visible fields are arranged as 32-bit words.

**Output:** no return value. Normal completion returns control to the caller. A metadata failure invokes a dependency identified as non-returning; this path has no normal return.

### Required memory layout

The following offsets are relative to the supplied object pointer:

| Byte offset | Width | Established use |
|---|---:|---|
| −8 | 32 bits | Allocation metadata; tested for validity and masked for accounting |
| −4 | 16 bits | Unsigned selector for one context-relative accounting location |
| −2 | 16 bits | Unsigned selector for another context-relative accounting location |
| 0 | 32 bits | Set to zero after the five initial dependency calls |
| +24 | 32 bits | Passed to F-453bd05dbaa8 |
| +28 | 32 bits | Passed to F-453bd05dbaa8 |
| +32 | 32 bits | Passed to F-ca75d2f72fc2 |
| +36 | 32 bits | Passed to F-ca75d2f72fc2 |
| +40 | 32 bits | Passed to F-ca75d2f72fc2 |

The two 16-bit selectors occupy adjacent halves of the four bytes immediately before the object. Their interpretation is by address and width; no byte-order assumption is needed to distinguish the two accesses.

No direct use of object offsets +4 through +20 is visible. The function does not explicitly clear the five values passed to cleanup dependencies.

### Preconditions for ordinary completion

For a non-null input:

- The five object fields must remain readable when their respective calls are reached.
- The first object word must be writable when it is cleared.
- The eight bytes preceding the object must support the metadata accesses.
- The shared-context reference and every computed accounting address must be valid for the corresponding read or write.
- Dependencies must return where subsequent behavior is expected.
- Any memory required after a dependency call must still be valid afterward.

These are access requirements, not checks implemented by this function. In particular, the evidence does not establish that arbitrary non-null input is safe.

The first metadata word must be nonzero and have its least significant bit clear to pass the explicit check. This condition alone does not establish that the block, selectors, or context are valid.

## Behavior

### Null input

A null pointer causes an immediate return.

This path performs none of the five cleanup calls, does not clear memory, does not invoke the context-preparation dependency, and does not inspect allocation metadata or shared accounting.

### Ordered object cleanup

For a non-null pointer, the function performs these calls in order:

1. F-453bd05dbaa8 receives the 32-bit value at object offset +24.
2. F-453bd05dbaa8 receives the 32-bit value at object offset +28.
3. F-ca75d2f72fc2 receives the 32-bit value at object offset +32.
4. F-ca75d2f72fc2 receives the 32-bit value at object offset +36.
5. F-ca75d2f72fc2 receives the 32-bit value at object offset +40.

Each argument is obtained when its call is reached. The supplied evidence does not show an initial snapshot of all five fields. A dependency that changes a later field can therefore affect a later argument.

There are no null checks on these field values. Zero values are passed through to the respective dependencies.

After all five calls return, the function writes a 32-bit zero at object offset 0. The write is not performed before the cleanup calls.

### Context selection and metadata validation

After clearing the first object word, the function invokes F-2fd8826557a4 without an explicit argument.

It then reads the shared context reference and retains the resulting context address for subsequent accounting and final dependency calls. Because the reference is read after F-2fd8826557a4, changes that dependency makes to the reference can affect which context is selected.

The allocation-header address is eight bytes before the object pointer. The first 32-bit word at that address is checked:

- A zero word fails.
- A word with bit 0 set fails.
- A nonzero word with bit 0 clear passes this particular check.

Failure invokes F-a817d8d58c10, marked as non-returning in the supplied evidence. The function does not provide a recoverable error value.

The check occurs after the five cleanup calls, the zero write, the context-preparation call, and the shared-context read. None of those preceding effects is rolled back.

### Accounting preparation and first four updates

After successful metadata validation, F-c0fc6d033caa is called with the retained context address.

Only after that call returns does the function obtain the first unsigned 16-bit selector, from object offset −4. It also obtains the accounting amount by masking the current 32-bit header word with **0x3ffffffc**.

The mask retains bits 2 through 29 and clears bits 0, 1, 30, and 31. Thus:

- The amount is a multiple of four.
- Its representable range is 0 through 0x3ffffffc, inclusive.
- A header that passes validation can still produce a zero amount.
- A passing header need not have bits 1, 30, or 31 clear.

The function subtracts this amount from four 32-bit accounting words, in the following order:

| Order | Accounting address relative to retained context |
|---:|---|
| 1 | 0x224 plus four times the unsigned selector from object offset −4 |
| 2 | 0x30c |
| 3 | 0xe0 plus four times the unsigned selector from object offset −2 |
| 4 | 0x220 |

The second selector is read when the third update is prepared, after the first two accounting updates. It is not visibly captured alongside the first selector.

The same captured masked amount is used for all four updates. No underflow check or accounting-consistency check is present.

F-c0fc6d033caa precedes the selector and amount reads. The specification therefore does not assume that metadata observed before this dependency remains unchanged afterward, or that the header is revalidated afterward.

### Count updates and final byte accounting

After the first four accounting updates, the function invokes F-09dcb55e8e0a without an explicit argument.

Its integer result selects a 32-bit count word at:

**retained context + 0x310 + four times the result**

The function decrements that count word by one. It then decrements the 32-bit count word at context offset 0x390 by one.

There is no visible validation of the returned selector and no check for a zero count before either decrement.

Finally, the function rereads the 32-bit allocation-header word, masks it with **0x3ffffffc**, and subtracts that newly obtained amount from the 32-bit word at context offset +8.

This last amount is not necessarily the amount used for the earlier four subtractions. F-09dcb55e8e0a runs between the two header-derived amounts, and earlier writes could also matter if storage aliases. The supplied evidence does not establish that either situation occurs, but it does not justify assuming the values must be equal.

### Final block-management calls

After all seven accounting updates, the function calls:

1. F-cc6c9b4a0f68 with the retained context address and the allocation-header address.
2. F-e586bff64fb7 with the retained context address.

Normal return follows if both dependencies return.

The pointer supplied to F-cc6c9b4a0f68 is the address eight bytes before the object, not the object pointer itself.

The retained context address is reused even if an intervening dependency changes the shared context reference.

## Side effects

### Direct writes

On a successful non-null path, the function directly performs:

- One 32-bit zero write to the object.
- Four amount-subtraction updates to context-relative accounting words.
- Two decrement updates to context-relative count words.
- One further amount-subtraction update at context offset +8.

These are seven accounting updates, not necessarily seven distinct locations. No selector-range constraints or non-aliasing guarantees are visible. Computed addresses can coincide with fixed accounting addresses or other relevant storage unless external invariants prevent that.

If locations overlap, the updates remain ordered. A later update operates on the state then present; the function does not compute all resulting values from one original accounting snapshot.

### Dependency effects and ownership

Dependency side effects are unknown. The function itself does not establish:

- Whether the five associated values represent owned resources.
- Whether their dependencies release, retain, mutate, or merely inspect them.
- Whether the block-management call releases the object's underlying storage.
- Whether any dependency changes global references or allocation metadata.
- Whether any dependency acquires or releases synchronization.

A caller must not infer continued object usability, successful deallocation, or ownership transfer solely from normal return.

### Failure behavior

The only explicit rejection is the allocation-header check, which invokes a non-returning dependency. Other invalid pointers, invalid selectors, accounting underflow, and dependency failures have no visible local recovery path.

The sequence is not transactional. A failure or non-returning dependency can occur after prior cleanup or accounting changes.

## Dependencies

| Opaque function ID | Invocation and established role |
|---|---|
| F-453bd05dbaa8 | Called twice, with object values from +24 and +28 |
| F-ca75d2f72fc2 | Called three times, with object values from +32, +36, and +40 |
| F-2fd8826557a4 | Called after the object zero write and before the shared-context reference is read |
| F-a817d8d58c10 | Invoked on zero or odd allocation metadata; marked non-returning |
| F-c0fc6d033caa | Receives the retained context before accounting metadata is read |
| F-09dcb55e8e0a | Returns the selector used for one context-relative count decrement |
| F-cc6c9b4a0f68 | Receives the retained context and the allocation-header address after accounting |
| F-e586bff64fb7 | Receives the retained context after the preceding block-management call |

Names such as “cleanup,” “preparation,” and “block-management” describe placement and inferred purpose. They do not add guarantees about dependency internals.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

### Null object

Given a null input, the function returns without invoking dependencies or accessing object or accounting storage.

### Ordered cleanup with a changed later field

Given a valid non-null object, suppose the first F-453bd05dbaa8 call changes the object word at +28 and returns.

The second F-453bd05dbaa8 call receives the changed value. An implementation that snapshots all five arguments before the first call would not preserve the established access ordering.

### Zero metadata

Given dependencies that return and leave the relevant storage accessible, suppose the header word is zero when checked.

The five initial cleanup calls and object zero write have already occurred. F-2fd8826557a4 and the shared-context read have also occurred. F-a817d8d58c10 is then invoked, with no subsequent accounting or final block-management calls.

### Passing metadata with a zero amount

Suppose the header word is 0x00000002 and remains unchanged.

It passes the explicit check because it is nonzero and bit 0 is clear. Its masked accounting amount is zero. The amount-subtraction updates still occur, the two count decrements still occur, and the final dependencies are still reached if all preceding operations complete.

### Ordinary accounting without aliases

Suppose the header word remains 0x00000102, both 16-bit selectors are zero, F-09dcb55e8e0a returns zero, and all addressed words are valid and distinct.

The amount is 0x100. The words at context offsets 0x224, 0x30c, 0xe0, and 0x220 each decrease by 0x100. The counts at 0x310 and 0x390 each decrease by one. The word at +8 then decreases by 0x100.

### Metadata changes before the final subtraction

Suppose the earlier masked amount is 0x100, and F-09dcb55e8e0a changes the header so its masked amount becomes 0x200.

The first four amount updates use 0x100. The final update at context offset +8 uses 0x200. No consistency comparison rejects this difference.

### Shared reference changes after selection

Suppose the shared reference identifies context A when read, and a later dependency changes that reference to context B.

Subsequent context-relative operations and final dependency arguments still use retained context A. This does not imply that A remains valid; its continued validity is an external requirement.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning, ownership, and lifetime of the object and its five associated values.
- The identity and invariants of the shared context.
- Whether the preparation and final dependencies implement locking.
- Legal selector ranges and the allocation extent of the accounting structures.
- Whether selectors, headers, objects, and accounting words are guaranteed not to alias.
- The legal result range of F-09dcb55e8e0a.
- Whether abnormal dependency behavior terminates, unwinds, blocks, or otherwise transfers control.
- The execution environment's precise rules for address overflow, alignment faults, and signed arithmetic overflow.

The evidence identifies 32-bit words, unsigned 16-bit metadata selectors, four-byte accounting strides, and the stated 32-bit mask. Each unsigned selector ranges from 0 through 65,535, so its scaled contribution ranges from 0 through 262,140 before adding the context and fixed offset.

The returned selector is integer-typed rather than established as an unsigned 16-bit value. No bounds check or negative-result exclusion is visible.

Amount-subtraction expressions involve an unsigned 32-bit amount. The count decrements are presented through signed integer accesses. A portable signed-overflow guarantee must not be inferred for those decrements, and no widening arithmetic is established.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

**Established static evidence:** the null return, field offsets and access widths, dependency order and arguments, object zero write, context-read timing, metadata predicate, accounting mask, ordered accounting updates, final header reread, and final dependency sequence.

**Inferred intent:** object destruction, resource cleanup, allocator accounting, and eventual block release.

**Unknown:** dependency implementations, synchronization guarantees, ownership rules, valid accounting ranges, runtime memory mappings, and actual failure behavior beyond the marked non-returning call.

No runtime tests have been performed.
