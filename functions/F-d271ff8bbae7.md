<!-- Trusted provenance: F-d271ff8bbae7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d271ff8bbae7

## Purpose

F-d271ff8bbae7 conditionally processes an entry selected through a caller-provided context, then optionally performs related-object table processing.

The established behavior has two sequential phases:

1. Prepare a backing object, optionally locate and modify an entry, and decrement a backing-object counter.
2. If the context’s enable byte is nonzero, obtain a status and conditionally inspect or write a selected table record.

The phases are not transactional. Earlier local writes can occur before a later dependency returns a nonzero status. This function performs no explicit rollback. Whether those writes remain observable after intervening dependency calls is not established.

A zero result does not imply that an entry was found, that an association changed, or that a record was written. “Entry,” “association,” and “record” identify structural roles, not established domain meanings.

## Interface

### Inputs and result

| Input | Represented width | Role |
|---|---:|---|
| Context address | 32-bit address | Identifies a structure containing a backing-object address and a second-phase enable byte. |
| Selector | 32-bit integer | Zero bypasses all work. Otherwise, it participates in lookup and table displacement arithmetic. |
| Supplied association | 32-bit address-valued integer | Compared with an entry’s association; also used to obtain dependency arguments and a related-object address. |
| Attribute | 16 bits | Stored in an active writable entry representation. |

The result is represented as a 32-bit integer:

- Zero for a zero selector.
- Zero when the second-phase enable byte is zero.
- Otherwise, the status returned by F-133a6620fb39, unless successful query processing reaches a nonmatching-record path, where the result of F-81c906bf6e3a becomes the status.

A matching-record path returns zero. The meanings of nonzero statuses are unknown.

Widths follow the supplied static representation. The actual ABI and pointer model have not been independently established.

### Accessed layouts

All offsets are byte offsets.

| Object | Offset | Access |
|---|---:|---|
| Context | `0x00` | Read a 32-bit backing-object address. |
| Context | `0x16` | Read a one-byte second-phase enable value. |
| Initially captured backing object | `0x0e` | Read a one-byte eligibility value. |
| Initially captured backing object | `0x20` | Read a 32-bit eligibility value. |
| Initially captured backing object | `0xe0` | Read and write a 32-bit counter. |
| Located entry | `0x22` | Read and conditionally write a signed 16-bit field. |
| Writable entry representation | `0x00` | Read a one-byte active value. |
| Writable entry representation | `0x10` | Write the 16-bit attribute. |
| Writable entry representation | `0x50` | Read and conditionally write a 32-bit association. |
| Old or supplied association | `0x48` | Conditionally read a 32-bit argument for a dependency. |
| Supplied association | `0x4c` | Read a 32-bit related-object address. |
| Related object | `0x34` | Read a 32-bit table-base address. |
| Selected record | `0x00` | Read and conditionally write a one-byte tag. |
| Selected record | `0x01` | Form an address passed to dependencies. |

The record-selection arithmetic uses a five-byte stride. Neither that stride nor the address passed at record offset `0x01` establishes how many bytes a dependency accesses.

### Preconditions

The zero-selector path requires no accessible input objects.

For other paths, every direct read or write must address suitable live storage when that access occurs. In particular:

- The context’s first word must initially be readable.
- The captured backing object must remain usable through its eligibility and counter accesses, including after intervening calls.
- A located entry must support its `0x22` access.
- F-7871c7eca1be must return an address supporting the active-byte read; there is no null check.
- Active-entry replacement requires any nonzero old association to support the `0x48` read and requires the supplied association to support its own `0x48` read.
- An enabled second phase requires the supplied association’s `0x4c` field to be readable.
- Table processing requires valid related-object storage and a valid selected record address.
- Storage passed to dependencies must satisfy their otherwise unknown requirements.

These are path-dependent requirements, not validations performed by the function. No explicit bounds, alignment, lifetime, selector-range, or arithmetic-overflow checks are present. Distinct structural roles are not established to denote disjoint storage.

## Behavior

### Zero-selector bypass

A zero selector produces zero immediately. There are no context or association dereferences and no dependency calls on this path.

The supplied representation also contains a later zero-selector alternative that assigns decimal 11. The enclosing nonzero-selector condition makes that alternative unreachable through the shown control flow. It does not establish an additional callable result.

### Preparation and entry selection

For a nonzero selector, the function captures the backing-object address from the context’s first word and calls F-b93e3fe564f6 with that address.

Eligibility is evaluated after the call, using the captured backing-object address:

- Its byte at `0x0e` must be nonzero.
- Its 32-bit field at `0x20` must compare equal to zero or decimal 13.

The conditions short-circuit. If the byte is zero, the eligibility word is not required by that expression. The word is tested against zero first and, if necessary, against 13; the representation contains a separate field-read expression for each comparison. This does not establish the number of physical machine loads.

When eligible, F-c1f06f89d6d7 receives the captured backing object and the selector. Its zero result means that no entry is processed. Otherwise, the returned entry’s signed 16-bit field at `0x22` governs the next action:

- Zero: call F-7d670f6f009a with the entry.
- Nonzero: read the field, add one, and store the converted result back into the same 16-bit location.

There is no explicit saturation or rejection at the field’s boundary. Under the represented signed-16-bit and 32-bit-integer model, the addition is promoted arithmetic, followed by conversion back to 16 bits. The conversion of a result outside the signed 16-bit range is not established as portable wrapping behavior.

### Counter decrement

Next, the function reads the initially captured backing object’s 32-bit counter at `0xe0`, decreases it by one, and writes it back.

This occurs even if eligibility failed or lookup returned zero. It follows any entry-field handling and precedes writable-entry processing.

The context’s first word is not reloaded for this update. A dependency changing that word therefore does not, by itself, redirect the counter access. Dependencies could nevertheless modify the captured object or its lifetime.

The counter is represented as a signed 32-bit integer. No lower-bound guard is present; decrementing its minimum representable value falls outside the established safe arithmetic domain.

### Writable-entry processing

If lookup returned a nonzero entry, F-7871c7eca1be is called with that entry. The function reads the first byte at the returned address.

If that byte is zero, the function performs no direct association read or attribute write through this branch. It still calls F-44d249c54e70 with the located entry.

If the byte is nonzero, the function reads the association at writable-entry offset `0x50` and compares it with the supplied association.

For equal values, it skips association replacement and both association-field dependency calls. It then writes the attribute at offset `0x10`.

For unequal values, the order is:

1. If the captured old association is nonzero, read its field at `0x48` and call F-44d249c54e70 with that field value.
2. Store the supplied association at writable-entry offset `0x50`.
3. Read the supplied association’s field at `0x48`.
4. Call F-7962111faa24 with that field value.
5. Store the attribute at writable-entry offset `0x10`.

The new association is not checked for zero before its `0x48` access. By contrast, equal zero associations do not trigger either `0x48` access.

After either active-entry path, F-44d249c54e70 is called with the located entry. This is a separate call from the possible old-association-field call.

The writable address and located entry remain captured values across these calls. The function does not recheck activity or association equality after a dependency returns. No dependency is assumed to preserve either object.

### Second-phase gate and query

After first-phase processing, the function reads the context byte at `0x16`. A zero byte returns zero.

For a nonzero byte, the function first captures the related-object address from supplied-association offset `0x4c`. It then:

1. Calls F-c76bdec3f9e1 with the context and selector.
2. Reads the context’s current first word.
3. Calls F-133a6620fb39 with that current backing-object value, the helper result, an address of temporary storage, and a fourth argument of zero.

The backing-object argument can differ from the initial capture. The function does not assume that previous calls preserved the context.

If F-133a6620fb39 returns nonzero, that status is returned immediately. This function performs no subsequent table inspection, record write, or final related-object call on that path.

Earlier local writes have occurred where their branches were taken. This function performs no explicit rollback before returning the query status; persistence of those writes across dependencies is not established.

### Table selection

When F-133a6620fb39 returns zero, F-c76bdec3f9e1 is called again with the context and selector.

Only this second result determines the table displacement: the selector minus the second result minus one, multiplied by five. The first helper result was used for the query, not retained as the table index input.

After the second helper returns, the table-base address is read from captured-related-object offset `0x34`. The selected record address is that base plus the displacement.

The arithmetic uses represented 32-bit integer values. No check establishes nonnegative displacement, representable intermediate results, valid address addition, or an in-bounds record. The expression must not be treated as validated indexing or guaranteed modulo arithmetic.

### Record comparison and conditional write

The selected record’s tag byte is examined.

- If the tag differs from decimal 5, the record takes the nonmatching path without a call to F-d1e995cfb2d3.
- If the tag equals 5, F-d1e995cfb2d3 receives the address immediately following the tag. Its return value is compared with the captured related-object address.

A tag of 5 together with an equal helper result selects the matching path. This establishes neither that the helper reads the supplied address nor that it decodes or preserves the record.

On the matching path, F-81c906bf6e3a and this function’s record-write operations are skipped. The status remains zero.

On the nonmatching path, F-81c906bf6e3a is called with the captured related object. Its result becomes the saved status:

- Nonzero: this function performs no record rewrite.
- Zero: this function writes tag 5, then calls F-cfb711b78946 with the address immediately following the tag and the captured related-object address.

The write and dependency arguments use the previously obtained table base and displacement. The table base is not reloaded after F-d1e995cfb2d3 or F-81c906bf6e3a. There is also no tag recheck after either call.

Every normally returning path through table processing then calls F-44d249c54e70 with the captured related object and returns the saved status. That final call’s return value does not replace the status.

## Side effects

Direct local writes are limited to the path-dependent operations described above:

- Incrementing the located entry’s 16-bit field.
- Decrementing the captured backing object’s 32-bit counter.
- Replacing the writable entry’s association.
- Writing the attribute.
- Writing record tag 5.

Dependencies may introduce other writes, invalidate storage, or alter values previously read or written. Consequently, absence of a direct write does not establish absence of a state change, and occurrence of a direct write does not establish the final contents of that storage.

