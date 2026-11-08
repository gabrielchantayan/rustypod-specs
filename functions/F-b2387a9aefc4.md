<!-- Trusted provenance: F-b2387a9aefc4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b2387a9aefc4

## Purpose

F-b2387a9aefc4 transforms an address-delimited range in place, treating successive element positions as 12 bytes apart. It selects between a short-range procedure using comparisons and relocations, and a recursive procedure that processes two subranges before invoking a combining dependency.

**Established static evidence:** the routine uses a 180-byte branch threshold, advances element positions by 12 bytes, explicitly copies four-byte fields at element offset 8, and distinguishes zero from nonzero comparison results. The recursive procedure completes the left subrange before starting the right.

**Inferred intent:** the short-range procedure resembles insertion sorting; the recursive procedure resembles divide-and-combine sorting. These descriptions do not establish sortedness, stability, preservation of all input values, or ownership behavior. Those properties depend on unavailable dependency contracts.

The contract below describes visible operations and their ordering. It does not assume that dependencies preserve globals, source objects, previously exposed local storage, or ownership.

## Interface

The routine accepts three inputs and has no explicit result value.

| Input | Representation | Role |
|---|---|---|
| Range start | 32-bit integer used as an address | First element position and anchor for the short-range procedure. |
| Range end | 32-bit integer used as an address | Intended exclusive boundary. Forward traversal stops on address equality with this value. |
| Context value | Opaque four-byte value | Copied into local storage for comparisons and passed by value to recursive invocations. |

### Element and temporary layout

Element positions are separated by 12 bytes.

- The element base address is supplied to dependencies.
- A four-byte field begins at offset 8.
- Relocation calls are followed by explicit reads and writes of that field.
- The meaning of the field and the access footprint of each dependency are unknown.

The visible local storage includes:

- A first temporary represented by an eight-byte region with an adjacent four-byte slot.
- A second temporary represented by a 12-byte region.
- Two separate four-byte context slots.

The first temporary’s adjacent four-byte slot supplies the trailing field restored on the nonzero-result branch. The routine does not explicitly initialize that slot before using it. Whether temporary preparation initializes it is **documented_only**.

The layout suggests a 12-byte element with an object-like component and an associated field, but it does not prove that dependencies access only the first eight bytes or operate on complete 12-byte objects.

### Preconditions for predictable processing

A conventional caller supplies a forward range whose byte length is a multiple of 12, with valid storage for all accessed elements. Dependencies must support the addresses, temporary storage, overlap patterns, and lifecycle sequence described below.

Additional conditions are necessary:

- Address arithmetic must remain meaningful in the target’s 32-bit representation.
- Backward comparisons must terminate before invalid storage is accessed.
- Recursive partition results must produce valid, strictly smaller subranges.
- Dependencies must return normally for later operations and cleanup to occur.

The routine does not validate these conditions.

## Behavior

### Path selection and arithmetic

The routine subtracts start from end and compares the result with 180 using the signed integer interpretation shown in the supplied representation.

- A difference below 180 selects the short-range procedure.
- A difference of 180 or more selects recursion.

For ordinary forward ranges without arithmetic overflow, 14 elements occupy 168 bytes and use the short-range procedure; 15 elements occupy 180 bytes and select recursion.

This is a byte-difference decision, not a validated element-count decision. Negative differences also satisfy the short-range condition in the represented signed arithmetic.

The relevant address and integer quantities are 32 bits. Operations include endpoint subtraction, multiplication of a partition result by 12, address addition, and subtraction of 12 during backward traversal. Exact overflow behavior is not established. No wrapping, saturation, or diagnostic behavior is promised.

### Short-range traversal

An empty range returns without dependency calls or element writes.

Otherwise, the first prospective candidate is 12 bytes after start. If that address equals end, the routine returns without processing an element. Thus a conventional one-element range performs no dependency calls.

For longer ranges, candidates advance by 12 bytes. Traversal terminates only when the next candidate address equals end; there is no greater-than-end check.

For each candidate, the routine performs this sequence:

1. Writes the original four-byte context input into the first context slot.
2. Calls F-4bbba16cfe23 with the first temporary’s destination address and the candidate address. This call’s return value is unused.
3. Calls F-e46aba7df1fa with the first context-slot address, the first temporary’s address, and range start.
4. Takes the zero-result or nonzero-result branch described below.
5. Calls F-289ec4311c64 on the first temporary’s original destination address.
6. Advances to the next candidate.

The context assignment occurs before temporary preparation, not immediately before comparison. Consequently, the first comparison receives the slot’s contents after the preparation call returns. The routine itself performs no intervening write, but it does not establish that the dependency cannot modify that storage through a previously retained address or other alias.

Each candidate begins with a fresh assignment from the original context input. This assignment does not imply preservation of that value across subsequent dependency calls.

### Zero-result branch

