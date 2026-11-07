<!-- Trusted provenance: F-b579a57cbe2c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b579a57cbe2c

## Purpose

F-b579a57cbe2c derives a candidate position from an object’s upper-bound field and a signed displacement, limits that candidate through two ordered signed comparisons, coordinates an embedded component, and conditionally updates the object’s selected-position field.

A changed selection triggers mode-dependent processing and indirect notifications. The routine does not merely update a field: dependency calls can occur even when the selected position ultimately remains unchanged, and callbacks may alter state that controls subsequent behavior.

**Inferred intent:** the fields appear to represent an upper bound, lower bound, current selection, and mode. These descriptive names identify their roles in this routine; they do not establish broader application semantics.

## Interface

### Inputs and result

The routine accepts:

- An object address, referring to readable and writable structured storage.
- A signed displacement, represented as a 32-bit integer in the supplied evidence.

No return value is exposed. Results are observable through object writes, dependency calls, indirect callbacks, and any effects of those callees.

The evidence uses 32-bit integer fields and a 32-bit pointer-oriented layout. The relevant object locations are:

| Byte offset | Width or extent | Role in this contract |
|---|---:|---|
| `0x000` | Pointer-sized, shown as 32 bits | Address of an indirect-call table |
| `0x018` | One inspected byte within a 32-bit field | Mode discriminator |
| `0x01C` | Embedded component; extent unknown | Address supplied to component dependencies |
| `0x8B8` | 32 bits | Selected position |
| `0x8BC` | 32 bits | Lower-bound input |
| `0x8C0` | 32 bits | Upper-bound input |

The mode test uses the field’s low-order byte as represented by the supplied source. The other bits of that field do not participate in the shown comparisons. Mapping that byte to a particular memory byte requires the target’s byte order, which is not independently supplied.

Indirect notifications use the function pointer at byte offset `0x58` within the table reached through the object’s first field. Each notification receives the object address followed by two additional values.

### Preconditions

For a path to complete normally:

- Every object field read or written on that path must be accessible.
- The embedded-component dependencies must accept the address at object offset `0x01C`.
- Any reached indirect dispatch must find a valid table and callable entry at table offset `0x58`.
- The object and its dispatch structure must remain usable after intervening calls.
- The mode-2 processing dependency must respect the lifetime and capacity of the supplied 40-byte temporary region if it accesses that region.

No null check, bounds check on the object allocation, dispatch validation, or explicit exception recovery is shown.

The routine itself does not enforce object lifetime or callback discipline. It is not established that any dependency preserves the object, global values, dispatch table, or selected position.

## Behavior

### 1. Derive the candidate before calling dependencies

The routine first reads the upper-bound field and adds the signed displacement to it. It then reads the upper-bound field again for comparison.

For a representable signed sum, the ordered limiting behavior is:

1. If the second upper-bound value is less than the sum, use that second upper-bound value as the candidate. The lower-bound field is not read on this branch.
2. Otherwise, read the lower-bound field. If the sum is less than that value, use the lower-bound value.
3. Otherwise, retain the sum.

With stable fields, an ordered interval, and no arithmetic overflow, this has the familiar effect of limiting “upper bound plus displacement” to the interval.

That simplification must not replace the actual branch order:

- A positive displacement that exceeds the upper bound selects the upper bound without checking the lower bound.
- An inverted interval is not rejected or normalized.
- The upper bound is read twice; treating those reads as one snapshot is not established.
- The routine does not re-limit the candidate after any dependency runs.

The supplied representation shows signed comparison and 32-bit arithmetic. Behavior when the addition exceeds the signed 32-bit range is not established; wrapping, saturation, and rejection must not be assumed.

### 2. Query and possibly signal the embedded component

The routine calls F-56fd8d6069bf with the embedded-component address.

It compares that dependency’s integer result with the already computed candidate. If they differ, it calls F-9c31c3e85ab1 with:

- The embedded-component address.
- The integer value `1`.

The candidate is not an argument to F-9c31c3e85ab1. The evidence therefore does not establish that this call sets the component’s position to the candidate.

These operations precede the selected-position equality check. Consequently, an invocation can return without changing the selected-position field while still having queried and signaled the embedded component.

### 3. Compare and store the selected position

After the component calls, the routine reads the selected-position field.

