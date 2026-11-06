<!-- Trusted provenance: F-3affa7926e3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3affa7926e3f

## Purpose

F-3affa7926e3f selects one record from caller-supplied state and partially clears that record. Selection has three cases:

- Use the head of a reusable-record list when that head is nonzero.
- Otherwise, use the sequential cursor when it differs from the sequential limit.
- Otherwise, obtain descriptor and buffer addresses through F-871e64ddc6de, publish the new descriptor, and select the beginning of its buffer.

The selected record receives four ordered stores: four-byte zeros at offsets 4, 8, and 12, followed by a one-byte zero at offset 0. The function does not clear a complete 24-byte record.

There is no explicit return value or selected-record output parameter. Its established outputs are memory changes and, on the growth path, two dependency calls. How callers subsequently obtain the selected record is not established.

The interpretation as a reusable-record pool backed by a chain of storage blocks is inferred intent. It does not establish allocation guarantees, ownership, synchronization, or validation.

## Interface

### Input and output

The sole input is a pointer to mutable state. The supplied representation uses four-byte fields, including address-bearing fields. The layouts below describe that represented environment; they are not a portable layout for environments with wider pointers.

Normal completion has no explicit result value or error status.

### State layout

| Byte offset | Width | Observed role |
|---|---:|---|
| 0 | 4 bytes | Current descriptor address |
| 4 | 4 bytes | Reusable-record head address |
| 8 | 4 bytes | Sequential cursor |
| 12 | 4 bytes | Sequential limit |

A zero reusable-record head selects the sequential path. A zero current-descriptor address selects the initial growth count, but that field is consulted only when growth is needed.

The cursor and limit are compared for equality. They are not checked for ordering or for the presence of a full record.

### Descriptor layout

The first dependency result on the growth path is used as a descriptor:

| Byte offset | Width | Observed role |
|---|---:|---|
| 0 | 4 bytes | Previous-descriptor address |
| 4 | 4 bytes | Unsigned count |
| 8 | 4 bytes | Record-buffer address |

An existing descriptor supplies its count for the growth calculation. Its other fields are not inspected for that calculation.

The first dependency call receives 12 as its first argument. This agrees with the accessed descriptor extent, but the dependency’s actual storage contract is unavailable.

### Record layout

The sequential stride is 24 bytes. The directly accessed record regions are:

| Byte offset | Width | Access |
|---|---:|---|
| 0 | 1 byte | Cleared last |
| 4 | 4 bytes | Cleared first |
| 8 | 4 bytes | Cleared second |
| 12 | 4 bytes | Read as a link on reuse; cleared third |

The reset writes 13 bytes in total. Bytes 1–3 and 16–23 are outside its direct write ranges. Their preservation requires that other writes and dependency effects do not overlap or alter them.

Only the reusable-link interpretation of offset 12 is established. The meanings of the other cleared fields are unknown.

### Preconditions for ordinary completion

Storage must support every access reached on the selected path:

- The state must permit the required four-byte reads and writes.
- A reused record must permit the four-byte link read at offset 12 and all reset stores.
- A sequentially selected record must permit all reset stores.
- A descriptor used for growth calculation must permit the count read at offset 4.
- The first dependency result must remain usable for descriptor initialization and the later buffer-address read.
- The buffer address ultimately selected must permit all reset stores.

These conditions are not checked. Alignment requirements, invalid-address behavior, and out-of-range address arithmetic depend on the execution environment.

Nonoverlapping state, descriptors, and records are necessary for the ordinary interpretation as separate objects. Nonoverlap is not enforced.

## Behavior

### Selection priority and reuse

The first selection read obtains the reusable-record head from state offset 4.

A nonzero head selects that record immediately. The cursor and limit are not inspected on this path. The selected record’s four-byte link at offset 12 is read and assigned to state offset 4 before any reset store occurs.

The link is accepted without validation. Zero, a self-link, or an address participating in a cycle does not cause rejection or change the current invocation’s selection.

This path makes no dependency calls. It performs no direct assignments to the current descriptor, cursor, or limit. Aliasing with the selected record can nevertheless make the later reset affect those locations.

### Sequential selection without growth

When the reusable-record head is zero, the existing cursor supplies the candidate record address. The cursor is compared with the limit.

If they differ, that candidate is selected without inspecting a descriptor or calling a dependency. The cursor is assigned the selected address plus 24 bytes before the record reset begins. The function does not directly update the limit or current descriptor on this path.

Inequality is the entire availability condition. A cursor beyond the limit, or less than 24 bytes before it, still takes this path if the addresses differ. This branch choice does not establish that the resulting accesses are valid.

### Growth calculation

Growth occurs only when the reusable-record head is zero and the cursor equals the limit.

If the current-descriptor address is zero, the selected count is 32. Otherwise, the count at descriptor offset 4 is interpreted as an unsigned 32-bit integer \(c\).

The two candidates are:

\[
g=\left(c+\left\lfloor c/2\right\rfloor+\left\lfloor c/8\right\rfloor\right)\bmod 2^{32}
\]