When the first comparison returns zero, the routine prepares a second temporary by calling F-4bbba16cfe23 with the second temporary’s destination address and the first temporary’s address.

Unlike the first preparation call, this call’s return value is retained and treated as an address. That returned address becomes:

- The element-like operand used in backward comparisons.
- The source address used for final placement.
- The base for the final offset-8 field read.

**Only after the second preparation call returns**, the routine copies the first context slot into the second context slot.

This ordering is significant. The first comparison has already received the first slot’s address and could retain it in shared state. The intervening preparation dependency could then modify that slot. Therefore, the second slot receives the value present after the preparation call, not necessarily the value left by the first comparison.

If the comparison changes the first slot and the intervening preparation call preserves that changed value, the second slot receives the comparison’s value. Without that preservation condition, propagation of the comparison’s value is not established.

The routine then searches backward, beginning with the candidate as the current insertion position:

- It computes the preceding position, 12 bytes earlier.
- It calls F-e46aba7df1fa with the second context-slot address, the retained source address, and the preceding position.
- A zero result stops the search.
- A nonzero result causes relocation of the preceding element into the current insertion position.

Each relocation occurs in this order:

1. F-46f0b3cb316e receives the current insertion position as destination and the preceding position as source.
2. After that call returns, the routine reads the source’s four-byte field at offset 8.
3. It writes that value to destination offset 8.
4. The insertion position moves backward by 12 bytes.

The field read therefore observes the source after the movement call, including any mutation caused by that call. No pre-call field snapshot is preserved.

The second context slot is initialized once for this backward search. Later comparisons receive the same slot address, without a fresh context assignment between comparisons. Comparisons and intervening dependencies may affect its contents if they can access it.

After a zero comparison stops the search:

1. F-46f0b3cb316e receives the final insertion position and the retained source address.
2. The routine reads four bytes at retained source offset 8 after the movement returns.
3. It writes those bytes to insertion-position offset 8.
4. F-289ec4311c64 receives the second temporary’s original destination address.
5. The common cleanup call for the first temporary follows.

There is no explicit range-start guard in the backward search. Safe termination depends on dependency behavior.

The second temporary’s cleanup address is its original destination, not necessarily the address returned by its preparation call. The required relationship between those addresses is unknown.

### Nonzero-result branch

When the first comparison returns nonzero, the routine:

1. Calls F-2355fc619e7e with start, candidate, and candidate plus 12.
2. Calls F-46f0b3cb316e with start as destination and the first temporary’s address as source.
3. Reads the first temporary’s associated four-byte slot after the movement call and writes it to start offset 8.
4. Performs the common cleanup call on the first temporary.

F-2355fc619e7e appears consistent with making room at the beginning of the processed range, but no exact rotation, shift, preservation, or ownership semantics are established.

The explicit trailing-field write uses the temporary slot’s current contents. It is not guaranteed to restore the field’s value from before the preceding dependencies.

### Recursive procedure

For a byte difference of at least 180, the routine performs these operations in order:

1. Calls F-870c4fcf7295 with the total byte difference and 24.
2. Multiplies the returned value by 12 and adds it to start to obtain the split address.
3. Recursively processes start through split, passing the original context value.
4. After that invocation returns, recursively processes split through end with the same context value.
5. Calls F-870c4fcf7295 with the right subrange’s byte difference and 12; discards the result.
6. Calls F-870c4fcf7295 with the left subrange’s byte difference and 12.
7. Calls F-1374f1f9e190 with start, split, end, and the immediately preceding helper result.

The discarded-result call remains observable and may have side effects. It is not interchangeable with omission.

The combining dependency’s fourth argument is derived from the left subrange. It is not the original context input.

If F-870c4fcf7295 performs ordinary nonnegative integer division, the initial calculation produces an approximately halfway, element-aligned split. That interpretation is conditional. The supplied routine alone does not establish the helper’s arithmetic semantics or recursive progress.

## Side effects

### Memory and lifecycle

Direct writes include context-slot assignments and four-byte element-field copies. Dependencies may modify additional bytes, temporary storage, shared state, or other reachable memory.

Normal cleanup order is:

- Zero-result branch: second temporary, then first temporary.
- Nonzero-result branch: first temporary only.

These calls establish an observable lifecycle sequence, not an allocation or ownership guarantee. They do not prove construction, destruction, reference counting, transfer, or release semantics.

### Aliasing and retained addresses

A dependency receiving a context-slot address may retain it. Later dependencies may access that slot indirectly through shared state. Absence of an explicit pointer argument at a later call does not prove preservation.

This applies particularly to:

- The first preparation call between context initialization and the first comparison, if an address remains reachable from earlier activity.
- The second preparation call between the first comparison and the context-slot copy.
- Movement calls between backward comparisons, after the second slot’s address has been exposed.

Likewise, dependencies are not established to preserve source elements, temporary fields, globals, or state shared between recursive invocations.

