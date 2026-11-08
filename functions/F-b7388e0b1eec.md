<!-- Trusted provenance: F-b7388e0b1eec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b7388e0b1eec

## Purpose

F-b7388e0b1eec performs three ordered traversals of a nested linked structure:

1. Clear bit 16 in each encountered item’s 32-bit state word.
2. Call F-a2ca640ca643 for each encountered item whose classification byte satisfies a fixed selection predicate.
3. Unlink each item encountered during final filtering whose bit 16 is clear when inspected.

These phases operate on the links visible when they execute, not on a fixed entry-time collection. In particular, dependency-driven changes can make previously visited items unreachable before filtering. Such items are not guaranteed to be unlinked from their original containing lists.

The sequence suggests a reset, mark, and filter operation. That interpretation is inferred intent: the visible function clears and tests bit 16 but never sets it. The dependency’s actual behavior is unknown.

The function returns no value. Its explicit observable effects are state-word writes, dependency calls, and item-list link replacements. No allocation or release operation appears in its body.

## Interface

### Input and representation

The sole argument represents a root address in the supplied 32-bit representation. It is used for address calculation, not as a count or identifier.

The accessed layout is:

| Object | Byte offset | Accessed field |
|---|---:|---|
| Root | +4 | First outer-node address |
| Outer node | +0 | Next outer-node address |
| Outer node | +8 | First middle-node address |
| Middle node | +0 | Next middle-node address |
| Middle node | +12 | First item address |
| Item | +0 | Next item address |
| Item | +8 | 32-bit state word |
| Item | +8 | Separately accessed unsigned classification byte |

Links occupy four-byte words. A zero link terminates the corresponding list. These fields describe accessed storage, not complete object definitions or allocation sizes.

The classification read accesses exactly one byte at item offset +8 and widens it without sign extension. Its value is therefore between 0 and 255 inclusive. The state operations access the entire four-byte word starting at the same address.

Byte order is not established. Consequently, the contract does not identify which numeric byte of the state word is the classification byte. The overlapping accesses must not be treated as independent storage.

### Output and completion

There is no returned status or value.

On ordinary completion, the function has:

- Written a bit-16-cleared state word for every first-phase item encounter.
- Made one dependency call for every qualifying second-phase encounter.
- Bypassed every item encountered with bit 16 clear during final filtering.

These are encounter-based guarantees. They do not establish that every item reachable on entry remains reachable for all phases, or that a mark observed during filtering remains unchanged afterward.

### Preconditions

Every address actually dereferenced must designate accessible storage sufficient for that access, with the alignment required by the execution environment.

The root must be valid even when its outer-list head is zero. A zero root argument is not recognized as an empty structure; the function attempts an access at root offset +4.

Ordinary finite traversal requires links eventually to reach zero. There is no cycle detection, structural validation, bounds checking, or explicit recovery from inaccessible storage.

Stable-list conclusions additionally require suitable topology assumptions. Shared tails, overlapping field storage, dependency-driven mutation, and concurrent mutation can change later reads and results. No synchronization or atomic read-modify-write operation is established.

Address calculations use the supplied representation without visible overflow checks. No safe behavior is established when an offset calculation wraps or leaves valid storage.

## Behavior

### Phase 1: clear bit 16

The function reads the root’s outer-list head. For each outer node, it reads the middle-list head at offset +8. For each middle node, it reads the item-list head at offset +12.

For each item encounter, the observable memory-operation order is:

1. Read the 32-bit state word at item offset +8.
2. Write that word back with bit 16 cleared.
3. Read the item’s next link at offset +0 to advance.

The clearing mask is `0xFFFEFFFF`; the cleared bit has mask `0x00010000`. Bit numbering is zero-based. Every other bit of the word just read is preserved. The operation is bitwise, not a signed comparison or arithmetic adjustment, and introduces no arithmetic-overflow case.

The state write occurs even if bit 16 was already clear.

After an item list ends, the function reads the current middle node’s next link. After a middle-node list ends, it reads the current outer node’s next link.

The entire clearing phase completes before the first dependency call. With valid, disjoint storage, stable finite topology, and no external mutation, all items reachable through the documented nesting have bit 16 clear at the end of this phase. Shared items can receive multiple writes.

### Phase 2: select and call

The function rereads the root’s outer-list head and traverses the nested lists again. It does not use a saved list of first-phase item addresses.

For each item encounter, it reads the unsigned classification byte at offset +8. The established numeric selection values are:

