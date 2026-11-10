<!-- Trusted provenance: F-5dca6bece01b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5dca6bece01b

## Purpose

F-5dca6bece01b filters an existing collection and, unless an auxiliary reference becomes nonzero during filtering, constructs an auxiliary object and converts its records into new entries.

The established sequence is:

1. Invoke an operation on any previously referenced auxiliary object, then clear the stored reference.
2. Examine existing entries and request removal according to a byte flag and conditional predicates.
3. If the auxiliary reference remains zero, prepare range-like data, construct and register an auxiliary object, and obtain a status.
4. Unless that status is 2, prepare shared data and create entries from a captured number of auxiliary records.
5. Return either literal zero for status 2 or the current auxiliary-reference field.

A time-related refresh is plausible from the helper sequence and literal 90, but its domain meaning is inferred. The evidence does not establish time units, record semantics, ownership, or the meaning of status 2.

## Interface

### Input and representation

The function accepts one pointer to mutable object storage and returns a 32-bit integer. The supplied representation uses 32-bit addresses and reference fields. All offsets below are byte offsets; these layouts are not a contract for a 64-bit representation.

The input object exposes these locations:

| Offset | Width | Observed role |
|---|---:|---|
| +0x00 | 32 bits | Virtual-operation table reference |
| +0x4C | Not established | Collection descriptor passed by address |
| +0x50 | 32 bits | Existing-entry count, compared as signed |
| +0x6C | 32 bits | Mutable auxiliary-object reference |
| +0xC8 | 32 bits | Argument to a formatting-like dependency |

Operations selected from the input object's virtual-operation table are:

- **+0xAC:** receives the input object and an existing entry selected for removal.
- **+0xA4:** receives the input object and a newly constructed entry.

Existing entries must have a readable byte at +0x2C.

An auxiliary object exposes a virtual-operation table reference at +0x00 and a 32-bit count at +0x04. That count is interpreted as unsigned. Its table supplies an initial operation at +0x04 and indexed access at +0x40.

Indexed access returns a pointer to a slot. The slot's first 32-bit word supplies the record reference. Each referenced record must expose a readable 32-bit field at +0x04.

New-entry processing directly accesses:

| Offset | Width | Observed role |
|---|---:|---|
| +0x08 | 32 bits | First conversion result |
| +0x0C | 32 bits | First conversion result, later overwritten |
| +0x10 | 32 bits | Zero-extended descriptor byte, later overwritten |
| +0x20 | Not established | Address passed to a combining dependency |
| +0x2C | 8 bits | Explicitly cleared |
| +0x30 | 32 bits | Cached source-record field value |

These accesses do not establish complete object layouts or disjointness between records, entries, and other storage.

### Return value

There are two normal return forms:

- Literal zero when the status dependency returns 2.
- Otherwise, the 32-bit value read from input +0x6C at function exit.

The latter is returned through an integer interface; its bits are not independently classified as success, failure, or a valid reference.

Status 2 is checked only after storing an auxiliary reference, performing registration-related calls, and reading the auxiliary count. Returning zero on this path does not imply that input +0x6C is zero.

### Preconditions

No explicit checks establish input validity, allocation success, slot validity, object lifetime, or callback safety.

Normal completion requires every accessed location and invoked operation to remain usable when accessed. This includes cached record and entry references used after dependency calls. Dependencies are not assumed to preserve global state, object fields, or previously supplied storage.

## Behavior

### 1. Operate on the previous auxiliary object and clear the field

The function reads input +0x6C. If nonzero, it invokes the operation selected at byte offset +0x04 of that object's virtual-operation table.

The supplied representation does not establish the complete calling convention or an explicit receiver argument for this operation.

After the optional call returns, the function writes zero to input +0x6C. This clear also occurs when the original reference was zero. A replacement reference written into that field by the initial operation would be overwritten.

Disposal or release is an inferred purpose, not an established effect. The visible behavior is an optional indirect call followed by a field clear.

### 2. Filter existing entries

Filtering begins at signed index zero. Each iteration is admitted only while that index is strictly less than the current signed 32-bit value at input +0x50. The count is reread for each loop-condition evaluation.

