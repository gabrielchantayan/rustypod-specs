<!-- Trusted provenance: F-bd7d3aa9e3fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd7d3aa9e3fb

## Purpose

F-bd7d3aa9e3fb rewires linked records in place. For valid, suitably distinct binary-tree records, its effect is a left rotation around the node supplied as the second argument:

- That node’s right child takes its position.
- The promoted node’s former left child becomes the original node’s right child.
- The original node becomes the promoted node’s left child.
- A distinguished root link or a child link in the original parent is redirected to the promoted node.

The tree interpretation is inferred from the field accesses and mutations. The established behavior is the ordered transfer and comparison of four-byte link values. The function does not establish that its inputs form a tree, inspect keys, validate parent-child consistency, or update balancing metadata.

Ordinary rotation postconditions require additional structural and memory-validity conditions. For overlapping or malformed records, the ordered accesses below govern instead.

## Interface

### Arguments and return

The function accepts two address-bearing arguments:

| Argument | Contract role |
|---|---|
| First | Address of a context record containing a link to a root-link holder. |
| Second | Address of the original node, whose right-child link supplies the promoted node. |

There is no return value and no returned success or error status. Results are represented by memory mutations.

The supplied representation expresses addresses through integer-typed values and uses four-byte link transfers. This supports a 32-bit link-value interpretation, but does not independently establish the complete ABI, native pointer width, integer-to-pointer conversion rules, or valid address range.

### Accessible layout

The following are descriptive roles, not declarations of complete record types.

| Record role | Byte offset | Access and meaning |
|---|---:|---|
| Context | +16 | Read to obtain the root-link holder’s address. |
| Root-link holder | +4 | Read as the distinguished root link; written on the root-replacement branch. |
| Node | +4 | Parent link. |
| Node | +8 | Left-child link. |
| Node | +12 | Right-child link. |

All listed link transfers are four bytes. Offsets are byte displacements; they are not element counts.

No meaning is established for offset zero or for other fields. Complete record sizes, alignment requirements, allocation boundaries, and payload layout are unknown. Interpreting the root-link holder as a header or sentinel is plausible but not required by the observed accesses.

### Preconditions

For successful completion, every selected read and write must address accessible storage with the permissions required by the execution environment.

For a conventional left rotation, stronger conditions apply:

- The original node is accessible and writable at the affected fields.
- Its right-child link identifies an accessible, nonzero promoted node.
- The promoted node’s left-child link is either zero or identifies a node with a writable parent field.
- The context identifies an accessible root-link holder.
- The root link is readable and, when selected for replacement, writable.
- On a non-root branch, the original node’s parent identifies an accessible record whose left-child field is readable and whose selected child field is writable.
- Participating storage remains valid during execution.
- Records do not overlap in ways that change later accesses through earlier writes.
- The relationships form the intended tree, and concurrent activity does not interfere with them.

The function checks none of these conditions comprehensively. A nonzero link is not validated for type, ownership, lifetime, alignment, or accessible extent.

## Behavior

### Terminology and retained values

The **original node** is the second argument.

The **promoted node** is the address read initially from the original node’s field at +12. That address is retained for subsequent promoted-node accesses; later changes to the original node’s right-child field do not replace it.

The **transferred subtree** is the conventional interpretation of the promoted node’s initial left-child link. That field is read multiple times rather than treated as one immutable snapshot.

The **root slot** is the field at +4 in the holder obtained through the context’s field at +16.

### Ordered reads and writes

The following order is established by the supplied static representation. It describes the represented accesses, not a separately verified machine-level concurrency or memory-ordering guarantee.

1. **Select the promoted node.**  
   Read four bytes from the original node at +12 and retain the resulting address.

2. **Replace the original right-child link.**  
   Read four bytes from the promoted node at +8. Write those bytes into the original node at +12.

3. **Conditionally repair the transferred subtree’s parent.**  
   Read the promoted node’s field at +8 again and compare it with zero.

   If that read is nonzero, read the same field once more to obtain a destination address, then write the original-node address into that destination’s field at +4.

   If the comparison read is zero, omit this write.

   The tested value and the value used as the destination come from separate reads. The function does not ensure that they are identical if memory changes between them.

4. **Copy the original parent link.**  
   Read four bytes from the original node at +4 and write them into the promoted node at +4.

   This copies the field’s value at this stage, after the preceding mutations. It is the entry-time parent value only when those mutations and external activity have not changed it.

