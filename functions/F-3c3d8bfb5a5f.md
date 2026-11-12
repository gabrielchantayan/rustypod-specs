<!-- Trusted provenance: F-3c3d8bfb5a5f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3c3d8bfb5a5f

## Purpose

F-3c3d8bfb5a5f traverses an ordered linked sequence of records, maintains a working set of indexed entities, and requests directed relationships between entities. It counts occurrences from one enumeration dependency, activates entities supplied by another, consumes occurrence counts, and updates two 16-bit entity fields.

Established behavior includes:

- Two record-list traversals, with the list head read separately for each.
- Initial relationship requests between active indices, in both directions.
- Per-record activation before occurrence consumption.
- Membership removal when an occurrence count becomes exactly zero, unless protected.
- One-level alternate-object selection for field updates, but not for occurrence counting or membership.
- Metadata-sensitive release of temporary count storage.

An overlapping-lifetime or interference-relationship interpretation is plausible intent only. The meaning and persistence of relationship requests depend on F-a085c1170eb2.

## Interface

**Input:** One 32-bit address-like value identifying a root object.

**Output:** No explicit return value. Observable behavior consists of memory accesses, dependency calls, and dependency-defined effects.

Relevant layouts use byte offsets:

| Object | Offset | Width and role |
|---|---:|---|
| Root | +4 | 32-bit associated-state address |
| Root | +12 | 32-bit record-list head |
| Root | +28 | 32-bit initialization-associated address/value |
| Root | +32 | 32-bit protection/selection-bitset address |
| Root | +44 | 32-bit increment operand |
| Associated state | +4 | 32-bit address of the initial context cell |
| Associated state | +24 | 32-bit address of an entity-reference table |
| Associated state | +32 | 32-bit entity count |
| Record | +0 | 32-bit next-record address |
| Entity | +4 | 32-bit optional alternate-object address |
| Entity | +16 | 32-bit index |
| Entity | +34 | 16-bit field updated during occurrence consumption |
| Entity | +36 | 16-bit field conditionally updated during activation enumeration |

The working bitset has a 32-bit unsigned bound at offset zero, followed by 32-bit membership words. Index \(j\) addresses the word at byte offset \(4+4\lfloor j/32\rfloor\), using bit \(j\bmod32\).

The protection/selection bitset uses the same membership-word addressing. This function does not inspect its leading word.

A local enumeration area contains 64 entries, each a 32-bit entity reference. An enumeration dependency returns an end address; entries are consumed from the area’s beginning up to, but excluding, that address. There is no visible capacity or end-address validation.

Ordinary operation requires accessible, appropriately sized storage; valid indices; terminating traversals; valid enumeration results; and allocations large enough for subsequent accesses. Dependencies must leave every object still needed by the function accessible. These are preconditions, not enforced checks.

## Behavior

### Initial context and count-storage allocation

The function first reads and retains the associated-state address from root offset +4. Through that retained state’s +4 field, it obtains an address and reads a 32-bit context value into a local cell.

F-106b019354f2 receives:

1. The context value.
2. The 32-bit value reached by reading the root’s +28 field as an address and dereferencing it once.

Its result is retained as the working-bitset address.

**After F-106b019354f2 returns, the allocation count is read through the associated-state address retained before that call.** The root’s +4 field is not reloaded for this observation. Consequently, replacing the root’s associated-state address during F-106b019354f2 does not redirect this count read. Mutation of the retained state’s count field can still affect it.

F-d33f6d597e5d receives that observed 32-bit count shifted left by two. Under valid nonoverflowing conditions, this requests four bytes per count entry.

Only after F-d33f6d597e5d returns does the function reload the associated-state address through root +4 and read the initialization count from that state’s +32 field. It zeroes consecutive 32-bit entries until an incrementing counter, starting at zero, equals this separately observed count.

The allocation count and initialization count therefore have different pointer provenance and observation times. Neither equality nor sufficient allocation capacity is checked. The initialization termination test is equality, not a signed less-than comparison.