Consequently:

- An initial count of zero or a negative value admits no iteration.
- Count changes can alter subsequent iteration admission.
- The index is not advanced after a removal request.

For each admitted index, F-82c63e47892d receives the address at input +0x4C and the index. Its returned slot is dereferenced once to obtain the entry reference.

Before inspecting the entry's flag, the function:

1. Passes a four-byte temporary region to F-6132bda8e448.
2. Calls F-ebc906d2c2d5 with the entry reference.
3. Passes that result and a 12-byte temporary region to F-dee541b2a059.

It then reads the byte at entry +0x2C. Removal is requested when either:

- The byte is zero; or
- The byte is nonzero, F-3cd57bab8f84 returns a negative signed result for the 12-byte and four-byte temporaries, and F-2b9d47918562 returns zero for the entry.

Evaluation is short-circuited. A zero flag bypasses both predicate calls. A nonnegative comparison result bypasses F-2b9d47918562. The three preparatory calls occur even for a zero flag.

For a selected entry, the input object's virtual operation at +0xAC receives the input object and entry. The filtering index remains unchanged. Otherwise, the index increases by one.

This accommodates removal that shifts a successor into the same slot, but the callback's actual collection behavior is unknown. If a removable entry remains at the same index and the count continues to admit that index, removal can be requested indefinitely.

The increment cannot overflow a signed 32-bit index. It occurs only when the nonnegative index is strictly below a signed 32-bit count, so its result is at most 2,147,483,647. At that value, no representable signed count admits another iteration. Count mutation can affect termination, but cannot enable overflow through this increment.

After filtering, the function rereads input +0x6C. A nonzero value skips all construction, status, and population work and leads to the final field read and return.

### 3. Prepare range-like storage and construct the auxiliary object

If input +0x6C is zero, F-7551853739b9 receives eight-byte storage and two zero arguments. F-7866cd84845e then receives the same storage.

The function copies its two 32-bit halves into a second eight-byte region. F-2f4a6c4b80c2 receives this copy, literal 90, and zero. F-69fade0587d8 receives a 16-byte output region and pointers to the original and copied eight-byte values.

The established contract is storage width, copying, argument values, and call order. No time arithmetic, unit, calendar rule, or normalization behavior is established.

The function next performs this sequence:

1. Call F-8157a180d901 with 16.
2. Call F-e194f3fbdfaa.
3. Store its 32-bit result at input +0x6C.
4. Call F-13832446aa57.
5. Call F-7f3dda340e54 with that result and the then-current input +0x6C value.
6. Call F-23e8d37d5f69.
7. Call F-00cae26c22a2 and retain its status.
8. Read and capture the unsigned 32-bit count at +0x04 of the auxiliary object then referenced by input +0x6C.

The argument 16 is size-like, but allocation semantics and coupling between the first two calls are not established.

The count read precedes the status comparison. Even status 2 therefore requires the count access to be valid. If the status is 2, the function returns zero immediately, without a local clear or rollback of input +0x6C.

### 4. Prepare shared data

For any other status:

1. F-398e0e9a318a receives an eight-byte shared temporary and a fixed static datum.
2. F-78db0cbe9071 receives another eight-byte temporary, the current input +0xC8 value, and the retained status.
3. F-46f0b3cb316e receives the shared temporary followed by the second temporary.
4. F-289ec4311c64 receives the second temporary.

Construction, formatting, combination, and cleanup are plausible interpretations. Exact contents, encoding, resource effects, and mutation contracts are unknown.

### 5. Convert auxiliary records into new entries

Population uses an unsigned 32-bit index beginning at zero and the count captured before the status check. The count is not reread.

For each index below that captured count, the function obtains the current auxiliary reference from input +0x6C and invokes its virtual operation at +0x40 with the auxiliary reference and index. The returned slot is dereferenced to cache the record reference for this iteration.

Replacing input +0x6C can therefore affect later indexed accesses, but does not replace the captured count. Cached record references remain subject to lifetime requirements.

#### Initial conversion and ordered initialization

For each record, the function:

- Passes eight-byte storage and two zeros to F-7551853739b9.
- Reads the record's 32-bit field at +0x04 and passes its value, the 16-byte range-like region, and that eight-byte storage to F-eecc5603eeba.
- Passes a four-byte descriptor region and the eight-byte value to F-f69722d7598a.
- Calls F-8157a180d901 with 56, then F-08291e7ca7a5 to obtain the entry reference.

**After F-08291e7ca7a5 returns, the record field at +0x04 is read again and cached before the entry flag is cleared.** The subsequent direct events are:

| Order | Access |
|---:|---|
| 1 | Read and cache the record's 32-bit field at +0x04 |
| 2 | Write zero to the byte at entry +0x2C |
| 3 | Write the cached 32-bit value to entry +0x30 |
| 4 | Call F-b8274406a146 with the first descriptor and literal 1 |
| 5 | Write that conversion result to entry +0x0C |
| 6 | Write the same result to entry +0x08 |
| 7 | Read the first descriptor's final byte and write its zero-extended value to entry +0x10 |
| 8 | Invoke the input object's virtual operation at +0xA4 with the input object and entry |

The read-before-clear relationship is significant: disjointness is not established. If the entry flag write overlaps the record field, the value written to entry +0x30 still comes from the preceding cached read.

The descriptor byte is read after F-b8274406a146 returns. Its value is not established as unchanged from the earlier descriptor-producing call.

The add operation occurs before the remaining per-entry work.

#### Post-add processing

After the add operation returns:

1. The record field at +0x04 is read again and passed with eight-byte output storage to F-1ae970f9f82f.
2. Both 32-bit halves of that output are copied into another eight-byte region.
3. F-f69722d7598a receives a second four-byte descriptor region and the copied value.
4. F-b8274406a146 receives this descriptor and literal 1.
5. Its result is written to entry +0x0C.
6. The second descriptor's final byte is read, zero-extended, and written to entry +0x10.
7. F-46f0b3cb316e receives entry +0x20 and the shared temporary.

These are **direct access and write guarantees**, not unconditional final-state guarantees.

There is no later direct assignment to entry +0x08 in this function. Nevertheless, absence of add-callback mutation alone does not prove that +0x08 retains the first conversion. Later dependencies can have unknown effects. Likewise, the second write to +0x0C does not prove that value persists through F-46f0b3cb316e, later iterations, or final shared-data processing.

Final-value claims require preservation by every intervening dependency and other relevant write, together with any necessary non-overlap assumptions.

After the population loop, F-289ec4311c64 receives the shared temporary. The function then reads and returns input +0x6C.

## Side effects

Established effects include:

- Clearing input +0x6C and conditionally storing a new reference there.
- Requesting removal through the input object's virtual operation.
- Performing construction-, registration-, and status-related calls.
- Passing 16 once and 56 per converted record to F-8157a180d901.
- Issuing the ordered new-entry writes described above.
- Calling the add operation before post-add conversion and combination.
- Passing stack-backed temporary storage to dependencies.

There is no visible local rollback, explicit null-result handling, or explicit failure propagation beyond the status-2 return. No local exception-handling behavior is established.

The function does not establish ownership transfer or release of auxiliary objects, records, entries, or temporary-managed resources. Nor does it establish that dependencies refrain from retaining temporary addresses. Safe retention and lifetime rules are documented_only.

## Dependencies

Only call-site roles are established; names below are opaque identifiers.

