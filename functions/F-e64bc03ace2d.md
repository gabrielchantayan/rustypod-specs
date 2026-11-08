<!-- Trusted provenance: F-e64bc03ace2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e64bc03ace2d

## Purpose

F-e64bc03ace2d is a stateful, potentially nonterminating routine that combines indirect dependency calls, address calculations, and writes through derived addresses. It has two visible return conditions:

- An early condition based on bit 15 of the second explicit input after adding ten.
- A later condition based on a fixed global data item being zero.

Both conditions use the same return construction. Otherwise, the routine repeats with updated explicit state and a changed target for its first dependency call.

The established behavior is low-level memory and control-flow behavior. The supplied evidence does not establish an application-level purpose, such as parsing, rendering, allocation, or object management.

## Interface

### Explicit inputs

The visible interface has two explicit inputs:

| Input | Width and interpretation | Role |
|---|---|---|
| First input | Unsigned 32-bit value | Shifted right logically by 25 before the first iteration; later replaced by a derived value. |
| Second input | Signed 32-bit value in the supplied representation | Used both as an address basis and as an arithmetic operand; replaced after each completed continuing iteration. |

The explicit signature is not a complete description of the routine’s requirements. The body also consumes:

- Two incoming register-derived values whose difference initializes carried state.
- A further incoming register-derived value used as an address offset.
- An incoming link-register value used to compute the first indirect call target.
- A stack-region address passed to a dependency.
- Additional register outputs from dependencies that are not represented as ordinary return values.

These values have no established public parameter names, validity constraints, or ownership rules.

### Result

The visible result is 64 bits, constructed from two 32-bit halves:

- The low half is twice an exit-specific 32-bit value, retaining the low 32 bits.
- The high half is a 32-bit read from a fixed data cell, reduced by 84, retaining the low 32 bits.

The return is a concatenation of halves, not an addition of a shifted high value to an unbounded low value. Overflow from forming the low half does not carry into the high half.

The high-half data cell is read at the common exit, after all dependency calls and writes that precede that exit.

### Arithmetic and memory interpretation

The source representation uses 32-bit integers, pointer-like integer values, 16-bit stores, and an 8-bit load. This contract describes the visible operations using 32-bit bit-vector arithmetic where wraparound is needed to make their low-level meaning explicit.

Signed right shifts are treated as arithmetic shifts for the described bit-level examples. The supplied representation alone does not establish a portable language-level overflow contract. Exact machine instruction semantics remain a documented uncertainty.

No alignment, accessible-address range, byte order, concurrency model, or exception-delivery mechanism is established.

## Behavior

### Initial state

Before entering the loop:

1. A carried 32-bit value is initialized to the difference between two incoming register-derived values.
2. The first explicit input is shifted right logically by 25. Its initial transformed value is therefore between 0 and 127 inclusive.
3. The second explicit input remains unchanged.

The loop has no visible iteration limit.

### Initial calls and unconditional per-iteration write

Each iteration begins with these operations, in order:

1. Invoke F-U01 with the current first-input state, current second-input state, and an address in this routine’s stack region.
2. Invoke F-U02 and obtain a visible 64-bit result.
3. Add ten to the current second-input state, producing a 32-bit adjusted value.
4. Store the carried 32-bit value at byte offset 62 from the current second-input state.
5. Evaluate the first exit condition.

The F-U01 target for the first iteration is the incoming link-register value with its bottom two bits cleared, plus 1,192. Before a continuing iteration completes, the link-register state used by this calculation is replaced with a fixed value. Consequently, subsequent iterations do not necessarily call the same target as the first iteration.

The store at offset 62 precedes the first exit decision. An input that satisfies the early return condition does not bypass that store or the two preceding calls.

### First exit condition

The visible condition tests whether shifting the adjusted second-input value left by 16 produces a negative 32-bit signed result. Under the bit-vector interpretation, this is equivalent to testing whether bit 15 of the adjusted value is set.

It is not an ordinary test that the adjusted value itself is negative.

If this condition holds:

- The low 32 bits of the F-U02 result become the exit-specific value.
- Control proceeds directly to the common return construction.
- None of the later stores or dependency calls in the iteration occur.

### Operations after passing the first condition

If the first condition does not hold, the routine continues in the following order.

1. **Store a halfword through the F-U02 result.**  
   Interpret the upper 32 bits of that result as an address basis. At byte offset 60, store the low 16 bits of the result.

2. **Invoke F-U03.**  
   No explicit arguments or ordinary result are represented.

3. **Invoke F-U04.**  
   Its visible 32-bit result is used as an address basis.

4. **Store a fixed halfword.**  
   At byte offset 2 from the F-U04 result, store the 16-bit value 38.