- `0x05` — 5
- `0x07` — 7
- `0x0A` — 10
- `0x0B` — 11
- `0x17` — 23

Two further comparisons use symbolic address values whose numeric values are not established. They are designated **selector A** and **selector B** here. These labels are descriptive, not additional arguments.

Selection compares the widened byte value with each comparison value. The symbolic comparisons do not read memory at the selector addresses. An unresolved selector can match only if its numeric value equals the classification value, and hence lies between 0 and 255 inclusive. For example, a selector equal to zero permits classification zero; a selector greater than 255 cannot match any classification byte.

If any comparison matches, F-a2ca640ca643 receives the current item’s address. It does not receive the classification value, the state word, or the parent’s head-field address.

A qualifying encounter produces exactly one call, even if several comparison values are equal. A nonqualifying encounter produces no call. The visible repeated evaluation of the same predicate introduces no additional classification read or dependency call.

Calls follow the traversal order: item order within a middle node, then middle-node order within an outer node, then outer-node order.

The current item’s next link is read **after** the dependency returns. Its successor is not saved before the call. Therefore:

- Changing that link can redirect subsequent item traversal.
- Invalidating the current item can make the subsequent link read unsafe.
- Creating a cycle can prevent completion.
- Changing only the parent’s item-list head does not itself replace the already-current item.

The function also reads middle-node and outer-node successors when their respective inner traversals finish. Dependency changes to those fields can affect subsequent traversal. Heads for structures not yet entered are read when those structures are reached.

A change to the root’s outer-list head does not itself restart the ongoing second-phase traversal. It does affect the head read that begins the final phase.

No preservation of global state, link topology, state words, or object lifetimes is established for the dependency.

### Phase 3: filter encountered items

The function again rereads the root’s outer-list head. For each reached middle node, filtering starts at that node’s item-list head field.

Filtering maintains an **incoming link location**: the storage containing the address of the next item to inspect. Initially this is the middle node’s head field. After retaining an item, it becomes that item’s next-link field.

Each iteration reads an item address from the incoming link location. A zero address ends that item-list traversal. Otherwise, the function reads the item’s 32-bit state word and tests bit 16.

If bit 16 is clear:

- Read the rejected item’s next-link value.
- Write that value into the incoming link location.
- Continue by rereading the same incoming link location.

This bypasses the rejected item. Keeping the incoming location unchanged allows consecutive rejections, including a rejected prefix.

If bit 16 is set:

- Do not write the incoming link.
- Use the retained item’s next-link field as the incoming location for the next iteration.

For an ordinary finite list with stable topology and relevant state during filtering, the resulting list contains the marked items in their original relative order.

The function does not clear a rejected item’s own next link or state word. It does not explicitly release rejected storage. A detached item can retain a link to another rejected item or to a surviving item.

Outer and middle nodes are not explicitly removed. A reached middle node remains in its enclosing list even if filtering changes its item-list head to zero.

### Reachability and phase boundaries

Initial marks do not by themselves ensure retention because phase 1 clears them on encounter.

Qualification does not ensure retention either. A qualifying item can be rejected if it is reached during filtering with bit 16 clear. Conversely, a nonqualifying item can be retained if another effect sets its bit before inspection.

Neither statement guarantees a final inspection. The dependency can detach an outer node, middle node, or item chain before filtering reaches it. Items in such detached structures can retain clear bits and unchanged local list links.

A successful state test describes the moment of inspection. Later state changes are not detected by a second inspection unless traversal encounters that item again.

## Side effects

The explicit writes are:

1. One 32-bit state-word write per first-phase item encounter.
2. One incoming-link write per rejected final-phase item encounter.

Under a nonoverlapping layout, incoming-link writes target either a middle node’s item-list head or a retained item’s next-link field. The function does not explicitly modify the root head, outer-node links, or middle-node successor links.

Arbitrary aliasing can invalidate those field-role distinctions. A write through one role may then affect storage used for another role.

Dependency calls add effects that cannot be bounded from this body. They may affect passed objects, unrelated memory, globals, storage lifetimes, or externally visible state; none of those behaviors is established or excluded.

There is no explicit allocation, deallocation, logging, output, or error-reporting operation in the function itself. This does not constrain the dependency.

Unlinking establishes a reachability change, not an ownership transfer or destruction. Ownership of detached storage and the validity of external references are unknown.

## Dependencies

### F-a2ca640ca643

| Aspect | Contract established by this caller |
|---|---|
| Invocation | Once per qualifying second-phase encounter |
| Argument | Current item address |
| Return handling | No returned value is used |
| Ordering | After the complete clearing phase; before final filtering begins |
| Subsequent access | Current item’s next link is read after return |
| Failure handling | No explicit status check or recovery path |

