<!-- Trusted provenance: F-3cf3e7effef1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3cf3e7effef1

## Purpose

F-3cf3e7effef1 searches a linked structure and either publishes a candidate value or delegates to F-246bc4d6ef61. Its output consists of a 32-bit word followed by a one-byte path flag.

**Established behavior:** search uses zero/nonzero comparison results to choose links. A container control byte can bypass candidate checking. Otherwise, conditional boundary checking and predecessor-like navigation precede a comparison with reversed operand order. A zero final comparison selects flag 0; every normally completed insertion-dependency path selects flag 1.

**Inferred intent:** the structure resembles an ordered tree, and the operation resembles insertion with optional duplicate handling. The supplied evidence does not establish ordering laws, equality semantics, successful insertion, balancing, or the control byte’s policy meaning.

## Interface

The function takes three inputs and has no explicit return value.

| Input | Established role |
|---|---|
| Result address | Destination for a four-byte word and a one-byte flag. |
| Container address | Base for header, control, and comparison-state locations. |
| Original operand | Opaque 32-bit value passed to comparison and insertion dependencies. Its underlying type is unknown. |

### Widths and output layout

The supplied representation supports a 32-bit model for links, candidate values, opaque words, and dependency results. Branches distinguish zero from nonzero, not negative from positive.

| Result offset | Width | Meaning |
|---|---:|---|
| 0 | 4 bytes | Candidate value read after the final comparison, or insertion output word. |
| 4 | 1 byte | 0 for the zero-final-comparison path; 1 for an insertion-dependency path. |

The word is written before the flag. These are separate stores, not an atomic publication. No explicit writes initialize result bytes beyond offset 4.

### Accessed layout

| Object | Offset | Width or use |
|---|---:|---|
| Container | 16 | Four-byte header address |
| Container | 24 | One-byte control field |
| Container | 25 | Address supplied as comparison state |
| Header | 4 | Four-byte initial search-node address |
| Header | 8 | Four-byte boundary value |
| Node-like object | 0 | One-byte marker |
| Node-like object | 4 | Four-byte parent-like link |
| Node-like object | 8 | Four-byte first-child link |
| Node-like object | 12 | Four-byte second-child link |
| Node-like object | 16 | Address supplied as a comparison operand |

“Parent,” “first child,” and “second child” describe traversal roles, not independently established structural invariants. Forming an offset-16 address does not itself establish the size or representation of the comparison operand stored there.

### Preconditions

All memory accesses reached on the selected path must be valid. There is no explicit argument validation, bounds checking, cycle detection, synchronization, or recovery from invalid links.

Dependencies must return normally for normal output publication. The result destination must remain writable through both final stores. Dependencies receiving local-slot addresses must respect their actual lifetimes; no lifetime extension is established.

Address calculations use the represented addresses and byte offsets. No explicit arithmetic-overflow checks appear. Behavior for an unrepresentable address calculation is **documented_only**, rather than a promised wrapping-address rule.

## Behavior

### 1. Initial reads

The remembered comparison result begins at 1.

The entry read order is:

1. Read the header address at container offset 16.
2. Read the initial search-node address at that header’s offset 4.
3. Read container offset 16 again and retain this second header value as the initial terminal candidate.

The two header reads are distinct. No coherent snapshot is guaranteed if the container changes between them.

If the initial search-node address is zero, no search comparison occurs. The terminal candidate remains the second header value, and the remembered result remains 1.

### 2. Directional search

For each nonzero search node, the function clears two local insertion-output words and calls F-31a0b2b5649a with:

1. Container address plus 25.
2. Original operand.
3. Search-node address plus 16.

After the call, the visited node becomes the terminal candidate and the returned integer becomes the remembered result.

The function then reads exactly the selected child link:

- Zero result: read and follow offset 12.
- Nonzero result: read and follow offset 8.

The child read occurs after the dependency returns. A dependency’s mutation of that link can therefore affect traversal.

Search stops when the selected link is zero. The terminal candidate is the last visited node, or the second header value if no node was visited. All nonzero comparison results select the same direction; −7 and 1 are not distinguished.

### 3. Control decision

After search finishes, the function reads the control byte at container offset 24.

- Nonzero: take the common insertion path without boundary checking or final comparison.
- Zero: copy the terminal candidate into an adjustable local candidate slot.

This control read occurs after all search comparisons. Its value is not an entry-time snapshot.

If the remembered search result is zero, candidate adjustment is skipped and processing reaches the final comparison. If it is nonzero, boundary checking occurs.

### 4. Boundary check

The function freshly reads container offset 16, then reads offset 8 of that header into a local boundary slot.

It calls F-a823726641db with the addresses of:

1. The adjustable-candidate slot.
2. The boundary slot.

Both slots are writable through the supplied addresses. The dependency’s implementation and retention behavior are unknown.

