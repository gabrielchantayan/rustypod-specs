<!-- Trusted provenance: F-df2c7f3bb9d4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df2c7f3bb9d4

## Purpose

F-df2c7f3bb9d4 conditionally removes a four-byte value from a selected destination region. A field in a separately selected source region supplies a one-based position. When accepted, later values move toward the beginning, the final visited position is cleared, and the destination count is decremented.

Two byte-valued gates control progress. Normal return values distinguish a disabled gate, an empty destination, a rejected position, and successful completion.

“Removal,” “entry,” and “count” describe inferred collection semantics. The supplied static representation establishes the accesses, widths, comparisons, and writes described below. No runtime tests have been performed.

## Interface

### Arguments

| Position | Width | Established role |
|---|---:|---|
| First | 32 bits | Selector. Exactly `1` selects regions displaced by 72 bytes. Every other value selects undisplaced regions. |
| Second | 32 bits | Its incoming value is never read by the represented body. |
| Third | 32 bits | Its incoming value is never read by the represented body. |
| Fourth | 32 bits | Its incoming value is never read by the represented body. On the nonnegative-candidate path, a destination limit replaces it before any value read. On the negative-candidate path, short-circuit evaluation prevents the expressions that could otherwise read it. |

The fourth argument supplies neither a position nor a limit. Its incoming value does not participate in subtraction, subtraction-overflow evaluation, acceptance, or the result. The local replacement of that argument is not a write to caller-owned storage.

### Result

The function returns a 32-bit value.

| Value | Meaning on a normal return |
|---|---|
| `0` | The accepted mutation path completed. |
| `1` | A gate byte was zero. |
| `5` | Both gates passed and the initial destination count was zero. |
| `6` | The initial destination count was nonzero, but the candidate position was rejected. |

Invalid accesses and arithmetic outside the established interpretation are not visibly converted into these return values.

### Global holders and selected storage

The function uses two global address holders to locate gate bytes, one global source-base holder, and one global destination-base holder.

Evaluating a holder and accessing memory through its value are distinct operations. The source and destination holders are evaluated separately; neither equal addresses nor disjoint storage is established.

For selector `1`, each base holder is evaluated twice. The selected region uses the second value plus 72 bytes. For every other selector, each base holder is evaluated once and used without displacement.

These are evaluation distinctions in the supplied representation. They do not independently establish volatile access, synchronization, or the exact physical loads produced by compilation.

### Selected-region layout

The selected source region supplies a signed 32-bit position at byte offset **64**.

The selected destination region has the following observed accesses:

| Byte offset | Width | Role |
|---|---:|---|
| Four times a zero-based entry position | 4 bytes | Shift source, shift destination, or trailing clear |
| `64` | 4 bytes | Signed destination count |
| `68` | 4 bytes | Signed position limit |

Sixteen four-byte entry positions precede the count field. The function does not enforce a sixteen-entry capacity.

Address calculations appear as 32-bit integer expressions. Physical pointer width, byte order, alignment requirements, and overflowing-address behavior are not independently established.

## Behavior

### First gate

The function evaluates the first gate’s global address reference and reads the byte reached through it.

If the byte is zero, the function returns `1`. It does not subsequently evaluate the source-base holder, read the source position, access the second gate, or access the destination.

Any nonzero byte permits progress. Locating the gate byte still requires evaluating its address reference; the byte read is not necessarily the first memory operation needed to perform the check.

### Source selection and candidate formation

After the first gate passes:

1. The source-base holder is evaluated.
2. If the selector equals `1`, the holder is evaluated again, and 72 bytes are added to that second value.
3. Otherwise, the first value is used without displacement.
4. The signed 32-bit source-position field at selected source offset 64 is read.
5. One is subtracted to form the zero-based **candidate position**.

For selector `1`, the first base evaluation occurs even though its value does not determine the selected source address.

The source-position field is read once. Later writes that alias that field do not cause another source-position read.

Subtracting one from the minimum signed 32-bit value is outside the representable signed range. Its execution semantics remain documented-only under **Uncertainties**.

### Second gate

After reading the source position and forming the candidate, the function evaluates the second gate’s global address reference and reads the reached byte.

If it is zero, the function returns `1`. No destination-base evaluation or destination-count read follows.

Consequently, a disabled second gate does not protect against an inaccessible source holder or source-position field: those accesses precede the second gate.

Any nonzero second-gate byte permits progress.

### Destination selection and empty check

After the second gate passes:

1. The destination-base holder is evaluated.
2. If the selector equals `1`, the holder is evaluated again, and 72 bytes are added to that second value.
3. Otherwise, the first value is used without displacement.
4. The signed 32-bit destination count at selected destination offset 64 is read.

The selected destination address is retained. The later limit read, entry accesses, repeated count reads, and count write use that address rather than reevaluating the destination-base holder.

If the initial count is exactly zero, the function returns `5`. This check takes precedence over candidate rejection. A negative count is not treated as empty.

