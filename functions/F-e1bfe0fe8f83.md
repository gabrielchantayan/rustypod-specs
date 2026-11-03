<!-- Trusted provenance: F-e1bfe0fe8f83; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e1bfe0fe8f83

## Purpose

F-e1bfe0fe8f83 traverses a linked source range, stopping at an exclusive endpoint. For each visited source element, it obtains a destination-node address from F-92ed9ef777de, conditionally initializes three 32-bit words, modifies linkage around a captured anchor, and increments a destination-associated 32-bit counter.

The established operation is partial field initialization followed by linkage modification, not a complete source-object copy. Two initialized words come from the current source element; the third comes from external storage.

The inferred intent is insertion before an anchor in a doubly linked structure. That interpretation requires valid, coherent storage and preservation of relevant state between accesses. The function itself does not establish those conditions, allocate storage directly, or define ownership.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Arguments

The function receives four address-valued arguments:

1. **Destination-state address.** Passed unchanged to F-92ed9ef777de on every iteration. The function increments a 32-bit field at byte offset 20.
2. **Anchor-reference address.** Identifies a 32-bit word containing an anchor address. The function reads this word separately at the start of each executed iteration.
3. **First-source address.** Identifies the initial source element.
4. **Exclusive-endpoint address.** Traversal stops when the current source address equals this value. The endpoint’s contents are not inspected merely to decide termination.

The supplied representation uses 32-bit words and 32-bit address encodings. Wider-pointer behavior is not established.

### Accessed storage

Offsets are in bytes. These are accessed regions, not complete object definitions.

| Storage role | Offset | Width | Direct use |
|---|---:|---:|---|
| Destination state | 20 | 32 bits | Read and increment |
| Anchor reference | 0 | 32 bits | Read anchor address |
| Current source | 0 | 32 bits | Read next-source address |
| Current source | 12 | 32 bits | Read first payload word |
| Current source | 16 | 32 bits | Read second payload word |
| Returned node | 0 | 32 bits | Write forward link |
| Returned node | 4 | 32 bits | Write backward link |
| Returned node | 8 | 32 bits | Conditionally write external datum |
| Returned node | 12 | 32 bits | Conditionally write first payload word |
| Returned node | 16 | 32 bits | Conditionally write second payload word |
| Captured anchor | 4 | 32 bits | Read twice, then overwrite |
| Selected predecessor | 0 | 32 bits | Write forward link |

The external datum is another 32-bit storage location. Its address and semantic meaning are not part of the established public interface.

No explicit overlap check appears. Different roles can identify the same bytes. In particular, existing storage roles can overlap even when every returned node is fresh and separate. Such overlap can change traversal, counter values, or final links.

### Return value

The normal result is a 64-bit composition of two 32-bit address encodings.

For a nonempty range that completes normally:

- Upper 32 bits: the anchor captured at the beginning of the final iteration.
- Lower 32 bits: the node address returned by the final F-92ed9ef777de call.

The upper component is not a final reread of the anchor-reference word.

For an empty range:

- Upper 32 bits: the anchor-reference argument itself.
- Lower 32 bits: the destination-state argument itself.

These rules specify component placement, not an in-memory byte order, a structure-return convention, or ownership transfer.

## Behavior

### Entry, termination, and access requirements

Traversal begins with the first-source address. The function compares the current source address with the exclusive endpoint before every iteration.

If they are equal at entry, the function performs no direct memory reads or writes through the arguments and makes no dependency call. It returns the empty-range composition described above. Readable argument storage is therefore unnecessary for its own direct operations on that path.

For a nonempty range, every reached access must be valid for the intended execution environment if normal completion is expected. There is no explicit null check, cycle detection, endpoint validation, or iteration bound.

Termination depends on the next-source values actually read during execution eventually reaching the endpoint. An initially well-formed source chain is insufficient if intervening effects change those values. The endpoint need not be readable when it is encountered only as a terminating address.

### Per-iteration access order

For each source address unequal to the endpoint, the supplied representation establishes this order:

1. **Capture the anchor.** Read the word at the anchor-reference address and retain its address value for this iteration.

2. **Obtain a node.** Call F-92ed9ef777de with the destination-state address and a second argument of zero. Use the result as the destination-node address.

3. **Conditionally initialize three words.** Test whether the address eight bytes beyond the returned node address is nonzero. If it is:
   - Read the external datum and write it at node offset 8.
   - Read the current source word at offset 12 and write it at node offset 12.
   - Read the current source word at offset 16 and write it at node offset 16.

4. **Set the node’s forward link.** Write the captured anchor address at node offset 0.

5. **Set the node’s backward link.** Read the captured anchor’s offset-4 word and write that value at node offset 4.

6. **Update a predecessor’s forward link.** Read the captured anchor’s offset-4 word again. Use this second value as an address and write the returned node address at its offset 0.

7. **Update the anchor’s backward link.** Write the returned node address at captured-anchor offset 4.

