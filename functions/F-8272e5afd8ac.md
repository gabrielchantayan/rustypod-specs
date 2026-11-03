<!-- Trusted provenance: F-8272e5afd8ac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8272e5afd8ac

## Purpose

F-8272e5afd8ac prepares a caller-provided buffer descriptor for a selected length. It conditionally requests storage, records the selected length after storage is accepted, and, for a nonnull source, invokes a transfer dependency followed by a single-byte zero write.

The established contract is not a complete string-assignment or allocation contract:

- Any negative requested length requests length derivation, unless the source is null, which causes immediate failure.
- A nonnegative requested length is used directly, including zero.
- Storage is requested according to the descriptor’s leading signed field and buffer address, not a measured allocation size.
- A zero storage result causes failure before the length assignment, transfer, or trailing zero write.
- A null source with a nonnegative length can reach successful completion without any transfer or direct payload initialization.
- The length assignment occurs before the transfer dependency. Unknown dependency effects and aliasing prevent an unconditional guarantee that the assigned value remains present at return.
- Restoring an address after storage failure does not establish restoration of storage contents, lifetime, ownership, or other state.

String-oriented assignment is inferred intent. Dependency semantics and memory ownership remain unresolved.

## Interface

### Arguments and result

| Item | Represented interpretation |
|---|---|
| Descriptor address | Address of a record with a leading signed field and a buffer-address field. |
| Source | Nullable, 32-bit address-like value. |
| Requested length | Signed 32-bit integer. Negative values request derivation; nonnegative values specify the length directly. |
| Return value | 32-bit result, explicitly either 0 or 1. |

The supplied representation uses 32-bit integers and address-bearing values. It does not establish a portable wider-pointer interface or a complete calling convention.

The routine explicitly returns **0** when either:

- The requested length is negative and the source is null.
- A required storage dependency returns zero.

It explicitly returns **1** after reaching the end of the remaining path. The transfer dependency’s result is ignored, so 1 is not independent confirmation that the transfer achieved any particular effect.

### Descriptor layout

| Byte offset | Width | Directly observed role |
|---|---:|---|
| 0 | 32 bits | Signed threshold for the storage decision; later assigned the selected length after storage acceptance. |
| 4 | 32 bits | Intervening region not directly accessed as a descriptor field. |
| 8 | 32 bits | Buffer address; zero denotes an absent address for storage selection. |

The full record size and meaning of the intervening region are unknown. Absence of a direct field access does not guarantee preservation: dependencies or aliased payload operations may affect descriptor memory.

The leading field is not established as a separately retained allocation capacity. A shorter successful operation can lower it even when existing physical storage is larger.

### Preconditions

The negative-request/null-source rejection does not access the descriptor and therefore does not require a valid descriptor.

Other paths require a descriptor supporting the reads and writes actually reached. Ordinary completion also requires:

- Dependencies to return normally.
- Source and destination memory to satisfy the invoked dependencies’ actual contracts.
- The final byte destination, when reached, to be writable.
- Arithmetic and address calculations to be valid under the execution model.
- Any overlap or aliasing to be compatible with the dependencies’ actual behavior.

These are execution requirements, not validation performed by this routine. In particular, a null source with an explicit nonnegative length does not bypass descriptor access or possible storage failure.

## Behavior

### Length selection

For a nonnegative requested length, that value becomes the selected length unchanged. The length dependency is not invoked, regardless of whether the source is null.

For every negative requested length:

- A null source causes immediate return of 0, without descriptor access or dependency calls.
- A nonnull source is passed to F-1cb585febd84. Its result becomes the selected length.

The original negative values have no individually distinguished meanings.

There is no subsequent check that a derived length is nonnegative or otherwise suitable. A negative dependency result continues into the signed storage comparison and subsequent operations.

Length derivation precedes the descriptor reads used for storage selection. Consequently, effects of F-1cb585febd84 may influence those reads. Preservation of the descriptor or global state by that dependency is not assumed.

### Storage selection and read order

The routine first reads the leading descriptor field and compares it with the selected length using signed comparison.

Storage is required if either:

- The leading field is strictly less than the selected length.
- The buffer-address field is zero.

The condition short-circuits. When the signed comparison is true, the address check belonging to that condition is skipped. Otherwise, the buffer-address field is read to determine whether storage is needed.

After entering the storage path, the routine reads the buffer-address field to obtain the saved address used for dependency selection and possible restoration. This is a separate read from the address check in the condition.

An equal leading field and nonzero address avoid storage. A zero selected length does not necessarily avoid storage: an absent buffer requires it, as does a negative leading field.

### Storage request, installation, and failure

The saved address determines which storage dependency is called:

- A zero saved address selects F-43da6b23bd44, with selected length plus one followed by two zero arguments.
- A nonzero saved address selects F-8b998a07b5e1, with the saved address, selected length plus one, and two zero arguments.

