<!-- Trusted provenance: F-2b1c5e575dcc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2b1c5e575dcc

## Purpose

F-2b1c5e575dcc updates two independent sets of interval-accounting fields in a caller-supplied state object, using a caller-supplied event record.

The visible behavior supports the following interpretation:

- Channel A starts on event `0x10` and finishes on events `0x11`, `0x0E`, or `0x0F`.
- Channel B starts on event `0x12` and finishes on events `0x13`, `0x0E`, or `0x0F`.
- Starting a channel stores an event value and increments a start counter.
- Finishing an active channel adds the difference between the current event value and the saved value to an accumulator, then clears the saved value.

These channel names and the interval-accounting interpretation are descriptive, not established domain terminology. The source establishes arithmetic and state transitions, but not whether the event value represents time, a sequence number, or another quantity.

Zero is the inactive sentinel for each saved-value field. Consequently, recording a start value of zero increments the relevant counter but does not leave that channel distinguishably active.

## Interface

### Inputs and result

The function receives two inputs:

1. **State base:** an address-like value identifying the state storage to update.
2. **Event record:** a pointer to a record containing an event discriminator and an event value.

There is no explicit return value. All defined results are observable through reads and writes to caller-supplied storage.

The supplied representation uses integer-based address arithmetic for the state base and integer-element indexing for the event record. It does not independently establish the calling convention, pointer width, or portability of that representation.

### Referenced storage

The visible layout uses four-byte integer slots at these byte offsets:

| Storage | Byte offset | Contract role |
|---|---:|---|
| Event record | `0x00` | Event discriminator |
| Event record | `0x08` | Event value used by starts and finishes |
| State | `0x58` | Channel A accumulator |
| State | `0x5C` | Channel A start counter |
| State | `0x60` | Channel A saved value; zero means inactive |
| State | `0x64` | Channel B accumulator |
| State | `0x68` | Channel B start counter |
| State | `0x6C` | Channel B saved value; zero means inactive |

The event slot at offset `0x04` is not accessed. No interpretation of it is required.

The state field at `0x64` is identified by its byte position; its source offset is expressed in decimal, but it belongs to the same contiguous four-byte field layout.

### Width and arithmetic qualification

The supplied source presents these fields as four-byte integer quantities, including explicit four-byte clearing stores. It uses signed integer expressions for comparisons, incrementing counters, subtracting saved values, and adding differences to accumulators.

For executions whose arithmetic is representable in the applicable signed integer model, the numerical behavior is specified below. Overflow, underflow, and wraparound behavior are not established by the supplied source alone. In particular, this contract does not promise modulo-\(2^{32}\) counter increments or duration accumulation.

The byte order of stored integers is not established.

### Preconditions and ownership

For an execution to complete normally:

- Every location actually read must be accessible and suitably aligned for the target’s integer accesses.
- Every location actually written must be writable.
- Address calculations must identify valid storage under the target execution model.
- Concurrent or asynchronous changes must not be assumed absent unless the caller provides that guarantee.

The function contains no pointer, bounds, alignment, or arithmetic validation. Unrecognized events do not cause an explicit error.

No storage is allocated, released, or transferred. The caller retains ownership of both inputs. There is no visible retention of either address beyond the call.

The interface does not visibly require the event record and state storage to be disjoint. Overlap can affect behavior, especially because the discriminator is read twice. The ordinary event descriptions below assume stable, non-overlapping storage unless stated otherwise.

## Behavior

### Execution structure

Processing has two stages: channel A handling followed by channel B handling.

The event discriminator is read at entry. Unless a particular channel A start condition returns immediately, the discriminator is read again after channel A handling. Channel B selection uses this second read, not necessarily the value observed at entry.

This distinction matters if an intervening state write aliases the event discriminator, or if another agent changes the event record. A specification that reads the discriminator once and reuses it throughout would not preserve the visible access structure.

### Channel A start: initial discriminator `0x10`

The function reads the channel A saved value.

If that value is nonzero, the function returns immediately:

- The event value is not needed.
- The channel A counter and accumulator are unchanged.
- The discriminator is not reread.
- Channel B processing is skipped entirely.

If the channel A saved value is zero:

1. The event value is read and stored into the channel A saved-value field.
2. The channel A start counter is read, incremented by one, and written back.
3. Execution proceeds to the discriminator reread and channel B stage.

