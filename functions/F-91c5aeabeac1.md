<!-- Trusted provenance: F-91c5aeabeac1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-91c5aeabeac1

## Purpose

F-91c5aeabeac1 rearranges links in caller-supplied records. Its apparent purpose is a left rotation in a parent-linked binary structure: the selected node’s right child takes its place, and the selected node becomes that child’s left child.

The established behavior is narrower than that interpretation: a fixed sequence of four-byte reads, conditional decisions, and five or six four-byte writes. The function does not validate tree relationships, enforce ordering, update balancing metadata, allocate records, or report success.

The rotation interpretation applies when the records have coherent links and suitable storage separation. For overlapping or inconsistent records, the ordered accesses—not an assumed tree invariant—define the behavior supported by the supplied body.

## Interface

### Inputs

The function receives two address-bearing values, represented as 32-bit integers in the supplied representation:

- **Context address:** identifies storage whose field at byte offset 16 supplies the address of a reference-holding record.
- **Selected-node address:** identifies the node whose links will be rearranged.

The selected node is called the **pivot** below. Its initially read right-child reference identifies the **promoted node**.

These inputs are used to form addresses. The function does not receive an explicit record size, validity flag, allocator, synchronization object, or error-output location.

### Relevant layout

All listed field accesses are four bytes wide. Offsets are measured in bytes.

| Record | Offset | Observed use |
|---|---:|---|
| Context | 16 | Read as the address of a reference-holding record |
| Reference-holding record | 4 | Compared with the pivot address and conditionally replaced |
| Participating node | 4 | Copied or replaced as a parent link |
| Participating node | 8 | Read or replaced as a left-child link |
| Participating node | 12 | Read or replaced as a right-child link |

“Parent,” “left child,” and “right child” are structural interpretations of the accesses. The function does not enforce a node type or verify that these roles are consistent.

The reference-holding record is called the **root-reference record** below. Interpreting its offset-4 field as a root reference is supported by the comparison and replacement pattern, but the record’s complete layout and relationship to any sentinel remain unknown.

No meaning is established for fields outside these offsets.

### Output

There is no explicit return value, status code, or success indicator. Observable results are mutations of the supplied memory.

### Preconditions for ordinary rotation behavior

The caller must provide memory permitting all accesses taken by the selected branch:

- The context’s offset-16 field and the pivot’s accessed fields must be readable.
- The pivot’s initial offset-12 value must identify a promoted node with accessible offset-4 and offset-8 fields. There is no zero check for this address.
- If the promoted node’s offset-8 value is nonzero when tested, the address obtained by the subsequent read of that field must permit a four-byte write at offset 4.
- The context’s offset-16 value must identify a record with a readable offset-4 field. That field must also be writable when it equals the pivot address.
- On the non-root branch, the pivot’s then-current parent reference must identify a record with a readable offset-8 field and a writable selected child field.
- Every direct write target must be writable.
- Links and storage separation must support the intended structural interpretation.

Stable links are additionally needed for the ordinary postconditions described below. These preconditions are not checked. Synchronization, mutation permission, and record lifetime remain caller responsibilities.

## Behavior

### Structural result under coherent inputs

Assume stable memory, distinct relevant fields, and consistent parent/root relationships. Call the promoted node’s initial left child the **transferred subtree**.

Normal completion produces these changes:

- The pivot’s right child becomes the transferred subtree.
- If the transferred subtree is nonzero, its parent becomes the pivot.
- The promoted node’s parent becomes the pivot’s former parent.
- Either the distinguished root reference or one child reference in the former parent is replaced with the promoted-node address.
- The promoted node’s left child becomes the pivot.
- The pivot’s parent becomes the promoted node.

The pivot’s left-child field and the promoted node’s right-child field are not direct write targets.

No ordering comparison, balancing decision, payload movement, or recursive traversal occurs.

### Ordered memory behavior

The following sequence describes the accesses expressed by the supplied body, assuming each access completes normally. Semantic names do not imply that every related field value is captured at entry.

**Promoted-node selection.** The function reads the pivot’s offset-12 field. The resulting promoted-node address is retained for the remainder of the operation; the pivot’s right-child field is not reread to select another promoted node.

**Replacement of the pivot’s right link.** The function reads the promoted node’s offset-8 field and writes that four-byte value into the pivot’s offset-12 field.

