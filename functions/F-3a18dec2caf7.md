<!-- Trusted provenance: F-3a18dec2caf7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3a18dec2caf7

## Purpose

F-3a18dec2caf7 conditionally transfers a portion of an initially measured outstanding quantity, advances a position through a dependency, records a state value in an associated object, and returns a 32-bit result.

The established behavior is a control-and-accounting operation. It includes polling, two alternative transfer callbacks, and completion-state updates. The source does not establish what the transferred data represents, whether the transfer is synchronous, or what the numeric result and state values mean outside this function.

Inferred intent: process one bounded portion of pending data after a readiness or cancellation check. This interpretation must not replace the exact branch and accounting behavior below.

## Interface

### Input and object layout

The function receives one pointer to a receiver object. The supplied representation uses 32-bit integer and pointer-sized fields.

The following byte offsets are relevant:

| Object | Offset | Observed use |
|---|---:|---|
| Receiver | `0x00` | Pointer to a dispatch table. |
| Receiver | `0x10` | Pointer to an associated object supplied to the direct dependencies. |
| Receiver | `0x50` | Pointer to a location from which one status byte is read during polling. |
| Associated object | `0x00` | Pointer to a dispatch table used to obtain another object. |
| Associated object | `0x58` | Destination of a 32-bit state write, with value `3` or `4`. |
| Obtained object | `0x00` | Pointer to a dispatch table used to obtain a 32-bit address contribution. |

Relevant dispatch-table byte offsets are:

- Receiver: `0x44`, `0x8c`, and `0x90`.
- Associated object: `0x04`.
- Obtained object: `0x08`.

These are accessed layout locations, not declarations of complete object types.

### Output

The function returns a 32-bit value. Its explicit return values are:

| Return value | Condition |
|---:|---|
| `8` | A polling iteration observes the first status-byte check equal to `1` after masking with `0x89`. |
| `3` | The additional receiver callback at dispatch offset `0x44` returns nonzero. |
| `0` | The initially measured quantity is zero, or the transfer/accounting path completes normally. |

The state written at associated-object offset `0x58` is separate from the return value. In particular, return value `0` can accompany either state `3` or state `4`.

### Preconditions

There is no explicit null, bounds, alignment, dispatch-target, or lifetime validation.

Every object, field, dispatch table, callback target, and status-byte location actually reached must be valid for its corresponding access. Called dependencies must satisfy the platform calling convention and return normally for subsequent behavior to occur.

The receiver must remain usable across callbacks. The function does not establish that its associated-object pointer, status pointer, or dispatch table remains unchanged across calls.

## Behavior

### 1. Measure the initial outstanding quantity

The function first calls:

1. F-cb7692f42f99 with the receiver’s associated-object pointer.
2. F-0597971e4c2b with the receiver’s associated-object pointer.

It subtracts the second 32-bit result from the first and retains the result as an unsigned 32-bit quantity.

For the rest of this contract, this retained value is the **initial quantity**. It is not recomputed after polling or transfer.

The represented arithmetic is 32-bit. There is no explicit overflow or underflow check. At the represented machine-value level, subtraction retains its low 32 bits; a negative mathematical difference can therefore become a large unsigned quantity.

If the initial quantity is zero, the function skips polling, object acquisition, transfer, and position advancement. It writes state `4` at offset `0x58` of the associated object then referenced by the receiver and returns `0`.

### 2. Poll the receiver

For a nonzero initial quantity, the function repeatedly invokes the receiver callback at dispatch offset `0x44`, passing the receiver.

If that invocation returns nonzero, polling ends without reading the status byte in that iteration.

If it returns zero, the function performs these checks in order:

1. Read the status byte through the receiver field at offset `0x50`.
2. Mask the byte with `0x89`.
3. If the masked value equals `1`, write state `4` to the associated object and return `8`.
4. Otherwise, read the status byte again through the receiver’s status pointer.
5. Mask this separately read byte with `0x89`.
6. If the masked value equals `8`, leave the polling loop.
7. Otherwise, invoke F-000000000001 with argument `100`, then begin another polling iteration.

The two status checks are distinct reads. They must not be treated as one cached observation when reasoning about changing memory.

The comparisons are exact masked-value comparisons, not independent tests for whether bit `0x01` or bit `0x08` is present. For example, a stable byte value of `0x09` matches neither condition.

F-000000000001 appears delay-like, but neither its time unit nor its actual scheduling behavior is established.

Polling has no explicit iteration bound or timeout.

