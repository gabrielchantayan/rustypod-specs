<!-- Trusted provenance: F-cb3a3b028def; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb3a3b028def

## Purpose

F-cb3a3b028def traverses a linked structure using an opaque comparison dependency, maintains a candidate handle, and conditionally accepts that candidate after two further dependency calls. It writes one 32-bit handle to caller-provided storage and returns a separate 64-bit packed result.

The observable algorithm is consistent with a bound-search followed by an equivalence check in a sentinel-based ordered tree. That interpretation is **inferred intent**, not an established container contract. The supplied evidence does not establish link ordering, comparator semantics, sentinel invariants, or the meaning of the payload returned by the accessor dependency.

In particular, the handle written to the caller is not necessarily the handle packed into the upper half of the return value.

## Interface

### Inputs

The function receives four arguments:

| Argument | Width and role |
|---|---|
| Output location | An address at which the function eventually writes one 32-bit value. Its address bits also initialize the lower half of the return value. |
| Structure address | A 32-bit address-like value used to locate a header field and an embedded comparison context. |
| Search operand | A 32-bit opaque value passed to the comparison dependency. Its interpretation is delegated to that dependency. |
| Fourth argument | A 32-bit value with no established effect on the result, calls, or caller-visible writes. |

The fourth argument is initially copied into temporary storage, but that storage is replaced before being passed to a dependency. It is not forwarded as an argument to any dependency.

### Address and data model

The supplied representation uses 32-bit integer fields for handles and address-bearing values. The return value is 64 bits.

All offsets below are **byte offsets**:

| Base | Offset | Established use |
|---|---:|---|
| Structure address | 16 | A 32-bit header handle |
| Structure address | 25 | Address passed as the comparison context |
| Header handle | 4 | Initial traversal handle |
| Traversal node | 8 | Next handle selected when the traversal comparison returns zero |
| Traversal node | 12 | Next handle selected when the traversal comparison returns nonzero |
| Traversal node | 16 | Address passed as the node-side comparison operand |

These observations describe only accessed fields and computed addresses. They do not establish complete object sizes, ownership, allocation types, or layouts of surrounding fields.

The comparison context and traversal payload are supplied by address. This function does not first load a 32-bit value from either location.

### Outputs

On normal completion, the function:

1. Writes exactly one 32-bit value through the output-location argument.
2. Returns a 64-bit value consisting of:
   - **Upper 32 bits:** the candidate temporary’s final bit pattern.
   - **Lower 32 bits:** the output-location address bits if no traversal iteration occurred, or zero if at least one iteration occurred.

The packing preserves the two 32-bit bit patterns. It is not a sign extension of the candidate and is not established as a success/failure encoding.

The upper return half can differ from the written output. A fallback writes a freshly loaded header handle without replacing the candidate temporary used for the upper return half.

## Behavior

### 1. Initial reads and candidate setup

The function first reads the header handle from the structure at offset 16. Through that handle, it reads the initial traversal handle at offset 4.

It then reads the structure’s header field at offset 16 again and uses this second value as the initial candidate.

These are distinct reads. The evidence does not justify replacing them with a single cached header read. Under stable memory they produce the same header handle, but the function itself does not establish stability or synchronization.

The output-location address is initially retained as the low-half return value.

### 2. Traversal

Traversal continues while the current traversal handle is nonzero.

At the start of every iteration, the retained low-half return value is set to zero. Thus, any traversal iteration is sufficient to make the returned lower 32 bits zero, regardless of subsequent candidate selection.

For each nonzero traversal handle, the function calls F-bcefa5957e76 with these arguments, in order:

1. The structure address plus 25.
2. The current traversal handle plus 16.
3. The search operand.

Only whether the returned 32-bit result equals zero affects this function’s control flow:

- **Zero result:** the function reads a 32-bit handle from the current node at offset 8. This newly read handle becomes both the next traversal handle and the candidate.
- **Nonzero result:** the function reads a 32-bit handle from the current node at offset 12. This newly read handle becomes the next traversal handle, while the candidate remains unchanged.

The selected link is read **after** the comparison call returns. Consequently, if that dependency changes the applicable node field, the subsequent link read observes the field at that later point.

