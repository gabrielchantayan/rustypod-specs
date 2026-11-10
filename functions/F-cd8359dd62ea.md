<!-- Trusted provenance: F-cd8359dd62ea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd8359dd62ea

## Purpose

F-cd8359dd62ea selects records through a linked traversal, compacts retained records in a contiguous array, repacks ordinary payloads, and reconstructs a contiguous doubly linked list.

Two dependency-produced unsigned 32-bit limits constrain ordinary records: a cumulative-weight limit and an accepted-record count limit. Records whose flag word contains mask `0x00000002` are exempt from both limits during linked selection. This contract calls those records **protected**; it calls other records **ordinary**. These terms do not establish application-level meaning.

Selection accepts ordinary records until the first rejection. After that rejection, all subsequent ordinary records are cleared, while protected records continue to be counted. Array compaction subsequently processes every slot whose flag word is nonzero, in array order.

Protected flag words are left unchanged by linked selection and suffix clearing. That guarantee does **not** extend to compacted destination slots: the function does not explicitly copy flag words during compaction.

The function exposes no explicit allocation, validation result, error return, rollback, or release operation.

## Interface

### Invocation and representation

The function receives one address identifying a mutable control object and has no explicit result value.

The supplied representation uses 32-bit scalar words and 32-bit addresses. All offsets below are byte offsets. They describe the observed representation, not a portable structure definition.

### Control object

| Offset | Width | Observed use |
|---|---:|---|
| `+0x00` | 32 bits | Payload base address |
| `+0x04` | 32 bits | Payload write position; reset and accumulated |
| `+0x08` | 32 bits | Input to the first limit-acquisition sequence |
| `+0x0c` | 32 bits | Contiguous record-array base |
| `+0x10` | 32 bits | Initial linked head; later replaced |
| `+0x14` | 32 bits | Linked tail; later replaced |
| `+0x18` | 32 bits | Array scan count initially; selected count afterward |
| `+0x1c` | 32 bits | First operand of the second limit input |
| `+0x24` | 32 bits | Subtracted operand of the second limit input |
| `+0x28` | 32 bits | Shared argument captured for both limit sequences |

No use is established for unspecified intervening fields.

### Records

Array records have a stride of `0xcc`, or 204 bytes.

| Offset | Width | Observed use |
|---|---:|---|
| `+0x00` | 32 bits | Previous-record address |
| `+0x04` | 32 bits | Next-record address |
| `+0x08` | Unknown extent | Beginning of a subobject passed to a dependency |
| `+0xbc` | 32 bits | Payload offset |
| `+0xc0` | 32 bits | Selection weight and payload length |
| `+0xc4` | 32 bits | Flag word |
| `+0xc8` | 8 bits | Additional metadata byte |

The extent of the subobject beginning at `+0x08` is unknown. Passing its address does not establish a whole-record copy or exclude writes to later record fields.

### Preconditions

Normal completion requires accessible control, record, and payload storage for all actual accesses, terminating linked traversal, valid dependency invocations, and arithmetic that produces usable addresses.

To interpret the result as precisely the selected collection, additional conditions are necessary:

- Linked traversal and array storage represent consistent membership.
- Dependencies do not invalidate storage or alter relevant state incompatibly.
- The protected-record counter does not overflow its signed 32-bit range.
- Unsigned count wrap does not invalidate the intended retained-count interpretation.
- The published count describes an accessible contiguous array prefix.

These conditions are not validated by the function. Overlap safety, synchronization, and ownership requirements imposed by dependencies remain unknown.

## Behavior

### 1. Acquire two limits

The first sequence occurs in this order:

1. Read control `+0x08` and call F-00b046cf541b with that 32-bit value.
2. After that call returns, read and capture control `+0x28`.
3. Call F-fdb31af1de99 with the intermediate result and the captured value.
4. Call F-eb34c4d75211 and retain its unsigned 32-bit result as the weight limit.

The second sequence follows:

1. Read control `+0x1c` and `+0x24`, and form their difference.
2. Call F-00b046cf541b with that difference.
3. Call F-fdb31af1de99 with the new intermediate result and the previously captured `+0x28` value.
4. Call F-eb34c4d75211 and retain its unsigned 32-bit result as the ordinary-record count limit.

