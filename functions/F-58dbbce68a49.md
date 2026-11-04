<!-- Trusted provenance: F-58dbbce68a49; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-58dbbce68a49

## Purpose

F-58dbbce68a49 updates a caller-supplied state record using a nesting counter, a requested 32-bit value, and an associated 32-bit payload. A zero mode selects a decrement operation; any nonzero mode selects an increment operation.

At particular counter boundaries, the function publishes a value and payload and sets a one-byte marker:

- An increment starting from zero publishes the requested value and supplied payload.
- A decrement ending at zero publishes a previously stored value and supplied payload.
- Other counter transitions do not publish values or set the marker.

The apparent intent is a nested override-and-restore mechanism. That interpretation is inferred rather than established: the supplied evidence does not identify the record’s owner, the meanings of the stored values, or the consumer of the marker.

## Interface

### Inputs

The function takes four arguments:

| Argument | Contract |
|---|---|
| State address | Address of a writable record containing the fields described below. The recovered interface represents this address as an integer. |
| Mode | Integer selector. Zero selects decrement; every nonzero value selects increment. Its magnitude otherwise has no effect. |
| Payload | Opaque 32-bit value, copied unchanged into the payload field on a publication boundary. |
| Requested value | 32-bit integer used by the increment-from-zero path for comparison and publication. It is not used by the decrement path. |

There is no explicit return value or status result.

### Referenced record layout

Offsets are byte offsets from the supplied state address. Descriptive field names below identify observable storage roles, not established application meanings.

| Offset | Width | Role |
|---|---:|---|
| `0x28` | 4 bytes | Current comparison value; conditionally copied into saved storage |
| `0x6c` | 4 bytes | Published payload |
| `0x70` | 4 bytes | Published value |
| `0x74` | 4 bytes | Nesting counter |
| `0x78` | 4 bytes | Saved value |
| `0x7c` | 1 byte | Gate controlling whether the current comparison value may be saved |
| `0x7d` | 1 byte | Publication marker, assigned the value one |

The recovered representation uses 32-bit integer fields and four-byte opaque values. Byte order and machine-level alignment requirements are not established by the supplied material. The byte gate is tested only for zero, so its signed interpretation does not affect the branch.

### Preconditions and access requirements

The state address must support every read and write reached by the selected path. No null-pointer, bounds, alignment, permissions, counter-range, or record-validity checks are visible.

The highest referenced byte is at offset `0x7d`. A conventional contiguous record backing all fields therefore needs at least `0x7e` bytes from its base, although an individual invocation may access only a subset of those fields.

The counter must permit the selected arithmetic operation for behavior to remain within the clearly established range of the recovered signed-integer representation. In particular:

- Decrementing the minimum signed 32-bit value is outside that range.
- Incrementing the maximum signed 32-bit value is outside that range.

No rejection or saturation behavior is shown for those cases. Exact overflow behavior is documented_only.

## Behavior

### Common entry behavior

The first represented state access is a four-byte read of the counter at offset `0x74`. The resulting entry value determines subsequent counter-boundary decisions.

The function does not first inspect the publication marker, gate, payload, published value, or saved value.

### Zero mode: decrement operation

The function computes the entry counter minus one and writes that result to offset `0x74`.

If the resulting counter is not zero, the operation ends. No other record field is read or written on that path.

If the resulting counter is zero, publication occurs in this order:

1. Read the four-byte saved value at offset `0x78` and write it to the published-value field at offset `0x70`.
2. Write the supplied 32-bit payload to offset `0x6c`.
3. Write the byte value one to the publication marker at offset `0x7d`.

Consequences within the non-overflowing range:

- An entry counter of one triggers restoration publication.
- An entry counter of zero becomes negative one and does not publish.
- A negative entry counter becomes more negative and does not publish.
- The gate and current comparison value are not consulted.
- The requested-value argument has no effect.
- The saved value is consumed as stored; this function does not establish whether it was initialized by an earlier invocation.

The counter write precedes all publication accesses. Thus, in the represented access sequence, the counter has already reached zero before the saved value is read or either publication field is written.

### Nonzero mode: increment operation

When the entry counter is nonzero, the function skips all comparison, saving, and publication work. It writes the entry counter plus one to offset `0x74` and ends.

When the entry counter is zero, the function first decides whether to update the saved value:

- It reads the four-byte current comparison value at offset `0x28` and compares it with the requested value.
- If they are equal, it does not read the gate and does not update the saved value.
- If they differ, it reads the one-byte gate at offset `0x7c`.
- If that gate is nonzero, it leaves the saved value unchanged.
- If that gate is zero, it reads the current comparison value again and writes that read’s value to offset `0x78`.

After this conditional saving step, the function publishes unconditionally for an entry counter of zero:

1. Write the supplied payload to offset `0x6c`.
2. Write the requested value to offset `0x70`.
3. Write the byte value one to offset `0x7d`.
4. Finally, write the entry counter plus one to offset `0x74`.

The saved-value decision does not determine whether publication occurs. Equal comparison and requested values still cause publication. A nonzero gate also still permits publication; it inhibits only replacement of the saved value.

The comparison and possible saving involve separate reads of offset `0x28`. Under stable-memory conditions they observe the same value. If the storage changes between those reads, the value saved need not be the value used in the comparison. No synchronization is visible.

### Counter boundaries are entry-sensitive

The increment path’s publication condition is that the **entry** counter equals zero, not that the resulting counter is positive or equals zero.

For example, incrementing negative one produces zero without publication. Conversely, the decrement path publishes when its **resulting** counter equals zero.

The function does not enforce a nonnegative counter invariant or verify that increment and decrement operations are paired.

## Side effects

### Writes and their order