8. **Increment the counter.** Read the destination-state word at offset 20, add one, and write the result back.

9. **Advance traversal.** Read the current source word at offset 0 and use it as the next current source address.

The next endpoint comparison follows that final read. This is the access ordering represented by the supplied source; it is not an atomicity or hardware memory-ordering guarantee.

### Reads are not snapshots

The anchor address is captured before the dependency call. Changing the anchor-reference word during that call does not change the captured address for the current iteration. The dependency can nevertheless change the captured anchor’s contents before later reads.

Payload values are read after the dependency returns. Each payload read also follows earlier initialization writes. Aliasing can therefore make a copied payload differ from its value at iteration entry.

The anchor’s backward link is read twice. The write to the returned node’s backward link lies between those reads. With overlap, the second read can differ from the first. Earlier initialization or forward-link writes can also change the anchor’s backward link before either read.

Traversal advancement is late: the source’s next address is not captured before the call, initialization, linkage writes, or counter increment. Any of those effects can change the next address that traversal follows.

The next iteration rereads the anchor-reference word. A value placed there by the dependency is used only if it survives all intervening effects.

### Conditions for the insertion interpretation

Under suitable storage and preservation conditions, the linkage writes insert the returned node immediately before the captured anchor.

Those conditions must cover **all intervening writes**, not merely dependency behavior or returned-node distinctness. Sufficient conditions include:

- Initially coherent anchor and predecessor links.
- Valid returned-node storage that does not overlap relevant existing storage.
- Preservation of the anchor’s original predecessor through both backward-link reads.
- No unintended overwriting of established links by initialization, counter updates, or other effects.
- Preservation of source payloads and next-source words until their required reads.
- Preservation of the counter except for intended increments.

For repeated insertion, later calls and writes must also preserve earlier inserted links.

Overlap among existing roles can defeat these conditions. For example, if destination-state offset 20 occupies the captured anchor’s backward-link word, the anchor update first writes the node address there. The subsequent increment reads that address as the counter and changes it. Neither the ordinary final linkage nor an entry-counter-plus-one result follows, even if the returned node is separate and the dependency preserves all state.

### Initialization guard

The guard tests the address at returned-node offset 8, not the returned address itself. A zero returned address therefore does not, under the represented offset calculation, skip initialization. There is no established null-result recovery.

Under a hypothetical 32-bit modular address calculation, a returned base of `0xfffffff8` makes the tested address zero. That interpretation would skip all three initialization assignments, but not the subsequent linkage operations, counter increment, or traversal read.

This is an arithmetic observation, not a supported exceptional-address protocol. Whether such address formation and later accesses are meaningful remains documented_only.

### Counter arithmetic

The counter is represented as a 32-bit integer field. Its read occurs after the dependency call and all linkage writes.

For values whose increment is representable in the supplied signed-integer interpretation, the stored result is the value read plus one. No explicit overflow branch, saturation, or wider accumulation appears. Defined modular wraparound is not established merely by the field width.

An entry value plus the number of iterations is a conditional result. It requires representable increments and preservation of the counter by every other effect, including direct writes through overlapping storage.

## Side effects

### Direct writes

Each executed iteration attempts:

- Three initialization writes at returned-node offsets 8, 12, and 16, if the guard succeeds.
- Two link writes at returned-node offsets 0 and 4.
- One write at offset 0 of the predecessor selected by the second backward-link read.
- One write at captured-anchor offset 4.
- One counter write at destination-state offset 20.

These roles are not guaranteed to identify distinct locations.

There are no explicit writes to source fields by role. Source fields, the anchor reference, or the external datum can nevertheless be modified through overlap with write destinations.

No synchronization or atomicity mechanism is shown. Concurrent conflicting access is not given a safe contract.

### Dependency effects and ownership

The function makes no explicit deallocation call. It does not establish allocation ownership, payload ownership, reference-counting duties, or release obligations.

F-92ed9ef777de may have additional effects. Nothing supplied establishes that it preserves globals, source storage, counters, references, existing links, or previously returned nodes. Nor is it established that the dependency returns fresh, valid, distinct, or initialized storage.

Application-level ownership transfer through linkage is possible inferred intent, not an established obligation.

### Errors and partial completion

There is no local rollback, diagnostic path, explicit error status, or validated allocation-failure path.

The function uses the dependency result without an established validity check. Invalid accesses or abnormal dependency termination can interrupt processing after partial effects. Initialization precedes linkage modification; linkage modification precedes the counter increment; source advancement follows the increment.

The packed normal return is not established as a success-or-error encoding.

## Dependencies

### F-92ed9ef777de

Observed interaction:

- Called once per executed iteration.
- Receives the destination-state address unchanged and a second argument equal to zero.
- Returns a value interpreted as a node address.
- Executes after anchor capture and before external-datum and source-payload reads.

Its allocation strategy, failure behavior, ownership rules, validity guarantees, initialization, and state preservation remain unknown.

