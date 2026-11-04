<!-- Trusted provenance: F-63a2b3ccf83b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-63a2b3ccf83b

## Purpose

F-63a2b3ccf83b rearranges links between a selected node, its linked successor, and their surrounding structure. It also replaces a designated root reference when that reference identifies the selected node.

**Inferred intent:** The field relationships match a right rotation in a parent-linked binary tree: the selected node’s left child is promoted, that child’s right subtree moves beneath the selected node, and the selected node becomes the promoted node’s right child.

**Established behavior:** The function performs the reads, comparisons, and four-byte writes described below. It does not check that the supplied structure is a tree, that parent and child links agree, or that the required addresses are valid. Tree terminology is therefore explanatory, not an independently enforced property.

## Interface

The function receives two machine-level values:

| Input | Interpretation supported by the supplied source |
|---|---|
| First argument | Address of a context containing a four-byte reference at byte offset 16. |
| Second argument | Address of the selected node whose links will be rearranged. |

The displayed interface has no return value. No explicit status, error code, or result object is produced.

The following names describe roles in this contract, not identifiers from the supplied source:

- **Selected node:** the node addressed by the second argument.
- **Promoted node:** the address initially read from the selected node at offset 8.
- **Context anchor:** the address subsequently read from the first argument at offset 16.
- **Root slot:** the four-byte location at offset 4 from the context anchor.

The accessed layouts are:

| Object | Byte offset | Accessed width | Observed role |
|---|---:|---:|---|
| Context | 16 | 4 bytes | Reference used to locate the context anchor |
| Context anchor | 4 | 4 bytes | Root slot |
| Node | 4 | 4 bytes | Parent reference |
| Node | 8 | 4 bytes | Left-child reference |
| Node | 12 | 4 bytes | Right-child reference |

These are partial layouts. The source does not establish total object sizes, allocation boundaries, alignment requirements, byte order, or the meaning of other fields.

The source represents addresses and links using 32-bit quantities. Link copying and comparisons concern the complete four-byte values; no tag masking, truncation to a smaller field, or pointer normalization is visible. Byte offsets are address displacements, not indices into an array of four-byte elements.

### Preconditions for ordinary successful use

The caller must provide accessible storage for every location reached by the executed branch. In particular:

- The selected node must support accesses at offsets 4 and 8.
- Its initial offset-8 reference must identify storage supporting accesses at offsets 4 and 12. A zero value is not accepted through an explicit guard.
- The context must support a read at offset 16, and the resulting anchor must support access to its offset-4 root slot.
- If the promoted node’s tested offset-12 reference is nonzero, the destination reached through the subsequent read of that field must support a write at offset 4.
- If the root slot does not equal the selected node’s address, the selected node’s then-current parent reference must identify storage supporting the required child-link access.

For the tree-rotation interpretation, callers additionally need a consistent, non-overlapping tree structure and exclusive access during the operation. Those conditions are not validated by this function.

## Behavior

### Ordered memory effects

The ordering below follows the supplied source. It describes source-level accesses, not a guarantee of processor-level atomicity, synchronization, or instruction scheduling.

1. **Capture the promoted node.**  
   Read the four-byte reference at offset 8 of the selected node. Retain that address as the promoted node for the remainder of the operation.

2. **Replace the selected node’s left-child reference.**  
   Read four bytes at offset 12 of the promoted node. Write that value to offset 8 of the selected node.

3. **Conditionally repair the transferred node’s parent reference.**  
   Read the promoted node’s offset-12 field again and compare it with zero.

   If this read is zero, perform no transferred-node parent write.

   If it is nonzero, read the promoted node’s offset-12 field once more to obtain the destination address. Write the selected node’s address at offset 4 of that destination.

   Thus, the value copied into the selected node, the value tested for zero, and the destination reference used by the conditional write come from distinct reads. They coincide for ordinary stable, non-overlapping storage, but the function does not explicitly enforce that agreement.

4. **Give the promoted node the selected node’s parent reference.**  
   Read the selected node’s offset-4 field. Write those four bytes at offset 4 of the promoted node.

5. **Locate the root slot.**  
   Read the context’s offset-16 reference. Use the resulting address plus 4 as the root-slot location. Read the root slot and compare it with the selected node’s address.

6. **Replace the incoming reference to the selected node.**

   - **Root match:** If the root slot equals the selected node’s address, write the promoted node’s address into that root slot.
   - **No root match:** Read the selected node’s offset-4 parent reference again. From that parent, read the offset-12 child reference.
     - If that reference equals the selected node’s address, write the promoted node’s address at offset 12 of that parent.
     - Otherwise, write the promoted node’s address at offset 8 of that parent.

   The non-root fallback does **not** verify that the parent’s offset-8 field previously referred to the selected node. A mismatch at offset 12 is sufficient to select the offset-8 write.

7. **Attach the selected node beneath the promoted node.**  
   Write the selected node’s address at offset 12 of the promoted node.

8. **Finish the selected node’s parent update.**  
   Write the promoted node’s address at offset 4 of the selected node.

The function then returns without a value.

### Result under consistent tree preconditions

For distinct, stable objects with consistent links:

- The promoted node replaces the selected node at its former position.
- The selected node’s former parent becomes the promoted node’s parent.
- The promoted node’s former right child becomes the selected node’s left child.
- If that transferred child is nonzero, its parent becomes the selected node.
- The selected node becomes the promoted node’s right child.
- The selected node’s parent becomes the promoted node.
- The root slot changes only when its comparison identifies the selected node as root.

