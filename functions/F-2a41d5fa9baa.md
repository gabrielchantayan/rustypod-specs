<!-- Trusted provenance: F-2a41d5fa9baa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2a41d5fa9baa

## Purpose

F-2a41d5fa9baa processes each group reachable from a supplied root through three ordered phases:

1. Establish relationships involving items, selected descriptors, and, conditionally, successive items.
2. Invoke an item-level preparation dependency.
3. Repeatedly invoke two processing dependencies across the group until every returned 32-bit status is zero within one complete sweep.

The traversal, field accesses, call ordering, and termination condition are established by the supplied static evidence. Interpreting the relationships as graph edges or the repeated processing as dataflow convergence is inferred intent, not a confirmed dependency contract.

The operation has no explicit return value. Its externally visible effects arise through dependency calls and any state those dependencies modify. The supplied body contains no direct field assignments, allocation, or release.

## Interface

### Entry and result

The function accepts one value representing the address of a root structure. Although the supplied signature represents this value as a 32-bit integer, the body uses it as a pointer.

There is no explicit result or error return. Normal completion occurs after all reachable groups have completed their repeated-processing phase.

The root must be readable at byte offset 4 even when there are no groups. A null root is not treated as an empty input.

### Representation and widths

The supplied evidence uses a 32-bit address and word model:

- Addresses stored in traversed fields occupy four bytes.
- Ordinary loaded words and dependency status results occupy 32 bits.
- Descriptor selectors and markers occupy one byte each.
- Zero-valued pointer fields terminate traversals.
- Offsets below are byte offsets from the beginning of the named object.

No structure names, signedness conventions for opaque words, byte order, or broader platform ABI are established beyond these access widths.

### Observable layout

Only fields accessed by this function are specified.

| Object | Offset | Accessed interpretation |
|---|---:|---|
| Root | 4 | Pointer to first group |
| Group | 0 | Pointer to next group |
| Group | 8 | Pointer to first item |
| Item | 0 | Pointer to next item |
| Item | 12 | Pointer to first descriptor |
| Item | 20 | Opaque 32-bit input to F-532059e16ca0 |
| Item | 28 | Opaque 32-bit input to F-532059e16ca0 |
| Item | 32 | Opaque 32-bit input to both repeated-processing dependencies |
| Item | 40 | Pointer to first propagation record |
| Descriptor | 0 | Pointer to next descriptor |
| Descriptor | 8 | One-byte selector |
| Descriptor | 9 | One-byte marker |
| Descriptor | 12 | Pointer to a readable 32-bit relationship value, used for selected descriptors |
| Propagation record | 0 | Pointer to next propagation record |
| Propagation record | 4 | Pointer to an associated object |
| Associated object | 28 | Opaque 32-bit input to F-75921aa89672 |

Fields with opaque values are passed onward without local interpretation. Their apparent pointer-like or handle-like meanings are not established.

### Preconditions

For defined, terminating execution:

- Every object and field actually reached must remain readable when accessed.
- Descriptor offset 12 must identify a readable four-byte value whenever the selector is 7 or 8.
- Every reached propagation record must identify an associated object readable at offset 28.
- Dependency calls must return normally for the caller to continue.
- Traversal links and dependency behavior must permit the traversals and repeated sweeps to terminate.

These are operational preconditions, not validations performed by the function. Cycles, invalid addresses, concurrent mutation, and dependency-induced invalidation are not detected.

No disjointness requirement between structures is established. Aliasing can therefore affect subsequent observations if dependencies modify shared storage.

## Behavior

### Group traversal and phase boundaries

The function reads the root’s group pointer once to begin traversal. It does not reload that root field between groups.

For each current group, it completes all three phases before advancing through that group’s offset-0 link. The next-group link is read after the current group’s repeated-processing phase finishes. A dependency mutation of that link may consequently affect which group is visited next.

Each phase starts by reading the current group’s item-head field again. Each repeated-processing sweep also rereads that field. The item population is therefore not captured as one immutable list at entry.

### Phase 1: descriptor relationships and conditional successor relationships

For each item reached in this phase, the function initializes a local marker state to false and traverses the descriptors starting at item offset 12.

For a descriptor whose selector byte is 7 or 8:

1. Read the pointer at descriptor offset 12 and load the 32-bit value it addresses.
2. Call F-8e7629ca9d99 with the current item and that loaded value, in that argument order.
3. After that call returns, call F-470ab68d515f with the same loaded value and the current item, in that argument order.

The relationship value is loaded once for the pair. A change to its backing storage during the first call does not cause the second call to receive a freshly loaded value.

