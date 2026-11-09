<!-- Trusted provenance: F-5dc6be8926dd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5dc6be8926dd

## Purpose

F-5dc6be8926dd traverses a linked structure using an opaque comparison dependency. It then either selects a candidate already present in that structure or calls an insertion-like dependency. On normal completion, it writes a 32-bit result word followed by a one-byte path indicator.

The indicator is:

- **Zero:** the candidate was returned without this function calling F-e5fa1f12a468.
- **One:** this function called F-e5fa1f12a468 and then copied its output storage into the result.

The indicator does not independently establish successful insertion, allocation, uniqueness, or ownership transfer.

**Inferred intent:** the structure resembles an ordered tree with a header, and the operation resembles insertion with duplicate detection. The established contract is the access and call behavior below, not that interpretation.

## Interface

### Arguments and result

The function receives three arguments:

| Argument | Established use |
|---|---|
| Result-storage address | Destination for a 32-bit word at byte offset `+0` and a one-byte indicator at `+4` |
| Container address | Base for field accesses and an argument to dependencies |
| Supplied operand | Opaque 32-bit value passed to comparison and insertion-like dependencies |

There is no explicit return value.

Result storage requires at least five writable bytes. No write to result offsets `+5` through `+7` is established. The function does not initialize the result before traversal or dependency calls.

The supplied representation uses 32-bit link and data words. Comparisons and the boundary helper produce 32-bit results tested only for zero versus nonzero. Address interpretation requires the corresponding 32-bit address model; the contract does not widen these words to a host-native pointer width.

### Accessed layout

All offsets are byte offsets.

| Object | Offset | Established role |
|---|---:|---|
| Container | `+0x10` | 32-bit header-like link |
| Container | `+0x18` | One-byte control flag |
| Container | `+0x19` | Address supplied as comparison context |
| Header-like object | `+4` | Initial traversal link |
| Header-like object | `+8` | Boundary word used on the helper path |
| Candidate node | `+0` | One-byte marker |
| Node | `+4` | Parent-like link |
| Node | `+8` | Link selected after a nonzero traversal comparison |
| Node | `+0xC` | Link selected after a zero traversal comparison |
| Node | `+0x10` | Address supplied as a traversal comparison operand |

No size or contents are established for the comparison context or node data beginning at `+0x10`.

### Dependency-facing temporary storage

Three distinct storage roles matter:

- **Candidate slot:** a 32-bit node value whose address is passed to F-2b931655bb54.
- **Early-output word:** a 32-bit word whose address is passed to F-e5fa1f12a468 only on the helper-selected early path.
- **Shared boundary/output word:** one 32-bit storage location used first for the boundary argument to F-2b931655bb54 and later, if reached, as the ordinary-path output argument to F-e5fa1f12a468.

The boundary slot and ordinary output slot are **the same storage**, not separate words with copied values. The early-output word is separate from both the candidate slot and the shared word.

Both output words are initialized to zero before **each traversal comparison**. They are not initialized there if traversal executes zero iterations. Subsequent assignments and permitted dependency mutations determine their later contents.

### Preconditions

Defined completion requires:

- Readable container fields and all objects reached by the selected accesses.
- A usable header-like object even when its initial traversal link is null.
- Valid result storage.
- Dependencies that accept the supplied values and temporary-storage addresses.
- Defined contents in any output bytes ultimately copied into the result.
- Termination of traversal, adjustment walks, and dependencies.

There are no explicit checks for invalid container, header, candidate, or result addresses. Cycles are not detected.

Non-aliasing is not established. Result writes can modify container or node storage if those regions overlap. Address additions use the represented address model; no bounds or overflow checks are shown.

## Behavior

### Initial traversal state

The remembered comparison outcome starts at nonzero.

The function first reads the container’s header-like link and then reads that object’s `+4` link as the initial traversal node. It separately reads the container’s header-like link again to initialize the retained node.

These are separate header-link reads. A single immutable header snapshot is not established.

### Traversal and per-iteration initialization

For every nonnull traversal node, the function performs the following ordered actions:

1. Writes zero to the entire 32-bit early-output word.
2. Writes zero to the entire 32-bit shared boundary/output word.
3. Calls F-bcefa5957e76 with the comparison-context address, supplied operand, and current node address plus `0x10`.
4. Retains the current node.
5. Reads the next link selected by the returned comparison result.

A zero comparison selects node offset `+0xC`. Any nonzero comparison selects node offset `+8`. Positive and negative nonzero results are indistinguishable for this decision.

The link read occurs after the comparison call. Dependency changes to the selected field can therefore affect traversal.

Initialization occurs on every iteration, not just the first. If traversal runs at least once, the last iteration establishes zero in both output words before its comparison call. This is a prior value, not a guarantee against later mutation through any accessible alias.

When traversal ends:

- The retained node is the last nonnull node visited.
- If no node was visited, it is the separately loaded header-like value.
- The remembered comparison outcome is the final traversal comparison result, or the initial nonzero value if traversal never ran.

### Control-flag decision

After traversal, the function reads the container’s one-byte control flag.

If nonzero, it proceeds directly to the ordinary insertion-like call. No candidate helper, adjustment, extraction, or final comparison occurs.

