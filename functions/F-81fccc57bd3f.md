<!-- Trusted provenance: F-81fccc57bd3f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-81fccc57bd3f

## Purpose

F-81fccc57bd3f conditionally transfers a 16-bit memory value, may perform a narrowly gated additional memory update, and then enters a dispatch loop containing two indirect calls followed by software interrupt `0x5B`.

The supplied representation establishes no application-level purpose, normal return path, or error-reporting protocol. It depends on incoming machine state despite having no explicit parameters.

Under the 32-bit address interpretation used in this contract, the additional memory update requires both the incoming continuation address and the numeric entry stack address to be zero. Conditional memory operations represented after that update are unreachable under the same interpretation.

These are static conclusions about the supplied representation, not observations of an executing program.

## Interface

### Explicit interface

- No explicit parameters.
- No explicit return value.
- No ordinary terminating path.
- No established calling convention.

The absence of explicit parameters does not imply an absence of inputs. Incoming machine state determines memory addresses, arithmetic gates, and the first indirect-call target.

### Implicit inputs

The following names describe semantic roles, not formal parameters or source identifiers.

| Input | Meaning |
|---|---|
| Control word | Its low byte determines whether the initial transfer occurs. |
| Source address | Starting address of the initial 16-bit read. |
| Offset word | Byte displacement used to calculate the transfer destination; also used in later gates. |
| Comparison word | Subtracted from the destination address and compared with the resulting address value. |
| Matching word | Required to equal the offset word for the additional update. |
| Addend word | Required to be zero for the additional update. |
| Continuation address | Used in exceptional-path gating and to derive the first dispatch target. |
| Entry stack address | Numeric address of the represented entry stack location; used in arithmetic. |

The entry stack address is an address value, not a value read from the stack. No direct read of the entry stack contents is established on the reachable paths described here.

### Widths and address interpretation

The directly reachable data accesses use 16-bit values occupying two bytes. The controlling scalar calculations use 32-bit words.

This contract interprets address addition, subtraction, and multiplication modulo \(2^{32}\). Displacements described as bytes are not scaled by the size of a halfword. Where the representation advances a halfword pointer by one element, the resulting byte displacement is two.

This is an explicit model for the supplied representation, not a claim that its pointer expressions are portable host-language operations. Actual address-space validity, alignment, and processor semantics remain unresolved.

No byte-order conversion is established. The initial transfer preserves the 16-bit value; the corresponding byte layout depends on the execution environment.

### Preconditions for continued execution

When the initial transfer is enabled:

- The source must support a 16-bit read.
- The calculated destination must support a 16-bit write.
- Any machine-specific alignment requirements must be satisfied.

For the additional update, the relevant reads through address zero and the selected destination write must be supported.

For dispatch to advance, each target must be callable, each dependency must return, and the interrupt must return. None of these requirements is visibly validated.

## Behavior

### Initial transfer selection and ordering

The initial transfer is enabled exactly when the low byte of the control word is zero.

The represented 32-bit multiplication by \(2^{24}\) discards all but that low byte. Therefore, a control word with nonzero upper bits can still enable the transfer.

If the low byte is nonzero:

- The initial transfer is skipped.
- The additional memory update is not reached.
- Execution proceeds toward dispatch without a directly represented data-memory access on that path.

If the low byte is zero, the transfer-related operations occur in this represented order:

1. Calculate the destination address as the source address plus the offset word, in bytes, modulo \(2^{32}\).
2. Read the 16-bit value at the source address.
3. Calculate the internal address value as the destination address minus the comparison word, modulo \(2^{32}\).
4. Write the previously read 16-bit value to the destination address.

The destination-minus-comparison calculation is therefore **after the source read and before the destination write**. It is not deferred until after the write.

This ordering describes the supplied representation; it does not independently establish physical instruction scheduling. The internal address calculation is not itself a memory read.

The transfer performs no represented sign extension or value transformation. Equal source and destination addresses still entail a read followed by a write. Partially overlapping two-byte regions are not rejected; the representation retains the read value for the later write.

The initial transfer occurs before every additional directly represented memory access and before dispatch. Consequently, its effects remain relevant even if a later gate fails. A fault during the transfer can prevent all subsequent behavior.

### Eligibility for the additional update

Under the stated 32-bit model, execution reaches the additional update only when all of the following conditions hold:

- The control word’s low byte is zero.
- The continuation address is zero.
- The entry stack address is zero.
- The matching word equals the offset word.
- Twice the offset word is zero modulo \(2^{32}\).
- The destination-minus-comparison address equals the comparison word.
- The addend word is zero.

The offset restriction permits exactly two 32-bit bit patterns:

- `0x00000000`
- `0x80000000`

The comparison condition is equivalently:

\[
2 \times \text{comparison word}
\equiv
\text{source address}+\text{offset word}
\pmod{2^{32}}.
\]