5. **Calculate an address component.**  
   Shift the adjusted second-input value left by 28 in 32-bit width, then shift that signed result right arithmetically by 9. Call this the scaled component. Only the bottom four bits of the adjusted input contribute to this component under the stated bit-vector interpretation.

6. **Form an intermediate sum and snapshot a global word.**  
   Add the scaled component to the adjusted second-input value. Read a fixed 32-bit global word, designated G-XOR.

7. **Invoke F-U05.**  
   Its represented arguments are:
   - Twice the F-U04 result, in 32-bit width.
   - The adjusted second-input value shifted left by 16, then reduced by 81, in 32-bit width.

8. **Finish the derived address and offset.**  
   Exclusive-OR the intermediate sum with the previously read G-XOR value. This yields the derived address. Add 47 to the scaled component to produce a derived offset.

The G-XOR value used here is the value read before F-U05. A write to G-XOR by F-U05 does not, by itself, change that already captured value.

### Derived-address operations

The routine next performs the following ordered actions:

1. Save the derived address into a visible 32-bit stack slot at byte offset 304 from the represented stack origin.
2. Read one byte at offset 1 from the derived address.
3. Invoke F-U06 with:
   - A fixed address-valued argument, numerically `0x085734e8`.
   - The byte just read.
   - The derived offset.
   - An additional register-derived value associated with the earlier dependency sequence.
4. Store the derived offset as a 32-bit word at byte offset 36 from the derived address.
5. Invoke F-U07.

The byte read occurs before F-U06. The word store occurs after F-U06. These operations must not be treated as a single atomic update or reordered across that call.

The meaning of the fixed address-valued argument is unknown. It is not established to identify readable text, an object, or a callable target.

### Final calls, continuation state, and second exit

Following F-U07:

1. Add 48 to an additional register output associated with F-U07, producing a working 32-bit value.
2. Replace the first-input state with twice the derived address, in 32-bit width.
3. Form a carried calculation by multiplying the working value by two, multiplying by the working value again, and multiplying by two again, with 32-bit intermediate widths. Its low-32-bit mathematical result is four times the square of the working value.
4. Invoke F-U08 with the derived address plus the incoming register-derived address offset.
5. Read a fixed 32-bit global word, designated G-ARG, and invoke F-U09 with that value.
6. Replace the link-register state used for the next iteration’s F-U01 target with a fixed value.
7. Invoke F-U10.
8. Read a fixed global data item, designated G-GATE, and test it against zero.

If G-GATE is zero:

- Arithmetically shift an additional register output associated with F-U10 right by 9.
- Use that result as the exit-specific value.
- Proceed to the common return construction.
- Do not perform the final calculated-address word store.

If G-GATE is nonzero:

- Store the carried calculation as a 32-bit word at the address obtained by doubling the working value and adding 52.
- Replace the second-input state with the additional register output associated with F-U10.
- Repeat the loop.

On this continuing path, the carried calculation becomes the value stored at offset 62 from the second-input state in the next iteration.

The final word store uses the working value derived after F-U07, not the right-shifted F-U10 output used on the zero-gate exit path.

### Common return

Both exits converge on the same return construction:

1. Read the fixed 32-bit data cell designated G-RETURN.
2. Subtract 84 to form the upper return half.
3. Double the exit-specific value to form the lower return half.
4. Return the concatenated 64-bit result.

No cleanup or error-status normalization is visible at this exit.

## Side effects

### Observable memory access order

On a continuing iteration, the explicit non-call memory accesses appear in this order:

1. A 32-bit store at current second-input state plus 62.
2. A 16-bit store at upper F-U02 result plus 60.
3. A 16-bit store at F-U04 result plus 2.
4. A 32-bit read of G-XOR.
5. A 32-bit stack-slot write containing the derived address.
6. An 8-bit read at derived address plus 1.
7. A 32-bit store at derived address plus 36.
8. A 32-bit read of G-ARG.
9. A read and zero comparison of G-GATE.
10. A 32-bit store at twice the working value plus 52.

The early exit stops after item 1 and then reads G-RETURN. The later exit stops after item 9 and then reads G-RETURN.

Dependency calls occur between these accesses as specified in the behavior section. Their own memory operations are unknown and may be externally observable.

### Ownership and aliasing

No allocation, deallocation, ownership transfer, or reference-counting operation is established by the visible body.

Passing a stack address to F-U01 does not establish that the dependency may retain it beyond the routine’s lifetime. Conversely, the evidence does not establish that it does not retain it.

Computed addresses, fixed data cells, stack storage, and dependency-managed memory may alias unless external preconditions rule that out. Such aliasing can affect later reads. In particular, no dependency is assumed to preserve globals.