\[
h=(c+32)\bmod 2^{32}
\]

The selected count \(n\) is the larger of \(g\) and \(h\), using unsigned 32-bit comparison. The two shifted contributions truncate independently.

Without overflow, this chooses between an additive increase of 32 and an increase consisting of half the count plus one eighth of the count. With overflow, the comparison is between wrapped candidates. There is no guarantee that \(n\) exceeds \(c\), and no minimum-count or overflow validation follows.

The size expression used for the buffer request is:

\[
s=(24n)\bmod 2^{32}
\]

This multiplication can wrap independently of the candidate additions.

### Growth calls and observable publication order

The growth path calls F-871e64ddc6de twice:

| Order | First argument | Second argument | Use of result |
|---|---:|---:|---|
| First | 12 | 0 | Descriptor address |
| Second | \(s\) | 0 | Buffer address |

The count is determined before either call. Both calls finish before the function directly initializes the returned descriptor. In particular, the first result is not initialized before the second call.

After both calls return, the observable descriptor and state operations have this order:

1. Store the second result into descriptor offset 8.
2. Read the state’s current-descriptor address and store it into descriptor offset 0.
3. Store \(n\) into descriptor offset 4.
4. Publish the new descriptor address at state offset 0.
5. Read the buffer address back from descriptor offset 8 and select it.
6. Write the limit as the selected address plus \(s\).
7. Write the cursor as the selected address plus 24 bytes.
8. Perform the four record-reset stores.

The previous-descriptor link therefore comes from a state read after both calls, not necessarily from the descriptor address used earlier to calculate growth. That read also occurs after the buffer-address write to the new descriptor.

The selected address is a reread of the descriptor’s buffer field after descriptor initialization and publication. It is not established merely by treating the second dependency result as permanently unchanged.

These distinctions matter if dependencies modify the state or if storage overlaps. Under valid, nonoverlapping storage and noninterfering dependencies, the new descriptor links to the former descriptor, the selected record is the first record in the returned buffer, and the cursor advances by one record.

The function does not revisit the reusable-record head after either dependency call. A dependency mutation of that field does not redirect the invocation to reuse.

### Record reset

All normally completing selection paths converge on the same reset:

| Order | Record-relative offset | Width | Stored value |
|---|---:|---:|---:|
| First | 4 | 4 bytes | Zero |
| Second | 8 | 4 bytes | Zero |
| Third | 12 | 4 bytes | Zero |
| Fourth | 0 | 1 byte | Zero |

There is no whole-record clearing operation or additional direct record store.

On reuse, the list head is updated before this reset. On either sequential path, the cursor is updated before it. On growth, descriptor publication and the limit update also precede it.

This is an ordering contract for the supplied operations, not a guarantee of atomicity, inter-thread visibility, or transactional completion.

### Arithmetic limits

Count arithmetic, candidate comparison, and the size multiplication use unsigned 32-bit values. Wrapped sizes are used without an explicit rejection path.

The function also forms addresses by adding the size or the 24-byte stride. The source does not establish a portable outcome for out-of-range address arithmetic. A representable size value does not prove that a corresponding buffer exists, and arithmetic wrapping does not make a subsequent memory access valid.

## Side effects

### Direct writes by path

For nonoverlapping storage:

| Path | State fields directly assigned | Other direct writes |
|---|---|---|
| Reuse | Reusable-record head | Four selected-record reset stores |
| Sequential without growth | Cursor | Four selected-record reset stores |
| Growth | Current descriptor, limit, cursor | Three descriptor fields and four selected-record reset stores |

Neither sequential path directly assigns the reusable-record head. The growth path does not restore that field if a dependency changes it.

No other record in a newly obtained buffer is directly initialized by this function.

### Ownership and lifetime

The growth path retains the first dependency result in the state and retains the second result in the descriptor. It also retains the later-read current-descriptor address as a backward link.

There is no explicit release, deallocation, ownership transfer notification, or rollback operation. Whether these addresses refer to newly allocated, borrowed, shared, or otherwise managed storage is unknown.

The first result must remain usable across the second call for ordinary completion. The source does not establish that the dependency provides this guarantee. Required lifetime after return and responsibility for eventual release are also unknown.

### Errors and partial progress

Neither dependency result is checked before use. There is no explicit failure status, recovery branch, or validation of a returned address.

If a dependency does not return, subsequent operations do not occur. If it returns an unusable address, a later access may fail; no particular exception or other defined failure result is established.

Failure during the second call has no explicit cleanup of the first result. Whether that entails a resource leak depends on the dependency’s unknown ownership and failure contract.

Updates are not rolled back if the reset fails or execution is interrupted. Partial progress can leave:

- A reusable head advanced before the selected record is fully reset.
- A cursor advanced before reset completion.
- A new descriptor published before the limit or cursor is updated.
- Only a prefix of the four reset stores completed.

Overlapping storage can alter these ordinary effects by changing later reads or overwriting earlier writes.

## Dependencies

### F-871e64ddc6de

This is the only explicit external dependency.

