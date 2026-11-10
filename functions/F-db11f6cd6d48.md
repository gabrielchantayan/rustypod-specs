<!-- Trusted provenance: F-db11f6cd6d48; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db11f6cd6d48

## Purpose

F-db11f6cd6d48 coordinates an operation on two word-array numeric objects. It selects a computational dependency, arranges result and scratch capacity, conditionally writes padding into an operand, and trims zero words from the high end of the working result. Exact destination/operand descriptor aliasing triggers an attempt to acquire a working descriptor.

**Inferred intent:** signed multiple-precision multiplication. The operand-count sum, exclusive-OR of sign-like fields, zero shortcut, and specialized equal-size computation paths support this interpretation. The supplied body does not establish the mathematical contracts of the computational dependencies.

The function returns a 32-bit status: 1 on its success paths and 0 on its explicitly handled failure paths. Neither status establishes dependency behavior beyond the checks described below.

## Interface

### Arguments and object layout

The function receives:

1. A destination descriptor pointer.
2. A first operand descriptor pointer.
3. A second operand descriptor pointer.
4. A 32-bit context token, forwarded unchanged to context-entry, context-exit, and descriptor-acquisition dependencies.

The observed descriptor layout is:

| Byte offset | Width | Observed role |
|---|---:|---|
| 0 | 32 bits | Word-storage address |
| 4 | 32 bits | Active word count |
| 8 | 32 bits | Available word capacity |
| 12 | 32 bits | Sign-like field |
| 16 | 32 bits | Flags, including mask value 2 |

Counts and capacities participate in signed 32-bit comparisons. Each storage word occupies four bytes. Normalization visits larger word indices first, supporting—but not independently proving—a least-significant-word-first numeric representation.

The sign-like assignment is a full-width, 32-bit exclusive-OR. No restriction to boolean values is imposed.

### Preconditions

There is no explicit validation of descriptor pointers, storage pointers, active counts, capacities, or context state. Safe operation requires accessible memory for the actual reads and writes of the selected path, including accesses following dependency calls.

Ordinary numeric use additionally requires:

- Nonnegative active counts consistent with accessible storage.
- Capacity sufficient for each access, whether initially present or established through a dependency.
- A usable original operand-count sum.
- Valid four-byte word-offset calculations.
- Writable padding storage whenever the padding branch is taken.
- Dependency behavior compatible with the caller’s later descriptor and storage accesses.

These are requirements for safe use, not checks performed by this function. Invalid pointers, negative counts, and inconsistent descriptors do not receive a defined validation error.

### Return value

**1** is returned when either:

- The zero-setting dependency returns nonzero; or
- A nonzero-operand path completes computation, normalization, any conditionally required transfer, and context exit.

**0** is returned after:

- A zero return from the zero-setting dependency;
- A null alias-working acquisition; or
- A null result from a checked capacity operation.

The function does not inspect computational return values or a transfer return value. An unchecked invalid access, such as dereferencing a null scratch descriptor, is not converted into status 0.

## Behavior

### Initial reads and zero shortcut

The function first captures the first operand’s active count, then the second operand’s active count. These captured values drive the initial path selection.

If either count is exactly zero, F-d4e7c5a40239 receives the destination and numeric value zero.

- A nonzero return produces immediate status 1.
- A zero return causes F-4bd8a1faf697 to receive the context token, followed by status 0.

Neither zero-shortcut branch calls context entry. The successful shortcut also bypasses context exit, sign assignment, computation, normalization, and working-result transfer.

A negative count does not satisfy the zero condition.

### Nonzero setup and working-descriptor selection

For two nonzero counts, their 32-bit sum is formed before context entry. This **original sum** remains the nominal result count even if a later padding operation increases an effective operand count.

F-ea92bf179106 is then called with the context token. Its return value is not inspected.

The destination initially serves as the working descriptor. If its pointer equals either operand descriptor pointer, F-834e12b6dfde is called with the context token.

- Null causes context exit and status 0.
- A nonnull return becomes the working descriptor.

Acquisition does not establish freshness, distinctness from the destination or operands, nonoverlapping storage, initialization, or ownership. In particular, a successful acquisition may return the destination descriptor itself. The final transfer decision uses a separate pointer comparison after normalization.

Only descriptor-pointer equality triggers acquisition. Distinct descriptors sharing storage are not detected.

After this selection, the operand sign-like fields are read and their 32-bit exclusive-OR is written to the working descriptor’s sign-like field. This occurs after context entry and any alias-working acquisition, but before subsequent capacity and computational calls.

The assignment is an observable write, not a guarantee about the eventual sign-like field. Later dependencies may change that field or other reachable state.

### Fixed eight-word path

When both captured counts equal 8, the function selects F-4701769f3e09.

The working capacity is compared, as a signed value, with 16. If insufficient, F-08bc2fc94856 receives the working descriptor and requested capacity 16. A null return causes context exit and status 0.

After successful capacity handling:

- The working active count is set to 16.
- F-4701769f3e09 receives the working storage address and the two operand storage addresses.
- Execution continues to normalization.

No count argument is supplied to this computational dependency.

