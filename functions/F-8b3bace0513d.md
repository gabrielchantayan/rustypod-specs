<!-- Trusted provenance: F-8b3bace0513d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8b3bace0513d

## Purpose

F-8b3bace0513d searches shared record storage for a requested identifier, rejects a selected record whose state field is nonzero, and attempts a staged activation sequence.

Complete success returns zero, publishes the selected record’s address through an output parameter, writes one to the record’s state field, and stores the requested identifier in a previously derived shared byte location. Some failures clear the state field and store a failure marker; other failures return without those completion writes.

“Activation,” “state,” and “timeout” are descriptive interpretations. The established contract consists of the accesses, dependency calls, branch conditions, and result propagation below. In particular, the shared table pointer is read repeatedly: this is not established to be an operation on one stable table.

## Interface

### Arguments and result

The function receives:

- **Requested identifier:** a 32-bit integer compared with a promoted one-byte record field.
- **Output location:** a pointer through which a four-byte representation of the selected record address is stored on complete success.

The supplied representation uses four-byte integers and address representations. Identifier-byte signedness and the complete target ABI are not independently established.

| Return value | Established condition |
|---|---|
| 0 | Final setup returned zero and success processing reached completion. |
| 5 | Lookup ended with the selected-address value equal to zero. Ordinarily this means no matching record was found. |
| 14 | The selected record’s four-byte state field was nonzero. |
| 91 | The initial operation returned zero, but classification returned something other than 3. |
| Other nonzero value | A nonzero result from the initial operation or final setup, returned unchanged. |

A dependency may itself return 5, 14, or 91. The numeric result therefore does not uniquely identify the path taken.

### Partial memory layouts

Each value read from the shared table pointer identifies a record-array base with the following relative layout:

| Offset | Width | Observed use |
|---|---:|---|
| −20 | 1 byte | Requested-identifier or failure-marker destination |
| −16 | 4 bytes | Unsigned record count |
| −12 | 4 bytes | First timeout-like field |
| −8 | 4 bytes | Second timeout-like field |
| 0 onward | 80 bytes per record | Record array |

Within a selected record:

| Offset | Width | Observed use |
|---|---:|---|
| +4 | 1 byte | Lookup identifier |
| +68 | 4 bytes | State field |
| +76 | 2 bytes | Preparation field, set to one |

Configuration objects reached through a separate shared pointer expose:

| Offset | Width | Observed use |
|---|---:|---|
| +164 | 2 bytes, unsigned | Bit 0 enables consideration of one optional call |
| +170 | 2 bytes, unsigned | Bit 0 suppresses that optional call; bit 4 suppresses another call |

These are accessed portions, not complete object definitions.

### Address identity and preconditions

The function does not validate its pointers, storage extents, alignment, or lifetimes.

Defined execution requires every access actually reached to address suitable storage. This includes objects retained across dependency calls and, on successful publication, a writable four-byte output location.

There are several distinct address sources:

- A first shared-table-pointer read supplies a retained base for timeout accesses.
- A second, separate read supplies the base from which the identifier destination is derived.
- Lookup independently reloads the shared table pointer for count accesses, identifier accesses, and matching-record address calculation.
- The selected record address is retained once calculated.
- Configuration accesses have their own reload and retention rules.

These addresses need not belong to the same objects without an additional stability assumption. Such stability is not enforced by this function.

An invalid output location is not rejected early. Failure paths omit the store through the output parameter; success can reach that store after substantial mutation and dependency activity.

## Behavior

### Initial address capture and lookup

Before scanning, the function performs two distinct shared-table-pointer reads, in this order:

1. It retains the first pointer value for subsequent timeout-field accesses at offsets −12 and −8.
2. It reads the pointer again and derives the one-byte identifier destination at offset −20. That derived destination is retained for completion.

Neither capture establishes the base subsequently used by lookup.

