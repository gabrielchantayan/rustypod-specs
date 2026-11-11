<!-- Trusted provenance: F-0ee6114cab5e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ee6114cab5e

## Purpose

F-0ee6114cab5e implements a 32-byte-aligned allocation and resize interface with allocation metadata, usage accounting, a high-water mark, and conditional recovery calls.

**Established behavior:** a zero input address selects allocation; a nonzero address with a positive signed size selects resize. Successful allocation or resize returns an aligned address within an underlying allocation. Certain failure or refusal paths return zero.

**Inferred intent:** the accounting fields represent current requested bytes, peak requested bytes, and a threshold that can trigger recovery. The function names and signatures of its dependencies are not sufficient to establish their actual allocation, recovery, or ownership guarantees.

The nonzero-address, nonpositive-size path calls a separate dependency and returns zero. It must not be described as freeing the supplied address without additional evidence.

## Interface

### Inputs and result

| Item | Width | Contract |
|---|---:|---|
| Input address | 32 bits | Zero, or an address whose surrounding metadata satisfies the layout below. |
| Requested size | 32 bits | Interpreted as signed for branch selection, despite its unsigned representation. |
| Result | 32 bits | Zero on the visible failure/refusal paths; otherwise a 32-byte-aligned address. |

Only requested-size bit patterns from `0x00000001` through `0x7fffffff` enter allocation or resize processing. Zero and all values with bit 31 set take the nonpositive-size paths.

Address arithmetic and ordinary size arithmetic are 32-bit operations. Arithmetic results wrap modulo \(2^{32}\). The accounting calculations explicitly combine 32-bit words into wider values.

### Allocation layout

Let:

- \(B\) be the underlying allocation address returned by a storage dependency.
- \(N\) be the requested size.
- \(A\) be the address returned to the caller.
- \(D=A-(B+8)\) be the displacement recorded for later recovery.

The wrapper requests \(N+44\) bytes and establishes:

| Location | Width | Contents |
|---|---:|---|
| \(B\) | 32 bits | Requested size \(N\). |
| \(B+4\) | 32 bits | Sign extension of \(N\); zero on every positive-size path. |
| \(A-4\) | 32 bits | Displacement \(D\). |
| \(A\) | Caller-visible region | Beginning of the aligned payload. |

The returned address is obtained by rounding \(B+44\) downward to a multiple of 32. This establishes 32-byte alignment, not necessarily zero-filled storage.

Ignoring address wraparound, \(A-B\) is between 13 and 44 bytes, and \(D\) is between 5 and 36 bytes. If the underlying allocation is four-byte aligned, the possible displacements are multiples of four from 8 through 36.

A nonzero input address is interpreted using this same layout:

1. Read the displacement at input address minus four.
2. Subtract it from the input address to recover \(B+8\).
3. Read the old requested size at \(B\).
4. Pass \(B\), rather than the caller-visible address, to the resize dependency.

The old size’s companion word at \(B+4\) is not read by this function.

### Preconditions

For defined, meaningful operation:

- A nonzero input address must identify a compatible allocation with readable displacement and size metadata.
- Metadata accesses must satisfy the platform’s address and alignment requirements.
- Successful storage dependency results must provide writable memory sufficient for the wrapper’s metadata accesses.
- The state pointers used for accounting and policy must identify accessible records.
- Address calculations must not wrap into unrelated storage.

These conditions are not checked locally. There is no visible validation of pointer provenance, displacement, old size, or state-record accessibility.

## Behavior

### Accounting model

Three paired-word fields are used in accounting records:

| Record offset | Interpretation |
|---|---|
| `+0x20`, `+0x24` | Threshold, low word then high word. |
| `+0x38`, `+0x3c` | Current usage, low word then high word. |
| `+0x40`, `+0x44` | Peak usage, low word then high word. |

Comparisons treat each pair as a signed 64-bit value. Low-word arithmetic is unsigned, with explicit carry or borrow propagated into the high word. Wider arithmetic wraps modulo \(2^{64}\); there is no visible saturation.

At entry, the function captures one accounting-record pointer. The fresh-allocation path later replaces its local accounting target with a pointer obtained from a separate global pointer location after its preparation call. The resize path retains the entry-captured pointer for final accounting, but obtains the record used for its threshold check through a global pointer location.

Consequently, threshold checking and successful accounting are not guaranteed to use the same record if dependencies or other execution contexts change global pointers.

### Zero address and nonpositive size

The function returns zero.

It makes no dependency calls and performs no allocation metadata or accounting-field writes on this path. The entry accounting-pointer capture still occurs.

### Zero address and positive size

The order is:

1. Call F-54383d765e10 without explicit arguments.
2. Capture the fresh-allocation accounting target.
3. Read a separate policy record’s 32-bit field at offset `+8`.
4. If that field is nonzero, evaluate the threshold condition.
5. Call F-19123db18ab4 with zero.
6. If its result is nonzero, return zero.
7. Otherwise, attempt storage acquisition.

