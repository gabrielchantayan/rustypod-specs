<!-- Trusted provenance: F-caf3343b3be6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-caf3343b3be6

## Purpose

F-caf3343b3be6 inserts a four-byte value into storage described by a three-field control record.

Two paths are established:

- **Spare recorded capacity:** advance the recorded end, shift existing four-byte values toward higher addresses, and write the supplied value at the insertion address.
- **No spare recorded capacity:** calculate a larger capacity, request storage, invoke a range-processing dependency twice around an inserted-value write, perform an empty traversal of the old range, invoke a release-like dependency, and replace the control record.

Interpreting this as growable-array insertion is inferred intent. The dependency implementations are unavailable, so allocation, copying, destruction, and release semantics are not independently established.

The spare-capacity path has an important restriction: its backward traversal supports insertion at an existing element, not ordinary insertion at the end of the sequence.

## Interface

The function accepts three address-valued arguments and has no explicit return value.

| Argument | Established role |
|---|---|
| Control-record address | Identifies three consecutive readable and writable 32-bit fields. |
| Insertion address | Identifies the position used for insertion and as a range boundary on the replacement-storage path. |
| Value address | Identifies the four-byte value read when the inserted-element write is reached. |

The control record occupies 12 bytes:

| Byte offset | Name used here | Intended interpretation |
|---|---|---|
| 0 | Begin | Address of the first element. |
| 4 | End | Address immediately after the logical sequence. |
| 8 | Limit | Address immediately after the recorded storage capacity. |

These interpretations describe the apparent data structure; the function does not validate them.

### Widths and arithmetic

The supplied representation uses 32-bit addresses and integer quantities. Elements are transferred as four-byte values without visible interpretation of their contents.

The initial element count is calculated by:

- Taking the signed 32-bit difference between the initially captured End and a read of Begin.
- Right-shifting that signed difference by two.
- Interpreting the result as an unsigned 32-bit count.

The recorded capacity is calculated analogously from Limit and Begin and compared as an unsigned count.

Unsigned growth calculations operate modulo \(2^{32}\). Signed subtraction, signed addition, and right shifts of negative signed values require separate caution: their extreme or negative cases are **documented_only**, not silently assigned unsigned wraparound or a particular signed-shift rule.

### Preconditions for ordinary insertion

A conventional successful insertion requires:

- A valid control record.
- A contiguous, accessible storage region with four-byte element spacing.
- Nonnegative, four-byte-aligned differences between Begin, End, and Limit.
- Sufficient writable destination storage for the selected path.
- A readable value address when its delayed read occurs.
- No interfering concurrent mutation.
- Separation of the control record from element writes if the ordinary metadata interpretation is to remain valid.

For the spare-capacity path, the sequence must also be nonempty, and the insertion address must identify an existing element from Begin through old End minus four, inclusive.

For the replacement-storage path, the dependency contracts must support the intended operation. Merely returning a storage address or returning normally is insufficient to establish preservation of the original elements, the value source, or the replacement elements.

None of these conditions is explicitly checked.

## Behavior

### Initial inspection and branch selection

The function first captures End and calculates the initial unsigned element count. It then calculates the unsigned recorded capacity.

- Capacity greater than the element count selects the spare-capacity path.
- Capacity less than or equal to the element count selects the replacement-storage path.

A record whose computed length exceeds its computed capacity therefore follows the replacement-storage path; there is no distinct malformed-record error.

Only the initial End and calculated quantities are retained at this point. Later reads of control-record fields on the replacement-storage path are not entry-time snapshots.

### Spare-capacity path

This path uses the End captured during initial inspection.

The observable operations occur in this order:

1. Store old End plus four in the control record’s End field.
2. If old End is nonzero, read the four-byte value at old End minus four and write it at old End.
3. Shift preceding elements backward through the source range, placing their values one element higher in memory.
4. Read four bytes from the value address and write them at the insertion address.

