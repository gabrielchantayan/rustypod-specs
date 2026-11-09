<!-- Trusted provenance: F-0afbac3aaabc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0afbac3aaabc

## Purpose

F-0afbac3aaabc obtains a node, copies two four-byte payload components into it when an address condition permits, attaches it at a caller-supplied position, performs color-dependent upward repair, and reports the obtained node through an output slot.

The established behavior includes a container-field increment, conditional updates to header links, one-byte color writes, and calls to structural dependencies. The function does not visibly search for an insertion position, validate ordering, or reject duplicates.

The layout and repair pattern suggest red-black tree insertion. That is **inferred intent**, not an established guarantee: the node provider, comparator, and structural dependencies have no supplied implementations.

## Interface

### Arguments and result

The five represented arguments occur in this order:

| Argument | Established role |
|---|---|
| Result destination | Address of a writable four-byte slot receiving the obtained node address on normal completion. |
| Container address | Base address for header storage, an incremented field, and comparator-related storage. |
| Side selector | A 32-bit integer. Any nonzero value forces left attachment. Zero permits comparison when the parent differs from the header. |
| Parent address | Address of the proposed parent, or the header for the special attachment case. |
| Payload address | Source of two four-byte payload components; also supplied directly to the comparator when comparison is required. |

No meaningful direct return value is established. The result destination is written only after attachment, repair, and the final root-color write.

### Widths and layouts

The representation uses four-byte addresses and four-byte integer fields. Color reads and writes are one byte wide.

| Container offset | Established use |
|---|---|
| +16 | Four-byte header address. |
| +20 | Four-byte field incremented once. |
| +25 | Address passed as comparator-related storage; no direct access width is established for that storage. |

The incremented field is plausibly a node count, but that interpretation is inferred.

Nodes and the header share the following accessed offsets:

| Offset | Node use | Header use |
|---|---|---|
| +0 | One-byte color | No direct header-color access is established as a separate operation. |
| +4 | Parent address | Root address |
| +8 | Left-child address | Left-extremum address |
| +12 | Right-child address | Right-extremum address |
| +16 | First four-byte payload component | No separate header-payload use established |
| +20 | Second four-byte payload component | No separate header-payload use established |

Bytes between +0 and +4 are not directly accessed here.

Color zero permits repair through a parent and selects recoloring when found in an existing uncle candidate. Any nonzero color stops repair through a parent or selects the structural-repair branch for an uncle candidate. Direct color writes use only zero and one. Interpreting these as red and black is inferred.

### Preconditions

Normal completion requires accessible storage for every read and write reached on the selected path, including:

- The container’s header and incremented fields.
- The parent’s selected child-link field.
- The obtained node’s parent field and payload fields when copied.
- Every relative’s color and link fields reached during repair.
- The final root’s color byte.
- The result destination.

The function supplies no visible checks for valid parent selection, acyclicity, consistency of parent and child links, non-overlap, or adequate storage extent. These requirements also depend on dependency behavior; unknown callees cannot be assumed to preserve storage, links, globals, or ownership.

## Behavior

### Node acquisition and payload transfer

The first dependency call is F-59b180f4ecba with the container address. Its returned address is retained as the node ultimately reported, independently of later movement of the repair cursor.

The function tests whether the returned address plus 16 differs from zero. This is not a test that the returned address itself is nonzero.

When the condition succeeds, the transfer consists of four ordered operations:

1. Read four bytes at the payload address.
2. Write that value at obtained-node offset +16.
3. Read four bytes at payload offset +4.
4. Write that value at obtained-node offset +20.

The transfer is neither established as atomic nor guaranteed to preserve an eight-byte source snapshot. In particular, the first destination write precedes the second source read, so overlap can affect the second copied value.

If the address condition fails, both payload reads and both payload writes are skipped. The remainder of the function is not skipped.

A zero node address is not handled as an ordinary failure result by this condition. No allocation-failure protocol follows from it. Address-wrap behavior and validity of unusual returned addresses remain unresolved.

### Container increment and side selection

After the payload stage, the function reads the four-byte container field at +20, adds one, and writes the result back. This occurs before comparison or attachment.

The field is represented as signed 32-bit storage. No overflow check, saturation, error result, or rollback is visible. A portable overflow contract is not established.

The function next reads the current header address and compares it with the supplied parent. Left attachment is selected without comparison when either:

- The parent equals that header; or
- The side selector is nonzero.

Otherwise, F-257a89ed2c43 is called with:

1. Container address plus 25.
2. The original payload address.
3. Parent address plus 16.

A nonzero result selects left attachment; zero selects right attachment. No ordering relation or equality policy is established.

The comparator sees the original payload storage after the preceding copy and increment. Its inputs are not guaranteed to match a pre-call payload snapshot, particularly under aliasing.

The function also retains two 32-bit values for a possible later six-argument structural call. They are the original selector and parent address when comparison is bypassed. Both are zero when comparison is performed, regardless of its result.

### Attachment and header updates

