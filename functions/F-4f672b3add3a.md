<!-- Trusted provenance: F-4f672b3add3a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4f672b3add3a

## Purpose

F-4f672b3add3a performs three calls to F-654a426acf73 and four explicit 32-bit writes into caller-supplied destination storage, then returns the original destination pointer.

The first two calls use selector zero and a selector obtained by subtracting one from a source field. The third uses the supplied index, optionally transformed according to the mode. A separate destination write records classification two for modes zero and one, or one for every other mode.

This is a contract for call order, selector calculation, and explicit writes—not an unconditional four-word final-result layout. Later dependency calls can change storage written earlier through aliases or other reachable state. Only destination offset 0 receives its specified result after the last dependency call. Final-content guarantees for the other three words require an explicit dependency noninterference assumption.

The source field may be count-like, and the selector transformations may distinguish paired entries. Those interpretations are inferred intent, not established semantics.

## Interface

### Inputs and return value

| Input | Supplied representation | Role |
|---|---|---|
| Destination | Pointer to 32-bit words | Identifies the storage for four explicit writes. |
| Source address | Address-bearing 32-bit integer | Used to locate a field and form the first argument of each dependency call. |
| Mode | 32-bit integer | Chooses the final selector transformation and classification. |
| Index | 32-bit integer | Supplies or determines the final selector. |

On normal completion, the return value is the original destination pointer. The function does not return a newly allocated destination or substitute a dependency result for that pointer.

The supplied representation establishes the word widths used below. It does not independently establish the original declared types, complete calling convention, or all properties of the target address model.

### Destination writes

Offsets are measured in bytes from the destination pointer.

| Offset | Width | Value explicitly written | Position in write order | Guarantee at normal return |
|---|---:|---|---:|---|
| 0 | 32 bits | Third dependency result | Fourth | The last explicit write assigns this result, after all dependency calls. |
| 4 | 32 bits | First dependency result | First | May subsequently be changed by the second or third dependency call. |
| 8 | 32 bits | Second dependency result | Second | May subsequently be changed by the third dependency call. |
| 12 | 32 bits | Classification | Third | May subsequently be changed by the third dependency call. |

These guarantees describe sequential execution without independent concurrent modification. They do not establish atomicity or synchronization.

In particular, normal return does not establish that offsets 4, 8, and 12 still contain the values explicitly written there. Their final contents can depend on dependency side effects.

### Source-relative locations

Two source-relative addresses occur:

- At source offset 4, the function explicitly reads one 32-bit integer field.
- At source offset 16, the function forms an address and passes it as the first argument of every dependency call.

Passing the address at offset 16 is not itself an explicit read of memory at that address by this function. Any access through that argument belongs to the dependency’s behavior.

### Preconditions for ordinary completion

Ordinary completion requires that:

- The destination permit the four represented 32-bit writes when they occur.
- The source field at offset 4 permit the represented read when it occurs.
- Source-relative address calculations be valid in the execution environment.
- Selector calculations have defined behavior for the supplied operands.
- Each dependency invocation satisfy its own preconditions and return normally.

Required alignment, the accessible extent of the object supplied to the dependency, and permission for source/destination overlap are **documented_only** unknowns. The function performs no explicit checks that establish these conditions.

## Behavior

### Ordered calls, reads, and writes

For an execution in which the represented operations succeed and all calls return normally, the established sequence is:

1. Call F-654a426acf73 with the source-relative address at offset 16 and selector zero.
2. Write the first returned 32-bit value to destination offset 4.
3. Read the 32-bit source field at offset 4 and subtract one to obtain the second selector.
4. Call F-654a426acf73 with the source-relative address at offset 16 and that selector.
5. Write the second returned 32-bit value to destination offset 8.
6. Determine the final selector and classification from the mode and index.
7. Write the classification to destination offset 12.
8. Call F-654a426acf73 with the source-relative address at offset 16 and the final selector.
9. Write the third returned 32-bit value to destination offset 0.
10. Return the original destination pointer.

The three calls remain distinct even when their selectors are equal. No earlier result is reused instead of making a later call.

All three calls receive the same source-relative address value, provided the represented address calculation is valid. This does not imply that the addressed object, destination storage, or other reachable state remains unchanged between calls.

### Observation of the source field

The source-field read occurs after both the first dependency call and the write to destination offset 4. It is not an entry-time snapshot.

Consequently:

- A field change made by the first call can affect the second selector.
- The first destination write can affect the second selector if it overlaps the source field and that overlap is valid.
- The selector is based on the value actually read at that point, not a presumed original count.

