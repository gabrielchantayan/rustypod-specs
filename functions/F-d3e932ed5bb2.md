<!-- Trusted provenance: F-d3e932ed5bb2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d3e932ed5bb2

## Purpose

This operation performs indirect calls, writes through supplied and computed addresses, traverses nested loops, and issues execution-control events. Its application-level purpose is not established.

The available static representation reports invalid instruction data and truncated control flow. This contract therefore describes the represented behavior conditionally: reached accesses, calls, and events have the relationships below, but executable validity, reachability, termination, and ordinary return are not established.

No runtime tests have been performed.

## Interface

### Explicit arguments

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits, signed | Used as an address and narrowed to supply the first stored halfword. |
| Second | 32 bits | No explicit use in the supplied body. |
| Third | 32 bits | No explicit use in the supplied body. |
| Fourth | 32 bits, signed | Supplies an operand of a subtraction passed to an early indirect dependency. |

Unused explicit arguments are not proven irrelevant to the actual calling convention. Indirect signatures and ambient register inputs are incomplete.

### Ambient inputs

Execution additionally depends on:

- An incoming return-address-like value that selects the first indirect target.
- A register-derived base for an early byte store.
- An auxiliary result register used after a later indirect call.
- A register-derived word-buffer pointer.
- Stack locations and their addresses.
- A global value supplying a stored byte.
- Memory reached through computed addresses and dependency results.

These inputs have no established validation or defaulting protocol.

### Outputs, widths, and preconditions

No ordinary return value is established. Observable results are memory writes, dependency calls, execution-control events, and possible faults or nontermination. The terminal represented path reaches invalid-instruction termination.

The representation uses 8-bit bytes, 16-bit halfwords, 32-bit words and address calculations, and 64-bit paired results. “Low” and “high” components mean the low and high 32 bits of a paired result; this does not establish memory byte order.

Narrow stores retain the low destination-width bits. Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). Signed arithmetic, signed left shifts, and pointer calculations are not automatically assigned that rule: overflow and invalid-address behavior remain unresolved where the representation does not establish them. Signed right shifts are treated as sign-propagating where expressly represented.

Any reached access requires suitable mapped memory and any reached call requires an executable target. Alignment, access permissions, trap handling, and the actual calling convention are environmental preconditions, not checks performed here.

## Behavior

### Entry and preparation

The first visible operation stores the low 16 bits of the first argument at the address four bytes beyond that argument. There is no preceding guard.

An indirect call then uses the incoming return-address-like value plus `0x500` as its target. It has no represented explicit arguments and produces a paired result.

The low component is doubled to form an index. The high component is shifted left by two to begin an accumulator. A byte obtained from global data is written at the ambient base plus that index.

Two further indirect calls follow; the second produces another paired result. Its low component plus 69 forms an unsigned intermediate, which is added to the accumulator. The next indirect call receives:

1. That intermediate.
2. The new high component bitwise-ANDed with the doubled earlier index minus 125.
3. The intermediate shifted logically right by 21.
4. The fourth original argument minus the updated accumulator.

The subtraction is inside the mask operand. This is not an established validation operation.

The updated accumulator, interpreted as signed and shifted right by 13, supplies the **derived base** used in the first loop phase. Four additional indirect calls occur before that phase. No intervening error checks are visible.

### First outer and inner loops

The outer offset initially equals 409.

The outer opening indirect target is loop-carried:

- Initially, its target base is `0x08572111`.
- On later outer iterations, its target base is `0x0857215D`.
- The target is the applicable base plus `0xF46`.

That call produces an integer passed to the first inner opening call.

The inner opening target also changes:

- For the first inner iteration of each outer iteration, its base is `0x08572115`.
- For subsequent inner iterations, its base is `0x08572125`.
- The target is that base plus `0xF30`.

On subsequent inner iterations, this opening call receives the derived base rather than the outer opening call’s result.

Each inner iteration then makes another indirect call, a paired-result-producing indirect call, and an indirect call receiving the paired result’s low component and its top seven bits. The last call also has a represented result retained in stack-associated state.

After these calls:

- The derived base minus the auxiliary result-register value is converted to unsigned 32 bits and shifted logically right by 16. This is retained as the **shifted subtraction**.
- A plain byte is read at the derived base plus the current outer offset.

The exact inner continuation predicate requires both:

- The derived base is nonzero.
- The test “derived base is negative” equals the signed-subtraction-overflow flag for subtracting 66 from the current outer offset.

It is not a conventional less-than comparison between the derived base and the offset. Neither operand is explicitly updated within the inner loop. For the two displayed offset values, subtracting 66 does not overflow signed 32-bit arithmetic; the predicate then requires a positive derived base. No finite iteration bound is established.

After the inner loop exits, another indirect call occurs. The derived base is multiplied by the byte just read. Plain-byte signedness and signed multiplication overflow remain unresolved. The operation then:

1. Stores the product’s low 16 bits at derived base plus 34.
2. Stores the derived base’s low byte at derived base plus one.
3. Makes two indirect calls, the second producing an integer.
4. Retains a selected stack-word location.

The outer offset becomes exactly `0x85723A`, replacing 409. This changes both subsequent byte-read addresses and the offset used by the inner predicate. The outer loop repeats when the product is zero; the other represented continuation flag is set true.

### Breakpoint and identified dependencies

After the first outer loop exits, halfword `0x723A` is stored at the shifted subtraction plus 52. The most recent indirect integer result is shifted arithmetically right by 13. A breakpoint event with immediate `0xAC` follows.

If execution continues, **F-c5ddd881f572** receives:

- A word read at the selected stack-word address plus the derived base.
- A byte read at derived base plus five.

Both reads precede the call. Their relative order is not established by the supplied call expression.

Using this dependency’s paired result, the operation then:

1. Stores the low byte of a stack address at the shifted indirect result plus the returned high component.
2. Stores the low 16 bits of the early accumulator shifted arithmetically right by 17 at returned low component plus four.
3. Writes the derived base to the selected stack word.
4. Reads the byte at derived base plus five again.
5. Calls **F-f28d5d0f667f** with the preceding returned low component and this freshly read byte.

The second byte read is distinct from the first. The intervening call and stores can affect it through aliasing or dependency side effects.

A software interrupt with immediate `0x53` follows. All later behavior requires continuation past that event.

### Second outer-loop entry

The next phase begins with an unsigned working value of zero and an ambient word-buffer pointer. Each outer iteration retains its current buffer pointer, then shifts the working value logically right by 12 and reads an unsigned byte at that result plus 20. Consequently, the first iteration attempts a byte read at numeric address 20.

The outer opening indirect target is also loop-carried:

- The initial target base is `0x085721AD`.
- Later iterations use the selected fixed data address `0x0894A20A` as their target base.
- Each call target is the applicable base plus `0xE54`.

Its four represented arguments are:

1. The current paired result’s low component.
2. Initially, the integer shifted right by 13 before the breakpoint; on later outer iterations, the preceding iteration’s retained control value minus 33.
3. The newly read unsigned byte.
4. A conversion-sensitive value obtained by interpreting the current pair as signed 64 bits, shifting arithmetically right by 52, converting to unsigned 32 bits, then shifting logically right by eight.

The fourth argument is not necessarily zero. The conversion between the shifts is part of the behavior.

A further indirect call produces a signed integer. That integer and its arithmetic-right-shift-by-22 result, converted to unsigned 32 bits, are the first two arguments supplied to the next inner opening call.

### Second inner-loop calculations

For this section:

- \(W\) is the current unsigned 32-bit working value.
- \(C\) is the incoming unsigned 32-bit control value.
- \(L\) and \(H\) are the latest paired result’s low and high components.
- \(T\) is an unsigned 32-bit threshold.

Initially, \(C\) comes from the unsigned byte read at outer entry. Later inner iterations carry it at 32-bit width; they do not narrow it back to a byte.

The inner opening target uses base `0x085721C9` on the first inner iteration and `0x085721ED` thereafter, adding `0xEB0` in either case. Its initial arguments are described above. On subsequent iterations, they are the retained subtraction described below and \(H\).

