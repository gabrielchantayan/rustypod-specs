<!-- Trusted provenance: F-eb6fb41bdc77; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb6fb41bdc77

## Purpose

F-eb6fb41bdc77 coordinates a range-like operation over a container-like object. It initially stores the container’s anchor reference in caller-provided output storage, then selects either:

- A bulk operation followed by an explicit reset of the anchor’s links and the container’s count-like field; or
- A loop that captures an entry reference, advances the supplied cursor, and invokes a per-entry operation.

The structure suggests removal of a range from a linked, possibly tree-based container. That interpretation is **inferred intent**, not an established semantic guarantee. The supplied body establishes field accesses, dependency calls, control flow, and stores; it does not establish what the dependencies compare, advance, remove, destroy, or release.

The result is a 64-bit value assembled from two 32-bit words. It must not be treated as a conventional success code or a single returned pointer without an external calling contract.

## Interface

### Inputs

The visible interface has four arguments:

1. **Output location**  
   An address at which this function writes one 32-bit word. Its initial contents are not explicitly read.

2. **Container reference**  
   A 32-bit address-bearing value identifying an object with accessible fields at byte offsets `0x10` and `0x14`.

3. **Mutable cursor location**  
   An address passed to dependencies. The function itself reads its first 32-bit word when entering a loop iteration.

4. **Boundary argument**  
   A 32-bit value passed directly to F-227cdabeb472. Its apparent use is as a comparison operand, but the supplied body does not establish whether it is an address, handle, or other encoded value.

The representation shown uses 32-bit words and address-bearing values. No 64-bit address arithmetic is established.

### Referenced layouts

The function directly uses the following container fields:

| Byte offset | Width | Established use |
|---|---:|---|
| `0x10` | 32 bits | Loaded as an anchor reference |
| `0x14` | 32 bits | Tested for nonzero on a conditional path; cleared by the bulk-reset path |

At the object reached through the anchor reference:

| Byte offset | Width | Established use |
|---|---:|---|
| `0x04` | 32 bits | Passed to the bulk dependency, then cleared on the bulk-reset path |
| `0x08` | 32 bits | Initially copied into temporary storage; later reset to an anchor reference |
| `0x0C` | 32 bits | Reset to an anchor reference on the bulk-reset path |

Calling these fields a root, leftmost link, rightmost link, or size is an interpretation of their arrangement, not a proven property.

### Output and return value

The output location receives:

- The initially observed anchor reference, unconditionally.
- A newly loaded anchor reference immediately before the bulk-path return; or
- The current 32-bit operation-result temporary after each per-entry dependency call.

There is no per-iteration output write when the loop executes zero iterations.

The returned 64-bit value has two independently significant halves:

- **Bulk path:** upper 32 bits are the anchor value loaded near the end of the path; lower 32 bits are the current entry/output-reference temporary.
- **Loop path, including zero iterations:** upper 32 bits are the current operation-result temporary; lower 32 bits are the current entry/output-reference temporary.

Before any loop iteration, those temporaries respectively contain the container reference and the output-location address. Thus, with no loop iterations and no dependency-induced mutation of those temporaries, the returned upper half is the container reference and the lower half is the output-location address.

Concatenation preserves the two 32-bit patterns. It is not addition or sign extension.

### Preconditions

For defined execution, all directly accessed addresses must designate suitably accessible storage. In particular:

- The output location must support a 32-bit write.
- The container reference must support the specified field reads and, if selected, the count-like field write.
- Loaded anchor references must support the field accesses reached on the selected path.
- A loop iteration requires a readable 32-bit word at the cursor location.
- Each dependency’s own preconditions must be satisfied.

The body performs no explicit null, bounds, lifetime, or alignment validation. It establishes no non-aliasing guarantee among arguments and referenced objects.

## Behavior

### 1. Initial output and anchor-derived comparison operand

The first visible operation loads the container field at `0x10` and stores that value through the output location.

The function then reads the container’s anchor field again, loads the 32-bit word at that anchor’s `0x08` offset, and saves the word in a temporary.

This ordering matters if the output location aliases the container or related storage. The second anchor read occurs after the initial output write; the function does not necessarily use one immutable anchor snapshot.

Two further temporaries initially hold:

- The output-location address.
- The container reference.

The function calls F-227cdabeb472 with:

1. The mutable cursor location.
2. The address of the temporary containing the anchor-derived `0x08` word.

The temporary is passed by address. Whether the dependency reads or changes it is not established.

### 2. Conditional bulk-path eligibility

