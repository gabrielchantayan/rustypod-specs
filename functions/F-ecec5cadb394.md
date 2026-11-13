<!-- Trusted provenance: F-ecec5cadb394; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ecec5cadb394

## Purpose

F-ecec5cadb394 conditionally transfers a 16-bit memory value, may perform a further exceptional memory operation, and then enters an unconditional sequence of indirect calls and software interrupts.

No application-level purpose is established. The supplied representation exposes hidden machine-state inputs rather than a conventional argument-and-return interface.

Under the 32-bit machine-arithmetic interpretation used here, the preliminary operations have sharply constrained reachability. A substantial later guarded region is unreachable under that interpretation. Its displayed memory operations are therefore not additional selectable behaviors of this contract.

The behavior below is established by static inspection and arithmetic reasoning, not by execution. Unknown environmental and dependency behavior remains explicitly unresolved.

## Interface

### Inputs

The function has no declared arguments. Its behavior depends on the following incoming state:

| Contract name | Incoming location | Meaning |
|---|---|---|
| Selector | R6 | Unsigned 32-bit selection and arithmetic input |
| Comparison input | R8 | Unsigned 32-bit comparison input |
| Displacement | R9 | Unsigned 32-bit byte displacement; also used in comparisons and doubled-address arithmetic |
| Transfer origin | R10 | Address from which the selected 16-bit transfer reads |
| Adjustment | R11 | Unsigned 32-bit amount subtracted from the transfer destination to obtain a working address |
| Accumulator | R12 | Unsigned 32-bit arithmetic input |
| Entry link | Link register | Address-valued input used in preliminary conditions and the first indirect-call target |
| Stack anchor | Address of the represented incoming stack slot | Address-valued input; its contents are not read on the established reachable paths |

These locations identify dependencies visible in the representation. They do not establish a supported calling convention, register-preservation contract, or valid entry procedure.

### Widths and layout

The behavioral reductions use the following machine-level interpretation:

- Integer and address arithmetic retains the low 32 bits.
- Address right shifts operate on unsigned 32-bit representations.
- A halfword access transfers 16 bits.
- Incrementing a halfword pointer by one advances its address by two bytes.
- Offsets applied after conversion to an integer address are byte offsets.
- A halfword converted to an unsigned address is zero-extended, not sign-extended.
- A value stored as a halfword retains its low 16 bits.

These are machine-address calculations, not guarantees that arbitrary arithmetic addresses designate valid objects. Endianness, access atomicity, alignment requirements, executable-address conventions, and memory-map permissions are not established.

### Outputs and completion

There is no explicit return value or normal return path.

If every selected preliminary operation completes and all dependencies return normally, execution continues indefinitely through the dispatch sequence. A memory fault, nonreturning call, interrupt action, or other exceptional control transfer may prevent later effects.

There is no explicit validation, error result, recovery mechanism, or rollback.

### Preconditions for continued execution

Progress requires that selected reads and writes be permitted, selected indirect targets be callable, and software interrupt `0x5B` have an execution-environment-defined meaning.

These are conditions on successful execution, not checks performed by the function. In particular, arithmetic reachability of an address-zero access does not establish that address zero is mapped or accessible.

Ownership and lifetime of every referenced memory location are unknown. No ownership right or valid lifetime follows merely from an address being supplied or calculated.

## Behavior

### Initial transfer

The initial selection value is the low 32 bits of Selector multiplied by `0x80000`.

The transfer is selected exactly when this value is zero. Equivalently, under the stated width model, Selector’s low 13 bits must all be zero. Selector itself need not be zero.

For a selected transfer, the destination is Transfer origin plus Displacement, modulo \(2^{32}\). The represented ordering is significant:

1. Read a 16-bit value from Transfer origin.
2. Update the working transfer address to destination minus Adjustment, modulo \(2^{32}\).
3. Write the previously read 16-bit value to the destination.

The working-address update therefore occurs **before** the destination write, not after its completion. It does not alter the saved halfword or the already calculated destination.

If the source read fails, neither the subsequent working-address update nor the destination write is reached. If the destination write fails, the representation has already performed the working-address update. Whether that working state is externally inspectable at a fault is not established.

When Displacement is zero, the selected operation still reads and then writes the same address. It is not a no-op for access-sensitive memory.

When the selection condition is false, neither this source read nor this destination write occurs.

### Conditions for the exceptional operation

After the possible initial transfer, zero-result gates determine whether an exceptional memory operation is reached.

Under the stated arithmetic model, all of the following must hold:

- Selector’s low 13 bits are zero.
- Entry link is zero.
- Stack anchor is zero.
- Comparison input equals Displacement.
- The updated working transfer address equals Adjustment.
- Accumulator is zero.
- Twice Displacement is zero modulo \(2^{32}\).

The working-address condition is equivalently:

\[
\text{Transfer origin}+\text{Displacement}
\equiv 2\cdot\text{Adjustment}\pmod{2^{32}}.
\]

The doubled-displacement condition permits exactly two 32-bit values: `0x00000000` and `0x80000000`.