### Preliminary occurrence counting

The function reads the root’s current list head and traverses records through their +0 links.

For each record, F-93a6f85390ff receives the record and local enumeration area. Each returned entity contributes one increment to the temporary 32-bit count selected by its +16 index. Duplicates within a record and across records are counted separately.

The next-record link is read after that record’s returned entries have been consumed. A dependency’s mutation of the link can therefore alter the remainder of traversal.

### Working-set preparation and initial relationships

F-44c315c6ba20 receives the retained working-bitset address and the root’s current +28 field value. Unlike the earlier argument to F-106b019354f2, this second argument has no additional dereference.

The function then traverses pairs of indices in ascending order:

- The outer index starts at zero and is compared against the unsigned working bound minus one.
- An outer index’s membership is tested once before its inner traversal.
- Higher indices are examined in ascending order, with their membership tested individually.
- For each observed active pair, F-a085c1170eb2 is called first from the lower index’s table-selected entity to the higher index’s entity, then in reverse.

The associated-state address and entity-table address are reloaded through the root before each directed call. Both entity references for a call come from that call’s table lookup. The reverse call need not use the same table or references as the forward call.

The working bound is reread at loop conditions rather than snapshotted for the phase. Relationship calls may affect later observations. Clearing the outer index during an inner traversal does not cause its membership to be retested before each later pair.

A bound of zero is not handled as an empty domain: unsigned subtraction produces \(2^{32}-1\) for the outer limit. The resulting traversal can access storage far beyond an ordinary empty bitset. A positive bound with all membership bits clear avoids initial relationship requests without this underflow.

### Per-record activation and conditional field update

The function rereads the root’s list head to begin its second traversal.

For each record, F-bf3b72c7b3bc receives the record, enumeration area, and address of the local context cell. It can change the context used by subsequent relationship calls.

For each returned entity, in enumeration order, the function tests membership using the entity’s original index. If inactive, it scans indices from zero upward while they remain below the currently observed working bound. For each index observed active, it requests:

1. A relationship from the newly encountered entity’s table-selected reference to the active index’s table-selected reference.
2. The reverse relationship.

The table is reloaded before each directed call. The enumerated entity’s index is reread for relevant accesses; it is not fixed for the entire scan.

Only after the scan does the function set membership for the enumerated entity’s then-observed index. Later entries in the same record can consequently relate to entities activated by earlier entries. With stable inputs, an already-active duplicate does not repeat the activation scan.

After membership handling, the function selects the field-update target:

- A nonzero alternate reference different from the original entity selects that alternate object.
- A null or self-reference leaves the original entity selected.

This selection follows only one reference.

The function tests the protection/selection bit for the **selected object’s** index. If set, it increments that object’s 16-bit field at +36 using the low 16 bits of the root’s current +44 operand. This consideration occurs for every returned entry, including entries that were already active.

### Per-record occurrence consumption and unconditional field update

After the first enumeration is consumed, F-93a6f85390ff is called for the same record with the enumeration area and, on this occasion, the context-cell address.

For each returned entity:

- Its original index selects a temporary count, which is decremented and written back.
- If the new count is exactly zero, the original index’s protection bit is tested.
- If that bit is clear, the original index’s working membership is cleared.
- The same one-level alternate-object selection is performed.
- The selected object’s +34 field is incremented unconditionally.

The count write precedes the possible membership clear, and both precede the field update. Alternate-object selection does not redirect counting, protection against removal, or membership removal.

A decrement below zero does not satisfy the zero test. A protected zero-count entity is not removed by this rule. These statements describe this function’s actions, not guarantees against dependency or alias effects.

The preliminary and consumption enumerations are separate calls. Their results are not compared or required by visible checks to agree.

The next-record link is read after both enumerations and all their entries have been processed.

### Widths and arithmetic

