<!-- Trusted provenance: F-92e1be48d3ce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-92e1be48d3ce

## Purpose

F-92e1be48d3ce conditionally updates a record obtained from F-7fba0909c452. It accepts a selector, a signed quantity, and an additional integer payload.

The established behavior has three outcomes:

- If the dependency supplies no record, return without further action by this function.
- If the selector is recognized and the quantity is nonnegative, write a configuration into the record, substituting a selector-dependent default when the quantity is zero.
- Otherwise, conditionally record an error value, preserving an existing nonzero error.

The apparent intent is to configure an operation on an implicit current object while maintaining a first-error-wins status field. The meanings of the operation, quantity, payload, and internal state value are not established.

## Interface

### Inputs and result

The function takes three integer inputs:

| Input | Contract |
|---|---|
| Selector | Compared for exact equality with one fixed value and three values obtained from shared state. |
| Quantity | Interpreted as signed. Negative values take the error path; zero requests default selection; positive values are stored unchanged on the success path. |
| Payload | Stored unchanged on the success path. No local validation or interpretation is visible. |

The represented interface returns no value. Callers cannot distinguish the outcomes through a return value.

The supplied representation uses integer fields and integer-indexed record accesses. This contract describes the apparent 32-bit integer layout: each field is four bytes, and signed quantity comparisons use a 32-bit signed interpretation. Exact ABI confirmation is **documented_only** because no independent type or target-layout evidence is supplied.

### Implicit inputs

The function obtains or reads:

- A record pointer returned by F-7fba0909c452.
- A shared integer used to form recognized selector values, called the **selector base** here.
- A shared integer used as the error value for recognized selectors with negative quantities, called the **configured error** here.
- The record’s existing error field, but only when an error candidate is nonzero.
- A record source field, but only after the first four success-path writes.

These descriptive names do not imply application-level meanings.

### Record layout

Offsets below are relative to the pointer returned by the dependency, under the four-byte field interpretation.

| Byte offset | Width | Role in this function |
|---|---:|---|
| `0x000` | 32 bits | Existing error; conditionally written |
| `0x170` | 32 bits | State field; assigned `3` on success |
| `0x174` | 32 bits | Stored selector |
| `0x178` | 32 bits | Stored payload |
| `0x17C` | 32 bits | Destination of a late record-field copy |
| `0x18C` | 32 bits | Stored effective quantity |
| `0xAD8` | 32 bits | Source of the late record-field copy |

For a successful update, these accesses require valid writable destinations through byte `0x18F` and a readable four-byte source at `0xAD8`. A conventional contiguous record supporting all accesses therefore extends through byte `0xADB`, requiring at least `0xADC` bytes from its beginning.

No record length, alignment, lifetime, or access-permission checks are visible. A non-null dependency result must already support whichever accesses the selected path performs.

## Behavior

### 1. Obtain the record

F-7fba0909c452 is called once, before this function reads the selector base, configured error, or record fields.

If it returns null, this function returns immediately. It does not locally validate the inputs, read the shared selector values, or access record memory.

This early return does not establish that the entire invocation is side-effect-free: the dependency’s own behavior is unknown.

### 2. Classify the selector

With a non-null record, the selector is recognized if it matches any of these alternatives:

- The fixed value `0x1400`, decimal `5120`.
- A value read from the selector base.
- A separately read selector-base value plus decimal `10`.
- A further separately read selector-base value plus decimal `4`.

The two derived values are retained for later default selection.

The visible evaluation order is:

1. Compare the selector with `0x1400`.
2. Read the selector base for direct equality.
3. Read the selector base again and form the value plus `10`.
4. Compare with that derived value.
5. Read the selector base again and form the value plus `4`.
6. Compare with that derived value.

Thus, the representation does not establish one shared snapshot of the selector base. With stable shared state, the recognized set is simply the fixed value, the base, the base plus `10`, and the base plus `4`. If the base can change between reads, the direct and derived alternatives can reflect different observations.