The Stack anchor condition requires some care. An earlier gate establishes only that its logical right shift by one is zero, permitting anchor values zero or one. A subsequent zero-result condition requires Stack anchor plus twice Displacement to be zero. Since twice Displacement is even, anchor one cannot satisfy that condition. The complete gate sequence therefore requires anchor zero.

The initial transfer occurs before these later gates. Failure of any later condition does not suppress or undo an already completed transfer.

These are arithmetic reachability conditions, not evidence that an ordinary execution environment can provide a meaningful zero Stack anchor.

### Exceptional state and memory effects

On the exceptional path, the function first assigns an additional machine-state location a value derived from 64 times Stack anchor. Because the complete path conditions require Stack anchor zero, the assigned value is zero.

The architectural identity and significance of that state location are unknown. Its assignment precedes the exceptional memory expression.

The memory expression uses two halfword reads through Entry link. Since Entry link must be zero on this path:

- One read at address zero supplies a zero-extended destination address.
- Another read at address zero supplies the arithmetic input.
- Fifteen is added to the arithmetic input.
- The low 16 bits of the result are written to the selected destination.

The representation does not establish the relative order of the two reads. It does not establish atomicity or guarantee that they return the same value.

If address zero contains a stable halfword \(h\), the resulting store is:

\[
\text{destination byte address}=h,\qquad
\text{stored halfword}=(h+15)\bmod 2^{16}.
\]

If the two reads return different values, the destination comes from the address-selecting read and the stored value comes from the arithmetic-source read. A single shared value must not be assumed.

Any completed initial transfer precedes both exceptional reads. Consequently, if that transfer modifies address zero or overlapping storage, the exceptional operation accesses memory after that modification, subject to the environment’s memory semantics.

The exceptional path also calculates a link-derived working address. Under the established path conditions, that value is zero. The represented update of the working link follows the exceptional store; it does not change the eventual first-call address relative to the incoming zero Entry link.

### Boundary of reachable guarded behavior

A second gated region follows the exceptional operation. Its substantive memory operations cannot be reached under the stated model.

If the exceptional operation is skipped, the gate entering that region is false.

If the exceptional operation completes, the relevant derived values are constrained as follows:

- The initial scaled Selector address is zero.
- The stack-derived comparison address is zero.
- The link-derived working address is zero.
- Incrementing the zero halfword address by one produces byte address two.

The next gates first retain a zero comparison result, then compare the address-two value against a zero-derived value. That comparison cannot produce zero. The remaining guarded chain is therefore disabled.

As a result, the later displayed pointer loads, paired 32-bit stores, signed-halfword-derived stores, and associated cursor adjustments are not reachable effects under this contract’s arithmetic interpretation.

This exclusion depends on the supplied control flow and machine-width interpretation being accurate. It is not a runtime observation.

### Repeated dispatch

After the reachable preliminary behavior, execution enters an unconditional loop.

On the first iteration, the established interaction order is:

1. Invoke F-INDIRECT-01 at Entry link plus `0xEB6`, modulo \(2^{32}\).
2. After that invocation returns, invoke F-INDIRECT-02 at `0x0857A94B`.
3. After that invocation returns, issue F-INTERRUPT-01, software interrupt `0x5B`.

F-INDIRECT-01 receives three explicit 32-bit argument patterns, in order:

- `0x00000000`
- `0x00000000`
- `0xFFFFFFA5`

The last pattern represents −91 when interpreted as a signed 32-bit integer. The callee’s interpretation is unknown.

F-INDIRECT-02 has no explicit arguments in the representation. This does not establish that ambient argument registers or stack locations are zero or irrelevant to its actual behavior.

After the interrupt returns normally, the loop assigns a fixed working target base. The next and every subsequent normally reached iteration calls F-INDIRECT-01 at `0x083D5B78`, with the same three explicit argument patterns, followed by F-INDIRECT-02 and F-INTERRUPT-01 in that order.

No dependency result is explicitly consumed. No ordinary exit condition is present. The preliminary transfer and exceptional operation are outside the loop and are not repeated by the loop itself.

## Side effects

### Explicit memory effects

The reachable explicit data-memory effects are limited to:

- A selected 16-bit source read and destination write.
- On the exceptional path, two halfword reads at address zero and a 16-bit write to an address supplied by one of those reads.

The working transfer address is updated between the initial source read and destination write. This state transition remains part of the represented ordering even when the destination write does not complete.

No reachable explicit allocation, deallocation, or ownership-transfer operation is shown. That absence does **not** establish that referenced memory is externally owned, borrowed, retained, or otherwise governed by a particular ownership contract. Ownership, lifetime, and the authority to access each location remain unknown.

### Machine state

The exceptional path writes one insufficiently identified machine-state location.

The function also changes working addresses and condition state. The complete architectural state exposed at a fault, interrupt, or nonreturning dependency is not established. In particular, a represented working-state update must not be promoted into an unsupported guarantee about a specific fault-visible register.

### Dependency effects and errors

The indirect calls and interrupt handler may modify memory, global state, machine state, resources, or control flow. Their ownership behavior and preservation guarantees are unknown; they must not be assumed to preserve globals or refrain from allocating or releasing memory.

