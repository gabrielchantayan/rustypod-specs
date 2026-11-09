<!-- Trusted provenance: F-639a5ab1ccf7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-639a5ab1ccf7

## Purpose

F-639a5ab1ccf7 initializes selected fields of a caller-supplied state object, performs dependency-driven setup, conditionally consumes one of two shared pending values, and enters an event-processing loop with no visible normal-return path.

An event matching the restart field begins another outer cycle; it does not return from the function. Record draining can itself continue indefinitely if its continuation dependency never returns zero.

The established contract concerns memory accesses, dependency calls, arguments, comparisons, and their ordering. Synchronization, notification, timing, and queue-processing interpretations are inferred intent, not established dependency semantics.

## Interface

### Input, result, and storage requirements

The sole explicit input is a pointer to writable state storage. There is no explicit length, result object, or defined error return.

The supplied representation establishes:

- 8-bit byte fields.
- Unsigned 16-bit record eligibility fields.
- 32-bit scalar fields and dependency results.
- A 32-bit pointer representation.
- Signed 32-bit interpretations for the record-selection index and event comparisons.

Fixed state accesses extend through offset `0x857`, inclusive. Record accesses can require additional storage, depending on the selected index. Dependencies receiving the state pointer or an interior pointer can impose further requirements that are not available here.

No null-pointer, alignment, storage-extent, index, or handle validation is visible.

### State layout

Offsets are relative to the input pointer.

| Offset | Width | Established use |
|---|---:|---|
| `0x00` | 1 byte | First argument to eligible-record processing |
| `0x01` | 1 byte | Selector read independently for four setup calls |
| `0x08` | 1 byte | Optional setup selector |
| `0x09` | 1 byte | Optional setup selector examined before `0x08` |
| `0x0c` | 4 bytes | Argument read during each outer cycle |
| `0x10` | 4 bytes | First descriptor word and restart comparison field |
| `0x14` | 4 bytes | Second descriptor word and record-drain comparison field |
| `0x18` | 4 bytes | Third descriptor word and callback comparison field |
| `0x1c` | 4 bytes | Value captured once at entry |
| `0x20` | 1 byte | Written with decimal `100` |
| `0x22` | Variable | Record-region base |
| `0x83c` | 4 bytes | Receives the address of state offset `0x22` |
| `0x840` | 4 bytes | Record-selection index and index-transform result |
| `0x851` | 1 byte | Cleared during every outer cycle |
| `0x854` | 4 bytes | Receives a dependency result during every outer cycle |

The selected record begins at state offset `0x22` plus the signed index multiplied by `0x206`, a stride of 518 bytes.

| Record-relative offset | Width | Established use |
|---|---:|---|
| `0x00` | 1 byte | Must be nonzero for processing eligibility |
| `0x02` | 2 bytes, unsigned | Must exceed `1` for processing eligibility |
| `0x04` | Unspecified | Address passed to F-fe0f8d21cd8c |

No record count or payload length is established. The payload address is not established to identify a string. Record addressing is calculated directly from the input pointer; the stored pointer at `0x83c` is not read for this calculation.

The layout does not establish disjoint regions. Sufficiently large record indices can overlap later fixed fields. Valid operation requires storage and indices consistent with every actual access.

### Shared-base global and shared storage

A global pointer supplies bases for shared accesses. Relative to a base obtained from that global:

| Offset | Width | Established use |
|---|---:|---|
| `0x00` | 1 byte | Written with `1` on the first-pending branch |
| `0x04` | 4 bytes | First pending field |
| `0x08` | 4 bytes | Second pending field |

The global pointer is evaluated separately for the pending-field accesses. A distinct, earlier evaluation is cached solely for the later flag-byte write. These evaluations are not established to return the same address.

A pending field is active whenever its 32-bit value is nonzero, including bit patterns interpreted as negative signed integers.

## Behavior

### Entry setup