The zero-result branch records the **selected link**, not the node on which the comparison was performed. If that selected link is zero, the candidate becomes zero and traversal ends. No local guard restores the previous candidate.

There is no iteration bound, cycle detection, or structural validation. A reachable cycle can prevent completion.

### 3. Post-traversal dependency check

After traversal ends, the function reads the structure’s header field at offset 16 again and stores that value in a separate 32-bit temporary.

It then calls F-2b931655bb54 with addresses of two distinct temporary slots:

1. The candidate slot.
2. The freshly captured header slot.

The call receives addresses, not merely copies of the two handle values. The visible evidence therefore does not establish that either temporary remains unchanged.

If F-2b931655bb54 returns nonzero, the function skips the accessor and final comparison and takes the fallback path.

If it returns zero, the function continues using the candidate slot’s value **after that call**.

### 4. Candidate acceptance check

On the zero-result path from F-2b931655bb54, the function:

1. Passes the current candidate value to F-0d2e87dfdbd4.
2. Receives a 32-bit opaque result.
3. Calls F-bcefa5957e76 with:
   - The structure address plus 25.
   - The search operand.
   - The accessor’s result.

The search operand occupies a different comparison position here than during traversal. During traversal it is the third argument; during this final comparison it is the second argument.

If this final comparison returns zero, the candidate slot is selected for the output write.

If it returns nonzero, the function takes the fallback path.

### 5. Fallback and final write

The fallback path reads the structure’s header field at offset 16 **at fallback time** and selects that newly read value for output.

This applies both when:

- F-2b931655bb54 returns nonzero; or
- The final comparison returns nonzero.

The fallback does not reuse the header snapshot passed by address to F-2b931655bb54. Dependency-induced changes to the structure’s header field can therefore affect the fallback output.

After either candidate acceptance or fallback selection, the function writes the selected 32-bit value to the caller’s output location.

It then returns the packed candidate and traversal-marker value described in the interface.

## Side effects

### Direct caller-visible writes

The only explicit write to caller-provided memory performed directly by this function is the final 32-bit output write.

The output location is not initialized on entry. Traversal and all applicable dependency calls occur before the output write. If execution fails or does not return before that point, this function provides no completed-output guarantee.

Output storage may alias readable structure fields insofar as the underlying environment permits such aliasing. The explicit output write occurs after the function’s own structural reads and dependency calls; there is no later explicit structure read.

### Dependency effects

Dependency calls may have additional effects not specified by the supplied evidence. This contract does not assume that they:

- Preserve global state.
- Preserve structure fields.
- Avoid allocation or deallocation.
- Leave the referenced temporary slots unchanged.
- Avoid retaining addresses passed to them.
- Return normally.

The function does not itself perform an explicit allocation or release operation. That does not establish allocation-free execution of the complete call chain.

The two temporary-slot addresses passed to F-2b931655bb54 are valid during that call. This function provides no supported post-return lifetime for those addresses. Whether that dependency retains them is unknown.

### Preconditions and failures

For normal completion, every actually executed memory access must be valid:

- The structure’s header field must be readable whenever accessed.
- The header used for the initial traversal read must support a readable 32-bit field at offset 4.
- Each visited node must support the selected 32-bit link read.
- The output location must support a 32-bit write.
- Dependency arguments must satisfy the dependencies’ own, unavailable contracts.

There is no explicit null-output check, invalid-handle check, error return, or recovery path. A zero initial traversal handle skips traversal, but a zero header handle is not specially protected before the initial header-relative read.

The evidence establishes no behavior for invalid addresses, arithmetic overflow in address formation, concurrent unsynchronized mutation, or dependency failures.

## Dependencies

### F-bcefa5957e76

Used as an opaque three-argument comparison operation returning a 32-bit result.

It is called once per visited traversal node and, conditionally, once more for candidate acceptance.

This function distinguishes only zero from nonzero. It does not distinguish positive from negative values. A conventional negative/zero/positive comparison contract must not be assumed.

### F-2b931655bb54

Called exactly once if traversal terminates normally.

Receives the addresses of the candidate and captured-header temporary slots. A zero result permits the accessor and final comparison; a nonzero result forces fallback.