- If it equals the candidate, the routine returns immediately.
- Otherwise, the routine writes the candidate to that field and continues.

This comparison observes the field after the preceding dependencies. It is not a comparison against an entry-time snapshot.

The direct selected-position write occurs before mode-specific processing. No rollback of that write is shown on any later path.

### 4. Capture two notification values

After storing the selection, the routine loads two opaque 32-bit values from static storage:

1. A **shared notification value**.
2. A **final notification value**.

These are captured before the first mode test. Subsequent calls use the captured values even if intervening callees modify their backing storage.

Other notification values are obtained at their respective call sites rather than all being captured here. The evidence supplies neither their contents nor their external meanings.

### 5. Perform mode-2 processing when applicable

The routine inspects the mode byte.

If it equals `2`, the routine calls F-bfda5fbfe0b0 with:

- The object address.
- A pointer to a 40-byte temporary region.
- The selected-position field, read at this point.
- The integer value `3`.

No initialization of the temporary region is shown. This routine does not subsequently inspect its contents.

The dependency’s integer return controls the next action:

- **Zero:** call F-97a072757e44 with the object, then call F-52cae007644e with the object and a fresh read of the selected-position field.
- **Nonzero:** perform one indirect notification using the captured shared notification value and an opaque mode-2 notification payload.

The selected position passed to F-52cae007644e is read after F-97a072757e44 returns. It need not equal either the original candidate or the value passed to F-bfda5fbfe0b0.

Following either mode-2 outcome, the mode byte is read again:

- If it remains `2`, processing proceeds to the two-notification sequence described below.
- If it no longer equals `2`, the routine classifies the current mode as described next.

Thus, entry into mode-2 processing does not permanently determine the remaining notification sequence.

### 6. Select the terminal notification sequence

For an initial mode other than `2`, or after mode-2 processing changes the mode away from `2`, the routine tests the current mode.

- Mode `1` selects the two-notification sequence.
- Otherwise, a subsequent mode test for `3` selects the four-notification sequence.
- Any other observed mode returns without a terminal notification sequence.

No additional dependency call occurs between these classification tests, but the source expresses separate mode reads.

#### Two-notification sequence

This sequence runs for mode `1` and for mode `2` that remains `2` after its processing.

It performs, in order:

1. An indirect call with an opaque pair-specific first argument value and an opaque pair-specific payload.
2. An indirect call with the previously captured final notification value and a second opaque pair-specific payload.

Once this sequence begins, the routine does not recheck the mode between the calls.

#### Four-notification sequence

Mode `3` performs, in order:

1. An indirect call with the captured shared notification value and the first mode-3 payload.
2. An indirect call with the captured shared notification value and the second mode-3 payload.
3. An indirect call with the captured shared notification value and the third mode-3 payload.
4. An indirect call with the captured final notification value and the fourth mode-3 payload.

The four payloads occupy distinct static-storage locations; their numerical values are unknown and are not established to be different.

The mode is not rechecked between these calls.

### 7. Resolve dispatch independently for each notification

The routine obtains the indirect-call target through the object’s dispatch table for each notification. It does not retain one dispatch target for an entire sequence.

An earlier callback can therefore affect which target a later callback uses by changing the object’s dispatch structure, provided execution remains valid. The routine does not guard against such changes.

## Side effects

### Established direct effects

The routine directly:

- Reads the bound, selected-position, mode, and dispatch-related fields as described above.
- Writes the candidate to the selected-position field when the post-component-call equality check fails.
- Exposes the embedded-component address to two dependencies.
- Exposes the object address to processing dependencies and indirect callbacks.
- Exposes a 40-byte temporary region to the mode-2 processing dependency.

No direct dynamic allocation, deallocation, object destruction, or explicit release is shown.

### Effects delegated to callees

The dependencies and callbacks may have additional effects not characterized by the supplied evidence. In particular, this contract does not assume that they preserve:

- The selected-position or mode fields.
- The bounds.
- Static notification storage.
- The dispatch table.
- Object or component lifetime.

The direct selection write remains observable before later calls. A later failure does not cause a shown restoration of the previous selection.

### Ownership and lifetime

Passing the object, embedded-component, or temporary-region address does not by itself establish an ownership transfer.

The temporary region belongs to this invocation’s stack lifetime. The evidence does not establish whether F-bfda5fbfe0b0 retains its address or whether such retention is permitted by a separate contract. No retained-pointer use after return can be justified from this routine alone.

