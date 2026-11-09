<!-- Trusted provenance: F-14aae3ee8b7e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-14aae3ee8b7e

## Purpose

F-14aae3ee8b7e searches a linked structure, then either returns a selected existing node or delegates an insertion request. It communicates the result through caller-provided storage: a four-byte value followed by a one-byte disposition.

Two policies are established:

- A nonzero policy byte requests insertion after the search, without checking an existing candidate.
- A zero policy byte enables candidate selection and a final comparison. A zero final comparison returns the candidate with disposition zero; a nonzero final comparison requests insertion.

The structure resembles an ordered tree with a header sentinel, predecessor navigation, and a policy controlling whether equivalent entries may coexist. These are inferred meanings, not established dependency contracts. In particular, the function does not establish comparator ordering, successful insertion, allocation, balancing, or ownership.

## Interface

The function receives three arguments and has no explicit return value.

| Argument | Established use |
|---|---|
| Result-storage address | Receives a four-byte result at offset 0 and a one-byte disposition at offset 4. |
| Structure address | Provides access to a header-pointer field, a policy byte, and an embedded comparison context. |
| Incoming value | An opaque four-byte value passed to comparison and insertion dependencies. |

The supplied representation uses four-byte scalar storage and address operations consistent with a 32-bit target. Pointer-bearing fields below describe that apparent layout; an independently verified ABI or wider-pointer interface is not available.

### Result layout

| Offset | Width | Meaning |
|---|---:|---|
| 0 | 4 bytes | Selected candidate address, or the contents of an insertion output slot after the insertion dependency returns. |
| 4 | 1 byte | Zero for a selected existing candidate; one for an insertion-request path. |
| 5 onward | Unspecified | No direct write by this function. |

Disposition one identifies a control-flow outcome, not independently verified insertion success.

### Structure layout

| Offset | Width or form | Use |
|---|---|---|
| 0x10 | Pointer-bearing field | Header or sentinel address, read at multiple distinct points. |
| 0x18 | 1 byte | Zero enables candidate checking; any nonzero value bypasses it. |
| 0x19 | Embedded address | Passed as comparison context. The function passes this address, rather than loading a pointer from it. |

### Header and node layout

| Offset | Width or form | Observed use |
|---|---|---|
| 0x00 | 1 byte | Zero/nonzero test during predecessor-like navigation. |
| 0x04 | Pointer-bearing field | Initial search link in the header; upward navigation link in nodes. |
| 0x08 | Pointer-bearing field | Search link for a nonzero comparison; boundary value in the header; navigation link in nodes. |
| 0x0c | Pointer-bearing field | Search link for a zero comparison and a downward-navigation link. |
| 0x10 | Embedded address | Node payload address passed to the search comparator. |

“Parent,” “left,” “right,” and “root” are plausible interpretations, not required meanings of these fields.

### Preconditions and widths

Normal completion requires accessible result storage, structure fields, reached nodes, and dependency-accessed objects. The initial header must be readable even when its initial search link is null. Navigation requires suitable link and sentinel invariants; not every pointer dereference is protected by a null check.

The incoming value and both insertion output slots are four bytes wide. The comparison result is tested only for zero versus nonzero. No sign-sensitive comparison, key arithmetic, size calculation, or integer scaling is performed by this function.

Address calculations use fixed byte offsets. Overflow, invalid address formation, alignment, cycles, and concurrent mutation are not checked. Safe overlap between result storage and the structure or nodes is not established.

## Behavior

### Search and retained insertion position

The remembered comparison outcome initially counts as nonzero.

Initialization performs distinct reads in this order: read the structure’s header field, read that header’s offset-0x04 link as the initial search node, then read the structure’s header field again as the retained insertion position. These reads do not establish a coherent snapshot if the header field changes.

Each visited search node causes both four-byte insertion output slots to be directly set to zero before the comparison call. F-31a0b2b5649a receives the embedded context address, the incoming value, and the current node’s offset-0x10 payload address.

After that dependency returns, the current node becomes the retained insertion position. The next link is then read:

- A zero comparison selects offset 0x0c.
- Any nonzero comparison selects offset 0x08.

Thus a negative comparison selects the same link as a positive nonzero comparison. The link read occurs after the comparison call and can observe link changes made by that dependency.

Search stops on a null selected link. The retained position is the last visited node, or the second header value if no node was visited. An empty search performs no comparison calls and does not perform the per-iteration zeroing of either insertion output slot.

### Policy and boundary decision

The policy byte is read after search.

A nonzero policy goes directly to ordinary insertion. A zero policy initializes a separate candidate slot from the retained insertion position.

If the last search comparison was zero, candidate checking proceeds without a boundary call or predecessor navigation.

If the remembered comparison is nonzero, including after an empty search, the function reads the structure’s header field again and copies that header’s four-byte offset-0x08 value into the **ordinary insertion output slot**. It then calls F-227cdabeb472 with:

- The address of the candidate slot.
- The address of that same ordinary insertion output slot.

Both slots are writable through the supplied addresses. The boundary dependency may therefore change the candidate and the ordinary output slot.