For left attachment, the function first writes the obtained node address to parent offset +8. It then reads the container’s header address again.

If that header differs from the supplied parent, the function reads header offset +8 and replaces it with the obtained node address only when its previous value equals the supplied parent.

If that header equals the supplied parent:

1. The header’s +4 root field receives the obtained node address.
2. The container’s header address is read again.
3. The +12 field of the header selected by that read unconditionally receives the obtained node address.

With a stable header equal to the supplied parent, the writes therefore set header +8, +4, and +12, in that order.

For right attachment:

1. Parent offset +12 receives the obtained node address.
2. The current container header address is read.
3. That header’s +12 field is read and replaced with the obtained node address only if it equals the supplied parent.

After either path, obtained-node offset +4 receives the supplied parent address.

Extremum updates depend on address equality, not a further comparison. Existing child links are overwritten without a visible check that they are empty.

Repeated header reads are part of the established access sequence. In particular, a header read after the comparator need not return the value observed before that call.

### Upward repair

Repair begins with the obtained node as the current node. At each loop check, the function reads the current header address and that header’s +4 root address.

If the root equals the current node, repair stops without reading the current node’s parent color for that iteration. Otherwise, the current node’s parent address and the parent’s color byte are read. A nonzero parent color stops repair.

For a zero-colored parent, the function reads the grandparent through the parent’s +4 field, then reads the grandparent’s +8 child address. Equality between that child and the parent selects the left-parent case; inequality selects the other case. The latter does not independently verify that the parent is the grandparent’s right child.

#### Parent equals the grandparent’s left child

The uncle candidate is read from grandparent offset +12.

If the candidate exists and its color is zero:

1. Write one to the parent’s color.
2. Write one to the candidate’s color.
3. Follow the current node’s parent chain and write zero to the reached grandparent’s color.
4. Follow that chain again to obtain the next current node.
5. Reevaluate repair.

If the candidate is absent or has nonzero color:

- If the current node equals the parent’s right child, call F-013646405fff with the container and parent, then make that former parent the current node.
- Read the resulting current node’s parent and write one to its color.
- Read the parent chain again and write zero to the reached grandparent’s color.
- Read that chain again and call F-c78be994279e with the container and reached grandparent.
- Reevaluate repair.

#### Parent differs from the grandparent’s left child

The previously read grandparent-left address is the uncle candidate.

If that candidate exists, its one-byte color is read and zero-extended to 32 bits. A zero value selects the same recoloring sequence described above, with no structural call during that step.

Otherwise, structural repair proceeds:

- If the current node equals the parent’s left child, call F-c78be994279e in its six-argument represented form, then make the former parent the current node.
- Write one to the parent reached from the resulting current node.
- Follow the parent chain again and write zero to the reached grandparent.
- Read that chain again and call F-013646405fff with the container and reached grandparent.
- Reevaluate repair.

For the six-argument call, the fourth represented argument is conditional:

- **Existing nonzero-colored candidate:** its color byte zero-extended to 32 bits, giving a value from 1 through 255.
- **Absent candidate:** the previously read four-byte grandparent address.
- **Existing zero-colored candidate:** this call is not reached for that repair step.

Links used after helper calls are read afterward. Their values are not guaranteed to match links observed before those calls.

### Finalization

After repair stops, the function reads the current header and its root address, writes color one at that root, and then writes the original obtained-node address to the result destination.

The reported node is not replaced by the repair cursor or by the final root.

## Side effects

Direct writes can affect:

- Two four-byte payload fields in the obtained node.
- The incremented container field.
- A selected child link in the supplied parent.
- Header root and extremum links.
- The obtained node’s parent link.
- Parent, uncle, grandparent, and final-root color bytes.
- The caller’s four-byte result slot.

The node’s initial color and child links are not directly initialized here. Suitable initial contents or dependency behavior are therefore necessary for the intended tree operation.

No direct release, destruction, duplicate rejection, transaction boundary, rollback, or error-status write is visible. Dependency calls may introduce additional effects, including effects on allocation, ownership, shared state, or globals; none are established.

Copying the payload words does not establish ownership transfer for resources they might represent. Responsibility for eventual node or payload destruction remains unknown.

There is no established concurrency or atomicity guarantee. If execution cannot complete, earlier mutations may remain. The function contains no visible recovery path for inaccessible storage or dependency failure.

Aliasing can also change later accesses. The access sequence remains established, but conceptual roles such as “header field,” “payload,” and “output slot” are not guaranteed to occupy disjoint storage.

## Dependencies

| Opaque ID | Established relationship | Unresolved contract |
|---|---|---|
| F-59b180f4ecba | Called first with the container; its result supplies the obtained node address. | Allocation, initialization, failure representation, ownership, and side effects. |
| F-257a89ed2c43 | Called only when the selector is zero and the observed header differs from the parent. Receives comparator-related storage and two payload addresses. | Ordering, equality treatment, ownership, and side effects. |
| F-013646405fff | Called with container and a node in left-parent inner repair or opposite-side outer repair. | Exact structural changes and all other effects. A left rotation is inferred. |
| F-c78be994279e | Called with two represented arguments in left-parent outer repair, and six in opposite-side inner repair. | Exact structural changes, actual argument consumption, and all other effects. A right rotation is inferred. |

