<!-- Trusted provenance: F-2c0ff27bcbf3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2c0ff27bcbf3

## Purpose

The represented routine performs an ordered sequence of indirect calls, 32-bit memory reads and writes, and byte-selector reads followed by byte writes. It eventually enters an unconditional loop that continues invoking dependencies and performing selector-directed zero writes.

No application-level purpose is established. Object initialization, decoding, allocation, and protocol processing are not established interpretations.

This contract distinguishes:

- **Established static evidence:** represented operations, argument relationships, captured-value reuse, and ordering.
- **Inferred intent:** descriptive interpretations that impose no acceptance requirement.
- **documented_only:** unresolved ABI, arithmetic, memory, dependency, and execution details.

The supplied representation reports invalid instruction data, removed unreachable regions, and overlapping global symbols. This contract describes its represented path, not a verified executable control-flow graph.

## Interface

### Inputs

| Input | Represented width | Established use |
|---|---:|---|
| First positional input | Signed 32 bits | Shifted left by 23 for the initial call. |
| Second positional input | Opaque 32 bits | No explicit use in the represented body. |
| Third positional input | Unsigned 32 bits | Logically shifted right by 7 for the initial call. |
| Fourth positional input | Signed 32-bit address base | Destination base for a word write at offset `0x74`. |
| Incoming link-register value | Address-bearing execution context | Determines the initial target by addition of `0x1e8`. |

The link-register value is an implicit context input, not an established fifth positional argument. Absence of an explicit use of the second input does not establish irrelevance under an unknown machine ABI.

### Outputs and execution conditions

The represented return type is void. No return, success indicator, error indicator, or loop exit appears.

If all reached calls return and all accesses complete, the routine reaches the infinite loop. A nonreturning dependency, fault, or nonlocal transfer prevents later operations; no handler or cleanup path is represented.

Each reached target must be callable, and each reached memory access must be permitted. No finite buffer requirement follows from the explicit inputs: dependency results and selector contents also determine addresses.

### Widths and arithmetic

Word reads and writes are 32 bits. Selector reads are unsigned 8-bit reads; byte stores retain eight bits. The initial dependency has a represented 64-bit result, divided into low and high 32-bit portions. Other captured ordinary results used below are represented 32-bit values.

Unsigned right shifts are logical. The formulas below describe represented word arithmetic. Where signed multiplication, signed left shifts, conversion to signed values, or address overflow occurs, portable behavior is **documented_only**. Examples explicitly invoking a 32-bit bit-vector interpretation assume reduction modulo \(2^{32}\); that interpretation is not an established language-level overflow rule.

Endianness, alignment, address-space mapping, and access-fault behavior are unknown.

## Behavior

### Observable-call notation

The following notation specifies observable invocation sequences, not helper functions or implementation algorithms. Every invocation remains a distinct event with its own represented indirect target. Reusing a sequence shape does not imply target equality.

- **Five-call chain, P(arguments):** five calls. The first receives the listed arguments. The next four have no explicit arguments. The fifth supplies the chain’s captured result, where one is named.
- **Nine-call forwarding chain, Q(arguments; suffix):** nine calls:
  1. The listed arguments.
  2. No explicit arguments; captures a first forwarding result.
  3. That first result as its sole explicit argument.
  4. No explicit arguments; captures a second forwarding result.
  5. That second result, followed by the listed suffix arguments.
  6–8. Three separate calls without explicit arguments.
  9. No explicit arguments; supplies the chain’s final captured result.
- **Seven-call forwarding chain, R(arguments):** the listed arguments; a no-explicit-argument call capturing a result; a call receiving that result; then four no-explicit-argument calls, the last supplying the final captured result.
- **Seven-call separated chain, S(first argument; later argument):** a call receiving the first argument; a no-explicit-argument call; a call receiving the later argument; then four no-explicit-argument calls, the last supplying the final captured result.

An empty Q suffix means that its fifth call receives only the second forwarding result. Results not named below have no explicit subsequent use in the represented body.

“No explicit arguments” does not mean no hidden inputs. None of these calls may be assumed to preserve memory, globals, resources, or ownership.

### Access conventions

A **selector store at a selector address** comprises, in order:

1. An unsigned byte read at that address.
2. A byte write at the read value plus `0x11`.

Thus its destination lies between `0x11` and `0x110`, inclusive. Every selector store performs a fresh selector read, including consecutive stores using the same selector address.

The descriptions below use semantic names for captured values. Later dependency calls do not cause an explicit reread of a saved memory snapshot.