If the first comparison result is zero, the bulk-path eligibility checks stop and execution proceeds to the loop condition.

If that result is nonzero:

1. The function reloads the container’s anchor reference into another temporary.
2. It calls F-227cdabeb472 with the boundary argument and the address of that temporary.
3. If this second result is nonzero, it reads the container field at `0x14`.
4. The bulk path is selected only if the second result and the field value are both nonzero.

The first two comparison results use **nonzero** as their enabling condition. They are not required to equal one.

If the second comparison returns zero, the field at `0x14` is not explicitly read by this eligibility sequence.

Neither the initial anchor-derived word nor the anchor reference used for the second comparison is reused as an authoritative anchor snapshot for the subsequent bulk operation. Later accesses reload the container field.

### 3. Bulk operation and explicit reset

On the bulk path, the function:

1. Reloads the container’s anchor reference.
2. Reads the 32-bit field at anchor offset `0x04`.
3. Calls F-57aa8dd68d76 with the container reference and that field value.
4. Performs the explicit reset described below.
5. Stores a final anchor value through the output location.
6. Returns without entering the per-entry loop.

The reset consists of these writes in this order:

1. Anchor field `0x08` receives a value loaded from the container’s anchor field.
2. Anchor field `0x04` receives zero.
3. Anchor field `0x0C` receives a value loaded from the container’s anchor field.
4. Container field `0x14` receives zero.

The anchor reference used to locate each destination is read from the container again. F-57aa8dd68d76 is not known to preserve the anchor field, so the reset must not be specified as necessarily targeting the anchor observed before that call.

Likewise, the self-reference-looking assignments must not be strengthened into an invariant under arbitrary aliasing. They explicitly use container-field reads to obtain both destination references and stored reference values.

After clearing the count-like field, the function reloads the anchor reference, writes it to the output location, and uses that loaded value as the upper return half.

The lower return half comes from the entry/output-reference temporary. The function does not directly reassign that temporary on the bulk path. Its initialized value is the output-location address; stronger preservation claims require a dependency and aliasing contract.

The body does not establish that the bulk dependency releases nodes or payloads, nor that the reset represents a semantically empty container. Those are inferred interpretations.

### 4. Loop condition

When the bulk path is not taken, the function repeatedly calls F-227cdabeb472 with the mutable cursor location and the boundary argument.

The loop terminates **only when the result equals exactly one**.

Consequently:

- A result of zero enters another iteration.
- A result of two also enters another iteration.
- Any other representable result unequal to one enters another iteration.

This differs materially from the earlier eligibility checks, which accept any nonzero result.

The condition is evaluated before the first iteration. It is evaluated again after each completed iteration. There is no visible iteration limit or independent progress check.

### 5. Per-entry sequence

For each iteration whose condition result is not one, the function performs the following operations in order:

1. Reads the first 32-bit word at the mutable cursor location and saves it as the selected entry reference.
2. Calls F-315d952a53b9 with the mutable cursor location.
3. Copies the previously captured entry reference into the entry/output-reference temporary.
4. Calls F-1b2c8ba4a1d5 with:
   - The address of the operation-result temporary.
   - The container reference.
   - The address of the entry/output-reference temporary.
5. Reads the operation-result temporary and writes that word through the output location.
6. Returns to the loop condition.

The entry reference is captured **before** F-315d952a53b9 runs. The value supplied through the third argument to F-1b2c8ba4a1d5 therefore originates from the pre-call cursor word, not from a fresh cursor read after advancement.

Both temporaries passed to F-1b2c8ba4a1d5 are addressable and may be changed by that dependency. The subsequent output write uses the operation-result temporary after the call. The final return also uses the then-current values of both temporaries.

No explicit decrement of the count-like field occurs in the loop body. Any such maintenance belongs to dependency behavior, if it occurs at all.

### 6. Completion and failures

After a loop-condition result of exactly one, the function returns the two current temporary words as described above. It performs no additional explicit output write at termination.

No explicit failure branch, error object, rollback, or cleanup-on-failure mechanism is visible. All loop-condition results except one mean “continue,” not “report error.”

If a dependency does not return normally, or if a memory access is invalid, the supplied body establishes no recovery behavior. Earlier writes may already have occurred.

## Side effects

The function’s explicit writes are limited to:

- Caller-provided output storage.
- Temporary storage passed by address to dependencies.
- Three anchor fields and the container’s `0x14` field on the bulk-reset path.

Dependencies may have additional effects. Their effects on globals, container state, cursor state, allocation state, and object lifetimes are unknown here.

