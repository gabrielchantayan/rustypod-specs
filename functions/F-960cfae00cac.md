<!-- Trusted provenance: F-960cfae00cac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-960cfae00cac

## Purpose

F-960cfae00cac rewires a small set of 32-bit links associated with a context and a node.

**Established static behavior:** the node initially referenced by the supplied node’s offset-8 field is promoted into the supplied node’s incoming position. The promoted node’s offset-12 link is transferred to the supplied node’s offset-8 field. Parent links and either a context-associated root link or an existing parent’s child link are then updated.

**Inferred intent:** under the conventional interpretation that offset 4 is a parent link, offset 8 is a left-child link, and offset 12 is a right-child link, this is a right rotation of a binary-tree subtree.

The tree interpretation requires structural preconditions. The function does not validate them. It does not inspect ordering keys, rebalance metadata, or establish that the input structure is a tree.

## Interface

### Inputs

The function takes two address-bearing arguments:

- **Context:** identifies an object containing a 32-bit address-bearing field at byte offset 16.
- **Pivot:** identifies the node whose links are to be rearranged.

The supplied representation declares both arguments as 32-bit integers and uses them as addresses. This supports a 32-bit address-bearing interface; it does not establish compatibility with a native 64-bit pointer ABI.

### Accessed layouts

The following are partial layouts only. No meaning is established for unmentioned fields.

| Object | Byte offset | Width | Observed use |
|---|---:|---:|---|
| Context | 16 | 4 bytes | Address of an anchor object |
| Anchor | 4 | 4 bytes | Link compared with the pivot and conditionally replaced |
| Node | 4 | 4 bytes | Parent link under the tree interpretation |
| Node | 8 | 4 bytes | Left-child link under the tree interpretation |
| Node | 12 | 4 bytes | Right-child link under the tree interpretation |

The **anchor** is the object reached through the context’s offset-16 field. Calling its offset-4 field a root link is an inferred structural interpretation. Its operational role is established: equality with the pivot selects one of the update paths.

The **promoted node** is the address read from the pivot’s offset-8 field at entry. This address is retained for subsequent operations.

The **transfer link** is the promoted node’s offset-12 field. For an ordinary well-formed tree, it identifies the subtree moved between the pivot and promoted node.

### Output

The function has no explicit return value. Its result is the sequence of memory writes described below.

It supplies no success flag, error code, or validation result.

### Preconditions

For a defined, normally completing execution:

- Every address actually dereferenced must designate readable or writable storage as required by that access.
- Four-byte accesses must satisfy the target’s alignment and memory-access requirements.
- Address-plus-offset calculations must identify valid fields without relying on unspecified overflow behavior.
- The promoted-node address must permit the accesses at offsets 4 and 12. It is not checked for zero before use.
- The anchor address obtained from the context must permit access at offset 4.
- A nonzero transfer-link address, when used, must permit a write at offset 4.
- On the non-anchor replacement path, the pivot’s parent address must permit the selected accesses.

For the **right-rotation interpretation**, additional structural preconditions apply:

- The pivot and promoted node are distinct.
- The pivot’s offset-8 link identifies the promoted node.
- If the anchor’s offset-4 link is not the pivot, the pivot’s parent identifies the pivot through one of its child links.
- Relevant objects and fields do not overlap in ways that make an earlier write corrupt a later input.
- No concurrent writer changes the relevant links during the operation.

These are caller obligations, not checks performed by the function.

## Behavior

### Observable access order

The following describes the sequence supported by the supplied static representation. Repeated reads matter for overlapping storage, concurrent modification, or memory with read side effects. It is not a guarantee about instruction-level ordering for unsynchronized ordinary memory.

1. **Select the promoted node.**  
   Read the pivot’s four-byte field at offset 8 and retain that address.

2. **Replace the pivot’s offset-8 link.**  
   Read the promoted node’s four-byte field at offset 12. Write that value into the pivot’s field at offset 8.

