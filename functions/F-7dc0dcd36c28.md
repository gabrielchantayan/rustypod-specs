<!-- Trusted provenance: F-7dc0dcd36c28; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7dc0dcd36c28

## Purpose

F-7dc0dcd36c28 rearranges a small set of linked records. Under the conventional interpretation of those records as binary-tree nodes, it performs a **left rotation around a supplied pivot**: the pivot’s right child takes the pivot’s former position, and the pivot becomes that child’s left child.

The rotation interpretation is inferred intent. The established behavior is a sequence of 32-bit reads and writes at fixed offsets, including replacement of either a designated root link or a link in the pivot’s parent.

The function does not search for a suitable pivot, validate the structure, compare node contents, or compute balancing information. It assumes that the supplied addresses and the links it follows are usable.

## Interface

### Inputs

The function accepts two address-bearing arguments:

| Argument | Role |
|---|---|
| Context address | Identifies an object containing, at byte offset `0x10`, an address of a root-holder record. |
| Pivot address | Identifies the node whose right-side link supplies the node to promote. |

The displayed interface represents both arguments as integer values. The accesses use four-byte fields, and values read from those fields are also used as addresses. This supports a 32-bit address-bearing representation for this contract; a particular machine ABI, pointer-conversion rule, or address-overflow rule is not independently established.

The arguments are passed by value. Their values are not returned or reassigned for the caller, but memory reachable through them is modified.

### Relevant layouts

Only the following fields are established:

| Record | Byte offset | Width | Role in this operation |
|---|---:|---:|---|
| Context | `0x10` | 32 bits | Address of root holder |
| Root holder | `0x04` | 32 bits | Designated root link |
| Node | `0x04` | 32 bits | Parent link |
| Node | `0x08` | 32 bits | Left-side child link |
| Node | `0x0c` | 32 bits | Right-side child link |

“Parent,” “left,” “right,” and “root” describe the inferred structural roles. The offsets and transfers are established directly.

The root holder need not be the context itself. It is reached through the context’s field at offset `0x10`.

No complete record size, allocation boundary, payload layout, or meaning of other bytes is established.

### Output

There is no explicit return value. The result is represented entirely by writes to linked records.

There is no success flag, error code, output parameter distinct from the structure, or documented failure return.

### Preconditions for the intended rotation

For the ordinary tree interpretation to apply:

- The pivot is a valid, writable node.
- The pivot’s right link identifies a valid, writable node. This node is called the **promoted node** below.
- The context identifies a readable root-holder address, and the root-holder root field is readable.
- If the pivot is the designated root, that root field is writable.
- If the pivot is not the designated root, its parent link identifies a node with readable left link and a writable child-link field selected by this operation.
- A nonzero left link of the promoted node identifies a node whose parent field is writable.
- Records and fields do not overlap in a way that changes the intended topology during the operation.
- Other agents do not modify relevant fields while the operation is in progress.

These are caller obligations, not checked conditions. The source does not establish alignment requirements beyond those imposed by the underlying execution environment’s 32-bit accesses.

## Behavior

### Structural result for a well-formed input

Let:

- **Pivot** be the supplied node.
- **Promoted node** be the pivot’s original right child.
- **Transferred subtree** be the promoted node’s original left child, possibly zero.
- **Former parent** be the pivot’s original parent.
- **Root holder** be the record reached through the context.

For a stable, well-formed structure, the intended final relationships are:

1. The pivot’s right link becomes the transferred-subtree link.
2. If that link is nonzero, the transferred subtree’s parent becomes the pivot.
3. The promoted node’s parent becomes the former parent.
4. The link that previously selected the pivot from above becomes a link to the promoted node:
   - If the root holder selects the pivot, its root link is replaced.
   - Otherwise, the former parent’s left link is replaced if it equals the pivot.
   - Otherwise, the former parent’s right link is replaced without an additional equality check.
5. The promoted node’s left link becomes the pivot.
6. The pivot’s parent becomes the promoted node.

The pivot’s left link and the promoted node’s right link are not explicit write targets. For distinct, nonoverlapping records, those links remain unchanged.

### Ordered memory behavior

The operation is not a single atomic structural update. Its reads and writes occur in the following source-established order.

1. **Select the promoted node.**  
   Read the 32-bit value at pivot offset `0x0c`. This address is retained as the promoted-node address for the remainder of the operation.

