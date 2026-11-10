<!-- Trusted provenance: F-deb4c7cd2bdf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-deb4c7cd2bdf

## Purpose

F-deb4c7cd2bdf performs an ordered sequence of memory accesses, dependency calls, pointer reconstruction, and repeated processing. The supplied representation contains no normal return path. Execution can continue indefinitely, encounter an invalid access, stop in a dependency, or reach a software breakpoint.

No application-level purpose is established. The evidence does not justify identifying the operation as initialization, allocation, reset handling, or device configuration.

This is a static behavioral contract. It distinguishes represented operations from arithmetic interpretations that require additional execution semantics. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has five argument positions:

| Position | Meaning established by the representation |
|---|---|
| 1 | Base pointer, used both as an address and as a numeric value. |
| 2 | 32-bit value with no visible use. |
| 3 | Unsigned 32-bit seed used in arithmetic, address formation, and stored data. |
| 4 | 32-bit value with no visible use. |
| 5 | Signed 32-bit offset used with an incoming working pointer. |

The unused argument positions have no established effect within the visible body. Their significance to the actual calling convention is unknown.

### Hidden entry inputs

Two incoming values are consumed before any represented assignment:

- A **working pointer**, used numerically during setup and subsequently dereferenced to read a 32-bit word.
- A **mask word**, used as stored data and combined with the seed.

These are effective inputs. The five explicit arguments alone do not determine the represented behavior.

### Outputs and termination

There is no represented return value, successful completion condition, or ordinary return path.

Observable outputs are memory accesses, dependency interactions, and execution interruptions. Reaching the bottom of the represented processing body causes continued iteration rather than completion.

### Widths and arithmetic

The represented address model uses 32-bit pointer values. Memory operations include 8-bit, 16-bit, 32-bit, and 64-bit stores, together with 32-bit reads. Selected dependencies produce 64-bit results whose halves are used separately.

Explicitly unsigned 32-bit addition, multiplication, and left shifts retain the low 32 bits. Unsigned right shifts are logical. Narrow stores write their specified width; exact signed narrowing conventions are not independently established.

Several operations instead pass through signed 32-bit values:

- Doubling the numeric base and adding `0x30`.
- Adding the entry working pointer, offset, and eight, then doubling that result.
- Adding `0x50` to a signed interpretation of a dependency-result high half.
- Multiplying a signed interpretation of a dependency-result low half by `0x20`.
- Doubling a signed interpretation of another returned address.
- Left-shifting signed interpretations of the destination and working pointer by two and eight.
- Signed shifts used for the branch discriminator and breakpoint continuation.

Signed overflow and invalid signed shifts are not specified by the supplied evidence. Conversion of a result to unsigned storage does not, by itself, establish wrapping semantics for the preceding signed operation.

Where this document discusses a **32-bit bit-vector interpretation**, that is an explicit conditional assumption, not a demonstrated property of execution. Memory byte order, alignment requirements, atomicity, and hardware-visible ordering remain unknown.

## Behavior

### Initial writes and setup

The following accesses occur in represented order, assuming each preceding operation completes:

1. Double the signed numeric interpretation of the base address. At that result plus `0x30`, write a halfword derived from the doubled value. Signed arithmetic may prevent a unique address or result from being established.
2. Write the 32-bit value `0x20000000` to fixed address `0x20000040`.
3. Write a zero byte at the base.
4. Write the mask word as a 32-bit word at the base.
5. Write `0x40` as a 32-bit word at base offset `0x28`.
6. Write the numeric base address as a 32-bit word at base offset `0x2C`.

The byte store and following word store are separate operations. A final-memory image does not capture their complete observable behavior.

The function establishes these working quantities:

- **Accumulator:** mask word bitwise-AND seed.
- **Scale:** seed multiplied by eight, minus two, using unsigned 32-bit arithmetic.
- **Stride:** seed multiplied by `0x02000000`, using unsigned 32-bit arithmetic.
- **Payload:** seed shifted left by 28 bits, using unsigned 32-bit arithmetic.
- **Destination:** address formed from seed multiplied by `0x01000000`, using unsigned 32-bit arithmetic.
- **Companion destination:** initially equal to the destination.
- **Phase target:** initially F-000000000001.

