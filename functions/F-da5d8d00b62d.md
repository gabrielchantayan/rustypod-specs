<!-- Trusted provenance: F-da5d8d00b62d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da5d8d00b62d

## Purpose

F-da5d8d00b62d searches a linked structure, then either publishes a candidate address or delegates an operation to F-0afbac3aaabc. Its result occupies five bytes: a four-byte result word followed by a one-byte disposition flag.

Two modes are established:

- A zero byte at container offset 0x18 enables additional candidate checks that can return without calling F-0afbac3aaabc.
- Any nonzero byte at that offset bypasses those checks and selects delegation after the initial search.

**Inferred intent:** the operation resembles insertion into an ordered tree, with the zero-control mode checking for an existing equivalent entry. Neither ordering semantics nor insertion success is established without dependency contracts. The branch conditions below define the behavior independently of that interpretation.

## Interface

### Arguments and result

The function receives three arguments:

1. **Result-storage address.** Must permit a four-byte write at offset 0 and a one-byte write at offset 4.
2. **Container address.** Identifies the representation described below.
3. **Incoming value word.** A four-byte value forwarded unchanged to dependencies. Its interpretation as an address, key, object, or handle is unknown.

No separate return value is established. On normal completion:

| Result offset | Width | Meaning |
|---|---:|---|
| 0 | 4 bytes | Candidate address or dependency-produced result word |
| 4 | 1 byte | Zero for the candidate-return path; one after a returning call to F-0afbac3aaabc |

The four-byte write precedes the one-byte write. These byte ranges are disjoint relative to the result-storage address. Aliasing result storage with container or node storage can modify those objects, but does not make the two output fields overlap.

### Accessed representation

| Object | Offset | Access |
|---|---:|---|
| Container | 0x10 | Header address |
| Container | 0x18 | One-byte control value |
| Container | 0x19 | Address supplied directly as the first comparator argument |
| Header | 0x04 | Initial search link |
| Header | 0x08 | Four-byte word copied for an additional dependency call |
| Candidate or node | 0x00 | One-byte special-case discriminator |
| Candidate or node | 0x04 | Link used during candidate adjustment |
| Candidate or node | 0x08 | Search or adjustment link |
| Candidate or node | 0x0C | Search or adjustment link |
| Candidate or node | 0x10 | Payload address supplied to the comparator |

The supplied representation uses four-byte words and four-byte address-bearing fields. This supports a 32-bit address interpretation; it does not establish portability to another address width, byte order, or alignment convention.

The argument at container offset 0x19 is the address of that location, not a pointer loaded from it. Likewise, a payload operand is the node address plus 0x10, not a payload word loaded there by this function.

### Preconditions

Normal completion requires:

- Readable container fields, header fields, and traversed links.
- Readable candidate bytes and links wherever the selected adjustment path dereferences them.
- Writable result storage for both output fields.
- Dependency calls that return normally and preserve whatever memory validity subsequent operations require.
- Terminating search and adjustment traversals.

There is no explicit null-input check, link validation, cycle detection, capacity check, or depth bound. Several adjustment dereferences have no preceding null check.

No general aliasing, concurrency, or reentrancy guarantee is established.

## Behavior

### 1. Initial search

The retained comparison state starts at integer one.

Initialization performs these accesses in order:

1. Read the header address from container offset 0x10.
2. Read the initial search link at that header’s offset 0x04.
3. Read the header address from container offset 0x10 again and retain this second value as the initial search candidate.

The two header reads are distinct. The function does not enforce their equality.

While the current search address is nonnull:

1. Clear two four-byte temporary words. These are later used, respectively, for immediate-delegation output and common-delegation output.
2. Call F-257a89ed2c43 with:
   - Container address plus 0x19.
   - Incoming value word.
   - Current search address plus 0x10.
3. Retain the visited search address as the search candidate.
4. Select and read the next link:
   - Comparator result zero: offset 0x0C.
   - Comparator result nonzero: offset 0x08.

Only zero versus nonzero matters. Negative and positive nonzero results select the same link.

The chosen link is read after the comparator returns. A comparator’s changes to that link can therefore affect traversal. Both links are not snapshotted before the call.

When traversal reaches null:

- The retained search candidate is the last visited address, or the second header address read during initialization if no node was visited.
- The comparison state is the last comparator result, or one if there was no search comparison.

The retained search candidate remains distinct from the addressable candidate slot used below.

### 2. Control selection

After the search, read the byte at container offset 0x18.

- **Nonzero:** proceed directly to common delegation.
- **Zero:** initialize an addressable candidate slot with the retained search candidate and perform the additional checks.

This is a post-search read. Mutations during search calls can affect the selected mode.

### 3. Additional dependency check

This step occurs only in zero-control mode when the retained comparison state is nonzero.