- Nonzero return: take the early insertion path.
- Zero return: perform predecessor-like navigation using the candidate slot’s value after the call.

Early insertion receives the retained terminal candidate, not the possibly modified adjustable candidate.

The boundary slot also serves as the common insertion-output slot on other paths. Its boundary value is not an independent insertion result.

### 5. Predecessor-like navigation

This phase occurs only after a nonzero remembered search result and a zero boundary-check result.

#### Special marker case

The function reads the candidate’s marker byte at offset 0.

If the marker is zero, it reads the candidate’s offset-4 link and then offset 4 of the linked object. If that second link equals the candidate, the function replaces the adjustable candidate with the candidate’s offset-12 link and proceeds to the final comparison.

If the marker is nonzero, those two parent-related reads are skipped. A zero marker whose subsequent equality check fails also proceeds to ordinary navigation.

#### Ordinary navigation: two distinct offset-8 reads

The function first reads the candidate’s offset-8 link and saves it as a prospective next node. It then reads the candidate’s offset-8 link again for a separate zero/nonzero decision.

These reads must not be collapsed:

- If the **second** read is zero, the function takes the upward path and does not descend through the first value.
- If the **second** read is nonzero, the function takes the downward path using the **first** value.

Thus a first value of zero followed by a nonzero second value selects descent through zero. The function supplies no reconciliation or protective null check for that discrepancy.

#### Downward path: two distinct offset-12 reads per iteration

Each downward iteration:

1. Replaces the adjustable candidate with the saved prospective next node.
2. Reads that candidate’s offset-12 link into the prospective-next-node temporary.
3. Reads that candidate’s offset-12 link again to decide whether to continue.

A zero **second** read stops descent at the current candidate. A nonzero second read starts another iteration using the **first** read’s value.

Consequently, a nonzero first read followed by zero stops without visiting the first-read node. A zero first read followed by nonzero attempts the next iteration using zero. No stable snapshot or guaranteed behavior under changing links is established.

#### Upward path

The upward path first reads the current candidate’s offset-4 parent-like link.

At each step, it reads the parent’s offset-8 link and compares it with the current candidate. It then replaces the candidate with that parent, regardless of the comparison outcome.

- Unequal: stop with that parent as the adjusted candidate.
- Equal: read the current parent’s offset-4 link and repeat.

The final candidate is therefore the parent involved in the first unequal relationship, not the child from which that relationship was examined.

No general null checks or cycle limits protect this navigation. Safety and termination depend on the accessed structure.

### 6. Final comparison and publication

Before the final comparison, the function clears the common insertion-output word. It also clears another local word that is not subsequently used in the shown behavior.

It reads the adjustable-candidate slot to form the comparison operand and calls F-31a0b2b5649a with:

1. Container address plus 25.
2. That candidate value plus 16.
3. Original operand.

This reverses the search comparison’s operand order.

A nonzero result selects common insertion.

A zero result causes a **new read of the adjustable-candidate slot after the dependency returns**. The function writes that then-current value to result offset 0, then writes flag 0 at offset 4.

The operand-forming candidate and the published candidate need not be identical. On paths through boundary checking, F-a823726641db has received the candidate slot’s address. No established contract prevents it from retaining that address or permits assuming that a later dependency cannot modify the slot indirectly. A zero final result therefore selects publication of the post-call slot value, not an unconditional return of the candidate whose offset-16 address was compared.

This function does not independently validate key equality or recompare a changed candidate.

### 7. Insertion paths

Both insertion paths call F-246bc4d6ef61 with:

1. Address of a local four-byte output slot.
2. Original container address.
3. Zero.
4. Retained terminal candidate from the initial search.
5. Original operand.

The early and common paths use different output slots. Neither substitutes the adjusted candidate for the retained terminal candidate.

After normal dependency return, the function reads the corresponding output slot, writes that word to result offset 0, then writes flag 1 at offset 4.

There is no independent success check or validation of the returned word.

## Side effects

### Direct writes and timing

Direct writes affect local storage and the two result fields. No direct container-field or node-link write appears.

All traversal and relevant dependency calls precede result publication. If result storage aliases a container or node, the final stores can overwrite that storage; aliasing is not checked.

Separate word and flag stores allow an intermediate state containing the new word and the previous flag. No atomicity or thread-safety guarantee is established.

### Output-slot initialization

Initialization depends on the path:

- Each search iteration clears both insertion-output words before comparison.
- Boundary checking overwrites the common output word with the freshly read boundary value and exposes its address to F-a823726641db.
- The final-comparison path clears the common output word before calling F-31a0b2b5649a.
- Empty search followed by nonzero control reaches common insertion without an established initialization of its output word.
- Empty search followed by early insertion reaches that path without an established initialization of its separate output word.

These stores do not establish a universal zero fallback. In particular, dependencies may mutate exposed slots. A meaningful insertion result requires an appropriate dependency output contract, which is unavailable.

