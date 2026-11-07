<!-- Trusted provenance: F-fef1ff9f317d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fef1ff9f317d

## Purpose

F-fef1ff9f317d prepares an object-associated record, conditionally inserts that record into a linked list, and then conditionally invokes an indirect dependency.

The established contract concerns memory accesses, dependency calls, branch conditions, and their ordering. Interpreting the record as scheduled work, its calculated key as a deadline, or the surrounding dependencies as synchronization is **inferred intent**, not established behavior.

Three decisions control execution:

- A nonzero record word at byte offset 28 causes an early return, but only after initial preparation and a write at offset 32.
- Otherwise, a nonzero word at offset 4 enables key calculation and insertion. Zero skips insertion, not post-processing.
- After post-processing, an indirect call occurs only if a pointer obtained through shared global state is nonnull and its first word is nonzero.

The same global location supplies the argument to F-338d16f9ce72 and, through a later reload and dereference, supplies the pointer used by the final dispatch check. These are separate reads; intervening dependencies may change that global.

## Interface

### Explicit inputs and return

| Input | Established use |
|---|---|
| Object address | Base for reads at byte offsets 176 and 184; also passed to F-5da16db99672. |
| Second 32-bit value | No explicit use in the visible body. |
| Third 32-bit value | Passed to every reached F-0d70d6f9010c invocation. |
| Fourth 32-bit value | Passed to every reached F-0d70d6f9010c invocation. |

The supplied representation uses a 32-bit object-address value and four-byte pointer-bearing fields. Record offsets and pointer arithmetic are consistent with a 32-bit address model. Required alignment, byte order, and the complete calling convention remain **documented_only**.

There is no explicit return value or status result.

F-0d70d6f9010c also receives a fifth value whose origin is not established as any of the four explicit inputs.

### Object layout

| Byte offset | Width | Established role |
|---|---:|---|
| 176 | 4 bytes | Associated-record address. Read twice, with F-6904f7699031 between the reads. |
| 184 | 4 bytes | Value passed to F-5da16db99672 alongside the object address. |

The first read of offset 176 supplies F-6904f7699031. The second read selects the retained record address used afterward. The two addresses need not agree.

### Record layout

| Byte offset | Width | Established role |
|---|---:|---|
| 0 | 4 bytes | Link to another record; assigned during insertion. |
| 4 | 4 bytes | Insertion-control value and multiplier input. |
| 8 | 4 bytes | Calculated key; its address is passed to the comparison dependency. |
| 24 | 4 bytes | Copy of a global value during insertion preparation. |
| 28 | 4 bytes | Early-return condition; assigned 1 during insertion. |
| 32 | 4 bytes | Copy of a global value before the early-return check. |

Unlisted bytes are not directly accessed by the visible body. Dependencies may access them.

### Global-state labels

The following neutral labels distinguish directly referenced global locations:

| Label | Established role |
|---|---|
| G-shared | Contains a four-byte address. Its value is first passed to F-338d16f9ce72 and later reloaded for the final pointer lookup. |
| G-copy32 | Supplies the word written to record offset 32. |
| G-head | Supplies the address of the list-head slot. |
| G-copy24 | Supplies the word written to record offset 24. |
| G-post | Supplies the argument to F-5bcae2eff9a7. |
| G-target | Supplies the indirect call target when final dispatch is enabled. |

These labels identify global storage locations, not application-level meanings. Values loaded from different locations are not assumed to point to disjoint memory.

In particular, G-shared has two distinct roles:

1. Its loaded value is an argument to F-338d16f9ce72.
2. Its later loaded value is treated as the address of a slot containing another pointer.

The second role adds a memory dereference; the final null check is not a check of G-shared’s value itself.

### Preconditions and ownership

Every reached memory access requires valid, suitably accessible storage. Every reached call requires a usable target and the calling conditions expected by that dependency.

There is no visible null check before dereferencing the object or retained record. On the final path, there is also no null check of the address loaded from G-shared before dereferencing that address.

The body performs no explicit allocation or deallocation. Ownership transfer, retention, and release by dependencies are unknown. Normal completion requires retained addresses to remain usable across intervening calls.

## Behavior

### Initial preparation and early return

The function reads object offset 184, then calls F-5da16db99672 with the object address and the word just read.

After that dependency returns:

1. Object offset 176 is read.
2. F-6904f7699031 is called with that address and the integer 125.
3. Object offset 176 is read again.
4. The second address is retained for subsequent direct record accesses.
5. F-936a8728139d is called without explicit arguments.
6. G-copy32 is read and its value is written to retained-record offset 32.
7. Retained-record offset 28 is read.

A nonzero word at offset 28 causes immediate return. F-338d16f9ce72, insertion processing, F-5bcae2eff9a7, and the final dispatch check are not reached.

This path still includes all three initial dependency calls and the offset-32 write. It is not a no-change fast path.