The fixed selector does not short-circuit these classification-time reads.

### 3. Select configuration or error handling

For an unrecognized selector:

- The error candidate is fixed at `0x500`, decimal `1280`.
- The configured-error global is not read by this function.
- The quantity does not affect the outcome.
- Processing continues to conditional error recording.

For a recognized selector:

- The configured error is read.
- The quantity is then checked for signed nonnegativity.
- A negative quantity goes to conditional error recording using the configured error.
- A zero or positive quantity goes to configuration storage.

The configured error is therefore read for recognized selectors even when configuration succeeds. It is not used to gate a successful update.

### 4. Resolve a zero quantity

Positive quantities are stored unchanged.

When the quantity is zero, default selection uses this precedence:

| Condition | Effective quantity |
|---|---:|
| Selector equals `0x1400` | `3` |
| Otherwise, selector equals a newly read selector base | `6` |
| Otherwise, selector equals either retained derived value | `12` |
| Otherwise | `0` |

The direct selector-base comparison in this stage is a new read, not a reuse of the value used for initial direct classification. It is reached only when the selector is not the fixed value.

The derived comparisons reuse the earlier base-plus-`4` and base-plus-`10` values; they do not recompute either sum.

With a stable base, every recognized selector receives one of the nonzero defaults. If shared state changes, a selector recognized solely through the earlier direct-base comparison can fail all default-selection comparisons and leave the effective quantity at zero. That is not treated as an error by this function.

If selector alternatives overlap, the fixed-value default has priority, followed by the fresh direct-base comparison, followed by either derived match.

### 5. Store the configuration

For a recognized selector with a nonnegative quantity, the visible access order is:

1. Write the original selector to `0x174`.
2. Write the payload to `0x178`.
3. Write the effective quantity to `0x18C`.
4. Write `3` to `0x170`.
5. Read the source field at `0xAD8`.
6. Write the value just read to `0x17C`.
7. Return.

The source value is not read before the other configuration writes.

The existing error field at `0x000` is neither inspected nor cleared on this path. A successful configuration can coexist with an earlier nonzero error.

There is no visible transactional mechanism, rollback, or atomic multi-field publication. The listed ordering is established by the supplied representation; cross-thread visibility and machine-level synchronization guarantees are not established.

### 6. Conditionally record an error

The error candidate is:

- `0x500` for an unrecognized selector.
- The configured error for a recognized selector whose quantity is negative.

If the candidate is zero, the function returns without reading or writing the record’s existing error field.

If the candidate is nonzero, the function reads the error field at `0x000`:

- If the field is zero, write the candidate there.
- If the field is nonzero, preserve it.

No configuration fields are accessed on this path. In particular, the source field at `0xAD8` is not read.

A negative configured error is still a nonzero candidate and can be stored. No restriction to positive error values is visible.

## Side effects

### Local record changes

A successful configuration performs five record writes and one record-source read, in the order specified above. It leaves the error field untouched.

An error path performs either:

- No record access, when the candidate is zero;
- One error-field read without a write, when an error already exists; or
- One error-field read followed by one error-field write, when the field was zero.

The first-error-wins behavior is local and unsynchronized. It is not evidence of an atomic compare-and-set operation.

### Shared state

This function visibly reads the selector base and, conditionally, the configured error. It does not directly write either shared value.

It must not be assumed that F-7fba0909c452 preserves those values. All local shared-state reads happen after the dependency returns.

### Ownership and lifetime

No explicit allocation, deallocation, reference-count update, ownership transfer, or pointer retention is visible in this function.

That observation does not establish the ownership policy of the dependency or returned record. The function relies on the returned pointer remaining usable for its subsequent accesses. Whether the dependency allocates, borrows, acquires, or otherwise manages that record is **documented_only**.

## Dependencies

### F-7fba0909c452

