<!-- Trusted provenance: F-1bf7bc1ad8ce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1bf7bc1ad8ce

## Purpose

F-1bf7bc1ad8ce conditionally refreshes state associated with a caller-supplied object, performs mode-dependent updates, and optionally passes a temporary two-word record through additional dependencies.

Two materially different return paths are established:

- If the object’s byte at offset `0x1C5` is nonzero, the function returns immediately with that byte in the low 32 bits and the second input in the high 32 bits.
- Otherwise, the function refreshes selected fields and returns a 64-bit value assembled from a temporary two-word record. That record initially contains the object address and the address at object offset `0x80`, but dependencies may modify it before return.

The interpretation of the refreshed values as configuration, identifiers, handles, or another domain-specific concept is not established. “Refresh” and “mode” describe the visible control flow, not a proven application-level purpose.

## Interface

### Inputs

The supplied representation exposes two inputs:

| Input | Width | Established use |
|---|---:|---|
| Object address | 32 bits | Base address for byte and 32-bit field accesses; also passed to dependencies. |
| Auxiliary value | 32 bits | Supplies the high half of the immediate-return result only. |

The object address must identify memory supporting the accesses actually reached. No null check, object-size check, or address-range validation is visible.

The second input has no established use on the normal refresh path.

### Result

The result is a 64-bit concatenation of two 32-bit words:

- Bits 31–0 contain the low word.
- Bits 63–32 contain the high word.

For the immediate-return path:

- The low word is the zero-extended byte read at object offset `0x1C5`.
- The high word is the auxiliary input.

For the normal-return path:

- The low word is the first word of a temporary record.
- The high word is the second word of that record.
- Before optional dependency calls, these are initialized to the object address and object address plus `0x80`, respectively.

This establishes word significance, not an external byte serialization format.

### Relevant object layout

Offsets are byte offsets from the object address.

| Offset | Access width | Established role |
|---|---:|---|
| `0x34` | 32 bits | Optional related-object address, captured before temporary-record processing. |
| `0x38` | 32 bits | Input to each of three query calls; later copied into `0x1E8` in one mode. |
| `0x80` | Address only | Interior address used in the temporary record and dependency arguments. |
| `0xA0` | 8 bits | A zero value permits the optional final processing block. |
| `0xA9` | 8 bits | Cleared on the non-immediate path. |
| `0x164` | 32 bits | Required to be zero for ordinary refresh processing. |
| `0x1C5` | 8 bits | Nonzero immediate-return discriminator and low-result value. |
| `0x1DC` | 32 bits | Cached first-query value, after zero-to-two normalization when updated. |
| `0x1E8` | 32 bits | Tested and replaced during mode-one processing. |
| `0x1EC` | 8 bits | Set to one during mode-zero processing. |
| `0x1ED` | 8 bits | Cleared when the cached pair is updated. |
| `0x1F0` | 32 bits | Receives the second query result; subsequently selects mode behavior. |
| `0x1F4` | 32 bits | Cached third-query value. |

Offset `0x1F4` is the same location as decimal offset 500. It is one field, not two separate fields.

## Behavior

### 1. Immediate return

The first visible action is a byte read at offset `0x1C5`.

If that byte is nonzero, the function returns the auxiliary input in the high word and the byte value in the low word. No object writes or dependency calls occur on this path.

The supplied representation reads the byte again to construct the result. With stable memory, the tested value and returned value are identical. Concurrent mutation semantics are not established; the contract does not convert these accesses into a single guaranteed snapshot.

### 2. Initial clearing and invariant enforcement

If the first byte read is zero:

1. The function writes zero to the byte at `0xA9`.
2. It captures a shared 32-bit query argument from an external location.
3. It reads the 32-bit field at `0x164`.

If `0x164` is nonzero, the function calls F-282eebd7c413 without explicit arguments, then calls F-feee9156a257 without explicit arguments. The latter is identified in the supplied representation as non-returning.

Consequently, the established ordinary path requires `0x164` to be zero at this check. A nonzero value does not produce an ordinary error return. The earlier write to `0xA9` has already occurred.

On the ordinary path, the function writes a 32-bit zero to `0x164`, even though the preceding check found it zero.

### 3. Query sequence

The function makes three calls to F-5710a8628cba in this order:

1. A call receiving the current 32-bit field at `0x38`, the captured shared argument, and a first externally supplied selector. Its 32-bit result is retained for later comparison and normalization.
2. A call receiving a fresh read of `0x38`, the same captured shared argument, and a second externally supplied selector. Its 32-bit result is written immediately to `0x1F0`.
3. A call visibly receiving another fresh read of `0x38` and the captured shared argument. Its 32-bit result is retained for cache comparison.

The supplied representation does not expose a third argument for the third call. Its actual calling convention and any additional effective inputs remain documented_only unknowns.

