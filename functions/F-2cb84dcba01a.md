<!-- Trusted provenance: F-2cb84dcba01a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2cb84dcba01a

## Purpose

F-2cb84dcba01a processes a message associated with a stateful receiver. It first enters a receiver-associated synchronization or guard mechanism, then examines the message’s payload reference.

A message with a null payload follows a disposal path and is not submitted to the queue-related dependencies. A message with a non-null payload can update receiver state according to a signed 32-bit discriminator. It is then wrapped, submitted through a queue-related dependency, followed by a notification-like operation and a separate dispatch operation.

The state fields’ application meanings, synchronization semantics, queue ownership rules, and dispatch semantics are not established. “Counter,” “mode,” “queue,” and “notification” below are descriptive interpretations, not independently verified API guarantees.

## Interface

### Inputs

The function receives two machine-level arguments:

- **Receiver address:** identifies writable receiver storage.
- **Message address:** identifies a message object whose payload-reference field is read.

The supplied representation uses 32-bit integer values, four-byte pointer fields, and byte-addressed offsets. The receiver argument is represented as an integer but used as an address. No conclusion is established about portability to a different pointer width or ABI.

### Required layouts

The receiver must support the following accesses:

| Byte offset | Required use |
|---|---|
| `+0x10c` | Address passed to F-338d16f9ce72; later read as a pointer to a 32-bit word |
| `+0x124` | Address passed to F-37efbe955e3e |
| `+0x12c` | Address passed to F-338d16f9ce72 and F-1333331b2da1 |
| `+0x134` | Address passed to F-acda426cbea1 |
| `+0x140` | Signed 32-bit mode field |
| `+0x144` | Signed 32-bit counter field |
| `+0x148` | Four-byte copied-value field |

The message must support:

| Byte offset | Required use |
|---|---|
| `+0x00` | Dispatch-table pointer, used only on the null-payload path |
| `+0x08` | Payload pointer, read before any effective message-null protection |

A non-null payload must support a signed 32-bit discriminator at `+0x04`. One discriminator branch additionally reads a pointer at payload offset `+0x0c`, then reads four bytes through that pointer.

On the null-payload path, the message’s dispatch table must contain a callable entry at byte offset `+0x04`. That operation receives the message address explicitly.

Two other indirect operations use tables obtained through F-87be4aa684d1:

- Entry at `+0x16c`: supplies a four-byte counter baseline.
- Entry at `+0x18c`: supplies a signed 32-bit effective mode.

The supplied representation shows no explicit arguments for those two indirect calls. Their complete ABI requirements are unknown.

### Preconditions and result

The function has no explicit result value.

A valid, readable message address is a precondition even when the intended payload is null. The function reads the message’s payload field before checking whether the message itself is null. Passing a null message is therefore not a supported disposal shortcut.

All receiver fields, payload fields, indirect tables, and pointed-to values actually accessed must be valid for the corresponding operation. No validation or recoverable error reporting is visible.

## Behavior

### 1. Entry and payload selection

The first explicit action is a call to F-338d16f9ce72 with the receiver address advanced by `0x10c`.

Only after that call returns does the function read the payload pointer from message offset `+0x08`. Consequently, this read observes memory as it exists after the dependency returns; it must not be modeled as an earlier snapshot.

If the payload pointer is null:

1. The function conditionally invokes the message’s dispatch-table entry at `+0x04`, passing the message address.
2. It skips discriminator processing and all queue, notification, allocation, and dispatch operations described below.
3. It proceeds directly to the common exit protocol.

The message-null condition preceding the indirect call does not protect the earlier payload-field read.

If the payload pointer is non-null, the function reads its signed 32-bit discriminator at `+0x04` and performs the state processing below.

### 2. Discriminator routing

Let **T** denote the signed 32-bit value held in the external discriminator-threshold datum. Its actual value is not supplied.

Routing gives equality with T first priority:

- If the discriminator equals T, write `1` to receiver offset `+0x140`.
- Otherwise, if the discriminator is signed-less-than T, use the lower-side cases.
- Otherwise, use the upper-side cases.

This ordering is part of the contract. A named numeric case is not unconditionally reachable: its side of T must also match. In particular, if a named numeric discriminator equals T, the equality action takes precedence over that numeric case.

### 3. Lower-side state processing

These cases apply only when the discriminator is signed-less-than T.

#### Discriminator `0x6600000d` or `0x66000017`

Perform two four-byte writes in this order:

1. Write `0` to the counter at `+0x144`.
2. Write `0` to the mode at `+0x140`.

No baseline-provider operation is called by these branches.

#### Discriminator `0x66000014`

This is the increment branch.

If the counter at `+0x144` equals `-1`:

1. Call F-87be4aa684d1.
2. Obtain the returned object’s dispatch-table entry at `+0x16c`.
3. Invoke that entry.
4. Store its four-byte result in the counter field.

After any initialization, read the counter field, add one using the represented signed integer operation, and write the result back.

The field is read after the baseline call and its result store. The operation is not specified as incrementing a value captured before that call.

No upper-bound check is visible. At the maximum signed 32-bit value, the supplied representation does not establish a portable arithmetic result. Neither wraparound nor saturation is promised.

#### Discriminator `0x66000015`

This is the decrement branch.

If the counter equals `-1`, obtain and store a baseline through F-87be4aa684d1 and the `+0x16c` indirect entry, exactly as in the increment branch.

Then read the counter:

- If it is signed-greater-than zero, subtract one and store the result.
- Otherwise, perform no decrement write.

Thus zero remains zero through this local operation, and negative values are not decremented. Baseline initialization can still write a negative result, including `-1`; there is no retry loop.

#### Other lower-side discriminators

Perform no explicit state-field update during discriminator processing.

### 4. Upper-side state processing

These cases apply only when the discriminator is signed-greater-than T.

#### Discriminator `0x6600001a`

Write `0` to the mode at `+0x140`.

This branch does not explicitly reset the counter.

#### Discriminator `0x6600001b`

Read the mode at `+0x140`.

If that value is `-1`:

1. Call F-87be4aa684d1.
2. Invoke the returned object’s dispatch-table entry at `+0x18c`.
3. Use the returned signed 32-bit value as the effective mode.

Otherwise, use the already-read mode as the effective mode.

Then:

- Effective mode `1`: write `2` to the mode field.
- Effective mode `2`: write `1` to the mode field.
- Any other effective mode: perform no explicit mode-field write.

The provider result is not first stored as a baseline mode. If the stored mode is `-1` and the provider returns another value outside `{1, 2}`, this branch does not itself overwrite the stored mode. Any changes made by the provider remain a separate possibility.

#### Discriminator `0x66000024`

Read the pointer at payload offset `+0x0c`, read four bytes through that pointer, and copy those bytes to receiver offset `+0x148`.

This is a single four-byte copy, not a deep copy. The function performs no visible null check on that additional pointer.

#### Other upper-side discriminators

Perform no explicit state-field update during discriminator processing.

### 5. Common non-null-payload submission sequence

Every non-null-payload message reaches this sequence after discriminator processing, including messages whose discriminator has no explicit state action.

The explicit operations occur in this order:

1. Call F-338d16f9ce72 with receiver offset `+0x12c`.
2. Call F-8157a180d901 with requested size `8`.
3. Write the original message address into the returned region at offset `+0x04`.
4. Call F-37efbe955e3e with receiver offset `+0x124` and the returned region.
5. Call F-acda426cbea1 with receiver offset `+0x134`.
6. Call F-1333331b2da1 with receiver offset `+0x12c`.
7. Call F-0a77364c41cc with requested size `12`.
8. Pass its result to F-afb630a7107c, followed by an external datum and two zero arguments.
9. Pass F-afb630a7107c’s result, the receiver address, and integer `1` to F-9a5c97dc580d.
10. Proceed to the common exit protocol.

Only the four bytes at offset `+0x04` of the eight-byte region are explicitly initialized here. The function does not visibly initialize its first four bytes before passing it onward.

There is no visible allocation-failure branch. In particular, the eight-byte allocation result is used for a write without a null check.

The external datum supplied to F-afb630a7107c is distinct from T. Its value and interpretation are unknown.

### 6. Common exit protocol

On both the null-payload and non-null-payload paths:

1. Read a pointer from receiver offset `+0x10c`.
2. If that pointer is null, return.
3. Otherwise, read the first signed 32-bit word through it.
4. If that word is zero, return.
5. Otherwise, invoke an external indirect target with no explicit arguments shown, then return if that invocation returns.

These reads occur after all preceding calls on the selected path. Their values must not be assumed unchanged since entry.

The supplied evidence does not establish the indirect target’s identity or semantics.

## Side effects

### Explicit state and memory writes

Depending on routing, the function can write:

- Mode at `+0x140`.
- Counter at `+0x144`.
- Four-byte copied value at `+0x148`.
- The message pointer at `+0x04` in a newly requested eight-byte region.

Counter reset precedes mode reset in the two combined-reset cases. All discriminator-dependent writes precede the non-null-payload submission sequence.

These are the function’s explicit writes, not an exhaustive list of effects of its dependencies.

### Ownership and lifetime

For a null payload, the message is passed to an indirect operation resembling disposal. Whether that operation destroys, releases, recycles, or merely updates the message is unknown. The function makes no subsequent explicit access through the message address on that path.