| Dependency | Observed role |
|---|---|
| F-82c63e47892d | Obtain an existing-entry slot |
| F-6132bda8e448 | Process four-byte filtering storage |
| F-ebc906d2c2d5 | Derive an argument from an entry |
| F-dee541b2a059 | Process a 12-byte comparison region |
| F-3cd57bab8f84 | Supply a signed comparison result |
| F-2b9d47918562 | Supply the secondary removal predicate |
| F-7551853739b9 | Process eight-byte storage with two zeros |
| F-7866cd84845e | Process the initial eight-byte value |
| F-2f4a6c4b80c2 | Process its copy with 90 and zero |
| F-69fade0587d8 | Process the 16-byte range-like region |
| F-8157a180d901 | Receive size-like arguments 16 or 56 |
| F-e194f3fbdfaa | Return the reference stored at input +0x6C |
| F-13832446aa57 | Supply a registration-related argument |
| F-7f3dda340e54 | Receive that argument and the current auxiliary reference |
| F-23e8d37d5f69 | Run before status retrieval |
| F-00cae26c22a2 | Return the status |
| F-398e0e9a318a | Process shared storage and fixed static data |
| F-78db0cbe9071 | Process input +0xC8 and status |
| F-46f0b3cb316e | Receive shared or entry-interior storage for combination-like work |
| F-289ec4311c64 | Process temporary storage at its final explicit local use |
| F-eecc5603eeba | Process a record value and range-like input |
| F-f69722d7598a | Process a four-byte descriptor and eight-byte input |
| F-08291e7ca7a5 | Return the new-entry reference |
| F-b8274406a146 | Convert a descriptor using literal 1 |
| F-1ae970f9f82f | Process a freshly read record value into eight-byte storage |

The four virtual-operation slots are additional dependencies. Their implementations and full contracts are unknown.

## Acceptance examples

These are conditional acceptance criteria, not executed tests.

1. **Zero flag:** For an admitted entry whose flag reads as zero, all three preparatory filtering calls occur, followed by the removal operation. Neither predicate dependency is called.

2. **Retained entry:** With a nonzero flag, negative comparison, and nonzero secondary result, removal is not requested and the index advances.

3. **No removal progress:** If removal leaves the same removable entry at the current index and the count continues to admit it, the function can repeatedly request removal.

4. **Filtering supplies an auxiliary reference:** If filtering ends with input +0x6C nonzero, range preparation, auxiliary construction, status retrieval, and population are skipped.

5. **Status 2:** The auxiliary count is read before zero is returned. There is no subsequent local clear of the stored reference or shared-data preparation.

6. **Zero auxiliary count:** For a non-2 status and captured count zero, shared-data preparation and final shared-temporary processing still occur; record indexing and entry construction do not.

7. **Aliased initialization:** If clearing entry +0x2C would change the record field at +0x04, entry +0x30 receives the value cached before that clear, not a reread afterward.

8. **Intermediate add state:** If storage is non-overlapping and intervening dependencies preserve the relevant locations, the add operation observes equal first-conversion values at entry +0x08 and +0x0C, with the flag cleared and +0x30 populated.

9. **Conditional completed state:** If all intervening dependencies and writes preserve the relevant fields, iteration completion leaves the first conversion at +0x08, the second at +0x0C, and the second descriptor byte zero-extended at +0x10. A preserving add callback alone is insufficient.

10. **Integer bounds:** A filtering index of 2,147,483,646 can advance to 2,147,483,647 under a maximum signed count, but cannot advance again. A captured unsigned population count with its high bit set remains a positive loop bound; even the maximum unsigned count is reached without an index wrap.

## Uncertainties

The following remain documented_only:

- Domain meanings, units, and arithmetic implemented by the range-like helpers.
- Static-data content, encoding, and formatting rules.
- Allocation, construction, failure, and exception contracts.
- Ownership, release, destruction, and temporary-retention rules.
- Actual collection mutation by removal and add operations.
- Disjointness of records, entries, descriptors, and other storage.
- Mutation and global-state preservation by every dependency.
- Reentrancy, synchronization, and callback-induced invalidation.
- Behavior after invalid references or unsuccessful allocation-like operations.

Neither loop's shown increment requires integer overflow. Filtering can still fail to terminate without index progress; population feasibility and reference validity remain separate requirements.

## Evidence

This contract derives solely from the supplied static function representation and opaque dependency mapping. Established claims concern visible access widths, signed and unsigned comparisons, cached values, direct writes, call ordering, branch conditions, and return expressions.

Time, text, allocation, registration, and cleanup meanings are inferred, not established dependency contracts. Persistent object state is not inferred merely from the function's direct assignments.

No runtime execution, smoke checks, or tests have been performed. Acceptance examples specify conditional behavior to verify; they are not observed results.
