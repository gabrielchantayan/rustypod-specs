<!-- Trusted provenance: F-0cc3a52d7ec5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0cc3a52d7ec5

## Purpose

F-0cc3a52d7ec5 updates a progression field, selects either target reconciliation or counter draining, and conditionally enters a shared finalization continuation.

Reconciliation computes an unsigned 32-bit target and invokes adjustment dependencies while comparing that target with shared counter storage. Draining invokes an adjustment dependency until separately selected counter storage reads zero. Neither route directly changes its counter.

Finalization compares shared state, may update two fields, and may submit a temporary three-word record. Its return is either an explicit status or, on the notification branch, the record’s second word **after** the notification dependency returns. An immediate reconciliation exit instead returns a fresh counter read.

These are behavioral descriptions, not established domain meanings. All evidence is static; no runtime tests have been performed.

## Interface

### Callable interface

- No explicit arguments.
- Unsigned 32-bit return value.
- Inputs and outputs otherwise consist of shared-memory accesses and dependency effects.

The supplied representation establishes unsigned 32-bit counter and target operations, signed 32-bit fields, signed 16-bit countdown accesses, and an opaque four-byte dependency result. Some external pointee declarations are unavailable.

### Shared storage

The following role names describe storage without identifying implementation symbols.

| Role | Established layout or access |
|---|---|
| Progression source | Compared with 3; read again for the increment branch. Its declared width and signedness are unavailable. |
| Progression destination | Signed 32-bit field at byte offset 16 in a separate object. |
| Configuration block | Unsigned 32-bit words: flags at offset 0, eligibility word at offset 12, limit at offset 16, requested quantity at offset 20. |
| Additional eligibility value | Read only after the initial eligibility expression is zero, then converted to unsigned 32 bits. Its original width and signedness are unavailable. |
| Reconciliation pointer object | Shared pointer object identifying unsigned 32-bit counter storage. Captured and fresh uses are distinguished below. |
| Drain pointer object | Shared pointer object identifying signed 32-bit counter storage. Only zero/nonzero tests are performed. |
| Finalization object | Signed 32-bit comparison field at offset 704, signed 16-bit countdown at offset 708, signed 32-bit discriminator at offset 712. |
| Finalization reference | Separately addressed signed 32-bit comparison value. |
| Notification record | Three consecutive four-byte words at offsets 0, 4, and 8. Their pre-call values are 15, 0, and 22. |

The reconciliation and drain pointer objects are separately addressed. Whether their pointees coincide, or whether other shared objects overlap, is unknown.

### Preconditions and completion

Normal completion requires valid storage for every reached access and normally returning dependencies. Adjustment routes additionally require shared-state changes that eventually make their loop conditions false.

There is no explicit pointer validation, loop bound, timeout, or recovery path. No contract for invalid storage, exceptional dependency behavior, or concurrent access is supplied.

## Behavior

### 1. Progression update

The progression destination is written before eligibility evaluation and before every ordinary return path.

First, the progression source is compared with 3:

- If equal, the destination receives 3.
- Otherwise, the source is read again, incremented by one, and assigned to the 32-bit destination.

This is not a general saturation rule. With a stable, representable source value of 4, the destination receives 5. Values above 3 are not explicitly clamped.

The comparison read and increment read are separate in the supplied representation. The increment can therefore use a value different from the comparison value if intervening changes are meaningful under the actual execution model. The source’s missing declaration prevents a complete contract for increment overflow or conversion to the destination.

### 2. Eligibility and route selection

A configuration-block pointer is retained before the initial eligibility expression. The initial expression separately addresses configuration storage through its shared pointer object; it is not evidence that every initial access uses the retained pointer.

The initial expression combines:

- Flags masked by hexadecimal 46, selecting bits 1, 2, and 6.
- The eligibility word at byte offset 12.

The combination is a bitwise inclusive OR. If nonzero, select draining without invoking the three eligibility dependencies below.

If zero, read the additional eligibility value and convert it to unsigned 32 bits. A nonzero converted value also selects draining.

Only if both checks yield zero:

1. Invoke **F-unknown-01**.
2. If its result is nonzero, select draining.
3. Otherwise invoke **F-unknown-02**, whose result is unused.
4. Invoke **F-unknown-03**.
5. Immediately after F-unknown-03 returns, capture the reconciliation pointer from its shared pointer object.
6. Test F-unknown-03’s result: zero selects reconciliation; nonzero selects draining.

Thus the reconciliation-pointer capture occurs even when F-unknown-03 returns nonzero and draining follows. That capture does not itself read the pointed-to counter.

No atomic snapshot is established. Dependencies can change shared data or shared pointer objects. The retained configuration pointer and captured reconciliation pointer remain the addresses used by their later designated operations, regardless of subsequent pointer-object changes.

### 3. Reconciliation target

This stage occurs only when F-unknown-03 returned zero.

Read flags and requested quantity from the retained configuration block. These values are retained for target calculation; the requested quantity is the starting target.

If flag bit 0 is clear, the limit and bit 7 do not affect this stage.

If bit 0 is set:

1. Read the unsigned 32-bit limit and compare it with the retained requested quantity.
2. If that limit read is less than or equal to the requested quantity, read the limit again and use **the second read** as the intermediate quantity.
3. Otherwise use the requested quantity.
4. Apply a lower bound of 1 if retained flag bit 7 is clear, or 0 if it is set.

The limit comparison and replacement read are distinct. If the limit is stable across them, this is equivalent to taking the smaller of limit and requested quantity, then applying the lower bound. Without that stability assumption, the selected second limit read need not be bounded by either the requested quantity or the earlier limit read.

The lower bound follows limit selection. With stable values, a zero limit and clear bit 7 therefore produce 1, not zero.

Next apply the overriding rules using the retained flags:

- If bit 3 is clear, bit 5 changes a nonzero target to zero. A zero target stays zero.
- If bit 3 is set, replace the target with 1, then replace it with zero if bit 4 is also set.

Bit 4 has no effect on the bit-3-clear branch. Bit 5 does not override the bit-3-set branch. All target comparisons are unsigned 32-bit comparisons.

### 4. Reconciliation equality exit and adjustment

The immediate equality check does **not** use the pointer captured after F-unknown-03. Instead, it obtains the reconciliation pointer afresh from its shared pointer object and reads the indicated unsigned 32-bit counter.

If that read equals the target, obtain the pointer afresh again and return a new counter read. The comparison and return involve separate pointer-object accesses and separate pointee reads. Neither the address nor the returned value is guaranteed to match the comparison observation.

This equality route bypasses adjustment setup, opaque-value acquisition, both adjustment loops, opaque-value handoff, and finalization.

If the comparison is unequal:

1. Invoke **F-3b7c5b30bd55** with 22.
2. Invoke **F-f120802e6343** and retain its opaque four-byte result.
3. While the target is less than the unsigned counter at the **earlier captured reconciliation address**, invoke **F-c6c2e1b896e3**.
4. Then, while that counter is less than the target, invoke **F-a870a0416476**.
5. Pass the retained opaque result to **F-82729ee22091**.
6. Enter finalization.

Each loop condition rereads the counter at the captured address. Changes to the shared reconciliation pointer object by setup, acquisition, or adjustment dependencies do not redirect these loops.

The two loops are sequential, not an alternating correction mechanism. If an upward adjustment overshoots the target, the downward loop is not revisited. There is no final equality requirement before handoff.

No local counter increment or decrement occurs. Termination and adjustment direction depend on dependency effects or other shared-state changes.

### 5. Draining

On the draining route, capture the drain pointer. The initial zero check separately obtains the pointer from the shared drain pointer object and reads its signed 32-bit counter.

If that read is zero, return zero immediately. The progression write and any eligibility dependencies already reached remain observable; no setup or finalization occurs.

Otherwise:

1. Invoke F-3b7c5b30bd55 with 22.
2. Obtain and retain the opaque four-byte result from F-f120802e6343.
3. While the counter at the **captured drain address** is nonzero, invoke F-c6c2e1b896e3.
4. Pass the retained result to F-82729ee22091.
5. Enter finalization.

The loop uses the captured address even if dependencies change the shared drain pointer object. Its condition tests nonzero, not positivity: a negative value also causes an adjustment call.

