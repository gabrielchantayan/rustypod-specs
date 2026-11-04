<!-- Trusted provenance: F-ba7d9afb55b3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ba7d9afb55b3

## Purpose

F-ba7d9afb55b3 rewires a small, linked structure around a designated node. It promotes the node referenced by the designated node’s field at offset +0x0C, moves the promoted node’s +0x08 link into the vacated position, and reconnects the surrounding parent or root link.

For a well-formed binary tree whose +0x08 field is the right-child link and whose +0x0C field is the left-child link, this is a right rotation. That interpretation is inferred from the link transformations. The directly established behavior is the sequence of 32-bit memory reads, comparisons, and writes described below.

The function does not allocate, release, copy, or initialize nodes. It does not validate the structure, report a status, or call another function.

## Interface

The function accepts two address-bearing arguments and returns no value:

- **Context address:** identifies an object containing an address-bearing field at offset +0x10.
- **Pivot address:** identifies the node around which links are changed.

The supplied representation uses integer-valued addresses and four-byte field accesses. The behavioral model therefore treats each accessed field as a 32-bit word, with addresses represented through those words. A native 64-bit pointer layout must not be substituted for this layout. The supplied material does not independently establish a broader platform ABI, byte order, or alignment policy.

The following names describe roles, not source identifiers:

| Role | Meaning |
|---|---|
| Context | Object identified by the first argument. |
| Pivot | Node identified by the second argument. |
| Promoted node | Node address initially read from Pivot +0x0C. |
| Transfer link | Value read from the promoted node’s +0x08 field. |
| Root holder | Object whose address is read from Context +0x10. |
| Previous parent | Address read from Pivot +0x04 when needed. |

Relevant layout:

| Object | Offset | Accessed role |
|---|---:|---|
| Context | +0x10 | Address of the root holder |
| Root holder | +0x04 | Root-node link |
| Node | +0x04 | Parent link |
| Node | +0x08 | First child link; interpreted as right child |
| Node | +0x0C | Second child link; interpreted as left child |

No meaning is established for other fields.

For ordinary, successful tree use, the caller must supply readable and writable storage for every access on the selected branch. In particular:

- Pivot and the promoted node must identify valid nodes.
- Context +0x10 must contain a valid root-holder address.
- A nonzero transfer link must identify a node whose +0x04 field is writable.
- If Pivot is not the root-holder’s current root, Pivot +0x04 must identify an accessible parent node.
- The initial links must be consistent if preservation of tree structure is required.
- Address-plus-offset calculations must designate valid storage. No address-wrap behavior is established.
- Access must not race with unsynchronized modification of these fields.

These are caller preconditions, not checks performed by the function. A zero transfer link is explicitly supported; a zero promoted-node address is not explicitly supported.

## Behavior

### Ordered memory operations

The order below is established for the supplied operation sequence. It matters because several fields are read again after earlier writes.

1. **Select the promoted node.**  
   Read the 32-bit word at Pivot +0x0C. Its value becomes the promoted-node address used throughout the remainder of the operation.

2. **Replace Pivot’s second child link.**  
   Read the word at promoted node +0x08 and write that value to Pivot +0x0C.

3. **Decide whether to repair a transferred node’s parent.**  
   Read promoted node +0x08 again and compare the result with zero.

   If that read is nonzero, read promoted node +0x08 again to obtain the target address, then write Pivot’s address to that target’s +0x04 field.

   Thus the transfer link is not captured once and reused for all three purposes. In an ordinary stable, non-overlapping structure, the reads return the same value. The function itself provides no synchronization or consistency check to ensure that result.

4. **Give the promoted node Pivot’s parent link.**  
   Read Pivot +0x04 and write that word to promoted node +0x04.

5. **Locate and inspect the root slot.**  
   Read Context +0x10 to obtain the root-holder address. The root slot is the word at that address plus +0x04. Read this slot and compare it with Pivot’s address.

