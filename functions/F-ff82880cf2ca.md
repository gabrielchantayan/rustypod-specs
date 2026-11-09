<!-- Trusted provenance: F-ff82880cf2ca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff82880cf2ca

## Purpose

F-ff82880cf2ca obtains a node, copies an eight-byte input record into it, attaches it to a pointer-linked structure, updates bookkeeping and boundary links, performs state-based balancing, and publishes the obtained node’s address through an output location.

**Established static evidence:** the function calls a node-providing dependency, conditionally copies two 32-bit words, increments a 32-bit container field, selects an attachment link, modifies links and one-byte states, invokes structural dependencies, sets the current root’s state to one, and finally writes the obtained address to the output.

**Inferred intent:** the structure is a red-black tree whose header stores the root and cached minimum and maximum. State zero appears to mean red; state one appears to mean black. F-e4074d2ad6d9 and F-9678b51c49d5 appear to implement complementary rotations. These interpretations do not establish the dependencies’ actual contracts.

The visible operations are specified below independently of those interpretations. Normal completion depends on valid memory, suitable node initialization, and coherent links; the function does not validate those conditions.

## Interface

The function has five arguments and no explicit return value.

| Argument | Established role |
|---|---|
| Output location | Receives the obtained node address as a 32-bit value at normal completion. |
| Container address | Identifies mutable container state and is passed to the node-providing and structural dependencies. |
| Placement selector | A 32-bit integer; any nonzero value bypasses comparison and selects the first-child path. |
| Attachment address | Identifies the node or header whose link is initially changed. |
| Input-record address | Identifies two readable 32-bit words; also passed unchanged to the comparison dependency when comparison is required. |

The supplied representation uses 32-bit addresses and 32-bit integer fields. The following offsets are byte offsets in that representation, not a portable layout for wider pointers.

### Container layout

| Offset | Visible use |
|---|---|
| +16 | 32-bit header address, reloaded at multiple stages. |
| +20 | 32-bit field incremented before placement selection. |
| +25 | Address passed as comparison context; no direct access width is established. |

The field at +20 is plausibly a node count, but only its increment is established.

### Node and header layout

| Offset | Ordinary-node use | Header use |
|---|---|---|
| +0 | One-byte balancing state | No direct state access established merely by its header role |
| +4 | 32-bit parent address | 32-bit current-root address |
| +8 | 32-bit first-child address | Inferred minimum-node address |
| +12 | 32-bit second-child address | Inferred maximum-node address |
| +16 | First 32-bit payload word | Not established |
| +20 | Second 32-bit payload word | Not established |

The function directly initializes the obtained node’s payload and parent field. It does not directly initialize that node’s child links or initial state byte.

### Preconditions and unsupported conditions

Normal completion requires:

- Valid readable and writable storage for every access reached on the selected path, with target-required alignment.
- A usable header address and root link whenever they are read.
- An obtained node that supports the payload, parent, and balancing accesses.
- Parent and child links suitable for the traversals performed.
- Dependencies that return normally and leave storage usable for subsequent operations.
- An output location writable at final publication.

No allocation-failure check, structural validation, alias rejection, or rollback is visible. Disjoint storage is not enforced. Aliasing can change later reads and invalidate conventional tree interpretations even though the access sequence remains the same.

## Behavior

### Node acquisition and payload copying

The first dependency call is F-79311d0195b9 with the container address. Its returned address is retained as the obtained node address and is the value eventually published.

Payload copying is guarded by whether the address **16 bytes beyond the obtained address** is nonzero. This is not a test that the obtained address itself is nonnull.

When the guard succeeds, the order is:

1. Read the first 32-bit input word.
2. Write that word to obtained-node offset +16.
3. Read the second 32-bit input word at input offset +4.
4. Write that word to obtained-node offset +20.

There is no atomic eight-byte snapshot. Overlap can allow the first destination write to alter the second source read.

A null obtained address is not handled as a clean failure: its address-plus-16 expression does not reject it, and the first destination would be address 16 under the represented address arithmetic. If a target address calculation makes the guarded address zero, only the payload copy is skipped; subsequent node accesses are not suppressed. Neither case establishes a supported failure convention.

### Bookkeeping increment

After the copy stage, the function reads the 32-bit field at container +20, adds one, and writes the result back.

This precedes comparison, attachment, balancing, and output publication. No capacity check or overflow branch is visible.

The supplied representation expresses the increment as signed 32-bit arithmetic. Behavior at the signed upper boundary is **documented_only**; a portable modulo-wrap guarantee is not established.

### Placement selection

The function first reads the header address at container +16 and compares it with the attachment address.

The first-child path is selected without comparison if either:

- The attachment address equals that header address; or
- The placement selector is nonzero.

The header equality check occurs before the selector can bypass comparison.

Otherwise, F-257a89ed2c43 receives, in order:

1. Container address plus 25.
2. The original input-record address.
3. Attachment address plus 16.

A nonzero result selects the first-child path. Zero selects the second-child path. Negative nonzero results are not treated differently from other nonzero results.