The function first reads the 32-bit value at state offset `0x1c` and retains it for all outer cycles. It then writes decimal `100` to `0x20` and the address of state offset `0x22` to `0x83c`.

Setup calls follow in this order:

1. Examine the byte at `0x09`. If its bit pattern is not `0xc8`, call F-ef10ee2809ab with the byte at `0x09` and zero.
2. Examine the byte at `0x08`. If its bit pattern is not `0xc8`, call F-17d6434f9fa3 with the byte at `0x08` and zero.
3. Call F-1a4256178b1e with the byte at `0x01` and `1`.
4. Call F-058705628f93 with the byte at `0x01`.
5. Call F-c1a1c3072914 with the byte at `0x01`.
6. Call F-3f5ace521ed4 with the byte at `0x01`.

Each selector argument is read at its call site. The optional calls have a selector examination followed, when enabled, by a selector read for the argument. There is no intervening dependency call between an optional selector’s examination and argument evaluation, but asynchronous stability is not established.

The four reads of `0x01` are independent. Earlier dependencies can therefore affect later arguments. The `0x08` examination also occurs after the optional `0x09` call returns.

The sentinel is an 8-bit pattern. Signedness and argument extension for other byte values remain unresolved.

### Initial shared-state consumption

The function calls F-a10000000001 with `14`. After that call returns, the shared-base evaluations and accesses occur in the following order:

1. Read the shared-base global and retain that address for a possible later flag-byte write.
2. Read the shared-base global again and read the 32-bit first pending field at offset `0x04` from this newly obtained base.
3. Select a branch using that pending value.

**If the first pending value is nonzero:**

- Read the shared-base global again and clear the 32-bit field at offset `0x04` from that newly obtained base.
- Call F-1b765582ee90 with `1`.
- Call F-a10000000003 with `0` and `2000`.
- Write byte value `1` at offset `0x00` of the address retained in step 1.
- Retain the originally read nonzero pending value as a local indicator.
- Do not examine the second pending field.

**If the first pending value is zero:**

- Read the shared-base global again and read the 32-bit second pending field at offset `0x08` from that newly obtained base.
- If this second value is nonzero, read the shared-base global yet again, clear the 32-bit field at offset `0x08` from that latest base, and retain the value previously read as a local indicator.
- If the second value is zero, perform no pending-field clear.
- Make neither of the first branch’s two dependency calls and perform no flag-byte write.

Both branches then call F-a10000000002 with `14`.

The distinction between bases is part of the contract. The function does not use the cached flag-write base for pending reads or clears. Nor does it reuse a pending-read base for its corresponding clear. If the global’s evaluations yield different addresses, a clear can target a different object from the one whose nonzero value selected the branch.

No dependency call intervenes between the initial base capture, pending reads, and the selected pending clear. Nevertheless, neither stability against asynchronous changes nor exclusion supplied by F-a10000000001 is established. The subsequent two calls on the first-pending branch can also affect the global, but the flag write still uses the earlier cached address.

Priority depends on the first value actually read: a nonzero first value prevents any second-field read. The retained pending values subsequently act only as nonzero indicators, not as forwarded payloads.

### Outer-cycle setup

Each outer cycle performs these operations in order:

1. Write decimal `100` to state offset `0x20`.
2. Call F-9956f7343fc0 with `8` and the value captured from `0x1c` at entry.
3. Read the 32-bit state field at `0x0c` and call F-a10000000004 with that value.
4. If either retained pending indicator is nonzero:
   - Call F-a10000000001 with `14`.
   - Call F-a10000000005 with `22`.
   - Call F-a10000000004 with `22`.
   - Clear both local pending indicators.
   - Call F-a10000000002 with `14`.
5. Clear the byte at state offset `0x851`.
6. Call F-a10000000006 without explicit arguments.
7. Store its 32-bit result at state offset `0x854`.

Assuming normal dependency returns, the pending-related sequence occurs at most once, in the first outer cycle. Later cycles do not recheck shared pending fields.