Each inner iteration makes nine indirect calls in sequence, including that opening call. The third produces an integer whose value minus three supplies the default threshold. The eighth produces an unsigned integer; its logical-right-shift-by-13 result is passed to the ninth, which produces the pair containing \(L\) and \(H\).

The meaningful arithmetic is:

- Form \(A=W+6\), modulo \(2^{32}\).
- Default retained buffer values are \(A\) and \(A\) logically shifted right by 12.
- If incoming \(C\) is nonzero:
  - Set \(T\) to the result of logically shifting \(A\) right by eight, interpreting that result as signed 32 bits, shifting arithmetically right by 12, and subtracting 85. The threshold is retained as unsigned 32 bits.
  - Replace the first retained buffer value with \(2L\), modulo \(2^{32}\).
  - Replace the second retained buffer value with the incoming \(W\).

The branch tests \(C\) before decrementing it.

Let \(S\) be one when bit 31 of \(L\) is set, otherwise zero. Each iteration then:

1. Replaces the working value with \(2L\), modulo \(2^{32}\).
2. Stores \(L\) as a word at current buffer plus 652.
3. Retains \(L-T-(1-S)\), modulo \(2^{32}\), for the next inner opening call.
4. Replaces the control value with \(C-83\), modulo \(2^{32}\).

Here \(1-S\) is the Boolean complement of the extracted sign-bit condition, not a 32-bit bitwise complement.

The loop continues if either \(T<L\) under unsigned comparison, or the unsigned difference \(L-T\) is less than \(S\). At equality, continuation therefore depends on bit 31 of \(L\).

### Buffer writes and outer continuation

After the inner loop exits, the fixed data address `0x0894A20A` becomes the next outer call-target base. A word is read at current buffer plus 716, followed by a halt event with immediate `0x26`.

If execution continues, the operation uses the final inner iteration’s incoming \(C\), not its decremented replacement, to form \(C-33\). It stores the low 16 bits of the computed pointer \(T+112\) at address \((C-33)+2L\).

Five words are then written relative to the current buffer, in this order:

| Offset | Stored value |
|---:|---|
| −4 | Fixed data address `0x0894A20A` |
| −8 | First branch-dependent retained value |
| −12 | Word previously read at offset 716 |
| −16 | \(H\) |
| −20 | Second branch-dependent retained value |

Next, the representation performs these separate reads in order:

1. A word at \(T+112\).
2. A signed integer at \(T+116\).
3. A 64-bit paired value beginning at \(T+112\).
4. A word at \(T+124\).

The overlapping reads must not be conflated into one stable snapshot without additional evidence.

The next buffer pointer is current buffer minus 20. The word read at \(T+124\) replaces the working value before the next outer iteration’s logical shift by 12. The overlapping 64-bit read supplies the next outer iteration’s paired result.

The outer loop repeats if the computed pointer \(T+112\) is null, or if the signed interpretation of \(T+62\) is less than −50. All four reads precede this test. It tests the computed pointer, not the word stored there, and is not a protective null guard.

### Terminal sequence

On outer-loop exit, the operation:

1. Reads a word at \(T+128\).
2. Stores its low byte 29 bytes beyond the word previously read at \(T+124\).
3. Stores the separately read word from \(T+112\) at the address formed from the retained \(C-33\) value plus the integer read at \(T+116\).
4. Calls **F-a041a7d81f49** with a word read at current buffer plus 588 and the integer from \(T+116\) increased by 80.
5. Reaches invalid-instruction termination.

The buffer-relative terminal argument uses the current iteration’s buffer, not the decremented next buffer. The relative evaluation order of the two final call arguments is not independently established.

## Side effects

Writes can affect caller-addressed memory, ambient-base-relative memory, stack memory, dependency-result-relative memory, threshold-relative memory, and a descending word buffer.

No rollback, atomicity, synchronization, or reentrancy guarantee is visible. Earlier writes remain observable even if a later access, dependency, or execution-control event prevents further progress.

Ownership is **documented_only**. No explicit allocation or deallocation is visible. Using returned components as addresses does not establish ownership transfer. Dependencies are not assumed to preserve globals or pointed-to storage, retain or release arguments, or preserve undocumented register state.

