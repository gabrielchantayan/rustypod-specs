<!-- Trusted provenance: F-0ff2c000bc5c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ff2c000bc5c

## Purpose

F-0ff2c000bc5c advances a caller-maintained cursor through address-bearing fields in linked records. It updates the supplied cursor storage directly and returns no value.

Two modes are established by the supplied static representation:

- When the initial record’s field at byte offset 12 is nonzero, the function selects that target and repeatedly follows nonzero fields at byte offset 8.
- When the initial record’s field at byte offset 12 is zero, the function obtains candidates through fields at byte offset 4. It accepts candidates while their offset-12 fields equal the current cursor, then applies a separate final replacement condition.

These relationships are consistent with finding an in-order successor in a parent-linked binary tree. That is **inferred intent**, not an established data-structure contract. Tree ordering, root conventions, sentinels, and traversal exhaustion are not specified by the available evidence.

## Interface

### Argument and result

The sole argument identifies a readable and writable integer-valued storage location, called the **cursor slot**.

The slot contains an address-bearing integer identifying the current record. The argument therefore identifies the cursor storage, not the record directly.

There is no return value. Results are conveyed through writes to the cursor slot. An invocation can perform multiple writes, exposing successive traversal positions in its represented access sequence.

### Accessed layout

Record-relative accesses use these byte displacements:

| Byte offset | Value accessed | Established use |
|---|---|---|
| 4 | Address-bearing integer | Supplies the first upward candidate and subsequent upward candidates |
| 8 | Address-bearing integer | Supplies successive descent candidates; zero ends descent |
| 12 | Address-bearing integer | Selects the initial branch, participates in upward comparisons, and controls final replacement |

No field at byte offset 0 is directly accessed. No complete record size or additional layout is established.

“Parent,” “left child,” and “right child” are possible interpretations of offsets 4, 8, and 12, respectively. They are not required meanings.

### Widths and arithmetic

The supplied representation uses an integer-typed cursor slot and integer-typed field accesses. All accessed values use that represented integer type.

The offsets are exactly 4, 8, and 12 bytes. They are not element indices, and their values do not independently establish the integer width.

A 32-bit interpretation would mean a four-byte cursor value and four-byte accessed fields. The available representation alone does not establish that interpretation as the original ABI. Exact integer width, pointer width, byte order, alignment requirements, and integer-to-address conversion rules remain **documented_only** unknowns.

Address-bearing integers undergo addition before the resulting locations are accessed. The function supplies no explicit overflow handling, address-range validation, or alignment correction. Under a signed-integer interpretation, execution also requires those additions not to overflow the represented signed type.

No arithmetic changes a link value merely to advance by one record. The only address displacements are the fixed field offsets.

### Preconditions

For defined, terminating execution, the caller must supply:

- A valid readable and writable cursor slot.
- An initial cursor value permitting an offset-12 field read.
- Valid storage for every field reached by the selected traversal.
- Address arithmetic and address conversions supported by the execution environment.
- A path that reaches the relevant stopping condition.

The function does not verify these conditions.

In particular, it does not test whether the initial cursor is zero before accessing the initial record’s offset-12 field. It also does not test an upward candidate for zero before accessing that candidate’s offset-12 field.

Disjoint cursor and record storage is the straightforward usage model, but disjointness is not enforced. Aliasing can make a cursor write modify a field that the function subsequently reads.

## Behavior

### Initial selection

The function reads the initial record address from the cursor slot, then reads that record’s field at byte offset 12.

A nonzero field selects descent. Zero selects upward traversal.

For descent, the value already read is retained as the first candidate. There is no additional read of the initial record’s offset-12 field before the first descent write.

### Descent

The function first writes the selected candidate into the cursor slot. It then reads that candidate’s field at byte offset 8.

If the field is nonzero, its value becomes the next candidate. The function writes that value into the cursor slot before reading the new candidate’s offset-8 field. This continues until an offset-8 read produces zero.

Established consequences:

- At least one cursor write occurs.
- Every descent candidate is written before its offset-8 field is read.
- The final cursor identifies the candidate whose offset-8 field supplied the terminating zero.
- The terminating zero is not written to the cursor slot.
- No offset-4 field is accessed on this branch.
- After initial selection, no offset-12 field is accessed on this branch.

Descent follows the retained candidate values. It does not reread the cursor slot to obtain the address for each offset-8 access.