### Position acceptance and unused fourth argument

With a nonzero initial count, acceptance requires all three signed conditions:

- The candidate is at least zero.
- The candidate is strictly below the destination limit.
- The candidate is strictly below the initial destination count.

Equality with either upper bound is rejected.

For a negative candidate, the function returns `6` without reading the destination limit. The relevant Boolean expressions short-circuit: neither their subtraction nor their subtraction-overflow evaluation reads the incoming fourth argument.

For a nonnegative candidate, the function reads the signed 32-bit destination limit at retained destination offset 68. That limit replaces the fourth argument’s local value before any subsequent read of that value. Thus, the incoming fourth argument is unused on this path as well.

The limit read occurs even if the candidate will later fail the count comparison. A negative candidate avoids the limit read; an empty destination avoids it regardless of candidate.

The represented sign and subtraction-overflow checks implement signed upper-bound ordering. Considering only the sign of a potentially overflowing subtraction would not preserve that behavior.

A nonnegative candidate cannot be accepted against a nonpositive limit or a nonpositive count.

### Shift loop

Acceptance establishes a positive initial count and a candidate smaller than that count.

If the candidate equals the initial count minus one, the shift loop is skipped. That candidate remains the position to clear.

Otherwise, the function enters a loop. Each condition reads the destination count again and compares the current position with that freshly read count minus one.

For every successful condition:

1. The four-byte value at the next position is read.
2. That value is written to the current position.
3. The current position advances by one.

The read precedes its corresponding write. Positions advance in increasing order.

The count is reread for the first condition and each later condition, including the terminating condition. The initial count is therefore not a snapshot controlling the whole shift.

### Clear and final count update

After skipping or completing the loop, the function:

1. Writes four zero bytes at the current position.
2. Reads the destination count again.
3. Subtracts one from that newly read count.
4. Writes the resulting count.
5. Returns `0`.

The final decrement uses the value read after the clear. It does not necessarily use the initial count or the last loop-condition count.

### Conditional ordinary-removal effect

With stable storage, valid accesses, representable arithmetic, and active entries that do not overlap metadata:

- Entries before the candidate are unchanged.
- Each later active entry moves one position toward the beginning.
- The former final active position becomes zero.
- The count decreases by one.
- The limit is not written.

These are conditional collection-level effects, not guarantees for arbitrary counts or aliasing.

Entry position 16 starts at byte offset 64 and overlaps the count. Position 17 starts at offset 68 and overlaps the limit. Larger positions reach further storage. The acceptance checks do not prevent these overlaps.

A shift or clear can therefore change metadata that a later operation reads. No explicit limit-field assignment is present, but an entry write can reach that field.

## Side effects

### Memory observation and mutation

On normal returns `1`, `5`, and `6`, the represented body performs no explicit memory writes. The reads differ according to the stopping point.

A successful path always performs a four-byte clear followed by a count read and count write. It may first perform shift reads and writes.

There are no explicit assignments to the named gate holders, gate bytes, or base holders. Aliasing can nevertheless make a destination write reach those locations.

The represented ordering distinguishes:

- Holder evaluation from access through the holder’s value.
- Initial and conditional second base evaluations.
- Initial count read from loop-condition and final count reads.
- The retained destination address from the destination-base holder.

These distinctions do not establish atomicity or synchronization.

### Ownership and lifetime

No function calls, allocation, deallocation, reference-count changes, destructor invocation, or ownership transfer are visible.

Entries are moved as uninterpreted four-byte values. If they encode resources, handles, or pointers, this function establishes no release behavior for the removed value or the cleared position.

The surrounding state must supply accessible holders, gate bytes, and fields for every reached access. Accepted mutations additionally require readable shift sources and writable destinations at every visited position. No allocation extent is independently checked.

### Failure and partial mutation

No rollback or fault recovery is visible.

If an access fails after mutation begins, earlier writes may already have occurred. No normal return value is established for that execution.

Repeated count reads make later control flow sensitive to changes in reached storage, including changes caused by aliasing. They do not establish that concurrent mutation is permitted or well-defined.

## Dependencies

No calls to other functions are visible. There are no established opaque function dependencies to enumerate.

The function depends on the global address/base holders and the memory reached through them. No external callee is assumed to preserve globals, retain or release memory, or provide synchronization.

## Acceptance examples

These are conditional static expectations, not runtime observations. Unless stated otherwise, assume accessible storage, stable holders and record contents, nonoverlapping active entries and metadata, representable arithmetic, and nonzero gate bytes.

### First gate disabled

With a zero first-gate byte:

- The first gate’s address reference is evaluated and the byte is read.
- The result is `1`.
- Neither base holder is evaluated.
- The source position, second gate, and destination are not accessed.

### Second gate disabled

With a nonzero first gate and zero second gate:

- Source-base evaluations required by the selector occur.
- The selected source position is read and reduced by one.
- The second gate is located and read.
- The result is `1`.
- The destination-base holder is not evaluated.

### Selector and changing holders

Suppose selector `1` produces source-base values A and then B. The source-position read uses B plus 136 bytes: 72 bytes of selection displacement and 64 bytes of field offset. Obtaining A does not itself cause a source-field read through A.

For selector `2`, there is one source-base evaluation, and the source field is read at that value plus 64 bytes.

Similarly, if selector `1` produces destination-base values C and then D, the selected destination starts at D plus 72 bytes. The initial count is read at D plus 136 bytes. Subsequent destination accesses retain that selected address even if the destination-base holder later changes.

These examples describe represented evaluations, not a guarantee of safe external mutation.

### Incoming fourth argument is irrelevant

With source position `0` and count `3`, the candidate is `−1`. The result is `6`; the limit is not read. Neither subtraction nor subtraction-overflow evaluation uses the incoming fourth argument.

With source position `2`, count `3`, and limit `3`, the candidate is `1` and is accepted. The limit replaces the fourth argument’s local value before any read of it.

Changing only the incoming fourth argument does not change either example’s represented accesses or result.

### Empty destination takes precedence

With source position `0` and count `0`:

- The candidate is `−1`.
- The result is `5`, not `6`.
- The limit is not read.
- No writes occur.

### Strict upper bounds

With source position `3`, the candidate is `2`:

- Count `5`, limit `2`: result `6`.
- Count `5`, limit `3`: accepted.

With source position `4`, the candidate is `3`:

- Count `3`, limit `10`: result `6`; the limit is still read.
- Count `4`, limit `10`: accepted, with the shift loop skipped.

A negative nonzero count is not empty, but cannot accept a nonnegative candidate.

### Middle removal

For entries `[11, 22, 33, 44]`, count `4`, limit `4`, and source position `2`:

- Zero-based position 2 is read and copied into position 1.
- Position 3 is read and copied into position 2.
- Position 3 is cleared.
- The final count read obtains `4`, and the count becomes `3`.
- The result is `0`.

The four positions become `[11, 33, 44, 0]`. Loop conditions reread the count even though it remains stable here.

### Single-entry removal

For entry `[11]`, count `1`, limit `1`, and source position `1`:

- The shift loop is skipped.
- Position 0 is cleared.
- The final count read obtains `1`.
- The count becomes `0`.
- The result is `0`.

### Count overlap

Assume count `17`, limit `16`, source position `16`, and otherwise accessible storage with no external changes. The candidate is `15`, so acceptance succeeds.

- The first loop condition permits a shift.
- Position 16, which overlaps the count, is read as `17`.
- That value is written into position 15.
- The next condition stops at position 16.
- Clearing position 16 sets the count to zero.
- The final count read obtains zero.
- The decrement writes `−1`.
- The result is `0`.

This behavior follows the represented checks but is not ordinary collection maintenance.

## Uncertainties

### Established static evidence

The representation establishes ordered gate checks, separate holder and reached-memory accesses, conditional second base evaluations, 72-byte selection displacement, a retained destination address, four-byte fields at offsets 64 and 68, signed acceptance conditions, repeated count reads, four-byte movement, and the four normal return values.

It also establishes that the incoming second, third, and fourth arguments are never read. In particular, short-circuit evaluation excludes use of the incoming fourth argument on the negative-candidate path; replacement by the destination limit precedes any value read on the nonnegative path.

### Inferred intent

The likely intent is deletion from one of two collections using a one-based source position.

Field names describe observed roles rather than confirmed application meanings. Sixteen entry positions before metadata suggest inline storage, but no sixteen-entry invariant is enforced here.

### Documented-only unknowns

The following remain unresolved:

- Application meaning, lifecycle, and stability of the gates and holders.
- Valid allocation extents and collection invariants.
- Aliasing among holders, gate bytes, source storage, and destination storage.
- Whether the limit denotes capacity, another count, or another boundary.
- Endianness, alignment requirements, and physical pointer width.
- Fault behavior and external memory-observation semantics.
- Concurrency rules, atomicity, and synchronization.
- Exact execution semantics for source-position, count, position-increment, and address overflow.

Subtracting one from the minimum signed 32-bit source position is not representable in that width. A wrapping interpretation would yield the maximum signed 32-bit value, but that behavior is not independently verified.

Malformed or changing storage can also expose overflow in later count-minus-one calculations, position advancement, or address formation. Those executions remain documented-only. No overflow qualification is needed for the incoming fourth argument because its value is never read.

## Evidence

This contract is based solely on the supplied static representation of F-df2c7f3bb9d4.

The described control flow distinguishes short-circuited expressions from evaluated expressions, local argument replacement from caller-visible writes, repeated base-holder evaluations from reached-memory accesses, and initial count checks from later count reads.

No runtime tests, implementation execution, external dependency analysis, or platform-level instruction verification have been performed.