3. **Conditionally repair the transferred node’s parent link.**  
   Read the promoted node’s offset-12 field again and compare it with zero.
   - If zero, perform no write through that link.
   - If nonzero, read that field again to obtain the destination address, then write the pivot address into the destination’s field at offset 4.

   The test and the destination selection are separate reads in the supplied representation. They must not be treated as a single cached lookup when describing behavior outside the stable-memory preconditions.

4. **Set the promoted node’s parent link.**  
   Read the pivot’s field at offset 4. Write that four-byte value into the promoted node’s field at offset 4.

5. **Locate and inspect the anchor link.**  
   Read the context’s field at offset 16 to obtain the anchor address. Read the anchor’s field at offset 4 and compare it with the pivot address.

6. **Replace the incoming link to the pivot.**
   - If the anchor’s offset-4 field equals the pivot, write the promoted-node address into that same field.
   - Otherwise, read the pivot’s offset-4 field again to obtain a parent address. Read that parent’s field at offset 12.
     - If the parent’s offset-12 field equals the pivot, replace it with the promoted-node address.
     - Otherwise, write the promoted-node address into the parent’s field at offset 8.

   The final alternative does **not** check that the parent’s offset-8 field previously referred to the pivot. It is the unconditional fallback after the other comparisons fail.

7. **Attach the pivot beneath the promoted node.**  
   Write the pivot address into the promoted node’s field at offset 12.

8. **Finish the pivot’s parent link.**  
   Write the promoted-node address into the pivot’s field at offset 4.

The function then returns.

### Result for a well-formed, stable tree

Let the pivot initially have a left child called the promoted node. Let the promoted node initially have a right child called the transferred subtree, possibly absent.

On normal completion:

- The pivot’s left child becomes the transferred subtree.
- If the transferred subtree exists, its parent becomes the pivot.
- The promoted node’s parent becomes the pivot’s former parent.
- The incoming link that previously identified the pivot is changed to identify the promoted node:
  - the anchor link if the pivot was the root;
  - otherwise the appropriate child link of the pivot’s parent.
- The promoted node’s right child becomes the pivot.
- The pivot’s parent becomes the promoted node.

The pivot’s right-child link and the promoted node’s left-child link are not directly written. No key, payload, color, height, count, or balance field is directly accessed.

Preservation of tree reachability and ordering is **conditional** on the input structure satisfying the tree interpretation and the necessary invariants. The function itself does not establish those conditions.

### Widths and arithmetic

All described field transfers are four bytes wide. Addresses are copied as four-byte values; no widening, truncation from a wider field, or numeric transformation of a link is shown.

The only address calculations shown add fixed byte offsets: 4, 8, 12, or 16. There is no element-count multiplication, allocation-size calculation, or loop.

Zero is tested only for the promoted node’s offset-12 link. The function does not generally treat zero as a safely accepted value for other address-bearing fields.

## Side effects

### Writes

For valid, stable, nonoverlapping inputs, the function performs:

- Five four-byte writes when the transfer link is zero.
- Six four-byte writes when the transfer link is nonzero.

Those writes affect the pivot, the promoted node, one incoming-link location, and optionally the transferred node.

These are write operations, not necessarily distinct destinations when callers violate the nonoverlap preconditions.

### Intermediate state

The rearrangement is not transactional. For example:

- The pivot’s offset-8 link changes before the incoming link is redirected.
- The promoted node’s parent changes before the pivot is attached beneath it.
- The pivot’s parent changes last.

An observer permitted to see intermediate writes may therefore see an inconsistent structure. No lock, atomic operation, memory barrier, or rollback mechanism appears in the supplied representation.

### Ownership and lifetime

No allocation, deallocation, reference-count update, or explicit ownership transfer is shown.

The operation changes structural links. It does not establish that any object is newly owned, released, or safe to destroy. Callers must keep all accessed objects alive throughout execution.

### Errors and invalid inputs

There is no explicit error path.

Invalid addresses, inaccessible storage, unsupported alignment, or invalid arithmetic may prevent normal completion. Exact fault or language-level behavior is not established by the supplied representation.