The write-before-read ordering is significant. A later invalid access can occur after earlier cursor updates. If cursor storage overlaps a candidate’s offset-8 field, the preceding cursor write can alter the value subsequently read there.

### Upward traversal

When the initial offset-12 field is zero, the function reads the cursor slot again and obtains its first candidate from the identified record’s offset-4 field.

For each candidate, it compares:

- The candidate’s offset-12 field.
- The current value in the cursor slot.

Both values are read for the comparison. Their relative read order is not established by the supplied expression.

If they are equal, the function writes the candidate into the cursor slot. It then reads the accepted candidate’s offset-4 field to obtain the next candidate.

The offset-4 access follows the cursor write and uses the retained accepted candidate address, not a new cursor-slot read. The next comparison does read the cursor slot again.

These comparisons continue until one is unequal.

Established consequences:

- A candidate’s offset-12 field is read before that candidate is accepted.
- Each successful comparison causes a cursor write.
- A write can store the same value already present.
- A successful step writes the cursor before reading the accepted candidate’s offset-4 field.
- A zero candidate receives no special stopping treatment before its offset-12 field is accessed.
- The cursor is not permanently cached at its original value.

If the first comparison fails, the loop performs no cursor write. If several comparisons succeed, all corresponding cursor updates have already occurred when traversal reaches the final check.

### Final replacement condition

After the first unequal upward comparison, the function retains the rejected candidate.

It reads the current record address from the cursor slot and then reads that record’s offset-12 field:

- If this field differs from the retained candidate, the function writes the candidate into the cursor slot.
- If it equals the retained candidate, the function performs no final write.

The failed loop comparison and final condition inspect different relationships:

| Check | Offset-12 field examined | Compared against |
|---|---|---|
| Upward loop comparison | Candidate’s field | Current cursor |
| Final replacement condition | Current cursor record’s field | Candidate |

These checks cannot be replaced by one comparison or an unconditional final assignment.

### Termination and errors

The function has no iteration limit, cycle detection, explicit error result, or recovery path.

A descent cycle can cause repeated cursor writes indefinitely. An upward cycle can likewise prevent termination when each comparison continues to succeed.

Invalid storage or unsupported address arithmetic can prevent normal completion. The precise failure mechanism is not established. No rollback is performed by the represented function; earlier writes precede the failing access in its access sequence.

Zero is not established as a supported exhaustion result. In descent, zero stops traversal without being stored. In upward traversal, a zero candidate is subjected to an offset-12 access before the final condition could select it. The contract does not invent a clean end-of-traversal response for that case.

## Side effects

The only explicit writes are integer-valued writes to the cursor slot.

The function does not explicitly write through a record-relative field address. Nevertheless, record storage can change when it overlaps the cursor slot. Such changes can influence later reads within the same invocation.

The represented function contains:

- No allocation or deallocation.
- No ownership transfer operation.
- No callback invocation.
- No explicit global-state access.
- No synchronization, locking, or atomic-update guarantee.

The caller retains responsibility for the lifetime and accessibility of the cursor slot and reached storage. The function does not establish or extend their lifetimes.

Intermediate cursor updates belong to the represented access sequence. This does not establish that another thread can safely observe them. Concurrent observation and mutation require environmental guarantees absent from the supplied evidence.

## Dependencies

There are no explicit calls to other functions. Consequently, there are no dependency function IDs to enumerate.

Execution depends on caller-provided memory and the target’s integer, pointer, alignment, and memory-access semantics. These are environmental prerequisites, not hidden validation or ownership services.

No behavior is attributed to unknown callees, including preservation of globals or release of memory.

## Acceptance examples

These examples are conditional predictions from static evidence. No runtime tests have been performed.

Unless an example states otherwise:

- All named records are distinct.
- The cursor slot is separate from record storage.
- All required arithmetic, conversions, and field accesses are valid.
- Link values remain stable throughout the invocation.

### Immediate descent result

Given:

- The cursor initially identifies record A.
- A’s offset-12 field identifies B.
- B’s offset-8 field is zero.

Expected behavior:

- The function writes B into the cursor slot.
- It then reads B’s offset-8 field.
- It returns with the cursor identifying B.
- It does not read any offset-4 field or write the terminating zero.

### Multiple descent steps

Given:

- **The cursor initially identifies record A.**
- A’s offset-12 field identifies B.
- B’s offset-8 field identifies C.
- C’s offset-8 field identifies D.
- D’s offset-8 field is zero.

Expected behavior:

- A’s offset-12 field selects descent and supplies B.
- Cursor writes occur in the order B, C, D.
- Each write precedes the corresponding candidate’s offset-8 read.
- The final cursor identifies D.
- The terminating zero is not written.

The initial-cursor condition is essential: the listed links alone do not select A as the starting record or establish this write sequence.

### First upward candidate selected by the final condition

Given:

- The cursor initially identifies A.
- A’s offset-12 field is zero.
- A’s offset-4 field identifies B.
- B’s offset-12 field does not identify A.
- B’s address is nonzero.

Expected behavior:

- The first upward comparison fails.
- No loop write occurs.
- The final condition reads A’s offset-12 field again.
- Because zero differs from B’s address, the function writes B into the cursor slot.

### One successful upward step followed by replacement

Given:

- The cursor initially identifies A.
- A’s offset-12 field is zero.
- A’s offset-4 field identifies B.
- B’s offset-12 field identifies A.
- B’s offset-4 field identifies C.
- C’s offset-12 field does not identify B.
- A and C have different address values.

Expected behavior:

- The first comparison succeeds, and the cursor is written with B.
- B’s offset-4 field then supplies C.
- C’s comparison fails.
- The final condition reads B’s offset-12 field, which identifies A.
- Because A differs from C, the function writes C.

The cursor-write sequence is B, then C.

### Final replacement omitted

Given:

- The cursor initially identifies A.
- A and B have nonzero addresses.
- A’s offset-12 field is zero.
- A’s offset-4 field identifies B.
- B’s offset-12 field identifies A.
- B’s offset-4 field identifies A.

Expected behavior:

- The first comparison succeeds and writes B.
- The next candidate is A.
- A’s offset-12 field is zero, so the next comparison fails against cursor B.
- The final condition finds that B’s offset-12 field equals candidate A.
- No final write occurs; the cursor remains B.

This establishes the final guard’s effect for these relationships, not a general sentinel convention.

### Zero upward candidate

Given:

- The cursor initially identifies A.
- A’s offset-12 and offset-4 fields are both zero.

Expected behavior:

- The function obtains zero as the first upward candidate.
- Before any upward cursor write, it attempts to read the offset-12 field associated with that candidate.
- A clean “no successor” return is not required.

Whether that access is valid, faults, or has another target-specific consequence remains unresolved.

### Cyclic descent

Given:

- The cursor initially identifies A.
- A’s offset-12 field identifies nonzero B.
- B’s offset-8 field identifies B itself.

Expected behavior:

- The function repeatedly writes B and rereads B’s offset-8 field.
- With stable accessible storage, it does not terminate or report a cycle.

## Uncertainties

### Inferred intent

The observed field relationships fit a parent-linked binary-tree successor operation:

- Offset 4 as parent.
- Offset 8 as left child.
- Offset 12 as right child.

The final condition may accommodate a special root or sentinel arrangement. Neither this interpretation nor any particular sentinel invariant is established.

### documented_only

The following remain unresolved:

- Exact integer and pointer widths.
- Original ABI, byte order, and alignment requirements.
- Integer-to-address conversion semantics.
- Whether a zero-based candidate access can be valid in the target environment.
- Complete record layout and allocation size.
- Tree ordering, balancing, root, sentinel, or end-iterator rules.
- Whether overlapping cursor and record storage is supported usage.
- Whether concurrent observation or mutation is permitted.
- Precise consequences of invalid addresses or unsupported arithmetic.

These unknowns do not authorize added validation, null handling, cycle recovery, ownership rules, or exhaustion behavior.

## Evidence

**Established static evidence:** The supplied representation has one cursor-slot argument, no return value, integer-typed accesses at byte offsets 4, 8, and 12, a branch selected by the initial offset-12 value, incremental cursor writes, and a distinct final replacement condition on the upward path. It contains no explicit function calls, allocation, deallocation, validation, synchronization, or error reporting.

**Inferred intent:** The access relationships resemble an in-order successor operation on parent-linked tree records. That interpretation does not establish the surrounding container’s invariants.

**Verification status:** Static analysis only. No runtime tests have been performed. The acceptance examples describe conditional outcomes, not observed executions.
