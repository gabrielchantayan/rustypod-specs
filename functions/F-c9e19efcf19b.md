<!-- Trusted provenance: F-c9e19efcf19b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c9e19efcf19b

## Purpose

F-c9e19efcf19b rearranges links around a caller-selected record and the record initially referenced by its field at byte offset +8. It redirects either a distinguished link reached through the first argument or a link in the selected record’s previous parent.

**Established static behavior:** the function reads and writes 32-bit fields, compares address-bearing values, and returns no value. It makes no visible function calls.

**Inferred intent:** the accessed layout and resulting relationships are consistent with a right rotation in a parent-linked binary tree. Under that interpretation:

- +4 is a parent link.
- +8 is a left-child link.
- +12 is a right-child link.
- The distinguished link is a root link.

These names describe an interpretation, not independently established object types or invariants. The operation does not inspect keys, establish ordering, validate a tree, or maintain any visible balance metadata.

## Interface

### Arguments and result

| Argument | Established use |
|---|---|
| Container address | Its 32-bit field at byte offset +16 supplies the address of an anchor object. The anchor’s field at +4 is the distinguished link. |
| Selected-record address | Identifies the record whose +8 link initially supplies the promoted-record address. Its +4 and +8 fields participate in the transformation. |

There is no return value, status code, or explicit error result.

The supplied representation uses 32-bit address-bearing arguments and 32-bit link fields. The contract does not establish compatibility with wider native pointers or define an ABI for another execution environment.

### Referenced storage

The following role names apply within one invocation:

- **Selected record:** the record identified by the second argument.
- **Promoted record:** the record identified by the selected record’s initial +8 field. This address is retained throughout the operation.
- **Transferred record:** a record reached through the promoted record’s +12 field when the relevant read is nonzero.
- **Previous parent:** the record identified by a read of selected +4 when the nondistinguished branch needs a parent.
- **Anchor object:** the object identified by container +16.
- **Distinguished slot:** the anchor object’s field at +4.

| Storage | Byte offset | Access width | Use |
|---|---:|---:|---|
| Container | +16 | 32 bits | Read anchor address |
| Anchor object | +4 | 32 bits | Read distinguished link; conditionally replace it |
| Selected record | +4 | 32 bits | Read parent-like value; finally replace it |
| Selected record | +8 | 32 bits | Read promoted address; replace with transfer link |
| Promoted record | +4 | 32 bits | Receive selected record’s current parent-like value |
| Promoted record | +12 | 32 bits | Read transfer link; finally receive selected address |
| Transferred record | +4 | 32 bits | Conditionally receive selected address |
| Previous parent | +12 | 32 bits | Read for branch selection; conditionally replace |
| Previous parent | +8 | 32 bits | Replace on the parent-link fallback branch |

No access to offset +0 is visible. These accesses do not establish complete object sizes, allocation boundaries, byte order, or alignment requirements.

### Preconditions

For the ordinary structural result described below, all required accesses must be valid, and all write targets must be writable. In particular:

- Selected and promoted records must be usable records, ordinarily distinct.
- Selected +8 must initially identify the promoted record.
- Promoted +12 must be zero or identify a usable transferred record with writable storage at +4.
- Container +16 must identify a usable anchor object.
- If the distinguished slot does not equal the selected-record address, selected +4 must identify a usable previous parent.
- For a valid parent-linked structure on that nondistinguished branch, the previous parent must actually reference the selected record through a child link.
- The relevant fields must remain stable except for this function’s own writes.
- Structural roles must not overlap in ways that make an earlier write change a later read.

These are caller-side conditions for the ordinary interpretation, not conditions checked by the function.

Address calculations use byte offsets +4, +8, +12, and +16. No bounds, overflow, or address-validity checks are visible. The effective addresses must be usable in the execution environment; behavior at arithmetic or address-space boundaries is not established.

## Behavior

### Transfer-link changes occur first

The promoted-record address is obtained from selected +8 and retained. The function reads promoted +12 and copies that 32-bit value into selected +8.

It then reads promoted +12 again to determine whether a transferred-record parent-link write is required. If that read is nonzero, the destination address is obtained through another read of promoted +12, and the selected-record address is written at that destination’s +4 field.

These are distinct reads. The transfer value copied into selected +8 is not a saved value reused for every subsequent transfer-related access. With stable, nonoverlapping storage, the reads agree. With aliasing or interference, they need not agree; the ordinary final-state description then does not apply unconditionally.