The threshold condition compares current usage plus the positive requested size with the threshold, using signed 64-bit comparison. If the projected usage is **greater than or equal to** the threshold, call F-570f57d734a2 with the requested size.

The policy field controls only this fresh-allocation threshold check. It does not suppress the later gate call or an allocation-failure recovery call.

Storage acquisition calls F-849ccca8c593 with \(N+44\). If it returns zero:

1. Call F-570f57d734a2 with \(N\).
2. Retry F-849ccca8c593 once with the same size.
3. Return zero if the second attempt also returns zero.

There is no third attempt and no second gate check.

After a nonzero storage result, the wrapper:

1. Writes the size’s high word at \(B+4\), which is zero.
2. Computes the aligned payload address.
3. Writes \(N\) at \(B\).
4. Writes the displacement at \(A-4\).
5. Reads the captured accounting target’s current usage and adds \(N\).
6. Writes current usage, low word first and high word second.
7. Compares the updated current usage with peak usage.
8. If current usage is strictly greater, copies the current-usage words into the peak fields, low word first.
9. Returns \(A\).

The wrapper does not initialize the payload bytes.

### Nonzero address and nonpositive size

The function calls F-94cb53877c15 without explicit arguments, then returns zero.

It does not visibly read the supplied address’s displacement or size metadata, pass the supplied address as an explicit argument, or update accounting fields on this path. Any release or state change performed by the dependency is unknown.

### Nonzero address and positive size

The wrapper first reads and retains the old displacement and old size. It then computes projected usage for the threshold check.

If \(C\) is current usage, \(N\) is the new requested size, and \(O\) is the old size word, the projected value is:

\[
C+\operatorname{signextend}_{32}(N)
-\operatorname{signextend}_{32}(O)
\pmod {2^{64}}.
\]

For compatible allocations produced through positive-size paths, both sizes are positive signed 32-bit values. The implementation nevertheless sign-extends the old size word; it does not read a stored 64-bit old size.

If projected usage is greater than or equal to the threshold, call F-570f57d734a2 with:

\[
N-O \pmod {2^{32}}.
\]

Unlike fresh allocation, this threshold check is not conditioned on the policy field.

Next, call F-19123db18ab4 with zero. A nonzero result causes an immediate zero return, with no storage-resize call or local metadata/accounting write.

Otherwise, call F-41c5eabb7406 with the recovered underlying allocation address and \(N+44\). If it returns zero:

1. Call F-570f57d734a2 with the full new size \(N\), not the size difference.
2. Retry F-41c5eabb7406 once with the same underlying address and requested storage size.
3. Return zero if that attempt also returns zero.

The same numeric underlying address is supplied to both attempts. Whether the first failed attempt or intervening recovery preserves that allocation is a dependency contract, not a local guarantee.

### Successful resize and payload adjustment

For a nonzero resize result \(B_{\text{new}}\), the wrapper:

1. Writes zero at \(B_{\text{new}}+4\).
2. Computes the new aligned address \(A_{\text{new}}\) and displacement \(D_{\text{new}}\).
3. Writes the new requested size at \(B_{\text{new}}\).
4. Calls the unmapped transfer dependency D-01.
5. Writes \(D_{\text{new}}\) at \(A_{\text{new}}-4\).
6. Updates current usage and, conditionally, peak usage.
7. Returns \(A_{\text{new}}\).

D-01 receives these three arguments, in order:

| Argument | Value |
|---|---|
| First | \(A_{\text{new}}\) |
| Second | \(A_{\text{new}}+D_{\text{old}}-D_{\text{new}}\), equivalently \(B_{\text{new}}+8+D_{\text{old}}\) |
| Third | Old requested size \(O\) |

**Inferred intent:** relocate the old payload within the resized allocation when alignment displacement changes.

**Established limitation:** the count is always the old requested size. It is not limited to the smaller of the old and new sizes. On shrink, the call can therefore name regions extending beyond the newly requested backing extent. Source and destination may also overlap or be identical. D-01’s actual read/write behavior and overlap guarantees are not supplied.

The new displacement word is written only after D-01 returns. The new size header is written before that call.

### Successful-resize accounting

Final accounting uses a signed, wrapped 32-bit difference:

\[
\Delta=\operatorname{signextend}_{32}
\left((N-O)\bmod 2^{32}\right).
\]

The wrapper adds \(\Delta\) to the current usage read after D-01 returns, writes the low word and then the high word, and raises peak usage only if updated current usage is strictly greater under signed 64-bit comparison.

This differs from separately sign-extending the two operands and subtracting them when arbitrary size metadata is allowed. For valid positive old and new sizes, the difference lies within signed 32-bit range, so the two interpretations agree.

No accounting value is preserved across D-01 for later restoration. If dependencies alter accounting fields, the final update uses the values observed after those calls.

## Side effects

### Directly established effects

- Reads global state-pointer locations, policy state, and accounting fields.
- Reads old displacement and size metadata on positive-size resize.
- Writes allocation headers and displacement metadata after storage success.
- Updates current usage and conditionally updates peak usage.
- Makes dependency calls in the branch-specific order described above.