No ownership transfer, acquisition, release, allocation, or destruction contract is established. In particular, the call positions of F-44d249c54e70 and F-7962111faa24 do not prove reference-management behavior.

There is no visible synchronization, exception handling, rollback, or compensation. The ordered accesses described here are source-level sequencing, not a machine-level memory-order guarantee.

## Dependencies

| Opaque ID | Established caller-side use |
|---|---|
| F-b93e3fe564f6 | Receives the initially captured backing-object address before eligibility checks. |
| F-c1f06f89d6d7 | Receives the backing object and selector; its result determines whether an entry is processed. |
| F-7d670f6f009a | Receives a located entry whose `0x22` field was zero. |
| F-7871c7eca1be | Returns the address used for active-byte, association, and attribute accesses. |
| F-44d249c54e70 | Receives an old association’s `0x48` value, the located entry, or the captured related object on distinct paths. |
| F-7962111faa24 | Receives the supplied association’s `0x48` value after the association store. |
| F-c76bdec3f9e1 | Receives context and selector before the query and again after a zero query status. |
| F-133a6620fb39 | Receives the current backing object, first selector-helper result, temporary-storage address, and zero; supplies a status. |
| F-d1e995cfb2d3 | Receives the address immediately after a tag-5 byte; its return value is compared with the captured related-object address. |
| F-81c906bf6e3a | Receives the related object on the nonmatching path; its status governs the local record rewrite. |
| F-cfb711b78946 | Receives the address immediately after the tag and the related object, after the local tag write. |

These descriptions specify call arguments and result use, not callee internals.

## Acceptance examples

These are conditional criteria, not executed tests. They assume valid storage throughout the relevant accesses, normally returning dependencies, and representable arithmetic unless a boundary is specifically discussed.

1. **Zero selector:** Return zero without input-object dereferences or dependency calls.

2. **Ineligible backing object, disabled second phase:** Call F-b93e3fe564f6, skip lookup, decrement the captured backing-object counter, and return zero.

3. **Lookup returns zero:** Skip writable-entry processing and the final entry call. Still decrement the counter and evaluate the second-phase enable byte.

4. **Nonzero entry field containing −1:** Write zero to the 16-bit field without calling F-7d670f6f009a for that field handling. This invocation does not revisit the zero-field decision.

5. **Inactive writable entry:** Skip direct association and attribute operations, but call F-44d249c54e70 with the located entry. Dependency-induced changes remain unrestricted.

6. **Active entry, equal associations:** Skip both association-field calls and the association store; write the attribute before the final entry call. If both associations are zero and the second phase is disabled, this path requires no supplied-association field read.

7. **Active entry, different nonzero associations:** Call with the old association’s `0x48` value, store the supplied association, call with its `0x48` value, write the attribute, and make the final entry call, in that order.

8. **Query returns nonzero:** Return that status without local table processing or the final related-object call. Earlier local writes have occurred where applicable, and this function performs no explicit rollback before returning. Their persistence is not guaranteed.

9. **Matching comparison:** Given tag 5 and an F-d1e995cfb2d3 result equal to the captured related-object address, skip F-81c906bf6e3a and both local record-update operations. Make the final related-object call and return zero. No preservation or decoding behavior is required of the comparison helper.

10. **Tag differs from 5:** Do not call F-d1e995cfb2d3. Call F-81c906bf6e3a; if it returns zero, write tag 5 before calling F-cfb711b78946.

11. **Nonmatching record, nonzero update status:** Skip the local record rewrite, make the final related-object call, and return the saved F-81c906bf6e3a status.

12. **Different selector-helper results:** Supply the first result to F-133a6620fb39. Use only the second result for table displacement.

13. **Dependency changes the table-base field:** After the table base has been captured, subsequent local record operations still use that captured base. Continued validity of the resulting address is a precondition, not a guarantee.

## Uncertainties

The following remain **documented_only**:

- Exact ABI, byte order, alignment, pointer validity, and object extents.
- Target-specific conversion of out-of-range values back to signed 16 bits.
- Behavior outside representable signed arithmetic and valid address arithmetic.
- Valid selectors, table bounds, and relationships between selector-helper results.
- Domain meanings of eligibility value 13, tag 5, and the accessed fields.
- Dependency status meanings, global effects, ownership rules, lifetime effects, and concurrency behavior.
- The size, initialization requirements, and use of temporary storage passed to F-133a6620fb39.
- Whether either record-address dependency reads or writes memory, and any associated representation.
- Behavior when dependencies do not return normally or invalidate storage required by later accesses.

Reference-management and encoded-record interpretations are inferred intent only, not requirements established by the supplied function.

## Evidence

This contract derives from the supplied static function representation and opaque dependency mapping. The prior draft, feedback, and embedded comments are not independent behavioral authority.

Established evidence comprises visible branch conditions, access widths and offsets, captured versus reloaded values, call arguments, direct writes, arithmetic expressions, and return-status selection.

Callee internals, persistence across calls, ownership effects, and machine-level execution details are not established. No runtime tests, implementation execution, or independent binary verification have been performed.