## Dependencies

| Opaque ID or dispatch | Arguments supplied | Result used here |
|---|---|---|
| F-56fd8d6069bf | Embedded-component address | Integer compared with the candidate |
| F-9c31c3e85ab1 | Embedded-component address, `1` | No result consumed |
| F-bfda5fbfe0b0 | Object, 40-byte temporary region, current selected position, `3` | Zero versus nonzero |
| F-97a072757e44 | Object | No result consumed |
| F-52cae007644e | Object, selected position read after the preceding call | No result consumed |
| Indirect table slot `0x58` | Object, notification value, notification payload | No result consumed |

The purposes of these dependencies are not established by their call shapes. Descriptions such as validation, refresh, synchronization, or event emission would be inferred intent, not proven dependency contracts.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Unless stated otherwise, assume stable readable fields, valid dispatch, returning callees, no unmentioned mutations, and representable signed arithmetic.

### Candidate limiting

With lower bound `10` and upper bound `20`:

| Displacement | Candidate |
|---:|---:|
| `-5` | `15` |
| `-30` | `10` |
| `0` | `20` |
| `7` | `20` |

For displacement `7`, the upper comparison selects `20` without reading the lower-bound field.

With inverted bounds—lower bound `30`, upper bound `20`:

- Displacement `1` selects `20` through the upper branch.
- Displacement `0` reaches the lower comparison and selects `30`.

An implementation that normalizes the interval or always checks both bounds would not match these cases.

### Unchanged selection with component signaling

If the candidate and selected position are both `15`, but F-56fd8d6069bf returns `14`:

- F-9c31c3e85ab1 is called with the embedded-component address and `1`.
- Assuming that call leaves the selected position at `15`, the routine returns.
- No direct selection write or mode-specific call follows.

### Equality uses post-call state

If the selected position initially differs from the candidate, but a component dependency changes it to the candidate, the routine returns at the equality check. The entry-time mismatch does not force a write or notification.

### Changed selection in mode 1

If the post-component-call selected position differs from the candidate and the mode is `1`:

- The candidate is stored.
- No mode-2 processing dependency is called.
- Exactly the two terminal notifications occur, assuming both return normally.

### Mode 2, zero processing result

If the mode is `2`, F-bfda5fbfe0b0 returns zero, and mode remains `2`:

- F-97a072757e44 runs.
- F-52cae007644e runs with a fresh selected-position read.
- The two-notification sequence follows.

If F-97a072757e44 changes the selected position to `17`, F-52cae007644e receives `17`.

### Mode 2, nonzero processing result and mode transition

If F-bfda5fbfe0b0 returns nonzero:

- The mode-2 notification occurs first.
- If that callback leaves mode `2`, the two-notification sequence follows.
- If it changes mode to `3`, the four-notification sequence follows.
- If it changes mode to an unrecognized value, no terminal sequence follows.

The previously stored selection is not directly undone in any of these cases.

### Dispatch mutation during notifications

If the first notification changes the object’s dispatch-table pointer, the next notification obtains its target through the changed pointer. Acceptance must not require all notifications to use the first target.

## Uncertainties

The following remain **documented_only**:

- Signed-addition behavior outside the representable 32-bit range.
- Target byte order and independently verified ABI details.
- The actual values, types beyond their shown storage width, and meanings of notification constants.
- Dependency implementations, side effects, ownership rules, and failure behavior.
- The required layout and meaning of the 40-byte temporary region.
- Whether concurrent or reentrant mutation is supported.
- Whether bounds are normally ordered or validated elsewhere.
- Whether any dependency can destroy the object or invalidate its dispatch structure.

No explicit error result is exposed. A nonzero result from F-bfda5fbfe0b0 selects a notification branch; it is not established to mean an error. Faults, nonlocal exits, and failures inside dependencies have no shown recovery path.

## Evidence

**Established static evidence:** the supplied routine shows the field offsets, signed comparisons, ordered dependency calls, one conditional direct selection write, mode rereads, 40-byte temporary region, zero/nonzero branch, and per-notification indirect dispatch described above.

**Inferred intent:** position limiting, selection management, and notification-oriented processing are plausible interpretations, not independently established semantics.

**Runtime evidence:** none. No runtime tests, dependency executions, or callback traces have been performed.