### 3. Perform the additional receiver check

After polling ends by either route, the function invokes the receiver callback at dispatch offset `0x44` again.

If this additional invocation returns nonzero, the function writes state `4` and returns `3`.

This is a fresh call, not reuse of the result that ended polling. Consequently:

- A nonzero polling result does not by itself guarantee return value `3`.
- A status-byte exit does not by itself guarantee transfer.
- The additional call’s result determines whether the transfer path is entered.

### 4. Obtain the transfer inputs

If the additional receiver check returns zero, the function performs the following operations in order:

1. Invoke the callback at associated-object dispatch offset `0x04`, with no explicit arguments in the supplied representation.
2. Retain its result as a pointer to an obtained object.
3. Call F-c581c64350a3 with the associated object and retain its 32-bit result as an address contribution.
4. Call F-0597971e4c2b with the associated object and retain its 32-bit result as the position used for the transfer address.
5. Call F-d3d76dd40ef8 with the associated object and compare its unsigned result with the initial quantity.

The **selected quantity** initially equals the initial quantity.

If the first result from F-d3d76dd40ef8 is smaller than the initial quantity, the function calls F-d3d76dd40ef8 a second time and uses that second result as the selected quantity.

This is not necessarily the mathematical minimum of two stable values. The first result controls whether the second call occurs; the second result supplies the quantity without another comparison or clamp. If the dependency changes its answer, the selected quantity can exceed the initial quantity.

Next, the function:

6. Invokes the obtained object’s callback at dispatch offset `0x08`, passing that object, and retains its 32-bit result.
7. Calls F-3a80612945c8 with the associated object to obtain a 32-bit flag value.

### 5. Invoke the selected transfer callback

The flag value selects exactly one receiver callback:

- Bit `0x2000` clear: dispatch offset `0x90`.
- Bit `0x2000` set: dispatch offset `0x8c`.

The selected callback receives:

1. The receiver.
2. A 32-bit address-like value formed by adding:
   - the obtained-object callback’s result,
   - the result from F-c581c64350a3,
   - the position obtained before quantity selection.
3. The selected quantity.

The address-like addition has no explicit overflow check, pointer validation, or range validation.

The callback’s return value is not inspected. The function does not receive or use an actual-transferred count through that return path. Any other effects of the callback are outside the established contract.

A selected quantity of zero does not suppress the callback.

### 6. Advance the position and record state

After the transfer callback returns, the function calls F-0597971e4c2b again with the associated object.

It adds the selected quantity to this fresh position result and passes the 32-bit sum, together with the associated object, to F-bec07220152b.

This advancement is not necessarily based on the position used to construct the transfer address. A dependency or callback may have changed the position between the two reads.

The function then computes the unsigned 32-bit difference:

**initial quantity minus selected quantity**

If that difference is greater than `1`, it writes state `3` and returns `0`.

Otherwise, it writes state `4` and returns `0`.

Thus, both zero and one unit of represented remainder produce state `4`. The state decision does not remeasure the total or read back the position after F-bec07220152b.

## Side effects

### Directly established writes

The function directly writes one 32-bit state value to associated-object offset `0x58` on each explicit return path reached normally:

- State `4` for an initially zero quantity.
- State `4` before return `8`.
- State `4` before return `3`.
- State `3` after transfer/accounting when the represented remainder exceeds one.
- State `4` after transfer/accounting otherwise.

There is no explicit state write before polling or before invoking the transfer callback.

### Dependency-mediated effects

The function invokes external and indirect dependencies that may change receiver fields, associated-object state, global state, memory, or external resources. Their preservation behavior is not established.

In particular, repeated accesses to the receiver’s associated-object field cannot safely be interpreted as accesses to an immutable object identity across calls. Previously returned values remain the values used in later calculations, while later receiver-field accesses may refer to a changed object.

### Ownership and lifetime

No explicit allocation, deallocation, reference acquisition, or reference release is visible.

The obtained object is used for one indirect callback. The source does not establish whether obtaining it transfers ownership, borrows it, increments a reference count, or requires cleanup. No release should be invented from this function alone.

The same limitation applies to the address-like value passed to the transfer callback: ownership, retention, copying, and validity duration are unknown.

### Failures and interruption

The function has no explicit handling for dependency failures beyond the nonzero receiver-callback results described above.

Invalid pointers, invalid dispatch targets, dependency nonreturn, and platform faults have no specified recovery path here. The normal state write and return are not guaranteed if an earlier call or access does not complete.