The subtraction operands are read after the first sequence. Their relative read order within the subtraction is not established. Control `+0x28` is not re-read for the second sequence.

The subtraction is represented as signed 32-bit arithmetic; safe mathematical subtraction requires avoiding signed overflow. No conversion, division, rounding, clamping, or exceptional-input semantics are established for the dependencies.

### 2. Select through linked traversal

Selection starts from the then-current head in control `+0x10`. Three counters start at zero:

- Cumulative ordinary weight: unsigned 32-bit.
- Accepted ordinary count: unsigned 32-bit.
- Encountered protected count: signed 32-bit.

For a protected record, increment the signed protected count. Leave its flag word unchanged and do not add its weight. Then follow its next address.

For an ordinary record:

- Add the record’s 32-bit weight to the unsigned cumulative weight.
- Reject the record if the resulting weight exceeds the weight limit, or if the accepted ordinary count is already at least the count limit.
- Otherwise, increment the unsigned accepted count and overwrite the entire flag word with exactly `1`.

The weight addition uses unsigned 32-bit arithmetic, including when the stored weight has its high bit set. Its result wraps modulo \(2^{32}\). Comparisons against both limits are unsigned.

Equality with the weight limit is allowed. The count comparison occurs before accepting the current record. A count limit of zero therefore rejects the first ordinary record, even if its weight is zero.

An ordinary record initially flagged zero can be accepted. Ordinary acceptance replaces all previous flag bits; it does not merely set one bit.

### 3. Clear the suffix after rejection

The first rejected ordinary record starts suffix clearing. The function never resumes ordinary selection.

For that record and every subsequent linked record:

- If flag bit 1 is clear, write zero to the entire flag word.
- Read the next-record address and advance traversal.
- If the record just processed was protected, increment the signed protected count.

Thus protected records are counted on either side of the rejection point. Their flag words are not directly changed in either linked phase.

The first rejected ordinary record’s weight has already been included in the unsigned cumulative sum. Later ordinary weights are not used for further fitting decisions.

The signed protected count is incremented using signed arithmetic. More than `0x7fffffff` protected encounters exceeds its representable positive range; a defined signed-wrap result is not established.

### 4. Compact array slots and repack payloads

After linked traversal, write zero to control `+0x04`.

Read the array base from control `+0x0c` to initialize both scan and destination addresses. Scan forward by 204 bytes per slot. Advance the destination address only when a slot is processed as retained.

The continuation comparison is an unsigned address comparison against an endpoint formed from the current control array base plus the current control count multiplied by 204. The endpoint is recomputed at subsequent comparisons, rather than captured as an immutable bound. A dependency that changes those control fields can therefore change the remaining scan range.

A zero flag word causes the slot to be skipped. A nonzero flag word causes processing at the current destination position. The ordinary/protected distinction is then determined from bit 1 of the source flag word.

#### Ordinary retained slot

The following operations occur in order:

1. Call F-opaque-01 with:
   - Destination: payload base plus the current payload write position.
   - Source: payload base plus the source slot’s payload offset.
   - Length: the source slot’s 32-bit length.
2. After the call returns, read the current control write position and store it in destination `+0xbc`.
3. Add the source slot’s then-read length to the current control write position and store the result in control `+0x04`.

The payload call occurs even for a zero length. The function provides no explicit temporary buffer or overlap check.

The length used after the call need not equal the length passed to it: the dependency’s effects on source records and control storage are unknown. Likewise, the offset stored after the call uses the then-current control value, not an established pre-call snapshot.

#### Protected retained slot

Copy the source slot’s payload offset to destination `+0xbc`.

Do not call F-opaque-01 for that slot and do not directly advance the payload write position for it. This does not establish that its payload is isolated from ordinary payload writes.

#### Common retained-slot processing

After the offset handling:

1. If source and destination addresses differ, call F-opaque-02 with destination `+0x08` followed by source `+0x08`.
2. Copy the source slot’s then-current 32-bit length to destination `+0xc0`.
3. Copy the source slot’s then-current byte at `+0xc8` to destination `+0xc8`.
4. Advance the destination address by 204 bytes.