The capacity helper’s return is used as a null/non-null indicator. It does not replace the working descriptor. Later accesses still use the original working descriptor, so a usable capacity result must be available through that descriptor; merely returning an unrelated replacement descriptor would not satisfy this caller.

### Large equal-size or equalizable path

Outside the eight-word case, large-path consideration requires both captured counts to exceed 15 under signed comparison.

The path can proceed to candidate-size evaluation when:

- The counts are equal; or
- They differ by exactly one, and the shorter operand qualifies for padding.

Padding requires both:

- The shorter operand’s flags lack mask value 2.
- Its captured count is strictly less than its then-current capacity.

When padding is allowed, the function writes a zero word immediately after the shorter operand’s captured active range. It then increases the corresponding effective count by one. It does not directly update that operand descriptor’s active count.

This padding write occurs before candidate-size evaluation, scratch acquisition, and subsequent capacity checks. It is not rolled back by any explicit action in this function. Later dependencies could nevertheless overwrite the padded location.

If unequal counts cannot be equalized under these rules, execution uses the general path without this padding write.

#### Candidate derivation and scratch acquisition

For equal or successfully equalized counts, F-f7e02acefeaa receives the effective first count, which is now the common count.

The function derives a 32-bit candidate by shifting a one-bit value left. The shift amount is the low eight bits of the dependency result minus one. Masking to eight bits permits values beyond 31; behavior outside the valid range for a 32-bit shift is not established by the supplied material.

F-834e12b6dfde is then called for a scratch descriptor **before** the common count is compared with the candidate.

If the common count differs from the candidate, execution falls back to the general path. Scratch acquisition has still occurred, and any prior padding remains part of the observable execution history.

#### Selected large computation

If the common count equals the candidate:

1. Scratch capacity is checked against four times the candidate, measured in words.
2. If insufficient, F-08bc2fc94856 is called for the scratch descriptor with that requested capacity.
3. A null result from this checked capacity operation causes context exit and status 0.
4. Working-result capacity is checked against the same requested capacity.
5. If insufficient, F-08bc2fc94856 is called for the working descriptor.
6. A null result causes context exit and status 0.
7. F-24e91ba66067 receives the working storage address, both operand storage addresses, the effective common count, and the scratch storage address.
8. After that call, the working active count is set to the original operand-count sum.

The scratch acquisition itself is not checked for null before scratch capacity is read on this selected branch. A null scratch descriptor can therefore cause an invalid access rather than a handled failure.

Both specialized capacity comparisons are signed 32-bit comparisons against the computed four-times-candidate value. For ordinary positive, nonoverflowing values, this represents four times the candidate in words, or sixteen times the candidate in bytes. Overflow behavior is not established.

As with working capacity handling, a capacity helper’s returned pointer does not replace the stored scratch descriptor pointer.

### General computation

The general path applies whenever neither specialized computation is selected.

The working capacity is compared, as a signed value, with the original operand-count sum. If insufficient, F-08bc2fc94856 receives the working descriptor and that sum. Null causes context exit and status 0.

After successful capacity handling:

- The working active count is set to the original sum.
- F-8eef14d02549 receives the working storage address, first operand storage address and effective count, and second operand storage address and effective count.
- Execution continues to normalization.

The effective counts can include a padding increment from an unsuccessful large-path attempt. Consequently, the dependency can receive padded counts while the caller’s nominal destination count and capacity request remain based on the original sum.

This does not establish that the dependency writes beyond the nominal result. Its written range and padded-count requirements remain unknown.

### Normalization, conditional transfer, and exit

Normalization reads the working descriptor’s **then-current** active count. It does not assume that a preceding computational or capacity dependency preserved a previously assigned count.

If that count is positive, scanning begins at the highest active word. Each observed zero causes the active count to decrease by one. Scanning stops at the first nonzero word or when the count reaches zero. If the initial count is not positive, the scan is skipped.

Normalization directly changes only the active count. It does not directly clear storage, reduce capacity, or write the sign-like field. In particular, an all-zero result does not cause a sign-clearing write.

After normalization, the function compares the destination and working descriptor pointers:

- If they differ, F-cec6695e396c receives the destination and working descriptor.
- If they are equal, no transfer call occurs.

This condition applies regardless of whether the working descriptor was acquired earlier. A successful acquisition alone does not imply a transfer.

The transfer result, if any, is ignored. The function then selects status 1, calls F-4bd8a1faf697 with the context token, and returns 1. No final validation of the destination follows transfer or context exit.

## Side effects

Established direct writes include:

- Assigning the working sign-like field.
- Writing a zero padding word into a qualifying operand’s storage.
- Assigning the working active count before fixed-size or general computation.
- Assigning the original sum after large specialized computation.
- Reducing the active count during normalization.

Failure is not transactional. Earlier writes and completed dependency effects have no explicit rollback. For example, a checked capacity failure may follow sign assignment, operand padding, or a successful earlier capacity operation. The eventual visibility of those changes also depends on later dependency effects, including context exit.

Read timing matters. Operand counts are captured before context entry. Sign fields, capacities, flags, and storage addresses are read later when needed. A dependency that changes a descriptor can therefore affect later reads without updating the already captured counts.