**Conditional transferred-node parent repair.** The function reads the promoted node’s offset-8 field again and tests that read against zero.

- A zero result skips the parent-repair write.
- A nonzero result causes another read of the same offset-8 field. This third read supplies the destination address for a four-byte write of the pivot address at offset 4.

Consequently, the promoted node’s left-child field is read twice when repair is skipped and three times when repair is taken. The value copied into the pivot, the value tested, and the address used for repair come from separate reads.

The nonzero test does not validate the value obtained by the third read. Under stable memory these values agree. If aliasing or external mutation changes the field between accesses, the function provides no reconciliation.

**Parent-link copying.** The function reads the pivot’s offset-4 field and writes its four-byte value into the promoted node’s offset-4 field.

**Distinguished-reference inspection.** The function reads the context’s offset-16 field and uses that address to locate the root-reference record’s offset-4 field. It reads that reference and compares it with the pivot address.

Exactly one of the following replacement paths is taken:

- **Reference equals the pivot:** the promoted-node address is written to the same root-reference field. Its destination has already been obtained; the context field is not reread for this write.
- **Reference differs from the pivot:** the pivot’s offset-4 field is read again to obtain a parent address. The parent’s offset-8 field is read and compared with the pivot address.
  - If equal, the promoted-node address is written to that parent’s offset-8 field.
  - Otherwise, the promoted-node address is written to that parent’s offset-12 field.

The non-root branch retains the parent address obtained for that branch. It does not reread the pivot’s parent field to choose the store destination.

Failure of the parent’s left-child comparison is sufficient to select its right-child field. The function does not read that right-child field to confirm that it previously contained the pivot.

The parent reference copied to the promoted node and the parent reference used on the non-root branch are separate reads. Their equality is not enforced.

**Final relationship writes.** The function writes the pivot address into the promoted node’s offset-8 field, then writes the retained promoted-node address into the pivot’s offset-4 field.

There is no subsequent validation, repair, or notification.

### Widths and arithmetic

Every field transfer described above is four bytes. Reference comparisons are equality or zero tests; no signed ordering comparison is performed.

Address formation uses fixed byte offsets 4, 8, 12, and 16. No explicit bounds check, overflow check, address normalization, or recovery is present. The supplied representation does not establish portable semantics for an address-bearing integer whose addition overflows, nor for an inaccessible, misaligned, or otherwise invalid resulting address.

Zero receives special treatment only when testing the promoted node’s left-child field. Other address-bearing fields are dereferenced without a corresponding zero check.

## Side effects

Normal completion executes **five four-byte stores when transferred-node parent repair is skipped, and six when it is performed**.

The five stores executed on every normally completing path target:

| Target | Stored value |
|---|---|
| Pivot offset 12 | Value from the first read of promoted-node offset 8 |
| Promoted node offset 4 | Value read from pivot offset 4 |
| One root or parent reference | Promoted-node address |
| Promoted node offset 8 | Pivot address |
| Pivot offset 4 | Promoted-node address |

The conditional sixth store targets offset 4 of the address obtained from the third read of promoted-node offset 8. It stores the pivot address.

The root/parent replacement contributes exactly one store: root-reference-record offset 4, parent offset 8, or parent offset 12. It is not an additional store on top of that selection.

These counts describe executed store operations, not necessarily distinct destinations. Aliasing can cause multiple stores to affect the same bytes, and an earlier store can alter a field read later.

No allocation, release, or record-copy operation appears. Rewiring existing references does not establish ownership transfer or lifetime management. No direct write targets a payload, counter, or balancing field as such, although overlapping storage may make an addressed link field serve another role.

The operation has no rollback path. Writes occur before the distinguished-reference inspection and parent-link selection. A later invalid access may therefore occur after earlier mutations. Exact failure and visibility semantics depend on the execution environment; no transactional guarantee is established.

## Dependencies

No calls to other functions appear in the supplied body. There are no dependency function IDs to enumerate.

The operation depends directly on caller-supplied memory and the execution environment’s address-access semantics. It does not delegate validation, allocation, release, synchronization, or error reporting to an observed callee.

No assumptions about unknown callees preserving globals or owning memory are needed.

## Acceptance examples