The source length and metadata byte are read after F-opaque-02 returns. That dependency may also affect fields previously written, including the destination payload offset; preservation is not established.

Compaction processes retained sources in **array order**, not necessarily linked order.

There is no explicit destination flag-word copy. For a moved record, the destination flag may remain from its previous occupant or may be changed by F-opaque-02. Neither protected status nor exact flag preservation at the destination is guaranteed by the visible stores. Vacated trailing slots are not explicitly erased.

### 5. Publish the selected count

After the array scan, add the accepted ordinary count and the protected count, then store the result in control `+0x18`.

Because the accepted counter is unsigned, the final addition converts the signed protected count to unsigned 32-bit and uses unsigned arithmetic. This does not make the preceding signed protected increments safe from overflow.

The published count is derived from linked selection, not from the number of nonzero array slots processed. Agreement requires suitable representation consistency and dependency behavior. The function performs no comparison between these quantities.

### 6. Rebuild contiguous links

The published unsigned count determines link reconstruction.

**Zero records**

1. Write zero to the tail.
2. Write zero to the head.

No record links are explicitly cleared.

**One record**

1. Read the current array base.
2. Write that address to the tail, then to the head.
3. Clear that record’s next field.
4. Clear the previous field at the current array base.

**More than one record**

In order:

1. Set the head to the current array base.
2. Set the tail to the final slot of the count-sized contiguous prefix.
3. Clear the final slot’s next field.
4. Clear the first slot’s previous field.
5. Set the first slot’s next field to the second slot.
6. Set the final slot’s previous field to its predecessor.
7. For each interior slot in ascending index order, write its previous address, then its next address.

The resulting links describe 204-byte-spaced array slots, subject to valid arithmetic and stable relevant storage. Reconstruction does not inspect their flag words.

### 7. Invoke final dependencies

After reconstruction:

1. Call F-1194c6159740.
2. **After that call returns, read the indirect-call target from a global slot.**
3. Invoke the target read from that slot, designated F-opaque-03.

The target is not captured before F-1194c6159740. That dependency may replace the global slot, and the subsequent invocation uses the post-call target.

Neither final call has explicit arguments in the supplied representation. Their effects and return behavior are unknown. No further direct mutation is visible after the indirect invocation, but final dependencies need not preserve the reconstructed state.

## Side effects

Direct mutations include:

- Accepted ordinary flag words overwritten with `1`.
- Rejected and subsequent ordinary flag words overwritten with zero.
- Payload write-position reset and accumulation.
- Compacted payload offsets, lengths, and metadata bytes.
- Publication of the selection-derived count.
- Replacement of head and tail.
- Link reconstruction across the published contiguous prefix.

Dependencies may cause additional memory, global-state, or ownership effects. The final global target read occurs after F-1194c6159740 and is itself part of the observable dependency ordering.

No explicit allocation, deallocation, destructor call, ownership transfer, or rejected-record release is established. Logical rejection does not imply resource release. Calling F-opaque-02 does not establish copy, move, destruction, or ownership semantics.

Changes are incremental. There is no visible recovery branch, transaction boundary, or rollback. Invalid storage, arithmetic hazards, nonterminating chains, and dependency failures have no established error-handling contract.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-00b046cf541b | Called twice with one 32-bit input; supplies a 32-bit intermediate |
| F-fdb31af1de99 | Called twice with an intermediate and the same captured control value |
| F-eb34c4d75211 | Called after each preceding pair; supplies an unsigned 32-bit limit |
| F-opaque-01 | Receives payload destination, payload source, and a 32-bit length |
| F-opaque-02 | Receives destination and source subobject addresses when a record moves |
| F-1194c6159740 | Called after count publication and link reconstruction |
| F-opaque-03 | Target loaded from a global slot after F-1194c6159740 returns, then invoked |

The F-opaque labels identify dependencies without supplied mapped identities. F-opaque-03 designates the target obtained at invocation time, not a proven fixed function identity.

No dependency is assumed to preserve globals, control fields, record contents, payload contents, or ownership.

## Acceptance examples

These are conditional static deductions, not observed executions. Unless stated otherwise, assume accessible storage, terminating traversal, consistent membership, safe address arithmetic, no counter overflow, and dependencies that preserve relevant state except for their stated operation. Results describe direct reconstruction before unspecified final-call effects.

