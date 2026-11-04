<!-- Trusted provenance: F-bead5e7925d5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bead5e7925d5

## Purpose

F-bead5e7925d5 performs a local pointer-topology transformation using two input addresses. Under the conventional binary-tree interpretation of the observed fields, it is a **left rotation** around the node supplied as the second argument.

The operation promotes that node’s right child, moves the promoted node’s left subtree into the original node’s right-child position, reconnects the promoted node to the original node’s former incoming link, and makes the original node the promoted node’s left child.

The pointer transformation is established by static evidence. The interpretation as a tree rotation is inferred from the field accesses and resulting relationships. No tree type, ordering invariant, balancing policy, or caller contract is available.

The function does not validate its inputs. In particular, it requires a usable right-child node even though it conditionally handles a null transferred subtree. It has no explicit result value, error result, allocation, release, or call to another function.

## Interface

### Arguments and result

The function accepts two values represented as 32-bit integers in the supplied representation, both used as addresses:

- **Context address:** identifies an object containing an indirect link to a root-holder object.
- **Pivot address:** identifies the node around which the transformation is performed.

The function returns no explicit value. Its observable result is the set of memory writes described below.

The argument representation establishes 32-bit address-bearing values for this contract. It does not independently establish a portable native-pointer ABI, calling convention, alignment rule, or address-space model.

### Observed layouts

Offsets below are byte offsets. Each listed field is accessed as a 4-byte value.

| Object | Offset | Observed role |
|---|---:|---|
| Context | `+0x10` | Address of a root-holder object |
| Root holder | `+0x04` | Root-node address |
| Node | `+0x04` | Parent-like link |
| Node | `+0x08` | Left-child-like link |
| Node | `+0x0C` | Right-child-like link |

These are partial layouts, not complete object definitions. No field at offset zero is accessed. No payload, key, color, size, height, or other metadata is accessed.

For the remaining sections:

- **Pivot** means the node named by the second argument.
- **Promoted node** means the address initially read from the pivot’s `+0x0C` field.
- **Transferred subtree** means the address read from the promoted node’s `+0x08` field.
- **Former parent** means the pivot’s parent-like link before the transformation.
- **Root slot** means the 4-byte field at offset `+0x04` in the root-holder object obtained through the context.

These labels describe roles, not additional stored state.

### Preconditions

For the intended tree operation:

1. The pivot and promoted node must identify valid, writable node storage.
2. The pivot’s right-child-like link must identify the promoted node; a zero value is not handled as an empty-child case.
3. The context’s `+0x10` field must identify a valid root holder with a readable root slot. That slot must also be writable when it equals the pivot.
4. If the transferred subtree is nonzero, its parent-like field must be writable.
5. If the root slot does not equal the pivot, the former parent must provide a readable left-child-like field and the selected incoming-link field must be writable.
6. Relevant addresses and field offsets must be valid in the execution environment.

A well-formed tree additionally requires consistent parent/child relationships and nonoverlapping structural objects. The function does not establish or check those conditions.

## Behavior

### Established access and mutation order

The following order is visible in the supplied representation. It describes ordinary execution through valid memory, not an atomic transaction.

1. **Select the promoted node.**  
   The pivot’s right-child-like field at `+0x0C` is read. That address is retained as the promoted node throughout the remaining operation.

2. **Replace the pivot’s right link.**  
   The promoted node’s left-child-like field at `+0x08` is read, and its 4-byte value is written into the pivot’s right-child-like field.

3. **Conditionally repair the transferred subtree’s parent link.**  
   The promoted node’s left-child-like field is read again and compared with zero.

   - If this read yields zero, no transferred-subtree parent write occurs.
   - If it is nonzero, that field is read again to obtain the destination node address. The pivot address is then written to that destination node’s parent-like field at `+0x04`.

   Thus, the transferred-subtree address is not captured once and reused for all three purposes. With stable, nonoverlapping memory, these reads produce the same value. The evidence does not justify assuming that equivalence in the presence of unusual aliasing or concurrent mutation.

4. **Assign the promoted node’s parent link.**  
   The pivot’s parent-like field at `+0x04` is read, and that value is written to the promoted node’s parent-like field.