2. **Replace the pivot’s right link.**  
   Read the promoted node’s field at offset `0x08`, then write that 32-bit value to the pivot’s field at offset `0x0c`.

3. **Test whether a parent repair is needed.**  
   Read the promoted node’s field at offset `0x08` again and compare it with zero.

   If the value is nonzero, read that same field again to obtain the destination-node address, then write the pivot address to that node’s field at offset `0x04`.

   These are separate reads in the supplied source. The operation does not retain and reuse only the value copied in the preceding step. Under stable-memory assumptions they agree; those assumptions are not enforced.

4. **Transfer the old parent link to the promoted node.**  
   Read the pivot’s field at offset `0x04` and write its 32-bit value to the promoted node’s field at offset `0x04`.

5. **Locate and inspect the designated root link.**  
   Read the context’s field at offset `0x10` to obtain the root-holder address. Select the four-byte field at root-holder offset `0x04`, and read it for comparison with the pivot address.

   This context/root-holder access occurs after the preceding structural writes, not before them.

6. **Replace the link from above.**

   - **Root case:** If the inspected root link equals the pivot address, write the promoted-node address to that root-link field. The function does not consult the pivot’s parent for choosing this branch.
   - **Non-root case:** Otherwise, read the pivot’s field at offset `0x04` again to obtain a parent address. Read that parent’s field at offset `0x08`.
     - If the value equals the pivot address, write the promoted-node address to the parent’s field at offset `0x08`.
     - Otherwise, write the promoted-node address to the parent’s field at offset `0x0c`.

   The non-root path does not verify that the right link actually points to the pivot before overwriting it. The left-link comparison alone selects between the two write destinations.

7. **Attach the pivot below the promoted node.**  
   Write the pivot address to the promoted node’s field at offset `0x08`.

8. **Finish the pivot’s parent update.**  
   Write the promoted-node address to the pivot’s field at offset `0x04`.

9. **Return without a value.**

This ordering describes the supplied source’s accesses. It does not establish a hardware memory-ordering protocol, atomicity guarantee, or concurrency-safe publication mechanism.

### Branch precedence

Root membership is determined by equality with the root-holder field, not by testing whether the pivot’s parent is zero or a sentinel.

If the root-holder field equals the pivot, the root branch wins regardless of what the pivot’s parent field contains.

On the non-root path, a matching left link wins. If both parent child links happen to equal the pivot, only the left link is selected for replacement.

### Arithmetic and field width

All identified field transfers are four bytes wide. Values are copied without an explicit mask, scaling operation, or arithmetic adjustment.

The only address arithmetic shown is addition of fixed byte offsets: `0x04`, `0x08`, `0x0c`, and `0x10`. There are no counts, loop bounds, subtree-size calculations, or balancing calculations.

The source does not establish behavior for address addition that overflows the applicable address representation. Such addresses are outside the supported preconditions of this contract rather than assigned guessed wraparound semantics.

## Side effects

### Writes and visibility

For stable, nonoverlapping records, the operation performs:

- One write to the pivot’s right link.
- One optional write to the transferred subtree’s parent link.
- One write to the promoted node’s parent link.
- One write to either the root link or a parent child link.
- One write to the promoted node’s left link.
- One write to the pivot’s parent link.

There are five stores when the transferred subtree is zero and six when it is nonzero. These counts describe executed stores, not necessarily distinct addresses under malformed aliasing.

Intermediate states are externally relevant if the structure is observed during execution. In particular, the pivot’s right link changes before the incoming root or parent link changes, and the incoming link changes before the pivot’s final parent update.

### Ownership and lifetime

No allocation, deallocation, reference-count operation, or external call appears in the function. It rewrites existing links; it does not explicitly create or destroy records.

A normal caller must keep every accessed record alive throughout the operation. Broader ownership semantics—such as exclusive ownership of children, membership in a particular container, or responsibility for later destruction—are not established by this function.

### Failure behavior

There are no explicit checks for:

- A zero context or pivot.
- A zero promoted-node address.
- A zero root-holder address.
- A missing parent on the non-root path.
- Invalid or read-only destinations.
- Cycles, self-links, or overlapping records.
- Inconsistent parent/child relationships.