6. **Reconnect the external link.**

   - If the root slot equals Pivot, write the promoted-node address to that root slot.
   - Otherwise, read Pivot +0x04 again to obtain the previous-parent address. Read previous parent +0x08.
     - If that field equals Pivot, write the promoted-node address to previous parent +0x08.
     - Otherwise, write the promoted-node address to previous parent +0x0C.

   The non-root fallback does not verify that previous parent +0x0C actually refers to Pivot. Failure of the +0x08 comparison alone selects the +0x0C write.

7. **Make Pivot a child of the promoted node.**  
   Write Pivot’s address to promoted node +0x08.

8. **Set Pivot’s new parent.**  
   Write the promoted-node address to Pivot +0x04.

9. **Return without a result value.**

The root-holder address is obtained after the initial child-link changes and after the promoted node’s parent-link write. Root recognition is based exclusively on the root-slot comparison at that point. It is not based on whether Pivot’s parent link is zero or has a special sentinel value.

### Result for a well-formed, stable structure

Let the initial promoted node be **L**, the initial value of L +0x08 be **T**, and Pivot’s initial parent link be **P**. Assuming the relevant objects and fields do not overlap unexpectedly and are not concurrently modified, normal completion establishes:

- Pivot +0x0C contains T.
- If T is nonzero, T +0x04 contains Pivot.
- L +0x04 contains P.
- L +0x08 contains Pivot.
- Pivot +0x04 contains L.
- The external link previously selecting Pivot is redirected to L:
  - the root slot when its value equals Pivot;
  - otherwise P +0x08 when that field equals Pivot;
  - otherwise P +0x0C.

In this ordinary layout, Pivot +0x08 and L +0x0C are not modified. No payload, ordering key, balance metadata, color field, or node count is explicitly accessed.

These final-state statements rely on the non-overlap and stability assumptions. They are not substitutes for the ordered-access contract when unusual aliasing is present.

### Branching and arithmetic

All field transfers and comparisons concern 32-bit words. The only value tests are:

- whether the re-read transfer link is zero;
- whether the root slot equals Pivot’s address;
- whether the previous parent’s +0x08 field equals Pivot’s address.

There are no signed ordering comparisons, count calculations, or payload comparisons. Address calculations use the fixed byte offsets shown above. The evidence does not establish behavior for arithmetic overflow, invalid address representations, or misaligned accesses.

## Side effects

The function mutates caller-accessible memory in place. On the ordinary path it performs five unconditional word writes:

1. Pivot’s +0x0C link;
2. the promoted node’s +0x04 link;
3. one external root or parent link;
4. the promoted node’s +0x08 link;
5. Pivot’s +0x04 link.

A nonzero transfer link adds one earlier write to the transferred node’s +0x04 field.

There is no transaction boundary or rollback mechanism. Earlier mutations occur before later parent and root accesses. If an access fails under the execution environment’s memory rules, the supplied evidence establishes neither recovery nor restoration of the previous structure.

There are no explicit allocations, deallocations, callbacks, I/O operations, or separately named global accesses. This does not prove that addressed storage is disjoint from global storage: either input may identify memory with broader visibility.

No memory-ownership transfer is expressed. The function changes structural links without visibly acquiring or releasing storage. Any ownership meaning attached to parent and child links belongs to an external contract not supplied here.

The operation is not established as atomic or thread-safe. Observers able to inspect intermediate states can encounter only partially completed rewiring.

## Dependencies

No external function dependencies are visible. F-ba7d9afb55b3 performs its work through direct memory accesses, comparisons, and stores.

The required supporting contract is a memory-layout contract: the caller must provide objects with the fields and access permissions described above. No allocator, destructor, balancing routine, validation routine, or error handler is invoked.

There are consequently no unknown callee effects to assume. In particular, no callee can be credited with preserving globals, validating addresses, or managing node lifetime.

## Acceptance examples

The following are conditional acceptance scenarios derived from static evidence. They have not been executed.