Established use:

- Exactly two calls on growth and none on the other paths.
- Two arguments per call.
- A second argument of zero for both calls.
- A first argument of 12 for the descriptor request.
- A first argument of \((24n)\bmod 2^{32}\) for the buffer request.
- Results subsequently used as addresses without validation.

An allocation-like role is inferred from these uses. The dependency’s contract is not supplied.

The following remain **documented_only**:

- Formal argument types and parameter meanings.
- Returned storage extent, alignment, and initialization.
- Failure signaling and behavior for any particular request size.
- Ownership, release requirements, and lifetime guarantees.
- Preservation or modification of caller state, globals, or previously returned storage.
- Reentrancy and concurrency behavior.

No claim here assumes that either call preserves the state or that the second call preserves the usability of the first result.

## Acceptance examples

These are conditional static expectations, not runtime results. Unless otherwise stated, storage is valid and nonoverlapping, address additions are usable, and dependencies return usable storage without modifying relevant objects.

### Reuse takes priority

Given a reusable head \(R\neq0\), with link \(S\) at record offset 12:

- State offset 4 receives \(S\) before the link is cleared.
- No dependency is called.
- The current descriptor, cursor, and limit receive no direct assignments.
- \(R\) receives the four ordered reset stores.
- Bytes 1–3 and 16–23 retain their prior contents.

If \(S=R\), the reusable head remains \(R\), while the record’s link becomes zero. The self-link is not rejected.

### Unequal cursor and limit

Given a zero reusable head and cursor \(P\) unequal to limit \(E\):

- The selected record is \(P\).
- The cursor receives \(P+24\) before reset.
- No descriptor is inspected and no dependency is called.
- The reset affects \(P\).

The same branch is selected if \(P\) is beyond \(E\). Access safety remains a separate precondition.

### Initial growth

Given a zero reusable head, equal cursor and limit, and a zero current descriptor:

- The selected count is 32.
- The dependency receives first arguments 12 and 768, in that order.
- No direct descriptor initialization occurs before the second call.
- The descriptor records a zero previous link, count 32, and the buffer address.
- The limit becomes the buffer address plus 768.
- The cursor becomes the buffer address plus 24.
- Only the first record receives the reset stores.

The zero previous link depends on the stated noninterference and nonoverlap assumptions.

### Ordinary growth counts

For \(c=32\):

- \(g=52\).
- \(h=64\).
- \(n=64\).
- The second call’s first argument is 1536.

For \(c=64\):

- \(g=104\).
- \(h=96\).
- \(n=104\).
- The second call’s first argument is 2496.

### Wrapped buffer-request size

For \(c=2^{31}=2{,}147{,}483{,}648\):

- \(g=3{,}489{,}660{,}928=\text{0xD0000000}\).
- \(h=2{,}147{,}483{,}680=\text{0x80000020}\).
- The selected count is \(g\).

The unbounded product is:

\[
24\times3{,}489{,}660{,}928=83{,}751{,}862{,}272
\]

Its unsigned 32-bit result is:

\[
83{,}751{,}862{,}272\bmod2^{32}
=2{,}147{,}483{,}648
=\text{0x80000000}
\]

The second dependency call therefore receives **2,147,483,648**, not zero, as its first argument. The limit calculation uses that same wrapped size. This example establishes neither zero-size behavior nor successful provision of the requested storage.

### Dependency mutation of the current descriptor

If either dependency changes state offset 0, the new descriptor’s previous link receives the value read after both calls, subject to effects of the preceding descriptor-buffer write.

That value need not equal the descriptor address used for the earlier growth-count calculation. The count is not recomputed in response to that change.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the cleared fields other than the reusable-link use.
- How callers retrieve or identify the selected record.
- Caller guarantees concerning alignment, capacity, cursor ordering, nonoverlap, and synchronization.
- Whether arithmetic-overflow cases are reachable in intended use.
- Exact runtime consequences of invalid addresses or out-of-range address calculations.
- Dependency allocation, mutation, failure, ownership, and lifetime behavior.
- Eventual cleanup of descriptors and buffers.
- Applicability of these layouts outside the represented four-byte address environment.

The pool interpretation, count-as-capacity interpretation, and allocation-like dependency interpretation are inferred intent, not additional established behavior.

## Evidence

Static inspection of the supplied source establishes:

- Four state fields at four-byte spacing.
- Reuse-first selection and a reusable link at record offset 12.
- Equality-only detection of sequential exhaustion.
- A 24-byte sequential stride.
- Unsigned 32-bit growth candidates and buffer-size multiplication.
- Two ordered calls to F-871e64ddc6de on growth.
- Descriptor initialization only after both calls.
- A post-call read of the current-descriptor field.
- Descriptor publication before the buffer-field reread, limit update, cursor update, and reset.
- Three four-byte reset stores followed by one single-byte reset store.
- No explicit return value, dependency-result check, release, or rollback.

The wrapped-size example is an arithmetic derivation from those widths and operations. No runtime tests have been performed. All acceptance examples are conditional static expectations.