### Ownership and errors

No direct allocation, release, ownership transfer, or lifetime extension is shown. Publishing a node-like word does not establish ownership of the referenced object.

The flag identifies the selected path, not verified success or failure. There is no explicit error result, exception handling, rollback, or recovery. If a dependency fails to return normally, completed result publication is not established.

## Dependencies

| Opaque ID | Established interaction | Unknown contract |
|---|---|---|
| F-31a0b2b5649a | Receives comparison-state address and two operands; returns a 32-bit zero/nonzero branch value. Search and final operand orders differ. | Ordering, equality, mutation, allocation, errors, and preservation of other state. |
| F-a823726641db | Receives writable candidate-slot and boundary-slot addresses. Nonzero selects early insertion; zero selects navigation. | Predicate meaning, slot mutations, retention of addresses, indirect interactions with later calls, and other effects. |
| F-246bc4d6ef61 | Receives output-slot address, container, zero, terminal candidate, and original operand. Its output slot is read after return. | Actual insertion, output guarantees, allocation, balancing, ownership, duplicate handling, and failure signaling. |

No dependency is assumed pure, globally preserving, or responsible for acquiring or releasing memory. Reads after calls remain distinct from earlier observations.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Control bypass.**  
   A nonempty search ends at candidate A, and the subsequently read control byte is nonzero. Boundary checking and final comparison are skipped. Insertion receives A. If its output slot contains W on return, publication is W followed by flag 1.

2. **Unadjusted candidate publication.**  
   The control byte is zero and the last search comparison returned zero. The final comparison uses the terminal candidate’s offset-16 address before the original operand. If it returns zero and the adjustable slot remains unchanged, that candidate is published with flag 0. Insertion is not called.

3. **Boundary mutation followed by early insertion.**  
   Boundary checking changes the adjustable slot from A to B and returns nonzero. Early insertion still receives terminal candidate A. No final comparison occurs.

4. **Stable-link descent.**  
   Boundary checking returns zero, the special marker case does not apply, and both offset-8 reads agree on a nonzero child. If each pair of offset-12 reads also agrees, descent reaches the last node whose offset-12 link is zero. Its offset-16 address supplies the final comparison operand. A zero result publishes that same node only if the candidate slot remains unchanged during the call.

5. **Disagreeing offset-8 reads.**  
   The first offset-8 read yields A; the second yields zero. The function takes the upward path, not descent through A. Conversely, a zero first value and nonzero second value select descent using zero, with no safety guarantee.

6. **Disagreeing offset-12 reads.**  
   During descent, the first offset-12 read yields A and the second yields zero. Descent stops at the current candidate rather than visiting A. If the first yields zero and the second is nonzero, the next iteration uses zero.

7. **Post-comparison candidate change.**  
   After boundary checking has exposed the candidate slot, the final comparison is called with candidate A plus 16. Suppose an indirect mutation changes the slot to B before that comparison returns zero. The function publishes B and flag 0 without comparing B.

8. **Upward stopping point.**  
   A candidate has no first child according to the second offset-8 read. At the first parent whose offset-8 link differs from the former candidate, ascent stops after installing that parent as the adjusted candidate.

9. **Empty search.**  
   An initial zero search-node address causes no search comparison. The remembered result remains 1. Nonzero control selects insertion using the second header value read at entry. Zero control first invokes boundary checking; subsequent safety depends on its result, mutations, and sentinel layout.

10. **Nonzero is not signed ordering.**  
    A search result of −7 selects offset 8 just as 1 does. A final result of −7 selects insertion, not flag 0.

## Uncertainties

The following remain **documented_only**:

- Exact container type, balancing discipline, and sentinel invariants.
- Whether the control byte enables duplicates or another policy.
- Whether comparison results implement an ordering or equality relation.
- Header metadata semantics beyond the observed accesses.
- Original operand representation and lifetime.
- Dependency mutations, retained-address use, ownership, output guarantees, and errors.
- Defined behavior for unwritten insertion output slots.
- Behavior under changing, inconsistent, cyclic, or invalid links.
- Target endianness, complete ABI, and invalid-address arithmetic semantics.

Repeated reads are established; coherent values across those reads are not. Candidate-slot exposure is established; whether any dependency actually retains or later mutates it is unknown.

## Evidence

**Established static evidence:** input usage, represented widths, offsets, repeated header reads, separate navigation-link reads, dependency argument order, zero/nonzero decisions, slot exposure, conditional initialization, post-comparison candidate reread, and result-store order follow from the supplied function representation.

**Inferred intent:** ordered-tree insertion, duplicate detection, and predecessor selection explain the structure but are not independent guarantees.

**Unavailable evidence:** dependency implementations, callers, complete types, allocation contracts, synchronization rules, and runtime observations.

No runtime tests have been performed.
