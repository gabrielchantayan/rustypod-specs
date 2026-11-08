<!-- Trusted provenance: F-51bee60da2e4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-51bee60da2e4

## Purpose

F-51bee60da2e4 performs an insertion-like update on a contiguous sequence of 16-byte records. It accepts a mutable storage descriptor, an insertion address, and an address supplying one record.

Two execution paths are established:

- When the descriptor’s recorded capacity exceeds a dependency-reported quantity, the function extends the logical end and moves records within existing storage.
- Otherwise, it computes a larger capacity, obtains replacement storage through a dependency, makes two transfer-related dependency calls, writes the supplied record into the replacement allocation, releases the previous storage through another dependency, and replaces the descriptor fields.

The in-place path directly establishes a rightward shift and a final record write, subject to valid addresses and termination conditions. The replacement-storage path has the apparent intent of preserving the records before and after the insertion position, but that preservation is **not established** by the supplied material: the transfer dependency’s behavior is unavailable, and its two calls expose different argument lists.

This contract distinguishes the observable operations from that inferred insertion intent. No runtime tests have been performed.

## Interface

### Inputs

The function receives three address-valued inputs in a 32-bit address model:

1. **Storage descriptor address.** Points to at least three consecutive 32-bit fields:
   
   | Byte offset | Field | Interpretation used by this function |
   |---|---|---|
   | 0 | Base | Beginning of the record allocation |
   | 4 | End | Address immediately after the logical records |
   | 8 | Limit | Address immediately after the available record storage |

   These interpretations follow from the function’s address arithmetic and updates, not from a supplied type declaration defining a container contract.

2. **Insertion address.** A destination position measured within the record storage. The function neither validates this address nor converts it into a checked record index.

3. **Source-record address.** Points to four readable 32-bit words, totaling 16 bytes. The words are copied as raw values. No field-specific interpretation, construction, or destruction is visible.

There is no explicit length input. The function obtains a 32-bit quantity from F-edaebf7a95e5 and uses descriptor fields independently of that quantity.

### Outputs

The function has no explicit return value or status result.

Results are exposed through:

- Changes to the descriptor.
- Writes to existing or replacement record storage.
- Calls to external dependencies.
- Possible allocation and release effects, whose precise ownership rules are not available.

### Preconditions and scope of the normal-behavior contract

For ordinary sequence behavior, the descriptor must reference accessible storage, address arithmetic must remain meaningful, and the record region must be compatible with a 16-byte stride.

The in-place path additionally requires:

- At least one existing record.
- Space for another 16-byte record.
- An insertion address equal to an existing record boundary between the base and the last record, inclusive.

The visible in-place movement does **not** support an insertion address equal to the original end. Its backward traversal starts at the last existing record and stops only when it reaches the supplied address.

The replacement-storage path requires sufficient readable source storage and suitable dependency behavior. Neither successful allocation nor correct transfer of surrounding records is checked locally.

Aliasing between the source record and the descriptor, record storage, or dependency-managed storage is not prohibited by a visible check. Such aliasing can affect the value ultimately copied; stable snapshot semantics must not be assumed.

## Behavior

### Capacity decision

Before the first dependency call, the function reads and retains the descriptor’s base and limit.

It then invokes F-edaebf7a95e5 with the descriptor address. The replacement-storage path is selected when the following unsigned comparison is true:

- The arithmetic-right-shifted, 32-bit difference between the previously read limit and base, shifted by four bits and then interpreted as unsigned, is less than or equal to the returned quantity.

Otherwise, the in-place path is selected.

For an ordinary nonnegative, nonoverflowing byte span divisible by 16, the left side is the capacity in records. Negative, overflowing, or malformed spans do not have a portable high-level container interpretation.

The initial base and limit reads occur before F-edaebf7a95e5 executes. If that dependency changes the descriptor, the branch decision still uses the earlier base and limit together with the dependency’s returned value. Later operations use additional descriptor reads.

### In-place path

Let the end read on entering this path be the **original end for this path**.

The function first writes the descriptor’s end field to 16 bytes beyond that original end. This descriptor update precedes every record copy on this path.

If the original end is nonzero, it copies the preceding 16-byte record into the 16-byte slot beginning at the original end. This duplicates the previous last record into the new final slot.

It then traverses backward in 16-byte steps. Starting with the previous last-record address as the stopping-position candidate, it repeatedly copies the record immediately before that candidate one slot to the right. Traversal stops when the candidate equals the insertion address.

Thus, for a valid insertion position, records originally at and after that position are shifted right by one record. If the insertion address is the previous last-record address, the initial duplication supplies the entire required shift, and the backward loop performs no iterations.

