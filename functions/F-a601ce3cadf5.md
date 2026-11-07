<!-- Trusted provenance: F-a601ce3cadf5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a601ce3cadf5

## Purpose

F-a601ce3cadf5 conditionally performs either a bulk operation on a container-like object or a sequence of position-by-position operations. It writes a 32-bit result through its first argument and returns a separate 64-bit packed value.

**Inferred intent:** the surrounding structure resembles a container with a header node, an element count, position comparison, position advancement, and element removal. The bulk branch resembles clearing the container when the requested range spans all elements. These interpretations are not established dependency contracts.

**Established behavior:** branch selection depends on calls to F-ca7a1f20809e. The first two calls treat any nonzero result as affirmative; the repeated loop terminates only on a result of exactly one. The bulk branch performs explicit structural writes after calling F-8907a0bdd6c2. The loop delegates its structural operation to F-aab94c7c2aad.

## Interface

### Arguments

The function has four arguments:

1. **Output address.** An address at which the function writes a 32-bit word. It is written before any dependency call and may be written again.
2. **Container address.** A 32-bit address representation used to access fields at byte offsets 16 and 20.
3. **Current-position address.** An address of 32-bit storage. It is passed to F-ca7a1f20809e and F-017ac946138e. On each loop iteration, the function also reads its first 32-bit word.
4. **Endpoint value.** A 32-bit value passed unchanged to F-ca7a1f20809e. Its representation and whether it is itself an address are not established.

All visible object fields and scratch values relevant to this function are 32 bits wide. The returned value is 64 bits wide.

### Observed layouts

The container-like object supplies these fields:

| Byte offset | Width | Observed use |
|---|---:|---|
| 16 | 32 bits | Address of a header-like object; read repeatedly |
| 20 | 32 bits | Nonzero gate for the bulk branch; set to zero by that branch |

The header-like object supplies these fields:

| Byte offset | Width | Observed use |
|---|---:|---|
| 4 | 32 bits | Supplied to F-8907a0bdd6c2; subsequently set to zero on the bulk branch |
| 8 | 32 bits | Copied into scratch storage for the first comparison; subsequently set to the current header address |
| 12 | 32 bits | Set to the current header address on the bulk branch |

No meaning is established for other fields. The field at container offset 20 is used only for a zero/nonzero test here; the function does not validate it as a nonnegative count.

### Return representation

The return value concatenates two 32-bit words:

- The upper 32 bits contain a branch-dependent result word.
- The lower 32 bits contain a branch-dependent scratch word.

This is a bit-level description, not a claim about a public structure-return convention or memory byte order.

The return value is not generally identical to the word written through the output address. The lower half is especially significant: on a path with no loop iterations, it contains the output address itself, represented as a 32-bit word.

### Preconditions

For ordinary completion, every address actually dereferenced must designate accessible storage of the required width. This includes the initial output destination, container fields, the initial header field at offset 8, and any fields reached on the selected branch.

The dependencies impose additional unknown preconditions.

The function performs no visible null checks, bounds checks, layout checks, or alias checks. The supplied evidence does not establish alignment requirements, address-overflow behavior, or a recoverable response to an invalid access.

## Behavior

### Initial observations and output

The function first reads the 32-bit header address from container offset 16 and writes that word through the output address.

It then reads container offset 16 again and copies the 32-bit word at offset 8 of that header into temporary storage.

These operations occur before the first dependency call. In particular:

- The first output write is observable even if no loop iteration occurs.
- The initial header-field read is not conditional on either comparison succeeding.
- The second header lookup is a new read, not a guaranteed reuse of the value written to the output.
- If the output address aliases the container or header, its first write can affect later reads.

The function retains two additional scratch words: one initially holding the output address and another initially holding the container address. These words later supply the packed return unless overwritten along the selected path.

### Initial comparison

The function calls F-ca7a1f20809e with:

- The current-position address.
- The address of the temporary 32-bit word copied from header offset 8.

A zero result bypasses consideration of the bulk branch and proceeds to the loop comparison.

Any nonzero result triggers a second comparison. The function freshly reads container offset 16 into a separate 32-bit temporary and calls F-ca7a1f20809e with:

- The endpoint value.
- The address of that temporary.