## Dependencies

The following direct dependency identities are established by the supplied mapping:

| Opaque ID | Observed role |
|---|---|
| F-cb7692f42f99 | Supplies the first value used to compute the initial quantity. |
| F-0597971e4c2b | Supplies the initial subtraction operand, transfer-address position, and fresh post-transfer position. |
| F-c581c64350a3 | Supplies an address contribution. |
| F-d3d76dd40ef8 | Supplies the quantity-comparison value and, conditionally, a separately obtained selected quantity. |
| F-3a80612945c8 | Supplies flags selecting the transfer callback. |
| F-bec07220152b | Receives the fresh position plus selected quantity. |

Additional dependency labels below are document-local opaque IDs; no cross-document identity is established:

| Opaque ID | Dispatch or invocation |
|---|---|
| F-000000000001 | External call with argument `100` on each continuing polling iteration. |
| F-000000000002 | Receiver dispatch offset `0x44`; called during polling and once more after polling exits. |
| F-000000000003 | Associated-object dispatch offset `0x04`; invoked without explicit arguments. |
| F-000000000004 | Obtained-object dispatch offset `0x08`; invoked with that object. |
| F-000000000005 | Receiver dispatch offset `0x90`; selected when flag bit `0x2000` is clear. |
| F-000000000006 | Receiver dispatch offset `0x8c`; selected when flag bit `0x2000` is set. |

These observed roles do not establish the dependencies’ complete contracts.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Unless stated otherwise, assume valid memory, normally returning calls, and stable object pointers.

1. **No outstanding quantity.**  
   The first two dependency results are equal. Expect no polling or transfer callback, state `4`, and return `0`.

2. **First status condition.**  
   The initial quantity is nonzero, the polling callback returns zero, and the first status read is `0x01`. Expect state `4` and return `8`, without the second status read, delay call, additional receiver check, or transfer.

3. **Masked bits matter.**  
   With both status reads equal to `0x09` and the polling callback returning zero, neither status comparison succeeds. Expect the external call with argument `100`, followed by another polling iteration.

4. **Status exit followed by rejection.**  
   A polling callback returns zero; the first status check does not match `1`, and the second matches `8`. The additional receiver check then returns nonzero. Expect state `4`, return `3`, and no transfer.

5. **Polling result changes before the additional check.**  
   A polling callback returns nonzero, but the additional receiver check returns zero. Expect the function to proceed to object acquisition and transfer rather than return `3`.

6. **Two-unit remainder.**  
   The initial quantity is `10`, and both quantity-dependency calls return `8`. After the selected transfer callback and position update complete, expect state `3` and return `0`.

7. **One-unit remainder.**  
   The initial quantity is `10`, and both quantity-dependency calls return `9`. Expect state `4` and return `0` after transfer/accounting.

8. **Changed quantity result is not clamped.**  
   The initial quantity is `10`; the first quantity result is `5`, and the second is `12`. Expect a transfer request of `12`. The unsigned remainder is `0xfffffffe`, so expect state `3` after accounting.

9. **Position changes during transfer.**  
   The address uses position `20`, selected quantity is `4`, and the fresh position after the transfer callback is `30`. Expect F-bec07220152b to receive `34`, not `24`.

10. **Zero selected quantity.**  
    The initial quantity is `10`, and both quantity results are zero. Expect a zero-quantity transfer callback, a fresh position read and update request, state `3`, and return `0`.

## Uncertainties

The following are **documented_only**:

- Semantic meanings of return values `3` and `8`, and state values `3` and `4`.
- Units and signed interpretation of dependency results beyond the explicit unsigned comparisons and retained 32-bit arithmetic.
- Data format, transfer direction, and synchronization behavior.
- Meaning and time unit of the external argument `100`.
- Whether any dependency changes pointers, dispatch tables, positions, flags, globals, or object lifetimes.
- Ownership and cleanup obligations for the obtained object and transfer address.
- Thread-safety, atomicity, and memory-ordering guarantees.
- Behavior on invalid inputs or nonreturning dependencies.

“In-progress,” “complete,” “ready,” and “cancelled” are possible interpretations, not established names for these values or branches.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes access offsets, call order, explicit arguments, conditional repeated reads and calls, 32-bit quantities, masks, unsigned comparisons, direct state writes, and return values.

Higher-level purpose and naming are identified as inferred intent. Dependency internals, ownership rules, concurrency guarantees, and runtime behavior remain unknown where not shown.

No runtime tests have been performed.