Finally, the function copies four 32-bit words from the source-record address into the insertion address.

The final source-record copy occurs **after** all shifting. Therefore, when the source aliases an affected record, the inserted value is taken from the source address’s post-shift contents, not necessarily its entry contents.

#### Word access order

The initial last-record duplication interleaves each source-word read with its corresponding destination-word write, proceeding from the lowest word to the highest word.

For each backward-shift iteration, the upper three source words are read first. The lowest source word is then read and written to the destination’s lowest word, followed by writes of the three retained upper words.

The final insertion copy likewise reads source words at byte offsets 4, 8, and 12 before reading and writing the word at offset 0. It then writes the retained words at offsets 4, 8, and 12.

These orders matter for overlapping storage. They do not establish atomic 16-byte accesses.

### Replacement-storage path

#### Capacity calculation and allocation request

The function calls F-edaebf7a95e5 a second time. This second result, rather than the first result, drives the capacity calculation.

It computes two unsigned 32-bit candidates:

- The second result plus 32 records.
- The second result plus its logical-right-shifted halves and eighths: the result plus the result shifted right by one bit plus the result shifted right by three bits.

It selects the larger candidate using an unsigned comparison. Additions are 32-bit operations; overflow is not checked.

The allocation-related dependency F-871e64ddc6de is called with:

- The selected capacity shifted left by four bits, as a 32-bit byte-count expression.
- A second argument of zero.

The returned 32-bit value is used as the replacement base. There is no general failure check before subsequent calls and arithmetic.

For nonoverflowing quantities, the capacity rule is the larger of an additional 32 records and approximately 1.625 times the reported quantity. This growth interpretation is inferred intent; the actual arithmetic remains subject to 32-bit wraparound.

#### First transfer-related call and inserted-record write

After the allocation call, the function invokes F-fb56d3ad25d5 with two visible arguments:

1. The descriptor’s base as read at that point.
2. The insertion address.

No explicit replacement destination is visible in this call. Its transfer result, if any, cannot be specified without the dependency contract.

After that call returns, the function reads the descriptor’s base again and computes a byte displacement:

- Subtract that base from the insertion address using 32-bit address arithmetic.
- Clear the low four bits of the result.

It adds this displacement to the retained replacement base to obtain the destination of the inserted record.

Clearing the low four bits rounds the displacement down to a multiple of 16 in the unsigned bit representation. The function does not reject a misaligned insertion address.

If the computed insertion destination is nonzero, the function copies the source record there. The word-read and word-write order matches the final insertion copy described for the in-place path.

The condition guards the **computed insertion destination**, not the allocation result. A zero allocation result with a nonzero displacement does not suppress the write. A computed zero destination suppresses this one copy but does not stop subsequent processing.

#### Second transfer-related call

The function calls F-fb56d3ad25d5 again, this time with four visible arguments:

1. The insertion address.
2. The descriptor’s current end.
3. The replacement base plus a newly evaluated, low-four-bits-cleared displacement from the descriptor’s current base to the insertion address, plus 16 bytes.
4. The descriptor address.

The displacement is evaluated again rather than guaranteed to reuse the first destination calculation. Aliasing or intervening dependency effects can therefore affect the arguments.

The supplied evidence does not establish whether this call copies, moves, constructs, updates state, or performs another operation.

#### Old-storage traversal, release, and descriptor replacement

After the second transfer-related call returns, the function:

1. Calls F-edaebf7a95e5 again and retains its returned 32-bit result for the final end calculation.
2. Reads the current descriptor base.
3. Advances a traversal address from that base by 16 bytes until it equals the descriptor’s end.

The traversal has no visible record reads, record writes, destructor calls, or other body effects. It is not evidence that records are destroyed. Termination requires that repeated 16-byte advancement can reach the observed end; otherwise, the visible logic does not establish a terminating execution.

The function then invokes F-ff1bdc376dcd with:

- The retained old base.
- The arithmetic-right-shifted difference between the current limit and that old base, shifted by four bits.
- Zero.

Finally, after that dependency returns, it writes the descriptor fields in this order:

1. Base becomes the retained replacement base.
2. End becomes the replacement base plus 16 times the last F-edaebf7a95e5 result, plus 16.
3. Limit becomes the replacement base plus 16 times the selected new capacity.

The final end is therefore based on the **third** quantity returned on this path, not necessarily the quantity used to select capacity. The descriptor replacement is not atomic.

## Side effects

### Direct effects

On the in-place path:

- The end field is advanced before copying begins.
- Existing record slots are overwritten during the right shift.
- The insertion slot receives a raw four-word copy.
- Base and limit are not directly rewritten.

