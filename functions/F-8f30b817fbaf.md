<!-- Trusted provenance: F-8f30b817fbaf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8f30b817fbaf

## Purpose

F-8f30b817fbaf conditionally performs a comparison-driven, gapped insertion rearrangement through dynamically dispatched operations on an object. After that rearrangement, or after bypassing it, the function writes a one-byte marker and traverses a linked structure, invoking a notification dependency on each visited node.

The rearrangement resembles Shell sort. This is **inferred intent**, not an unconditional sorting guarantee. The arithmetic helper, element-access dependency, comparison operation, and staging and placement operations have no supplied behavioral contracts.

The following are established by the supplied static evidence:

- An entry-time signed 32-bit bound controls rearrangement.
- An object query is called unconditionally.
- Rearrangement is entered when the query returns zero or the captured bound exceeds one.
- Negative comparison results cause preceding values to be fetched again and passed to a placement operation.
- The marker write precedes reading the notification-list head.
- Each node’s successor is read after notifying that node.
- Once the notification-list head has been read, this function makes no further accesses through the main object address.

## Interface

### Input and result

The function accepts one main-object address and returns no explicit value.

There is no null check, representation validation, bounds validation, or explicit error result. Each address, receiver, and argument must satisfy the requirements of the accesses and dependencies actually reached.

Validity requirements are access-specific. An object need not remain valid after its final use merely because a later dependency returns normally.

### Main-object representation

The supplied representation uses four-byte integer and address slots.

| Byte offset | Interpretation in this function |
|---|---|
| 0 | Address of the main object’s dispatch table |
| 4 | Signed 32-bit bound, captured before the initial query |
| 12 | Address of the first notification node, or zero |
| 28 | Four-byte argument used by staging, comparison, and final placement |
| 32 | One-byte marker written with value 1 |
| 36 | Address of a comparison receiver |

The marker write affects exactly one byte at offset 32. The function does not explicitly overwrite adjacent bytes.

The offset-28 value has no established type beyond its four-byte representation. It could designate storage, carry a handle, or have another dependency-defined meaning. Calling it a “scratch argument” describes its use, not ownership or allocation.

### Dynamic operations

Dispatch offsets are byte offsets within the relevant receiver’s dispatch table.

| Receiver | Dispatch offset | Established interface |
|---|---:|---|
| Main object | 100 | Receives the main object; returns an integer tested against zero |
| Main object | 24 | Receives the main object; returns a four-byte value retained during rearrangement |
| Main object | 164 | Receives the main object, a candidate position, and the current offset-28 value; result unused |
| Main object | 168 | Receives the main object, a destination position, and a four-byte source value; result unused |
| Comparison receiver | 36 | Receives that receiver, the current offset-28 value, the retained four-byte value, a lookup result, and the retained value again; returns a signed 32-bit comparison result |

No opaque function IDs are supplied for these dynamically selected targets. Their behavior remains identified by receiver and dispatch offset, without assigning invented dependency identities.

### Notification-node representation

For each nonzero notification-node address, the function reads a four-byte successor address at node offset 16 after calling F-5dd155b2ada8 on that node.

No other node fields are directly accessed by this function.

## Behavior

### 1. Capture the bound and query

The function first reads the signed 32-bit bound at main-object offset 4. Subsequent changes to that field do not replace the captured bound.

It then invokes the main object’s dispatch operation at offset 100. This query occurs for every bound, including zero and negative values.

Rearrangement is entered if either:

- the query result equals zero; or
- the captured bound is greater than one.

Consequently, a nonzero query result bypasses rearrangement only when the bound is at most one. The marker is not read when making this decision.

### 2. Retain a comparison argument

If rearrangement is entered, the main object’s dispatch operation at offset 24 is called once.

Its four-byte result is retained throughout the rearrangement and supplied twice to each comparator invocation. The function does not refresh it for each gap, candidate, or comparison.

Its meaning, ownership, and validity requirements are unknown. Retaining the representation does not establish that any referenced storage remains valid.

### 3. Select the initial gap

F-870c4fcf7295 is called with the captured bound and integer 9.

Starting at gap 1, the function repeatedly increases the gap to three times its previous value plus one while the gap is less than the helper result.

Under representable arithmetic:

- A helper result at most 1 selects initial gap 1.
- Larger results select the first member of 1, 4, 13, 40, and subsequent members that is at least the helper result.

The helper’s operation is not established. In particular, passing 9 does not prove division by 9.

Gap construction uses signed 32-bit values. No supplied evidence defines overflow behavior or guarantees that this construction terminates for every helper result.

### 4. Process each positive gap

A pass is performed for each positive gap.

The first candidate position equals the gap. A candidate is processed only if its position plus one is no greater than the captured bound. After processing a candidate, the next candidate position is one greater.

For ordinary nonoverflowing arithmetic, a pass therefore processes candidate positions from the gap through bound minus one. A gap at least as large as the bound produces no candidate operations.