### Initial result, publication, and first descriptor

The initial call receives, in order, the third input shifted right by 7 and the first input shifted left by 23. Its target is the incoming link-register value plus `0x1e8`.

Its low result word becomes the **initial word**. Its high result word shifted left by 3 becomes the **initial expanded high word**.

The following events occur in order:

| Event | Observable behavior |
|---|---|
| Initial publication | Write the initial word at fourth-input base plus `0x74`. |
| Initial forwarding | Q(initial expanded high word, initial expanded high word, initial word, initial word; empty) captures an **initial destination index**. |
| Second publication | Write the initial word at four times the initial destination index plus `0x74`. |
| Descriptor acquisition | P(four times the initial destination index) captures the **first descriptor**. |
| Descriptor publication | Write the descriptor with its top two bits cleared and shifted right by 5 at sixteen times the descriptor plus `0x14`. |
| Destination acquisition | P(four times the first descriptor) captures a **descriptor destination index**. |
| Descriptor republication | Write the same saved masked-and-shifted-by-5 value at four times that destination index plus `0x14`. |
| Payload acquisition | P(four times the descriptor destination index) captures the **first byte payload**. |

The **first selector base** is the first descriptor with its top two bits cleared, shifted right by 6. Its computation introduces no additional memory read.

### First byte pair and global publication

The next events are:

| Event | Observable behavior |
|---|---|
| First byte write | Selector store at first selector base plus 1, writing the low byte of first byte payload shifted left by 2. |
| Intervening calls | P(first byte payload shifted left by 2) captures a **first byte continuation result**. |
| Second byte write | Fresh selector store at the same selector address, writing the low byte of first byte payload with its top two bits cleared and shifted right by 20. |
| Continuation calls | P(first byte continuation result shifted left by 2); its final result is not explicitly retained. |
| Global capture | Read the represented global associated with address `0x3c`, convert to 32 bits, and retain the **global snapshot**. |
| First global publication | Write that snapshot at four times the initial word logically shifted right by 7, plus `0x14`. |
| Intervening calls | P(global snapshot) captures a **global continuation result**. |
| Second global publication | Repeat the preceding word write using the same saved address and snapshot. |
| Index acquisition | P(global continuation result shifted left by 2) captures the **first index**. |

The global’s underlying access width is unresolved because of the overlapping-symbol warning. The conversion result used by both stores is 32 bits.

The second payload-byte expression retains original payload bits 20–27. Calls between the two selector stores can change their destinations.

### Indexed word phases

The **primary snapshot** and **secondary snapshot** below are unsigned 32-bit reads. For each index, “four-times value” and “sixteen-times value” mean products derived from that particular captured index.

The **retained shifted index** initially equals the first index shifted left by 26. It is logically shifted right by 7 during the second-index phase and again during the fifth-index phase.

| Phase | Ordered memory operations and calls |
|---|---|
| First index | Write its four-times value at its sixteen-times value plus `0x14`. Then read the primary snapshot at its four-times value plus `0x14`. Q(sixteen-times value, four-times value, retained shifted index, four-times value; empty) captures a continuation result. Repeat the original write using its saved address and value. P(four times that continuation result) captures the second index. |
| Second index | Shift the retained shifted index right by 7. Write this index’s four-times value at its sixteen-times value plus `0x14`; then read the secondary snapshot at its four-times value plus `0x14`. Q(four-times value, four-times value; empty) captures a continuation result. Write secondary snapshot shifted right by 8 to the secondary read address. P(four times the continuation result) captures the third index. |
| Third index | Write its four-times value at its sixteen-times value plus `0x14`. P(primary snapshot shifted right by 16) captures a continuation result. Write this index’s four-times value at its four-times value plus `0x14`. P(continuation result shifted left by 2) captures the fourth index. |
| Fourth index | Write its four-times value at its sixteen-times value plus `0x14`. Q(sixteen-times value; primary snapshot shifted right by 16) captures a continuation result. Repeat the original write. P(four times the continuation result) captures the fifth index. |
| Fifth index | Shift the retained shifted index right by 7 again. Write this index’s four-times value at its sixteen-times value plus `0x14`. Form the **expanded primary quantity** as eight times the primary snapshot shifted right by 8. Q(four-times value, expanded primary quantity; primary snapshot shifted right by 16) captures a continuation result. Write this index’s four-times value at expanded primary quantity plus `0x14`. P(four times the continuation result) captures the sixth index. |
| Sixth index | Write its four-times value at its sixteen-times value plus `0x14`. P(expanded primary quantity, expanded primary quantity, primary snapshot shifted right by 16, retained shifted index shifted right by another 7) captures a continuation result. Write the expanded primary quantity at expanded primary quantity plus `0x14`. P(continuation result shifted left by 2) ends this phase. |