Entity fields at +34 and +36 are read and stored as 16-bit quantities. Their stored result is addition modulo \(2^{16}\), using the increment operand’s low 16 bits. No saturation or carry into adjacent fields is visible.

Counts occupy 32-bit entries. Working bounds, membership words, and the initial bound-minus-one comparison use 32-bit unsigned operations. Bit positions use the index’s low five bits; bitset word selection uses the unsigned index shifted right by five.

The supplied representation does not establish a portable contract for overflowing signed count operations, allocation-size shifts, invalid address arithmetic, or unreachable enumeration end addresses. Ordinary-operation preconditions must exclude those cases rather than assign them guessed outcomes.

## Side effects

Direct writes affect temporary counts, working-bitset words, selected entity fields, and allocator-accounting storage. Dependencies may additionally modify globals, root fields, records, entities, tables, bitsets, or retained aliases. No general preservation guarantee is established.

There is no explicit release of caller-supplied records, entities, or tables. There is also no explicit release of the working-bitset result. This absence does not establish ownership or prove that dependencies cannot release or transfer those objects.

### Temporary-storage release

Cleanup is entered only for a nonzero retained count-storage address. A zero allocation result does not itself stop earlier initialization or count accesses.

For nonzero storage, the visible cleanup order is:

1. Call F-187ab9ba4eb6, then F-2fd8826557a4.
2. Read the allocator-state address from global storage. That address is retained for subsequent accounting and cleanup calls.
3. Inspect the 32-bit header eight bytes before the allocation. A zero header invokes nonreturning F-a817d8d58c10. Otherwise, another header read tests its least significant bit; a set bit invokes the same nonreturning dependency.
4. Call F-c0fc6d033caa with the retained allocator-state address.
5. Read the unsigned 16-bit selector four bytes before the allocation and form the first category-counter location.
6. Then read the allocation header and mask it with `0x3ffffffc` to obtain an accounted size.
7. Subtract that size from the first category counter, at allocator-state offset \(548+4\times\text{first selector}\).
8. Subtract the same retained size from the aggregate counter at +780.
9. Only now read the unsigned 16-bit selector two bytes before the allocation.
10. Subtract the retained size from the counter at \(224+4\times\text{second selector}\), then from the aggregate counter at +544.
11. Call F-09dcb55e8e0a. Decrement the 32-bit counter at offset \(784+4\times\text{returned index}\).
12. Decrement the 32-bit counter at +912.
13. Reread the allocation header, mask this later value with `0x3ffffffc`, and subtract that size from the counter at +8.
14. Call F-cc6c9b4a0f68 with the retained allocator-state address and header address.
15. Call F-e586bff64fb7 with the retained allocator-state address.

These are ordered observations, not a common metadata snapshot or an atomic transaction. In particular:

- The first selector is read before the accounted size.
- The second selector is read after the first two counter updates.
- The final header read occurs after further accounting and a dependency call.
- Disjointness between allocation metadata and accounting storage is not established.

Thus earlier accounting writes can affect later metadata observations if those locations alias. F-c0fc6d033caa can also change metadata after the initial validity checks; no repeated validity check is visible before size extraction.

No rollback, relationship undo, or normal error status is provided by this function.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-106b019354f2 | Receives initial context and indirectly loaded initialization value; returns working-bitset address. |
| F-d33f6d597e5d | Receives count-storage byte-size request; returns storage address. |
| F-93a6f85390ff | Returns the end of a record’s entity enumeration; preliminary call supplies two arguments, consumption call supplies three. |
| F-44c315c6ba20 | Receives working bitset and current root +28 value before initial pair processing. |
| F-a085c1170eb2 | Receives current local context and two table-selected entity references for a directed request. |
| F-bf3b72c7b3bc | Supplies activation enumeration and receives the context-cell address. |
| F-187ab9ba4eb6 | First dependency in nonnull count-storage cleanup. |
| F-2fd8826557a4 | Called before the allocator-state global is read. |
| F-a817d8d58c10 | Nonreturning invalid-header path. |
| F-c0fc6d033caa | Receives allocator state before selector reads and accounting updates. |
| F-09dcb55e8e0a | Returns an accounting-counter index. |
| F-cc6c9b4a0f68 | Receives allocator state and allocation-header address after accounting. |
| F-e586bff64fb7 | Final visible cleanup dependency. |