No explicit widening or overflow check is shown for the addition to the signed 32-bit selected length. In particular, the behavior of adding one at the signed upper boundary is unresolved. The dependencies’ interpretation of the size argument is also unknown.

The returned address is written to the descriptor’s buffer-address field before it is checked for zero.

If the returned value is zero:

- The field has already been written with zero.
- The saved address is then written back to that field.
- The routine returns 0.

This exit precedes the routine’s leading-field assignment, transfer call, and trailing zero write. It applies equally to positive, zero, and derived selected lengths whenever their execution reaches a storage request that returns zero.

The routine does not directly assign the leading field on this failure path. That does not guarantee its entry value survives: an earlier dependency may have modified it.

Restoration concerns only the saved address value. It does not prove that the saved storage remains live, that contents are preserved, or that ownership is unchanged. Other dependency effects are not rolled back.

A nonzero returned address remains installed and execution continues.

### Length assignment and source-dependent work

Once storage is accepted—either because no request was required or because a required request returned nonzero—the routine assigns the selected length to the leading descriptor field.

For a null source, it then returns 1. There is no transfer dependency call or direct payload-byte write. Newly obtained storage is not shown to be initialized.

For a nonnull source, the observable order is:

1. The leading field is assigned the selected length.
2. The buffer-address field is read for the transfer destination.
3. D-01 is called with that destination, the source, and the selected length.
4. After D-01 returns, the buffer-address field is read again.
5. One zero-valued byte is written at the newly read address plus the selected length.
6. The routine returns 1.

The transfer result is ignored. The final byte write uses the post-transfer address read, not necessarily the address supplied to D-01.

D-01 may alter descriptor fields through aliases or other access. The routine does not restore the leading field after that call. The trailing byte write can itself affect descriptor bytes if its address overlaps the descriptor.

Likewise, if the source aliases the leading field, the transfer dependency may observe bytes reflecting the newly assigned length rather than the descriptor’s entry state. No source snapshot or alias protection is established.

### Zero selected length

Zero follows the same storage gate as other selected lengths.

If storage is required and its dependency returns zero, the routine installs zero in the address field, restores the saved address, and returns 0. It does not reach the leading-field assignment, transfer, or trailing zero write.

If no storage is required, or a required request returns nonzero:

- With a null source, the routine assigns zero to the leading field and returns 1 without a direct payload or terminator write.
- With a nonnull source, it assigns zero, calls D-01 with a zero length, and writes a zero byte at the buffer address read after D-01 returns.

The direct byte write therefore still occurs for a nonnull source and zero selected length, but only after storage acceptance and normal return from D-01. Whether D-01 itself accesses either address for a zero-length request remains unknown.

### Subsequent calls

The leading-field assignment changes the threshold used by later calls, provided no subsequent dependency effect, aliased write, or intervening activity changes that field.

For example, an explicit null-source request for length 10 against a leading field of 100 and a nonzero buffer requires no storage and assigns 10. Without intervening mutation, a later request for length 50 requires storage, even if the physical allocation could already accommodate that request.

For a nonnull-source operation, the same conclusion additionally depends on D-01 and the trailing byte write leaving the leading field unchanged.

## Side effects

The routine’s direct writes are limited to:

- Installing the storage dependency’s result in the buffer-address field.
- Restoring the saved address after a zero storage result.
- Assigning the selected length after storage acceptance.
- Writing one zero-valued byte after the transfer dependency returns, when the source is nonnull.

The bulk transfer, if any, is delegated. Its exact accesses and their order are not established.

No explicit release dependency is invoked. This does not establish that old storage is preserved: a storage dependency may move, release, replace, or otherwise affect it according to an unknown contract.

There is no shown synchronization or transactional update. Installation of zero and restoration of the saved address are ordered writes, not an atomic failure rollback. The leading-field assignment similarly precedes completion of the source-dependent work.

Dependencies may affect memory, descriptor fields, or global state. No general preservation guarantee is available.

## Dependencies

| Opaque function ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-1cb585febd84 | Receives a nonnull source when the requested length is negative; its result selects the length. | Scan rules, accessed memory, result range, error signaling, and state changes. |
| F-43da6b23bd44 | Receives selected length plus one and two zeros when storage is required and the saved address is zero. | Size units, allocation semantics, initialization, ownership, and external effects. |
| F-8b998a07b5e1 | Receives the saved nonzero address, selected length plus one, and two zeros when storage is required. | Movement, preservation, release behavior, ownership, and failure guarantees. |
| D-01 | Receives destination, source, and selected length after the leading-field assignment; its result is ignored. | Transfer semantics, overlap support, access order, zero-length behavior, failures, and state mutation. |

D-01 is a document-local opaque identifier for the transfer dependency whose published opaque ID was not supplied.