No second explicit read of that field is shown. Mutations after the read do not retroactively change the selector already supplied to the second call.

### Mode-dependent behavior

| Mode | Final selector | Classification explicitly written |
|---|---|---:|
| Zero | Index shifted left by one bit | 2 |
| One | Index multiplied by two, then increased by one | 2 |
| Any other value | Index without transformation | 1 |

The mode-one expression and the mode-zero shift have the same mathematical doubling effect only where their respective arithmetic is defined. Their exceptional-operand behavior must not be assumed identical.

Every mode other than zero and one follows the unchanged-index case. Under the supplied signed-integer interpretation, this includes negative modes and positive modes greater than one. The function does not reject those modes or treat every nonzero mode as mode one.

Classification is a value explicitly written before the third call. It is not necessarily the value remaining at destination offset 12 when the function returns.

### Widths and arithmetic boundaries

Dependency results and destination writes are 32 bits wide. The source field, mode, index, and selectors are represented as 32-bit integers.

The relevant arithmetic includes:

- Forming source-relative addresses at offsets 4 and 16.
- Subtracting one from the source field.
- Shifting the index left by one for mode zero.
- Multiplying the index by two and adding one for mode one.

The supplied representation does not justify an unconditional wraparound contract. Address overflow, signed subtraction overflow, and invalid signed shifts must not be silently assigned modular behavior.

Under a conventional signed 32-bit interpretation, conservative arithmetic ranges are:

| Calculation | Range with straightforward representable arithmetic |
|---|---|
| Source field minus one | Source field from −2,147,483,647 through 2,147,483,647 |
| Mode-zero left shift | Index from 0 through 1,073,741,823 |
| Mode-one multiplication and increment | Index from −1,073,741,824 through 1,073,741,823 |
| Unchanged index for other modes | No additional selector arithmetic is performed |

These ranges concern arithmetic only. They do not establish which selectors F-654a426acf73 accepts. A nonnegative selector is not proven in bounds, and a negative selector is not proven invalid.

For a source field of zero, subtraction yields −1 under ordinary signed arithmetic. There is no explicit empty-source branch or suppression of the second call.

### Validation and errors

No explicit validation is shown for null addresses, storage size, source-field positivity, selector bounds, overlap, or dependency results.

Each normally returned dependency result is written as an opaque 32-bit value. There is no shown status interpretation, sentinel check, conversion, or alternative error return.

If a memory access or dependency call prevents normal continuation, later operations and the normal return are not reached. The function contains no shown rollback or recovery path. The actual fault, exception, or nonlocal-transfer mechanisms remain unknown.

## Side effects

### Explicit memory effects

The function directly performs one source-field read and four destination writes. This establishes which values it writes and when, not preservation of every value until return.

At the relevant operation boundaries, absent independent concurrent modification:

- Immediately before the second call, destination offset 4 has received the first result, and no intervening dependency call has occurred since that write.
- Immediately before the third call, destination offset 8 has received the second result and offset 12 has received the classification. Offset 4 was written earlier, but the second call may already have changed it.
- After the third call, offsets 4, 8, and 12 may differ from their earlier explicitly written values.
- The final explicit write replaces destination offset 0 with the third result, including when the third call itself changed that word.

Thus, a third call that changes the classification word is not followed by a classification-restoration write. Likewise, a second call that changes destination offset 4 is not followed by another explicit write to that offset.

### Dependency effects and ownership

Dependency calls may cause additional reads, writes, allocations, releases, or other externally observable effects. Their scope is unknown. Nothing establishes that the dependency preserves globals, leaves the source object unchanged, or cannot reach the destination.

No direct allocation, release, or ownership-transfer operation is shown in this function. The destination is caller-supplied storage. Writing a dependency result does not itself establish reference acquisition, ownership transfer, or a release obligation. The results are opaque 32-bit values, not established pointers.

### Partial completion and concurrency

Writes are not presented as a transaction. If execution stops during a later call, earlier explicit writes have occurred, but the call that interrupts execution may itself have modified them.

There is no shown rollback, locking, atomic publication, or cross-thread memory-order guarantee. The ordered behavior above describes the represented sequential operations, not a concurrent observer’s guaranteed view.

## Dependencies

### F-654a426acf73

There are three calls on the normal path. Each receives:

1. The source-relative address at offset 16.
2. A 32-bit selector.

The selectors, in order, are zero; the observed source field minus one; and the mode-dependent selector.

Each normally returned 32-bit value is explicitly written to its assigned destination offset. Equality of arguments does not establish equality of results.