The six represented arguments to F-c78be994279e, in order, are:

1. Container address.
2. The parent address observed for that repair step.
3. Uncle candidate address, including zero when absent.
4. The existing candidate’s nonzero color byte zero-extended to 32 bits, or the previously read grandparent address when the candidate is absent.
5. Original side-selector value if attachment bypassed comparison; otherwise zero.
6. Original supplied parent address if attachment bypassed comparison; otherwise zero.

The fourth argument is not a Boolean. A candidate color of 255 produces the 32-bit value 255, not one or zero. A zero-colored existing candidate takes recoloring instead and does not reach this invocation.

The significance of the extra arguments to the dependency’s actual interface is **documented_only**. Their represented values are established; whether they are consumed or ignored is not.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes accessible storage and sufficiently stable, non-overlapping state except where stated. Structural scenarios additionally require dependencies to leave the relevant links suitable for the described continuation.

### Header-parent attachment

Given that the supplied parent equals the relevant header throughout attachment:

- Comparison is skipped regardless of selector.
- The payload transfer occurs if its address condition succeeds.
- The container field is incremented once, subject to unresolved overflow semantics.
- Header +8, +4, and +12 receive the obtained node address, in that order.
- The obtained node’s parent becomes the header.
- If the root remains the obtained node, repair stops at its first check.
- The final root color becomes one, followed by the result-slot write.

### Forced-left and comparison-selected right attachment

For a non-header parent and nonzero selector, comparison is skipped and the parent’s left link is overwritten. The header’s left-extremum link changes only if it identifies that parent.

For a zero selector, non-header parent, and zero comparator result, the parent’s right link is overwritten. The observed header’s right-extremum link changes only if it identifies that parent.

Neither case establishes that the overwritten child link was empty or that ordering is preserved.

### Nonzero parent color

If the obtained node is not the root and its parent color is any value from 1 through 255:

- That iteration stops before reading the grandparent or uncle.
- No structural helper is called from that iteration.
- The final root still receives color one.
- The obtained node address is reported.

### Zero-colored uncle

With a zero-colored parent and an existing zero-colored uncle candidate:

- Parent and uncle receive one.
- The grandparent reached through the current parent chain receives zero.
- Repair continues from the grandparent.
- No structural helper is called for that recoloring step.

This includes the case where the parent differs from the grandparent’s left child: the candidate’s zero color selects recoloring, not the six-argument invocation.

### Six-argument call with an existing candidate

Suppose the parent differs from the grandparent’s left child, the current node equals the parent’s left child, and the uncle candidate exists with color 128.

The six-argument F-c78be994279e invocation receives that candidate address as its third argument and the 32-bit value 128 as its fourth. Colors one and 255 analogously produce fourth arguments one and 255.

If attachment used comparison, arguments five and six are both zero. If comparison was bypassed, they carry the original selector and supplied parent address.

### Six-argument call with no candidate

Under the same parent/current-node relationships, but with an absent uncle candidate:

- Argument three is zero.
- Argument four is the previously read grandparent address.
- No uncle color is read.

The fourth argument is not replaced with zero merely because the candidate is absent.

### Skipped payload transfer

If obtained-node address plus 16 evaluates to zero under the applicable address semantics, no payload reads or writes occur. The increment, attachment, repair, and finalization still remain reachable. Skipping the copy does not establish that subsequent addresses are valid.

## Uncertainties

The following remain **documented_only**:

- Node acquisition, initialization, failure handling, and lifetime.
- Meaning and overflow behavior of the incremented field.
- Comparator ordering and duplicate policy.
- Structural dependency implementations and actual consumption of extra arguments.
- Payload-resource ownership and eventual destruction responsibilities.
- Dependency effects on globals, container state, storage validity, and ownership.
- Target-level address arithmetic, wrapping, alignment, and pointer validity.
- Outcomes for malformed links, cycles, inaccessible storage, or dependency failure.
- Termination and tree validity for arbitrary initial states or arbitrary dependency implementations.

These unknowns do not change the established fourth-argument distinction: existing nonzero candidate color is zero-extended; candidate absence retains the earlier grandparent address; existing zero color selects recoloring.

## Evidence

The supplied static representation establishes the argument roles, accessed widths and offsets, payload-operation order, increment placement, attachment branches, header rereads, repair conditions, color writes, represented dependency arguments, final root-color write, and result assignment.

In particular, the opposite-side repair branch conditionally replaces the retained four-byte grandparent address with an unsigned conversion of the candidate’s one-byte color only when that candidate exists. Zero color selects recoloring; nonzero color can reach the six-argument structural call.

Red-black insertion and rotation names are inferred intent. Dependency contracts, ownership rules, and portable overflow or address-wrap semantics are not established.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed.