The saved-value write precedes the counter update. The counter increments even when the stored event value is zero.

This branch does not update the channel A accumulator.

### Channel A finish: initial discriminator `0x11`, `0x0E`, or `0x0F`

The function reads the channel A saved value.

If it is zero, channel A receives no write.

If it is nonzero, the function updates the channel A accumulator to:

**event value − saved value + previous accumulator**

The subtraction is the inner operation in the supplied expression. Subject to representable arithmetic, the effect is to add the difference between the event value and the saved value.

The accumulator write occurs before a four-byte zero is written to the channel A saved-value field. The counter is not changed.

Execution then proceeds to the discriminator reread and channel B stage.

No check requires the event value to be greater than or equal to the saved value. A negative difference can reduce the accumulator when the arithmetic is representable.

### Other initial discriminators

If the initial discriminator is not `0x10`, `0x11`, `0x0E`, or `0x0F`, channel A handling performs no state access for that channel. Execution proceeds to the second discriminator read.

### Channel B finish: reread discriminator `0x13`, `0x0E`, or `0x0F`

The function reads the channel B saved value.

If it is zero, the function returns without a channel B write.

If it is nonzero, the channel B accumulator becomes:

**event value − saved value + previous accumulator**

The accumulator is written first. The channel B saved value is then cleared with a four-byte zero store. The channel B counter is unchanged, and the function returns.

For stable, disjoint inputs, `0x0E` and `0x0F` therefore finish channel A first and channel B second. Each channel is checked independently; one can finish while the other is inactive.

### Channel B start: reread discriminator `0x12`

The function reads the channel B saved value.

If it is nonzero, the function performs no channel B write and returns.

If it is zero:

1. The event value is read and stored into the channel B saved-value field.
2. The channel B start counter is incremented by one.
3. The function returns.

The saved-value write precedes the counter update. A zero event value still increments the counter but leaves the inactive sentinel in place.

### Other reread discriminators

Any second discriminator other than `0x12`, `0x13`, `0x0E`, or `0x0F` causes the function to return without channel B state access.

### Stable-input event summary

For disjoint storage and a discriminator unchanged during the call:

| Event | Channel A action | Channel B action |
|---|---|---|
| `0x10` | Start if inactive; otherwise return immediately | None |
| `0x11` | Finish if active | None |
| `0x12` | None | Start if inactive |
| `0x13` | None | Finish if active |
| `0x0E`, `0x0F` | Finish if active | Finish if active |
| Any other value | None | None |

“Start if inactive” includes incrementing the counter even when the event value is zero. “Finish if active” clears the saved value after accumulation.

## Side effects

### Observable access ordering

The visible control flow establishes these ordering constraints:

- The initial discriminator read precedes all state processing.
- Channel A processing precedes the second discriminator read.
- The second discriminator read precedes channel B processing.
- Each successful start writes its saved value before updating its counter.
- Each successful finish writes its accumulator before clearing its saved value.
- The immediate return for an already-active channel A start prevents the second discriminator read and every channel B access.

The supplied expression does not establish a complete machine-level order among the individual operand reads used to calculate a finish result. It references the event value, saved value, and previous accumulator, but the exact operand-load order is documented_only.

The event value is not globally captured once for the call. If both channels finish, each finish expression obtains its event value separately. The channel A writes occur before the channel B finish expression is reached. Therefore, overlap or external mutation can make the two calculations use different values.

### Write footprint

Only the six listed state fields are explicit write destinations. The event record has no explicit write destination, although overlapping storage could make a state write modify part of that record.

There are no visible calls, logging operations, I/O operations, synchronization primitives, or explicit exception/error-reporting operations.

No rollback mechanism is shown. If an execution fault occurs after an earlier write, the source provides no recovery path that restores prior state.

### Error behavior

Invalid addresses and arithmetic outside the supported signed range are not translated into a status result. Their target-specific consequences are unknown.

A duplicate start is not reported as an error:

- For channel A, it returns immediately.
- For channel B, it leaves that channel unchanged and returns.

A finish while inactive is ignored for that channel. An unknown event discriminator is also ignored, subject to the second-read behavior described above.

## Dependencies

No external or internal function calls are visible in F-2b1c5e575dcc. There are consequently no dependency function IDs to list.