1. Reload the header address from container offset 0x10.
2. Read its four-byte word at offset 0x08.
3. Copy that word into temporary storage.
4. Call F-d747682ef415 with:
   - The address of the candidate slot.
   - The address of the copied word.

The copied-word storage is also the storage later used for common-delegation output. Both arguments point to writable local storage. Their values are not protected against dependency modification.

If F-d747682ef415 returns nonzero, perform immediate delegation:

1. Call F-0afbac3aaabc with:
   - Address of the immediate-delegation output word.
   - Container address.
   - Zero.
   - Retained search candidate.
   - Original incoming value word.
2. Copy the output word to result bytes 0–3.
3. Write one to result byte 4.
4. Finish.

The candidate argument is the retained search candidate, not a reread of the addressable candidate slot. Modification of that slot by F-d747682ef415 does not substitute a different insertion argument.

If F-d747682ef415 returns zero, adjust the candidate using the slot’s post-call value.

### 4. Candidate adjustment

Adjustment occurs only after the additional dependency check returns zero.

First read the byte at offset 0x00 of the candidate currently held in the slot.

If that byte is zero:

1. Read the candidate’s link at offset 0x04.
2. Read that linked object’s link at offset 0x04.
3. Compare the resulting address with the candidate.

If the byte was zero and those addresses match, replace the candidate slot with the candidate’s link at offset 0x0C. No other adjustment is performed.

Otherwise, inspect offset 0x08:

- Read the link at offset 0x08 into traversal storage.
- Read offset 0x08 again to choose between descent and ascent.

These are separate reads in the supplied body.

**Descent:** If the second offset-0x08 read is nonnull, begin with the address obtained by the first read. Assign it to the candidate slot, then follow offset-0x0C links until the next link is null. Each iteration reads offset 0x0C once to obtain the next address and again for the continuation test. The final slot value is the last assigned candidate.

**Ascent:** If the second offset-0x08 read is null:

1. Read the candidate’s offset-0x04 link as the first ancestor.
2. Compare the ancestor’s offset-0x08 link with the current candidate-slot value.
3. Assign the ancestor to the candidate slot.
4. If the comparison was equal, read that ancestor’s offset-0x04 link and repeat.
5. Stop at the first unequal comparison.

Thus, the ancestor becomes the candidate even on the iteration that stops ascent.

The special-case chain and ascent dereference links without checking them for null. Validity and termination depend on structural conditions not verified here. If repeated reads observe inconsistent link values, ordinary tree-traversal interpretations do not supply a defined safety guarantee.

**Inferred intent:** these operations resemble predecessor traversal with a sentinel special case. The explicit reads and assignments, not that interpretation, govern this contract.

### 5. Final candidate comparison

This step occurs in zero-control mode when either:

- The last search comparison was zero; or
- The additional dependency returned zero and adjustment completed.

Before the call, the function clears two four-byte temporary words, including the common-delegation output word.

It then calls F-257a89ed2c43 with:

1. Container address plus 0x19.
2. The candidate-slot value, as evaluated for this call, plus 0x10.
3. The incoming value word.

The operand roles are reversed relative to search comparisons.

Two candidate values must be distinguished:

- **Comparison candidate:** the slot value used to form the second comparator argument.
- **Published candidate:** the slot value read after the comparator returns zero.

If the comparator returns zero, the function rereads the addressable candidate slot, writes that value to result bytes 0–3, then writes zero to result byte 4. It does not call F-0afbac3aaabc.

On the adjustment path, F-d747682ef415 has already received the slot’s address. Without dependency contracts, the final comparator could modify the slot through a reference retained or otherwise made available by that earlier call. Consequently, the published candidate need not equal the comparison candidate.

If the comparator returns nonzero, proceed to common delegation.

### 6. Common delegation

Call F-0afbac3aaabc with:

1. Address of the common-delegation output word.
2. Container address.
3. Zero.
4. Retained search candidate.
5. Original incoming value word.

The fourth argument is not replaced with either the adjusted candidate or the candidate-slot value after the final comparator.

After the call returns:

1. Copy the temporary output word to result bytes 0–3.
2. Write one to result byte 4.

A flag of one establishes that F-0afbac3aaabc was called and returned normally. It does not independently establish allocation success, structural insertion, or ownership transfer.

## Side effects

### Direct effects and ordering

The established external writes are the four-byte result write followed by the one-byte flag write. No direct allocation, deallocation, structural-link update, or balancing operation is shown.

Relevant ordering includes:

- Search comparison before reading the selected next link.
- Search completion before the control-byte read.
- Header reload and word copy before F-d747682ef415.
- Dependency return before adjustment reads the candidate slot.
- Final comparator argument formation before the call.
- Final comparator return before rereading the candidate slot on the zero-result path.
- Delegation return before publishing its output.
- Result-word write before result-flag write.