The retained record address is selected before F-936a8728139d. A change to object offset 176 during that call does not cause the record address to be refreshed.

### Processing an unmarked record

When offset 28 is zero, the function reads G-shared and passes that loaded value to F-338d16f9ce72. It does not dereference that value itself at this stage.

After F-338d16f9ce72 returns, F-add44d4e9aaa is called with the retained record address.

Only after F-add44d4e9aaa returns does the function read record offset 4. Consequently, changes made by either dependency may affect the insertion decision.

If offset 4 is zero, key calculation and insertion are skipped. Execution proceeds directly to post-processing.

### Key calculation

For a nonzero word at offset 4, the function calls F-unmapped-01 without explicit arguments. That dependency supplies a result represented as a 32-bit integer.

The key is the dependency result plus 1000 times the offset-4 value captured before the call. A dependency change to offset 4 during F-unmapped-01 does not replace the captured multiplier input.

After F-unmapped-01 returns, the function:

1. Reads G-head and retains the resulting address of the list-head slot.
2. Writes the calculated key to record offset 8.
3. Reads G-copy24 and writes its value to record offset 24.
4. Reads the current head pointer through the retained head-slot address.

The represented multiplication and addition use 32-bit integer quantities, and the destination is four bytes. No explicit overflow detection, saturation, range check, or negative-value rejection is visible.

For inputs whose multiplication and addition both fit the signed 32-bit range, the numerical result is established. Behavior outside that range remains **documented_only**: the supplied representation does not establish a complete overflow contract.

### Traversal and insertion point

Traversal starts at the head pointer with no preceding node.

For each reached nonnull node, F-0d70d6f9010c receives:

- The address at byte offset 8 in the current node.
- The address at byte offset 8 in the retained record.
- The third explicit input.
- The fourth explicit input.
- The additional value of unresolved origin.

A zero result continues traversal. Any nonzero result stops traversal at the current node. Null also stops traversal, without a comparison call for that null address.

When continuing, the function reads the current node’s offset-0 link **after** the comparison returns. It records that current node as the predecessor and advances to the newly read link. A comparison dependency can therefore affect the next node visited by changing the current node’s link.

The addresses of key fields are passed, not captured key values. No preservation of those fields by comparison dependencies is established.

The head-slot address captured before traversal is retained. Changes to G-head during comparison do not redirect the eventual head-slot write. Mutations to list contents can still affect traversal and insertion.

No ascending order, descending order, equality rule, or stable placement is established. The observable selection rule is solely “continue on zero; stop on nonzero or null.”

### Insertion writes

Once traversal stops, direct writes occur in this order:

1. Publish the retained record address:
   - Through the captured head-slot address if there is no predecessor.
   - Into the predecessor’s offset-0 link otherwise.
2. Write 1 to retained-record offset 28.
3. Write the stopping-node address to retained-record offset 0.

The final link is null when traversal stopped at the end of the list.

Publication precedes the marked-state write, which precedes initialization of the inserted record’s next link. No guarantee hides these intermediate states from observers.

No check rejects a record already present in the list. The initial zero-at-offset-28 condition alone does not establish that the record is absent from existing topology.

### Post-processing and shared-global reload

Both successful insertion and the zero-at-offset-4 path reach post-processing.

The function reads G-post and calls F-5bcae2eff9a7 with that value.

After this call returns, the final check performs these ordered accesses:

1. Reload G-shared.
2. Dereference the newly loaded address to obtain a candidate-object pointer.
3. If that pointer is nonnull, read the candidate object’s first four-byte word.
4. If that word is also nonzero, obtain the indirect target from G-target and invoke F-unmapped-02 without explicit arguments.

If the candidate pointer is null, its first word is not read and dispatch is suppressed. If the pointer is nonnull but its first word is zero, dispatch is also suppressed.

The candidate’s first word is only a condition. It is not the call target.

The earlier G-shared value passed to F-338d16f9ce72 is not retained for this lookup. Any intervening dependency may change G-shared, including F-338d16f9ce72, F-add44d4e9aaa, the key or comparison dependencies, and F-5bcae2eff9a7. The later lookup may therefore use an entirely different slot address. Even if G-shared is unchanged, the slot’s pointer contents may have changed.

There is no visible validity check for the reloaded slot address or the enabled indirect target. The indirect dependency’s result is unused; if it returns normally, this function returns.

## Side effects

Direct writes are limited to reached paths:

| Destination | Condition |
|---|---|
| Record offset 32 | After initial preparation returns normally, before checking offset 28. |
| Record offsets 8 and 24 | Offset 28 was zero and the later offset-4 value was nonzero. |
| Head slot or predecessor link | Traversal completes on the insertion path. |
| Record offset 28 | Assigned 1 after publication. |
| Record offset 0 | Assigned the stopping-node address after offset 28. |

These statements describe direct writes, not guarantees that other storage remains unchanged. Aliasing and dependency effects may produce additional changes.