The arguments `14`, `22`, and `2000` are exact values; their meanings and units are not established.

### Event acquisition and dispatch

Every inner iteration prepares a temporary descriptor containing four consecutive 32-bit words. The fourth word is set to zero first. The first three words are then filled, in order, from state offsets `0x10`, `0x14`, and `0x18`.

F-a10000000007 receives the descriptor address and returns the 32-bit event value.

Dispatch uses fresh state reads after this call:

1. Compare the event with the current field at `0x18`. Equality calls F-abff49696f96 with the state pointer.
2. Only if that comparison fails, compare the event with the current field at `0x14`. Equality enters record draining.
3. After the selected action completes, or after neither comparison matches, compare the event with the current field at `0x10`.
4. Equality begins another outer cycle. Inequality acquires another event.

Thus callback dispatch takes priority over record draining when both fields match. Restart is independent and checked afterward. A callback or record-drain dependency can change the field used by the restart comparison.

The descriptor is a snapshot assembled from separate reads, not an authoritative dispatch table retained for the iteration. Dependency mutations can make later comparisons differ from the values supplied in the descriptor. No event value, including a negative value, has a separate error branch.

### Record draining

F-1468513d650e receives the state pointer before each potential record iteration. Zero ends draining; every nonzero result permits one iteration.

For each permitted iteration:

- Read the signed 32-bit index at `0x840` and select a record using the `0x206` stride.
- Read the record’s first byte.
- Only if that byte is nonzero, read the unsigned 16-bit field at record offset `0x02`.
- Only if that field exceeds `1`, call F-fe0f8d21cd8c with the current state byte at `0x00` and the address at record offset `0x04`.
- Reread the 32-bit field at `0x840` and pass that current value to F-a62e1bd60223.
- Store its 32-bit result at `0x840`.
- Ask F-1468513d650e again whether draining continues.

Index transformation occurs even for ineligible records. Eligible-record processing can change the index before the reread, so the transform argument need not equal the selection index.

No nonnegativity check, bounds check, or arithmetic-overflow check is visible. The record offset calculation includes multiplication of a signed 32-bit index by 518. Valid operation requires meaningful, accessible resulting addresses; wrapping or other behavior for overflowing arithmetic is not established as a portable contract.

## Side effects

Direct state writes target `0x20`, `0x83c`, `0x840`, `0x851`, and `0x854`. Shared writes can clear one pending field and, on the first-pending branch, set a flag byte using the separately cached base. Aliasing can make these writes affect other logical fields.

Dependencies may have additional effects. No dependency is assumed to preserve globals, selectors, indices, event fields, or reachable memory. Changes to the shared-base global do not redirect the already cached flag-write address.

The function performs no explicit allocation, release, or ownership transfer. The interior pointer stored at `0x83c` does not establish ownership. The descriptor is temporary local storage; whether F-a10000000007 retains its address is unknown.

There is no visible cleanup, exception handling, or general failure recovery. Dependency results are used as drain decisions, event values, replacement indices, or stored values, without separate error classification. A dependency that does not return prevents subsequent operations.

## Dependencies

Only observed uses are established; opaque IDs imply no API identity.

| Opaque ID | Established use |
|---|---|
| F-ef10ee2809ab | Optional setup with selector from `0x09` and zero |
| F-17d6434f9fa3 | Optional setup with selector from `0x08` and zero |
| F-1a4256178b1e | Setup with selector from `0x01` and `1` |
| F-058705628f93 | Setup with selector from `0x01` |
| F-c1a1c3072914 | Setup with selector from `0x01` |
| F-3f5ace521ed4 | Setup with selector from `0x01` |
| F-1b765582ee90 | First-pending branch call with `1` |
| F-9956f7343fc0 | Per-cycle call with `8` and the captured entry value |
| F-abff49696f96 | Priority event callback receiving state |
| F-1468513d650e | Record-drain continuation decision |
| F-fe0f8d21cd8c | Eligible-record processing |
| F-a62e1bd60223 | Current-index transformation |
| F-a10000000001 | Call with `14` before each pending-related sequence |
| F-a10000000002 | Later call with `14` |
| F-a10000000003 | First-pending branch call with `0`, `2000` |
| F-a10000000004 | Call with state field `0x0c`, or separately with `22` |
| F-a10000000005 | One-time pending-related call with `22` |
| F-a10000000006 | Supplies the 32-bit value stored at `0x854` |
| F-a10000000007 | Receives the descriptor and supplies the event |

