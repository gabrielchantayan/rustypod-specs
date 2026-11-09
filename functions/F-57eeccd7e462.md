<!-- Trusted provenance: F-57eeccd7e462; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-57eeccd7e462

## Purpose

F-57eeccd7e462 updates derived values in a shared record containing 256 indexed entries, then checks active entries against record-wide results. A detected mismatch writes the 32-bit value 14 to a status field in an enclosing context.

The established behavior consists of:

- Finding two maxima, each with a minimum value of 1.
- Filling zero-valued cache fields for active entries while preserving existing nonzero values.
- Computing two record-wide results through F-870c4fcf7295.
- Applying a signed division-by-eight adjustment when the record’s mode is not 3.
- Checking active entries in ascending index order, with conditional early termination.

The data suggests normalization or size-consistency processing, but the application-level meaning of the quantities is not established. In particular, F-870c4fcf7295 must not be assumed to perform division merely because its arguments resemble a numerator and denominator.

## Interface

### Invocation and result

The recovered interface has no explicit parameters and no return value.

Inputs and outputs are accessed through shared memory:

1. A shared root reference supplies an enclosing context address.
2. A pointer at context byte offset `0x24` supplies the working record address.
3. Context byte offset `0x34` holds a 32-bit status field.

The context address and working-record pointer are captured at entry. Later changes to the shared root reference or to the context’s record-pointer field do not cause this function to select a different context or record.

### Record layout

The supplied representation uses 32-bit integer fields and four-byte indexed strides. Offsets below are relative to the captured working-record address.

| Byte offset | Field | Role |
|---|---|---|
| `0x0000` | Mode | Value 3 bypasses division-by-eight adjustments; every other value enables them. |
| `0x0010` | Scale B | Input to second-cache and record-wide B calculations. |
| `0x0014` | Scale A | Input to first-cache and record-wide A calculations. |
| `0x0420 + 4 × index` | A entry | Activity predicate, first maximum input, and first-cache calculation input. |
| `0x0820 + 4 × index` | B entry | Second maximum input and second-cache calculation or validation input. |
| `0x1020 + 4 × index` | A cache | Conditionally initialized per-entry result. |
| `0x1420 + 4 × index` | B cache | Conditionally initialized per-entry result. |
| `0x1824` | B maximum output | Receives the scanned B maximum. |
| `0x1828` | A maximum output | Receives the scanned A maximum. |
| `0x182C` | A result | Record-wide derived result, subsequently checked against active entries. |
| `0x1830` | B result | Record-wide derived result; zero disables B validation. |

Each array has exactly 256 entries, indexed 0 through 255. Unlisted portions of the record have no established meaning here.

### Preconditions and ownership

Successful execution requires a valid shared root reference, a readable context record-pointer field, and accessible storage for every field reached by execution. The function contains no visible null, extent, or alignment checks.

The function itself performs no explicit allocation, release, ownership transfer, or pointer replacement. Ownership and lifetime behavior inside F-870c4fcf7295 are unknown. In particular, this contract does not assume that the dependency preserves the working record, context, or unrelated shared state.

## Behavior

### 1. Scan both maxima

The function initializes two local maxima to 1 and visits all 256 entries in ascending index order.

For each index:

1. It compares the B entry against the current B maximum using signed comparison.
2. If the B entry is greater, it updates the B maximum from that entry.
3. It compares the A entry against the current A maximum using signed comparison.
4. If the A entry is greater, it updates the A maximum from that entry.

Both arrays participate regardless of whether an entry is active. A B entry whose corresponding A entry is zero can therefore determine the B maximum.

For stable input memory, the resulting maxima are the maximum of 1 and the respective 256 signed entry values. Zero and negative values do not lower either maximum.

The supplied representation shows a comparison read followed by a further entry read when a maximum is updated. Whether these correspond to separate externally observable machine reads is not established without lower-level evidence.

### 2. Initialize missing per-entry caches

A second ascending traversal processes all 256 indices.

An entry is active when its A entry is nonzero. Negative A entries are active. A B entry alone does not make an entry active.

For each active entry:

- If its A cache is zero, F-870c4fcf7295 is invoked with two recovered arguments:
  - Scale A multiplied by the current A entry, minus 1.
  - The A maximum captured by the first pass.
  
  The returned value plus 1 is stored in the A cache.