If zero, it initializes the candidate slot from the retained node.

The control flag is read after traversal comparisons. Those dependencies are not assumed to preserve it.

### Boundary helper and early path

When the control flag is zero and the remembered traversal outcome is nonzero, the function freshly reads the header-like link and then its `+8` word.

That boundary word is assigned to the **shared boundary/output word**, overwriting its previous contents. In particular, this overwrites the traversal-established zero if traversal ran.

F-2b931655bb54 then receives:

1. The candidate-slot address.
2. The shared boundary/output-word address.

Both arguments identify mutable temporary storage. Neither value is assumed preserved.

If the helper returns nonzero, F-e5fa1f12a468 is called immediately with:

1. The early-output-word address.
2. The container address.
3. A zero-valued argument.
4. The retained traversal node.
5. The supplied operand.

No candidate adjustment, extraction, or final comparison occurs on this path.

The retained node is separate from the candidate slot. Changing the candidate slot does not itself change the retained-node argument.

The early-output word retains its traversal-established zero if traversal ran and nothing subsequently changed it. If traversal never ran, no prior initialization of that word is established. The boundary assignment initializes the shared word, not the early-output word.

After the insertion-like call returns, the early-output word is copied to result offset `+0`, followed by indicator one at result offset `+4`.

### Candidate adjustment after a zero helper result

If F-2b931655bb54 returns zero, adjustment starts from the candidate slot’s **post-call value**.

The function first reads the candidate’s marker byte.

If the marker is zero, it follows two consecutive `+4` links and compares the resulting address with the candidate. If equal, it replaces the candidate with its `+0xC` link.

Otherwise, adjustment uses the candidate’s `+8` link:

- One read obtains a prospective next node.
- A separate read tests whether the field is null.

If the null test is nonzero, the function moves to the prospectively read node and descends through `+0xC` links. At each node, one read obtains the prospective continuation and another tests for null. Under stable field values, the terminal node with a null `+0xC` link becomes the candidate.

If the `+8` null test is zero, the function starts from the candidate’s `+4` link. At each step it:

- Compares that node’s `+8` link with the previous candidate.
- Replaces the candidate with that node, whether or not the comparison matched.
- Follows another `+4` link only when the comparison matched.

Thus the first nonmatching parent-like node is still installed as the candidate before adjustment ends.

Repeated reads are not a cached snapshot. If a field changes between the prospective-value read and its null test, the selected value and tested value can differ. No validation repairs such inconsistencies.

**Inferred intent:** this resembles predecessor selection with a header special case. Ordering, sentinel validity, and balancing invariants are external to this function.

### Candidate extraction and final comparison

Candidate extraction is reached when the control flag is zero and either:

- The remembered traversal outcome was zero, bypassing the helper and adjustment; or
- The helper returned zero and adjustment completed.

F-0d2e87dfdbd4 receives the candidate and returns a 32-bit value.

F-bcefa5957e76 then receives:

1. The comparison-context address.
2. The extracted 32-bit value.
3. The supplied operand.

This reverses the operand roles relative to traversal: traversal supplies the original operand before a node-data address; the final comparison supplies an extracted value before the original operand.

If the final comparison returns zero, the function writes the candidate to result offset `+0`, then writes indicator zero at `+4`. F-e5fa1f12a468 is not called.

A nonzero result selects the ordinary insertion-like path.

### Ordinary insertion-like path

F-e5fa1f12a468 receives:

1. The shared boundary/output-word address.
2. The container address.
3. A zero-valued argument.
4. The retained traversal node.
5. The supplied operand.

The output word’s prior contents depend on the path:

| Path | Established prior contents, absent intervening mutation |
|---|---|
| Nonzero control flag after at least one traversal iteration | Zero |
| Nonzero control flag with no traversal iteration | No initialized value established |
| Zero control flag, zero remembered traversal outcome, nonzero final comparison | Zero from traversal |
| Zero helper result followed by nonzero final comparison | Boundary assignment, as subsequently modified through the shared slot |

The third case necessarily includes a traversal comparison: the initial remembered outcome is nonzero.

On the helper path, there is **no reset** of the shared word between the helper and ordinary insertion-like call. A helper mutation can persist through candidate adjustment, extraction, and final comparison, subject to any intervening dependency effects.

After F-e5fa1f12a468 returns, the shared word is copied to result offset `+0`, followed by indicator one at `+4`.

## Side effects

### Result and temporary writes

On every normally completing path, the caller-visible result writes occur in this order:

1. One 32-bit result word at offset `+0`.
2. One byte at offset `+4`.

Between these writes, the function stages the indicator in the low-order byte of the early-output word. Consequently, the early-output word’s full earlier contents are not preserved through completion. On the early insertion-like path, its full output value is copied to the caller before that staging overwrite.

Other established temporary writes include the per-iteration zeroing, candidate updates, and boundary assignment described above.

The function does not directly assign container fields or node links. This does not exclude changes through dependency calls or aliasing.

### Partial or absent dependency output

F-e5fa1f12a468 is given existing storage, not a guaranteed fresh unspecified word.