The final call of the last chain is followed by use of an **additional register result**, distinct from an explicitly captured ordinary return. Its register convention and production semantics are unresolved.

The reads in the first and second phases occur after their corresponding first writes. Aliasing can therefore change the snapshots. Later stores use the captured snapshots despite intervening calls.

### Additional register result and final word phases

The additional register result produces a **register-derived payload** by multiplication by `0x01000000`.

| Phase | Ordered behavior |
|---|---|
| Register-derived publication | Write the payload at additional register result multiplied by `0x04000000`, plus `0x14`. P(register-derived payload) captures a destination index. Write the same payload at four times that index plus `0x14`. P(four times that destination index) captures the seventh index. |
| Seventh index | Write its four-times value at its sixteen-times value plus `0x14`. Q(four-times value; empty) captures a destination index. Write the saved four-times value at four times that destination index plus `0x14`. P(four times that destination index) captures the eighth index. |
| Eighth index | Write its four-times value at its sixteen-times value plus `0x14`. Q(four-times value; empty) captures a destination index. Write primary snapshot shifted right by 24 at four times that destination index plus `0x14`. P(four times that destination index) captures the ninth index. |
| Ninth index | Write its four-times value at its sixteen-times value plus `0x14`. P(sixteen-times value, sixteen-times value, primary snapshot shifted right by 23) captures a continuation result. Write its sixteen-times value at its sixteen-times value plus `0x14`. P(continuation result shifted left by 2) captures the second byte payload. |

The **second selector base** is the eighth index’s four-times value, with its top bit cleared, shifted right by 7.

The **primary high byte** is the primary snapshot shifted right by 24. It is retained independently of subsequent calls.

### Second byte pair and explicit zero stores

The remaining finite byte phase has this order:

| Event | Observable behavior |
|---|---|
| First payload byte | Selector store at second selector base plus 1, writing the low byte of second byte payload shifted left by 2. |
| Forwarding calls | R(second byte payload shifted left by 2, repeated three times) captures a continuation result. |
| Second payload byte | Fresh selector store at the same selector address, writing the low byte of second byte payload with its top two bits cleared and shifted right by 20. |
| Next payload acquisition | P(continuation result shifted left by 2) captures the **third byte payload**. |
| Direct byte write | Write the low byte of third byte payload shifted left by 2 at primary high byte shifted right by 7, plus `0x1c`. |
| First zero store | Selector store of zero at second selector base plus 1. |
| Separated calls | S(primary high byte shifted right by 7; third byte payload with its top two bits cleared and shifted right by 6) captures a continuation result. |
| Second zero store | Fresh selector store of zero at second selector base plus 1. |
| Loop-field acquisition | P(continuation result shifted left by 2) captures a result whose top two bits are cleared and which is then shifted right by 25, yielding the **initial loop field**. |

The direct byte destination is `0x1c` or `0x1d`. The **final shifted payload** is the retained 32-bit value of third byte payload shifted left by 2.

### Loop entry and recurring target transition

The fixed **loop selector address** is 1024 times the final shifted payload, plus 1, subject to the arithmetic qualifications above.

Before the first iteration:

1. A fresh selector store writes zero using the loop selector address.
2. One dependency receives twice the initial loop field.
3. One no-explicit-argument dependency supplies the initial **loop carry**.
4. The target selected for the first call of the first iteration is the **entrance-head target**.

The entrance-head target differs from the **steady-head target** established near the end of each iteration. This is a target transition, not merely an argument update.

Each iteration contains exactly ten calls:

| Position | Observable invocation or intervening access |
|---|---|
| Call 1 | Current head target, receiving loop carry and loop field logically shifted right by 8. |
| Calls 2–4 | Three separate calls without explicit arguments. |
| Call 5 | No explicit arguments; captures an **iteration continuation result**. |
| Access | Fresh selector store of zero using the fixed loop selector address. |
| Call 6 | Receives iteration continuation result shifted left by 2. |
| Calls 7–9 | Three separate calls without explicit arguments. |
| Call 10 | No explicit arguments; captures the **next-state result**. |
| State transition | Next carry is next-state result shifted left by 2; next field is next-state result shifted left by 26. The next iteration uses the steady-head target. |

