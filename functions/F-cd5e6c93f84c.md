<!-- Trusted provenance: F-cd5e6c93f84c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd5e6c93f84c

## Purpose

F-cd5e6c93f84c performs a local pointer-topology transformation consistent with a **right rotation of a binary-tree subtree**. It promotes the node currently reached through the selected node’s offset-8 link, moves the promoted node’s offset-12 subtree into that vacated link, reconnects the promoted node to the selected node’s former incoming connection, and makes the selected node the promoted node’s offset-12 child.

The established behavior is a sequence of 32-bit memory reads, pointer comparisons, and 32-bit memory writes. Calling these fields “parent,” “left child,” “right child,” and “root” is an interpretation supported by their use, not evidence of a complete tree implementation or an enforced data-structure invariant.

The function does not validate its inputs, allocate storage, release storage, invoke another function, or report success or failure. Its behavior is suitable for a tree rotation only when the caller supplies a compatible, writable structure.

## Interface

### Inputs

The function accepts two arguments represented as 32-bit integer values in the supplied evidence. Both are used as addresses:

- **Context address:** Identifies an object containing a 32-bit control-record address at byte offset 16.
- **Selected-node address:** Identifies the node whose local topology is to be transformed.

The interface does not establish signed arithmetic as part of the intended behavior. The arguments and link fields are used as address-sized bit patterns, not as counts or signed ordering values.

### Referenced layouts

Only the following fields are established. These are partial layouts, not complete object definitions.

| Object | Byte offset | Width | Operational role |
|---|---:|---:|---|
| Context | 16 | 32 bits | Address of a control record |
| Control record | 4 | 32 bits | Root-like node link |
| Node | 4 | 32 bits | Parent-like link |
| Node | 8 | 32 bits | Left-child-like link |
| Node | 12 | 32 bits | Right-child-like link |

For the rotation interpretation:

- **Selected node** is the node supplied by the second argument.
- **Promoted node** is the address initially read from the selected node’s offset-8 field.
- **Transferred subtree** is the address read from the promoted node’s offset-12 field.
- **Former parent** is the selected node’s offset-4 value.
- **Root link** is the offset-4 field of the control record reached through the context’s offset-16 field.

A zero transferred-subtree address is explicitly supported. A zero promoted-node address is not checked before use.

### Output

The function has no return value. Its observable result is the memory mutation described below.

There is no status flag, error code, replacement-root return value, or separate ownership result.

### Preconditions

For normal, defined operation under the tree interpretation:

- The context’s offset-16 field must be readable.
- The control-record address obtained from that field must identify a readable and writable offset-4 field.
- The selected node’s offset-4 and offset-8 fields must be readable and writable as required below.
- The selected node’s initial offset-8 link must identify a valid promoted node with accessible offset-4 and offset-12 fields.
- A nonzero transferred-subtree link must identify storage with a writable offset-4 field.
- If the root link does not equal the selected-node address, the selected node’s offset-4 link must identify a valid former parent. That parent’s offset-12 field must be readable, and either its offset-12 or offset-8 field must be writable.
- Address calculations and four-byte accesses must be valid for the actual target environment.

The function itself does not enforce these conditions.

A conventional rotation additionally requires compatible tree topology and nonconflicting storage. Those stronger structural requirements explain the operation’s intended result but are not independently checked by the function.

## Behavior

### Established access and update sequence

The following ordering describes the supplied source-level behavior. It does not claim a particular processor instruction sequence or a universal evaluation order between address calculations within one expression.

1. **Obtain the promoted-node address.**  
   The function reads the selected node’s offset-8 field. This address remains the promoted node used throughout the operation.

2. **Replace the selected node’s offset-8 link.**  
   The function reads the promoted node’s offset-12 field and writes that 32-bit value into the selected node’s offset-8 field.

3. **Repair the transferred subtree’s parent-like link when nonzero.**  
   The promoted node’s offset-12 field is read again and compared with zero. If nonzero, that field is read to obtain the address whose offset-4 field receives the selected-node address.

   These accesses are not established as a single cached snapshot. In ordinary, nonoverlapping storage without outside mutation, they identify the same transferred subtree. Aliasing or external mutation can invalidate that simplification.