The function itself contains no visible allocation or deallocation primitive. That does **not** establish allocation-free execution or preservation of referenced objects: the bulk and per-entry dependencies may allocate, destroy, or release memory.

No ownership transfer can be established merely from passing a reference by value or by address. In particular, the captured entry reference must not be assumed valid after F-1b2c8ba4a1d5 returns.

Aliasing is consequential. For example, an output write that overlaps cursor storage can affect the next comparison or entry capture. The contract does not impose independence that the visible interface does not provide.

## Dependencies

| Opaque ID | Invocation and established role | Not established |
|---|---|---|
| F-227cdabeb472 | Supplies the results controlling both bulk eligibility and loop continuation; receives different operand combinations across those uses | Equality semantics, Boolean normalization, operand preservation, global-state preservation |
| F-57aa8dd68d76 | Called once on the bulk path with the container reference and a freshly read anchor `0x04` word | Traversal, destruction, deallocation, anchor preservation |
| F-315d952a53b9 | Called once per loop iteration after entry capture and before the per-entry operation | Direction or extent of advancement, guaranteed progress, reference preservation |
| F-1b2c8ba4a1d5 | Called once per iteration with two writable temporary addresses and the container reference | Removal semantics, count maintenance, lifetime effects, exact values written to the temporaries |

No dependency contract is supplied. Their effects must remain explicit unknowns rather than being filled in from inferred container conventions.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, assume valid independent storage and no dependency effects beyond those described in each example.

### Immediate loop termination

Given that the first comparison returns zero and the first loop-condition call returns one:

- The output location receives the initial anchor reference.
- Neither bulk nor per-entry dependency runs.
- The returned upper 32 bits equal the container reference.
- The returned lower 32 bits equal the output-location address.

This example distinguishes the output word from the two-word return value.

### Non-one comparison result continues

Given that bulk eligibility fails and loop-condition results are two, then one:

- Exactly one iteration occurs.
- The cursor word is captured before F-315d952a53b9 is called.
- F-1b2c8ba4a1d5 receives that captured reference through a temporary.
- The output receives the operation-result temporary’s post-call value.

A nonzero result of two must not terminate this loop.

### Bulk path with non-unit truth values

Given that the first eligibility comparison returns two, the second returns a nonzero value, and the count-like field is nonzero:

- The bulk path is selected.
- F-57aa8dd68d76 is called once.
- The explicit reset writes occur in the stated order.
- No per-entry advancement or operation runs.

The eligibility checks must not require results equal to one.

### Zero count-like field

Given that both eligibility comparisons are nonzero but the count-like field is zero:

- The bulk dependency is not called.
- Execution proceeds to the loop condition.
- The subsequent loop behavior depends on that condition, not on the zero field alone.

### Anchor replacement by the bulk dependency

Given an initial anchor A, and F-57aa8dd68d76 changes the container’s anchor field to valid anchor B:

- Subsequent reset accesses use fresh reads of the container’s anchor field.
- If that field remains B and no aliasing changes it, B receives the link resets.
- The final output and upper return half are B.

Continuing to use a cached A would not match the visible behavior.

### Writable per-entry temporaries

Given one iteration in which F-1b2c8ba4a1d5 sets the operation-result temporary to X and the entry/output-reference temporary to Y, followed by immediate termination:

- The iteration writes X through the output location.
- The return has X in its upper 32 bits and Y in its lower 32 bits.

The initially captured entry reference does not necessarily remain the lower return half.

## Uncertainties

The following are **documented_only**:

- The semantic container type and meaning of its fields.
- Whether F-227cdabeb472 implements equality or another relation.
- Whether its result domain is restricted to zero and one.
- The representation and validity requirements of the boundary argument.
- Whether the cursor operation guarantees progress.
- Which objects either operation destroys or releases.
- Whether dependencies preserve globals, anchor references, or other caller-accessible memory.
- The external interpretation of the packed 64-bit return.
- Required aliasing restrictions, alignment, concurrency discipline, and failure behavior beyond the visible accesses.

The strongest inferred intent is range removal with an optimized whole-container case. That inference does not override the exact comparison rules, repeated anchor reads, or packed-return behavior.

## Evidence

The supplied static body establishes the field offsets, 32-bit accesses, dependency-call order, conditional branches, explicit writes, and construction of the 64-bit return.

Dependency implementations, external type definitions, ownership contracts, and caller expectations were not supplied. No runtime tests have been performed. The acceptance examples describe consequences of the visible body under stated conditions; they are not execution results.