A zero offset-4 value does not directly clear the mark, reset the key, or unlink the record. A nonzero offset-28 value does not prevent the earlier offset-32 write.

Dependencies may mutate globals, object fields, record contents, list topology, or storage lifetime. Their preservation and ownership contracts are not available.

No explicit error result, rollback, cleanup handler, or recovery path is visible. Invalid memory, invalid call targets, and dependency failures have no specified recovery behavior. A cycle whose comparisons continually return zero can prevent traversal from terminating.

Synchronization and concurrency safety are not established. In particular, no balanced-lock guarantee or atomic insertion guarantee follows from the dependency sequence alone.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-5da16db99672 | Receives the object address and its offset-184 word. |
| F-6904f7699031 | Receives the first post-initial-call offset-176 address and 125. |
| F-936a8728139d | Called without explicit arguments before the offset-32 copy. |
| F-338d16f9ce72 | Receives the first reached load from G-shared when offset 28 is zero. |
| F-add44d4e9aaa | Receives the retained record address before offset 4 is read. |
| F-0d70d6f9010c | Compares field addresses for each examined nonnull node; zero advances, nonzero stops. |
| F-5bcae2eff9a7 | Receives G-post’s value before G-shared is reloaded. |
| F-unmapped-01 | Supplies the integer contribution to the key. |
| F-unmapped-02 | Indirect dependency selected through G-target after both final conditions pass. |

F-unmapped-01 and F-unmapped-02 are document-local opaque identifiers. No corresponding identifiers were supplied in the dependency mapping. Their implementations and broader contracts remain **documented_only**.

## Acceptance examples

These are conditional predictions from static evidence, not executed tests. Unless an example states otherwise, required memory remains valid, dependencies return normally, and dependencies preserve the conditions specified in that example.

### Marked record

If offset 28 is nonzero when inspected:

- The three initial dependencies have already run in order.
- Offset 32 has received the value read from G-copy32.
- No read of G-shared, insertion, post-processing, or final dispatch check follows.

### Record address changes during preparation

Suppose F-6904f7699031 receives record address A and changes object offset 176 to address B.

Subsequent direct record accesses use B. F-6904f7699031 still received A. No equality between those addresses is required.

### Zero insertion-control value

If offset 28 is zero and offset 4 is zero after F-add44d4e9aaa:

- F-338d16f9ce72 has received a value read from G-shared.
- No key dependency or comparison is reached.
- No direct insertion writes occur.
- F-5bcae2eff9a7 and the final G-shared reload still occur.

### Empty-list insertion

If offset 4 is 2, F-unmapped-01 returns 500, and the captured head slot contains null:

- Offset 8 receives 2500.
- Offset 24 receives the value read from G-copy24.
- No comparison occurs.
- The head slot receives the record address, offset 28 receives 1, and offset 0 receives null, in that order.

### Comparison advances once

If the first comparison returns zero and the second returns nonzero:

- The first node’s link is read after its comparison returns.
- That link determines the second node visited.
- Publication updates the first node’s link.
- The inserted record’s final link is the stopping second-node address.

### Shared global changes before dispatch

Suppose G-shared initially contains slot address A. F-338d16f9ce72 receives A. Before returning, F-5bcae2eff9a7 changes G-shared to slot address B.

The final lookup reads the pointer stored at B, not the pointer stored at A. If B contains null, dispatch is suppressed even if A points to an object whose first word is nonzero.

Conversely, if B contains a valid nonnull candidate with a nonzero first word, dispatch is enabled regardless of A’s contents.

### Final gates and missing slot validation

If the reloaded G-shared value names a valid slot containing null, no candidate first-word read occurs.

If that slot contains a valid candidate whose first word is zero, G-target is not invoked.

If both conditions pass, the target comes from G-target, not from the candidate’s first word.

If G-shared itself contains null, the function still attempts the slot dereference. It does not provide a defined “null means no dispatch” path for that case.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the object, record, globals, and list.
- Whether the key represents time or has particular units.
- Overflow behavior outside the established in-range arithmetic cases.
- Comparison ordering, equality treatment, and fifth-argument provenance.
- Dependency preservation, ownership, lifetime, and synchronization contracts.
- Complete calling conventions and indirect-target validity requirements.
- Required alignment, byte order, complete storage sizes, and aliasing constraints.
- Behavior under concurrent mutation or invalid memory.
- Significance of the unused second explicit input outside this body.

“Deadline,” “lock,” and “scheduled item” are possible interpretations, not contract requirements.

## Evidence

This contract is based solely on the supplied static representation and dependency mapping. Established claims concern visible four-byte accesses, offsets, argument flow, branch conditions, and ordering.

The shared-global relationship is directly visible: the same global location supplies the F-338d16f9ce72 argument and is reloaded after F-5bcae2eff9a7 before the candidate-pointer dereference.

No dependency implementations or external behavioral contracts were supplied. No runtime tests have been performed. Acceptance examples are conditional predictions, not observed results.