For a non-null payload, the function passes onward a newly requested region containing the original message address. It does not explicitly copy the message or payload, and does not visibly dispose of them afterward.

The ownership contract for the message, the eight-byte region, and the twelve-byte allocation is **documented_only**. No transfer, retention, release, or eventual destruction may be assumed from this function alone.

### Synchronization and failures

The paired operations around receiver offset `+0x12c`, and the entry/exit handling involving `+0x10c`, suggest synchronization or guard management. Their actual exclusion, nesting, release, and memory-order guarantees are unknown.

No error result, rollback, or recovery path is visible. Dependency failure, non-returning calls, invalid pointers, and arithmetic overflow are not converted into an explicit error response. Earlier writes and calls precede any later failure; no transactional guarantee is established.

## Dependencies

| Opaque ID or indirect operation | Established interaction |
|---|---|
| F-338d16f9ce72 | Called first with receiver `+0x10c`; called again with receiver `+0x12c` on the non-null-payload path |
| F-87be4aa684d1 | Supplies an object for baseline-related indirect calls when a relevant field equals `-1` |
| Indirect entry `+0x16c` | Returns the counter baseline stored before increment or decrement processing |
| Indirect entry `+0x18c` | Returns an effective mode used without an unconditional baseline store |
| F-8157a180d901 | Receives allocation-size argument `8` |
| F-37efbe955e3e | Receives receiver `+0x124` and the eight-byte region |
| F-acda426cbea1 | Receives receiver `+0x134` after submission |
| F-1333331b2da1 | Receives receiver `+0x12c` after the preceding operation |
| F-0a77364c41cc | Receives allocation-size argument `12` |
| F-afb630a7107c | Receives that allocation result, an external datum, zero, and zero |
| F-9a5c97dc580d | Receives the preceding result, receiver address, and `1` |
| Message indirect entry `+0x04` | Receives the message on the null-payload path |
| External exit target | Invoked only when the final pointer and its first word are both nonzero |

No listed dependency is assumed to preserve globals, receiver fields, or memory ownership.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, dependencies return normally, all accessed memory is valid, and dependencies do not independently alter the fields discussed.

1. **Null payload:** A valid message whose payload field is null causes the entry operation, the message indirect operation, and the common exit protocol. No eight-byte allocation or submission operation occurs.

2. **Threshold equality wins:** If T equals `0x66000014`, that discriminator writes mode `1`. It does not select the increment branch, regardless of the counter’s initial value.

3. **Combined reset:** If `0x6600000d` is below T, processing it writes counter `0` before mode `0`, then follows the non-null-payload submission sequence.

4. **Increment with baseline:** If `0x66000014` is below T, the counter is `-1`, and the baseline operation returns `7`, the function stores `7`, then stores `8`.

5. **Decrement boundary:** If `0x66000015` is below T, an initial counter of `0` produces no decrement write. An initial counter of `1` becomes `0`.

6. **Negative baseline:** On that decrement branch, a baseline result of `-3` is stored and is not decremented.

7. **Mode toggle:** If `0x6600001b` is above T, mode `1` becomes `2`, and mode `2` becomes `1`. Stored mode `3` produces no explicit mode write.

8. **Mode baseline is not cached automatically:** On the same branch, stored mode `-1` with provider result `0` produces no explicit mode write. It does not automatically store `0`.

9. **Indirect value copy:** If `0x66000024` is above T and its extra pointer identifies the four-byte value `0x12345678`, those four bytes are copied to receiver `+0x148` before submission.

10. **Unrecognized discriminator:** A non-null payload with no matching reachable case still follows the full submission sequence.

11. **Conditional exit call:** A null final pointer or a zero first word suppresses the exit target invocation. A non-null pointer with a nonzero first word causes that invocation.

## Uncertainties

The following remain **documented_only**:

- The actual value of T and therefore the unconditional reachability of numeric cases.
- The meaning of the external datum supplied during twelve-byte-object preparation.
- Application meanings and allowed domains of the receiver fields.
- Full calling conventions for indirect operations.
- Allocation behavior, ownership, initialization, and failure handling inside dependencies.
- Whether submission retains or consumes either the wrapper or message.
- Synchronization semantics and the identity of the final indirect target.
- Dependency-induced mutations of receiver state or globals.
- Runtime behavior of signed increment at the maximum 32-bit value.

The inferred intent is a state-update-and-submit handler with lazy baseline acquisition. That interpretation does not strengthen the established memory, ownership, or synchronization guarantees.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence covers explicit field widths and offsets, branch priority, call arguments, local read/write ordering, and the absence of visible validation or recovery branches. Semantic labels such as allocation, disposal, queueing, and synchronization are interpretations where dependency bodies are unavailable.

No runtime tests, executions, or independent implementation checks have been performed.