5. **Locate and inspect the root slot.**  
   Read the context’s field at +16, calculate the holder’s field address at +4, and read that slot. Compare its contents with the original-node address.

   - If equal, write the promoted-node address into that same root slot.
   - If unequal, read the original node’s parent field at +4 again. Use that address to read the parent’s left-child field at +8.
     - If the left-child field equals the original-node address, write the promoted-node address into the parent’s field at +8.
     - Otherwise, write the promoted-node address into the parent’s field at +12.

   The parent address selected on this non-root branch is used for the branch’s child-field write. The function does not read or validate the existing right-child value before overwriting it.

6. **Attach the original node beneath the promoted node.**  
   Write the original-node address into the promoted node at +8.

7. **Set the original node’s parent.**  
   Write the promoted-node address into the original node at +4.

The function then returns without a value.

The context lookup occurs after the transferred-subtree handling and promoted-parent assignment. Root detection therefore uses the root slot’s contents at that later stage, not a root value captured at entry.

### Result for a valid, stable tree

With suitably distinct records and consistent parent-child relationships, successful completion produces:

| Field | Final value |
|---|---|
| Original node’s right child | Promoted node’s former left child. |
| Transferred subtree’s parent, if nonzero | Original node. |
| Promoted node’s parent | Original node’s former parent. |
| Root link, if it identified the original node | Promoted node. |
| Original parent’s selected child link, otherwise | Promoted node. |
| Promoted node’s left child | Original node. |
| Original node’s parent | Promoted node. |

The original node’s left-child field and promoted node’s right-child field are not directly selected for writing. Under non-overlapping storage, they remain unchanged.

For a valid binary tree, this restructuring preserves relative in-order placement. That is a consequence of the inferred rotation structure, not an invariant checked by the function.

### Branch rules and malformed inputs

Root replacement depends only on equality between the inspected root link and the original-node address. A null parent, sentinel parent, or flag does not independently select that branch.

On the non-root branch, equality with the parent’s left-child link selects the left-child update. Every other value selects the right-child update. The function does not require the parent’s right child to identify the original node.

Zero receives special handling only in the promoted-left test. There is no corresponding guard for the original node, promoted node, context, root-link holder, or non-root parent. An invalid value in any of those roles is used in subsequent address calculations and accesses.

### Widths and address arithmetic

The link copies and comparisons operate on the represented four-byte values. No widening, truncation, sign-extension policy, or portable pointer interpretation beyond the supplied representation is established.

Address formation uses the fixed byte offsets +4, +8, +12, and +16. The function contains no explicit range or overflow checks. Behavior when address arithmetic overflows, wraps, or produces an invalid address is documented_only; portable wrapping behavior must not be assumed.

## Side effects

### Writes

A completed path performs five unconditional four-byte write operations:

1. Original node’s right-child field.
2. Promoted node’s parent field.
3. The root slot or one child field of the selected parent.
4. Promoted node’s left-child field.
5. Original node’s parent field.

A sixth four-byte write occurs when the promoted-left comparison is nonzero: the destination node’s parent field is set to the original-node address.

These counts describe write operations, not necessarily distinct destinations. Aliasing may cause multiple writes to affect the same bytes or fields with other nominal roles.

No direct payload, count, balancing, or other metadata update is shown.

### Ownership and lifetime

There are no function calls and no explicit allocation or deallocation operations. The function rearranges stored address values.

No ownership transfer, reference-count adjustment, destructor action, or lifetime extension is established. Any surrounding ownership policy is documented_only. Accessible storage must remain valid for the accesses actually performed.

### Partial state and concurrency

Writes occur incrementally. No rollback, lock, transaction, or explicit synchronization operation is shown.

If execution is interrupted or an access fails after a write, earlier mutations are not explicitly undone. The precise failure mechanism and resulting runtime behavior are unknown.

The operation must not be specified as atomic for concurrent observers. Even for valid records, intermediate links need not satisfy the final parent-child relationships. Whether unsynchronized concurrent access is permitted at all depends on runtime rules not established here.

## Dependencies

No external function calls are present, so there are no dependency function IDs to enumerate.

Direct requirements are:

- The accessible field layout described above.
- A representation supporting the observed four-byte reads, writes, and equality comparisons.
- Runtime support for the selected address calculations and memory accesses.
- Storage validity throughout execution.

There are no observed callees whose global-state, ownership, or memory-release behavior must be assumed.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Except where explicitly discussing aliasing or invalid input, assume valid, stable, suitably distinct storage and address calculations supported by the execution environment.

### Root replacement with no transferred subtree

Initially:

- The root link identifies node A.
- A’s right child is B.
- B’s left child is zero.
- A’s parent field contains H.