### Exact weight boundary

With ordinary weights 4 and 6, weight limit 10, and count limit 2:

- Both ordinary records are accepted.
- Their selection-phase flag words become `1`.
- The published count is 2.

Weight equality is allowed.

### Rejection ends ordinary selection

With ordinary weights 6, 5, and 1, weight limit 10, and count limit at least 3:

- The first record is accepted.
- The second is rejected when cumulative weight becomes 11.
- The third is cleared without an individual fitting decision.
- The published count is 1.

### Protected records on both sides of rejection

Consider array and linked order consisting of ordinary weight 4, protected, ordinary weight 7, protected. Let the weight limit be 10 and the ordinary count limit be **at least 1**.

- The first ordinary record is accepted.
- The second ordinary record is rejected.
- Both protected flag words remain unchanged during linked selection and suffix clearing.
- The published count is 3.
- Compaction processes the first, second, and fourth source records in that order.

The final protected source moves into the third array slot, whose ordinary flag was cleared. There is no explicit flag copy to that destination. Its resulting flag depends on F-opaque-02, even if that dependency leaves unrelated storage unchanged. If it does not write the destination flag, that flag remains zero while link reconstruction still includes the slot.

With count limit zero instead, the first ordinary record is rejected and the published count is 2.

### Unsigned cumulative wrap

With ordinary weights `0xffffffff` and 1, weight limit `0xffffffff`, and count limit 2:

- The first sum equals the limit.
- The second sum wraps to zero.
- Both records are accepted.

The criterion is not an unlimited-precision total-weight bound.

### Mixed payload handling

With retained sources ordered ordinary length 3, protected, ordinary length 2:

- Ordinary payload destinations begin at offsets 0 and 3.
- The protected source supplies its existing payload offset without directly advancing the write position.
- The final write position is 5.

This requires the dependencies to preserve the relevant offsets, lengths, and accumulator. It establishes neither overlap safety nor destination flag preservation.

### Final target replacement

If the global target initially identifies one callable, and F-1194c6159740 changes the slot to identify another callable before returning, the function subsequently reads and invokes the replacement target.

### Empty result

With no protected records and immediate ordinary rejection:

- The published count is zero.
- Tail and head are cleared, in that order.
- The payload write position remains zero if no scanned slot is nonzero.
- Trailing record storage is not generally erased.

## Uncertainties

### Arithmetic boundaries

Established unsigned 32-bit operations include cumulative ordinary weight, accepted ordinary count, their comparisons, and the final mixed-counter addition.

The protected count instead uses signed 32-bit increments. Signed overflow is a distinct hazard; modulo behavior is not established for those increments.

Other signed-represented expressions include the second limit-input subtraction, payload-position addition, and parts of endpoint/address construction. A portable wraparound contract is not established for those expressions. Inputs avoiding overflow and address wrap are the conservative precondition.

### Inferred intent

Selection, compaction, payload repacking, and link rebuilding suggest trimming a collection while exempting a distinguished record class. The units of the limits, meaning of metadata, and reason for exemption remain unknown.

### documented_only unknowns

- Limit-dependency semantics, rounding, and exceptional inputs.
- Payload-transfer overlap behavior.
- Metadata-transfer extent, destination flags, and ownership effects.
- Resource disposition of rejected or overwritten records.
- Dependency mutation of storage, control fields, and globals.
- Final indirect-target identity and termination behavior.
- Guaranteed relationship between linked and array order.
- Possible overlap between protected payloads and ordinary destinations.
- Outcome when scanned retention and published selection counts disagree.
- Synchronization, reentrancy, and concurrent-mutation guarantees.

## Evidence

This contract derives from static inspection of the supplied function representation and dependency-ID mapping. The prior draft, comments, and reviewer feedback are not independent behavioral evidence.

Field offsets, operand types, branch conditions, store order, call order, counter-type distinctions, absence of explicit flag copying, and the post-F-1194c6159740 global target read support the established behavior above. Dependency names do not establish their implementations.

No runtime tests have been performed. Acceptance examples are conditional deductions from the supplied static evidence.