The backward traversal initially regards old End minus four as its source position. If that position is not the insertion address, both traversal positions move down by four before the next four-byte transfer. This repeats until the source position equals the insertion address.

For an ordinary valid interior insertion, the suffix beginning at the insertion address is consequently shifted right by one element. Transfers proceed from higher addresses toward lower addresses, so later source elements are moved before earlier ones overwrite their old positions.

No dependency is called on this path.

#### Metadata is published before element movement

End is increased before the first element transfer and before the value source is read. A failing access or interrupted execution can leave the record reporting an increased length while storage is only partly updated.

There is no rollback.

#### Last-element insertion

If the insertion address is old End minus four, the initial duplication moves the last element into the newly exposed slot. The backward loop performs no iterations. The final write replaces the previous last element.

#### Insertion at End and empty sequences

An insertion address equal to old End does not cause an immediate empty shift. The traversal starts below that address and proceeds downward.

Ordinary append behavior is therefore not established for this path. Invalid accesses or traversal outside the intended storage region can occur.

An empty sequence with spare capacity likewise has no valid existing insertion element. When old End is nonzero, the initial duplication attempts to read four bytes before it. If old End is zero, that particular transfer is skipped, but the backward traversal and final write remain; the null check is not an empty-sequence handling branch.

#### Delayed value read and aliasing

The value source is not captured at entry. It is read after all shifting.

If the value address points into a slot overwritten by the shift, the inserted value is the contents of that slot at the delayed read, not necessarily its entry-time contents.

If element writes overlap the control record, they can also alter its fields. Claims such as “Begin and Limit remain unchanged” therefore require the ordinary disjoint-layout precondition.

### Replacement-storage path

#### Growth calculation

Let \(n\) be the initial unsigned element count. Two unsigned 32-bit capacity candidates are calculated:

- \(n + 32\)
- \(n + \lfloor n/2 \rfloor + \lfloor n/8 \rfloor\)

The larger candidate is selected by unsigned comparison. The fractional terms are truncated separately.

Each unsigned addition may wrap modulo \(2^{32}\). Selection occurs after those additions, so overflow does not preserve a mathematical “at least 32 more elements” guarantee.

The selected capacity is shifted left by two to obtain the 32-bit byte request. High bits discarded by that shift are not checked.

For non-overflowing ordinary counts, the policy chooses the greater of 32 additional elements and approximately 1.625 times the old count.

#### Storage request

The function invokes **F-871e64ddc6de** with:

1. The calculated 32-bit byte request.
2. Zero.

The returned 32-bit value is retained as the replacement base.

There is no explicit failure test, retry, or early return based on that result. The function does not establish that this dependency preserves the control record, source elements, value source, or other accessible state.

#### First range-processing call

After the storage request returns, the function invokes **F-000000000001** with:

1. Begin, read from the control record for this call.
2. The insertion address.
3. The retained replacement base.
4. The control-record address.

The result is unused.

Because the dependency receives the control-record address, subsequent field reads must not be treated as necessarily retaining their earlier values. The storage-request dependency could also have changed state before this call.

#### Inserted-element destination and write

After the first range call returns, the function reads Begin again. It calculates a 32-bit difference between the insertion address and that Begin, clears the low two bits of the difference, and adds the resulting offset to the replacement base.

Clearing the low bits rounds that computed offset down to a multiple of four. It does not validate either address and does not alter the insertion address passed to the range dependency.

If the calculated destination is nonzero:

- Read four bytes from the value address.
- Write those four bytes to the calculated destination.

If the calculated destination is zero, both that value-source read and that destination write are skipped.

This is a destination-address check, not a replacement-base check. A zero replacement base can still yield a nonzero insertion destination.

The value read can observe changes made by either preceding dependency call.

#### Second range-processing call

The function invokes **F-000000000001** again with:

1. The insertion address.
2. End, read from the control record for this call.
3. The replacement base plus a newly calculated aligned insertion offset, plus four.
4. The control-record address.