State mutation, link mutation, global effects, allocation, release, and termination behavior are documented_only unknowns.

If a call does not return normally, completion of the current traversal and execution of final filtering are not established.

There are no other explicit function dependencies in the supplied body.

## Acceptance examples

These are conditional static consequences, not runtime observations. Unless further qualified, assume accessible disjoint storage, finite lists, no concurrent mutation, and normally returning dependency calls. These baseline assumptions do **not** imply that the dependency preserves topology.

### Empty outer list

If the root’s outer-list head remains zero for all three phase-entry reads, no item-state writes, dependency calls, or filtering writes occur. The root head is nevertheless read once at the start of each phase.

### Existing marks without replacement, with preserved topology

Suppose a middle node initially contains several marked items. Require that the relevant outer, middle, and item-list topology remain intact through the start of filtering, except for the filtering writes themselves. Also require that nothing sets bit 16 after phase 1 clears it.

Final filtering reaches those items and rejects them all. The middle node’s item-list head becomes zero. No explicit release occurs.

Without the topology condition, the guaranteed result is narrower: items actually encountered during filtering with bit 16 clear are bypassed.

### Root disconnection prevents filtering of the original lists

Suppose a qualifying item is reached in phase 2. Its dependency call sets the root’s outer-list head to zero, sets no state bits, preserves accessible object lifetimes, and leaves the currently traversed lists finite.

Phase 2 can finish through its current traversal positions. Phase 3 reads the zero root head and visits no original middle nodes or items.

The original middle node’s item-list head can remain nonzero. Clearing every original item’s bit does not, by itself, guarantee that its containing list is subsequently emptied.

### Callback qualification is not retention

An item has classification `0x05`, and its second-phase encounter invokes F-a2ca640ca643.

If the item is subsequently encountered during final filtering with bit 16 still clear, that encounter bypasses it. If dependency-driven topology changes prevent the encounter, removal from its former containing list is not guaranteed.

### Selective retention

A list is reached in phase 3 with four items in order. Its topology remains stable except for filtering, and only the second and fourth items have bit 16 set when inspected.

The resulting list contains the second followed by the fourth. The head bypasses the first, and the second item’s next link bypasses the third. The rejected items’ own next links are not cleared.

### Consecutive rejected prefix

A reached list begins with two clear-bit items followed by a marked item, with no intervening mutation.

The head field is first rewritten to the second item, then to the third. The third becomes the retained head.

### Dependency changes the successor

A dependency call replaces the current item’s next link with a different valid item address and preserves the current item’s storage.

After return, phase 2 follows the replacement link, not a saved successor.

### Unresolved selector

For classification zero, qualification depends on whether selector A or selector B equals zero. The five established numeric values alone do not settle this case.

### Shared item address

The same item is reachable through two middle-node lists. If both qualifying encounters occur, the dependency is called twice with the same address. There is no once-per-address guarantee.

## Uncertainties

### Established static evidence

The body establishes the three-phase ordering, accessed offsets, four-byte state and link operations, unsigned one-byte classification read, bit-16 masks, selection comparisons, dependency argument, post-call successor read, and incoming-link filtering.

It also establishes the absence of explicit validation, ownership management, and status reporting in this function.

### Inferred intent

Bit 16 plausibly serves as a retention mark, and selected items plausibly initiate marking through the dependency. Neither interpretation establishes dependency behavior or guarantees that marked objects remain reachable.

### documented_only unknowns

- Numeric values of selector A and selector B.
- Byte order and the classification byte’s numeric position within the state word.
- Complete object layouts and application meanings.
- Dependency effects, including global-state changes and storage lifetime.
- Ownership and disposal of detached items.
- Whether sharing or aliasing is valid in the surrounding system.
- Concurrency guarantees.
- Execution-environment behavior for invalid addresses, misalignment, cycles, and address wraparound.

These unknowns are not replaced with assumed preservation, release, or recovery behavior.

## Evidence

This contract is based on the supplied static function body and opaque dependency mapping. The prior draft and review feedback are not treated as independent behavioral authority.

The distinction between bypassing an encountered item and failing to reach an item follows from the phase-entry head reads, traversal-link reads, and absence of a saved entry graph. The root-disconnection example is a conditional consequence of those operations, not a claim about the dependency’s actual implementation.

No runtime tests have been performed. All acceptance examples are static expectations under their stated conditions.