A nonzero boundary result selects early insertion. A zero boundary result selects navigation from the candidate value present after the boundary call. The retained insertion position remains separate from the writable candidate slot.

The boundary dependency’s actual comparison rule is unknown; “boundary” is an explanatory label.

### Predecessor-like navigation and repeated reads

Navigation begins by reading the candidate’s byte at offset 0x00.

If that byte is zero, the function reads the candidate’s offset-0x04 link and then the referenced object’s offset-0x04 link. If the second link equals the candidate, the candidate is replaced by its offset-0x0c link and no other navigation route is taken.

Otherwise, the function uses offset-0x08 links to choose between downward and upward navigation. **Two separate reads occur:** the candidate’s offset-0x08 link is first saved as a possible next node, and that field is read again to select the branch. Agreement between these reads is not guaranteed by the visible body.

If the second read is nonnull, downward navigation begins using the **first** read’s value. At each downward step:

- The saved next node becomes the candidate.
- Its offset-0x0c link is read and saved for a possible next step.
- That same field is read again to decide whether navigation continues.

A nonnull second read continues using the first read’s saved value. A null second read stops with the current candidate, regardless of the saved value.

Under stable links, this follows offset 0x08 once and then offset 0x0c until reaching a node with a null offset-0x0c link. Without stability, the saved link and branch-controlling read may differ. For example, continuation may be selected while the saved next value is null, causing a later invalid dereference. A nonnull terminal candidate is therefore not an unconditional guarantee.

If the second offset-0x08 read is null, upward navigation starts from the candidate’s offset-0x04 link; the first offset-0x08 value is discarded. Each upward step reads the referenced node’s offset-0x08 link, compares it with the current candidate, and replaces the candidate with the referenced node. This replacement occurs even on the final unequal comparison. Equality causes another upward step through offset 0x04.

The upward route and the special zero-byte route contain unguarded dereferences. Their safety depends on valid links and sentinel invariants.

### Candidate extraction and acceptance

After candidate selection, an additional four-byte local slot is set to zero. Its address is not passed to the extraction dependency; this assignment does not initialize either insertion output slot.

F-434e8632518e receives the candidate address and returns a four-byte value. F-31a0b2b5649a then receives the embedded context address, that returned value, and the incoming value.

This reverses the operand roles relative to search comparisons: the incoming value was second during search and is third in the final comparison.

A zero final comparison causes the candidate address to be written to result offset 0, followed by disposition zero at offset 4. No insertion dependency is called on that path.

A nonzero final comparison selects ordinary insertion. Zero is the established acceptance condition, but it is not independently established to mean equality. Equivalence would require additional comparator and search-invariant contracts.

### Insertion calls and output-slot identity

Both insertion paths call F-46e7f9a196d3 with an output-slot address, the structure address, literal zero, the retained insertion position, and the incoming value.

The output slots are distinct:

| Insertion path | Output storage supplied | Contents established before the call |
|---|---|---|
| Early insertion after a nonzero boundary result | Separate early-insertion slot | Directly zeroed during every search iteration; therefore zero after a nonempty search, but uninitialized by this function after an empty search. |
| Ordinary insertion after a boundary call | The same four-byte slot previously supplied as the boundary dependency’s second argument | Header offset-0x08 value, including changes made through that slot by the boundary dependency. No subsequent direct reset occurs before insertion. |
| Ordinary insertion without a boundary call, after nonempty search | Ordinary insertion slot | Directly zeroed during the search. |
| Ordinary insertion without a boundary call, after empty search | Ordinary insertion slot | Uninitialized by this function. |

These are the function’s established assignments and addressed-slot relationships. Unknown dependency effects, including writes through retained addresses during intervening calls, are not ruled out.

For either insertion path, the function consumes the output slot’s four-byte contents after normal return and writes them to result offset 0. It subsequently writes disposition one at offset 4.

The helper need not overwrite the slot merely to make an already-initialized value readable. If it leaves an initialized slot unchanged, the function consumes the prior contents; whether those contents constitute a valid insertion result is unknown. An uninitialized slot left uninitialized has no established valid four-byte result.

Insertion always receives the retained search position, not a candidate subsequently replaced by the boundary dependency or navigation.

## Side effects

### Direct effects and ordering

The function directly reads structure, header, and node fields; writes local storage; calls dependencies; and writes the result.

On every normally completed result path, the four-byte result write precedes the one-byte disposition write. The pair is not established as an atomic publication.

The byte used for disposition occupies the low-addressed byte of the early-insertion output slot. After early insertion, the full four-byte helper output is copied to the result before that byte is changed to one. Ordinary insertion also sets that byte to one; existing-candidate selection sets it to zero. Consequently, a dependency retaining the early output address would not be entitled to assume its contents remain unchanged after return.

The visible body performs no direct tree-link updates, allocation, release, balancing, or explicit ownership transfer.

### Dependency effects and lifetime

No dependency is established to preserve globals, the structure, links, payloads, context, or previously exposed local storage. Dependencies may have externally visible effects not described by this body.

The local addresses supplied to dependencies designate temporary storage. No right to retain or use them after this function returns is established. Ownership and lifetime of the incoming value, extracted value, selected node, and insertion result remain undocumented.