After this conditional write, the function reads selected +4 and copies its current 32-bit contents into promoted +4. Thus, the copied value is read after the earlier transfer-related writes, not captured at entry.

### Distinguished-link selection precedes parent-link selection

The function next reads container +16 to locate the anchor object and reads the distinguished slot at anchor +4.

If the distinguished slot equals the selected-record address, that slot is replaced with the retained promoted-record address. This branch does not inspect a previous parent’s child fields.

Otherwise, the function reads selected +4 again to obtain the previous-parent address. That read is separate from the earlier read used to populate promoted +4. It then reads previous-parent +12:

- If previous-parent +12 equals the selected-record address, that +12 field is replaced with the promoted-record address.
- Otherwise, previous-parent +8 is replaced with the promoted-record address.

The fallback does not read or validate previous-parent +8 before overwriting it.

Consequently, child-link precedence applies **only when the distinguished slot does not identify the selected record**. On that branch, if both parent child fields identify the selected record, +12 is chosen. If neither identifies it, +8 is still overwritten. When the distinguished slot identifies the selected record, neither of those parent-child cases controls the incoming-link update.

### Final relationship writes

After redirecting the distinguished slot or a previous-parent child slot, the function writes the selected-record address into promoted +12. It then writes the promoted-record address into selected +4 and returns.

The order matters: the transfer adjustment and promoted parent-like field update occur before incoming-link selection, and incoming-link replacement occurs before these final two writes. This describes the visible static sequence, not atomicity or a cross-thread memory-order guarantee.

### Ordinary final state

With valid, stable, nonoverlapping records:

| Field or slot | Value after return |
|---|---|
| Selected +8 | Promoted record’s former +12 value |
| Transferred +4, if the transfer link was nonzero | Selected-record address |
| Promoted +4 | Selected record’s former +4 value |
| Chosen incoming slot | Promoted-record address |
| Promoted +12 | Selected-record address |
| Selected +4 | Promoted-record address |

The chosen incoming slot is the distinguished slot when it initially identifies the selected record. Otherwise, it is previous-parent +12 if that field identifies the selected record, and previous-parent +8 if it does not.

Under the inferred tree interpretation, the promoted record takes the selected record’s former position, the selected record becomes its right child, and the promoted record’s former right subtree becomes the selected record’s left subtree.

Selected +12 and promoted +8 are not direct write targets. They remain unchanged only insofar as they do not overlap another write target.

### Boundary and malformed inputs

A zero transfer link suppresses the transferred-record +4 write, but zero is still copied into selected +8.

There is no corresponding check that the promoted-record address is nonzero before accessing promoted +12. Nor are the container, selected record, anchor, or required previous parent explicitly validated.

On the distinguished branch, selected +4 is copied into promoted +4 but is not used to access a previous-parent object. A zero or sentinel parent-like value can therefore be copied on that branch if all other accesses are valid.

Malformed parent linkage is not rejected. **On the nondistinguished branch, with selected +4 identifying the previous parent**, the +12 comparison alone selects between the two parent write targets. No inconsistency status is produced when the fallback overwrites a +8 field that did not identify the selected record.

No rollback or recovery path is visible. If execution fails after a write, the function provides no mechanism to undo earlier changes.

## Side effects

The function mutates caller-reachable storage in place. It does not produce a separate result object.

For ordinary nonoverlapping inputs, the operation performs:

- Five 32-bit writes when the transfer link is zero.
- Six 32-bit writes when the transfer link is nonzero.

Exactly one write redirects an incoming link: distinguished slot, previous-parent +12, or previous-parent +8.

Container +16 is read but is not a direct write target. No named global storage is directly accessed in the supplied body. Caller-provided addresses may nevertheless identify globally reachable or shared storage.

No allocation, deallocation, logging, I/O, callback, or synchronization operation is visible. There is no established acquisition, release, or transfer of allocation ownership. Changing reachability does not itself establish an ownership policy.

The operation is not transactional. Intermediate storage can represent an incomplete rearrangement, and no observer-consistency or concurrency-safety guarantee is established.

## Dependencies

F-c9e19efcf19b has no visible function-call dependencies.

Its external requirements are usable address-bearing values, the referenced memory layout, writable target fields, and sufficient object lifetimes for the accesses performed.

No allocator, destructor, comparator, validator, or external state-preservation contract is evidenced. No unknown callee’s preservation of globals or management of memory is assumed.

