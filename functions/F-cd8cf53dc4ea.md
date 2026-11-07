<!-- Trusted provenance: F-cd8cf53dc4ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd8cf53dc4ea

## Purpose

F-cd8cf53dc4ea adjusts accounting associated with a record: it subtracts a caller-supplied amount using the record’s existing classification fields, replaces those fields, and adds an amount extracted from the record’s first word using the replacement classifications.

It also decrements and increments classification counters and updates a high-water field using an unsigned comparison.

**Established behavior:** subtraction uses the supplied amount, whereas addition uses a separately extracted, masked value. They need not match. Existing and replacement classification indices also need not match.

**Inferred intent:** the routine appears to reclassify or resize an accounted record while maintaining category totals, aggregate totals, a distribution of record counts, and a peak aggregate. These meanings are interpretations of the access pattern, not independently established domain semantics.

The operation is an ordered sequence of in-place changes, not a transactional update. Intermediate accounting states exist across dependency calls.

## Interface

The routine has four inputs and no explicit return value:

| Input | Representation | Role |
|---|---|---|
| Accounting base | 32-bit address-valued input | Base address for accounting fields and indexed entries. |
| Record address | Address of writable storage | Supplies a 32-bit packed word and two 16-bit classification fields. |
| Removal amount | 32-bit integer | Subtracted from four accounting locations. |
| Replacement primary index | 32-bit integer | Used at full width for one accounting address calculation; its low 16 bits are stored in the record. |

The record’s minimum directly accessed extent is eight bytes:

| Byte offset | Width | Access |
|---|---:|---|
| 0 | 32 bits | Read; masked to obtain the addition amount. |
| 4 | 16 bits | Read as the previous primary index, then overwritten with the low 16 bits of the replacement primary index. |
| 6 | 16 bits | Read as the previous secondary index, then overwritten with a dependency result. |

The old primary index is obtained by reading the 32-bit word at offset 4 and retaining its low 16 bits. The old secondary index is obtained by a separate 16-bit read at offset 6. Their interpretation as adjacent halves of one word is consistent with a little-endian representation; the supplied material does not separately establish the platform’s byte order.

The accounting storage has the following directly accessed layout. All listed entries are 32 bits wide:

| Offset from accounting base | Role in this contract |
|---|---|
| 0x0E0 + 4 × secondary index | Secondary-category accounting entry |
| 0x220 | Secondary aggregate |
| 0x224 + 4 × primary index | Primary-category accounting entry |
| 0x30C | Primary aggregate |
| 0x310 + 4 × dependency-selected index | Distribution counter |
| 0x390 | Overall counter |
| 0x394 | High-water field |

These labels describe usage rather than proven business meaning. No array bounds or non-overlap guarantees are supplied.

### Preconditions

For normal completion:

- Every directly accessed address must refer to accessible storage with the required read or write permission.
- Every index-derived address must be valid, including indices returned by dependencies.
- Storage must support the observed 16-bit and 32-bit accesses under the target’s alignment rules.
- Both dependencies must return normally.
- If meaningful independent accounting entries are required, the caller must provide a layout and indices that prevent unintended overlap.

There are no explicit null checks, index checks, amount checks, or overflow checks.

## Behavior

The following order is established for the routine’s own operations. Dependency behavior remains unconstrained except for the result widths consumed by this routine. Consequently, any field read after a dependency returns may reflect changes made by that dependency.

### 1. Remove the supplied amount from existing accounting

The routine:

1. Reads the record word at offset 4 and retains its low 16 bits as an unsigned primary index.
2. Reads the primary-category entry selected by that index, subtracts the removal amount, and writes the result back.
3. Reads the primary aggregate at offset 0x30C, subtracts the removal amount, and writes it back.
4. Reads the 16-bit secondary index at record offset 6.
5. Reads the secondary-category entry selected by that index, subtracts the removal amount, and writes the result back.
6. Reads the secondary aggregate at offset 0x220, subtracts the removal amount, and writes it back.

The secondary index is read after the first two accounting writes. It is not established that both old indices are captured together. If the record overlaps accounting storage, those earlier writes can affect the subsequent secondary-index read.

The removal amount is not derived from the packed word and is not validated against any existing total.

### 2. Remove one distribution contribution

The routine calls F-09dcb55e8e0a with no explicit argument in the supplied representation.

After the call returns, it uses the returned 32-bit integer as a distribution index. It decrements the selected distribution counter by one, then decrements the overall counter at offset 0x390 by one.