Other selector values cause neither of these calls for that descriptor.

After any selector-triggered calls have returned, descriptor traversal proceeds in this observable order:

1. Read the current descriptor’s next pointer.
2. Read the current descriptor’s marker byte at offset 9.
3. Replace the local marker state with whether that byte equals `0x25`, the percent-sign character.
4. Continue at the next pointer already read.

The marker is therefore read after the relationship calls and after the next pointer. It belongs to the descriptor just processed, not its successor. Dependencies may affect that marker or next pointer before these reads.

The marker state is overwritten on every descriptor. It does not record whether any descriptor in the list had the marker. For an unmodified finite list, only the last descriptor’s marker determines the final state. With no descriptors, the state remains false.

Once descriptor traversal ends, the function considers a successor-item relationship:

- If the final marker state is true, it does not inspect the item’s successor for this conditional relationship.
- Otherwise, it reads the current item’s offset-0 link and tests whether it is nonzero.
- If that test succeeds, it calls F-8e7629ca9d99 with only the current item as an explicit argument.
- It then reads the item’s offset-0 link again and calls F-470ab68d515f with that newly read value followed by the current item.

The successor value used for the second call is not the value retained from the initial nonzero test. If the first dependency changes the link to zero, the second dependency still executes and receives zero as its first argument.

After the conditional relationship handling, traversal reads the current item’s next link again to select the next item. This is another independent read; either dependency may have affected it.

The one-argument invocation of F-8e7629ca9d99 differs from its two-argument invocation for selected descriptors. The static evidence establishes those two call forms but does not establish a compatible prototype or the semantics of an omitted argument.

### Phase 2: item-level preparation

After phase 1 finishes for the group, the function rereads the group’s item-head field.

For every item reached in this new traversal, it invokes F-7d22e5885a46 with the item as its sole explicit argument. The next-item link is read after that call returns.

Thus, all phase-1 processing for a group precedes its first preparation call. Preparation is not interleaved with descriptor processing on an item-by-item basis.

No return value from this dependency contributes to the repeated-processing termination condition.

### Phase 3: repeated processing

The function performs at least one complete sweep for each reached group, even if the group’s item list is empty.

At the start of each sweep, it sets a local 32-bit accumulator to zero and rereads the group’s item-head field.

For each item reached during that sweep:

1. Traverse propagation records starting from item offset 40.
2. For each reached record, invoke F-75921aa89672 with:
   - the current value at item offset 32;
   - the 32-bit value at offset 28 of the associated object identified by the record’s offset-4 pointer.
3. Bitwise-OR each returned 32-bit result into the sweep accumulator.
4. After all propagation records for the item have been processed, invoke F-532059e16ca0 with the current item fields at offsets 28, 32, and 20, in that argument order.
5. Bitwise-OR that returned 32-bit result into the same accumulator.
6. Read the item’s next link to continue the sweep.

Propagation-record next links are read after the corresponding dependency call. Item fields are not captured once for the entire sweep: item offset 32 is read for each propagation call and again for the later F-532059e16ca0 call.

Argument positions are established. The relative evaluation order of separate field loads used to assemble a single call is not established by the supplied representation.

A nonzero result does not short-circuit a sweep. Remaining records and items are still processed.

At the end of the sweep:

- Accumulator zero ends repeated processing for the current group.
- Accumulator nonzero starts another complete sweep, with a newly zeroed accumulator.

This is bitwise accumulation, not addition. Results cannot cancel each other, and there is no arithmetic overflow from combining them. Any set bit in any result keeps the final accumulator nonzero for that sweep.

The function does not independently compare state between sweeps. A dependency that changes state but returns zero does not force another sweep. Conversely, a dependency that continually returns nonzero can prevent termination even if its visible state stops changing.

## Side effects

The caller itself performs reads, dependency calls, and local state updates. No direct writes to root, group, item, descriptor, propagation-record, or associated-object fields are present in the supplied body.

Dependency side effects are nevertheless observable through later reads. In particular:

- Descriptor relationship calls precede reads of that descriptor’s next link and marker.
- The conditional one-argument call precedes a fresh successor read.
- Preparation precedes all repeated processing for the group.
- Propagation calls precede the per-item F-532059e16ca0 call.
- Traversal links are generally read after the dependency calls associated with the current node.

The function does not establish ownership transfer, retention, allocation, or deallocation rules for any dependency. Its own lack of explicit release does not prove that dependencies leave storage allocated.

