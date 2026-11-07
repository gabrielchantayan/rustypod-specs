<!-- Trusted provenance: F-ecde43d0a2b1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ecde43d0a2b1

## Purpose

F-ecde43d0a2b1 classifies a height derived from a cursor and a record reference, conditionally performs a record transition, and checks the cursor against a limit.

The visible outcomes are:

- **Negative height:** request a diagnostic, then write status 99 and return 1.
- **Zero height:** skip the record transition and perform the limit check.
- **Positive height:** perform the record transition, then perform the limit check.
- **Cursor below the limit:** clear a one-byte context flag and return 0.
- **Cursor at or above the limit:** conditionally request a diagnostic, then write status 98 and return 1.

The positive-height transition precedes the limit check. A failure therefore does not imply that no writes occurred. The body performs no rollback, but the final contents of previously written locations are not guaranteed: later aliased writes or diagnostic side effects may overwrite them.

“Profile,” “segment,” and “storage cursor” describe inferred application intent. The established contract concerns addresses, field accesses, arithmetic, calls, and branch outcomes.

## Interface

### Argument and result

The function accepts one address-valued argument identifying a mutable context. The supplied representation uses 32-bit addresses and a 32-bit return value.

| Return value | Established meaning |
|---|---|
| 0 | The body selected the branch where the unsigned cursor was strictly below the unsigned limit, and performed the success-flag store. |
| 1 | The body selected negative-height failure or cursor-limit failure, and performed the corresponding status store. |

These outcomes assume that all required memory accesses complete and every invoked dependency returns normally.

The result is not the detailed status. The body explicitly stores status only on failure. On success, it performs no store addressed as the context status field; this is **not** a guarantee that the underlying status bytes retain their previous value.

### Context fields

Offsets are in bytes from the context address.

| Offset | Access width | Role in this function |
|---|---:|---|
| +0x28 | 32 bits | Limit for the unsigned cursor comparison. |
| +0x2c | 32 bits | Cursor and source of the new-record address. |
| +0x30 | 32 bits | Explicit failure-status destination. |
| +0x58 | 16 bits | Counter incremented on the positive-height path. |
| +0x5b | 8 bits | Flag explicitly cleared on success. |
| +0x5c | 32 bits | Current-record address. |

There is no explicit access addressed as context +0x5a. That does not establish preservation of those bytes under arbitrary aliasing or dependency behavior.

### Record fields

Offsets are in bytes from whichever record address the relevant access selects.

| Offset | Access width | Role in this function |
|---|---:|---|
| +0x08 | 32 bits | Reference used in height calculation; later initialized through the current-record address. |
| +0x10 | 32 bits | Height destination; also receives zero through the selected new-record address. |
| +0x14 | 32 bits | Value supplied to the positive-height diagnostic. |
| +0x1c | 32 bits | Link destination in the record selected for closing. |

The cursor advances by 0x20 bytes on the positive-height path. This establishes a 32-byte stride, not a complete structure size or initialization guarantee.

### Preconditions and ownership

The function provides no visible null check, alignment check, allocation, capacity growth, or release operation.

The caller must provide readable or writable storage for every access reached on the selected path. In particular:

- The initial record is read before the limit check.
- Positive-height record writes occur before the limit check.
- A later numerical limit failure does not protect earlier writes from invalid storage.
- A configuration object used after a diagnostic must still be accessible, including the object whose address was captured at entry.

No ownership transfer is established. Whether the diagnostic dependency changes ownership, frees storage, changes mappings, or retains references is documented_only.

Nonoverlap is not enforced. Claims about particular fields retaining their values require both absence of relevant aliasing and absence of dependency or external mutations.

## Behavior

### Initial capture and height classification

The function first captures the address held by a global configuration reference. That address is retained for a possible overflow-diagnostic threshold lookup.

It obtains the current-record address from context +0x5c. It calculates height from the 32-bit cursor at context +0x2c minus the 32-bit reference at the initial record’s +0x08, followed by a signed right shift of two bits.

The represented subtraction and shifted result are signed 32-bit operations. The resulting height selects the negative, zero, or positive path.

This must not be replaced by unsigned subtraction or by division with truncation toward zero. With arithmetic right shift, a difference of −1 produces height −1, whereas truncation-toward-zero division by four would produce zero.

Exact behavior for signed subtraction overflow and the target’s signed-shift rules is not independently established. Examples below avoid subtraction overflow and explicitly condition negative fractional-unit behavior on arithmetic right shift.

The representation establishes the data dependencies of the height expression, but not a separate machine-level ordering guarantee between its operand reads.

### Negative height

A negative height causes an unconditional call to F-4457a9a44a68 for the negative-height diagnostic. There is no threshold test before this call.

If the call returns normally, the body:

1. Writes the 32-bit value 99 at context +0x30.
2. Returns 1.

This path does not execute the positive-height transition, counter increment, final limit comparison, or explicit success-flag store.

Those are statements about operations omitted by the body. They do not guarantee unchanged record or flag bytes: the dependency may mutate them, and the final status destination may overlap other logical objects.