If an access fails after earlier writes, no restoration is shown. Partial modification is therefore a risk, not a reported recoverable outcome.

A structurally invalid but accessible parent relationship may complete without a fault while corrupting links: on the non-anchor path, failure to find the pivot in the parent’s offset-12 field causes the offset-8 field to be overwritten without verification.

## Dependencies

No function calls are present in the supplied body. No opaque callee IDs are therefore required for this function.

Execution depends on the target’s four-byte memory-access and address-arithmetic behavior. The supplied representation does not establish endianness, exception handling, or synchronization guarantees.

No explicit global variable is accessed. Reachable objects may nevertheless be shared or globally significant; their storage duration and ownership are unknown.

## Acceptance examples

These are conditional contract examples derived from static evidence. They are not reports of executed tests.

### Root rotation with a transferred subtree

Assume a valid tree in which:

- The anchor’s root link identifies the pivot.
- The pivot’s left child is the promoted node.
- The promoted node’s right child is a nonzero transferred subtree.
- The pivot has an existing parent-field value.

After normal completion:

- The anchor’s root link identifies the promoted node.
- The promoted node’s parent contains the pivot’s former parent-field value.
- The promoted node’s right child is the pivot.
- The pivot’s parent is the promoted node.
- The pivot’s left child is the transferred subtree.
- The transferred subtree’s parent is the pivot.

The root path does not dereference the pivot’s former parent value. It copies that value into the promoted node’s parent field.

### Non-root pivot occupying its parent’s right-child slot

Assume the anchor link is not the pivot, the pivot’s parent is valid, and that parent’s offset-12 field equals the pivot.

After normal completion, that parent’s offset-12 field identifies the promoted node. Its offset-8 field is not directly written. The remaining local links change as specified above.

### Non-root pivot occupying its parent’s left-child slot

Assume the anchor link is not the pivot, the parent’s offset-12 field does not equal the pivot, and its offset-8 field does equal the pivot.

After normal completion, the parent’s offset-8 field identifies the promoted node. Its offset-12 field is not written.

### Absent transferred subtree

Assume the promoted node’s offset-12 field remains zero until the final attachment.

The pivot’s offset-8 field becomes zero. No parent-field write is attempted through the absent transfer link. The promoted node’s offset-12 field subsequently becomes the pivot address.

### Inconsistent incoming relationship

Assume all storage is accessible, the anchor link is not the pivot, and neither child field of the pivot’s stated parent identifies the pivot.

The function still writes the promoted-node address into that parent’s offset-8 field. Acceptance of the raw behavior must not be confused with acceptance as a valid tree rotation: the structural preconditions were violated.

### Missing promoted node

A zero pivot offset-8 link is not an accepted empty-subtree case. The function attempts to access offset 12 through that value without a preceding zero check. No successful result or diagnostic is specified.

## Uncertainties

- **Inferred intent:** the parent/left/right/root names and right-rotation purpose follow the link pattern; no type declarations independently establish them.
- **documented_only — complete layouts:** object sizes, other fields, allocation mechanisms, and sentinel conventions are unknown.
- **documented_only — root parent convention:** the pivot’s old parent value is copied unchanged. Whether it is zero, an anchor address, or another sentinel is unknown.
- **documented_only — machine details:** endianness, exact ABI, alignment tolerance, and address-overflow behavior are not established.
- **documented_only — concurrency:** no supported concurrent access model is supplied.
- **documented_only — malformed aliasing:** overlapping fields may change later reads and branch selection. No useful tree-level guarantee is asserted for such inputs.
- **documented_only — wider invariants:** ordering, balancing, ownership, and external metadata requirements are not established by this function.

## Evidence

The evidence consists solely of the supplied static function representation.

It establishes four-byte field accesses, fixed offsets, conditional link replacement, repeated reads, the write sequence, and the absence of explicit calls or error reporting.

The binary-tree interpretation is an inference from that access pattern. No runtime tests have been performed, and none of the acceptance examples has been dynamically verified.