For each candidate:

1. The main object’s offset-164 operation receives the candidate position and the then-current offset-28 value.
2. A backward-search position is initialized to candidate position plus one minus the gap.
3. Positive search positions are processed as described below.
4. On leaving that search, the offset-168 operation receives the final destination and the then-current offset-28 value.

The final placement call occurs even if the backward search makes no movement.

Describing the offset-164 call as “staging the candidate” is inferred intent. The function itself does not establish a local element copy or a stable scratch buffer.

### 5. Compare a preceding position

For each positive backward-search position, the following ordering is established:

1. The comparison-receiver address is read from main-object offset 36.
2. F-3b5d52fd8e4a is called with the main object and search position minus one.
3. The selected comparison receiver’s dispatch operation at offset 36 is invoked.

The comparator receives, in order:

- the selected comparison receiver;
- the current four-byte value at main-object offset 28;
- the retained four-byte argument;
- the lookup result;
- the retained argument again.

The comparison-receiver address is selected before the lookup. If that lookup changes main-object offset 36, the immediately following comparison still uses the previously selected receiver address.

The comparison target is resolved through that selected receiver after the lookup. A lookup that changes the selected receiver’s dispatch table can therefore affect the immediately following target.

The offset-28 comparison argument is read after the lookup; it is not an entry-time or candidate-time snapshot.

### 6. Move or stop

The comparator’s signed result determines the next action:

- A result of zero or greater stops the backward search.
- A negative result causes a second call to F-3b5d52fd8e4a with the same main object and preceding position.

For a negative result, the second lookup’s result is supplied to the main object’s offset-168 operation. Its destination is search position plus gap minus one.

The function does not reuse the value that was compared. A stateful lookup can therefore return one value for comparison and a different value for movement.

After this placement, the search position is reduced by the gap. Search continues only while the resulting position is positive.

When search ends, whether because a comparison was nonnegative or because the position became nonpositive, a final offset-168 call receives:

- destination: final search position plus gap minus one;
- source: the then-current offset-28 value.

Under ordinary positive-gap arithmetic, crossing below zero still produces a nonnegative final destination. Search need not terminate at exactly zero.

Equality stops local movement. This does not establish stability across different gap passes.

### 7. Advance the gap

After each completed pass, F-870c4fcf7295 is called with that pass’s gap and integer 3. Its result becomes the next gap.

Only positive results cause another pass. There is no separate progress guard.

A helper that repeatedly returns the same positive gap can cause indefinite repetition. A helper result that decreases to zero or a negative value ends rearrangement, assuming all other reached operations terminate.

### 8. Write the marker and traverse notifications

After rearrangement finishes, or immediately after its bypass, the function writes byte value 1 at main-object offset 32.

It then reads the notification-list head from main-object offset 12. This is the function’s final access through the main object address.

For each nonzero node address:

1. F-5dd155b2ada8 is called with that node and integer 1.
2. The successor address is read from offset 16 of that same node.
3. Traversal continues with the successor.

The successor is not cached before notification. A notification that changes the current node’s link can change the next node visited.

The main object’s list head is not reread. There is no cycle detection or visited-node tracking.

## Side effects

### Direct and dependency-mediated effects

The sole explicit data write by this function is the one-byte marker write. Rearrangement and notification effects are mediated by dependencies.

Each negative comparison produces this observable call sequence:

- lookup of a preceding position;
- comparison;
- second lookup of that position;
- placement using the second result.

Every processed candidate also produces a final placement after its backward search.

The marker write precedes notification, but the marker is not guaranteed to remain 1. Dependencies can have additional effects not described by the supplied evidence.

Main-object dispatch-table accesses are not replaced with one entry-time table snapshot. Dependency mutations can affect later dispatches.

### Ownership and validity boundaries

No allocation, ownership transfer, or release is explicitly performed by this function. No such restriction is established for its dependencies.

Required lifetimes follow subsequent actual uses:

- The main object must support each field read, dispatch access, and dependency use reached before and including the final list-head read.
- After that head read, this function no longer requires the main object to remain valid.
- A selected comparison receiver must support its later dispatch access and comparator invocation, even though a lookup intervenes.
- Each notified node must still permit reading its successor at offset 16 after notification returns.
- Any nonzero successor must support the later operations reached on that node.
- Referents represented by retained or fetched four-byte values must satisfy the unknown contracts of their consumers.

A notification may release or invalidate the main object without invalidating this function’s subsequent accesses, provided the traversed nodes remain independently valid and dependency-specific requirements are satisfied.

That conclusion does not extend automatically to nodes stored within the released object. If releasing the object also invalidates a current or future node, subsequent node access has no established valid behavior.

Likewise, a notification cannot be assumed to release the current node safely: this function reads that node’s successor after the notification returns.

### Errors and partial completion

