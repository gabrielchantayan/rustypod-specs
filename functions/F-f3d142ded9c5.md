<!-- Trusted provenance: F-f3d142ded9c5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f3d142ded9c5

## Purpose

F-f3d142ded9c5 processes a counted sequence of signed 32-bit identifiers using a context obtained from F-7fba0909c452.

For each nonzero identifier, it conditionally clears two context fields and classifies a selected unsigned 32-bit table word:

- An odd word causes processing to continue without a direct table, head, or counter update.
- A zero word causes a head-derived tagged value to be stored through a separately evaluated table address, followed by head and counter updates.
- A nonzero even word causes two dependency calls, the second annotated as non-returning.

A negative count selects conditional initialization of the context’s first signed 32-bit word. Zero count does not select initialization.

The table updates resemble insertion into a tagged free list. This is **inferred intent**, not evidence of memory release, object destruction, or ownership transfer.

## Interface

### Inputs and result

| Item | Established contract |
|---|---|
| Count | Signed 32-bit quantity. Negative, zero, and positive values select distinct paths. |
| Identifier sequence | Address of consecutive signed 32-bit identifiers, traversed in four-byte increments. |
| Result | No explicit return value or status. Ordinary paths return without a result. One path reaches a dependency annotated as non-returning. |

F-7fba0909c452 is called once before count classification. A null result causes immediate return without subsequent direct sequence, context-field, or initialization-global access.

With a non-null context:

- A negative count does not dereference the sequence.
- Zero count does not dereference the sequence or evaluate the initialization global.
- A positive count normally consumes exactly that many identifiers, unless a dependency or an unsupported operation prevents completion.

A zero identifier consumes one counted element; it does not terminate processing.

### Directly accessed storage

The following offsets are byte offsets from the context address returned by F-7fba0909c452.

| Offset | Width | Observed role |
|---|---:|---|
| 0x000 | 32 bits, signed destination | Conditional initialization destination |
| 0xAC8 | 32 bits | Address-bearing table-base field |
| 0xACC | 32 bits, signed | Counter read and incremented after a zero-entry store |
| 0xAD4 | 32 bits, signed | Head used to construct a tag, then replaced |
| 0xAD8 | 32 bits, signed | First identifier field, conditionally cleared |
| 0xADC | 32 bits, signed | Second identifier field, conditionally cleared |

A separate global expression participates in conditional initialization. **Its declaration, type, width, and access width are not supplied.** Its comparison with zero and assignment into the signed 32-bit destination do not establish those properties.

Initialization uses two separate evaluations of that global expression when the destination is eligible for assignment. The second evaluated value undergoes conversion to the destination’s type. The conversion rules and outcomes are not fully established without the global’s declaration and the applicable language and target rules.

A table access selects an unsigned 32-bit word using the address-bearing field at 0xAC8 plus four times the signed identifier. The zero-entry store independently evaluates that address expression again.

The representation uses a four-byte address-bearing context field. It does not independently establish a complete pointer model, address space, or target ABI.

### Preconditions and arithmetic

Every address actually reached must support the required access:

- Consumed sequence elements must be readable.
- Reached context fields must be readable and, where changed, writable.
- Each selected table word must be readable.
- A zero-entry store destination must be writable, including when its independently evaluated address differs from the lookup address.
- Initialization-global evaluations must be valid under the global’s actual declaration and access semantics, which remain unresolved.

There are no explicit identifier bounds, alignment, table-extent, overlap, or arithmetic-overflow checks. Negative identifiers are not rejected.

The signed arithmetic shown does not establish arbitrary modulo-\(2^{32}\) behavior:

- Multiplying an identifier by four is representable in signed 32-bit arithmetic for identifiers from −536,870,912 through 536,870,911. Adding the address-bearing field imposes a separate representability constraint.
- A nonnegative head from 0 through 1,073,741,823 can be shifted left once without exceeding the positive signed 32-bit range.
- Incrementing a counter of 2,147,483,647 is not established as wrapping.

These are arithmetic boundaries, not validations performed by the function. Unsupported arithmetic and address-conversion behavior remain **documented_only**.

## Behavior

### Context acquisition and count selection

The first operation is a call to F-7fba0909c452 without explicit arguments.

If that call returns null, the function returns. This limits subsequent direct accesses by F-f3d142ded9c5; it does not constrain effects of the dependency itself.

For a non-null result:

- Negative count selects conditional initialization.
- Zero count returns without sequence access or initialization.
- Positive count selects sequence processing.

For nonnegative counts, the representation tests whether the remaining count is zero and decrements that count before acting on the test result. The decrement on the terminating check changes zero to −1, which is representable. This local change is not a returned result.

Normal exhaustion of positive-count processing returns directly. It does not fall through to initialization.

### Element consumption and preliminary clearing

Each consumed element provides one signed 32-bit identifier. Sequence traversal advances by four bytes before the identifier’s subsequent processing.

If the identifier is zero, the function consumes the element and continues without accessing the identifier fields or table for that element.

For each nonzero identifier, direct accesses occur in this order:

1. Read the field at 0xAD8 and write zero there if it equals the identifier.
2. Read the field at 0xADC and write zero there if it equals the identifier.
3. Read the table-base field at 0xAC8.
4. Read the unsigned 32-bit word at the derived table address.

Both identifier fields may be cleared. These operations precede table classification, including classification that leads to dependency calls.

The accesses are incremental, not a snapshot. If supported overlap permits an earlier write to affect a later read, that later read uses the affected storage.

### Odd table word

Every captured table word whose low bit is set takes the continuation branch, including patterns with the high bit set.

The function performs no direct table, head, or counter update on this branch. Any preceding identifier-field clearing has already occurred.

### Zero table word

When the captured table word is zero, the function constructs a table store using:

- A second evaluation of the table-base field at 0xAC8, combined with the same identifier displacement.
- A read of the head at 0xAD4, shifted left by one bit and combined with a set low bit.

The representation does not establish the relative evaluation order of this second table-base access and the head-value calculation. Both contribute to the store; neither is specified as necessarily preceding the other.

After the store:

1. Write the current identifier to the head at 0xAD4.
2. Read the counter at 0xACC.
3. Add one and write the result to 0xACC.
4. Continue processing.

Thus, the table store precedes the head write, which precedes the counter read and update.

The store address is not explicitly retained from the initial lookup. It matches that lookup address only when the two base evaluations and applicable address arithmetic yield the same address. If a permitted execution model allows different base values, the store may target another word. That second destination is not separately read or classified before the write.

The counter increment uses the counter value read after the preceding writes. For example, supported overlap between the table destination and the counter can make the increment operate on the tagged value just stored.

For a representable nonnegative head, the tag is twice the head plus one. Negative-head shifts, unrepresentable shifts, and counter overflow have no explicit check or recovery.

### Nonzero even table word

When the captured unsigned word is nonzero and even:

1. Call F-66f29ea52eb3 with that captured word.
2. If it returns, call F-feee9156a257 with the same captured word.

The table word is not re-read between these calls. Changes made by the first dependency do not change the captured argument selected for the second.

F-feee9156a257 is annotated as non-returning. Under that annotation, processing does not reach another element or initialization. If the first dependency does not return, the second is not reached.

The annotation does not identify an exception, termination status, cleanup action, or other control-transfer mechanism.

If F-feee9156a257 returns contrary to its annotation, the supplied representation places the initialization checks after the call, rather than resuming the sequence loop. Actual behavior contrary to the annotation remains **documented_only**.

### Conditional initialization

The established ordinary entry to this path is a negative count with a non-null context.

The source-level evaluation and access order is:

1. Evaluate the initialization global and compare that evaluated value with zero.
2. If the comparison is equal, return without reading the first context word.
3. Otherwise, read the signed 32-bit context word at 0x000.
4. If that word is nonzero, return without assigning it and without a second global evaluation.
5. If that word is zero, evaluate the global again and assign the result to the signed 32-bit destination, subject to the applicable conversion to that destination type.

The first global evaluation controls eligibility for the context check. The second supplies the assignment operand. There is no established atomic snapshot.

The contract does **not** require the second evaluated value to be stored unchanged. Its source type is unknown, so conversion may matter even when both evaluations produce a stable value. The representation alone does not establish conversion outcomes for values outside the destination’s representable range or for an unknown source type.

If the execution model permits intervening changes, the two global evaluations may also produce different values. A second value that converts to zero can leave the destination zero despite the first comparison being nonzero. The function performs no second nonzero check before assignment.

These are source-level evaluations. The number and width of physical memory accesses cannot be established without the global declaration and applicable access semantics.

## Side effects

### Direct writes

Depending on the path, the function may:

- Clear either or both identifier fields.
- Store a head-derived tag through the recomputed table address.
- Replace the head with the current identifier.
- Increment the counter after the head write.
- Assign a separately evaluated initialization-global value, converted to the signed 32-bit destination type, to the context’s first word.

The function does not explicitly assign through the sequence pointer or explicitly assign the initialization global. Supported storage overlap could nevertheless cause a context or table write to affect storage also serving another role.

### Ordering and partial progress

Processing is not transactional; no rollback is shown.

Within a zero-entry update, write order is table destination, head, then counter. Relative evaluation order between the table-store address and head-derived value is unresolved.

Writes completed for an earlier element precede dependency calls reached for a later element. This does not guarantee preservation after those calls: dependency behavior is unknown.

No lock, atomic operation, memory barrier, or error-recovery mechanism is visible. Concurrent-access validity and visibility are not established. Separate expressions in the representation must not be treated as a guarantee of distinct physical loads under an unspecified memory model.

### Ownership and failures

The function directly manipulates storage and calls dependencies. It contains no explicit allocation or release operation.

No ownership transfer, object destruction, or lifetime guarantee follows from the tagging pattern. Dependencies may affect memory, globals, resources, or ownership; those effects are not established here.

Invalid accesses, unsupported arithmetic, and unresolved assignment-conversion cases have no explicit error status or recovery path. Their runtime consequences remain unspecified by this contract.