The scan starts at index zero and advances in increasing order. The index and count comparison are unsigned and four bytes wide. Each attempted iteration has these observable accesses:

- The loop condition reloads the shared table pointer and reads the count at offset −16.
- If the index is below that count, the identifier access reloads the shared table pointer and reads the byte at index × 80 + 4.
- If the identifier compares equal, calculating the selected record address reloads the shared table pointer yet again and uses index × 80.

Consequently, the count, compared byte, and selected address can come from different table bases. Even a successful byte comparison does not establish that the selected address denotes the record whose byte was compared.

With stable storage, the first matching index wins. Duplicate identifiers are not resolved by state or suitability. If the first match is ineligible, later matches are not examined.

The comparison promotes the stored byte before comparing it with the requested integer. It is not established as a low-eight-bit comparison. A requested value outside the byte’s promotion range cannot match merely because its low byte agrees.

After scanning, a zero selected-address value produces return value 5. This includes the ordinary no-match case and, if address arithmetic could produce it, a matching calculation whose result is zero. This return performs no direct stores and calls no dependencies.

### Eligibility and preparation

For a nonzero selected address, the function reads the four-byte state field at offset +68.

A nonzero value causes immediate return of 14, without direct stores or dependency calls. A zero value permits preparation:

1. Write the two-byte value one at selected record +76.
2. Call F-95caa07de452 with argument 2.
3. Multiply its integer result by 1000 and store the four-byte result at retained timeout base −12.
4. Store the same scaled expression at retained timeout base −8.
5. Call F-d0ab8e462b50 with argument 1; ignore its result.

The two timeout stores target the base retained by the first table-pointer read, not a freshly loaded table base. There is no explicit range check or saturation for multiplication.

### Initial operation

After F-d0ab8e462b50 returns, the function freshly reads the four-byte value at retained timeout base −12. It then calls F-7bfe7eab76c1 with:

1. The selected record address.
2. Zero.
3. Zero.
4. Zero.
5. That freshly read timeout value.

The fifth argument can differ from the earlier stored value if intervening effects changed that storage.

A nonzero dependency result selects unsuccessful completion:

1. Write four-byte zero at selected record +68.
2. Store the one-byte conversion of −1 at the retained identifier destination.
3. Return the dependency result unchanged.

No store through the output parameter occurs. Earlier preparation and timeout stores are not rolled back.

### Classification gate

If the initial operation returns zero, the function calls F-2706939fe3c5 with the selected record and a separate shared auxiliary value read for that call.

Only a classification result equal to 3 permits continuation. Every other result causes immediate return of 91.

This rejection bypasses common completion:

- No final state-field store occurs.
- No identifier-destination store occurs.
- No store through the output parameter occurs.
- Earlier direct stores remain unless overwritten by other effects.

Because dependencies may modify the record, this path does not guarantee that its state field remains zero.

### Provisional activation and final setup

After classification returns 3, the following order is established:

1. Call F-780925cf348e with the selected record and 1; ignore its result.
2. Read the shared configuration pointer for the setup argument.
3. Write four-byte one at selected record +68.
4. Call F-0bfad00ead24 with the selected record and that configuration pointer.
5. Read the shared configuration pointer again and retain this second value for a later bit-4 test.

The post-setup pointer read occurs even when setup returned nonzero.

A nonzero setup result selects unsuccessful completion: write state zero, store the byte conversion of −1 at the retained identifier destination, and return the setup result unchanged. No store through the output parameter occurs.

The provisional state value of one is visible when setup is called. Clearing it on failure does not undo other direct stores or dependency effects.

### Success-path configuration decisions

When setup returns zero, the first optional call is controlled by two separate current-pointer accesses:

1. Read the current shared configuration pointer and inspect the unsigned two-byte field at +164.
2. Only if its bit 0 is set, independently read the current shared configuration pointer again and inspect the unsigned two-byte field at +170.
3. Call F-f58de5c36c29 with the selected record only if the first bit was set and the second field’s bit 0 was clear.