The aligned offset is recalculated from Begin read for this call. It need not equal the offset used for the inserted-element write.

The result is unused.

Under an additional conventional copying contract, the two calls would transfer the prefix before the insertion address and the suffix beginning there, with one destination element between them. That interpretation is not a supplied dependency guarantee.

The representation does not establish a separate ordering among argument-expression field reads within one call.

#### Capture and empty traversal

After the second range call returns, the function captures the current Begin and End.

It then advances a traversal position from captured Begin in four-byte increments until it equals captured End. The loop body contains no explicit read, write, or dependency call.

For an ordinary finite aligned range, this traversal has no visible element-storage effect. In particular, it does not visibly invoke element destruction.

If the endpoint cannot be reached by the four-byte progression, normal completion is not established. Extreme traversal arithmetic also retains the uncertainty about signed overflow.

#### Release-like call

After the traversal, the function invokes **F-ff1bdc376dcd** with:

1. Captured Begin.
2. A count calculated by signed subtraction of captured Begin from the then-current Limit, followed by a signed right shift by two.
3. Zero.

The function does not inspect a result.

The first argument is captured; Limit is read later for this call. No ownership transfer, release action, or preservation guarantee can be inferred solely from the call shape.

#### Final control-record stores

After the release-like call returns, the function stores the fields in this order:

1. Begin becomes the retained replacement base.
2. End becomes the replacement base plus four times the signed, right-shifted captured End-minus-Begin difference, plus four.
3. Limit becomes the replacement base plus four times the capacity selected before the storage request.

Precisely, let \(D\) be the signed 32-bit difference between captured End and captured Begin, and let \(Q\) be the result of signed right-shifting \(D\) by two. The End calculation is:

\[
\text{replacement base} + 4Q + 4.
\]

It is **not generally** replacement base plus the unmodified captured byte difference plus four.

For nonnegative \(D\) without arithmetic overflow, the contribution from the captured range is \(4\lfloor D/4\rfloor\); low-order remainder bytes are discarded. For negative differences or overflowing signed operations, the exact result remains **documented_only**.

This distinction is retained even though an unaligned captured range also raises a separate termination problem for the preceding traversal. There is no guarantee that such a case reaches the final stores.

Changes made by dependencies to Begin or End before capture can affect the release address and the final End calculation. The chosen capacity remains based on the initial count. Changes made to the record by the release-like dependency are subsequently overwritten by the three final stores, assuming the record remains writable and execution continues normally.

## Side effects

Established direct effects include:

- Early publication of an increased End on the spare-capacity path.
- Individual four-byte reads and writes during backward shifting.
- A delayed value-source read, conditional on a nonzero destination on the replacement-storage path.
- One storage-request call, two range-processing calls, and one release-like call on the replacement-storage path.
- Final replacement of Begin, End, and Limit in that order.
- Potential partial modification if execution fails or a dependency does not return normally.

The function provides no explicit status, error result, bounds validation, overflow report, or rollback.

The call pattern suggests acquisition of replacement storage and release of old storage, but ownership is unresolved without dependency contracts. Neither preservation of globals nor preservation of memory reachable by dependencies is established.

## Dependencies

| Opaque ID | Established invocation | Unresolved behavior |
|---|---|---|
| **F-871e64ddc6de** | Byte request and zero; return value retained as replacement base. | Allocation, alignment, failure signaling, ownership, and preservation of existing state. |
| **F-000000000001** | Two range-like endpoints, a destination, and the control-record address; invoked twice; results unused. | Copying, moving, construction, destruction, record mutation, source preservation, and other side effects. |
| **F-ff1bdc376dcd** | Captured Begin, a signed shifted capacity-like count, and zero; called before final record stores. | Release semantics, expected count units, failure behavior, and preservation of replacement storage or other state. |

**F-000000000001** is a document-local opaque dependency ID. It does not assert an identity supplied by the dependency mapping.

## Acceptance examples