On the replacement-storage path:

- The function requests replacement storage.
- It conditionally writes the inserted record into a computed replacement address.
- It invokes transfer-related operations and a release-related operation.
- It replaces all three descriptor fields after the release-related call.

### Ownership and failures

The names “allocation-related” and “release-related” describe inferred roles, not verified ownership contracts.

It is unknown whether the dependencies:

- Return owned, borrowed, shared, or specially managed storage.
- Release the provided allocation.
- Preserve globals or descriptor fields.
- Fail by returning a sentinel, terminating, unwinding, or some other mechanism.

There is no visible rollback, explicit error return, allocation-failure branch, or bounds validation. If an operation does not return normally, earlier writes may already have occurred. In particular, the in-place end update precedes potentially invalid memory accesses.

No synchronization or concurrency guarantee is established.

## Dependencies

| Opaque function ID | Visible use | Established limits |
|---|---|---|
| F-edaebf7a95e5 | Receives the descriptor; supplies a 32-bit quantity for branching, growth, or final end calculation | One call on the in-place path; three on the replacement path. Purity and meaning of the quantity are unknown. |
| F-871e64ddc6de | Receives the computed 32-bit byte-count expression and zero; returns the replacement base value | Allocation semantics, alignment, failure behavior, and ownership are unknown. |
| F-fb56d3ad25d5 | Called twice on the replacement path, first with two visible arguments and then with four | The differing visible call shapes prevent a complete transfer contract from being established. |
| F-ff1bdc376dcd | Receives the retained old base, a capacity-like difference shifted by four, and zero | Release behavior and effects on other state are unknown. |

These calls occur in the order described above. No assumption that a dependency preserves globals, preserves the descriptor, or manages memory in a particular way is part of this contract.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Interior insertion with spare capacity

Assume the descriptor represents records A, B, and C with room for at least one more record; F-edaebf7a95e5 returns 3 without modifying relevant state; and an external, nonoverlapping source contains X.

With the insertion address at B:

- The end advances by 16 bytes before record writes.
- C is duplicated into the new final slot.
- B is copied into C’s former slot.
- X is copied into B’s former slot.
- The resulting logical record sequence is A, X, B, C.

### Insertion before the last record

Under the same assumptions, with the insertion address at C, the backward loop is skipped. The initial duplication and final source copy produce A, B, X, C.

### Source affected by shifting

Assume records A, B, and C, spare capacity, insertion at A, and the source address initially at B.

After shifting, B’s original address contains A. The final insertion reads that modified source address. The resulting records are A, A, B, C, rather than B, A, B, C.

This demonstrates why entry-time source snapshot semantics are not guaranteed.

### Replacement capacity without overflow

Assume both initial quantity calls return 8, the recorded capacity is 8, and dependencies leave relevant state unchanged.

The growth candidates are 40 and 13 records. The allocation-related request is therefore 640 bytes with a second argument of zero.

If the final quantity call also returns 8 and all dependencies return normally, the committed descriptor end is replacement base plus 144 bytes, and the limit is replacement base plus 640 bytes. Preservation of surrounding record contents remains conditional on F-fb56d3ad25d5.

### End-position insertion on the in-place path

If the insertion address equals the original end, the backward traversal moves away from that address rather than reaching it through the ordinary occupied range. This case is outside the supported normal-behavior preconditions; successful append behavior must not be accepted from this evidence.

## Uncertainties

The following are **documented_only**:

- The semantic meaning and side effects of every dependency.
- The complete calling contract of F-fb56d3ad25d5.
- Correct preservation of prefix and suffix records during replacement.
- Allocation failure signaling and ownership transfer.
- Validity requirements beyond those inferable from the visible memory accesses.
- Defined behavior for signed arithmetic overflow, invalid addresses, or nonterminating traversal.
- Whether callers guarantee nonaliasing, nonempty input on the in-place path, and insertion at an existing record boundary.
- Whether any external mechanism supplies error recovery or synchronization.

Inferred intent is limited to a growable, contiguous, 16-byte-record insertion operation. It does not override the documented read/write order or fill gaps in dependency behavior.

## Evidence

Established static evidence consists solely of the supplied function body and dependency-ID mapping.

That evidence supports the descriptor layout, 16-byte stride, 32-bit word copies, branch comparison, growth arithmetic, visible dependency arguments, mutation order, and absence of local validation or rollback.

It does not establish dependency implementations, platform-specific fault behavior, runtime reachability of malformed cases, or successful end-to-end replacement insertion.

No runtime execution, smoke run, or tests have been performed.