This condition requires an even destination address under the arithmetic model, but it does not establish that the corresponding memory is accessible.

The zero entry stack address is a derived requirement, not merely an assumption. An earlier gate restricts the logical right shift of that address by one bit to zero, leaving only zero or one. A later gate requires the stack address plus twice the offset word to be zero. Since twice a 32-bit word is even modulo \(2^{32}\), a stack address of one cannot satisfy that gate.

Along the successful gating path, the control-dependent contribution to the addend calculation and its incoming carry are zero. Requiring that calculation to produce zero therefore requires the addend word itself to be zero.

These conditions characterize arithmetic eligibility. They do not guarantee that the preceding transfer or the additional update completes.

### Additional update

When the gates succeed, the representation orders the following actions before dispatch:

- Write an auxiliary machine-state location with a value derived by multiplying the entry stack address by 64.
- Calculate an adjusted continuation address using a byte displacement of 64 times the entry stack address.
- Perform the additional 16-bit memory update.
- Retain the adjusted continuation address for subsequent control-flow calculations.

The identity and hardware meaning of the auxiliary machine-state location are unknown.

The additional memory update uses halfword contents obtained through the incoming continuation address in two roles:

- A 16-bit value, zero-extended to 32 bits, selects the write address.
- A 16-bit value has 15 added to it; the result is truncated to 16 bits for the write.

The address selected from a halfword therefore lies between `0x00000000` and `0x0000FFFF`, inclusive. It is not necessarily aligned.

The representation does not establish an evaluation order between the source evaluations used for the destination and the written value. It also does not justify a particular physical read count. If the memory changes between evaluations, the destination selection and value calculation are not guaranteed to use identical contents.

For every arithmetically eligible state:

- The entry stack address is zero.
- The incoming continuation address is zero.
- The auxiliary-state value is zero.
- The calculated continuation displacement is zero.
- The continuation address remains zero.
- The update requires reading through address zero.

On an environment where address zero is unreadable, this path cannot complete normally. No visible null-address check or recovery mechanism intervenes.

The earlier transfer can affect memory used by this update. No disjointness between the transfer regions, address zero, and the selected update destination is established.

### Later conditional regions

The supplied representation contains further conditional operations, including pointer loads, signed halfword interpretation, paired stores, and additional memory writes. They are unreachable under the stated model.

The relevant control-flow facts are:

- Without execution of the additional update, the enabling condition for these regions is false.
- When the update executes, the earlier gates have forced the relevant base address quantities to zero.
- A halfword-pointer advancement then produces numeric address two.
- The next decisive equality gate compares a result of two against zero and fails.

No directly represented memory operations in those later regions are therefore included in the reachable contract.

This conclusion depends on the supplied control flow and the 32-bit address interpretation. It does not resolve whether the representation faithfully captures an underlying executable.

### Dispatch loop

On reaching dispatch, the represented order is:

1. Invoke F-unknown-01.
2. If it returns, invoke F-unknown-02.
3. If it returns, issue software interrupt `0x5B`.
4. If the interrupt returns, install the fixed base used for the next iteration and repeat.

The first F-unknown-01 target is the effective continuation address plus `0xEB6` bytes, modulo \(2^{32}\).

On every reachable path established above, the effective continuation address equals its incoming value. The only reachable adjustment adds zero.

F-unknown-01 receives three represented 32-bit argument words, in order:

- `0x00000000`
- `0x00000000`
- `0xFFFFFFA5`

The last bit pattern represents −91 if interpreted as signed 32-bit data. The callee’s actual argument types are unknown.

F-unknown-02 is targeted at `0x0857A94B`. The representation supplies no explicit arguments to this call. This does not imply that machine argument locations are cleared, preserved, or irrelevant.

After a returning interrupt, the next and subsequent F-unknown-01 targets are `0x083D5B78`. The same three represented argument words are supplied each time.

There is no visible loop exit. Nonreturning dependencies, faults, or interrupt-driven control transfers can prevent later operations. The loop establishes represented call order, not successful progress or dependency preservation guarantees.

## Side effects

### Direct memory effects

The reachable direct memory effects are:

- When enabled, a 16-bit source read followed by a 16-bit destination write, with the internal address calculation between them.
- When exceptionally eligible, source evaluations through address zero followed by a 16-bit write to the selected zero-extended halfword address.

No direct reachable allocation, release, reference-count adjustment, or ownership transfer is established.

Access to memory does not establish ownership, exclusive access, lifetime extension, or permission to access adjacent bytes.

### Machine state and dependencies

The additional update writes an unidentified auxiliary machine-state location. Its assigned value is zero on every eligible path under this model.

Temporary arithmetic assignments do not establish caller-visible register outputs or preservation rules.

Indirect calls and the interrupt may alter memory, globals, registers, ownership, or control flow. No preservation behavior may be assumed from their lack of resolved identities.

### Errors and abnormal execution