Neither field access uses the pointer retained immediately after setup. The two fresh pointer reads are not established to identify the same configuration object. If the +164 bit is clear, the +170 access for this decision does not occur.

Next, call F-143c91667f91 with the selected record.

After that call, inspect the unsigned two-byte field at +170 through the configuration pointer retained immediately after setup:

- Bit 4 clear: call F-6c31d4a4862c with the selected record.
- Bit 4 set: skip that call.

This field read occurs after F-143c91667f91. Changes to the retained object can affect the decision. Merely replacing the shared configuration pointer does not redirect this access.

### Successful publication

After the flag-controlled calls:

1. Call F-c38e978ba2dc with the selected record.
2. Store the four-byte selected record address through the output parameter.
3. Write four-byte one at selected record +68.
4. Store the requested identifier, converted to one byte, at the retained identifier destination.
5. Return zero.

The results of success-path processing dependencies are ignored. Normal return from them does not introduce another status gate.

## Side effects

### Direct writes and persistence

For a complete successful execution, direct stores occur in this order:

1. Preparation field receives two-byte one.
2. First timeout field receives the scaled result.
3. Second timeout field receives the scaled result.
4. State field receives provisional four-byte one.
5. Output location receives the selected record address.
6. State field receives final four-byte one.
7. Retained identifier destination receives the converted requested identifier.

Initial-operation failure omits provisional activation and publication, then writes state zero and the failure marker. Setup failure includes provisional activation before those unsuccessful-completion writes. Classification rejection performs neither completion store.

Failures are not transactional. Preparation, timeout changes, and dependency effects can survive.

### Output aliasing

“No store through the output parameter” does not mean that the storage it designates is unchanged.

There is no overlap check. Other direct stores can modify an aliased output location, and dependencies can modify it independently. For example:

- On unsuccessful completion, an output location overlapping the state field can receive zero from the state-clear store.
- On success, an output location exactly equal to the state field first receives the record-address representation and then receives one.
- Overlap with the identifier destination can allow the final byte store to alter the published representation.

Similarly, overlapping table headers, records, pointer storage, or configuration can affect later reads. The access order remains significant even when ordinary layout assumptions do not hold.

### Ownership and concurrency

No explicit allocation, release, or ownership-transfer operation appears. The published address is calculated from shared record storage; it is not evidence of a newly allocated object.

Dependency contracts do not establish preservation of globals, continued object validity, or memory ownership. The function nevertheless continues using retained record, timeout-header, identifier-destination, and configuration addresses after calls.

No lock, atomic publication protocol, or stable snapshot is visible. The contract does not establish safe concurrent access or guarantee that distinct pointer reads observe a common value.

## Dependencies

| Opaque function ID | Invocation and result handling |
|---|---|
| F-95caa07de452 | Argument 2; integer result multiplied by 1000 for two four-byte stores. |
| F-d0ab8e462b50 | Argument 1 after timeout stores; result ignored. |
| F-7bfe7eab76c1 | Selected record, three zeros, freshly read timeout; nonzero result propagated after unsuccessful completion. |
| F-2706939fe3c5 | Selected record and shared auxiliary value; only result 3 continues. |
| F-780925cf348e | Selected record and 1; result ignored. |
| F-0bfad00ead24 | Selected record and configuration pointer read before provisional activation; zero enables success processing. |
| F-f58de5c36c29 | Selected record; conditional on the independently accessed bit-0 flags; result ignored. |
| F-143c91667f91 | Selected record; unconditional after successful setup and the first optional-call decision; result ignored. |
| F-6c31d4a4862c | Selected record; conditional on bit 4 through the retained configuration pointer; result ignored. |
| F-c38e978ba2dc | Selected record; last dependency before publication; result ignored. |