If it writes only part of that word, the remaining bytes retain their prior contents. If it leaves the word unchanged, the entire prior value is copied. Therefore the result can depend on:

- Traversal’s zero initialization.
- The boundary assignment.
- Helper mutations to the shared word.
- Other intervening mutations through accessible aliases.

Where no initial value is established, a partial or absent write does not establish a fully defined result. The function performs no initialization check.

### Ownership and failures

No allocation, release, ownership transfer, or reference-count operation is directly established.

Temporary addresses designate storage available during this invocation. No permission or lifetime for using those addresses after return is established. Dependencies are not assumed to release memory, retain no pointers, or preserve globals.

There is no explicit error result, rollback, exception handling, or recovery path. A dependency that does not return normally can prevent either result write. Invalid accesses or nonterminating walks do not produce a guaranteed result.

## Dependencies

| Opaque ID | Established interface use | Unresolved contract |
|---|---|---|
| F-bcefa5957e76 | Three arguments; 32-bit result tested for zero/nonzero | Ordering, operand representations, purity, state preservation |
| F-2b931655bb54 | Candidate-slot address and shared boundary/output-word address; zero/nonzero branch result | Predicate meaning, permitted mutations, other effects |
| F-0d2e87dfdbd4 | Candidate argument; 32-bit return used in final comparison | Extraction semantics, representation, ownership, lifetime |
| F-e5fa1f12a468 | Output-word address, container, zero, retained node, supplied operand | Insertion, allocation, balancing, failure signaling, output coverage, ownership |

No dependency is assumed to preserve globals, container contents, node contents, or storage reachable through a previously obtained address.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, memory is valid and stable between direct accesses, dependencies return normally, and dependencies make no mutations beyond those specified.

### Traversal initializes an unchanged ordinary output

Suppose traversal executes at least once, the control flag is nonzero, and F-e5fa1f12a468 leaves its output unchanged.

The ordinary output word remains zero from the final traversal iteration. The result is word zero and indicator one. The indicator does not prove insertion.

### Empty traversal does not initialize the early output

Suppose the initial traversal link is null, the control flag is zero, and the helper returns nonzero.

The shared word receives the boundary value, but the separate early-output word has no established initialization. If F-e5fa1f12a468 leaves that early word unchanged, a defined 32-bit result is not established.

If instead it fully writes a defined word, that word is returned with indicator one.

### Shared boundary value survives into ordinary output

Suppose the helper path assigns boundary word `0x2468ACE0`, the helper returns zero without changing it, adjustment completes, and the final comparison is nonzero.

If the ordinary insertion-like call leaves its output unchanged, the result word is `0x2468ACE0`, not the earlier traversal-established zero. The indicator is one.

### Helper mutation and partial output combine

Suppose the helper changes the shared word to `0x12345678` and returns zero. Adjustment and final comparison lead to the ordinary path without further changes to that word.

If F-e5fa1f12a468 replaces only its low-order byte with `0xAB`, the result word is `0x123456AB`, followed by indicator one. This example specifies a numeric low-order-byte replacement, not a target byte-order assumption.

### Helper-selected path uses separate storage

Suppose traversal ran, the helper changes the shared word, and then returns nonzero.

If F-e5fa1f12a468 leaves the early-output word unchanged, the result word is zero, provided no other mutation changed that separate word. The helper’s shared-word update is not copied into it.

### Selected candidate

Suppose the control flag is zero and the final traversal comparison was zero.

No helper or candidate adjustment occurs. If extraction completes and the final comparison returns zero, the retained node is written as the result with indicator zero. No insertion-like call occurs.

### Adjustment and comparison-dependent traversal

With a zero helper result, no marker special case, and stable nonnull candidate `+8`, adjustment descends that link and then `+0xC` links to their terminal node.

Separately, if a traversal comparison changes the link selected by its return value, traversal reads the changed link after the call. An entry-state-only traversal prediction is insufficient.

## Uncertainties

The following remain **documented_only**:

- Exact container, node, sentinel, and comparison-context types.
- Meaning of the marker and control bytes.
- Ordering and equivalence semantics of comparison results.
- The helper’s predicate and mutation guarantees.
- Extracted-value representation, lifetime, and ownership.
- Whether the insertion-like dependency inserts, allocates, rejects duplicates, balances, or reports failure.
- Whether it fully initializes its output word.
- Dependency behavior through retained aliases or globals.
- Thread safety, reentrancy, synchronization, and permitted concurrent mutation.
- Behavior under invalid structure invariants, inaccessible memory, or address wraparound.
- Concrete contents of output words on paths with no established initialization.

Documented initialization and storage identity constrain prior values; they do not resolve unknown dependency behavior.

## Evidence

This contract derives from static inspection of the supplied function representation and opaque dependency mapping.

Established evidence includes field offsets and widths, dependency argument order, zero/nonzero decisions, repeated link reads, result-write ordering, per-iteration initialization of both output words, and reuse of the boundary slot as ordinary output storage.

Tree ordering, predecessor selection, and insertion terminology are inferred intent rather than dependency guarantees.

No dependency implementations or runtime observations were available. No runtime tests have been performed.