Comparison occurs after node acquisition, copying, and the bookkeeping increment. Its input is the original record address, not the obtained node’s payload. It therefore observes whatever contents are present at that address when called, including effects of preceding aliasing or dependency activity.

For one later expanded structural call, the function also retains two values determined by this decision: the original selector and attachment address when comparison was bypassed, or zero and zero when the comparison path was entered.

### Attachment and header updates

On the **first-child path**, the obtained address is written to attachment +8. The header address is then reloaded.

If that reloaded header differs from the attachment:

- Read header +8.
- Replace that field with the obtained address only if it equals the attachment address.

If the reloaded header equals the attachment:

- Write the obtained address to header +4.
- Reload the header address again.
- Write the obtained address to +12 of that newly read header.

Thus, with stable header identity and non-overlapping storage, attachment to the header writes its +8, +4, and +12 fields in that order. Under the inferred layout, these become minimum, root, and maximum.

On the **second-child path**:

- Write the obtained address to attachment +12.
- Reload the header address.
- Read its +12 field.
- Replace that field with the obtained address only if the field equals the attachment address.

After either path’s attachment and header updates, write the attachment address to obtained-node +4.

This order publishes a link to the obtained node before initializing its parent field. There is no atomic insertion boundary.

### Balancing loop

The working node initially is the obtained node.

At each loop test, the function reloads the header address and reads its root field at +4. If the working node equals that root, balancing stops.

Otherwise, the function reads the working node’s parent and then the parent’s state byte. A nonzero parent state stops balancing. This test does not require the state to equal one.

For a zero-state parent, the function reads the parent’s parent address as a 32-bit unsigned value and reads that ancestor’s first-child link. The following two orientations are distinguished.

#### Parent equals the ancestor’s first child

The ancestor’s second child is the opposite-side relative.

If that relative is nonnull and has state zero:

- Write one to the parent’s state.
- Write one to the relative’s state.
- Follow the working node’s current parent and grandparent links and write zero to the resulting ancestor’s state.
- Follow those links again to select the next working node.
- Repeat the loop.

If the relative is null or has nonzero state:

- If the working node equals the parent’s second child, call F-e4074d2ad6d9 with the container and saved parent address, then make that saved parent the working node.
- Read the working node’s current parent link and write one to that parent’s state.
- Read the working node’s current parent and grandparent links and write zero to that ancestor’s state.
- Read the ancestor address again through the current links and call F-9678b51c49d5 with the container and that address.

Links used after the preliminary helper are read after it returns. No assumption that the helper preserves those links is needed to describe these accesses.

#### Parent does not equal the ancestor’s first child

The previously read first child is the opposite-side relative. The function does not additionally check that the parent equals the ancestor’s second child.

If the relative is nonnull, its one-byte state is read and widened to a 32-bit unsigned value. A zero state takes the recoloring and upward-movement path described above.

Otherwise, the structural path is taken:

- If the working node equals the parent’s first child, call F-9678b51c49d5 using the expanded argument representation described below, then make the saved parent the working node.
- Write one to the state of the working node’s current parent.
- Write zero to the state of its current grandparent, found through current links.
- Read the grandparent address again and call F-e4074d2ad6d9 with the container and that address.

The expanded call’s fourth supplied value depends on short-circuit evaluation:

- **Null relative:** no relative-state read or assignment occurs. The fourth value retains the ancestor address read earlier, represented as a 32-bit unsigned value.
- **Nonnull relative:** reaching this structural path means its state was nonzero. The fourth value is that nonzero byte widened to 32 bits.

Both structural paths return to the loop test. Neither has an unconditional exit after the final helper.

### Final root state and output publication

After balancing stops, the function reloads the header and its root link, then writes one to the current root’s state byte.

Only afterward does it write the original obtained-node address to the output location. The published address is not necessarily the final root or the final working node.

If an earlier memory access or dependency does not complete normally, the final output write is not reached. The function establishes no defined error value.

## Side effects

Direct writes may affect:

- Two 32-bit payload fields in the obtained node.
- The 32-bit bookkeeping field.
- An attachment child link.
- Header root and boundary links.
- The obtained node’s parent link.
- State bytes of parents, opposite-side relatives, ancestors, and the final root.
- The output location.

Header reloads are behaviorally relevant. Earlier writes, aliasing, or dependency effects can change which header is used later. Likewise, repeated parent-link reads can observe changes made by earlier writes or helpers.

No direct deallocation is visible. The function uses the obtained address as live linked storage, but allocation ownership, lifetime, release obligations, and any ownership effects of dependencies remain unknown.