The dependency receives writable temporary storage in both comparisons. Any alteration of those temporaries is not directly used later as the source of a structural write. This does not imply that the dependency has no other effects.

If the second comparison returns zero, execution proceeds to the loop comparison without reading container offset 20 for the bulk-branch gate.

If the second comparison returns nonzero, the function reads container offset 20. The bulk branch is selected only when that field is also nonzero.

Consequently, the bulk branch requires all three conditions:

- First comparison result is nonzero.
- Second comparison result is nonzero.
- The subsequently observed field at container offset 20 is nonzero.

A negative 32-bit field value satisfies the last condition just as any other nonzero bit pattern does.

### Bulk branch

The function freshly reads the header address from container offset 16, reads the word at header offset 4, and calls F-8907a0bdd6c2 with the container address and that word.

The dependency’s return value is not used.

After the call returns, the function performs these writes in order:

1. Write the current header address into offset 8 of the current header.
2. Write zero into offset 4 of the current header.
3. Write the current header address into offset 12 of the current header.
4. Write zero into container offset 20.

The container’s header field is read again for these post-call operations. The branch does not retain the pre-call header as an authoritative target. A change made by F-8907a0bdd6c2 to container offset 16 can therefore redirect the subsequent structural writes.

After the resets, the function reads container offset 16 again, writes the resulting word through the output address, and returns:

- Upper half: the header word captured for that final output write.
- Lower half: the retained output-address word.

The upper half is the captured word, even if the final output write aliases and changes container storage afterward.

No loop comparison, position advancement, or call to F-aab94c7c2aad occurs on this branch.

### Iterative branch

The iterative branch begins by calling F-ca7a1f20809e with the current-position address and the endpoint value directly. Unlike the two preliminary comparisons, its second argument is not replaced by the address of a newly prepared temporary.

A result of **exactly one** terminates the loop immediately.

Every other result, including zero, two, and negative values, causes an iteration. Each iteration has the following observable order:

1. Read the 32-bit word at the current-position address and retain it.
2. Call F-017ac946138e with the current-position address.
3. Copy the retained pre-call word into writable 32-bit scratch storage.
4. Call F-aab94c7c2aad with:
   - The address of the scratch word initially containing the container address.
   - The original container address.
   - The address of the scratch word containing the retained pre-advancement word.
5. Read the first scratch word after that call and write it through the output address.
6. Return to the loop comparison.

The ordering in steps 1–4 is significant. F-aab94c7c2aad receives a word captured before F-017ac946138e, although the call to F-017ac946138e occurs first. If that dependency changes the current-position storage, its updated value does not replace the captured word for the current iteration.

The first scratch word is initialized to the container address only once. The function does not restore that initial value between iterations. Changes made through its address by F-aab94c7c2aad persist into later iterations.

The second scratch word is replaced with each iteration’s newly captured position word before F-aab94c7c2aad receives it. That dependency may then alter it.

When the loop terminates, the packed return contains:

- Upper half: the final first scratch word.
- Lower half: the final second scratch word.

If there were no iterations, these remain the container address and output address, respectively. In that case, the output destination still contains the value written during initialization, subject to any dependency side effects.

The function has no visible iteration limit. Termination depends on a later loop comparison returning exactly one.

## Side effects

### Direct memory writes

The function directly writes:

- One 32-bit word through the output address during initialization.
- On the bulk branch, header fields at offsets 8, 4, and 12, followed by container offset 20, followed by another output write.
- On each loop iteration, one 32-bit word through the output address after F-aab94c7c2aad returns.

Temporary words are also exposed by address to dependencies. Those calls can affect later output and return values as described above.

### Dependency effects and ownership

No allocation or release operation is directly identifiable in this function. The calls do not establish whether nodes, elements, payloads, or header objects are destroyed or released.

In particular:

- F-8907a0bdd6c2 may perform substantial work before the explicit resets, but its ownership behavior is unknown.
- F-aab94c7c2aad may mutate the container and both scratch words supplied by address.
- F-017ac946138e may mutate the current-position representation.
- F-ca7a1f20809e cannot be assumed pure or restricted to returning a comparison result.

The function does not establish that dependencies preserve globals, container fields, output storage, or other reachable memory.

### Errors and partial progress