These are conditional static consequences, not executed tests. Ordinary examples assume valid, aligned, accessible storage, a disjoint control record, no arithmetic overflow, and no concurrent interference.

1. **Interior insertion with spare capacity.**  
   Starting with `[10, 20, 30]`, capacity for at least four elements, insertion at the second element, and an independent value source containing `99`, normal completion produces `[10, 99, 20, 30]`. End increases by four; Begin and Limit are unchanged.

2. **Insertion at the last existing element.**  
   With the same storage and value source, insertion at the third element produces `[10, 20, 99, 30]`. The backward loop performs no iterations after the initial last-element duplication.

3. **Value source overwritten by shifting.**  
   Starting with `[10, 20, 30]`, spare capacity, insertion at the first element, and a value address pointing to the second element, normal completion produces `[10, 10, 20, 30]`. The delayed read sees the former first element after it has been shifted into the second slot.

4. **Attempted append with spare capacity.**  
   For a nonempty sequence and an insertion address equal to old End, ordinary append is not an acceptable guaranteed outcome. The traversal starts below the insertion address and moves downward.

5. **Full three-element sequence with explicitly noninterfering dependencies.**  
   Length and capacity both equal to three select capacity 35 and byte request 140. A final-content expectation additionally requires all of the following:
   - The storage-request dependency returns a sufficiently large, aligned, writable region disjoint from the old storage, control record, and independent value source.
   - That dependency preserves the original record, `[10, 20, 30]`, and the value source containing `99`.
   - The range dependency performs the intended prefix and suffix copies, preserves the control-record fields, and does not otherwise disturb the old elements, value source, or replacement contents needed for the result.
   - The release-like dependency returns normally and leaves the replacement region and control record valid and writable; it does not alter the replacement elements.
   
   Under these assumptions, insertion at the second element yields `[10, 99, 20, 30]` in replacement storage, with End four elements after the replacement base and recorded capacity 35.

6. **Empty replacement case.**  
   A zero-length, zero-capacity record selects capacity 32 and requests 128 bytes. Successful insertion still requires suitable storage and valid, noninterfering handling of empty ranges by the dependencies.

7. **Zero computed insertion destination.**  
   If the replacement-path insertion destination evaluates to zero and the first range call returns, that particular value read and write are skipped. This does not itself skip the second range call, traversal, release-like call, or final stores.

8. **Captured-length rounding.**  
   For a captured nonnegative byte difference of 12, the final End arithmetic contributes 12 bytes from the captured range, then adds four. A difference of 13 contributes only 12 bytes in that calculation. The latter is not a normal-completion example: the preceding four-byte traversal may not reach its endpoint.

## Uncertainties

The following remain **documented_only**:

- Dependency semantics, ownership rules, failure signaling, and nonreturning behavior.
- Whether any dependency preserves the control record, original elements, value source, replacement elements, or globals.
- Signed-overflow behavior and right-shift semantics for negative signed values.
- Outcomes for inaccessible, malformed, misaligned, or unexpectedly overlapping addresses.
- Termination for captured endpoints not reachable by four-byte traversal.
- Caller guarantees excluding empty spare-capacity insertion or insertion at End.
- Endianness; it is unnecessary for describing the raw four-byte transfers.
- Any significance of the empty traversal beyond the supplied static representation.

The growable-array interpretation must not override the literal operation order, delayed reads, field reloads, conditional write, or final shifted-length calculation.

## Evidence

The supplied static representation establishes the 12-byte control-record layout, four-byte transfers, unsigned branch comparison, growth arithmetic, dependency-call order, delayed inserted-value read, backward traversal, empty old-range traversal, and ordered final metadata stores.

It specifically establishes that final End uses a signed right shift of the captured End-minus-Begin difference followed by multiplication by four, rather than directly adding the captured byte difference.

The supplied mapping identifies **F-871e64ddc6de** and **F-ff1bdc376dcd**. No dependency implementation or preservation contract is supplied.

No runtime tests have been performed. All acceptance examples are conditional deductions from static evidence.