- Next, if its B cache is zero, F-870c4fcf7295 is invoked with:
  - The current B entry multiplied by Scale B, minus 1.
  - The B maximum captured by the first pass.
  
  The returned value plus 1 is stored in the B cache.

A nonzero cache is preserved by this function during this pass, including a negative cache value. A zero cache on an inactive entry is also preserved.

The A-cache operation precedes the B-cache zero check. A dependency side effect during the first operation can therefore affect the second check and its argument reads. The active-entry predicate is not rechecked between these operations.

There is no retry if a newly stored cache value is zero.

### 3. Publish maxima and compute record-wide results

After cache initialization, the function writes:

1. The captured A maximum to `0x1828`.
2. The captured B maximum to `0x1824`.

It then calls F-870c4fcf7295 with Scale A minus 1 and the captured A maximum. The return value plus 1 is stored as the A result. This call occurs unconditionally, including when Scale A is zero.

The B result is computed next:

- If the currently read Scale B is zero, the function stores zero as the B result without making this B-result dependency call.
- Otherwise, it calls F-870c4fcf7295 with Scale B minus 1 and the captured B maximum, then stores the return value plus 1.

Thus, zero Scale B has a special effect on the record-wide B calculation. It does not suppress earlier B-cache initialization for active entries.

### 4. Apply the mode-dependent adjustment

After both record-wide results have been written, the function reads the mode.

If the mode is 3, it performs no adjustment at this stage.

Otherwise:

- The A result becomes the signed quotient, truncated toward zero, of the raw return from the record-wide A dependency call divided by 8, plus 1.
- If the current stored B result is nonzero, the B result becomes the signed quotient, truncated toward zero, of that stored value minus 1 divided by 8, plus 1.
- If the current stored B result is zero, it remains zero.

The A adjustment uses the locally retained dependency return, not a fresh read of the stored A result. The B adjustment uses the stored B result. This distinction matters if a dependency has modified record fields.

For a signed 32-bit value within a range where the surrounding arithmetic does not overflow, division here truncates toward zero. For example, division of −1 through −7 by 8 produces zero, not −1.

### 5. Validate active entries

A third ascending traversal checks active entries. The activity predicate is read again; it is not a snapshot from either earlier pass.

For each active entry, the function performs the following sequence.

**A check**

It calls F-870c4fcf7295 with the A cache minus 1 as the single recovered explicit argument. The supplied representation does not establish any second argument for this call.

After the call, it reads the mode. If the mode is not 3, it divides the returned value by 8 with signed truncation toward zero. It adds 1 and compares the result against the currently stored A result.

If they differ, it writes 14 to the context status field. This mismatch does not immediately return.

**Conditional B check**

It reads and locally retains the current B result. If that value is zero, it skips B validation.

If the retained B result is nonzero, it calls F-870c4fcf7295 with:

- The B cache minus 1.
- The current B entry.

The function then reads the mode again and applies signed division by 8 when that mode is not 3. It adds 1 and compares against the B result retained before the dependency call.

A mismatch writes 14 to the context status field and returns immediately.

**Status check**

If execution has not already returned, it reads the context status field. Any nonzero value causes return. Otherwise, processing continues to the next index.

Inactive entries do not trigger either validation call or this final status check.

### Arithmetic boundaries

The visible computations use 32-bit integer quantities, including entry products, subtraction of 1, helper returns, addition of 1, and stored results. Maxima and mode comparisons are signed.

Products are not visibly widened. No overflow checks, range checks, saturation, or arithmetic-error reporting are present. For inputs that overflow signed products or surrounding additions and subtractions, an exact portable arithmetic result is not established by the supplied representation.

The division-by-eight transformation is established for signed 32-bit inputs without the surrounding arithmetic overflowing. Its negative-value correction distinguishes truncation toward zero from an uncorrected arithmetic right shift.

## Side effects

The function directly writes only:

- A zero A cache or B cache belonging to an entry active during cache initialization.
- The two maximum-output fields.
- The two record-wide result fields.
- The context status field when a validation comparison fails.

Intermediate writes are observable in order: cache initialization completes before maximum outputs are published; both maximum outputs are published before the record-wide A call; unadjusted result fields are written before any mode-dependent replacements.

The function does not clear status on entry or on success. It can overwrite an existing status value with 14 when a mismatch is detected.