Its apparent role could be a handle or iterator comparison, but that is **inferred intent**. Mutation of the referenced slots is not excluded by the available evidence.

### F-0d2e87dfdbd4

Called exactly once when F-2b931655bb54 returns zero, and not called otherwise.

Receives the candidate value as it exists after the preceding dependency call. Its 32-bit result becomes the final comparison’s third argument.

No payload layout, ownership transfer, lifetime guarantee, or validity rule for this result is established.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, they assume valid memory, normally returning dependencies, stable structure fields, and no dependency mutation of the temporary slots.

### Empty traversal with fallback

Suppose the initial header-relative traversal handle is zero, and F-2b931655bb54 returns nonzero.

Expected behavior:

- No traversal comparison occurs.
- The accessor and final comparison are skipped.
- The current structure header is written to the output.
- The return upper half contains the initial candidate header.
- The return lower half contains the output-location address bits.

### One zero-result traversal branch ending at zero

Suppose the initial traversal handle identifies node A, the traversal comparison returns zero, and A’s offset-8 link is zero.

Expected behavior:

- The candidate becomes zero.
- Traversal ends after one iteration.
- The lower return half is zero.
- F-2b931655bb54 receives a candidate slot containing zero.

Whether the accessor can subsequently accept zero is unknown. This function does not reject that value itself.

### Traversal takes only nonzero-result branches

Suppose all traversal comparisons return nonzero and offset-12 links eventually reach zero.

Expected behavior:

- The candidate remains the initially captured header, unless later changed through the candidate-slot address.
- The lower return half is zero because traversal occurred.
- Post-traversal dependency processing still runs.

### Candidate accepted

Suppose the candidate is handle K, F-2b931655bb54 returns zero, the accessor returns value V, and the final comparison of the search operand against V returns zero.

Expected behavior:

- K is written to the output, assuming the candidate slot is not subsequently changed.
- The return upper half contains K.
- The return lower half depends only on whether traversal occurred.

### Candidate rejected after accessor

Suppose the candidate is K and the final comparison returns nonzero.

Expected behavior:

- The function reloads the structure header and writes that handle to the output.
- The return upper half still contains K.
- A mismatch between the written output and the return upper half is expected behavior.

### Header changed by a dependency

Suppose a post-traversal dependency changes the structure header from H to J, and execution takes fallback.

Expected behavior:

- The fallback output is J.
- The earlier captured header value does not control fallback.
- The upper return half remains the candidate slot’s final value, which need not be J.

### Candidate slot changed by F-2b931655bb54

Suppose that dependency replaces candidate K with M and returns zero.

Expected behavior:

- The accessor receives M.
- If final acceptance succeeds without further candidate-slot changes, M is written.
- The return upper half contains the candidate slot’s final value, rather than necessarily K.

## Uncertainties

The following remain **documented_only**:

- The abstract container type and ordering invariant.
- Whether either link should be described as left, right, predecessor, or successor.
- Whether the header is a sentinel, end marker, or ordinary node.
- The comparison dependency’s semantic relation.
- Whether candidate acceptance denotes equality or membership.
- The accessor result’s representation and lifetime.
- Dependency mutation, allocation, release, and address-retention behavior.
- Required alignment and the precise target memory model.
- Overflow behavior for address calculations outside valid object ranges.
- Thread-safety and supported concurrent mutation.
- The intended external meaning of the packed return value.

The supplied representation supports 32-bit field and 64-bit return descriptions. It does not justify extrapolating a different-width ABI or treating address additions as portable, checked arithmetic.

## Evidence

**Established static evidence:** explicit field offsets, 32-bit reads and writes, dependency argument order, zero/nonzero branching, candidate updates, repeated header reads, the final output store, and concatenation into a 64-bit return value.

**Inferred intent:** an ordered linked-structure search, a sentinel-like header, a bound candidate, and a final equivalence check. These interpretations are not prerequisites for the literal behavioral contract.

**Unknown:** dependency implementations, complete object layouts, ownership rules, concurrency guarantees, and the abstract meaning of the returned halves.

No runtime tests have been performed. The acceptance examples state consequences of the supplied static evidence under their explicit conditions; they are not runtime verification results.