Before processing begins, the function also performs the following accesses:

1. Form a signed intermediate from the numeric entry working pointer, the signed offset, and eight. Write that intermediate as a 32-bit word at twice its value. Both formation and doubling are signed-overflow-sensitive.
2. Write the original seed as a 32-bit word at base offset `0x04`.
3. Write the low byte of the scale at destination offset `0x14`.

The signed-intermediate store precedes the seed and destination-offset stores.

### Re-entry writes and argument initialization

Initial entry and one later branch pass through this ordered sequence:

1. Write a zero byte at the current destination.
2. Write a halfword derived from the scale at the same address.
3. Write the payload as a 32-bit word at the same address.
4. Add the current stride to the accumulator, modulo \(2^{32}\).
5. Write the updated accumulator as a 32-bit word at the companion destination.

Initially, both destinations are identical. The accumulator store therefore follows the payload store at the same address, unless an earlier access interrupts execution.

After these writes, processing reaches an initializer that sets the next phase-call argument to zero.

A short-result iteration also returns to this initializer, without repeating the re-entry writes. Breakpoint continuation does **not** return to this initializer.

### Common iteration

At the start of an iteration, the function saves the low byte of the current stride. That saved byte remains the byte used later in the iteration, even after the stride changes.

The current phase target is called with a represented 32-bit argument:

- Zero after initial entry.
- Zero after the short-result path.
- Zero after returning through the re-entry writes.
- The previously computed signed payload-right-shift-by-29 intermediate after breakpoint continuation.

The phase call produces a represented 64-bit result. The function then performs these operations in order:

1. Use the low half as an address and write a halfword derived from that same low half there.
2. Multiply the scale by `0x40`, modulo \(2^{32}\), and write the resulting 32-bit value at the numerically identical address.
3. Read a 32-bit word through the current working pointer. Replace the stride with that word.
4. At address scale plus four, write a halfword formed by narrowing the old working pointer to a signed halfword and adding four.
5. Read a 32-bit pointer from an address formed by interpreting the phase result’s high half as signed and adding `0x50`. Replace the working pointer with the loaded value.
6. Call F-000000000005.
7. Call F-000000000006.
8. Call F-000000000007 with the newly read stride.
9. Call F-000000000008 with that stride logically shifted right by three.

F-000000000008 returns a represented 64-bit result:

- Its high half becomes the destination.
- Its low half is interpreted as a signed 32-bit integer and multiplied by `0x20`. The result is assigned to an unsigned 32-bit quantity, called the **scaled result** here.

The signed multiplication is important. For signed operands from `−0x04000000` through `0x03FFFFFF`, multiplication by `0x20` remains within signed 32-bit range. Outside that range, the supplied representation does not establish a reliable result without additional arithmetic semantics.

Once a scaled result is available, the function writes it as a 32-bit word at the numerically identical address. Only afterward does it compare the result, as unsigned, with `0x14`.

#### Short-result path

If the scaled result is below `0x14`, the function:

- Selects F-000000000002 as the phase target.
- Returns to the argument initializer.
- Starts another iteration with argument zero.

It does not return from the function or repeat the re-entry writes.

For non-overflowing multiplication, the unsigned result is a multiple of 32; being below `0x14` therefore means it is zero. In that defined range, this occurs only when the signed low half is zero.

Under an additionally assumed wrapping 32-bit bit-vector interpretation, the same comparison selects zero exactly when the original low half’s low 27 bits are zero. That broader equivalence is conditional: it must not be applied unconditionally to overflowing signed operands.

In either interpretation, whenever the scaled result is zero, a word store of zero to address zero precedes the branch.

### Extended processing path

If the scaled result is at least `0x14`, the function continues in this order:

1. Write the current scale as a 32-bit word at scaled-result address plus `0x58`.
2. Write a halfword derived from the current working pointer at address scale plus `0x0E`.
3. Replace the stride with the scale shifted left by two, using unsigned 32-bit arithmetic.
4. Write the previously saved byte to a fixed global byte whose label suggests reset-related use; that purpose is not established.
5. Call F-000000000009 with the numeric working pointer logically shifted right by two.
6. Call F-000000000010.

