<!-- Trusted provenance: F-2ca62cf8d9c9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ca62cf8d9c9

## Purpose

F-2ca62cf8d9c9 selects storage for one 24-byte record and clears four fields within that record. Selection prioritizes a reusable-record chain. If that chain is empty, selection uses the current region cursor, obtaining a new descriptor and record region when the cursor equals its limit.

The function has no explicit return value. Its observable results are context mutations, initialization and publication of a new descriptor when growth occurs, and partial initialization of the selected record.

**Established static behavior:** record selection, capacity arithmetic, two ordered dependency calls on the growth path, pointer-bearing field updates, and ordered clearing writes.

**Inferred intent:** management of a pool of reusable, fixed-stride records. This interpretation does not establish field meanings, ownership, allocation guarantees, or validation requirements beyond the accesses described below.

## Interface

### Argument and context layout

The sole argument identifies a context containing at least four consecutive 32-bit fields:

| Byte offset | Role |
|---|---|
| 0 | Descriptor-chain head |
| 4 | Reusable-record-chain head |
| 8 | Next-unused-record cursor |
| 12 | Limit compared with that cursor |

Zero has special significance for the descriptor and reusable-record heads. The cursor and limit are compared for equality; neither is independently checked for zero.

The supplied representation uses four-byte integer and pointer-bearing fields. This contract describes that representation, not a portable layout for wider pointers.

There is no local validation of the context address, its extent, field alignment, cursor ordering, or addresses obtained through its fields.

### Descriptor layout

A newly obtained descriptor has the following 12-byte layout:

| Byte offset | Value written |
|---|---|
| 0 | Context descriptor head read during publication |
| 4 | Calculated capacity, in records |
| 8 | Result of the record-region dependency call |

An existing descriptor contributes only its unsigned 32-bit capacity at offset 4 to the growth calculation. Its predecessor and region fields are not traversed for that calculation.

### Record layout

Selected records are spaced by 24 bytes. This function directly accesses these portions:

| Byte offset | Width | Access |
|---|---:|---|
| 0 | 1 byte | Final zero write |
| 4 | 4 bytes | Zero write |
| 8 | 4 bytes | Zero write |
| 12 | 4 bytes | Reusable-link read on the reusable path; zero write on every path |

Bytes 1–3 and 16–23 receive no direct writes from this function. It does not establish a wholly zero-filled record.

### Result and safe-use preconditions

There is no explicit returned record address, status, or error code. How a caller subsequently identifies or uses the selected record is not supplied.

Normal completion requires usable storage for every access actually reached. On a growth path, the dependency results must support the descriptor and record writes. The function does not check these requirements.

Independent effects on the context, descriptor, and record assume non-overlapping storage. When those objects overlap, earlier writes can change later reads or writes. No synchronization or concurrent-use protocol is established.

## Behavior

### Selection priority

The first selection read obtains the context’s reusable-record head at offset 4.

If that value is nonzero, it identifies the selected record. The function reads the four-byte link at record offset 12 and writes that value to context offset 4. It then performs the common record-clearing sequence.

This path does not directly inspect the cursor, limit, or descriptor capacity and does not call F-871e64ddc6de. A nonzero reusable head therefore takes priority even when the cursor equals the limit.

The link is consumed before it is cleared. There is no validation of the link address, chain membership, or cycles.

### Selection from an existing region

If the reusable head is zero, the function reads the cursor at context offset 8 and compares it with the limit at offset 12.

When they differ:

- The cursor value identifies the selected record.
- The context cursor is updated to the selected address plus 24.
- The common clearing sequence follows.

This is an equality test, not an available-space or ordered-bounds test. A cursor beyond the limit, a misaligned cursor, or any other non-equal pair still selects the cursor path. The function does not verify that 24 bytes remain or that the selected record is writable.

No descriptor is created on this path.

### Growth when cursor equals limit

When the reusable head is zero and the cursor equals the limit, the function calculates a capacity, calls F-871e64ddc6de twice, initializes and publishes a descriptor, and selects the new region’s first record.

#### Capacity arithmetic

A zero descriptor head produces a capacity of 32 records.

For a nonzero descriptor head, the function reads its capacity as an unsigned 32-bit value \(c\). It calculates:

- Increment candidate: \(c + 32\).
- Fractional candidate: \(c + (c \gg 1) + (c \gg 3)\).

Both shifts are unsigned right shifts. Each addition uses unsigned 32-bit arithmetic, wrapping modulo \(2^{32}\). The larger candidate is selected by unsigned comparison. Equality selects the fractional candidate, with no numerical difference.

Without overflow, the fractional candidate is:

\[
c + \lfloor c/2 \rfloor + \lfloor c/8 \rfloor.
\]

The resulting capacity is multiplied by 24 using unsigned 32-bit arithmetic to form the record-region byte request. This multiplication also wraps modulo \(2^{32}\). There is no overflow check or minimum-result check.

Consequently, “growth” describes inferred intent, not a guaranteed increase in capacity or usable storage for arbitrary input capacities.

#### Dependency calls

The dependency calls occur in this order:

1. F-871e64ddc6de receives 12 and zero. Its result is retained as the new descriptor address.
2. F-871e64ddc6de receives the wrapped capacity-times-24 byte request and zero. Its result is used as the region address.

The function performs no descriptor initialization between these calls. It does not validate the first result before making the second call.

The capacity has already been calculated before either call. Changes by a dependency to the context or an existing descriptor do not cause capacity recalculation.

#### Descriptor publication and cursor updates

After both calls return, the direct operations occur in this order:

1. Write the second result to new-descriptor offset 8.
2. Read the context’s then-current descriptor head and write it to new-descriptor offset 0.
3. Write the calculated capacity to new-descriptor offset 4.
4. Write the new descriptor address to context offset 0.
5. Read the region address back from new-descriptor offset 8 and select that address.
6. Write the selected address plus the wrapped capacity-times-24 quantity to context offset 12.
7. Write the selected address plus 24 to context offset 8.
8. Perform the common record-clearing sequence.

The descriptor head used to calculate capacity need not equal the predecessor stored in the new descriptor. The predecessor is read after both dependency calls and after the region-address store.

The region address is read back from the descriptor rather than merely treated as an unchanged call result. With overlapping storage, intervening writes can therefore affect selection.

The function performs no direct write to the reusable-head field on the growth path; its final value depends on dependency effects. In particular, a dependency can change context offset 4 after the initial zero read, and this function neither rechecks nor restores that field.

For disjoint storage and dependencies that preserve the context, the new descriptor links to the previous head, the limit marks the end of the requested region, and the cursor advances past its first record.

### Common partial initialization

Every normally completing selection path ends with these direct writes, in order:

1. Four zero bytes at selected-record offset 4.
2. Four zero bytes at selected-record offset 8.
3. Four zero bytes at selected-record offset 12.
4. One zero byte at selected-record offset 0.

The reusable-head update or cursor update precedes these writes. On growth, descriptor publication and the limit update also precede them.

This ordering describes the supplied static operations. It does not establish atomicity, memory barriers, or safe intermediate-state observation by concurrent readers.

## Side effects

### Direct mutations

With context, descriptors, and records disjoint:

| Path | Direct context writes | Other direct writes |
|---|---|---|
| Reusable record | Reusable head | Selected-record clearing |
| Existing region | Cursor | Selected-record clearing |
| New region | Descriptor head, limit, cursor | Three descriptor fields, then selected-record clearing |

On the growth path, the function performs no direct write to context offset 4. Its final value is not guaranteed to be zero merely because storage is non-overlapping: either dependency call may change it.

More generally, the table lists direct writes by F-2ca62cf8d9c9, not restrictions on dependency effects. Context fields without direct writes can still change through dependency behavior.

The function does not directly release storage, maintain a visible active-record count, or clear every byte of the selected record.

### Reachability, ownership, and lifetime

The new descriptor address is published in the context, and the region address is stored in that descriptor. This establishes reachability under ordinary disjoint-storage conditions.

It does not establish ownership transfer, exclusive access, allocation lifetime, or an eventual release obligation. No release operation appears here, and no valid release mechanism is identified.

### Failures and incomplete execution

There is no explicit error branch for either dependency result. Both results are used as addresses without local validation.

If the first call returns an unusable descriptor address, the second call is still attempted. After both calls return, the first descriptor access is the write at offset 8.

If a call does not return normally, later operations are not established as occurring. If an access fails, the resulting exception, diagnostic, termination behavior, or recovery mechanism is unknown.

There is no explicit rollback. Earlier dependency effects, descriptor stores, context publication, cursor updates, or clearing writes may remain observable if execution stops later. In particular, record initialization occurs after the selected record has already been removed from the reusable chain or passed by the cursor.

## Dependencies

### F-871e64ddc6de

Established call-site behavior:

- Called only after an initial zero reusable head and equal cursor and limit.
- Called twice, in order, if execution reaches and returns from both calls normally.
- First call arguments: 12 and zero.
- Second call arguments: the unsigned 32-bit wrapped product of calculated capacity and 24, and zero.
- First result: used as a descriptor address.
- Second result: stored as a region address and subsequently read back.

**Inferred intent:** obtain storage for a descriptor and a region of records.