Accounting pairs are written as separate 32-bit words, not as an atomic 64-bit update. No local locking, atomic operation, rollback, or synchronization mechanism is visible.

### Ownership and errors

The caller receives an interior aligned address, not the storage dependency’s underlying address. Any compatible later operation must recover the underlying address through the displacement metadata.

A zero result can mean:

- A nonpositive-size branch.
- A nonzero gate result.
- Two failed storage attempts.

No separate status code or local diagnostic is produced. Dependency-side error reporting is unknown.

A zero result does not imply absence of side effects: preparation, threshold recovery, gate evaluation, storage attempts, or the nonpositive-size dependency may already have run.

On resize failure, continued ownership and validity of the original allocation cannot be established without F-41c5eabb7406 and F-570f57d734a2 contracts.

## Dependencies

| Opaque ID | Established use | Unknown contract |
|---|---|---|
| F-54383d765e10 | Called first on positive-size fresh allocation; no explicit arguments. | Preparation effects, synchronization, and global changes. |
| F-19123db18ab4 | Called with zero before storage acquisition or resize; zero permits continuation. | Meaning of refusal and side effects. |
| F-849ccca8c593 | Receives requested size plus 44; zero causes recovery and one retry. | Allocation ownership, initialization, alignment, and failure effects. |
| F-41c5eabb7406 | Receives recovered underlying address and new size plus 44. | Preservation, movement, release, and failure semantics. |
| F-570f57d734a2 | Called for threshold conditions and first-attempt storage failure. | Whether it reclaims memory, changes globals, invalidates allocations, or can fail to return. |
| F-94cb53877c15 | Called without explicit arguments for nonzero address and nonpositive size. | Whether it releases anything and how it identifies a target. |
| D-01 | Unmapped dependency receiving two addresses and the old byte count. | Actual transfer direction, overlap safety, bounds requirements, and ownership effects. |

D-01 is a document-local opaque identifier; no supplied published function ID maps to this dependency.

## Acceptance examples

These are conditional contract examples, not executed tests.

1. **Zero-size fresh request.** With address zero and requested size zero, expect result zero, no dependency calls, and no local accounting writes.

2. **High-bit size classification.** With address zero and size `0x80000000`, expect the nonpositive fresh path, not a large allocation attempt. With a nonzero address and the same size, expect F-94cb53877c15 followed by zero.

3. **Threshold equality.** Given enabled fresh-allocation policy, current usage 100, threshold 120, and request 20, expect F-570f57d734a2 with 20 before the gate call. Equality triggers recovery.

4. **Disabled fresh policy.** With that policy field zero, expect no threshold-triggered recovery on fresh allocation. A failed first storage attempt still triggers recovery.

5. **Fresh-allocation retry.** For request 64, with a permitting gate, first storage result zero, and second result nonzero, expect storage arguments of 108 on both attempts and an intervening recovery argument of 64.

6. **Alignment and accounting.** If a successful fresh allocation of 64 returns underlying address `0x1000`, expect payload address `0x1020`, displacement 24 at `0x101c`, size 64 at `0x1000`, and zero at `0x1004`. If current usage was 100 and peak was 150 and dependencies leave them unchanged, expect both to become 164.

7. **Shrink accounting.** With valid old size 100, new size 40, current usage 200, and peak 300, successful resize with no dependency accounting changes produces current usage 140 and leaves peak 300. D-01 still receives count 100.

8. **Displacement change.** If old displacement is 24 and new underlying address is `0x1010`, the new aligned address is `0x1020` and new displacement is 8. D-01 receives first address `0x1020`, second address `0x1030`, and the old size.

9. **Gate refusal after threshold recovery.** If projected usage reaches the threshold and the gate subsequently returns nonzero, expect threshold recovery followed by the gate and result zero, without a storage call.

10. **Resize retry arguments.** For old size 100 and new size 160, threshold recovery, if triggered, receives 60. If the first resize attempt fails, its recovery call receives 160 instead.

## Uncertainties

The following are **documented_only**:

- Exact dependency semantics, including memory preservation, release, transfer behavior, and nonlocal exits.
- Whether state-pointer locations always refer to the same record.
- Whether dependencies mutate globals, accounting, policy, or allocation metadata.
- Thread-safety and synchronization outside this function.
- Behavior for corrupted metadata, invalid addresses, inaccessible records, or address wraparound.
- Safety of shrinking while passing the old size to D-01.
- Whether nonpositive-size handling is intended as a release operation.
- Whether signed accounting values have meaningful negative states.

No runtime tests have been performed. The contract does not assume that unknown callees preserve globals or retain, acquire, or release ownership.

## Evidence

This contract is based solely on the supplied static function body and dependency-ID mapping.

Established evidence covers branch conditions, explicit call arguments, retry count, metadata locations, alignment arithmetic, accounting widths, signed comparisons, and visible read/write ordering.

Allocator, recovery, release, and transfer interpretations are identified as inferred intent or unknown dependency behavior. No implementation execution, external dependency inspection, or runtime validation supports stronger claims.