5. **Locate and inspect the root slot.**  
   The context field at `+0x10` is read to obtain the root-holder address. The root holder’s `+0x04` field is then read and compared with the pivot address.

6. **Replace the incoming link to the pivot.**

   - **Root case:** If the root slot equals the pivot, the promoted node address is written to that root slot. No former-parent child-link test is performed.
   - **Non-root case:** Otherwise, the pivot’s parent-like field is read again to obtain a parent address. That parent’s left-child-like field at `+0x08` is read and compared with the pivot:
     - If equal, the promoted node address is written to the parent’s left-child-like field.
     - If unequal, the promoted node address is written to the parent’s right-child-like field at `+0x0C`.

   The final alternative is unconditional after the failed left-link comparison. There is no check that the parent’s existing right link actually equals the pivot.

7. **Attach the pivot beneath the promoted node.**  
   The pivot address is written to the promoted node’s left-child-like field.

8. **Complete the pivot’s parent link.**  
   The promoted node address is written to the pivot’s parent-like field.

The function then returns without an explicit result value.

### Result under stable, well-formed-tree assumptions

Assuming valid, nonoverlapping objects, stable memory during execution, and consistent incoming links, the completed transformation has these properties:

- The pivot’s right link names the transferred subtree.
- If the transferred subtree is nonzero, its parent link names the pivot.
- The promoted node’s parent link is the pivot’s former parent link.
- The root slot, or the appropriate child link in the former parent, names the promoted node instead of the pivot.
- The promoted node’s left link names the pivot.
- The pivot’s parent link names the promoted node.

The pivot’s left link and the promoted node’s right link are not written. No other subtree links are traversed.

For a valid binary-search-tree arrangement, this topology change preserves the relative in-order arrangement of the affected subtrees. That is a consequence of the inferred tree interpretation, not a property checked by this function: no keys are read or compared.

### Root determination and malformed inputs

Root handling is selected solely by comparing the root slot with the pivot address. It is not selected by testing whether the pivot’s parent link is zero or names a sentinel.

Consequently:

- A pivot recognized as the root does not require a dereference of its former parent. Its former parent-link value is still copied into the promoted node.
- A pivot not recognized as the root causes a former-parent dereference, even if its parent link is zero or inconsistent.
- A non-root pivot absent from both child links of its claimed parent still causes that parent’s right link to be overwritten, provided the accesses remain valid.
- A zero promoted-node address is not rejected before accesses through that address.

The function provides no diagnostic for any of these cases.

### Width and arithmetic

Structural values are read and written as complete 4-byte fields. The observed zero test is a full-width comparison; there is no masking, tagging operation, or truncation to a smaller field.

Address calculations add fixed byte offsets to address-bearing values. There is no element-count multiplication, allocation-size calculation, or numeric update of node metadata.

The supplied representation does not settle signed-overflow behavior or pointer-provenance rules for invalid address arithmetic. Acceptance therefore requires valid field addresses; it does not promise wraparound addressing.

## Side effects

### Write footprint

For stable, well-formed objects, execution performs five unconditional 4-byte writes:

- Pivot right link.
- Promoted-node parent link.
- Selected incoming link: either the root slot or one former-parent child link.
- Promoted-node left link.
- Pivot parent link.

A sixth write occurs when the transferred-subtree check is nonzero: its parent-like field is assigned the pivot address.

These are write operations, not necessarily changes in value. Aliasing can make distinct described writes target the same storage and invalidate the ordinary tree postconditions.

### Intermediate state

The transformation is not atomic. In particular:

- The pivot’s right link changes before the incoming link is redirected.
- The incoming link is redirected before the promoted node’s left link is set to the pivot.
- The pivot’s parent link is updated last.

Observers interleaved with these writes could see an incomplete topology. No lock, atomic primitive, publication barrier, or rollback mechanism is visible.

### Ownership and lifetime

No allocation or deallocation is visible. The function stores existing address values into existing storage; it does not visibly create, destroy, or clear a node.