The first distribution selection must not be assumed to classify the removal amount or any particular record field. The visible call does not establish such a relationship.

The distribution counter and overall counter are read after the dependency returns. Their prior values are not saved across the call.

### 3. Capture the addition amount and replace the record classifications

After the decrements, the routine reads the record’s first 32-bit word and applies the mask **0x3FFFFFFC**.

The resulting addition amount:

- Has bits 31 and 30 cleared.
- Has bits 1 and 0 cleared.
- Ranges from zero through 0x3FFFFFFC.
- Is always divisible by four.

The original word is not overwritten by this masking operation.

The routine then calls F-unknown-01, an opaque label assigned here to the external dependency absent from the supplied dependency map. No explicit arguments are visible. Its result is consumed as an unsigned 16-bit value.

Following that call, the routine performs these record writes in order:

1. Writes the 16-bit dependency result at record offset 6.
2. Writes the low 16 bits of the replacement primary index at record offset 4.

The addition amount was captured before this dependency call. Changes made by the dependency to the record’s first word therefore do not change the amount used by the remaining direct operations.

The replacement primary index has two distinct uses:

- Its full 32-bit input value selects the primary accounting destination.
- Only its low 16 bits are stored in the record.

The routine does not reconcile those uses. A value outside the unsigned 16-bit range can therefore select an accounting address inconsistent with the index subsequently stored in the record.

### 4. Add the captured amount to replacement accounting

Using the captured addition amount, the routine performs these read-modify-write operations in order:

1. Adds it to the primary-category entry selected by the full-width replacement primary index.
2. Adds it to the primary aggregate at offset 0x30C.
3. Adds it to the secondary-category entry selected by the captured 16-bit result from F-unknown-01.
4. Adds it to the secondary aggregate at offset 0x220.

These operations read current storage values. They do not restore or reuse values from before either dependency call.

The secondary destination uses the captured dependency result, rather than rereading the record’s secondary field. If overlapping writes alter that record field, the destination selection still uses the captured result.

### 5. Add one distribution contribution

The routine calls F-09dcb55e8e0a again, this time with the captured addition amount as an explicit argument.

After the call returns, it increments the distribution counter selected by the returned 32-bit integer.

The first and second results are independent for purposes of this contract. Equal indices, distinct indices, out-of-range indices, and dependency-side state changes are not ruled out by the supplied evidence.

### 6. Update the high-water field and restore the overall contribution

The routine reads the high-water field at offset 0x394 into a candidate value. It then compares the high-water field with the primary aggregate at offset 0x30C, treating both as unsigned 32-bit values.

If the high-water field is less than the primary aggregate, it reads the primary aggregate for use as the candidate. It then writes the candidate to offset 0x394, including when the comparison does not select a new value.

Finally, it reads the overall counter at offset 0x390, increments it by one, and returns without a value.

Under stable storage, this high-water operation stores the unsigned maximum of the previous high-water value and the current primary aggregate. It occurs after the second F-09dcb55e8e0a call, so that call’s changes can affect the comparison.

### Arithmetic and address formation

Both existing record indices and the replacement secondary index are zero-extended from 16 bits before scaling by four.

The replacement primary index and both distribution indices participate in full-width address calculations. There is no visible truncation to 16 bits for those calculations and no visible range enforcement.

Accounting and counter stores are 32 bits wide. For non-overflowing signed operations, ordinary integer arithmetic describes the result. Addition of the masked amount is expressed through unsigned 32-bit values and has modulo-2³² result semantics.

Several subtractions, counter changes, and address expressions are presented as signed operations. The supplied material alone does not establish a portable signed-overflow policy. A target-width wraparound interpretation must therefore be identified as a machine-level assumption where relevant, not treated as a universally defined source-language guarantee.

## Side effects

Direct effects include:

- Four subtraction-based accounting writes.
- One distribution decrement and one overall-counter decrement.
- Two 16-bit record writes.
- Four addition-based accounting writes.
- One distribution increment.
- An unconditional high-water-field write.
- One overall-counter increment.

Additional effects from dependencies are unknown.

The routine does not directly allocate or release memory, transfer an ownership token, return storage, or clear the record’s first word. This does **not** establish that its dependencies preserve ownership, avoid allocation, avoid release, or preserve any global state.

No rollback, error result, or recovery branch is visible. If an access fails or a dependency does not return normally, earlier writes may already have occurred.