**documented_only:** the second argument’s meaning, result alignment, initialization state, failure signaling, non-return behavior, ownership transfer, and effects on globals or caller-accessible storage are unavailable.

No preservation guarantee is established for either call. This includes the context’s reusable head, descriptor head, cursor, and limit. A changed descriptor head can become the new descriptor’s predecessor; a changed reusable head is not overwritten by this function on the growth path.

No other direct dependency is shown.

## Acceptance examples

These are conditional static acceptance examples, not runtime observations. Unless an example states otherwise, assume valid writable addresses, the supplied 32-bit representation, disjoint objects, normally returning dependencies, and dependencies that leave the context unchanged.

### Reusable record takes priority

Given a nonzero reusable head pointing to record A, with record A’s offset-12 link identifying record B:

- A is selected even if the cursor equals the limit.
- The reusable head becomes B before A’s link is cleared.
- No dependency call occurs.
- A receives the four clearing writes in the specified order.
- The context cursor and limit remain unchanged.

If A’s link instead points to A, the reusable head remains A while A’s link becomes zero. The function does not reject this cycle.

### Existing-region selection

Given a zero reusable head and a cursor P different from the limit:

- P is selected.
- The cursor becomes P plus 24 before clearing begins.
- No descriptor is created.
- Record bytes 1–3 and 16–23 retain their previous values.

The same selection branch is taken if P is already beyond the limit. Normal completion still requires writable storage at the accessed addresses.

### First region

Given a zero reusable head, equal cursor and limit, and a zero descriptor head:

- Capacity is 32.
- Dependency requests are 12 bytes and 768 bytes, in that order.
- For returned descriptor D and region R, D receives predecessor zero, capacity 32, and region address R.
- The context head becomes D, limit becomes R plus 768, and cursor becomes R plus 24.
- The first record at R is partially initialized.
- The reusable head remains zero under the stated dependency-preservation assumption, not because this function writes it.

### Dependency changes context during growth

Suppose capacity calculation uses an initial descriptor head H. During one of the two calls, the dependency changes the context descriptor head to J and the reusable head to Q. Assume these changes persist until publication and all objects remain disjoint and usable.

Then:

- Capacity remains the value calculated from H.
- The new descriptor’s predecessor becomes J.
- The reusable head remains Q; this function does not restore zero.
- Selection still uses the newly obtained region, without reconsidering Q.

This distinguishes the initial selection decision from context values observed or retained after dependency calls.

### Ordinary capacity growth

For existing capacity 32, the candidates are 64 and 52. Capacity becomes 64, requesting 1,536 bytes.

For existing capacity 64, the candidates are 96 and 104. Capacity becomes 104, requesting 2,496 bytes.

### Wrapped capacity arithmetic

For existing capacity \(2^{32}-1\):

- The increment candidate wraps to 31.
- The fractional candidate wraps to 2,684,354,557.
- The fractional candidate is selected.
- Its byte request wraps to 4,294,967,224.

Acceptance requires these wrapped unsigned calculations. Unbounded arithmetic, saturation, or overflow rejection would introduce behavior not shown here. The arithmetic result alone does not establish successful storage acquisition or valid pointer arithmetic.

## Uncertainties

- **documented_only — Field semantics:** The meanings of the cleared fields are unknown. A one-byte zero write at offset 0 does not establish a particular flag type.
- **documented_only — Record delivery:** No explicit result communicates the selected address; caller-side discovery or use is unavailable.
- **documented_only — Dependency contract:** Storage guarantees, ownership, side effects, failure signaling, and preservation of context or global state are unknown.
- **documented_only — Address arithmetic:** Integer capacity operations have unsigned 32-bit width. The full platform behavior for pointer overflow, invalid addresses, and alignment violations is not supplied.
- **documented_only — Pool invariants:** Valid descriptors, aligned records, ordered cursors, acyclic chains, and non-overlap are not enforced here.
- **documented_only — Concurrency:** No locking, atomic publication, or safe concurrent-use contract is established.
- **Inferred intent:** Reusable-record selection and growable-region management explain the layout and operations, but do not add guarantees.

## Evidence

This contract is based exclusively on the supplied static representation of F-2ca62cf8d9c9 and the opaque dependency mapping.

Established evidence comprises the context and descriptor offsets, 24-byte record stride, reusable-link read, equality-based exhaustion decision, unsigned capacity arithmetic, two ordered dependency calls, descriptor-publication sequence, and ordered partial clearing.

The absence of a direct growth-path write to the reusable head is distinct from preservation of its value. No dependency implementation or preservation guarantee was supplied.

No caller implementation, runtime observations, or execution-platform failure contract was supplied. No runtime tests have been performed.