### External storage

A 32-bit external word supplies the value written at returned-node offset 8. It is read separately during each iteration whose initialization guard succeeds.

The function does not snapshot that value for the entire traversal. Its meaning, mutability rules, and relationship to dependency effects are unknown.

## Acceptance examples

These are conditional predictions from static evidence, not executed tests.

### Empty range

**Given:** First-source and exclusive-endpoint addresses are equal.

**Expected:** No dependency call or direct memory access through the arguments. The upper return component is the anchor-reference argument; the lower component is the destination-state argument.

### One ordinary insertion

**Given:**

- One source element leads to the endpoint.
- All required accesses are valid and the initialization guard succeeds.
- The source object, destination-state object, anchor-reference word, external word, initial anchor, initial predecessor, and returned node occupy mutually nonoverlapping storage.
- The initial anchor and predecessor have coherent links.
- The dependency preserves source values, the selected reference, existing links, and the counter.
- The counter increment is representable.

**Expected:** The returned node receives the external datum and two source payload words. Its forward link identifies the anchor; its backward link identifies the former predecessor. The predecessor points forward to the node, and the anchor points backward to it. The counter becomes its entry value plus one. The result contains the captured anchor above the returned node address.

The nonoverlap conditions exclude counter/link overlap and other direct-write interference, not just overlap involving the returned node.

### Two insertions before a stable anchor

**Given:**

- Two source elements lead to the endpoint.
- Required accesses are valid and both initialization guards succeed.
- Both returned nodes are distinct.
- Both source objects, destination-state storage, reference word, external word, initial anchor, initial predecessor, and both returned-node storage regions are mutually nonoverlapping.
- Initial links are coherent and the reference continues to select the same anchor.
- Each dependency call preserves traversal, counter, anchor, predecessor, and previously inserted linkage state.
- Both counter increments are representable.

**Expected:** The final local chain is former predecessor, first returned node, second returned node, anchor. Backward links reflect the same order. The counter becomes its entry value plus two. The result contains the anchor and second returned node.

Without protection against both dependency effects and unintended direct-write overlap, neither the final-chain prediction nor the counter total follows.

### Dependency changes the anchor reference

**Given:** The dependency changes the anchor-reference word to another address, while the previously captured anchor remains valid.

**Expected:** The current iteration still uses the previously captured anchor. A later iteration selects the new address only if it survives until that iteration’s reference read.

### Insertion overwrites a dependency-written reference

**Given:**

- The anchor-reference word occupies the original predecessor’s forward-link location and initially identifies the captured anchor.
- The dependency writes a different anchor address into that word.
- The captured anchor’s backward link retains the original predecessor through **both** subsequent backward-link reads, including across initialization and returned-node link writes.
- Required accesses are valid.
- The predecessor-forward-link write and all remaining effects leave the reference containing the returned node address until the next iteration’s reference read.

**Expected:** The predecessor-forward-link write overwrites the dependency-written reference with the returned node address. If another iteration occurs, it selects that node as its anchor.

Dependency preservation alone is insufficient. If initialization overlaps and changes the anchor’s backward link, the later predecessor write may target different storage and need not overwrite the reference.

### Payload or traversal changes

**Given:** The dependency changes a source payload, initialization proceeds, and that value survives until its individual read.

**Expected:** The changed value is copied.

**Given:** Any intervening effect changes the source’s offset-0 word, and the final read is valid.

**Expected:** Traversal follows the value present at that final read. A newly introduced endpoint terminates traversal; a newly introduced cycle can prevent termination.

## Uncertainties

The following remain documented_only:

- Complete types, object sizes, alignment requirements, and valid-caller aliasing restrictions.
- Whether an anchor is a sentinel or ordinary node.
- The external datum’s meaning and mutability rules.
- Dependency internals, allocation, ownership, release, and failure behavior.
- Exceptional-address validity and arithmetic semantics.
- Counter behavior outside representable signed increments.
- Synchronization and concurrent-access requirements.
- ABI details beyond the represented return-component composition.
- Application-level ownership transfer and intended container abstraction.

Valid access, preserved relevant state, representable increments, and eventual endpoint reachability are sufficient ordinary-use requirements, not checks performed by this function.

## Evidence

**Established static evidence:** Exclusive-end comparison; per-iteration anchor capture; one F-92ed9ef777de call with a zero second argument; guarded initialization of three 32-bit words; two separate anchor-backward-link reads; four linkage writes; a 32-bit counter increment; late next-source reading; and a 64-bit return composed from two address-valued components.

**Inferred intent:** Repeated partial node initialization and insertion before a doubly linked anchor. Coherent final links and source-order insertion require the preservation and noninterference conditions stated above.

**Unknown:** Runtime storage validity, dependency guarantees, complete object identities, ownership, overflow semantics outside the representable range, and application-level invariants.

No runtime execution, memory observation, or test results support this contract.