Authoritative prototypes, complete side effects, ownership transfers, synchronization, and dependency failure behavior are unavailable.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless specified otherwise, storage is valid, arithmetic stays within supported ranges, dependencies return normally, and relevant data remains stable.

1. **Retained allocation-state provenance.** The initially retained associated state has count 3. F-106b019354f2 changes root +4 to another state with count 5, without changing the retained state. The allocation request is 12 bytes, not 20. If the second state remains current after allocation, initialization attempts five count writes. The function does not reconcile this mismatch.

2. **Mutation of the retained state.** If F-106b019354f2 changes the initially retained state’s count to 4, that changed value governs the allocation request even if root +4 now points elsewhere.

3. **Initial pair order.** With bound 6 and only indices 1, 3, and 5 active, directed requests occur as 1→3, 3→1, 1→5, 5→1, 3→5, 5→3.

4. **Sequential activation.** With only index 1 active, activation entries 4 then 6 produce 4→1 and 1→4 before activating 4. Entry 6 then produces 6→1, 1→6, 6→4, and 4→6 before activation.

5. **Duplicate activation entry.** An inactive entity returned twice performs its activation scan once if its membership remains set. Its conditional +36 update is considered twice.

6. **Final occurrence.** A count of one becomes zero during consumption. With the original entity’s protection bit clear, membership is cleared before the selected target’s +34 update. With that bit set, this function leaves membership unchanged.

7. **Alternate-object distinction.** A distinct alternate object receives the +34 update. It receives the +36 update only if its own index is selected by the protection/selection bitset. Counting and removal still use the original entity.

8. **Field wrapping.** A selected 16-bit field containing `0xffff`, incremented by an operand whose low 16 bits are 2, stores `0x0001`.

9. **Ordered cleanup metadata.** The first selector is observed before the masked size. If the subsequent +780 counter write aliases and changes the second selector’s storage, the later second-selector read observes the changed storage, not a previously captured selector. Likewise, the final +8 subtraction uses the later header observation.

10. **Empty membership.** A positive bound with no membership bits set emits no initial relationships. Bound zero is not equivalent because the outer limit underflows.

## Uncertainties

- **documented_only — Dependency semantics:** Enumeration meanings, relationship effects, allocation guarantees, and bitset preparation remain unknown.
- **documented_only — Enumeration interface:** F-93a6f85390ff has different visible argument counts across its two uses; its authoritative interface is unavailable.
- **documented_only — Ownership and lifetime:** Allocation mechanisms, working-bitset ownership, retained aliases, and dependency release behavior are unresolved.
- **documented_only — Mutation and aliasing:** No guarantee excludes changes to globals, context, links, entity indices, tables, or bitsets. Metadata and accounting-storage disjointness is not established.
- **documented_only — Unsupported inputs:** Invalid indices, oversized enumerations, malformed storage, failed allocations, nontermination, and arithmetic outside valid ranges have no established recovery contract.
- **Inferred intent:** The ordered activation and removal rules resemble lifetime-based relationship construction. They do not establish deduplication, transactional symmetry, or absence of self-relationships when dependencies mutate state or table entries alias.

## Evidence

This contract rests solely on static examination of the supplied function representation and opaque dependency mapping.

Established evidence includes access widths and offsets, retained versus reloaded address provenance, loop conditions, dependency ordering, membership transitions, one-level target selection, field-update widths, and the ordered cleanup metadata reads and accounting writes.

No dependency implementations, authoritative type declarations, external ownership documentation, or runtime tests have been examined. Acceptance examples express conditional consequences of static evidence, not observed executions.