### Root promotion with a nonzero transfer link

Assume:

- The root slot contains Pivot.
- Pivot +0x0C contains L.
- Pivot +0x04 contains P, which may be a caller-defined root-parent marker.
- L +0x08 contains a valid, nonzero node T.
- The relevant objects do not overlap.

On normal completion:

- The root slot contains L.
- L’s parent link contains P.
- L’s +0x08 link contains Pivot.
- Pivot’s parent link contains L.
- Pivot’s +0x0C link contains T.
- T’s parent link contains Pivot.

No dereference of P is required by the external-link branch in this scenario. P is copied into L’s parent field, but the root-slot match bypasses the previous-parent child-link lookup.

### Non-root promotion through the parent’s first child link

Assume the root slot does not contain Pivot, Pivot’s parent link contains valid node P, and P +0x08 contains Pivot.

On normal completion, P +0x08 contains L. In a non-overlapping layout, P +0x0C is unchanged by the external-link update. The remaining local links change as described above.

### Non-root promotion through the fallback child link

Assume the root slot does not contain Pivot, P is valid, and P +0x08 does not contain Pivot.

The function writes L to P +0x0C without comparing that field with Pivot. For a consistent tree, this is the case where Pivot was P’s second child. For an inconsistent structure, an unrelated +0x0C link can be overwritten; no validation failure is reported.

### Empty transfer subtree

Assume L +0x08 is zero and remains zero during the repeated reads.

Pivot +0x0C becomes zero. The transferred-node parent repair is skipped. The function still reconnects the external link, writes Pivot into L +0x08, and makes L Pivot’s parent. There is no dereference through the zero transfer link on this path.

### Both parent child fields initially reference Pivot

Assume a non-root case in which both P +0x08 and P +0x0C contain Pivot, without other overlapping fields.

The +0x08 comparison succeeds, so only P +0x08 is redirected to L. P +0x0C remains Pivot. The function does not detect or repair the duplicate reference.

### Invalid promoted-node address

If Pivot +0x0C contains zero or another invalid node address, the function does not reject it explicitly. It attempts an access at that address plus +0x08 before completing any structural validation. No successful return or portable failure mechanism is promised for this input.

## Uncertainties

- **Inferred intent:** The link pattern matches a binary-tree right rotation when +0x08 means right child and +0x0C means left child. Those field names are interpretations rather than independently supplied type definitions.
- **documented_only — broader data structure:** Tree ordering, balancing policy, sentinel conventions, payload representation, and node ownership are unknown.
- **documented_only — machine environment:** Byte order, alignment rules, memory-fault behavior, and the complete calling convention are not established beyond the supplied four-byte access representation.
- **documented_only — malformed aliasing:** No well-formedness checks exclude self-links, cycles, or overlap among nodes, Context, and the root holder. Such overlap can make earlier writes alter later reads. The ordered-access description governs; the ordinary rotation postconditions do not.
- **documented_only — concurrent modification:** No synchronization or stable-read guarantee is visible. In particular, the transfer-link test and its subsequent target-address read need not be treated as one captured value.
- **documented_only — failures:** There is no explicit error result or recovery path. Effects of invalid accesses are execution-environment dependent.
- **documented_only — external invariants:** No evidence establishes whether callers require additional metadata updates before or after this operation.

## Evidence

The established behavior comes solely from the supplied static function representation:

- Two address-bearing inputs and no returned value.
- Four-byte field accesses at the listed offsets.
- Repeated reads of the promoted node’s +0x08 field.
- One conditional transferred-node parent write.
- Root-slot selection before the non-root parent-child branch.
- An unchecked second-child fallback.
- Final writes linking Pivot beneath the promoted node.
- No visible function calls or explicit validation.

The rotation interpretation and ordinary final-state description are deductions from those operations under the stated structural preconditions. No runtime tests, memory traces, or independent type definitions have been provided or performed.