Only the first iteration uses the entrance-head target. Every subsequent iteration uses the steady-head target, which is re-established by the same represented target-selection sequence each time.

The initial loop field is between 0 and 31, so call 1’s second argument is zero on the first iteration. Later values follow the stated recurrence. No exit condition appears.

## Side effects

The explicit effects are the ordered calls and accesses described above. No comparison, alias check, validation, rollback, synchronization, or atomic grouping is represented.

Repeated writes are distinct observable events even when address and value match. Dependencies may observe or change the destination between them. Selector reads can observe changes caused by dependencies or earlier explicit writes.

No allocation, release, ownership transfer, or reference-count operation is explicitly identifiable. This establishes neither their absence from dependencies nor continued validity of previously accessed memory after a call.

Offsets `0x14` and `0x74` do not establish an object layout: many bases are scaled integers rather than demonstrated object pointers.

A failure or nonreturning call may leave any completed prefix of the effects in place.

## Dependencies

The supplied opaque function identity is **F-2c0ff27bcbf3**. No opaque callee function IDs are supplied.

Dependencies are therefore identified here by invocation position and, for the changing loop head, local descriptive target labels. These are not invented provenance-backed function IDs.

Established dependency relationships include:

- The context-derived initial target and its represented 64-bit result.
- Every explicit argument, return forwarding, and call count specified by P, Q, R, S, and the loop table.
- Use of an additional register result after the final call preceding register-derived publication.
- Distinct entrance-head and steady-head loop targets.

Dependency purity, hidden inputs, memory preservation, ownership behavior, error signaling, and normal return remain **documented_only**. An unused ordinary result does not imply an effect-free call.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **Initial split and forwarding.** With initial low word `0x11223344` and high word 3, the first write publishes `0x11223344`. The next call receives 24, 24, `0x11223344`, and `0x11223344`. Eight further calls complete that Q chain, including both independent return-forwarding pairs.

2. **Descriptor argument relationship.** Under 32-bit bit-vector arithmetic, descriptor 8 produces a word value of zero at destination `0x94`. The immediately following P chain begins with argument 32; its remaining four calls have no explicit arguments.

3. **First-index four-argument call.** For first index 1 under the bit-vector interpretation, the first indexed write stores 4 at `0x24`, followed by a read at `0x18`. The next call receives 16, 4, `0x04000000`, and 4, in that order.

4. **Fifth-index dependency input.** If fifth index is 3 and primary snapshot is `0x00001200`, its forwarding chain begins with arguments 12 and `0x90`. Its fifth call receives its second forwarding result followed by zero.

5. **Fresh selector reads.** Selector values `0x20` and, after intervening calls, `0x40` select destinations `0x31` and `0x51`. The second store must not reuse the first selector value.

6. **Payload truncation.** Payload `0x12345678` yields bytes `0xe0` and `0x23` in either specified payload-byte pair.

7. **Captured snapshot reuse.** If the global capture produces `0x89abcdef`, both global-publication stores use those bits even if the intervening calls change the global.

8. **Direct-address boundary.** Primary high bytes `0x7f` and `0x80` select direct byte destinations `0x1c` and `0x1d`, respectively.

9. **Loop target and state.** The first iteration calls the entrance-head target with second argument zero. Under bit-vector arithmetic, next-state result 1 produces next carry 4 and next field `0x04000000`; the following iteration calls the steady-head target with second argument `0x00040000`.

10. **Interrupted execution.** If any dependency never returns, later accesses and calls are not required to occur. No cleanup behavior is established.

## Uncertainties

The following remain **documented_only**:

- Executable fidelity of the represented path and removed regions.
- Actual entry ABI, hidden register inputs, and additional-result convention.
- Callee identities and semantic contracts beyond the represented relationships.
- Signed overflow, signed left shifts, conversions, and address arithmetic outside established ranges.
- Underlying global-access width at `0x3c`.
- Endianness, alignment, permissions, and low-address memory meaning.
- Aliasing among destinations, selectors, globals, and dependency-accessed memory.
- Dependency-induced mutation, allocation, release, faults, exceptions, or nonlocal transfer.
- Higher-level format, object model, or application intent.

## Evidence

This contract uses only the supplied static representation. The call-sequence shapes preserve distinct invocations, explicit argument order, return forwarding, and placement relative to memory operations. The loop account separately preserves argument recurrence and the entrance-head-to-steady-head target transition.

No runtime tests, executable inspection, dependency tracing, or memory experiments have been performed. Acceptance examples are conditional arithmetic and ordering deductions, not observed execution results.