### Errors and abnormal completion

There is no explicit error result, validation path, recovery, or cleanup.

Disposition one follows a normally returning insertion call without a separate success check. The helper’s behavior when reading, partially overwriting, or leaving its output storage unchanged is unknown.

Invalid pointers, malformed links, nonterminating traversal, dependency failures, and mutation-related inconsistencies are not converted into defined failures. If execution stops before the final writes, complete result initialization is not guaranteed.

## Dependencies

| Opaque ID | Observed interface | Unestablished contract |
|---|---|---|
| F-31a0b2b5649a | Context address and two operands; result tested for zero/nonzero. Used during search and final candidate checking. | Ordering, equality, stability, mutation, failure behavior. |
| F-434e8632518e | Candidate address; returns a four-byte value. | Extraction meaning, side effects, ownership, lifetime. |
| F-227cdabeb472 | Addresses of the candidate slot and ordinary insertion output slot; result selects early insertion or navigation. | Comparison semantics, slot modifications, retention, external effects. |
| F-46e7f9a196d3 | Output-slot address, structure address, zero, retained search position, incoming value. | Reads versus overwrites of output storage, insertion success, allocation, balancing, failure reporting, ownership. |

Passing a slot address is materially different from passing its value. In particular, ordinary insertion receives the identical storage object previously exposed to the boundary dependency, when that call occurred.

The evidence does not establish argument-expression evaluation order within individual calls.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless an example states otherwise, storage and links are valid, links remain stable, dependencies return normally, and no additional dependency mutations occur.

1. **Empty search with nonzero policy.** No search comparison or boundary call occurs. Ordinary insertion receives the second header value as its retained position and an output slot uninitialized by this function. If the helper writes four-byte value R, the result becomes R with disposition one.

2. **Nonempty search with nonzero policy and no output overwrite.** Both insertion slots have been zeroed by the search. If ordinary insertion returns without changing its output slot, the result value is zero and disposition is one. The read is not uninitialized solely because the helper did not write; semantic insertion success remains unknown.

3. **Boundary changes ordinary output storage.** The boundary dependency replaces the header-derived slot contents with M and returns zero. Navigation and candidate checking complete, with a nonzero final comparison. Ordinary insertion receives the same slot containing M. If it leaves that slot unchanged, the result is M with disposition one.

4. **Early insertion uses separate storage.** After a nonempty search, the boundary dependency changes its second addressed slot to M and returns nonzero. Early insertion receives its separate zeroed slot, not the slot containing M. If the helper leaves its output unchanged, the result is zero with disposition one. Extraction and final comparison do not occur.

5. **Empty-search early insertion.** The boundary call initializes and may modify the ordinary slot, then returns nonzero. The separate early slot remains uninitialized by this function. A helper that does not establish its contents leaves no defined valid result value.

6. **Direct candidate acceptance.** The last search comparison is zero and its offset-0x0c link is null. Boundary checking and navigation are skipped. If extraction followed by final comparison produces zero, the selected search node is returned with disposition zero.

7. **Navigation does not replace insertion position.** Boundary checking returns zero and navigation selects another candidate. A nonzero final comparison causes insertion with the original retained search position, not the navigated candidate.

8. **Repeated navigation reads disagree.** The first offset-0x08 read is null, but the second is nonnull. Downward navigation is selected using the saved null value. The next candidate-field access is not protected by a null check; successful navigation is not guaranteed.

9. **Downward continuation reads disagree.** A downward step saves a null offset-0x0c link, then rereads a nonnull value for continuation. The next iteration uses the saved null link. Conversely, a saved nonnull link followed by a null continuation read stops without following the saved link.

10. **Negative search comparison.** Any negative nonzero comparison selects offset 0x08, exactly as a positive nonzero comparison does.

## Uncertainties

### Established static evidence

The supplied body establishes the field offsets, distinct header reads, zero/nonzero decisions, operand order, repeated navigation reads, separate candidate and retained-position roles, insertion-slot identity and initialization paths, and result-write ordering.

### Inferred intent

The layout suggests an ordered tree with a header sentinel. Navigation resembles predecessor selection. The result resembles a node-or-iterator paired with an insertion indicator. These interpretations do not strengthen unknown dependency contracts.

### documented_only

The following remain unresolved:

- Independently verified ABI, pointer width, alignment, and byte order.
- Comparator ordering and equivalence semantics.
- Header, node-byte, and sentinel meanings.
- Valid empty-structure configuration.
- Boundary and extraction semantics.
- Insertion behavior when output storage already contains a value.
- Output initialization, insertion success, and failure guarantees.
- Allocation, ownership, release, and lifetime rules.
- Aliasing support, thread safety, reentrancy, and callback mutation.
- Guarantees making repeated link reads agree.
- Defined behavior for violated link invariants or invalid addresses.

## Evidence

This contract derives from static inspection of the supplied body for F-14aae3ee8b7e and its supplied opaque dependency mapping. Dependency implementations, authoritative type definitions, and external contracts were not provided.

No runtime tests have been performed. Acceptance examples state conditional consequences of the visible behavior, not observations of an executing implementation.