## Acceptance examples

These are conditional consequences of the static body, not executed tests. Unless an example says otherwise, assume stable, usable, nonoverlapping storage and representable effective addresses. A is the selected record; B is the promoted record.

### Distinguished selected record with a transfer record

Initially:

- The distinguished slot identifies A.
- A +8 identifies B.
- A +4 contains value P.
- B +12 identifies C.

After return:

- A +8 identifies C.
- C +4 identifies A.
- B +4 contains P.
- The distinguished slot identifies B.
- B +12 identifies A.
- A +4 identifies B.

No previous-parent child field is inspected. A +12 and B +8 remain unchanged under the nonoverlap assumption.

### Distinguished selected record with a zero transfer link

Initially, the distinguished slot identifies A, A +8 identifies B, and B +12 is zero.

After return, A +8 is zero, B +4 contains A’s former +4 value, the distinguished slot identifies B, B +12 identifies A, and A +4 identifies B. No transferred-record +4 write occurs.

A’s former +4 value need not identify a usable parent object for this branch, because it is copied rather than dereferenced as a parent.

### Nondistinguished selected record referenced through parent +12

Initially:

- The distinguished slot does not identify A.
- A +4 identifies P.
- P +12 identifies A.
- A +8 identifies B.

The incoming-link update replaces P +12 with B. P +8 is not directly written, and B +4 receives P. The selected, promoted, and optional transferred-record fields receive the other changes specified above.

### Nondistinguished selected record referenced through parent +8

Initially:

- The distinguished slot does not identify A.
- A +4 identifies P.
- P +12 does not identify A.
- P +8 identifies A.
- A +8 identifies B.

The incoming-link update replaces P +8 with B. P +12 remains unchanged, and B +4 receives P.

### Neither previous-parent child link identifies the selected record

Initially, the distinguished slot does not identify A, A +4 identifies P, and neither P +8 nor P +12 identifies A.

If all required accesses remain valid, the function still writes B into P +8. It provides no inconsistency indication. This is malformed-link behavior, not a valid-tree guarantee.

### Both previous-parent child links identify the selected record

Initially:

- The distinguished slot does not identify A.
- A +4 identifies P.
- Both P +8 and P +12 identify A.
- A +8 identifies B.

The parent +12 comparison succeeds, so P +12 is replaced with B. P +8 continues to identify A under the nonoverlap assumption. No uniqueness check occurs.

Both child fields identifying A is not sufficient by itself to select this outcome: the nondistinguished condition and the A-to-P parent link are required.

### Distinguished selection takes precedence over duplicate parent child links

Initially, the distinguished slot identifies A, A +4 identifies P, and both P +8 and P +12 identify A.

The function replaces the distinguished slot with B. It does not inspect P +12 to select an incoming link and does not directly write either P child field. B +4 still receives P through the earlier parent-like value copy.

## Uncertainties

The following remain **documented_only**:

- **Data model:** tree, root, parent, left-child, and right-child meanings are inferred from the link transformation.
- **ABI and representation:** calling convention, byte order, alignment, and compatibility with wider address representations are not established.
- **Invalid accesses:** the exact fault, exception, or language-level outcome of unusable addresses is not established.
- **Arithmetic boundaries:** no overflow guard is visible; exact behavior when byte-offset address calculations exceed usable limits is unresolved.
- **Aliasing:** no overlap check is present. Earlier writes can alter later reads, including transfer-link and parent-like reads. No general alias-safe final-state guarantee is established.
- **Concurrency:** locking, atomicity, visibility, and safe concurrent observation are not established.
- **Ownership and lifetime:** the function supplies no allocation-lifetime policy.
- **Higher-level invariants:** ordering, balance, metadata, record counts, and application-level consistency are outside the established evidence.

## Evidence

This contract is based on the supplied static function body, evaluated independently of the prior draft.

The body establishes the accessed offsets and 32-bit field widths, retained promoted-record address, repeated transfer-link reads, repeated selected-parent read on the nondistinguished branch, distinguished-link precedence, parent +12 comparison, unchecked +8 fallback, write ordering, lack of visible calls, and void return.

The right-rotation interpretation is inferred intent. Complete types, ownership rules, address-fault semantics, and broader invariants are not established.

No runtime tests or fault experiments have been performed. The acceptance examples state conditional static consequences, not observed execution results.