There is no direct storage-release operation in the body. Descriptor acquisition and context exit do not establish allocation, ownership, release, or lifetime rules. Capacity operations may affect storage and descriptors, but their exact effects are unavailable.

No dependency is assumed to preserve globals, descriptors, storage, or ownership. The contract distinguishes this function’s direct actions from any final-state guarantee that would require such preservation.

## Dependencies

| Opaque ID | Observed interaction | Unestablished contract |
|---|---|---|
| F-d4e7c5a40239 | Destination and zero; nonzero return selects immediate success | Zero representation, writes, allocation, ownership |
| F-ea92bf179106 | Context token before nonzero-path working selection | Context effects, failure signaling, preservation |
| F-4bd8a1faf697 | Context token on common exit and failed zero shortcut | Release, rollback, descriptor or global changes |
| F-834e12b6dfde | Context token; result used as working or scratch descriptor | Freshness, distinctness, initialization, lifetime, nullability |
| F-08bc2fc94856 | Descriptor and requested word capacity; null checked | Storage policy, preservation, ownership, exact mutation |
| F-4701769f3e09 | Three storage addresses for the eight-word case | Arithmetic, overlap rules, written range, other effects |
| F-f7e02acefeaa | Effective common count; result determines candidate shift | Meaning, width interpretation, range guarantees |
| F-24e91ba66067 | Result and operand storage, common count, scratch storage | Arithmetic, scratch requirements, written ranges, preservation |
| F-8eef14d02549 | Result storage and both operand address/count pairs | Arithmetic, padded-count handling, overlap, written ranges |
| F-cec6695e396c | Destination and distinct working descriptor | Copy versus exchange, failure behavior, ownership, final contents |

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. They assume valid accesses and dependency behavior sufficient to reach the stated observation.

1. **Successful zero shortcut.** If either captured count is zero and F-d4e7c5a40239 returns nonzero, status is 1. Neither context-management dependency is called by this function.

2. **Failed zero shortcut.** If the same dependency returns zero, F-4bd8a1faf697 is called and status is 0. No preceding context-entry call occurs on this path.

3. **Eight-word operands.** With captured counts 8 and 8, and successful capacity handling, the working active count is assigned 16 before F-4701769f3e09 receives the three storage addresses.

4. **High-word trimming.** If normalization begins with active count 6, words at indices 5 and 4 are zero, and word 3 is nonzero, normalization leaves count 4. It directly changes neither storage nor capacity.

5. **All-zero working result.** If normalization begins with a positive count and every scanned word is zero, it reduces the count to zero. Normalization itself leaves the sign-like field unchanged. Retention of the earlier XOR through computation, transfer, and context exit additionally requires those dependencies to preserve or propagate it; that preservation is not established.

6. **Exact destination alias.** If the destination equals an operand descriptor, F-834e12b6dfde is called. A nonnull return becomes the working descriptor. If execution reaches the final comparison and that descriptor differs from the destination, F-cec6695e396c is called. If it equals the destination, transfer is skipped.

7. **Allowed padding.** With captured counts 17 and 16, the second operand lacking flag mask 2, and its capacity greater than 16 when checked, word index 16 is written as zero. Its descriptor count is not directly changed; its effective count becomes 17.

8. **Blocked padding.** With the same counts but mask value 2 set on the shorter operand when checked, no padding write occurs and the general path is selected.

9. **Candidate mismatch.** If equalized counts differ from the derived candidate, scratch acquisition still occurs before general-path capacity handling and computation.

10. **Checked capacity failure.** If a checked capacity operation returns null, context exit occurs and status is 0. Normalization and transfer are not reached. Earlier writes have no explicit rollback.

11. **Selected branch with null scratch.** If the candidate matches and scratch acquisition returned null, scratch-capacity access precedes any null check of a capacity result. A clean status-0 outcome is not guaranteed.

## Uncertainties

The following remain **documented_only**:

- The exact arithmetic operation, numeric representation, and computational written ranges.
- The meaning and valid result range of F-f7e02acefeaa.
- Shift behavior when the derived amount exceeds 31.
- Overflow behavior in count sums, differences, capacity products, and address calculations.
- Whether acquired descriptors are fresh, distinct, initialized, or nonoverlapping.
- Scratch nonnull guarantees.
- Behavior with distinct descriptors sharing storage.
- Preservation of sign-like fields, active counts, globals, and storage across dependencies.
- Final destination contents after transfer and context exit.
- Allocation, ownership, lifetime, release, and transfer semantics.
- Failures not represented by an explicitly checked return value.

No invented recovery behavior is assigned to invalid pointers, inconsistent capacities, negative counts, or unchecked dependency failures.

## Evidence

This contract uses static inspection of the supplied body and opaque dependency mapping. Established observations include field offsets and widths, signed comparisons, direct writes, call arguments, checked outcomes, and control-flow ordering.

Multiplication intent, numeric sign meaning, and word significance are inferences. Dependency preservation, ownership, freshness, and arithmetic guarantees are not established by their apparent roles or names.

No runtime tests have been performed.