These calls do not establish a stable snapshot of the object:

- `0x38` is read separately for each call.
- The shared argument is captured once and reused.
- The selectors are external values whose contents are not supplied.
- The write to `0x1F0` occurs before the third call.
- No preservation of object fields or global state by F-5710a8628cba is assumed.

Thus, later mode processing uses the value actually present at `0x1F0` when read, not necessarily an immutable copy of the second query result.

### 4. Cached-pair comparison and normalization

The function may skip the cached-pair update and mode handling, but only when all of the following hold:

- The third query result is nonzero.
- The existing 32-bit field at `0x1F4` equals the third query result.
- The existing 32-bit field at `0x1DC` equals the first query result.
- The first query result is nonzero.

The cached `0x1DC` field is read for this comparison only after a nonzero third result matches `0x1F4`.

Every other case enters the update path. In that path, a zero first-query result is replaced by the 32-bit value two. A nonzero first-query result is retained unchanged.

In particular:

- A zero third-query result always causes an update.
- A zero first-query result always causes an update and becomes two.
- A matching cached pair with a zero first-query result does not qualify for the skip.
- The second query result does not participate in this skip decision.

On the update path, the following writes occur in order:

1. Store the third query result at `0x1F4`.
2. Store the normalized first query result at `0x1DC`.
3. Clear the byte at `0x1ED`.

The update path then performs mode handling. The skip path bypasses these writes and all mode handling, but still continues into the final optional-processing stage.

### 5. Mode handling

Mode handling is reached only after the cached-pair update.

#### Mode one

If a 32-bit read of `0x1F0` equals one:

- Read `0x1E8`.
- If it is nonzero, call F-ecc1f4c34dbc with the object address, then call F-40696ba6c03c with the object address.
- After those calls, or immediately if `0x1E8` was zero, read `0x38` and store that 32-bit value at `0x1E8`.

The final copy reads `0x38` after the calls. If either dependency changes `0x38`, the newly observed value is copied.

This branch does not directly set or clear `0x1EC`.

#### Mode zero

If the first mode comparison is not equal to one, the function performs another 32-bit read of `0x1F0` and compares it with zero.

If that read equals zero:

1. Write one to the byte at `0x1EC`.
2. Call F-40696ba6c03c with the object address.

There is no subsequent direct assignment to `0x1E8` in this branch.

#### Other modes

If neither comparison selects a branch, there are no additional direct mode-specific writes or calls.

The two mode tests are distinct reads. No concurrency or atomicity guarantee is established.

### 6. Temporary-record initialization and eligibility checks

Both the cached-pair skip path and the completed update path initialize a temporary record containing two adjacent 32-bit words:

| Word | Initial value |
|---|---|
| First | Object address |
| Second | Object address plus `0x80` |

The function then calls F-3fac671648b8 without explicit arguments.

The optional processing block is entered only if all three conditions succeed, in this order:

1. F-3fac671648b8 returns a nonzero 32-bit value.
2. The byte at object offset `0xA0` is zero.
3. A byte reached through an external pointer is nonzero.

The checks short-circuit. If the dependency returns zero, neither later byte check is required. If `0xA0` is nonzero, the external byte is not read.

F-3fac671648b8 is called even when the cached-pair update was skipped.

### 7. Optional temporary-record processing

If all eligibility checks pass:

1. Capture the 32-bit related-object address from `0x34`.
2. Call F-e57b05520b65 with the address of the temporary record and two copies of the object’s interior address at `0x80`.
3. If the captured related-object address is nonzero, call F-ee8733a08210 with the temporary-record address and the captured address plus `0xD0`.
4. Call F-ffe0d92c57b7 with the object address.
5. If its 32-bit result is nonzero, call F-c9b5d17d8d1e with that result and the temporary-record address, followed by F-4d037ecf0f82 with the same result and record address.

The related-object address is captured before F-e57b05520b65. A later change to `0x34` does not change the already captured argument used for the `0xD0` calculation.

Likewise, the nonzero result from F-ffe0d92c57b7 is reused for both subsequent calls. The second call is not conditional on any visible return value from the first.

Finally, the function returns the temporary record’s current first word as the low result word and its current second word as the high result word.

## Side effects

### Direct writes

On the non-immediate ordinary path, `0xA9`, `0x164`, and `0x1F0` are written before cached-pair skipping can occur.

The fields `0x1F4`, `0x1DC`, and `0x1ED` are written only when the cached pair is updated.

The field `0x1E8` is directly assigned only in mode one. The byte `0x1EC` is directly assigned only in mode zero.

Fields not directly cleared by a branch retain their prior values only to the extent that invoked dependencies leave them unchanged. For example, the function does not directly clear `0x1EC` when entering mode one.

### Ownership and lifetime

No explicit allocation, release, reference-count operation, or ownership transfer is established by the visible body.

