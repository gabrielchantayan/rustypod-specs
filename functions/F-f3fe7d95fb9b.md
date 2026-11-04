<!-- Trusted provenance: F-f3fe7d95fb9b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f3fe7d95fb9b

## Purpose

F-f3fe7d95fb9b performs an in-place pointer rearrangement consistent with a left rotation of a parent-linked binary tree.

The established behavior is narrower than that interpretation: the function selects the object referenced at offset `0x0C` of its second input, transfers that object’s offset-`0x08` link to the second input, reconnects the selected object through either a root slot or an existing parent, and finally links the second input beneath the selected object.

The tree interpretation is inferred from these accesses and writes. The supplied material does not establish a particular tree implementation, balancing policy, sentinel convention, or validity checker. There is no visible key comparison, balancing-metadata update, allocation, deallocation, or call to another function.

## Interface

The function accepts two inputs and has no return value:

- **Container address:** an address used to locate a root-holding object through a four-byte field at offset `0x10`.
- **Pivot address:** the address of the object whose links are rearranged.

The supplied representation uses 32-bit integer values for the inputs and four-byte fields for the relevant stored addresses. The contract therefore describes four-byte address-bearing fields, not native-width pointers on an arbitrary host.

The accessed layout is:

| Object role | Offset | Width | Established use |
|---|---:|---:|---|
| Container | `0x10` | 4 bytes | Address of a root-holding object |
| Root-holding object | `0x04` | 4 bytes | Root slot compared against the pivot address and conditionally replaced |
| Pivot or related node | `0x04` | 4 bytes | Parent-like link |
| Pivot or related node | `0x08` | 4 bytes | Left-child-like link |
| Pivot or related node | `0x0C` | 4 bytes | Right-child-like link |

The role names describe the apparent structure; they do not establish an external type definition. No behavior is shown for offset `0x00`, payload fields, or any other metadata.

For a conventional tree rotation, the expected preconditions are:

- The container and pivot designate valid objects with the required readable and writable fields.
- The pivot’s offset-`0x0C` field designates a valid node to promote.
- The root-holding object is accessible through the container.
- If the pivot is not the recorded root, its parent-like link designates a valid object whose appropriate child slot refers to the pivot.
- Any nonzero transferred-subtree address designates an object with a writable parent-like field.
- The relevant objects and fields do not overlap in ways that invalidate the tree interpretation.
- No concurrent actor changes the relevant links during the operation.

These are structural preconditions for the intended result, not conditions checked by the function.

## Behavior

### Selection and subtree transfer

The function first reads the pivot’s four-byte field at offset `0x0C`. That address selects the node to be promoted and is retained for subsequent accesses.

It then reads the promoted node’s field at offset `0x08` and writes that value to the pivot’s field at offset `0x0C`. Under the tree interpretation, the promoted node’s left subtree becomes the pivot’s right subtree.

The promoted node’s offset-`0x08` field is then read again for a zero comparison. This is a separate read; the earlier value is not simply reused.

If this comparison is nonzero, the function reads the same field once more to obtain the destination object address, then writes the pivot address to that object’s offset-`0x04` field. Under ordinary nonoverlapping, stable-memory conditions, this repairs the transferred subtree’s parent link.

The distinction between these reads matters for aliasing or concurrent modification. The value written into the pivot’s child slot, the value tested for zero, and the address used for the parent-link write are obtained separately. The supplied representation does not guarantee a single snapshot across them.

### Parent-link preparation and root decision

After the subtree transfer and its conditional parent-link repair, the function reads the pivot’s offset-`0x04` field and writes that value to the promoted node’s offset-`0x04` field.

It next reads the container’s offset-`0x10` field to locate the root-holding object. The root slot is the four-byte field at offset `0x04` within that object.

The function reads this root slot and compares it with the pivot address:

- **If equal:** it replaces the root slot with the promoted-node address.
- **If unequal:** it reconnects the promoted node through the pivot’s parent-like link, as described below.

The root decision occurs after the earlier link writes. It is not an entry-time determination that the pivot was the root.

### Non-root reconnection

On the non-root branch, the function reads the pivot’s offset-`0x04` field again. It does not reuse the parent value previously copied into the promoted node.

Using this newly read parent address, it reads the parent’s offset-`0x08` field:

- If that field equals the pivot address, the function replaces it with the promoted-node address.
- Otherwise, the function writes the promoted-node address to the parent’s offset-`0x0C` field.

The second case does not first verify that the parent’s offset-`0x0C` field actually points to the pivot. An inconsistent parent relationship therefore does not produce a reported validation error; the function proceeds with the right-slot write.

### Final linkage

After either the root-slot update or the non-root reconnection, the function performs two final writes in this order:

1. The promoted node’s offset-`0x08` field receives the pivot address.
2. The pivot’s offset-`0x04` field receives the promoted-node address.

It then returns without a value.

For a valid, stable, nonoverlapping tree, the resulting relationships are:

- The promoted node occupies the pivot’s former position relative to its parent or the root slot.
- The pivot becomes the promoted node’s left child.
- The promoted node’s former left subtree becomes the pivot’s right subtree.
- The promoted node receives the pivot’s former parent.
- The transferred subtree, if nonzero, receives the pivot as its parent.

The promoted node’s right-child-like field and the pivot’s left-child-like field are not directly modified.

## Side effects

All visible effects are reads and writes through the supplied addresses and addresses reached from them. No explicit allocation, release, logging, callback, or unrelated global access appears.

For distinct objects in a valid tree, the write order is:

1. Replace the pivot’s right-child-like link.
2. If the separately read transferred-subtree link is nonzero, update the addressed object’s parent-like link.
3. Set the promoted node’s parent-like link.
4. Replace either the root slot or one child slot of the pivot’s parent.
5. Set the promoted node’s left-child-like link to the pivot.
6. Set the pivot’s parent-like link to the promoted node.

There are five explicit four-byte writes when the conditional subtree-parent write is skipped, and six when it is taken. Aliasing may cause multiple writes to affect the same storage.

The operation is not presented as atomic. Its intermediate states need not satisfy normal bidirectional tree-link invariants. No lock, synchronization primitive, rollback, or recovery mechanism is visible. If an access fails after earlier writes, the supplied material provides no mechanism to restore the previous links.

No ownership transfer is established beyond pointer rearrangement. The function does not visibly create or destroy nodes. Whether an external ownership discipline permits this rearrangement is outside the available evidence.

The source-level order above is established by the supplied representation. Hardware memory ordering, compiler transformations, and behavior under data races are not established.

## Dependencies

No calls to other functions are present. There are therefore no identified callee dependencies to express using opaque function IDs.

The operation depends on the caller supplying accessible storage and the expected address-bearing layout. It also depends on external structural consistency if its result is to be interpreted as a valid tree rotation.

No assumptions about unknown callees preserving globals, retaining memory, or releasing memory are needed: no such callee appears in the supplied material.

## Acceptance examples

These examples are conditional behavioral expectations derived from static evidence. They have not been executed. Unless stated otherwise, assume valid four-byte address representations, distinct objects, no concurrent modification, and consistent parent and child links.

### Pivot is the recorded root, with a nonzero transferred subtree

Initially:

- The root slot refers to the pivot.
- The pivot’s right child is the promoted node.
- The promoted node’s left child is a separate subtree node.

Expected completion:

- The root slot refers to the promoted node.
- The promoted node’s parent field contains the pivot’s former parent value.
- The promoted node’s left child is the pivot.
- The pivot’s parent is the promoted node.
- The pivot’s right child is the transferred subtree.
- The transferred subtree’s parent is the pivot.

The pivot’s former parent value is copied as stored; the function does not require it to be zero on the root branch.

### Pivot is its parent’s left child

Initially, the root slot does not refer to the pivot, and the pivot’s parent has the pivot in its offset-`0x08` field.

Expected completion:

- That parent’s offset-`0x08` field refers to the promoted node.
- The parent’s offset-`0x0C` field is not directly written.
- The promoted node’s parent is the pivot’s former parent.
- The remaining links follow the rotation relationships described above.

The root slot is read but is not written on this branch.

### Pivot is its parent’s right child

Initially, the root slot does not refer to the pivot, the parent’s offset-`0x08` field does not equal the pivot, and its offset-`0x0C` field does equal the pivot.

Expected completion: the parent’s offset-`0x0C` field is replaced with the promoted-node address. The parent’s offset-`0x08` field remains directly unchanged.

### No transferred subtree

Initially, the promoted node’s offset-`0x08` field is zero and remains zero until the final linkage.

Expected completion:

- The pivot’s offset-`0x0C` field becomes zero.
- The conditional subtree-parent write is skipped.
- The promoted node’s offset-`0x08` field ultimately becomes the pivot address.

A zero transferred-subtree link is explicitly handled. A zero promoted-node address is not.

### Inconsistent parent relationship

Initially, the pivot is not the recorded root, and neither child slot of its recorded parent refers to it.

Provided all accesses remain valid, the function still overwrites that parent’s offset-`0x0C` field with the promoted-node address. It does not report the inconsistency. This example specifies observable behavior, not a valid-use recommendation.

### Missing promoted node

If the pivot’s offset-`0x0C` field is zero, the next relevant operation attempts to read the four-byte field at offset `0x08` relative to that address.

There is no explicit null check, early return, or error result. The environment’s handling of that access is unknown; successful rotation is not an acceptance condition for this input.

## Uncertainties

- **Documented only — structural identity:** The links strongly suggest a left rotation, but the exact data structure and sentinel conventions are not established.
- **Documented only — address arithmetic:** Four-byte stored values and fixed byte offsets are visible. Address-space limits, overflow behavior, alignment requirements, and treatment of invalid addresses are not supplied. No wraparound behavior is promised.
- **Documented only — byte order:** Endianness is not established.
- **Documented only — aliasing:** Nonstandard overlap among nodes, the container, and the root-holding object can make earlier writes affect later reads. Ordinary rotation postconditions do not cover such arrangements.
- **Documented only — concurrency:** Repeated reads can differ if storage changes externally. No synchronization contract is visible.
- **Documented only — failures:** No explicit error channel or recovery path appears. Fault behavior and externally installed handlers are unspecified.
- **Documented only — ownership and lifetime:** Required object lifetimes and any external ownership rules are unavailable.
- **Inferred intent:** The operation is intended to rotate a valid tree without changing payloads or tree membership. That interpretation is not an independently verified caller contract.

## Evidence

The established evidence is the supplied static function representation: two address-bearing inputs, four-byte field operations, the ordered link updates, root comparison, parent-child branch, and absence of function calls or explicit error returns.

The left-rotation interpretation follows from the relationship among those fields. Broader claims about balancing, ordering invariants, sentinel identity, ownership, platform fault behavior, or caller correctness are not established.

No runtime tests have been performed. The acceptance examples are conditional specifications derived from the static evidence, not observed execution results.