4. **Copy the selected node’s parent-like link to the promoted node.**  
   The function reads the selected node’s offset-4 field and writes its 32-bit value into the promoted node’s offset-4 field.

5. **Locate and inspect the root link.**  
   Only after the preceding writes, the function reads the context’s offset-16 field and locates the offset-4 field of the resulting control record. It reads that root link and compares it with the selected-node address.

6. **Reconnect the promoted node through one of three paths.**
   - If the root link equals the selected-node address, the root link is overwritten with the promoted-node address.
   - Otherwise, the function reads the selected node’s offset-4 field again to obtain the former-parent address. It reads that parent’s offset-12 field.
     - If the parent’s offset-12 field equals the selected-node address, that field is overwritten with the promoted-node address.
     - Otherwise, the parent’s offset-8 field is overwritten with the promoted-node address.

   The final alternative is unconditional once the parent’s offset-12 comparison fails. The function does **not** verify that the parent’s offset-8 field previously referred to the selected node.

7. **Make the selected node the promoted node’s offset-12 child.**  
   The function writes the selected-node address into the promoted node’s offset-12 field.

8. **Make the promoted node the selected node’s parent.**  
   The function writes the promoted-node address into the selected node’s offset-4 field, then returns.

The two final writes occur after reconnection through the root or former parent.

### Result for a well-formed tree

With distinct, stable node storage and valid tree links, the completed operation has these effects:

- The selected node’s former incoming link now refers to the promoted node.
- The promoted node’s parent-like link becomes the selected node’s former parent-like link.
- The promoted node’s former right subtree becomes the selected node’s left subtree.
- A nonempty transferred subtree receives the selected node as its parent.
- The selected node becomes the promoted node’s right child.
- The selected node’s parent becomes the promoted node.

The root case copies the selected node’s former parent-like value to the promoted node just as the nonroot case does. The function does not independently clear that field or substitute the control-record address.

Under these preconditions, the selected node’s offset-12 field and the promoted node’s offset-8 field are not changed. No payload, key, balance, color, height, or size field is explicitly updated.

### Widths and arithmetic

Every explicit memory access in the supplied body is four bytes wide. Link replacement copies the entire 32-bit field; no masking, tagging, truncation from a wider source, or partial-field update is shown.

The only address displacements are 4, 8, 12, and 16 bytes. There is no index multiplication, loop, counter, or subtree-size arithmetic.

The evidence does not establish the source language’s original pointer types, endianness, alignment requirements, or behavior for address overflow. Valid address formation is a caller/environment precondition rather than a checked condition.

## Side effects

### Memory writes

On a successful execution using ordinary disjoint tree storage:

- Five four-byte writes occur if the transferred subtree is zero.
- Six four-byte writes occur if the transferred subtree is nonzero.

The writes target:

1. The selected node’s offset-8 field.
2. The transferred subtree’s offset-4 field, conditionally.
3. The promoted node’s offset-4 field.
4. Exactly one incoming-link field: the root link, the former parent’s offset-12 field, or the former parent’s offset-8 field.
5. The promoted node’s offset-12 field.
6. The selected node’s offset-4 field.

These counts describe write operations, not necessarily distinct locations under malformed aliasing.

### Ownership and lifetime

No allocation, deallocation, reference-count adjustment, or destructor call is present. The function rewrites links in existing storage.

It does not establish that it owns the referenced objects or that ownership transfers with any link. The caller must ensure that all accessed objects remain usable for the duration of the operation.

### Intermediate states and failures

The mutation is not transactional. The selected node’s child link and potentially the transferred subtree’s parent link are changed before the root or former-parent connection is updated.

No rollback or explicit error handling is shown. If an access cannot complete, earlier writes may already have occurred; the specific fault behavior is not established.

No synchronization or atomic multi-field update is present. Concurrent access requires external guarantees. The supplied evidence does not define thread-safe observation of intermediate states.