- Called exactly once per invocation.
- Receives no explicit arguments in the supplied representation.
- Supplies the pointer used for every subsequent record access.
- A null result suppresses all further local processing.
- A non-null result is used without further validation.

Unknown dependency properties include global-state preservation, allocation behavior, ownership, lifetime guarantees, synchronization, and its own failure reporting.

No other function calls are visible.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, assume a valid record, stable selector base, representable derived sums, and no concurrent mutation.

### Null dependency result

For any selector, quantity, and payload, if F-7fba0909c452 returns null:

- No local shared-state or record accesses follow.
- The function returns without a result value.
- Dependency side effects remain unspecified.

### Fixed selector with default quantity

Given selector `0x1400`, quantity zero, payload `37`, and source field `0xAD8` containing `91`:

- Store `0x1400` at `0x174`.
- Store `37` at `0x178`.
- Store `3` at `0x18C`.
- Store `3` at `0x170`.
- Then read `0xAD8` and copy its value to `0x17C`.

Any existing error remains unchanged. The fixed default wins even if the selector base also equals `0x1400`.

### Direct-base and derived defaults

Given a stable selector base of `2000` and quantity zero:

- Selector `2000` produces effective quantity `6`.
- Selector `2004` produces effective quantity `12`.
- Selector `2010` produces effective quantity `12`.

Each successful case also stores the selector, payload, state, and late source-field copy.

### Positive quantity with a pre-existing error

Given a recognized selector, quantity `25`, and a nonzero existing error:

- Store quantity `25`, without substituting a default.
- Perform all success-path writes.
- Do not read or change the existing error.

### Unrecognized selector

Given stable base `2000` and selector `2001`:

- Use error candidate `0x500`, regardless of the quantity.
- Do not read the configured error.
- If the existing error is zero, write `0x500`.
- Otherwise preserve the existing error.
- Do not access configuration fields or the late-copy source.

### Recognized selector with a negative quantity

Given a recognized selector, quantity `−1`, and configured error `17`:

- If the existing error is zero, store `17`.
- If it is nonzero, preserve it.
- Perform no configuration updates.

If the configured error is instead zero, return without even reading the existing error field.

### Changed base during zero-default resolution

Assume the selector is recognized through the initial direct-base equality, but is neither `0x1400` nor either retained derived value. If the later direct-base read no longer equals the selector:

- The effective quantity remains zero.
- Configuration still succeeds.
- Zero is stored at `0x18C`.

This case is conditional on shared-state changes being possible; the surrounding synchronization model is unknown.

## Uncertainties

- **Integer layout — documented_only:** Four-byte integer fields are the apparent model. The supplied material does not independently establish the ABI.
- **Arithmetic overflow — documented_only:** Adding `10` or `4` near the signed maximum may exceed the signed range. The evidence does not establish wrapping, trapping, or another defined outcome. Boundary acceptance cases require confirmed arithmetic semantics.
- **Concurrency — documented_only:** Multiple selector-base reads are visible, but no locking, volatile-access contract, or memory-order guarantees are supplied.
- **Aliasing — documented_only:** No proof establishes that shared values cannot overlap record storage. No stronger non-aliasing guarantee is imposed here.
- **Dependency behavior — documented_only:** The dependency’s effects and returned-pointer ownership are unknown.
- **Domain meaning — inferred intent:** Labels such as configuration, state, and error describe observed usage rather than confirmed application concepts.
- **Failure during memory access — documented_only:** No local recovery is visible for an invalid record, inaccessible memory, or interrupted partial update.

## Evidence

Established static evidence consists solely of the supplied function representation and dependency mapping. It supports the branch conditions, constants, access indices, local access order, single dependency call, conditional error preservation, and absence of a returned value described above.

The four-byte offsets are derived from the represented integer indexing under the stated 32-bit layout assumption. Behavioral names are explanatory, not source-defined API guarantees.

No implementation was executed, and no runtime tests have been performed. The acceptance examples are static consequences under their stated preconditions, not observed results.