### Failures and termination

There are no visible checks for null addresses, alignment, accessible memory, arithmetic overflow, or callable-target validity.

Invalid addresses or targets may prevent completion according to the execution environment. The routine does not visibly translate those failures into a return code.

If dependencies return normally and neither exit condition occurs, the routine continues indefinitely.

## Dependencies

The following opaque IDs distinguish call sites whose actual function identities are not supplied. They do not assert that the targets are distinct implementations.

| ID | Established interaction |
|---|---|
| F-U01 | Computed target; three represented arguments, including a stack address. |
| F-U02 | Fixed target; supplies the visible 64-bit result used for the early return and halfword store. |
| F-U03 | Fixed target; no represented arguments or ordinary result. |
| F-U04 | Fixed target; supplies a 32-bit address-like result. |
| F-U05 | Fixed target; receives two calculated 32-bit values. An additional register value is subsequently consumed. |
| F-U06 | Fixed target; receives the fixed address-valued argument, a loaded byte, the derived offset, and the additional register value. |
| F-U07 | Fixed target; an additional register output is used to form the working value. |
| F-U08 | Fixed target; receives a calculated address-like value. |
| F-U09 | Fixed target; receives the value read from G-ARG. |
| F-U10 | Fixed target; an additional register output controls later exit calculation or next-iteration state. |

The supplied mapping also identifies F-882dff43a182 at the shared exit location. The visible body branches to that location; it does not establish an ordinary call to F-882dff43a182 or an independent interface for it.

Dependency memory effects, register conventions beyond the represented data flow, failure behavior, and ownership behavior are documented_only.

## Acceptance examples

These examples are conditional bit-level expectations, not runtime observations. They assume required accesses and calls complete normally.

### Initial first-input transformation

If the first explicit input is `0xffffffff`, the first F-U01 invocation receives 127 as its first represented argument.

If the first explicit input is `0x01000000`, that argument is zero.

Neither example constrains later iterations, where the first-input state is derived from an address instead.

### Early exit at a bit boundary

If the current second-input state is `0x00007ff6`, adding ten produces `0x00008000`. Bit 15 is set, so the routine takes the early exit after the offset-62 word store.

If the current second-input state is `0x0000fff6`, adding ten produces `0x00010000`. Bit 15 is clear, so the early condition does not cause a return.

### Early return value

Suppose the early condition holds, F-U02 returns a value with low half `0x80000001`, and G-RETURN is 100 when the common exit reads it.

The returned upper half is 16 and the lower half is 2. The full result is `0x0000001000000002`.

The F-U02-derived halfword store does not occur on this path.

### Halfword truncation

Suppose the early condition is false and F-U02 returns upper half `0x20001000` and lower half `0x12345678`.

The routine stores the 16-bit value `0x5678` at address `0x2000103c`. Byte order within that halfword remains platform-dependent.

### Global snapshot timing

Suppose G-XOR contains one value when read, and F-U05 subsequently changes it.

The derived address uses the earlier captured value. The dependency’s write may still affect other later behavior through aliasing or subsequent reads.

### Later exit

Suppose G-GATE is zero after F-U10, its associated additional register output is `0x00000400`, and G-RETURN is 84.

The exit-specific value is 2, so the returned low half is 4 and the high half is zero. The final calculated-address word store does not occur.

### Continuing iteration

Suppose G-GATE is nonzero.

The final calculated-address word store occurs before the loop repeats. The next iteration uses the F-U10-associated output as its second-input state and stores the preceding iteration’s carried calculation at that new state plus 62.

## Uncertainties

- **documented_only — Calling convention:** The complete entry register state and dependency ABI are not supplied.
- **documented_only — Arithmetic:** Exact signed-overflow and shift semantics require machine-level confirmation.
- **documented_only — Memory layout:** Address validity, alignment requirements, endianness, and the extent of the stack region are unknown.
- **documented_only — Global declarations:** G-GATE’s storage width and the semantic types of the fixed data items are not established.
- **documented_only — Dependencies:** Calls may modify globals or other memory; preservation, allocation, release, and retention behavior are unknown.
- **documented_only — Shared exit identity:** The separate mapped identity at the common exit does not establish a separately callable routine.
- **Inferred intent:** Address-like uses suggest structured state manipulation, but no higher-level object model or purpose is established.

## Evidence

This contract is based exclusively on the supplied static function representation and dependency mapping. It distinguishes visible access order and data flow from unknown dependency behavior and inferred intent.

The conditional examples are derived calculations. No runtime tests, execution traces, memory inspections, or independent machine-level validation have been performed.