The last call returns a represented 64-bit value. Its low half supplies a base address, and its high half becomes the companion destination.

The function then performs these writes:

1. Write the saved byte at twice the signed interpretation of the returned low-half address. The doubling is signed-overflow-sensitive.
2. Write the low byte of the returned high half at low-half address plus eight.
3. Write the destination from F-000000000008 as a 32-bit pointer value at twice the accumulator, using unsigned arithmetic for that address calculation.
4. Write the complete 64-bit result from F-000000000010 at its low-half address.

The final store may overlap earlier accesses. Neither its byte order nor whether it constitutes one hardware transaction is established.

Next, the function updates working state and memory:

1. Replace the accumulator with the result of interpreting the destination as signed and left-shifting by two, then assigning to unsigned storage.
2. Replace the scale with the result of interpreting the working pointer as signed and left-shifting by eight, then assigning to unsigned storage.
3. Write the current stride as a 32-bit word at the destination.
4. Write the new scale at destination plus four.
5. Write the numeric working pointer at destination plus eight.
6. Replace the payload with the current stride multiplied by `0x100`, using unsigned 32-bit arithmetic.

Neither signed left shift is unconditionally a wrapping operation. For a nonnegative signed operand, the shift-by-two result is representable through `0x1FFFFFFF`; the shift-by-eight result is representable through `0x007FFFFF`. Other cases require semantics not supplied here.

### Discriminator and breakpoint continuation

The branch discriminator is formed from the working pointer through:

- A signed interpretation and left shift by 31.
- A subsequent signed right shift by 32.
- Conversion to unsigned 32-bit form.
- A logical right shift by 20.

The full-width right shift, together with the signed left shift, prevents a portable numeric interpretation. The discriminator must remain unresolved.

Before the branch, the destination is set to address zero.

If the discriminator is zero:

- The phase target is F-000000000003.
- Execution returns to the re-entry writes.
- Those writes use destination zero and the companion destination from F-000000000010.
- The subsequent argument initializer sets the phase-call argument to zero.

If the discriminator is nonzero:

1. Write it as a 32-bit word at payload address plus `0x20`.
2. Select F-000000000004 as the phase target.
3. Compute an intermediate by interpreting the payload as signed 32-bit and right-shifting it by 29.
4. Invoke F-000000000011, the software-breakpoint operation, with immediate four.

If execution resumes, the scale is replaced by the companion destination interpreted as signed, right-shifted by 18, and assigned to unsigned storage.

Control then continues directly to the next iteration within the inner loop. Its phase call receives the saved payload-right-shift-by-29 intermediate—not zero. For negative signed payloads, that argument depends on the unresolved signed-right-shift convention. Updating the scale after the breakpoint does not recompute the argument.

## Side effects

The function accesses raw addresses derived from inputs, mutable memory, and dependency results. No bounds, null, alignment, permission, or validity checks are represented.

Effects include overlapping stores of different widths, fixed-global writes, pointer reads, possible zero-address accesses, external calls, and a breakpoint.

Aliasing can change later observations. In particular, earlier stores can affect memory subsequently read through the working pointer or a reconstructed pointer. Dependencies may also modify globals or accessible memory; preservation is not assumed.

No visible allocation, release, ownership transfer, lifetime guarantee, cleanup, rollback, or error-reporting protocol is established. Dependencies may allocate or release resources, but their behavior is unknown. An interruption may leave any completed prefix of accesses applied.

## Dependencies

These opaque IDs distinguish represented call targets or operations; they do not prove distinct implementations.