The result update is not established as atomic.

If result storage aliases a container or node, publication can overwrite that object’s bytes. This does not cause result bytes 0–3 and byte 4 to overlap each other.

### Dependency effects and ownership

Dependency implementations are unavailable. They may mutate reachable objects, global state, or storage whose address they have received; preservation cannot be assumed.

No ownership transfer or release is directly established for the incoming value, returned candidate, or delegated result. No direct cleanup or rollback path is shown.

Addresses passed for temporary output, copied-word storage, and the candidate slot refer to call-local storage. Their validity does not extend beyond this invocation. Retention and use during the invocation are not ruled out; use after its lifetime would require guarantees absent here.

## Dependencies

| Opaque ID | Established use | Unresolved contract |
|---|---|---|
| F-257a89ed2c43 | Receives a container-relative address and two operands; its integer result is tested for zero/nonzero. | Ordering relation, operand representation, mutation, retained-reference access, errors |
| F-d747682ef415 | Receives addresses of the candidate slot and a copied four-byte header word; nonzero selects immediate delegation. | Argument mutation, reference retention, semantic meaning, errors |
| F-0afbac3aaabc | Receives output storage, container, zero, retained search candidate, and incoming value. | Output initialization, allocation, structural effects, success conditions, ownership, errors |

The two delegation paths use different output words. If F-0afbac3aaabc does not write its output:

- The immediate-delegation word was cleared during each search iteration, but has no established initialization when the search was empty.
- The common-delegation word was cleared during search iterations and again before the final comparison. It is not initialized on an empty-search, nonzero-control path.
- The common word can also have held the copied header word and been exposed to F-d747682ef415. A retained reference could permit later dependency modification.

A reliable result therefore requires an output contract from F-0afbac3aaabc. The surrounding function does not validate the produced word.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Nonzero control.**  
   Given a terminating search and a nonzero control byte when read, skip F-d747682ef415 and the final comparison. Delegate with the retained search candidate, publish the dependency output, then flag one.

2. **Direct candidate selection.**  
   Given zero control, a zero last search comparison, and a zero final comparator result, skip both the additional dependency and delegation. Publish the candidate slot’s post-comparator value, then flag zero. If the slot is preserved, this is the retained search candidate.

3. **Immediate delegation despite slot mutation.**  
   Given zero control, nonzero retained comparison state, and nonzero F-d747682ef415 result, delegate without adjustment or final comparison. Even if that dependency changes the candidate slot, pass the original retained search candidate.

4. **Adjusted candidate with preservation.**  
   Given zero from F-d747682ef415, valid terminating adjustment, preservation of the candidate slot during the final comparator, and a zero comparator result, publish the adjusted candidate with flag zero.

5. **Adjusted candidate changed during comparison.**  
   Under the preceding traversal conditions, suppose the comparator receives a payload address derived from candidate A, changes the slot to B through an available retained reference, and returns zero. Publish B with flag zero, not A. This scenario is conditional on dependency behavior; such mutation is not independently established.

6. **Final comparison selects delegation.**  
   After adjustment, a nonzero final comparator result selects common delegation. Pass the retained search candidate, regardless of the adjusted or post-comparator slot value.

7. **Negative search result.**  
   A negative comparator result selects offset 0x08, exactly as a positive nonzero result does.

8. **Empty search.**  
   If the initial search link is null, make no search comparison. Retain comparison state one and the separately reread header address. The subsequent control-byte read still determines whether to delegate directly or perform the additional check.

9. **Disjoint publication fields.**  
   On normal completion, write result bytes 0–3 before byte 4. Even if these bytes alias a node, the flag write does not overwrite any byte of the four-byte result field.

## Uncertainties

The following remain **documented_only**:

- Exact container type, ordering relation, sentinel rules, and balancing invariants.
- Semantic meaning of the control byte and candidate byte at offset 0x00.
- Full ABI, byte order, alignment, and address-conversion rules.
- Dependency preservation, reference retention, output guarantees, allocation, ownership, and failure behavior.
- Supported aliasing, concurrent mutation, and reentrant-callback behavior.
- Termination and safety for malformed or cyclic structures.
- Outcomes for invalid addresses or dependencies that fail to return normally.

There is no explicit count, capacity, or size arithmetic. Address formation uses fixed offsets; no wraparound check or portable out-of-range behavior is established.

## Evidence

The supplied static function body establishes the argument forwarding, field offsets, branch conditions, separate header reads, candidate-slot exposure, post-comparator slot reread, delegation arguments, and ordered output writes described here.

Tree insertion, equivalence checking, and predecessor traversal remain inferred intent. Dependency semantics are not established by this body.

No runtime tests have been performed. The acceptance examples describe conditional behavior for later verification, not observed execution.