There is no explicit error return or rollback path.

Dependency results are used as branch controls, not decoded as documented error statuses. If a dependency fails to return normally or a memory access fails, the supplied evidence establishes no recovery behavior. Earlier output writes, completed iterations, or completed structural writes may already have taken effect.

## Dependencies

| Opaque ID | Observed role and call shape |
|---|---|
| F-ca7a1f20809e | Comparison-like dependency. Called first with current-position storage and an address of a copied header field; optionally with the endpoint value and an address of a copied header address; then repeatedly with the current-position address and endpoint value directly. |
| F-8907a0bdd6c2 | Bulk-operation dependency. Called with the original container address and the freshly read word at current header offset 4. Its result is ignored. |
| F-017ac946138e | Position-operation dependency. Called once per loop iteration with the current-position address, after its first word has been captured. |
| F-aab94c7c2aad | Per-position operation. Receives two writable scratch-word addresses and the original container address. The first scratch word subsequently supplies the output write; both supply the eventual packed return. |

These are observations of how the dependencies are used, not independent specifications of their behavior.

## Acceptance examples

The following examples are conditional static acceptance cases. They assume accessible, nonoverlapping storage and only the explicitly stated dependency effects. They have not been executed.

### Immediate loop termination

Assume the first comparison returns zero and the first loop comparison returns one.

Expected observations:

- The output receives the initially observed header address.
- The second preliminary comparison does not occur.
- Container offset 20 is not read as a bulk-branch gate.
- Neither position-operation dependency is called.
- The packed return has the container address in its upper half and the output address in its lower half.

### Bulk operation selected

Assume both preliminary comparisons return nonzero, container offset 20 is nonzero when checked, and F-8907a0bdd6c2 preserves the header address.

Expected observations:

- F-8907a0bdd6c2 receives the word read from header offset 4.
- After that call, header offsets 8 and 12 contain the header address, header offset 4 contains zero, and container offset 20 contains zero.
- The output is written again with the header address.
- The return’s upper half contains that header address; its lower half contains the output address.
- No loop comparison or per-position operation occurs.

### Header replaced by the bulk dependency

Assume the bulk branch is selected and F-8907a0bdd6c2 changes container offset 16 from header A to an accessible header B.

Expected observations:

- The dependency’s second argument was obtained from header A before the call.
- The subsequent explicit header resets target header B.
- The final output write and upper return half use the subsequently observed header B address.

This case prevents treating the pre-call header as cached across the dependency.

### Noncanonical comparison result

Assume the bulk branch is not selected and a loop comparison returns two.

Expected observations:

- The loop does not terminate.
- The function captures the current-position word, calls F-017ac946138e, calls F-aab94c7c2aad, and performs the output write.
- A later comparison must return exactly one for normal loop termination.

### Captured position and writable results

Assume one iteration begins with the current-position word equal to X. F-017ac946138e changes that storage to Y. F-aab94c7c2aad changes its first scratch word to R and its second scratch word to S. The next comparison returns one.

Expected observations:

- F-aab94c7c2aad initially receives X through its third argument, not Y.
- The output receives R.
- The packed return contains R in its upper half and S in its lower half.

## Uncertainties

The following are **documented_only**:

- The public calling convention and intended interpretation of the packed return.
- Whether the header is a sentinel, and whether its fields are parent, root, minimum, maximum, or other links.
- Whether container offset 20 is an element count.
- The representation of the endpoint value and the full position representation.
- Dependency contracts, ownership rules, destruction behavior, global effects, and failure semantics.
- Valid aliasing arrangements and whether callers are required to keep the arguments disjoint.
- Alignment, concurrency, synchronization, and memory-fault behavior.
- Whether the comparison dependency is guaranteed to return only zero or one.
- Whether the iterative operation makes progress or eventually terminates.

The range-removal and container-clear interpretations are inferred intent only. They do not replace the established branch conditions, access ordering, or packed-return behavior.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

Established evidence includes the 32-bit field accesses, byte offsets, dependency argument shapes, output-write ordering, explicit bulk resets, exact-one loop termination condition, and concatenation of two 32-bit words into a 64-bit return.

No dependency implementations, caller contracts, runtime observations, or runtime tests were supplied or used. No runtime tests have been performed.