| ID | Established interaction |
|---|---|
| F-000000000001 | Initial phase target; receives zero and produces a represented 64-bit result. |
| F-000000000002 | Phase target following the short-result path; receives zero after argument initialization and produces a represented 64-bit result. |
| F-000000000003 | Phase target following extended-path re-entry writes; receives zero after argument initialization and produces a represented 64-bit result. |
| F-000000000004 | Phase target after breakpoint continuation; receives the prior signed payload-right-shift-by-29 intermediate and produces a represented 64-bit result. Negative-payload argument semantics remain unresolved. |
| F-000000000005 | Called after working-pointer replacement; no explicit arguments or consumed result represented. |
| F-000000000006 | Called immediately afterward; no explicit arguments or consumed result represented. |
| F-000000000007 | Receives the newly read 32-bit stride; no consumed result represented. |
| F-000000000008 | Receives stride logically shifted right by three; returns the 64-bit value used for destination and signed multiplication. |
| F-000000000009 | Receives numeric working pointer logically shifted right by two; no consumed result represented. |
| F-000000000010 | Returns the 64-bit value used for companion destination and subsequent stores. |
| F-000000000011 | Software breakpoint with immediate four; continuation behavior unknown. |

Hidden arguments, calling conventions, memory effects, ownership effects, and non-returning behavior are unknown. “No explicit arguments” does not mean no effective inputs.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes preceding operations complete and relevant addresses permit the described accesses.

1. **Separate initial accesses.** With mask `0x12345678`, the base receives a zero-byte store followed by a word store of `0x12345678`. Observing only the final word is insufficient.

2. **Initial destination aliasing.** With seed one and mask one, both destinations are `0x01000000`. The initial payload is `0x10000000`, stride is `0x02000000`, and accumulator is one. The re-entry sequence ends with a word store of `0x02000001` there, after its byte, halfword, and payload stores.

3. **Zero seed.** Scale becomes `0xFFFFFFFE`; destination, stride, and payload become zero. The destination-offset byte store targets `0x14`, followed by re-entry stores at zero. No seed validation bypasses these accesses.

4. **Overflow-sensitive short result.** If F-000000000008 returns low half `0x08000000`, its signed multiplication by `0x20` exceeds signed 32-bit range. No unconditional result follows. **If wrapping bit-vector multiplication is assumed**, the result is zero, a zero word is stored at zero, and the next phase call uses F-000000000002 with argument zero.

5. **Non-overflowing extended result.** A returned low half of one produces `0x20` without signed overflow. The function writes `0x20` at address `0x20` before entering the extended path.

6. **Saved byte precedes stride replacement.** If the incoming stride ends in `0xAB` and the working-pointer read returns a word ending in `0x34`, the extended path writes `0xAB` to the reset-labelled global byte. F-000000000007 receives the newly read word.

7. **Breakpoint continuation preserves the computed argument.** Conditional on reaching the nonzero discriminator branch with payload `0x20001800`, the signed right shift by 29 produces one. If the breakpoint resumes, the next F-000000000004 call receives one, not zero, after the scale update.

8. **Negative continuation argument remains conditional.** With payload `0x80001800`, the signed interpretation is negative. Under arithmetic signed right shift, the next argument is `−4`, represented in 32 bits as `0xFFFFFFFC`. That value is not unconditional because the signed-right-shift convention is unknown.

## Uncertainties

The following remain **documented_only**:

- Actual ABI, hidden input origins, pointer model, and dependency clobbers.
- Signed overflow, narrowing, pointer-to-signed conversion, and signed-shift semantics.
- Overflowing signed multiplication used to form the scaled result.
- Signed destination and working-pointer left shifts used to update state.
- The discriminator’s full-width shift and resulting branch reachability.
- Memory mapping, accessibility of address zero, alignment, byte order, atomicity, and hardware ordering.
- Dependency implementations, global preservation, ownership, failures, and termination.
- Breakpoint handling and whether execution can resume.
- Completeness of the represented control flow: an attached warning reports removed unreachable material.
- Fixed-global layout: another warning reports overlapping symbols.
- Application-level intent, including any reset semantics.

## Evidence

Established static evidence consists of the represented access widths, operation order, constants, signed and unsigned conversions, call sequence, branch placement, argument-initialization boundaries, and absence of a normal return.

Inferred intent is deliberately limited: descriptive names organize working state but do not establish data structures or a higher-level protocol.

No dependency implementations, machine instructions, memory map, ABI documentation, or execution traces were supplied. The acceptance examples are conditional arithmetic and control-flow consequences. No runtime tests have been performed.