The maximum represented write sequences are:

| Path | Ordered writes |
|---|---|
| Decrement, result nonzero | Counter |
| Decrement, result zero | Counter; published value; payload; marker |
| Increment, entry nonzero | Counter |
| Increment, entry zero, no save | Payload; published value; marker; counter |
| Increment, entry zero, save allowed | Saved value; payload; published value; marker; counter |

The published value is written before the payload on restoration, but after the payload on increment-from-zero. These are distinct sequences, not a single uniform publication protocol.

The function never visibly:

- Clears the marker.
- Writes the current comparison value.
- Writes the gate.
- Clears the saved value after restoration.
- Allocates or releases memory.
- Calls another function.
- Performs I/O or produces an explicit error report.

The listed fields are the only explicit memory locations modified in the supplied body.

### Ownership and lifecycle

The caller supplies the record. No ownership transfer, allocation, deallocation, or lifetime extension is established.

The payload and requested value are stored as 32-bit values. Even if an application interprets either as an address or resource handle, the supplied behavior does not dereference it, retain an associated object through a dependency, or release any resource it denotes.

The saved value remains available after restoration. An increment-from-zero operation may deliberately leave it unchanged, including when the current comparison value already equals the requested value.

### Observation and concurrency limits

The ordering above describes the recovered body’s represented accesses. It does not establish atomic multi-field publication, memory barriers, interrupt exclusion, or a thread-safe protocol.

In particular, setting the marker does not itself prove that another observer receives a coherent snapshot. The source provides no separate notification operation or synchronization primitive.

Invalid memory accesses have no explicit recovery path. The precise fault or process-level consequence is outside the established contract.

## Dependencies

No calls to other opaque function IDs are present in the supplied body. F-58dbbce68a49 has no identified callee dependency.

Its external dependencies are the caller-provided memory and whatever code initializes or consumes that memory. Those participants are not identified, so their initialization, synchronization, ownership, and interpretation rules are documented_only.

No assumption about an unknown callee preserving globals or owning memory is needed: no such callee is shown.

## Acceptance examples

These examples are conditional consequences of the static evidence, not runtime test results. Unless stated otherwise, memory is valid and stable, arithmetic does not overflow, and fields not listed as written retain their previous contents.

### Increment from zero with saving enabled

Given an entry counter of zero, current comparison value 17, requested value 29, gate zero, and payload 41:

- The saved value becomes 17.
- The payload becomes 41.
- The published value becomes 29.
- The marker becomes one.
- The counter becomes one.

The saved-value write precedes publication, and the counter write follows it.

### Increment from zero with equal values

Given an entry counter of zero and both the current comparison value and requested value equal to 29:

- The saved value remains unchanged.
- The gate is not read.
- The supplied payload and requested value are published.
- The marker becomes one.
- The counter becomes one.

A later restoration may therefore publish an older saved value; equality does not initialize or refresh that storage.

### Increment from zero with saving inhibited

Given an entry counter of zero, unequal comparison and requested values, and any nonzero gate byte:

- The saved value remains unchanged.
- Publication still occurs.
- The counter becomes one after publication.

Gate values one and 255 have the same branch effect.

### Nested increment

Given an entry counter of two and any nonzero mode:

- The counter becomes three.
- The payload, published value, saved value, and marker are unchanged.
- The comparison field and gate are not read.

The requested value and supplied payload have no observable effect on this path.

### Final decrement

Given an entry counter of one, saved value 17, and supplied payload 53:

- The counter becomes zero first.
- The published value becomes 17.
- The payload becomes 53.
- The marker becomes one.

The payload comes from this decrement invocation, not from the invocation that previously saved the value.

### Non-final decrement and unmatched decrement

Given an entry counter of three and zero mode, only the counter changes, becoming two.

Given an entry counter of zero and zero mode, only the counter changes, becoming negative one. The function neither reports an error nor publishes a restoration.

### Increment from a negative counter

Given an entry counter of negative one and nonzero mode:

- The counter becomes zero.
- No publication occurs.
- The saved value and marker remain unchanged.

This distinguishes incrementing to zero from decrementing to zero.

### Already-set marker

If the marker is already one and the selected path does not publish, it remains one. If the selected path publishes, the function writes one again. There is no marker acknowledgement or reset behavior in this function.

## Uncertainties

The following remain documented_only:

- Application meanings of the current comparison value, payload, published value, saved value, gate, and marker.
- Whether callers are required to maintain nonnegative counters or balanced operations.
- Initialization requirements for the saved value before a restoring decrement.
- Exact overflow behavior at signed 32-bit limits.
- Exact address representation, byte order, alignment requirements, and hardware access properties.
- Whether the record can reside in volatile, device-backed, shared, or concurrently modified memory.
- Atomicity and ordering guarantees beyond the represented access sequence.
- How any consumer detects, acknowledges, or acts on publication.
- Whether stored 32-bit values represent numbers, addresses, handles, or other application data.

The nested override-and-restore interpretation is inferred intent. The field accesses, conditions, assignments, and absence of explicit calls are the established static evidence.

## Evidence

This contract is based solely on the supplied function body for F-58dbbce68a49 and its recovered type annotations.

The evidence establishes:

- An initial counter read.
- A zero-versus-nonzero mode split.
- A decrement-result-zero restoration boundary.
- An increment-entry-zero publication boundary.
- Conditional saving controlled by value inequality followed by a zero gate.
- Four-byte state values and one-byte gate and marker accesses.
- Different publication write orders on the two boundary paths.
- No explicit validation, allocation, deallocation, dependency calls, or status return.

No runtime tests have been performed. The acceptance examples are derived from static control flow and memory accesses, not observed executions.