### Zero height

A zero height skips the positive-height diagnostic and the entire record transition. It proceeds directly to the final cursor-limit comparison.

Thus the body performs no record-addressed write, cursor advancement, or counter increment on this path. Zero height is not itself an error.

Under arithmetic right shift, positive cursor/reference differences of 1, 2, or 3 also produce zero height. Equality of cursor and reference is sufficient, but not necessary, for this path.

### Positive-height diagnostic

For positive height, the function reads the **current** global configuration reference and examines the signed 32-bit threshold at that object’s +0x28.

If the threshold is greater than 5, it calls F-4457a9a44a68 with a positive-height diagnostic and these associated values:

- The record address originally used for height calculation.
- The 32-bit value read from that record at +0x14.
- The previously calculated height.

A threshold of exactly 5 does not enable this call.

This lookup uses the global reference again rather than the address captured at entry. The two addresses may differ; the function does not establish their equivalence.

After the optional call, the body reloads the current-record address from the context. It does **not** recalculate height. A dependency that changes the current-record address or cursor can therefore cause the transition to combine the original height with newly read state.

### Positive-height transition

The following access sequence is established by the visible body:

1. Read the current-record address from context +0x5c. Retain this address as the record selected for closing.
2. Write the calculated height at that record’s +0x10.
3. Read the cursor from context +0x2c. Retain this value as the selected new-record address.
4. Write the selected new-record address to context +0x5c.
5. Write the selected new-record address plus 0x20 to context +0x2c.
6. Write 32-bit zero at the selected new-record address +0x10.
7. Read the then-current values of context +0x5c and context +0x2c, and store the latter at +0x08 of the record addressed by the former.
8. Read the then-current context +0x5c and store it at +0x1c of the record selected for closing.
9. Read, increment, and store the counter at context +0x58 using a 16-bit destination.

Within step 7, the representation does not independently establish the order of the two operand reads.

The distinction between retained addresses and reread context fields matters under aliasing. An earlier write may change a context field before a subsequent read. Consequently, the +0x08 initialization need not target the address originally selected from the cursor.

The body has no separate initialization operation for the selected new record’s +0x14 or +0x1c fields. Their underlying bytes are not guaranteed untouched: another destination may alias them.

The counter update has no saturation check or counter-overflow failure branch. Under the represented low-16-bit narrowing behavior, 0xffff increments to 0x0000. The cursor addition likewise has no explicit overflow check; exact overflowing signed/address arithmetic behavior remains documented_only.

### Final cursor-limit decision

The zero-height and positive-height paths compare the then-current context cursor at +0x2c with the limit at +0x28 as unsigned 32-bit values.

If cursor is strictly less than limit, the body:

1. Writes one-byte zero to context +0x5b.
2. Returns 0.

It makes no explicit status-field store on this branch. Preservation of an existing status requires that earlier writes did not alias it and that no dependency or external actor changed it.

If cursor equals or exceeds limit, the function reads the signed 32-bit threshold at +0x28 of the configuration object whose address was captured at entry. A threshold greater than zero enables an overflow-diagnostic call to F-4457a9a44a68.

After that call returns, or immediately if it is skipped, the body writes 98 at context +0x30 and returns 1.

The body does not repeat the limit comparison after the diagnostic. A dependency that repairs the cursor or limit does not change the already selected failure path. It may nevertheless change or undo previous record and context mutations.

Capturing the configuration address does not snapshot its contents. The overflow threshold is read later from that captured object.

## Side effects

### Explicit operations by path

| Path | Operations performed by the body |
|---|---|
| Negative height | Negative diagnostic; status store of 99; return 1. |
| Zero height, below limit | Success-flag store; return 0. |
| Zero height, at or above limit | Optional overflow diagnostic; status store of 98; return 1. |
| Positive height, below limit | Optional positive diagnostic; record transition; counter increment; success-flag store; return 0. |
| Positive height, at or above limit | Optional positive diagnostic; record transition; counter increment; optional overflow diagnostic; status store of 98; return 1. |

This table describes executed operations, not guaranteed independent final field values.

### Aliasing and preservation

A record-addressed write can overwrite context bytes, another record, or configuration data. A context-addressed write can likewise affect an overlapping logical object.

Accordingly:

- Success does not unconditionally preserve status.
- Failure does not unconditionally preserve the flag.
- Skipping an explicit field operation does not guarantee preservation of the underlying bytes.
- A write earlier in the transition does not guarantee that its value survives later stores.
- “No initialization operation” does not mean “unchanged memory.”

Under valid nonoverlapping storage, no external mutation, and diagnostics that preserve the relevant state, the ordinary field-level interpretation applies: success leaves status untouched, failure leaves the flag untouched, and only the listed record fields are written.

### Rollback and diagnostics

The body performs no rollback after a limit failure. It does not restore the cursor, counter, links, or heights.

This is narrower than saying the positive-height mutations remain in place. An overflow diagnostic runs after those operations and may alter or undo them. Aliased later stores, including the final status store, may also overwrite earlier results.