No synchronization is visible. Atomicity, thread safety, and coherent observation by concurrent readers are not established.

Aliasing matters. For example, an indexed accounting address can coincide with an aggregate, a counter, or part of the record if the supplied addresses and indices permit it. Such cases follow the ordered accesses above, not a model of independent fields.

## Dependencies

| Opaque ID | Established interaction | Unknowns |
|---|---|---|
| F-09dcb55e8e0a | Called once without a visible explicit argument, then once with the captured addition amount. Each result selects a 32-bit distribution counter at a four-byte stride. | First-call input convention, result bounds, determinism, state changes, memory effects, ownership effects, failure behavior. |
| F-unknown-01 | Called once after capturing the addition amount. No explicit arguments are visible. Its result is consumed as 16 bits and used for both a record field and secondary accounting selection. | Identity in the supplied dependency map, input convention, selection policy, side effects, ownership effects, failure behavior. |

F-unknown-01 is a document-local opaque label, not a supplied provenance identifier.

The two different visible call forms of F-09dcb55e8e0a do not establish a reliable complete signature. In particular, the first call must not be rewritten conceptually as an explicit call with the removal amount.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless otherwise stated, storage is valid and non-overlapping, dependencies return normally without changing relevant storage, and arithmetic stays within non-overflowing ranges.

### Distinct old and replacement categories

Suppose:

- The removal amount is 12.
- The packed word is 0x80000017.
- Both replacement categories differ from their respective old categories.
- The first distribution result is 2 and the second is 5.

The masked addition amount is 20. Acceptance requires:

- Each old category entry decreases by 12.
- Each replacement category entry increases by 20.
- Each aggregate increases by a net 8.
- Distribution entry 2 decreases by one, and entry 5 increases by one.
- The overall counter returns to its initial value.
- The record stores the replacement primary index’s low 16 bits and the external dependency’s 16-bit result.
- The high-water field receives the unsigned maximum of its initial value and the resulting primary aggregate.

### Same categories and same amount

If both replacement categories equal their old categories and the masked addition amount equals the removal amount, each category total and aggregate returns to its initial value under the stated assumptions.

This does not eliminate intermediate writes. The record is still rewritten, dependencies are still called, and the high-water field is still written.

If both distribution calls also select the same valid entry, that counter returns to its initial value after a decrement followed by an increment.

### Zero addition amount

If the packed word is 0xC0000003, the masked addition amount is zero.

The removal phase still subtracts the removal amount. Classification replacement, both distribution-counter changes, both overall-counter changes, all four addition-phase stores, and the high-water update still occur.

### Different stored and effective primary indices

For a replacement primary index of 0x00010002, the record receives 0x0002, but the primary addition address is calculated using 0x00010002.

Acceptance of this case requires that the full-width destination address be valid. No rejection or normalization to index 2 is visible.

### Dependency changes an aggregate

If the second F-09dcb55e8e0a call changes the primary aggregate before returning, the high-water comparison uses the aggregate present after that call.

An expected result computed solely from the four direct accounting additions would be insufficient.

### Unsigned high-water comparison

With a current high-water value of 0x80000000 and a primary aggregate of 0x7FFFFFFF at comparison time, the high-water value remains 0x80000000 under stable storage. The comparison is unsigned.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the record, classifications, totals, and distribution counters.
- Complete calling conventions and signatures of dependencies.
- Whether the first distribution selection has an implicit relationship to record state or the removal amount.
- Permitted index ranges, allocation extents, and intended non-overlap constraints.
- Whether callers guarantee that the replacement primary index fits in 16 bits.
- Whether callers guarantee that the removal amount equals a prior recorded size.
- Target-specific signed overflow, address overflow, byte order, and alignment behavior.
- Dependency effects on globals, pointed-to storage, memory ownership, exceptions, or process state.
- Validity under concurrent access or asynchronous mutation.

No runtime tests have been performed.

## Evidence

The established behavior comes from the supplied static representation: access widths and offsets, ordered reads and writes, explicit masks, visible dependency calls, result consumption, unsigned comparison, and absence of explicit validation or rollback branches.

The supplied dependency map identifies F-09dcb55e8e0a. It does not identify the external dependency labeled F-unknown-01 in this document.

Accounting terminology and the reclassification interpretation are inferred intent. Dependency semantics, domain invariants, and runtime outcomes remain unverified.