The first loop condition occurs after setup and acquisition. If those calls make the captured counter zero, no adjustment call occurs, but handoff and finalization still do.

### 6. Finalization

The shared continuation is **F-5a0e2e961d01**, whose included behavior is described here.

Capture the finalization object’s address and the address of its comparison field at offset 704. Invoke **F-unknown-04** before inspecting finalization state.

Then compare the captured object’s signed 32-bit comparison field with the separately addressed reference value. The reference is obtained for this comparison after F-unknown-04. A dependency-induced change to the shared finalization-object pointer does not redirect the already captured object.

#### Comparison mismatch

If the values differ:

- Invoke **F-unknown-05**.
- Set the return value to 9 and return it.

There is no direct countdown update or notification on this branch.

#### Matching comparison and countdown at least 2

Read the signed 16-bit countdown for the branch test. If it is at least 2:

- Read the countdown again.
- Subtract one from that second signed value.
- Store the result into the 16-bit countdown field.
- Invoke F-unknown-05.
- Set the return value to 10 and return it.

The test and decrement use distinct reads. With a stable countdown, the decrement is from the range 2 through 32767 and remains representable as signed 16 bits.

Without stability, the second read can differ. The subtraction itself fits a signed 32-bit intermediate for every signed 16-bit input, but a second read of −32768 produces −32769, outside the signed 16-bit destination range. The supplied material does not establish the resulting narrowing behavior; wrapping must not be assumed.

#### Matching comparison, countdown below 2, discriminator zero

If the countdown test read is below 2 and the signed 32-bit discriminator is zero:

- Write zero to the 16-bit countdown.
- Write zero to the 32-bit comparison field.
- Invoke F-unknown-05.
- Set the return value to zero and return it.

The countdown write precedes the comparison-field write. Negative countdown test values reach this branch when the discriminator is zero.

#### Matching comparison, countdown below 2, discriminator nonzero

If the discriminator is nonzero:

- Invoke F-unknown-05.
- Initialize a temporary record with four-byte words 15, 0, and 22 at offsets 0, 4, and 8.
- Pass the record’s address to **F-c81f7910921f**.
- After that dependency returns, return the record’s second word as an unsigned 32-bit value.

Zero is the second word’s **pre-call initializer**, not a guaranteed return value. Because F-c81f7910921f receives the record’s address, it may modify that word. A zero return requires that its post-call value be zero.

This branch performs no direct countdown or comparison-field write.

## Side effects

### Direct writes and ordering

Direct shared writes are limited to:

- The progression destination on every normally reached route.
- The finalization countdown on the decrement and reset branches.
- The finalization comparison field on the reset branch.

Neither adjustment counter is directly written.

Established sequencing includes progression before eligibility, setup before acquisition, acquisition before loop calls, loop completion before handoff, and handoff before finalization. Finalization state inspection follows F-unknown-04. Explicit finalization writes precede F-unknown-05. Notification-record initialization and submission follow F-unknown-05.

These are source-level sequencing facts, not synchronization guarantees. The relative order of operand reads within a single expression is not independently established.

### Pointer retention and dependency effects

Retaining a pointer retains an address, not a snapshot of its pointee. Retained pointee contents may still change. Conversely, changing a shared pointer object does not redirect an already captured pointer.

No dependency is assumed to preserve shared globals, shared storage, or the notification record.

### Ownership and lifetime

The four-byte result from F-f120802e6343 is passed unchanged to F-82729ee22091 after either adjustment route completes normally. “Acquisition” and “handoff” describe data flow only; ownership, locking, restoration, and release semantics are unproven.

The notification record is temporary storage available during F-c81f7910921f and inspected afterward for its second word. No copying, retention, ownership transfer, or safe use after the enclosing call returns is established.

No direct allocation or deallocation is visible.

## Dependencies

F-unknown-01 through F-unknown-05 are document-local opaque identifiers for otherwise unmapped dependencies.