The following dependency properties are **documented_only** unknowns:

- Meaning and layout of the object addressed by the first argument.
- Selector domain and treatment of out-of-range or negative selectors.
- Meaning of returned values and any error representation.
- Whether calls mutate the source, destination, globals, or other reachable state.
- Whether calls acquire, allocate, release, or transfer resources.
- Whether calls can fail to return normally.
- Whether repeated calls with equal arguments return equal values.

Any stronger final-layout claim must state noninterference explicitly. A sufficient assumption is that dependency calls do not modify destination words after this function has written their intended values, and that no independent actor modifies the destination.

## Acceptance examples

These are conditional acceptance criteria, not executed tests. Unless stated otherwise, assume valid storage and address calculations, defined selector arithmetic, no independent concurrent modification, and normally returning dependency calls.

### Mode zero with noninterference

Suppose the field read yields 6, the mode is zero, and the index is 3. The selectors must be zero, 5, and 6, in that order. The explicit classification write must be 2.

If the dependency does not alter the destination, the returned destination contains the third result at offset 0, the first at offset 4, the second at offset 8, and 2 at offset 12. The returned pointer equals the supplied destination pointer.

Without that noninterference assumption, the same calls and explicit writes are required, but only the final assignment at offset 0 establishes its stated return-time content.

### Mode one and another mode

With an observed field of 6 and index 3:

- Mode one requires selectors zero, 5, and 7, with an explicit classification write of 2.
- Mode −1 requires selectors zero, 5, and 3, with an explicit classification write of 1.

Neither case guarantees that the classification remains unchanged through the third call.

### Field changed by the first call

Suppose the source field initially holds 8, but the first call changes it to 4. If the subsequent destination write does not change that field and nothing else intervenes, the second selector must be 3, not 7.

### Overlap affecting the field read

Suppose destination offset 4 and source offset 4 identify the same valid 32-bit storage. If the first result is 9, the explicit destination write places 9 in the field before its read. The second selector must therefore be 8.

This example assumes the overlap is permitted by the execution environment; it does not establish general overlap support.

### Second call changes an earlier result

Suppose the first result is 10 and the second call changes destination offset 4 to 99 before returning 20.

Before the third call, offset 4 need not contain 10: it may contain 99. Offset 8 has received 20, and offset 12 has received the selected classification. The function does not restore 10 at offset 4.

### Third call changes earlier words

Suppose the third call writes 70, 80, and 90 to destination offsets 4, 8, and 12, respectively, and then returns 30.

If nothing else modifies those words, normal return leaves 30 at offset 0 and the dependency-written values 70, 80, and 90 at the other offsets. This satisfies the contract despite differing from the earlier explicit writes and classification.

### Zero field and repeated selectors

An observed field of zero requires a second selector of −1 under ordinary signed arithmetic. The function does not skip that call; its outcome depends on the dependency.

An observed field of 1, mode zero, and index zero require three calls with selector zero. The results may differ, and earlier destination writes may be changed by later calls.

### Third call does not return normally

If the first two calls return but the third does not, the first result, second result, and classification have each been explicitly written before entry to that call. This does not establish that they remain present during or after the interrupted call. The final explicit write at offset 0 and the ordinary return are not reached.

## Uncertainties

**Established static evidence:** three dependency calls; one explicit source-field read after the first destination write; the selector branches; classification values; four ordered destination writes; and return of the original destination pointer.

**Inferred intent:** the source field may represent a count, its decremented value may select a last entry, and transformed indices may select members of pairs. None is required by this contract.

**Documented_only unknowns:**

- Original types, calling convention, and complete address model.
- Arithmetic behavior outside the conservative ranges described above.
- Byte order, alignment, and source-object extent.
- Dependency semantics, side effects, resource ownership, and failure behavior.
- Validity of overlapping storage.
- Final contents of offsets 4, 8, and 12 without noninterference assumptions.
- Concurrency, atomicity, and synchronization guarantees.

No runtime tests have been performed.

## Evidence

This contract derives from the supplied static representation of F-4f672b3add3a, using F-654a426acf73 as the opaque dependency identity.

The represented sequence establishes the calls, source-relative offsets, arithmetic branches, explicit writes, and pointer return. It also places dependency calls between earlier destination writes and normal return. Because the dependency implementation is unavailable, preservation of those earlier writes cannot be established.

No dependency implementation, callers, surrounding data definitions, or runtime observations were supplied. Acceptance examples state conditional consequences of the represented behavior, not observed execution.