Caller-side ownership, reference counting, garbage-collector interaction, and lifetime guarantees are unknown. The observed link replacement must not be interpreted as an ownership release.

### Errors and partial mutation

There is no explicit error branch or status report. Invalid memory can be encountered after earlier writes have occurred. For example, the pivot’s right link is already overwritten before the context’s root-holder link is read.

The exact consequence of an invalid access is execution-environment-dependent. No recovery or restoration is established.

## Dependencies

No function calls appear in the supplied body. No additional opaque function IDs are therefore required for direct dependencies.

The operation depends on caller-provided memory, valid object lifetimes, and the structural preconditions above. The context’s indirect root-holder reference is the only externally rooted state accessed; no named global is accessed directly.

## Acceptance examples

The following are conditional specification examples, not executed tests. They assume stable memory, valid 4-byte field accesses, and distinct objects unless stated otherwise.

### Root pivot with an empty transferred subtree

Initially:

- The root slot names the pivot.
- The pivot’s right link names the promoted node.
- The promoted node’s left link is zero.
- The pivot’s parent link contains an existing marker value.

Expected outcome:

- The root slot names the promoted node.
- The promoted node’s parent link contains the marker value.
- The promoted node’s left link names the pivot.
- The pivot’s parent link names the promoted node.
- The pivot’s right link is zero.

No transferred-subtree parent write occurs. The marker value is copied, not dereferenced as a parent in this branch.

### Root pivot with a nonempty transferred subtree

Initially, the root slot names the pivot, and the promoted node’s left link names a valid subtree node.

Expected outcome includes the root transformation above, except that:

- The pivot’s right link names that subtree node.
- That subtree node’s parent link names the pivot.

The subtree’s child links are not changed.

### Non-root pivot occupying its parent’s left link

Initially:

- The root slot does not name the pivot.
- The pivot’s parent link names a valid parent.
- That parent’s left link names the pivot.

Expected outcome:

- The parent’s left link names the promoted node.
- The parent’s right link is not written.
- The promoted node’s parent link names the parent.
- The pivot becomes the promoted node’s left child.

The root slot is read but not written.

### Non-root pivot occupying its parent’s right link

Initially, the root slot does not name the pivot, the former parent’s left link does not name the pivot, and its right link does.

Expected outcome:

- The former parent’s right link names the promoted node.
- Its left link is unchanged.
- The remaining local links have the rotation outcome described above.

### Inconsistent former-parent relationship

Initially, the root slot does not name the pivot, and neither child link of the pivot’s claimed parent names the pivot.

Provided all accesses remain valid, the parent’s right link is nevertheless replaced with the promoted node. Rejection, diagnosis, or preservation of that existing right link is not an acceptance requirement.

### Missing right child

If the pivot’s right link is zero, successful rotation is not guaranteed. The function attempts to access the promoted-node fields without a null guard. A no-op or clean error return is not specified.

## Uncertainties

The following remain **documented_only**:

- Complete object layouts, allocation sizes, field alignment, and native calling convention.
- Whether the root holder is a sentinel node, container header, or another structure.
- Whether parent links use zero, a sentinel address, or another marker at the root.
- Caller guarantees concerning topology, aliasing, synchronization, and memory lifetime.
- Whether child links may contain tagged values. No tag handling is visible.
- Execution behavior for invalid addresses, offset overflow, inaccessible memory, or concurrent mutation.
- Any requirement to update balancing metadata elsewhere.
- Ownership semantics beyond the absence of visible allocation and release.
- Formal language-level sequencing guarantees beyond the access order expressed by the supplied representation.

No runtime tests have been performed.

## Evidence

**Established static evidence:** The supplied body contains 4-byte field accesses, a zero test on the promoted node’s left-like link, a root-slot equality test, a parent-left-link equality test, the ordered writes described above, and no function calls or explicit result value.

**Inferred intent:** The offsets and completed relationships match a left rotation of a binary-tree node. Parent, left-child, right-child, and root terminology are semantic labels for that interpretation.

**Not established:** Tree validity on entry, balancing correctness, ownership transfer, concurrency safety, invalid-input recovery, and execution-environment details. These are not supplied as guarantees by this contract.