These are conditional consequences of static inspection, not runtime test results. Unless stated otherwise, they assume stable memory, valid accesses, coherent links, and non-overlapping participating fields.

### Root pivot with a nonzero transferred subtree

Initially, the distinguished reference identifies the pivot, the pivot’s right child identifies the promoted node, and the promoted node’s left child identifies a nonzero transferred subtree.

After normal completion:

- The distinguished reference identifies the promoted node.
- The promoted node’s parent contains the pivot’s original parent-link value.
- The pivot’s right child identifies the transferred subtree.
- The transferred subtree’s parent identifies the pivot.
- The promoted node’s left child identifies the pivot.
- The pivot’s parent identifies the promoted node.

Six four-byte stores execute, including the transferred-subtree parent repair. The pivot’s original parent-link value is copied without interpreting whether it represents zero, a sentinel, or another record.

### Non-root pivot selected through its parent’s left link

Initially, the distinguished reference differs from the pivot, and the pivot’s recorded parent has the pivot address at offset 8.

The parent’s offset-8 field is replaced with the promoted-node address. Its offset-12 field is not a direct write target. The distinguished reference is read but not written.

Five stores execute if transferred-node repair is skipped; six execute if it is taken.

### Non-root pivot not selected through its parent’s left link

Initially, the distinguished reference differs from the pivot, and the recorded parent’s offset-8 field does not equal the pivot address.

The parent’s offset-12 field is replaced with the promoted-node address regardless of its previous contents. A valid prior right-child relationship is not required to select this branch.

This demonstrates the replacement rule, not a guarantee that inconsistent input becomes a valid tree.

### Zero transferred-subtree reference

If the promoted node’s offset-8 field remains zero through the relevant reads:

- Zero is written into the pivot’s offset-12 field.
- No transferred-node parent field is written.
- The root/parent replacement and both final relationship writes still occur.
- Exactly five four-byte stores execute.

This is not an early return or a no-op.

### Invalid promoted-node reference

If the pivot’s initial offset-12 value is zero, the next field read is still attempted at that value plus 8. There is no branch that treats this as an absent right child and returns successfully.

Whether the resulting address is accessible, faults, or has other consequences is environment-dependent. No explicit error result is provided.

### Overlapping records

If an earlier write changes storage later read under another role, entry-state rotation postconditions do not apply automatically.

For example, agreement among the separate reads of promoted-node offset 8 is not guaranteed by the function itself. The retained promoted-node address remains the one selected initially, while subsequent field values follow the ordered accesses. No overlap detection or corrective path exists.

## Uncertainties

### Established static evidence

The supplied body establishes:

- Four-byte field transfers at the listed offsets.
- Retention of the initially selected promoted-node address.
- Repeated reads of the promoted node’s left-child field.
- Separate parent-link reads for copying and non-root replacement.
- The branch conditions and ordered writes.
- Five stores without transferred-node repair and six with it.
- No explicit calls, validation result, allocation, release, or rollback.

### Inferred intent

The field layout and rewiring strongly support a left rotation in a parent-linked binary structure. The distinguished reference likely represents a tree root.

Neither interpretation establishes a particular tree family, balancing policy, payload ordering, container implementation, or sentinel convention.

### documented_only

The following remain unresolved:

- Exact runtime architecture, byte order, alignment requirements, and address representation.
- Full record sizes and meanings of unaccessed fields.
- Invalid-address and integer-overflow semantics.
- Sentinel conventions, including whether a nonzero sentinel may occupy a child field.
- Caller-enforced structural invariants and access permissions.
- Ownership, allocation provenance, and lifetime requirements beyond accessibility during the accesses.
- Concurrency rules, atomicity, and memory-order guarantees.
- Whether surrounding operations perform additional balancing or metadata maintenance.

These unknowns do not justify assuming validation, safe failure, ownership transfer, or synchronization.

## Evidence

This contract is based solely on static inspection of the supplied body for F-91c5aeabeac1. The layouts, access sequence, branches, and store counts follow the visible operations. Structural purpose is identified separately as inference.

The store count consists of five always-executed writes on normal completion, plus one conditional transferred-node parent write. The three root/parent alternatives collectively contribute one of those five writes.

No runtime tests have been performed. The acceptance examples are conditional specifications, not execution observations.