Its observable behavior depends directly on caller-supplied memory and the target’s integer, address, alignment, and memory-access rules. No assumption about another function preserving global state or managing memory is needed.

## Acceptance examples

These are conditional acceptance expectations derived from static evidence, not recorded runtime results. Unless otherwise stated, assume valid, disjoint, stable storage, four-byte signed integer fields, and representable arithmetic. Unmentioned fields remain unchanged.

### Starting channel A

Given channel A saved value `0`, start counter `7`, and event `0x10` with value `100`:

- The saved value becomes `100`.
- The counter becomes `8`.
- Neither accumulator changes.
- Channel B remains unchanged.

The saved-value write occurs before the counter update.

### Duplicate channel A start

Given channel A saved value `100` and event `0x10`:

- No state field changes.
- The function returns before rereading the discriminator.
- Neither the event value nor any channel B field is required by the visible path.

### Finishing channel A

Given channel A saved value `100`, accumulator `20`, and event `0x11` with value `145`:

- The accumulator becomes `65`.
- The saved value becomes `0`.
- The start counter is unchanged.
- Channel B is unchanged.

### Finishing both channels

Given:

- Channel A saved value `100`, accumulator `20`.
- Channel B saved value `120`, accumulator `5`.
- Event `0x0E` with value `150`.

The final state is:

- Channel A accumulator `70`, saved value `0`.
- Channel B accumulator `35`, saved value `0`.
- Both counters unchanged.

Channel A’s accumulator update and clearing occur before channel B’s update and clearing. Event `0x0F` has the same stable-input effect.

### Independent inactive handling

Given channel A saved value `0`, channel B saved value `120`, channel B accumulator `5`, and event `0x0F` with value `150`:

- Channel A is unchanged.
- Channel B accumulator becomes `35`.
- Channel B saved value becomes `0`.

The inactive channel A does not prevent channel B from finishing.

### Zero-valued starts

Given channel B saved value `0`, counter `3`, and event `0x12` with value `0`:

- The saved value remains numerically zero.
- The counter becomes `4`.

A subsequent `0x12` event with value `9` can start the channel again and increment the counter to `5`. The first zero-valued start does not establish a distinguishably active interval.

### Decreasing event values

Given channel A saved value `100`, accumulator `20`, and event `0x11` with value `90`:

- The accumulator becomes `10`.
- The saved value becomes `0`.

There is no monotonicity check or clamping.

### Unrecognized event

Given stable event discriminator `0x20`:

- The discriminator is read at entry and again before channel B selection.
- No state field or event-value field is needed by the visible path.
- No state field changes.

### Aliased discriminator

Suppose an initially zero channel A saved-value field occupies the same storage as the event discriminator. For initial discriminator `0x10`, those premises are incompatible: the same storage cannot simultaneously contain zero and `0x10`.

Aliasing examples must therefore satisfy the actual shared-memory values rather than independently assigning contradictory field values. The general acceptance requirement is to preserve the second discriminator read and all preceding writes, not to apply the stable-input event table unconditionally.

## Uncertainties

The following remain documented_only:

- The semantic names, units, and valid ranges of event values and accumulators.
- The target ABI, pointer width, byte order, and exact alignment requirements.
- Exact signed-overflow behavior and whether a lower-level implementation wraps.
- Machine instruction ordering among operands within an arithmetic expression.
- Atomicity, concurrent-access guarantees, and suitability for device or volatile memory.
- Whether callers guarantee non-overlapping inputs or immutable event records.
- Whether zero-valued starts are intentional, invalid caller input, or a limitation of the sentinel design.
- Any broader lifecycle in which counters or accumulators are initialized, reset, or consumed.

The two-channel interval interpretation is inferred intent. The branch conditions, field offsets, calculations, explicit writes, and early-return structure are established static evidence.

## Evidence

This contract is based solely on the supplied function body for F-2b1c5e575dcc.

The body establishes two discriminator reads, the conditional early return during channel A start handling, six state-field write destinations, zero-based activity checks, counter increments on accepted starts, and subtraction-plus-accumulation followed by clearing on active finishes.

No implementation dependencies, caller definitions, type declarations for the complete records, target instruction listing, or execution traces were supplied. No runtime tests have been performed.