A preexisting nonzero status does not prevent the maxima scan, cache initialization, or record-wide computations. During validation, it normally causes return after the first active entry’s checks, unless dependency side effects change status before that read. If every entry is inactive, the function reaches the end without reading status in the validation phase.

An A mismatch can be followed by a B dependency call before return. Consequently, writing 14 for an A mismatch does not establish that no further dependency side effects occur.

No atomicity, locking, rollback, or transactional behavior is visible.

## Dependencies

### F-870c4fcf7295

This is the sole explicit callable dependency.

It is used for:

1. Missing A-cache initialization.
2. Missing B-cache initialization.
3. Record-wide A-result calculation.
4. Conditional record-wide B-result calculation.
5. A validation for each reached active entry.
6. Conditional B validation for each reached active entry.

Its return is treated as a 32-bit integer. Calls other than A validation show two arguments in the supplied representation. A validation shows only one.

The dependency’s mathematical operation, valid argument ranges, actual calling convention, side effects, failure behavior, and memory-lifetime effects are undocumented here. In particular:

- A two-argument invocation is not proof of division semantics.
- A zero or negative second argument during B validation has no established meaning.
- The missing second argument at A validation must not be filled in by assumption.
- Shared state must be considered subject to dependency modification unless separately constrained.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Except where noted, they assume stable input storage, no dependency side effects, valid memory, and arithmetic without overflow.

### All entries inactive

With all A entries zero:

- Neither per-entry cache array is written by this function.
- The A maximum output is 1.
- The B maximum output still reflects all B entries, with a minimum of 1.
- The record-wide A dependency call still occurs.
- The record-wide B dependency call occurs only if Scale B is nonzero.
- No validation dependency calls occur.
- An existing nonzero status is not cleared and does not cause an earlier return.

### An inactive entry controls the B maximum

If every B entry is at most 1 except index 200, whose B entry is 12, the B maximum is 12 even when index 200 has A entry zero. That entry’s caches remain untouched during initialization.

### Preserve a nonzero cache

For an active entry with A cache −4 and B cache zero, initialization preserves the A cache and makes only the B-cache call for that entry. The negative A cache does not prevent later A validation.

### No retry after a zero result

For an active entry with A cache zero, suppose its initialization call returns −1. The function stores zero in that cache and proceeds. It does not call again merely because the cache remains zero.

### Mode-sensitive A result

Suppose the record-wide A call returns 15:

- Mode 3 produces A result 16.
- A mode other than 3 produces A result 2.

If a reached A-validation call also returns 15 and the mode remains unchanged, its computed comparison value matches that A result.

If the raw return is −7 instead, mode 3 produces −6, whereas another mode produces 1. This distinguishes truncation toward zero from rounding toward negative infinity.

### A mismatch does not immediately stop B processing

Suppose an active entry’s A comparison fails, its B result is nonzero, and dependencies do not alter status. The function writes status 14, still performs that entry’s B-validation call, and then returns either on a B mismatch or on the subsequent nonzero-status check. Later indices are not checked.

### B mismatch uses the retained record-wide value

If B validation begins with B result 5, that comparison target is retained across the dependency call. Even if the dependency changes the stored B result, the current comparison remains against 5.

## Uncertainties

The following remain **documented_only**:

- The application meaning of the record, entries, caches, scales, mode, and status value 14.
- The implementation and full interface of F-870c4fcf7295.
- The correct interpretation of the one-argument A-validation call.
- Exact overflow behavior outside non-overflowing signed arithmetic.
- Pointer validity, backing allocation sizes, aliasing relationships, and memory ownership.
- Dependency effects on shared state and storage lifetime.
- Concurrent mutation semantics and which displayed repeated reads survive in the underlying machine instructions.
- Whether invalid dependency inputs cause a return, exception, trap, nontermination, or another outcome.

No runtime tests have been performed.

## Evidence

Established static evidence comes solely from the supplied representation: three ascending 256-entry traversals, signed maxima initialized to 1, conditional cache writes, explicit dependency-call arguments where shown, result-field write order, mode comparisons against 3, signed division-by-eight transformations, status writes of 14, and the visible return conditions.

The field names in this contract describe operational roles rather than original names or established domain concepts. Normalization and consistency checking are inferred intent; the observable calculations, state changes, and control-flow conditions above are the behavioral contract supported by the supplied evidence.