There is no explicit status return, error callback, exception translation, or fault recovery.

Invalid accesses, alignment violations, invalid call targets, and interrupt failures are delegated to the execution environment. The representation does not establish whether such events terminate execution, transfer control, or permit resumption.

## Dependencies

The following opaque IDs distinguish unresolved dependency roles. They do not establish recovered external identities.

| Opaque ID | Established contract |
|---|---|
| F-unknown-01 | First indirect call in each iteration. Initially uses the continuation-derived target; later uses `0x083D5B78`. Receives the three represented 32-bit argument words. |
| F-unknown-02 | Second indirect call in each iteration, targeting `0x0857A94B`. Explicit argument semantics are unknown. |
| F-unknown-03 | Execution-environment handling of software interrupt `0x5B`. |

No dependency return value is visibly consumed.

Termination behavior, calling conventions, global-state preservation, memory effects, and ownership effects are documented_only unknowns.

## Acceptance examples

These are conditional static cases, not executed tests. Continued execution always depends on supported accesses and returning dependencies.

### Transfer disabled

Given a control word whose low byte is nonzero:

- No directly represented data-memory transfer occurs before dispatch.
- F-unknown-01 is targeted at the incoming continuation address plus `0xEB6`, modulo \(2^{32}\).
- F-unknown-02 follows only if F-unknown-01 returns.
- Interrupt `0x5B` follows only if F-unknown-02 returns.

### Transfer enabled with nonzero upper control bits

Given:

- Control word `0x12345600`.
- Source address `0x1000`.
- Offset word `0x20`.
- Comparison word `0x100`.
- Source halfword `0xBEEF`.
- A nonzero continuation address.

The represented transfer sequence is:

1. Calculate destination `0x1020`.
2. Read `0xBEEF` at `0x1000`.
3. Calculate internal address `0x0F20`.
4. Write `0xBEEF` at `0x1020`.

The additional update is skipped because the continuation address is nonzero.

### Identical source and destination

Given an enabled transfer and offset zero, the source halfword is read, the internal subtraction is calculated, and the captured halfword is written back to the same address.

The transfer is not omitted merely because the addresses coincide.

### Address wraparound

Given an enabled transfer, source address `0xFFFFFFF0`, and offset `0x20`, the destination calculation produces `0x00000010`.

This arithmetic result does not establish that either access is valid.

### Additional update with truncation

Given:

- An enabled transfer.
- Zero continuation address and entry stack address.
- Zero offset word, matching word, and addend word.
- Source address `0x2000`.
- Comparison word `0x1000`.
- Stable halfword contents `0xFFF8` at address zero.
- Supported accesses throughout.

Expected behavior:

- The initial transfer reads and writes at `0x2000`, with the internal address calculation between those accesses.
- The additional update is arithmetically eligible.
- The auxiliary-state write assigns zero.
- The continuation address remains zero.
- The selected update destination is `0x0000FFF8`.
- Adding 15 and truncating to 16 bits produces `0x0007`, which is written there.
- The later conditional memory regions remain unreachable.
- If dispatch is reached, its first F-unknown-01 target is `0x00000EB6`.

The stable-memory condition is necessary to make both source roles use the stated value.

### Repeated dispatch

If both indirect calls and the interrupt return on every iteration:

- The first iteration uses the continuation-derived F-unknown-01 target.
- Subsequent iterations use `0x083D5B78`.
- Every iteration orders F-unknown-01 before F-unknown-02 before interrupt `0x5B`.
- No normal exit occurs.

## Uncertainties

The following remain documented_only:

- Actual processor, endianness, alignment rules, and address-space behavior.
- Whether the incoming state is valid at a genuine function boundary.
- Fidelity of the supplied control-flow representation.
- Calling conventions and argument types of the indirect dependencies.
- Identity and meaning of the auxiliary machine-state location.
- Interrupt `0x5B` behavior.
- Physical read count and evaluation ordering for the additional update’s source roles.
- Behavior of device-backed or externally changing memory.
- Fault handling and possible resumption.
- Dependency effects on globals, memory, registers, ownership, and lifetime.

An application-level interpretation is not established. The presence of transfers and repeated dispatch does not justify assigning a higher-level algorithm or purpose.

## Evidence

This contract is based solely on static examination of the supplied representation.

Direct static evidence establishes the low-byte-dependent transfer, halfword access widths, transfer operation ordering, arithmetic gates, auxiliary-state assignment, represented call arguments, call-target calculations, interrupt number, and unconditional repetition.

The exceptional eligibility conditions, unchanged effective continuation address, and unreachability of the later conditional memory regions are derived conclusions under the explicit 32-bit model.

Inferred intent is limited to descriptive terms such as “transfer,” “additional update,” and “dispatch.” No application-level intent is asserted.

No runtime tests, memory observations, dependency inspection, or independent implementation validation have been performed.