The temporary two-word record is local to the invocation. Dependencies receive its address during calls. Whether they retain that address, allocate resources, or release resources is unknown. The visible caller does not establish that retaining the temporary address is safe.

Object fields and returned 32-bit values may represent pointers or handles, but their ownership semantics are not established merely by their use as arguments.

### Arithmetic and failure conditions

Addresses and record words are represented as 32-bit quantities. Relevant address calculations are additions of `0x80` and `0xD0`, plus fixed object-field offsets.

The supplied evidence does not establish checked arithmetic, overflow rejection, valid wraparound addressing, alignment recovery, or memory-fault handling. Required storage and address calculations must be valid for the execution environment.

No ordinary error result is explicitly constructed. Zero query results have branch-specific meanings rather than a uniform failure interpretation.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-282eebd7c413 | Called when the checked field at `0x164` is nonzero; no explicit arguments. |
| F-feee9156a257 | Called next on that path; marked non-returning in the supplied representation. |
| F-5710a8628cba | Called three times to obtain 32-bit values; third-call interface is incomplete. |
| F-ecc1f4c34dbc | Called with the object during mode one when `0x1E8` is nonzero. |
| F-40696ba6c03c | Called after F-ecc1f4c34dbc in that case, or after setting `0x1EC` in mode zero. |
| F-3fac671648b8 | No explicit arguments; nonzero result permits further eligibility checks. |
| F-e57b05520b65 | Receives the temporary record and two identical interior object addresses. |
| F-ee8733a08210 | Receives the record and captured related-object address plus `0xD0`. |
| F-ffe0d92c57b7 | Receives the object; a nonzero result enables two subsequent calls. |
| F-c9b5d17d8d1e | Receives that result and the temporary record. |
| F-4d037ecf0f82 | Receives the same result and record after F-c9b5d17d8d1e. |

No dependency is assumed to preserve globals, preserve object fields, or implement a particular resource-management operation.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, memory is valid and stable except for specified writes, dependencies return normally where expected, and no unspecified dependency mutation changes the scenario.

1. **Immediate byte result.** With `0x1C5` equal to `0x7F` and auxiliary input `0x12345678`, the result is `0x123456780000007F`. No direct writes or dependency calls occur.

2. **Invariant violation.** With `0x1C5` zero and `0x164` nonzero, `0xA9` is cleared before F-282eebd7c413 and F-feee9156a257 are called. No normal return is established.

3. **Matching nonzero pair.** If the first query returns seven, the third returns nine, and the cache contains seven at `0x1DC` and nine at `0x1F4`, the pair update and mode handling are skipped. The second query has nevertheless already run and its result has been stored at `0x1F0`.

4. **Zero normalization despite a match.** If the first query returns zero and the third returns nine, even cached values zero and nine do not permit the skip. The update stores two at `0x1DC`, nine at `0x1F4`, and zero at `0x1ED`.

5. **Zero third result.** If the first query returns seven and the third returns zero, the update occurs even if the cache already contains seven and zero.

6. **Mode-one call order.** If an update occurs, the mode read is one, and `0x1E8` is nonzero, F-ecc1f4c34dbc precedes F-40696ba6c03c. The subsequent value read from `0x38` is then stored at `0x1E8`.

7. **Mode-zero visibility.** If mode zero is selected, F-40696ba6c03c is invoked after the byte at `0x1EC` has been set to one.

8. **Final gate disabled.** If F-3fac671648b8 returns zero, no later eligibility reads or temporary-record dependencies occur. The normal result has the object address in its low word and object address plus `0x80` in its high word.

9. **Record mutation affects return.** If optional processing occurs and its dependencies leave the record’s first word as `0x11223344` and second word as `0x55667788`, the result is `0x5566778811223344`.

## Uncertainties

The following remain documented_only:

- Application-level meanings of the object, selectors, modes, cached values, and external byte.
- The complete interface and effective inputs of the third F-5710a8628cba call.
- Dependency side effects, ownership rules, retained references, and internal errors.
- Whether the non-returning designation for F-feee9156a257 is accurate in every execution environment.
- Concurrency, atomicity, memory ordering, and behavior under asynchronous mutation.
- Concrete external values and initial object contents.
- Address-overflow behavior and the target’s memory-validity guarantees.

The two-word normal result may suggest an address pair or transformed record, but that interpretation is inferred intent, not an established interface type.

## Evidence

Established behavior derives solely from the supplied static representation: access widths and offsets, branch predicates, visible call order, direct writes, argument construction, temporary-record initialization, and return-word concatenation.

Dependency names have been replaced with opaque function IDs. No claim about dependency internals follows from a name or presumed purpose.

No runtime tests have been performed. The acceptance examples express consequences of the visible control flow under explicit conditions; they are not observations of execution.