### Errors and partial completion

There is no explicit status result, validation failure, rollback, or error-handling branch.

If a dependency fails to return normally, later operations and cleanup are not established to execute. Failure after relocation begins may leave the range partially modified.

Malformed boundaries can prevent equality-based forward termination. Persistent nonzero backward results can lead to access before start. Invalid partition results can cause invalid accesses, repeated subranges, or nontermination.

## Dependencies

| Opaque ID | Observable role | Unresolved contract |
|---|---|---|
| F-4bbba16cfe23 | Prepares temporary storage from a supplied source; the second call’s result is used as an address. | Access width, initialization, source mutation, returned-address relationship, ownership, shared-state effects. |
| F-e46aba7df1fa | Receives a context-slot address and two element-like addresses; zero versus nonzero controls relocation. | Relation implemented, context mutation or retention, element mutation, failure behavior. |
| F-46f0b3cb316e | Receives destination and source addresses before an explicit trailing-field copy. | Copy or move semantics, overlap support, access width, source preservation, ownership. |
| F-289ec4311c64 | Receives original temporary destination addresses after use. | Cleanup semantics, access width, release behavior, external effects. |
| F-2355fc619e7e | Receives start, candidate, and candidate plus 12. | Exact transformation, boundary interpretation, preservation and overlap requirements. |
| F-870c4fcf7295 | Supplies the partition multiplier and combining argument; another result is discarded. | Arithmetic, signedness, side effects, valid partition guarantees. |
| F-1374f1f9e190 | Receives three range boundaries and the left-derived helper result after both recursive calls. | Combining semantics, ordering assumptions, layout and ownership requirements. |

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests.

### Empty and one-element ranges

For equal endpoints with a zero difference, no dependencies are called and no elements are written.

For end exactly 12 bytes after start, the prospective candidate equals end. No temporary preparation, comparison, relocation, or cleanup occurs.

### Two elements with a nonzero first comparison

For a valid 24-byte range:

- The first temporary is prepared from the second element.
- A nonzero first comparison selects F-2355fc619e7e with start, start plus 12, and end.
- Movement into start follows.
- The temporary’s associated four-byte slot is read after movement and written to start offset 8.
- The first temporary is cleaned up.

Final values remain dependent on the callees.

### Two elements with zero comparisons

If both the initial comparison and first backward comparison return zero:

- The second temporary is prepared.
- The first context slot is copied only after that preparation returns.
- No preceding element is shifted.
- Final movement targets the original candidate position and uses the returned source address.
- The trailing field is read after movement.
- Cleanup addresses are the second temporary’s original destination, then the first temporary’s original destination.

### Context preservation and intervening mutation

Suppose the first comparison writes a changed context value and returns zero.

- If the second preparation call leaves that slot unchanged, the second slot receives the changed value.
- If the comparison retained the slot address and the second preparation call uses that alias to replace its contents, the second slot receives the replacement value.
- The routine performs no second-slot assignment before that intervening call returns.

Subsequent backward comparisons share the second slot, subject to further dependency effects. The next outer candidate begins by assigning the original input value to the first slot again; its preservation until comparison remains dependency-dependent.

### Threshold and recursive order

A conventional 168-byte range selects the short-range procedure.

A 180-byte range selects recursion. If the arithmetic helper performs ordinary integer division, the split is 84 bytes after start: seven elements on the left and eight on the right.

After both recursive calls return, the helper receives 96 and 12, then 84 and 12. The latter result becomes the combining call’s fourth argument. Side effects of earlier calls may still affect later dependency behavior.

### Unsupported backward behavior

If backward comparisons continue returning nonzero, no explicit start-boundary test stops relocation. Safe completion is not guaranteed.

## Uncertainties

The following remain **documented_only**:

- Comparison semantics and any sortedness, stability, or permutation guarantee.
- Dependency access widths and complete element layout.
- Initialization and preservation of temporary trailing fields.
- Relationships between temporary destinations and returned source addresses.
- Ownership, allocation, release, and source-preservation behavior.
- Context-slot preservation across dependencies, including access through retained aliases.
- Arithmetic-helper semantics, overflow behavior, and recursive progress.
- Dependency effects on globals and other shared storage.
- Abnormal-exit propagation and cleanup.
- Behavior for reversed, misaligned, oversized, or otherwise malformed ranges.

No stronger promise is made by assuming these unknowns resolve favorably.

## Evidence

This contract is based on the supplied static representation and opaque dependency mapping.

Established evidence includes call order, argument relationships, four-byte context transfers, the context copy after the second preparation call, 12-byte traversal, the 180-byte threshold, post-movement offset-8 reads, temporary cleanup order, and left-before-right recursion.

Sorting intent and ordinary-division partitioning are explicitly inferred or conditional. Dependency implementations were not supplied. No runtime tests have been performed.