| Opaque function ID | Established use |
|---|---|
| F-unknown-01 | First eligibility query; zero permits later calls. |
| F-unknown-02 | Called between eligibility queries; result unused. |
| F-unknown-03 | Final eligibility query; followed by reconciliation-pointer capture before its result is tested. |
| F-3b7c5b30bd55 | Called with 22 before nontrivial reconciliation or draining. |
| F-f120802e6343 | Supplies an opaque four-byte value. |
| F-c6c2e1b896e3 | Called while the captured reconciliation counter exceeds target, or the captured drain counter is nonzero. |
| F-a870a0416476 | Called while the captured reconciliation counter is below target. |
| F-82729ee22091 | Receives the retained four-byte value after the loops. |
| F-5a0e2e961d01 | Shared finalization continuation described above. |
| F-unknown-04 | Called before finalization comparison. |
| F-unknown-05 | Called once on each normally completing finalization branch. |
| F-c81f7910921f | Receives the temporary record’s address; the record’s post-call second word determines this branch’s return. |

## Acceptance examples

These are conditional static examples, not executed tests. Values and pointer objects remain stable unless a change is explicitly stipulated. Dependencies return normally.

1. **Progression before immediate exit.** With progression source 3, a nonzero eligibility word, and drain counter zero, the destination receives 3 and the function returns zero without adjustment or finalization. A representable progression source of 4 instead produces destination 5.

2. **Stable lower bound.** With successful eligibility queries, only flag bit 0 set, limit zero, and requested quantity 7, the target is 1. Setting bit 7 as well makes the target zero.

3. **Changing limit.** With bit 0 set, bit 7 clear, and requested quantity 7, suppose the limit comparison reads 5 and the subsequent replacement read yields 9. The target before overrides is 9, not 5 or 7.

4. **Override precedence.** With bit 3 set and bit 4 clear, the target is 1 even when bit 5 is set. Setting bit 4 makes the target zero.

5. **Eligibility short circuit.** A set flag bit 1 makes the initial expression nonzero. None of F-unknown-01, F-unknown-02, or F-unknown-03 is invoked.

6. **Captured versus fresh reconciliation addresses.** Suppose the captured address contains 4, target is 2, and the fresh equality-check address also contains 4. If setup redirects the shared pointer elsewhere, loop conditions still read the captured address. If downward calls change that counter to 3 and then 2, exactly two downward calls occur and no upward call is required.

7. **Fresh immediate return.** If the equality-check read is 2 for target 2, but the subsequent fresh pointer lookup identifies a counter containing 8, the immediate return is 8. No adjustment or finalization occurs.

8. **Countdown reads.** With matching comparison values and both countdown reads yielding 2, the countdown is written as 1 and the return is 10. If the test reads 2 but the decrement read yields 0, the write is −1 instead; the return remains 10.

9. **Reset versus mutable notification.** With matching comparison values and countdown test value −1, discriminator zero causes countdown and comparison-field resets and a zero return. A nonzero discriminator instead submits words initially containing 15, 0, and 22. If F-c81f7910921f leaves the second word unchanged, the return is zero; if it changes that word to 37, the return is 37.

10. **Mismatch precedence.** If finalization comparison values differ, F-unknown-05 is called and the return is 9, regardless of countdown or discriminator. No direct countdown update or notification occurs.

## Uncertainties

The following remain **documented_only**:

- External pointer declarations, initialization, validity, alignment, and aliasing.
- Progression-source and additional-eligibility-value widths and signedness.
- Progression overflow and conversion behavior.
- Out-of-range signed 16-bit countdown-store behavior after differing reads.
- Domain meanings of flags, counters, statuses, and notification words.
- Dependency synchronization, global mutation, progress, overshoot, and termination.
- Notification mutation, copying, retention, and ownership rules.
- Exceptional control flow, invalid-memory behavior, and concurrent-access guarantees.

Downward adjustment, upward adjustment, reconciliation, draining, and notification are inferred intent labels. They do not strengthen the observable contract.

## Evidence

Established static evidence consists of the supplied body and opaque dependency mapping: access widths and offsets, pointer-capture positions, separate pointer and pointee accesses, conditional repeated limit and countdown reads, short-circuit gates, unsigned target comparisons, signed countdown testing, call sequencing, loop ordering, direct writes, and explicit return data flow.

In particular, the notification return is read from the second record word after the address-taking dependency call; it is not an unconditional zero return.

No runtime execution, dependency implementation inspection, or test results support stronger claims.