## Acceptance examples

These examples are conditional static criteria, not observed execution results. Unless stated otherwise, assume valid accesses, representable arithmetic, normally returning dependencies, and no descriptor mutation beyond the described effects. Normal return from a storage dependency does **not** imply a nonzero result; examples reaching the length assignment explicitly avoid storage or require a nonzero result.

1. **Negative request with null source.**  
   Any negative requested length and null source produce return 0 without descriptor access or dependency invocation.

2. **Explicit length with null source and sufficient threshold.**  
   With leading field 12, a nonzero buffer address, and requested length 5, no dependency is called. The routine assigns 5 to the leading field and returns 1 without writing a payload byte.

3. **Exact threshold match.**  
   With leading field 5, nonzero buffer address, nonnull source, and requested length 5, storage is not requested. The routine assigns 5, invokes D-01 with length 5, and writes zero at the post-transfer buffer address plus 5.

4. **Absent buffer and successful storage.**  
   With leading field 100, zero buffer address, and requested length 3, F-43da6b23bd44 receives size argument 4 and two zeros. If it returns a nonzero address, that address is installed before the leading field is assigned 3. Subsequent transfer behavior depends on source nullness.

5. **Growth request from an existing address.**  
   With leading field 3, nonzero buffer address, and requested length 8, F-8b998a07b5e1 receives the saved address, size argument 9, and two zeros. A zero result causes restoration and return 0; a nonzero result permits the length assignment.

6. **Storage failure after dependency mutation.**  
   If a storage dependency changes the leading field and returns zero, the routine writes zero to the address field, restores the saved address, and returns 0. It does not undo the dependency’s leading-field change or establish that saved storage remains valid.

7. **Derived length.**  
   For a nonnull source and negative request, a result of 4 from F-1cb585febd84 selects length 4. If storage is required, its size argument is 5. Continuation past storage requires a nonzero result. No second derivation call occurs.

8. **Zero-length transfer after storage acceptance.**  
   With a nonnull source and selected length zero, if no storage is required or a required request returns nonzero, the routine assigns zero, invokes D-01 with zero length, and writes zero at the address read after D-01 returns.

9. **Zero length with absent buffer and failed storage.**  
   With a zero buffer address and explicit requested length zero, F-43da6b23bd44 receives size argument 1 and two zeros. If it returns zero, the routine installs zero, restores the saved zero address, and returns 0 before the leading-field assignment. Neither D-01 nor the trailing byte write occurs, whether the source is null or nonnull.

10. **Zero length, null source, and accepted storage.**  
    If no storage is required or a required request returns nonzero, an explicit zero-length request with null source assigns zero and returns 1 without a direct payload write. This outcome is not guaranteed merely because the requested length is zero.

11. **Transfer changes descriptor state.**  
    If D-01 replaces the buffer address with another valid address, the trailing write uses the replacement plus the selected length. If D-01 also changes the leading field, the routine does not restore the assigned length; the trailing write may independently affect it through overlap.

12. **Repeated threshold reduction.**  
    Starting with leading field 100 and a nonzero buffer, a null-source request for 10 assigns 10 without storage. Without intervening mutation, a subsequent request for 50 enters the storage path regardless of unmeasured physical capacity.

## Uncertainties

The following remain **documented_only**:

- Complete dependency contracts, including memory accesses, global effects, ownership, initialization, release, and failure behavior.
- Whether D-01 performs a byte copy or supports overlap.
- Zero-length access behavior inside D-01.
- Validity and interpretation of negative derived lengths.
- Length conversions inside dependencies.
- Overflow behavior for selected length plus one and for final address arithmetic.
- Behavior with invalid descriptors, inaccessible memory, insufficient storage, or concurrent mutation.
- Whether dependency-side descriptor mutations occur in practice.
- Complete record layout and native calling convention.
- Exceptional or nonreturning dependency behavior.

No validation, safety, ownership, or rollback guarantee is inferred to fill these gaps.

## Evidence

**Established static evidence:** The supplied representation supports the signed length selection, early null-source rejection, descriptor offsets, short-circuit storage condition, storage arguments, result installation before testing, saved-address restoration, length assignment after storage acceptance, transfer call, post-transfer address reload, single-byte zero write, and explicit return values.

The storage-failure branch precedes all three later operations: leading-field assignment, transfer, and trailing zero write. This ordering applies to zero selected lengths as well as other lengths.

**Inferred intent:** The storage request with one additional unit, transfer-shaped dependency, and trailing zero suggest string-oriented buffer assignment. They do not establish retained capacity, byte-copy semantics, or allocation ownership.

**Unknowns:** Dependency implementations, full ABI details, arithmetic boundary behavior, and exceptional execution are not supplied.

No runtime tests have been performed. The acceptance examples are conditional consequences of static evidence, not reported test results.