There is no explicit rollback, recovery path, or error reporting.

Failure before the marker write may leave partial rearrangement effects. Failure during notification occurs after the marker write and may follow successful notifications of earlier nodes.

Behavior for invalid addresses, rejected positions, exceptions, nonlocal exits, and arithmetic overflow is not established.

## Dependencies

| Opaque function ID | Established use | Unresolved contract |
|---|---|---|
| F-870c4fcf7295 | Receives the captured bound and 9 for the initial threshold; receives each completed positive gap and 3 for gap advancement | Arithmetic meaning, rounding, valid ranges, side effects, termination guarantees |
| F-3b5d52fd8e4a | Receives the main object and a position; returns a four-byte value for comparison or placement | Element interpretation, bounds handling, purity, ownership, lifetime |
| F-5dd155b2ada8 | Receives each visited notification node and integer 1 | Meaning of 1, mutations, release behavior, reentrancy, failure behavior |

The dynamic dependencies are described by receiver and dispatch offset in the Interface section. Nothing establishes that any dependency preserves globals, object fields, dispatch tables, scratch contents, nodes, or external state.

## Acceptance examples

These are conditional acceptance requirements derived from static evidence, not observed executions.

### Bypass with a bound of one

Given a captured bound of 1 and a nonzero query result:

- The query is called.
- No offset-24 operation or gap-helper call occurs.
- No rearrangement staging, lookup, comparison, or placement occurs.
- The marker is written.
- The list head is read, then notification traversal begins if it is nonzero.

The same bypass applies to any captured bound at most one with a nonzero query result.

### Zero query with no candidates

Given a captured bound of 0, a zero query result, and helper results of zero for both reached helper calls:

- The offset-24 operation is called once.
- The helper receives 0 and 9.
- Initial gap 1 produces no candidates.
- The helper receives 1 and 3.
- Its zero result ends rearrangement.
- The marker write and list-head read follow.

Entering rearrangement does not imply that element operations occur.

### Two-element movement

Given a captured bound of 2 and helper behavior equivalent to truncating division for the reached positive arguments:

- Rearrangement is entered regardless of the query result.
- Initial gap 1 processes candidate position 1.
- Position 0 is looked up and compared.

For a nonnegative comparison, position 0 is not looked up again, and final placement targets position 1.

For a negative comparison, position 0 is looked up again and the second result is placed at position 1. Final placement then targets position 0.

### Conditional ordering

Additionally assume that staging saves the candidate in stable scratch storage, lookups return current elements, placements assign elements, and negative comparison means that the scratch element precedes the fetched element.

Under those assumptions, the two-element sequence 7 then 3 becomes 3 then 7.

Without those dependency contracts, this example establishes a conditional interpretation, not guaranteed sorting behavior.

### Stateful lookup

Suppose one preceding position returns value A for comparison, the comparator returns a negative result, and the second lookup returns value B.

The movement placement receives B, not A. The second lookup cannot be omitted as observationally redundant.

### Notification replaces its successor

Suppose the first node initially links to a second node, but its notification replaces that link with a third node.

The next successor read obtains the third node. The original second node is not visited through the replaced link.

### Notification releases the main object

Suppose the captured head points to an independently allocated node chain. During the first notification, the main object is released without invalidating those nodes.

This function subsequently accesses the nodes, not the released main object. Continued traversal therefore does not require the main object’s lifetime to extend beyond that release.

If the release also invalidates any subsequently accessed node, the example no longer satisfies the required validity conditions.

## Uncertainties

The following remain **documented_only**:

- Exact arithmetic semantics and side effects of F-870c4fcf7295.
- Semantic meanings of the object fields and dynamic operations.
- Element representation and scratch-storage behavior.
- Ownership or lifetime associated with four-byte values.
- Whether comparison supplies a consistent ordering relation.
- Dependency effects on globals, object fields, dispatch tables, links, or storage.
- Behavior under concurrent mutation, reentrancy, exceptions, or nonlocal exits.
- Behavior when arithmetic intermediates exceed signed 32-bit range.
- Termination with cyclic notification structures or nonprogressing gap results.
- Any additional lifetime obligations imposed by dependency implementations.

The control arithmetic is signed 32-bit, not unbounded mathematical arithmetic. This contract does not define overflow as wrapping. Examples assume representable intermediate results.

## Evidence

Established static evidence consists of the supplied function body and dependency-ID mapping: four-byte field accesses, dispatch offsets, call arguments, signed comparisons, loop progression, the one-byte marker write, and successor reads after notification.

The lifetime boundary for the main object follows from its last access: the list-head read after the marker write. Later traversal uses node addresses and does not access the main object again.

Shell-sort terminology, element terminology, and scratch-storage terminology express inferred intent. They require dependency contracts before becoming unconditional semantic guarantees.

No runtime tests have been performed. No dependency implementations or execution outcomes are asserted.