A zero transferred-subtree link is handled by skipping its parent update. This is the only null-link case explicitly accommodated.

Invalid accesses may fail according to the execution environment. No particular exception, signal, diagnostic, or recovery mechanism is established. Because writes precede some later reads, a failure can leave a partially modified structure. No rollback is present.

## Dependencies

No called-function dependency is visible in F-7dc0dcd36c28. There are therefore no additional opaque function IDs to enumerate.

The operation depends directly on caller-supplied memory and on the execution environment’s support for the shown four-byte reads and writes. The behavior of faults, concurrent accesses, and invalid address representations remains documented_only.

## Acceptance examples

These examples are conditional predictions from static evidence, not executed tests. Unless stated otherwise, records are valid, stable, and nonoverlapping.

### Root rotation with a nonempty transferred subtree

Initially:

- The root holder selects the pivot.
- The pivot’s parent is a sentinel value.
- The pivot’s right link selects the promoted node.
- The promoted node’s left link selects a transferred subtree.

Expected result:

- The root holder selects the promoted node.
- The promoted node’s parent contains the pivot’s former sentinel value.
- The promoted node’s left link selects the pivot.
- The pivot’s parent selects the promoted node.
- The pivot’s right link selects the transferred subtree.
- The transferred subtree’s parent selects the pivot.

The pivot’s left link and promoted node’s right link remain unchanged.

### Non-root pivot selected by its parent’s left link

Initially, the root holder does not select the pivot, and the pivot’s parent has a left link equal to the pivot.

Expected result:

- That left link is replaced with the promoted-node address.
- The parent’s right link is unchanged.
- The promoted node receives the pivot’s former parent link.
- The remaining rotation links change as described above.

The root-holder field is read but not written.

### Non-root pivot selected through the right-side branch

Initially, the root holder does not select the pivot, and the pivot’s parent’s left link differs from the pivot.

Expected result:

- The parent’s right link is overwritten with the promoted-node address.

This expectation does not require the old right link to equal the pivot. Consequently, inconsistent input can overwrite an unrelated right-side link rather than producing a validation error.

### Empty transferred subtree

Initially, the promoted node’s left link is zero.

Expected result:

- The pivot’s right link becomes zero.
- No transferred-subtree parent write occurs.
- The promoted node’s left link later becomes the pivot.
- The other applicable root or parent updates still occur.

### Parent field does not determine root status

Initially, the root holder selects the pivot, but the pivot’s parent field contains a nonzero value.

Expected result:

- The root link is replaced with the promoted-node address.
- The nonzero parent value is copied into the promoted node’s parent field.
- The function does not inspect that parent’s child links on this branch.

Whether the nonzero value is a legitimate sentinel or an inconsistency is outside this function’s validation.

### Missing promoted node

Initially, the pivot’s right link is zero.

The function does not treat this as a no-op or return an error. Its next required access uses the promoted-node address plus offset `0x08`. Successful rotation is not promised; the resulting memory-access behavior is environment-dependent.

## Uncertainties

- **Inferred intent:** The operation is a binary-tree left rotation. Field roles strongly support this interpretation, but no enclosing data-type declaration or caller contract is supplied.
- **documented_only — ABI:** Four-byte accesses are established; the exact machine ABI and integer-to-address rules are not independently established.
- **documented_only — sentinels:** The meaning of a root’s parent field and any header or sentinel convention is unknown.
- **documented_only — malformed aliasing:** Overlapping records can make earlier stores change later reads. The ordered access description governs what is visible in the source; the ordinary rotation postconditions require suitable separation.
- **documented_only — concurrency:** No synchronization or atomicity contract is supplied.
- **documented_only — faults:** Invalid-access outcomes and partial-write visibility depend on the execution environment.
- **documented_only — wider invariants:** Search ordering, balance, colors, heights, membership rules, and payload semantics are not established.

## Evidence

Established static evidence consists of the supplied body of F-7dc0dcd36c28: fixed-offset four-byte accesses, a zero test for the promoted node’s left link, root-link equality selection, parent-left-link equality selection, ordered link stores, and return without an explicit value.

The function body contains no calls, loops, allocation operations, explicit validation path, or rollback path.

The rotation interpretation and well-formed-tree postconditions are derived from those accesses and are conditional on the stated preconditions. No runtime tests have been performed.