## Acceptance examples

These are conditional static-contract examples, not executed tests. Unless stated otherwise, they assume valid storage, stable relevant fields, and normal dependency returns.

1. **Sentinel setup.** If both optional selector examinations observe `0xc8`, neither optional call occurs. All four setup calls using `0x01` still occur.

2. **Competing pending values with a stable shared base.** If both pending fields are nonzero, the first is read and cleared; the second is not read or cleared by this logic. The first branch’s two calls precede its flag write. The first outer cycle performs the identifier-`22` sequence.

3. **Only the second pending value.** If the first read yields zero and the second yields nonzero, the second field is cleared through a fresh base evaluation. There is no flag write or initial first-branch call pair. The first outer cycle still performs the identifier-`22` sequence.

4. **Different observed shared bases.** If the initial global evaluation yields address A, the first-pending read’s evaluation yields B, and the clear’s evaluation yields C, a nonzero value read from B plus `0x04` causes a zero write at C plus `0x04`. After the two branch calls, the flag write targets A. This describes consequences of those observed evaluations, not a concurrency guarantee.

5. **Global mutation during branch calls.** If either first-branch dependency changes the shared-base global, the later flag write still targets the address captured before the first pending read.

6. **Callback and drain match together.** If the event equals both dispatch fields when examined, F-abff49696f96 runs and record draining is skipped. Restart comparison still follows.

7. **Callback followed by restart.** If the event matches the callback field and matches the restart field after the callback returns, the callback completes before the next outer cycle begins.

8. **Ineligible record.** A zero record flag suppresses the 16-bit eligibility read and processing call, but does not suppress index transformation.

9. **Unsigned boundary.** With a nonzero record flag, eligibility value `1` suppresses processing; `2` and `0xffff` permit it.

10. **Index mutation and entry capture.** If record processing changes `0x840`, the transform receives the changed value. If any dependency changes `0x1c`, later per-cycle calls still receive the value captured at entry.

## Uncertainties

The following remain **documented_only**:

- Dependency argument domains, meanings, failure conventions, and termination behavior.
- Whether calls using `14` provide exclusion, atomicity, ordering, or protection against asynchronous changes.
- Whether separate shared-base evaluations identify the same allocation, and the lifetime of each accessed allocation, including the cached flag target.
- Whether `2000` denotes time, or `0x854` stores a timestamp.
- Byte argument signedness and extension.
- Record count, payload format, valid index domain, and index-transform semantics.
- Endianness, alignment requirements, and behavior for invalid storage or overflowing address arithmetic.
- Dependency retention, ownership, release, and mutation behavior.
- Platform-level semantics for concurrent accesses.

Synchronized pending-work consumption and event-driven record processing are plausible intent. Neither interpretation strengthens the established contract.

## Evidence

This document derives from static inspection of the supplied function representation and dependency-ID map. Prior prose and reviewer feedback were treated as claims to check against that representation.

The shared-access distinction is directly supported by a standalone global-pointer capture, separate global evaluations for pending reads and clears, and use of the captured address for the later flag write. The intervening dependency calls occur after the first-pending clear and before that flag write.

Other established claims concern visible widths, offsets, access ordering, call arguments, comparisons, and loop structure. Unavailable dependency behavior is not inferred from names or constants.

No runtime tests have been performed. These acceptance examples do not claim observed execution, concurrency safety, or platform-level memory safety.