Memory must remain usable through the last access actually performed. For example, a selected descriptor is still read after both relationship calls; freeing or invalidating it during either call would conflict with those later accesses.

There is no local rollback, error translation, retry bound, cycle detector, or cleanup path. A dependency failure or invalid memory access has no recovery behavior established here.

## Dependencies

| Opaque function ID | Observed use | Unestablished contract |
|---|---|---|
| F-8e7629ca9d99 | Called with an item and a loaded relationship value for selector 7 or 8; also called with only an item before a conditional successor relationship | Meaning of either call form, prototype compatibility, mutation, ownership, errors |
| F-470ab68d515f | Called after F-8e7629ca9d99, with relationship value or freshly read successor first and current item second | Whether this records, updates, removes, or otherwise acts on a relationship |
| F-7d22e5885a46 | Called once per item reached during the preparation traversal | Preparation semantics, mutations, ownership, failure behavior |
| F-75921aa89672 | Called for each reached propagation record on every sweep; its 32-bit result contributes to the accumulator | Meaning of input words and individual result bits; mutation behavior |
| F-532059e16ca0 | Called after an item’s propagation records on every sweep, with three item fields; its 32-bit result contributes to the accumulator | Meaning of fields and individual result bits; mutation behavior |

No dependency is assumed to preserve globals, list links, object fields, or allocation lifetimes. The caller’s behavior must be understood against whatever mutations actually occur.

## Acceptance examples

These examples are conditional on valid readable storage, normally returning dependencies, and the stated absence of mutations. They are static contract examples, not executed tests.

### Empty root list

If the root’s offset-4 field is zero, the function returns without invoking any dependency. The root field is still read.

### Group with no items

If a reached group’s item-head field remains zero, all three phases have empty item traversals. The repeated-processing phase performs one empty sweep whose accumulator remains zero, then traversal proceeds to the next group.

### Selected descriptor and percent marker

Suppose an item has one descriptor, its selector is 8, its relationship value is `R`, and its marker is `0x25`.

The function calls F-8e7629ca9d99 with the item and `R`, then F-470ab68d515f with `R` and the item. The marker suppresses the conditional successor relationship even if the item has a successor.

Selector 7 has the same established behavior.

### Only the final marker controls suppression

Suppose an item has two descriptors and the dependencies do not change them.

- First marker `0x25`, second marker different: successor handling is eligible.
- First marker different, second marker `0x25`: successor handling is suppressed.

Selector values do not alter this marker rule.

### No descriptors but a successor

If an item has no descriptors and its successor is nonzero, its marker state remains false. The function makes the one-argument F-8e7629ca9d99 call, then calls F-470ab68d515f with the freshly read successor and the current item.

If that first call changes the successor from `A` to `B`, the second call receives `B`. If it changes the successor to zero, the second call receives zero rather than being skipped.

### Whole-group phase ordering

With two unchanged items, both complete phase-1 processing before either receives F-7d22e5885a46. Both preparation calls finish before any F-75921aa89672 or F-532059e16ca0 call begins.

### Nonzero status requires another full sweep

If one call in a sweep returns `0x80000000` and every other status is zero, the function still finishes the sweep and then repeats it. If every status in the next sweep is zero, repeated processing ends.

### Items without propagation records

An item with no propagation records still receives F-532059e16ca0 on every sweep. Its return value alone can require another sweep.

## Uncertainties

The following remain **documented_only**:

- Business meaning of groups, items, descriptors, relationships, preparation, and propagated state.
- Exact dependency prototypes, especially the differing F-8e7629ca9d99 call forms.
- Dependency ownership rules, global effects, allocation behavior, and error behavior.
- Meanings of opaque input words and individual status bits.
- Whether list mutation is intended, prohibited by a broader contract, or merely possible.
- Guarantees of acyclic lists or convergence.
- Thread safety and behavior under concurrent or reentrant mutation.
- Relative order of field loads within a single dependency’s argument preparation.
- Target byte order and representation details not fixed by the observed access widths.

A convergence-oriented graph-processing interpretation is plausible but remains inferred intent. It must not be used to assume monotonic updates, bounded iterations, or preservation of list structure.

## Evidence

Established behavior is derived solely from the supplied static function body and the provided mapping to opaque dependency IDs.

That evidence supports the field offsets, access widths, traversal structure, dependency argument positions, phase ordering, post-call reads, marker replacement rule, and 32-bit bitwise termination accumulator described above.

Dependency implementations and authoritative type definitions were not provided. No runtime tests have been performed. Acceptance examples express conditional consequences of the static evidence, not observed execution results.