## Dependencies

| Function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-7fba0909c452 | Called once before count selection; its result supplies the context address or null | Return guarantee, lifetime, allocation, ownership, global effects, synchronization |
| F-66f29ea52eb3 | Receives the captured nonzero even unsigned 32-bit table word | Return guarantee, resource effects, memory changes, ownership effects |
| F-feee9156a257 | Called if the preceding dependency returns; receives the same captured word; annotated as non-returning | Non-return mechanism and memory, resource, global, and ownership effects |

No dependency is assumed to preserve globals, prior writes, context storage, or ownership.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless stated otherwise, assume valid, stable, non-overlapping storage, representable arithmetic, and dependency returns where continuation requires them.

1. **Null context.** With any count, a null result from F-7fba0909c452 causes immediate return without subsequent direct sequence, context-field, or initialization-global access.

2. **Zero count bypasses initialization.** With a non-null context and count zero, the first context word is unchanged. Neither the sequence nor the initialization global is evaluated directly.

3. **Negative count with representable initialization.** Suppose the global has an actual type for which each evaluation yields numeric 37 and conversion to the signed 32-bit destination produces 37. With a negative count and a zero destination, initialization assigns 37 without reading the sequence. This example does not establish the global’s width.

4. **First global value compares equal to zero.** On the negative-count path, the function returns without reading the first context word or evaluating the global a second time.

5. **Existing destination is nonzero.** If the first global value compares nonzero and the first context word is nonzero, the destination remains unchanged and no second global evaluation occurs.

6. **Second evaluation supplies a converted value.** Suppose the first global value compares nonzero, the destination is zero, and the second global evaluation yields a value whose conversion to signed 32-bit produces zero. The assignment writes zero. This example requires a source type and conversion for which that outcome is defined; it does not assume an arbitrary conversion rule.

7. **Zero identifier consumes one element.** Count one with identifier zero reads that element and returns without identifier-field or table access.

8. **Clearing before an odd-entry skip.** If both identifier fields contain 5 and the selected table word is 9, both fields are cleared. No direct table, head, or counter update follows for that element.

9. **Ordinary zero-entry update.** Identifier 5 selects zero; both table-base evaluations select the same address; head is 3 and counter is 8. The table destination receives 7, then the head receives 5, then the counter receives 9.

10. **Recomputed destination matters.** If a permitted execution model yields different bases for lookup and store, a zero lookup through the first base causes a tagged store through the second. The second destination need not contain zero. No relative order between the second base evaluation and the head read is implied.

11. **Duplicate identifier under stable storage.** Two occurrences of identifier 5 initially select a zero word. The first writes an odd tag and increments the counter. The second observes the odd tag and does not increment again.

12. **Captured dependency argument.** A selected word of 12 causes F-66f29ea52eb3 to receive 12. If it returns, F-feee9156a257 receives 12 even if the first dependency changed the table storage.

13. **Earlier progress before dependency effects.** If one element completes a zero-entry update and a later element selects a nonzero even word, the earlier writes precede the dependency calls. Their survival after those calls requires additional assumptions about dependency behavior.

## Uncertainties

The following remain **documented_only**:

- The initialization global’s declaration, type, width, physical access width, and access qualifiers.
- The applicable conversion from the global’s second evaluated value to the signed 32-bit destination, including outcomes outside the destination’s representable range.
- Whether repeated global evaluations correspond to separate physical reads or can observe changes.
- Semantic identities and valid lifetimes of the context, table, head, counter, and global.
- Valid identifier bounds and intended treatment of negative identifiers.
- Complete pointer, alignment, address-conversion, and arithmetic rules.
- Outcomes of unsupported signed shifts, arithmetic overflow, or invalid accesses.
- Permitted storage overlap and aliasing rules.
- Concurrency, synchronization, and volatile-access semantics.
- Whether repeated table-base evaluations yield the same value.
- Relative evaluation order of the zero-branch store address and head-derived value.
- Dependency ownership, preservation, resource, and control-transfer behavior.
- Runtime behavior if the annotated non-returning dependency returns.

These unknowns are not replaced with assumptions about ordinary global integers, unchanged assignment values, memory ownership, or dependency preservation.

## Evidence

This contract is based on the supplied static representation and opaque dependency mapping. Embedded comments, the prior draft, and reviewer statements are not independent behavioral evidence.

Established static evidence includes count-based selection, four-byte sequence traversal, the listed context accesses, unsigned table classification, separate table-base expressions, ordered updates, captured dependency arguments, and the non-return annotation.

For initialization, the evidence establishes a global expression compared with zero and, conditionally, a second evaluation assigned to a signed 32-bit context destination. It does **not** establish the global’s own type or width, physical read width, or unchanged preservation of its value during assignment.

The tagged-free-list interpretation is inferred intent. Ownership, synchronization, dependency preservation, and unresolved arithmetic or conversion outcomes are not established.

No runtime tests, smoke runs, dependency executions, or additional implementation inspection have been performed.