There can be at most two diagnostic calls in one execution: a positive-height diagnostic followed by an overflow diagnostic. Negative height causes one diagnostic and bypasses the other paths.

The dependency’s return value is unused. All subsequent behavior is conditional on normal return. No synchronization or transactional protection is visible.

## Dependencies

### F-4457a9a44a68

| Diagnostic role | Condition | Associated values |
|---|---|---|
| Negative height | Calculated signed height below zero | No additional values visible. |
| Positive height | Height above zero and current configuration threshold above 5 | Initial record address, its +0x14 value, calculated height. |
| Overflow | Unsigned cursor at or above limit and captured-object threshold above zero | No additional values visible. |

The diagnostic categories are established; exact output wording, formatting, destination, and emission success are not specified here.

The dependency must not be assumed to preserve globals, context fields, records, storage lifetimes, or ownership.

### Global configuration reference

There are two distinct uses:

- Capture the referenced object’s address at entry for the later overflow threshold.
- Reload the current reference for the positive-height threshold.

Both thresholds are signed 32-bit reads at object +0x28. Replacing the global reference does not replace the captured address. Mutating the captured object can affect the later overflow threshold.

## Acceptance examples

Unless stated otherwise, these examples assume valid, suitably aligned, nonoverlapping storage; no arithmetic overflow; no external mutation; and diagnostics that return without changing relevant state.

### Positive height with space

With cursor 0x2000, record reference 0x1ff0, limit 0x2100, and counter 7, height is 4.

The selected old record receives height 4 and a link to 0x2000. Context current-record becomes 0x2000 and cursor becomes 0x2020. The new record receives height zero and reference 0x2020. Counter becomes 8.

The final comparison succeeds, the flag is cleared, and the return is 0. Under these nonoverlap and preservation assumptions, an existing status remains unchanged.

### Positive height reaching the limit

With the same inputs except limit 0x2020, the transition occurs before equality selects failure.

The body optionally requests an overflow diagnostic, writes status 98, and returns 1. Under the example’s assumptions, the transition results remain present. Without those assumptions, only the executed operations and absence of body rollback are established.

### Success overwriting status through overlap

This example deliberately permits context/record overlap while keeping every accessed location valid.

Let:

- Context address be 0x1000.
- Current-record address be 0x1014.
- Record reference at 0x101c be 0x1ff0.
- Cursor be 0x2000.
- Limit be 0x2100.
- Diagnostics be disabled.

Height is 4. The old-record link destination is 0x1014 + 0x1c = 0x1030, which is also context +0x30.

The link operation stores 0x2000 into those bytes. The advanced cursor is 0x2020, so the function clears the flag and returns 0. Status bytes nevertheless contain 0x2000, despite no explicit success-path status store.

### Zero and negative fractional differences

Equal cursor and record reference produce zero height. If cursor is below limit, the body clears the flag and returns 0 without performing the transition.

Under arithmetic right shift, differences of +1 through +3 also take that path.

A difference of −1 instead produces height −1: the function requests the negative diagnostic, writes status 99, and returns 1 without checking the limit.

### Diagnostic changes state

If an enabled positive-height diagnostic changes the current-record address and cursor, the transition uses the reloaded record and changed cursor but retains the original height.

If an enabled overflow diagnostic subsequently restores earlier record contents or changes the cursor to below the limit, the function still writes status 98 and returns 1. There is no repeated comparison, and no guarantee that the earlier transition values survive.

### Counter rollover

With counter 0xffff on a positive-height transition, represented low-16-bit narrowing yields 0x0000. Counter rollover alone selects no error branch.

## Uncertainties

The following are documented_only:

- Exact machine behavior for overflowing signed subtraction, signed right shift, overflowing cursor addition, and narrowing outside the signed 16-bit range.
- Complete structure sizes, alignment requirements, and fields outside the listed accesses.
- Whether callers guarantee nonoverlap or four-byte cursor/reference differences.
- Diagnostic internals, output destination, failure behavior, ownership effects, and normal return.
- Storage validity after diagnostic calls.
- Application-level meanings of the counter, flag, and status.
- Concurrency and asynchronous-mutation guarantees.
- Machine-level read ordering where the supplied expression does not establish it.

No runtime tests have been performed.

## Evidence

**Established static evidence:** field offsets and access widths; signed height classification; zero-height bypass; diagnostic conditions; entry-captured versus reloaded configuration references; current-record reload before transition; ordered stores and later context rereads; 0x20 cursor advancement; unsigned strict limit comparison; failure status stores of 99 and 98; return values 0 and 1; success-only explicit flag clearing; and absence of body rollback.

**Derived consequence of those accesses:** aliasing can overwrite logically unrelated fields, including status on success. Executed stores therefore cannot be treated as unconditional final-state or preservation guarantees.

**Inferred intent:** closing a profile segment, beginning another segment, counting completions, and reporting storage exhaustion.

**Not established:** dependency behavior, external ownership rules, nonoverlap invariants, complete initialization, exceptional arithmetic semantics, or runtime correctness.
