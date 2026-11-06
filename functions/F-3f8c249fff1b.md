<!-- Trusted provenance: F-3f8c249fff1b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f8c249fff1b

## Purpose

**Established behavior:** F-3f8c249fff1b conditionally updates a structure containing three 32-bit address fields. A nonzero third argument selects a path that may insert a three-word record. A zero third argument selects a path that may remove or otherwise process an existing record.

Both paths begin with an unconditional call to F-7a71c02c943a. The relationship between that call’s result and the structure’s current address at offset 12 determines whether further work is attempted.

**Inferred intent:** The address fields resemble the beginning, logical end, and storage limit of a contiguous collection of 12-byte records. The nonzero path resembles insertion in unsigned-key order; the zero path resembles removal involving a position returned by the initial dependency. These interpretations are useful descriptions, not independently established guarantees about the dependencies.

The function has no explicit return value or direct error-reporting channel.

## Interface

The function accepts four machine-word arguments:

| Argument | Width and interpretation |
|---|---|
| Structure address | A 32-bit address used to access fields at byte offsets 8, 12, and 16. |
| Key | A 32-bit unsigned integer. It is compared with record keys and becomes the third word of the temporary insertion record. |
| Mode | A 32-bit value tested only for equality with zero. Every nonzero bit pattern selects the same branch. |
| Auxiliary word | An uninterpreted 32-bit value used in initial local storage but not retained in the record subsequently supplied to insertion dependencies. |

The function returns no value.

### Structure and record layout

The directly accessed structure fields are:

| Byte offset | Width | Established use |
|---|---|---|
| 8 | 32 bits | Supplies the first record address for the nonzero-mode scan. The address of this field is also passed to collection-related dependencies. |
| 12 | 32 bits | Supplies an end-like address. It is compared with the initial dependency result, updated during direct append, and decremented during the zero-mode path. |
| 16 | 32 bits | Supplies a storage-limit-like address, consulted only when the scan finishes at the saved end-like address. |

A scanned record has a stride of 12 bytes. Its key is the unsigned 32-bit word at record offset 8.

The temporary insertion record occupies three consecutive 32-bit words:

1. A value read from one shared 32-bit datum.
2. A value read from that same shared datum.
3. The supplied key.

The source does not establish a semantic type for the first two words, such as pointers, counters, or ownership handles. Nor does it establish byte order beyond the execution environment’s native interpretation of 32-bit accesses.

### Preconditions

The source performs no null checks, bounds validation, alignment validation, or overflow checks.

Defined use therefore requires the structure fields and all actually accessed record words to be readable or writable as appropriate. In particular, the scan may read the key word at the end-like address before determining that it has reached that address.

Any stronger requirements imposed by the dependencies remain documented_only. The function itself does not verify that the three address fields describe a consistent allocation, that the records are sorted, or that a returned dependency address belongs to the collection.

## Behavior

### Common entry

Before calling any dependency, the function initializes a local three-word area from the key, mode, and auxiliary arguments.

It then calls F-7a71c02c943a without explicit arguments and retains its 32-bit result. This call occurs even when the later branch immediately returns.

The mode test follows that call. Consequently, the structure comparisons described below observe state after F-7a71c02c943a returns. The source does not justify assuming that this dependency preserves the structure or shared state.

### Nonzero mode

The function first reads the structure’s field at offset 12 and compares it with the retained result from F-7a71c02c943a.

If they differ, the function returns immediately. It makes no further dependency calls and performs no direct structure write on this path.

If they are equal, the function prepares the temporary insertion record:

- Its first two words are assigned from the shared 32-bit datum.
- Its third word is subsequently assigned the supplied key before either insertion dependency is called.

After initializing the first two words, the function reads the field at offset 12 again and saves that value as the scan boundary. It also reads the field at offset 8 as the initial scan address.

#### Scan and access order

At each scan position, the function first reads the unsigned 32-bit key at that address plus 8.

It advances the scan address by 12 only if both conditions hold:

- The observed record key is strictly less than the supplied key.
- The current scan address is not equal to the saved boundary.

The key comparison is evaluated before the boundary comparison. Reaching the boundary therefore does **not** prevent a key read at boundary plus 8.

The scan stops at either:

- A position whose key is greater than or equal to the supplied key; or
- The saved boundary.

This function has no explicit duplicate-key rejection inside the scan. Equality stops the scan at the first equal key encountered. Whether the earlier dependency prevents insertion of an already-present key is unknown.