No explicit write changes the promoted node’s left-child field or the selected node’s right-child field. With non-overlapping objects, those fields remain unchanged.

### Branch priority and malformed structures

Root-slot equality takes precedence over inspection of the selected node’s parent’s child fields. On the root branch, those child fields are not examined.

On the non-root branch, the parent’s offset-12 link takes precedence. If both parent child links identify the selected node, only offset 12 is replaced.

The implementation does not reject cycles, self-links, shared children, contradictory parent references, or overlapping objects. Such inputs can make earlier writes affect later reads. For those cases, the ordered effects above are authoritative; the normal rotation result is not promised.

## Side effects

For ordinary non-overlapping storage, the function makes five unconditional four-byte writes:

1. Selected node, offset 8.
2. Promoted node, offset 4.
3. Either the root slot or one child-link field in the selected node’s parent.
4. Promoted node, offset 12.
5. Selected node, offset 4.

A sixth four-byte write occurs when the promoted node’s offset-12 field passes the nonzero test: the reached node’s offset-4 parent field is set to the selected node’s address.

The operation exposes intermediate states. For example, the selected node’s left-child link changes before the transferred child’s parent is repaired, and the incoming parent or root link changes before the last two writes complete the rotation. No lock, transaction, rollback, or atomic publication mechanism is visible.

There is no explicit allocation, deallocation, ownership transfer, payload copying, logging, or external I/O. Existing references are rearranged. The source does not establish a broader ownership policy or prove that storage remains alive.

Invalid accesses have no explicit recovery path. If execution faults after an earlier write, the source provides no mechanism to restore the prior structure. Exact fault behavior is environment-dependent.

## Dependencies

No direct function calls are present in F-63a2b3ccf83b. Accordingly, there are no callee contracts or additional opaque function IDs to specify.

The operation depends on:

- Caller-provided storage and reference validity.
- The target’s representation of four-byte references and zero.
- The memory environment’s access, alignment, and fault semantics.
- Caller-managed synchronization where concurrent access is possible.

No assumption about an unknown callee preserving globals or managing memory is necessary. The function contains no visible named-global access, but reachable storage may itself be shared or globally accessible.

## Acceptance examples

These are conditional contract examples, not runtime test results. Unless stated otherwise, assume distinct accessible objects, stable memory, and consistent links.

### Root rotation with a nonzero transferred child

Initially:

- The root slot identifies the selected node.
- The selected node’s left child is the promoted node.
- The promoted node’s right child is a third node.
- The selected node’s parent contains some four-byte reference.

After normal completion:

- The root slot identifies the promoted node.
- The promoted node’s parent contains the selected node’s former parent reference.
- The selected node’s left child identifies the third node.
- The third node’s parent identifies the selected node.
- The promoted node’s right child identifies the selected node.
- The selected node’s parent identifies the promoted node.

The parent reference copied into the promoted node is not automatically cleared merely because this is the root branch.

### Root rotation with no transferred child

If the promoted node’s right-child reference remains zero through the relevant reads:

- The selected node’s left-child reference becomes zero.
- No transferred-child parent write occurs.
- The root replacement and final attachment writes still occur.

This zero handling applies to the transferred child, not to the promoted node itself.

### Non-root replacement through the parent’s right link

If the root slot does not identify the selected node and the selected node’s parent has its offset-12 link equal to the selected node:

- That parent’s offset-12 link becomes the promoted node.
- Its offset-8 link is not explicitly written.
- The root slot is not written.

### Non-root fallback through the parent’s left link

If the root slot does not identify the selected node and the parent’s offset-12 link differs from the selected node:

- The parent’s offset-8 link is overwritten with the promoted node.
- This occurs even if that offset-8 link did not previously identify the selected node.

A contract requiring rejection of this inconsistency would not match the supplied behavior.

### Missing promoted node

If the selected node’s initial offset-8 value is zero, there is no early return or explicit error result. Execution proceeds to an access at offset 12 relative to that value. Whether that access faults depends on the environment; successful tree behavior is not established for this input.

## Uncertainties

- **Inferred intent:** Parent, left-child, right-child, and right-rotation terminology follows the link relationships. No external type or API declaration confirms those names.
- **Documented only:** Exact pointer ABI, endianness, alignment rules, and address-overflow behavior are not supplied. The visible fields are four bytes wide; behavior at invalid or wrapping addresses is not established.
- **Documented only:** No allocator, ownership, lifetime, sentinel-node, or tree-balancing policy is provided.
- **Documented only:** Concurrent mutation, memory-mapped storage, and aliases can make repeated reads differ. No synchronization or general correctness guarantee for those cases is visible.
- **Documented only:** Object overlap can cause a write described for one role to modify another role’s fields. No non-alias validation is present.
- **Documented only:** The context anchor may be a header, sentinel, or another object. Only its use to locate the root slot is established.
- **Documented only:** Fault delivery and partial-update consequences depend on the execution environment.

## Evidence

The supplied static source establishes the accessed offsets and four-byte widths, the initial promoted-node capture, the repeated reads of its offset-12 field, the zero-controlled parent repair, root-slot comparison, right-link-first non-root branch, unconditional left-link fallback, and final two attachment writes.

It also establishes the absence of explicit calls, validation, allocation, release, rollback, and a return value within this function.

The rotation interpretation and ordinary postconditions are deductions under the stated consistency and non-overlap preconditions. No runtime tests have been performed.