Aliasing is not excluded. Distinct represented accesses can reach the same memory, and calls may alter later-read data.

## Dependencies

| Opaque function ID | Established relationship | Unresolved contract |
|---|---|---|
| **F-c5ddd881f572** | Receives a stack-relative word and a base-relative byte; produces a represented 64-bit result used in address calculations. | Argument meanings, failure signaling, ownership, memory effects, preservation rules. |
| **F-f28d5d0f667f** | Receives the preceding low component and a freshly read byte; supplies the initial second-phase paired result. | Result validity, ownership, side effects, failure behavior. |
| **F-a041a7d81f49** | Receives a buffer-relative word and a computed integer near termination. | Return behavior, argument interpretation, side effects. |

Other dependencies are indirect and have no supplied opaque IDs. Their target-selection rules are described above without inventing identities. Recovered argument lists do not establish complete machine interfaces.

Breakpoint, interrupt, halt, and invalid-instruction handling depend on the execution environment. Continuation past any such boundary is conditional.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial narrowing.** For first argument `0x12345678`, assuming the destination is writable, the first store writes halfword `0x5678` at `0x1234567C` before any dependency call. Byte order is unspecified.

2. **Conversion between shifts.** A current paired value of `0x1000000000000000` gives fourth outer-call argument 1: the signed shift by 52 produces `0x100`, conversion retains it, and the logical shift by eight produces 1. A signed value of −1 produces `0x00FFFFFF`.

3. **Loop-carried targets.** If execution reaches later iterations, acceptance requires the changed opening targets in both outer loops and both inner loops. Reusing each initial target indefinitely does not match the representation.

4. **First-loop predicate.** At offset 409, subtraction of 66 has no signed overflow. A positive derived base satisfies the inner continuation predicate; a negative or zero derived base does not. Reaching another outer iteration changes the offset to `0x85723A`.

5. **Non-byte control.** If incoming control is 1, its next value is `0xFFFFFFAE`, not byte value `0xAE`. Subsequent nonzero tests use the 32-bit value.

6. **Threshold branch.** If \(W=0\) and incoming control is nonzero, \(A=6\), the replacement threshold is unsigned `0xFFFFFFAB`, and the retained buffer values become \(2L\) and zero.

7. **Equality boundary.** If \(L=T\), the inner loop continues exactly when bit 31 of \(L\) is set.

8. **Unordered and fresh reads.** Either argument-read order is consistent with the call to **F-c5ddd881f572**. A later change to the base-relative byte must be reflected in the separate read passed to **F-f28d5d0f667f**.

9. **Late null test.** Even when computed pointer \(T+112\) is null, the represented reads occur before its null comparison. A safe null-error return is not required.

10. **Execution boundary.** If interrupt `0x53` does not resume, later loop effects need not occur. If all represented terminal operations are reached, an ordinary successful return is still not established.

## Uncertainties

The following remain **documented_only**:

- Intended application purpose and executable validity of the represented instruction stream.
- Complete calling convention, indirect signatures, and ambient register requirements.
- Signed-overflow behavior, plain-byte signedness, alignment, and byte order.
- Dependency ownership, mutations, preservation guarantees, and failure protocols.
- Trap handling, continuation, and control flow beyond truncation.
- Loop termination, iteration bounds, and required memory extent.
- Whether represented repeated or overlapping reads correspond to distinct physical loads.
- Reachability of particular paths under a real execution environment.

“Buffer,” “threshold,” and “derived base” describe computational roles, not established object types or ownership models.

## Evidence

**Established static evidence:** the supplied representation contains the accesses, narrowing operations, conversions, call relationships, loop-carried target assignments, predicates, and execution-control events described here. It explicitly reports invalid instruction data and truncated control flow.

**Inferred intent:** no application-level intent is asserted. Address-like results and descending writes suggest memory-oriented processing, but do not establish a valid data structure or useful outcome.

**Verification limit:** this contract uses only the supplied static material. No executable, dependency implementation, instruction-level trace, or runtime observation was available. No runtime tests have been performed.