If the boundary is not reached and every observed key remains smaller than the supplied key, the source supplies no independent termination condition. Address wrapping, inaccessible storage, or a malformed range are not handled.

#### Choosing the insertion operation

After the scan, the temporary record’s third word is set to the supplied key.

If the scan address equals the saved boundary, the function reads the structure’s field at offset 16. When that field differs from the saved boundary:

1. It writes the saved boundary plus 12 to the structure’s field at offset 12.
2. It calls F-c2194f7aec32 with:
   - The address of the structure’s field at offset 8;
   - The saved boundary address;
   - The address of the temporary three-word record.

Thus the end-like field is advanced **before** F-c2194f7aec32 executes. The source contains no rollback if that dependency does not complete normally.

In every other case, the function calls F-248413be431d with:

- The address of the structure’s field at offset 8;
- The final scan address;
- The address of the temporary three-word record.

This alternative includes both insertion before the saved boundary and arrival at a boundary equal to the storage-limit-like field. When the scan stops before the boundary, the field at offset 16 is not read by this decision.

After either insertion dependency returns normally, the function calls F-ed06b5c03b38 with the address of the temporary record, then returns.

The source establishes that the temporary record is processed after insertion. It does not establish whether that processing releases resources, clears values, decrements references, or does something else.

### Zero mode

The function reads the structure’s field at offset 12 and compares it with the retained result from F-7a71c02c943a.

If they are equal, it returns immediately.

If they differ, it calls F-c8b7397c6007 with the address of the structure’s field at offset 8.

A nonzero result from F-c8b7397c6007 causes an immediate return. This function performs no subsequent direct write or additional call on that path. Effects already produced by either dependency are not undone.

If F-c8b7397c6007 returns zero, the function proceeds as follows:

1. It reads the structure’s field at offset 12 and compares it with the retained initial dependency result plus 12.
2. If they differ, it calls F-343aa283b650 with the retained initial dependency result plus 12.
3. It reads the structure’s field at offset 12 and calls F-ed06b5c03b38 with that value minus 12.
4. After F-ed06b5c03b38 returns, it reads the field at offset 12 again, subtracts 12, and writes the result back to that field.
5. It returns.

These separate reads matter. The address supplied to F-ed06b5c03b38 and the final stored end-like address are not necessarily derived from the same field value. A dependency may change the field between those operations.

Likewise, F-343aa283b650 may change the structure before the address for F-ed06b5c03b38 is obtained. Its single explicit argument does not establish that its effects are limited to that address.

### Arithmetic and comparison rules

Record stepping and end-field adjustments use 12-byte increments or decrements in the 32-bit address representation.

Record-key ordering is unsigned. For example, a key with its high bit set compares greater than a small nonnegative key; it is not treated as a negative number.

Address comparisons in this function are equality or inequality tests, not ordering tests. The function does not establish that the beginning precedes the end or that the end precedes the storage limit.

No check prevents addition past the representable address range or subtraction below zero. The source does not establish portable behavior for such cases; valid use should avoid them.

## Side effects

### Directly established effects

The function can:

- Read structure words at offsets 8, 12, and conditionally 16.
- Read unsigned record keys at successive 12-byte positions.
- Read the shared datum used to initialize the temporary record’s first two words.
- Write the structure’s field at offset 12 before a direct-append call.
- Write that same field after the zero-mode cleanup call.
- Supply stack-local record storage to insertion-related and cleanup-related dependencies.

It does not directly write the scanned record keys or the structure’s other two address fields.

### Dependency-mediated effects and ownership

All dependency effects beyond their explicit arguments are unknown. No dependency may be assumed to preserve globals, preserve the structure, allocate storage, release storage, or transfer ownership.

The temporary record has storage local to this invocation. Its address is supplied to dependencies only after all three words have been replaced with insertion values. The incoming mode and auxiliary values therefore are not forwarded in that temporary record.

The insertion dependencies may copy, move, inspect, or otherwise process the temporary record; their exact behavior is not established here. The subsequent F-ed06b5c03b38 call does not by itself prove either ownership transfer or resource release.

There is no explicit error recovery, transaction boundary, or synchronization. If a dependency fails to return normally, subsequent calls and writes are not established to occur.

## Dependencies