There is no explicit error handling around these interactions. A completed earlier effect is not rolled back when a later operation fails. The unconditional repetition is not evidence of a dependency-specific retry policy.

## Dependencies

The following opaque IDs label interactions, not resolved implementation identities:

| Dependency ID | Selection | Established interaction |
|---|---|---|
| F-INDIRECT-01 | Entry link plus `0xEB6` initially; `0x083D5B78` thereafter | First call in each iteration; three explicit 32-bit argument patterns |
| F-INDIRECT-02 | `0x0857A94B` | Called after F-INDIRECT-01 returns; no explicit arguments shown |
| F-INTERRUPT-01 | Software interrupt `0x5B` | Issued after F-INDIRECT-02 returns |

The first and later F-INDIRECT-01 targets may designate different executable behavior. A repeated numeric address does not itself guarantee unchanged executable contents.

**documented_only:** Dependency return behavior, calling conventions, state preservation, memory ownership, allocation and release behavior, synchronization, and failure semantics are unresolved.

## Acceptance examples

These examples are conditional obligations derived from static evidence. None has been executed.

### Transfer omitted

Given Selector equal to one, the scaled 32-bit selection value is nonzero.

Expected behavior: no preliminary explicit data-memory access occurs; the exceptional operation is not selected; the first dispatch target is Entry link plus `0xEB6`.

### Nonzero Selector still selecting a transfer

Given Selector `0x00002000`, nonzero Entry link, readable Transfer origin containing `0xBEEF`, Displacement six, and a writable destination:

- Read `0xBEEF` from Transfer origin.
- Update the working address to Transfer origin plus six minus Adjustment.
- Write `0xBEEF` six bytes after Transfer origin.
- Skip the exceptional operation.
- Enter dispatch.

All address arithmetic wraps at 32 bits. This case distinguishes the low-13-bit selection rule from an incorrect requirement that Selector itself be zero.

### Self-transfer and write failure

Given a selected transfer and Displacement zero, the function reads the source, updates the working address to Transfer origin minus Adjustment, and attempts to write the saved value back to the source address.

If that write faults, the source read and represented working-address update have already occurred. No rollback is specified.

### Exceptional operation with stable memory

Given:

- Selector, Entry link, Stack anchor, Comparison input, Displacement, and Accumulator are zero.
- Transfer origin is `0x2000`.
- Adjustment is `0x1000`.
- Address zero contains stable halfword `0x0100`.
- Every relevant access is permitted.

Expected behavior:

- Read the halfword at `0x2000`.
- Update the working transfer address to `0x1000`.
- Write the saved halfword back to `0x2000`.
- Assign zero to the additional machine-state location.
- Perform both exceptional reads at address zero, in an unspecified relative order.
- Write `0x010F` to byte address `0x0100`.
- Begin dispatch at `0x00000EB6`.

The later guarded stores do not execute.

### Halfword overflow

Under otherwise valid exceptional conditions, stable halfword `0xFFFA` at address zero selects destination `0x0000FFFA` and produces stored halfword `0x0009`.

The stored sum wraps at 16 bits. The destination is zero-extended.

### Changing address-zero reads

Under valid exceptional conditions, suppose the address-selecting read yields `0x0100` and the arithmetic-source read yields `0x0020`.

The exceptional store writes `0x002F` to address `0x0100`. This example does not prescribe which read occurs first.

### Dispatch continuation

If all dependencies return normally, the sequence is the initial F-INDIRECT-01 call, F-INDIRECT-02, and F-INTERRUPT-01, followed indefinitely by F-INDIRECT-01 at `0x083D5B78`, F-INDIRECT-02, and F-INTERRUPT-01.

If any interaction does not return, subsequent interactions are not guaranteed.

## Uncertainties

- **documented_only — Entry convention:** No supported caller contract or complete incoming machine-state specification is supplied.
- **documented_only — Execution model:** The analysis uses 32-bit modular machine arithmetic and the supplied control flow; underlying instruction semantics have not been independently verified.
- **documented_only — Memory:** Endianness, alignment, mapping, access atomicity, ownership, and lifetime are unspecified.
- **documented_only — Exceptional reads:** Their relative order and consistency are not established.
- **documented_only — Additional state:** The architectural role of the exceptional state assignment is unresolved.
- **documented_only — Dependencies:** Preservation, resource management, termination, and failure contracts are unavailable.
- **Inferred intent:** The repeated interactions might serve persistent dispatch or another unusual control-flow purpose. No application-level interpretation is established.

## Evidence

The supplied static representation establishes the incoming-state dependencies, selected halfword transfer, working-address update before its destination write, exceptional state assignment, two-read exceptional expression, indirect-call arguments, dispatch order, interrupt number, and unconditional repetition.

The selection rule, complete exceptional-path conditions, and exclusion of the later guarded effects follow from 32-bit arithmetic, zero-result gates, carry propagation on the reachable path, and two-byte halfword-pointer scaling.

No implementation access, dependency inspection, or runtime tests have been performed. Acceptance examples describe conditional behavior, not observed results.