Internal effects, semantic error meanings, ownership policies, and normal-return guarantees are not established for these dependencies.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Except where stated otherwise, assume valid, non-overlapping storage, stable shared pointers, representable arithmetic, and normally returning dependencies that make no additional mutations.

1. **Empty table.** The first loop-condition count is zero. The result is 5, with no dependency calls and no direct stores.

2. **Duplicate identifier.** Two records match, and the earlier record has nonzero state. The result is 14; the later record is not considered.

3. **Distinct initial captures.** The first table-pointer read identifies table A, the second identifies table B, and lookup subsequently uses table C. Timeout accesses target A’s header, completion targets B’s identifier byte, and selected-record accesses target the address calculated during lookup from C.

4. **Lookup changes between accesses.** At a particular index, the count comes from table A, the matching identifier byte comes from table B, and the matching-address calculation uses table C. Subject to valid accesses, the selected address is in C; the function does not recheck that address’s identifier.

5. **Initial-operation failure.** The scaling dependency returns 7 and the initial operation returns 23. Both timeout stores receive 7000. Completion writes state zero and the failure marker, returns 23, and performs no store through the output parameter.

6. **Fresh timeout argument.** F-d0ab8e462b50 changes the first timeout field to 9000. The initial operation receives 9000 as its fifth argument.

7. **Classification rejection.** The initial operation returns zero and classification returns 2. The result is 91, with no final state or identifier store and no store through the output parameter.

8. **Setup failure.** Classification returns 3 and setup returns 27. Setup observes provisional state one. Completion writes state zero and the failure marker, then returns 27.

9. **Independent first-decision configuration reads.** The +164 access uses configuration A and finds bit 0 set. The separately reloaded pointer for +170 identifies configuration B, whose bit 0 is clear. F-f58de5c36c29 is called, regardless of the corresponding bit in the configuration retained after setup.

10. **Short-circuit suppression.** The +164 bit is clear. No shared-pointer reload or +170 field read occurs for that first decision. The later retained-pointer bit-4 access still occurs.

11. **Retained later configuration.** The pointer retained after setup identifies configuration A. F-143c91667f91 replaces the shared pointer with configuration B but leaves A valid. The later bit-4 decision still reads A.

12. **Failure with output aliasing.** The output location equals the selected state-field address, and setup fails. There is no store through the output parameter, but unsuccessful completion writes zero to that same storage.

13. **Success with output aliasing.** The output location equals the selected state-field address. Publication writes the record address there; the following final state store overwrites it with one.

## Uncertainties

The following remain **documented_only**:

- **Identifier signedness:** whether bytes at or above 128 promote to negative or nonnegative integers.
- **Arithmetic boundaries:** the signed 32-bit multiplication by 1000 has no explicit overflow handling. The index and index-based stride expression use unsigned 32-bit arithmetic in the supplied representation; target address validity and signed address-offset overflow behavior are not independently established.
- **Target representation:** complete ABI, alignment rules, and machine-level consequences of invalid or overflowing address calculations.
- **Pointer stability:** whether separate table or configuration pointer reads identify the same objects.
- **Dependency effects:** global mutations, storage lifetime changes, ownership, failure effects, and whether calls return normally.
- **Auxiliary argument:** its domain meaning and independently established width are absent.
- **Concurrency:** whether external execution can change shared pointers, counts, records, or configuration during the operation.
- **Domain interpretation:** activation, timeout, and selection terminology describes observed use rather than an established external protocol.
- **Invalid-memory outcomes:** no explicit status handles inaccessible storage, invalid pointers, or a failed output store.

## Evidence

Established behavior comes from static inspection of the supplied representation: access widths, record stride, separate shared-pointer reads, retained addresses, short-circuit evaluation, dependency arguments, branch conditions, direct-store ordering, and returned values.

Domain purpose is inferred. Unknown dependency contracts, target-specific behavior, and environmental stability remain documented_only rather than assumed.

No runtime tests have been performed.