| Opaque function ID | Established interaction | Unknown contract |
|---|---|---|
| F-7a71c02c943a | Called once at entry without explicit arguments. Its 32-bit result controls branch eligibility and supplies an address used in zero mode. | How it obtains its result, whether it searches, and what state it changes. |
| F-c2194f7aec32 | Called on the direct-append path after the end-like field is advanced. Receives the collection-header address, saved boundary, and temporary-record address. | Construction semantics, resource behavior, and failure behavior. |
| F-248413be431d | Receives the collection-header address, scan position, and temporary-record address for other insertion cases. | Shifting, allocation, field updates, and ownership behavior. |
| F-c8b7397c6007 | Called only in zero mode when the initial result differs from the current end-like field. A nonzero result stops further processing. | Meaning of its result and any state changes. |
| F-343aa283b650 | Conditionally called in zero mode with the initial dependency result plus 12. | Whether it shifts records, changes boundaries, or performs another operation. |
| F-ed06b5c03b38 | Processes the temporary record after insertion, or an address derived from the current end-like field in zero mode. | Whether it destroys, releases, resets, or otherwise processes its argument. |

No recursive call to F-3f8c249fff1b is visible.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Unless stated otherwise, dependencies return normally and do not change relevant fields.

1. **Nonzero-mode early return.**  
   If F-7a71c02c943a returns an address different from the structure’s field at offset 12, no scan, insertion call, cleanup call, or direct structure write follows.

2. **Insertion before a larger key.**  
   Suppose the scan begins at a readable record whose key is 20, the supplied key is 10, and the scan address differs from the saved boundary. The scan stops at that record. F-248413be431d receives its address and a temporary record whose third word is 10. F-ed06b5c03b38 subsequently receives the temporary record’s address.

3. **Equal-key scan position.**  
   If a scanned key equals the supplied key, the scan stops at that position. Subject to the entry gate allowing the operation, this function does not itself discard the request as a duplicate.

4. **Unsigned ordering.**  
   With supplied key 2 and an observed key of 4,294,967,295, the scan stops at that record. It does not advance on the basis of signed negative ordering.

5. **Direct append with spare storage.**  
   Suppose the scan reaches the saved boundary, the key word at boundary plus 8 is readable, and the field at offset 16 differs from the boundary. The field at offset 12 is written to boundary plus 12 before F-c2194f7aec32 is called.

6. **Boundary equal to storage limit.**  
   Under the same readable-boundary conditions, if the saved boundary equals the field at offset 16, F-248413be431d is selected instead. This function does not perform the direct-append end-field write.

7. **Empty-range boundary access.**  
   If the beginning and saved boundary are equal, the scan still reads the word at that address plus 8. Equality of those fields alone does not make the nonzero path safe for an unreadable boundary.

8. **Zero-mode gate.**  
   If the initial result differs from the end-like field but F-c8b7397c6007 returns nonzero, neither F-343aa283b650 nor F-ed06b5c03b38 is subsequently called.

9. **Zero-mode single-record distance.**  
   If the end-like field equals the initial result plus 12 after the gate returns zero, F-343aa283b650 is skipped. F-ed06b5c03b38 receives end minus 12, and the end-like field is then decremented by 12.

10. **Cleanup changes the end-like field.**  
    If F-ed06b5c03b38 changes that field during zero mode, the final write uses the changed value minus 12. It does not restore a value calculated before the call.

## Uncertainties

The following remain **documented_only**:

- The semantic type and full layout of the enclosing structure.
- Whether the three address fields actually implement a contiguous dynamic collection.
- The origin and meaning of the shared initialization datum.
- Whether F-7a71c02c943a identifies a key, an element, a sentinel, or another position.
- Whether dependencies preserve sorted order, reject duplicates, reallocate storage, or modify shared state.
- The meaning of the nonzero gate result in zero mode.
- Allocation, deallocation, reference-counting, and ownership rules.
- Required capacity or readable padding at the scan boundary.
- Failure signaling and behavior during abnormal dependency termination.
- Concurrency and reentrancy guarantees.
- Valid behavior for malformed addresses, inconsistent fields, or arithmetic overflow.

The scan supports an interpretation of ascending unsigned-key insertion only when the existing range already satisfies that ordering and the dependencies maintain it. The function neither checks nor independently guarantees that invariant.

## Evidence

The established claims come solely from the supplied static function body: its argument usage, 32-bit word accesses, offsets, comparison order, branch conditions, dependency arguments, and direct writes.

Collection, insertion, append, and removal terminology expresses inferred intent where dependency behavior is unavailable. It does not replace the explicit access and call sequence specified above.

No runtime tests have been performed. No dependency implementation or external behavioral contract was available.