There is no visible synchronization or transactional rollback. Mutation can remain partial if execution stops after bookkeeping or link updates. With overlapping storage, even the final output write may overwrite structure state; disjointness must be supplied externally for the conventional insertion interpretation.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-79311d0195b9 | Called first with the container; returns the retained node address. | Initialization, allocation mechanism, failure convention, ownership, other mutations. |
| F-257a89ed2c43 | Conditionally receives container +25, original input address, and attachment +16. Zero selects the second-child path. | Ordering semantics, record interpretation, mutation, failure behavior. |
| F-e4074d2ad6d9 | Receives container and a parent or current ancestor address during balancing. | Exact transformation, preserved state, lifetime and ownership effects. |
| F-9678b51c49d5 | Receives container and a parent or current ancestor address; one call has an expanded representation. | Exact transformation, authoritative signature, consumption of additional values. |

The expanded F-9678b51c49d5 call supplies these values in order:

1. Container address.
2. Saved parent address.
3. Opposite-side relative address, possibly null.
4. If the relative is null, the earlier ancestor address as a 32-bit unsigned value; otherwise, the relative’s nonzero state byte widened to 32 bits.
5. Original placement selector when comparison was bypassed; zero when comparison was entered.
6. Original attachment address when comparison was bypassed; zero when comparison was entered.

In particular, a null third value does **not** establish a zero fourth value. The short-circuited state assignment leaves the previously loaded ancestor-address value in place.

These supplied values are established by the static representation. Whether F-9678b51c49d5 consumes the trailing values is **documented_only**. The representation does not establish an authoritative six-argument API, nor justify treating those values as irrelevant.

No dependency is assumed to preserve globals, header identity, payloads, or node lifetime.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, they assume valid disjoint storage, stable header identity, initialized node links and state, and dependencies compatible with the described accesses.

### Header attachment

If the attachment equals the header:

- Comparison is bypassed.
- The payload copy and bookkeeping increment precede attachment.
- Header +8, +4, and +12 receive the obtained address in that order.
- The obtained node’s parent becomes the attachment address.
- The first balancing test sees the obtained node as root and skips the body.
- The root state becomes one before output publication.

### Forced first-child placement

For a non-header attachment and any nonzero selector:

- Comparison is not called.
- Attachment +8 receives the obtained address.
- Header +8 changes only if its subsequently read value equals the attachment.
- A nonzero parent state stops balancing without a structural call on that iteration.
- Finalization still forces the current root’s state to one.

### Comparison-selected placement

For a non-header attachment and zero selector:

- Comparison receives the original input address after copying and incrementing.
- A zero result selects attachment +12 and the conditional header +12 update.
- Any nonzero result selects attachment +8 and the corresponding first-boundary update.
- The retained fifth and sixth values for a later expanded call are both zero, whichever side comparison selects.

### Zero-state opposite-side relative

With a zero-state parent and nonnull zero-state opposite-side relative:

- Parent and relative states become one.
- The current grandparent’s state becomes zero.
- Balancing moves upward through freshly read links.
- That iteration makes no structural call.

### Expanded call with a null relative

Suppose the parent is not the ancestor’s first child, that first-child link is null, and the working node equals the parent’s first child.

Then the preliminary F-9678b51c49d5 call has a null third value. Its fourth value is the earlier ancestor address represented as an unsigned 32-bit value—not a substituted zero. No state byte is read through the null relative.

If placement bypassed comparison, the final two supplied values are the original selector and attachment. If placement entered comparison, they are zero and zero.

### Expanded call with a nonnull relative

Under the same orientation and inner-child condition, a relative with state 7 yields a fourth supplied value of 7, widened to 32 bits.

A relative with state zero instead takes the recoloring path and does not reach that expanded call during that iteration.

### Null obtained address

A null result from F-79311d0195b9 is not rejected by the payload guard and does not establish a clean null-output result. Successful insertion under that condition is unsupported.

## Uncertainties

The following remain **documented_only**:

- Authoritative argument types, calling conventions, and dependency signatures.
- Whether the expanded structural call’s additional supplied values are consumed.
- Comparator semantics and the meaning of its context.
- Payload interpretation beyond two copied 32-bit words.
- Node initialization and allocation-failure behavior.
- Dependency failure and nonlocal-exit behavior.
- Signed bookkeeping overflow and address-boundary arithmetic.
- Ownership, lifetime, permissible aliasing, and release obligations.
- Structural dependencies’ effects on globals, header identity, and storage validity.
- Termination for malformed or cyclic links.
- Runtime validity of the inferred red-black-tree and rotation interpretation.

No runtime tests have been performed.

## Evidence

The evidence is the supplied static function representation and opaque dependency mapping.

Direct evidence establishes access widths and offsets, payload-copy ordering, the address-based copy guard, bookkeeping order, placement conditions, attachment writes, repeated header and parent-link reads, state transitions, dependency call sites, and final publication.

For the expanded call specifically, the ancestor address is first loaded as an unsigned 32-bit value. Assignment of the relative’s widened state occurs only when the relative is nonnull. Short-circuit evaluation therefore preserves the ancestor-address value when the relative is null; a nonnull relative reaching that call contributes its nonzero widened state instead.

Tree terminology, boundary-cache interpretation, color meanings, and rotation roles are inferred intent. Dependency implementations and executable behavior are not established by the available evidence.