After completion:

- The root link identifies B.
- B’s parent field contains H.
- B’s left child identifies A.
- A’s parent identifies B.
- A’s right child is zero.

No transferred-subtree parent write occurs. A’s left child and B’s right child remain unchanged. H need not be interpreted as a particular sentinel type; its stored value is copied.

### Root replacement with a transferred subtree

Initially, the root link identifies A, A’s right child is B, and B’s left child is a nonzero accessible node C.

After completion:

- A’s right child identifies C.
- C’s parent identifies A.
- B’s parent contains A’s former parent-link value.
- The root link identifies B.
- B’s left child identifies A.
- A’s parent identifies B.

C’s parent is updated before the root slot is replaced.

### Non-root left-child replacement

Initially, the root link does not identify A, A’s parent is P, P’s left child identifies A, and A’s right child is B.

After completion, P’s left child identifies B and B’s parent identifies P. The other affected links follow the normal rotation result. P’s right-child field is not selected for writing.

### Non-root right-child replacement

Initially, the root link does not identify A, A’s parent is P, P’s left child does not identify A, and P’s right child identifies A.

After completion, P’s right child identifies the promoted node. The left-child field is read for branch selection but not written.

### Inconsistent parent-child relationship

Initially, the root link does not identify A, A’s parent is P, and neither child field of P identifies A.

If all accesses remain valid, the function nevertheless replaces P’s right-child link with the promoted-node address. It returns no inconsistency indication. Preservation of the unrelated right-child link is not an acceptance condition.

### Missing promoted node

If A’s right-child field is zero, the function does not select a no-op or return an error status. It proceeds to access a field at offset +8 from that value.

The resulting fault, invalid-access behavior, or other platform outcome is documented_only.

### Promoted node equals the original node

Assume the original node A initially has:

- A right-child link identifying A itself.
- A left-child link of zero.
- A parent-link value H.
- A root slot identifying A.

Also assume the context and root holder do not overlap A and that all selected accesses are supported.

The first write sets A’s field at +12 to zero. It does **not** change the subsequent promoted-left reads, which access A’s distinct field at +8. Those reads still obtain zero, so no transferred-subtree parent write occurs.

The promoted-parent copy writes A’s parent value back to the same field. Root replacement stores A into the root slot. The final writes set A’s left child and parent to A.

Thus the final state includes:

- Root link: A.
- A’s right child: zero.
- A’s left child: A.
- A’s parent: A.

This self-aliasing case does not satisfy ordinary rotation postconditions. In particular, the final parent field is A rather than a preserved H when H differs from A. The failure of ordinary postconditions is not caused by the first write changing the promoted-left field.

Other overlapping layouts can make an earlier write affect a later read, but require separate address-specific analysis under the unresolved address-arithmetic rules.

## Uncertainties

### Established static evidence

The supplied representation establishes:

- The listed field offsets and four-byte transfers.
- The described ordering of represented reads and writes.
- Repeated promoted-left reads.
- A later reread of the original parent on the non-root branch.
- Equality-based root selection.
- Left-child comparison with an unconditional right-child fallback.
- No returned status and no explicit validation, allocation, deallocation, calls, rollback, or synchronization.

### Inferred intent

A binary-tree left rotation is strongly supported by the non-overlapping, structurally valid case. A header or sentinel role for the root-link holder is plausible.

Neither inference establishes complete tree invariants, ownership rules, balancing policy, or caller obligations beyond those needed for the described result.

### Documented-only unknowns

Unresolved matters include:

- Exact ABI, native pointer width, alignment, and signedness conventions.
- Address-conversion and address-overflow behavior.
- Runtime consequences of invalid, inaccessible, or unaligned accesses.
- Caller enforcement of tree validity and non-overlap.
- External synchronization.
- Allocation, ownership, and lifetime policy.
- Metadata maintained elsewhere.
- Whether volatile or device-backed storage is permitted.
- Machine-level access ordering beyond the supplied static representation.

No runtime outcome is invented to resolve these unknowns.

## Evidence

This contract is derived from the supplied static representation of F-bd7d3aa9e3fb.

The represented accesses support the field layout, mutation order, branch conditions, write counts, and conditional examples. The self-aliasing example distinguishes the first destination at +12 from subsequent promoted-left reads at +8; it does not attribute an overwrite to distinct fields.

Tree terminology expresses inferred structural intent rather than independently established surrounding invariants.

No runtime tests, implementation execution, surrounding-source inspection, or external dependency inspection have been performed.