## Dependencies

No function calls appear in the supplied body. There are no external function dependencies requiring additional opaque function IDs.

The operation depends on caller-provided memory, the partial layouts above, and the target environment’s address and memory-access behavior. No claim is made about another routine preserving globals or owning or releasing memory.

There is no explicit access to a fixed global object. The context may reference shared state, but whether it does so is unknown.

## Acceptance examples

These examples are conditional specifications, not reports of executed tests. Unless stated otherwise, assume valid four-byte fields, nonoverlapping objects, stable memory, and no concurrent mutation.

### Root rotation with a nonempty transferred subtree

Initially:

- The root link identifies node S.
- S’s offset-8 link identifies node P.
- P’s offset-12 link identifies node T.
- S’s offset-4 field contains value A.

After completion:

- The root link identifies P.
- P’s offset-4 field contains A.
- S’s offset-8 link identifies T.
- T’s offset-4 field identifies S.
- P’s offset-12 link identifies S.
- S’s offset-4 field identifies P.

S’s offset-12 field and P’s offset-8 field retain their initial values.

### Root rotation with no transferred subtree

Use the preceding setup, except P’s offset-12 field is zero.

After completion, S’s offset-8 field is zero, and no transferred-subtree parent field is written. The remaining root and parent-link results are the same. In particular, the function does not access address zero merely because this transferred-subtree link is zero.

### Selected node is the former parent’s right child

Assume the root link does not identify S, S’s offset-4 field identifies A, and A’s offset-12 field identifies S.

After completion, A’s offset-12 field identifies P. A’s offset-8 field and the root link are unchanged. The local child and parent updates follow the general rotation result.

### Selected node takes the fallback incoming-link path

Assume the root link does not identify S, S’s offset-4 field identifies A, and A’s offset-12 field does not identify S.

The function writes P into A’s offset-8 field. This occurs whether or not A’s offset-8 field initially identified S.

Thus, on inconsistent input, the function can overwrite an unrelated left-child-like link. Rejection, diagnosis, or preservation of that unrelated link is not part of the established behavior.

### Root detection takes precedence

Assume the root link identifies S while S’s offset-4 field contains a nonzero value A.

The function copies A to P’s offset-4 field and replaces the root link with P. It does not inspect A’s child fields to choose an incoming link, and it does not normalize P’s parent-like field to zero.

### Missing promoted node

If S’s offset-8 field is zero, the function attempts to read a field at displacement 12 from that value. There is no checked empty-child result or early return. A normal rotation is outside the established valid-input domain in this case.

## Uncertainties

### Inferred intent

The offsets and rewiring strongly support a right-rotation interpretation with parent, left-child, right-child, and root links. The broader container type, its invariants, and the reason for performing the rotation are not established.

Preservation of search-tree ordering depends on the input tree’s ordering invariant. This function neither reads nor compares keys.

### documented_only

The following remain unknown:

- The complete node, context, and control-record layouts.
- Whether a root’s parent-like field normally contains zero, a sentinel, or another value.
- Whether stored links have any broader ownership meaning.
- Required alignment and byte order.
- Original pointer types and address-overflow semantics.
- Permitted aliasing among the context, control record, and nodes.
- Platform behavior for inaccessible or invalid addresses.
- Whether external synchronization protects these mutations.
- Any caller-side validation or subsequent metadata repair.

Malformed aliases may cause an earlier write to change a field read later, including a repeated transferred-subtree read or the later context lookup. The ordinary rotation postcondition must not be extended to such cases without additional evidence.

## Evidence

**Established static evidence:** The supplied body contains four-byte link accesses, two conditional reconnection decisions, a conditional transferred-subtree parent update, no function calls, and no explicit validation or recovery path. Its ordered writes support the operational contract above.

**Inferred intent:** The local topology change is consistent with a right rotation. Tree terminology is explanatory; the reads, comparisons, and writes define the established behavior.

**Verification limit:** No runtime tests have been performed. The acceptance examples are consequences of the supplied static evidence under their stated preconditions, not observed execution results.
