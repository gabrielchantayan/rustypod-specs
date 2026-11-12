<!-- Trusted provenance: F-b57a75cc1feb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b57a75cc1feb

## Purpose

F-b57a75cc1feb describes a non-returning operation that performs computed memory accesses, fixed-location assignments, indirect calls, and a conditional software-breakpoint request before repeatedly processing call results.

No application-level purpose is established. Some assignments involve locations associated with executable content, but intentional instruction modification, device control, or execution-state manipulation remains **inferred intent**, not established behavior.

This contract is based on static evidence. Every later action is conditional on preceding accesses and calls completing. Neither successful invocation nor reachability of the entire sequence is established.

## Interface

### Explicit inputs

The displayed interface has nine input positions and no explicit returned value.

| Position | Representation | Visible use |
|---|---|---|
| 1 | 32 bits | Unused. |
| 2 | Signed 32 bits | Base for initial address calculations. |
| 3 | 32 bits | Unused. |
| 4 | Unsigned 32 bits | Initial call-argument component; conditionally replaced. |
| 5–6 | 32 bits each | Unused. |
| 7 | 32 bits | Locally replaced with zero; not subsequently consumed. |
| 8 | 32 bits | Unused. |
| 9 | Unsigned 32 bits | Locally replaced with a zero-extended byte; not subsequently consumed. |

Additional incoming machine state supplies an independent self-reference address, a 16-bit assignment value, a signed 32-bit accumulator increment, a comparison pointer, an indirect-call destination, and an initial current pointer.

Their initialization and complete calling convention are **documented_only**. The independent self-reference address is not derived from input 2.

### Results, ownership, and preconditions

There is no displayed normal return. Outputs consist of memory effects, calls, and a possible breakpoint request.

Reached accesses require appropriate mappings, permissions, and alignment support. Reached calls require callable destinations. These are execution prerequisites, not conditions checked by this operation.

No allocation, release, ownership transfer, validation, rollback, or recovery protocol is shown. Unknown callees must not be assumed to preserve globals or memory, terminate, or acquire or release ownership.

### Width and arithmetic rules

Explicit access types establish byte, 16-bit, 32-bit, or 64-bit memory operations. A named object’s conversion into a working value does not establish its underlying read width. Similarly, a narrowed assignment expression does not establish the destination object’s storage width.

Working words and represented pointers are 32 bits. Consumed 64-bit call results have upper and lower 32-bit halves; these designate bit positions, not memory endianness.

Explicit unsigned 32-bit arithmetic has modulo-\(2^{32}\) behavior. Signed overflow and signed left shifts must not be treated as unsigned wrapping operations. Conversions from unsigned or pointer representations into signed 32-bit values may also require environment-specific semantics.

Signed right shifts occur with fixed counts, a byte-derived count, and the anomalous count 32. Negative-operand behavior and invalid or out-of-range counts remain **documented_only** where the supplied representation does not resolve them.

## Behavior

### Entry accesses

Two addresses are derived from input 2 by adding 64 and 68 bytes.

The first address is read twice as a 32-bit source. One read supplies a destination address. The other supplies a payload whose low 16 bits are written through that destination. The two reads need not agree; their relative evaluation order is not established.

Next, the independent incoming self-reference address is written as 32 bits at that address plus 20.

An unsigned 16-bit value is then read through the sum of the two derived entry addresses and zero-extended into the working word. Under a wrapping address interpretation, this address equals twice input 2 plus 132. That equivalence does not establish wrapping semantics for the signed calculations.

Only afterward is the first derived address tested for zero. The test therefore protects none of the preceding accesses.

If it is nonzero, the following occur in order:

1. Assign the signed 16-bit truncation of that address multiplied by 4096 at **0x0897f07a**.
2. Replace the first call-argument component with **0x0897f06c**.
3. Assign **0x6000** at **0xbf836004**.
4. Replace the second component with **0xc1b00000**.
5. Obtain the named value at **0xc1b00050**, converting it to the unsigned 32-bit working word.
6. Assign the incoming 16-bit value at **0xbf836002**.

The underlying widths of these named objects are unavailable. In particular, neither the narrowed expression in step 1 nor the conversion in step 5 establishes a memory-access width.

On the zero path, the components remain input 4 and the address derived by adding 68 to input 2.

An address seed is initialized to zero and a call-target base to **0x0897f0c3**. A named value at **0x0897f454** is captured in signed 32-bit working storage before the incoming indirect destination is called.

That call receives one displayed argument: the components’ unsigned XOR, converted to signed 32 bits and subjected to a signed right shift by eight. Its result is unused.

### Initialization after the incoming call

The previously captured value controls the following sequence; the named source is not reloaded:

1. Read an unsigned byte at the captured signed value multiplied by 32.
2. Locally replace input 9 with that byte, zero-extended.
3. Write the captured value multiplied by 16 as 32 bits at the multiplied-by-32 address.
4. Assign zero to the named object at **0x00000009**.
5. Assign a 16-bit zero expression to the named object at **0x00000030**.
6. Explicitly write the captured value’s low 16 bits at **0x0000000e**.

The byte read precedes the potentially overlapping 32-bit write. The named stores at addresses 9 and 48 have unresolved widths. Signed multiplication constraints apply to the computed accesses.

The halfword-address adjustment is initialized to zero, and the current pointer is obtained from incoming register-associated state.

### Outer setup

Each outer iteration retains the location four bytes before the current pointer and writes the working word there as 32 bits.

It then performs these actions in order:

1. At the address seed multiplied by eight, write that same address as 32 bits.
2. Shift the unsigned seed left by 17, convert the result to signed 32 bits, apply a signed right shift by 32, add 16, and write 32-bit zero through the resulting address.
3. Explicitly write halfword **0xc470** at the address formed from the working word logically shifted right by four, minus 16 times the adjustment, minus one if working-word bit 3 is clear, plus 38.
4. Assign **0x8000** at **0xfe238028**.
5. Replace the seed with **0x0897f228**.
6. Assign the self-address value **0x225fc470** at its associated named location.
7. Reset an offset accumulator to zero.
8. Assign **0x225f** at **0x0000229f**.
9. Obtain the named value at **0x00000022**, shift it right by 16, and write the converted result as 32 bits at current pointer plus 84.

Step 2 shifts an unsigned value before converting it to signed. Its right shift by 32 remains unresolved. This order differs from the discriminator calculation described below.

The source type, read width, and right-shift signedness in step 9 are unknown. The named destinations in steps 4, 6, and 8 likewise lack established store widths.

The incoming comparison pointer is compared with current pointer plus 20. Equality skips a call. Inequality calls the previous target base plus **0x2e**, with no displayed arguments, after retaining **0x0897f193** as the new base. The initial optional target is **0x0897f0f1**.

### Inner processing

Setup establishes:

- A zero signed running accumulator.
- Working address **0x0897f1f8**.
- A 64-bit pair with that address in its upper half and zero in its lower half.
- Derived offset **0x0897f22d**.
- Adjustment **0x0897f1f8** for an outer restart.

Each inner pass performs the following:

1. Write the derived offset as 32 bits at the pair’s lower-half address plus 12.
2. Interpret the lower half as signed 32 bits and right-shift it by the lower half’s low-byte value.
3. Record whether subtracting one from that shifted result would overflow signed 32-bit arithmetic.
4. Read 32 bits through twice the shifted result.
5. Replace the working word with **0x0897f1ec**.
6. Assign the read value’s low 16 bits to storage associated with **F-422afe36d401**.
7. Explicitly write **0x0897f1f8** as 32 bits at the read value’s address plus four.
8. Replace the current pointer with the retained preceding location.

The assignment associated with F-422afe36d401 is not a call. Its narrowed payload does not establish its storage width.

A true signed-overflow flag restarts outer setup. This is neither an unsigned-borrow condition nor a general negativity test.

Otherwise:

1. Write byte **0xd8** through the pair’s upper-half address.
2. Replace the working word with **8**.
3. Call the current target base plus **0xe56**, passing zero, after retaining **0x0897f1d7** as the new base.
4. Replace the pair with the returned 64-bit result.
5. Add the incoming signed increment to the signed accumulator.
6. Obtain the named value at **0x0897f216**, convert it to unsigned 32 bits, and use it as the current pointer.
7. Read a byte at the returned lower half plus **0x0897f1f8**, retaining it as the byte source.

Step 6 establishes neither a 16-bit read nor zero-extension from a 16-bit object.

A zero condition byte is then compared with the earlier overflow flag. On the represented non-overflow path, they compare equal. An unsigned 16-bit destination address is read at returned lower half plus the derived offset; the signed accumulator is written as 32 bits through that zero-extended address. The repeated cycle begins with argument zero.

Initially, the pair’s lower half and its shift count are zero. Subtracting one from the resulting zero does not overflow.

### Repeated cycle and larger branch

Each cycle saves the byte source’s low byte, establishes continuation base **0x0897edf1**, and calls target **0x07f755f6** with the cycle argument.

The returned upper half becomes a field pointer. The lower half is converted to signed 32 bits and multiplied by 32; the product is assigned to an unsigned 32-bit address. A 32-bit self-address write occurs there before that address is compared, unsigned, with 20. The signed multiplication must not be replaced by unsigned wrapping multiplication.

If the address is below 20, the continuation argument becomes zero and the larger branch is skipped.

Otherwise:

1. Write the working address as 32 bits at the multiplied address plus 88.
2. Write the current pointer’s low 16 bits at working address plus 14.
3. Replace the working word with the unsigned working address shifted left by two.
4. Combine the reset-associated source’s byte contribution as the upper byte with the saved byte as the lower byte.
5. Assign that combined 16-bit expression to reset-associated storage.
6. Call target **0x0897ee9f**, passing the current pointer logically shifted right by two.
7. Select continuation base **0x0897ee1d**.
8. Call target **0x0897ee1a + 0xffe**, with no displayed arguments, obtaining a second 64-bit result.

The reset-associated source and destination have no supplied declarations or numeric addresses. Their access widths are not established by the byte composition.

Using the second result, write in order:

- The saved byte at twice its lower-half address.
- The upper half’s low byte at lower-half address plus eight.
- The earlier field pointer as 32 bits at twice the signed accumulator.
- The full 64-bit result through its lower-half address.

Next, convert the earlier field pointer to signed 32 bits and multiply by four to replace the accumulator. Convert the current pointer to signed 32 bits and left-shift by eight to form the new working address. These are not established unsigned wrapping operations.

Through the earlier field pointer, write three 32-bit fields in order: the working word at offset zero, the updated working address at offset four, and the current pointer at offset eight.

Multiply the unsigned working word by 256 using unsigned arithmetic, then convert that result into a signed 32-bit working quantity.

The discriminator calculation has this exact arithmetic order:

1. Convert the current pointer to signed 32 bits.
2. Left-shift that signed value by 31.
3. Apply a signed right shift by 32.
4. Convert the result to unsigned 32 bits.
5. Shift logically right by 20.

Two independent uncertainties apply: the signed left shift has signed-operand validity and representability constraints, and the subsequent right shift has count 32. The first shift is **not** an unsigned wrapping shift followed by signed interpretation. Neither uncertainty may be resolved by assuming a masked shift count or a particular machine instruction.

If a discriminator value is obtained and is zero:

- Assign the signed working quantity’s low 16 bits to reset-associated storage.
- Add the unsigned working word to the accumulator, with the usual unsigned participation and conversion back to signed working storage.
- Write that accumulator as 32 bits through the second result’s upper-half pointer.
- Select continuation argument zero.

If a nonzero discriminator is obtained:

- Write it as 32 bits at signed working quantity plus 32.
- Select continuation base **0x08790d52**.
- Select the signed working quantity right-shifted by 29 as the argument.
- Request software breakpoint **4**.
- If execution resumes, replace the working address with the second result’s upper-half pointer converted to signed 32 bits and right-shifted by 18.

Neither discriminator outcome can be eliminated from the supplied evidence.

### Cycle continuation

The selected continuation base plus **0xa1e** is called:

| Path | Target |
|---|---|
| Below-20 path | **0x0897f80f** |
| Larger branch, zero discriminator | **0x0897f83b** |
| Resumed breakpoint path | **0x08791770** |

After it returns:

1. Write its lower half’s low 16 bits through that lower-half address.
2. At working address multiplied by 64, write that same address as 32 bits.
3. Read and retain a 32-bit word through the current pointer.
4. At working address plus four, write the current pointer’s low 16 bits plus four, truncated to 16 bits.
5. Replace the current pointer with a 32-bit pointer read at returned upper-half address plus 80.
6. Call **0x0897f191**, then **0x0897fddd**, with no displayed arguments.
7. Replace the byte source with the existing working word.
8. Call **0x0897f5e1**, passing the retained word.
9. Select the retained word logically shifted right by three as the next cycle argument.
10. Replace the working word with the retained word.

Control returns to cycle entry. The next saved byte comes from the preceding working word, not necessarily the newly retained word.

## Side effects

Explicit memory operations range from 8 to 64 bits. Named-object accesses have unresolved widths where declarations are absent.

Computed destinations may alias fixed locations, earlier sources, reset-associated storage, or executable-associated locations. Symbolic naming does not establish disjoint objects; the supplied warning explicitly identifies overlapping global representations.

No atomicity, barriers, cache synchronization, or protection mechanism is shown.

Calls intervene between accesses and may affect subsequently read storage. The displayed captures are not implicit reloads. A fault, nonreturning call, or breakpoint without resumption prevents subsequent actions without undoing earlier writes.

## Dependencies

- **F-422afe36d401:** associated storage at **0x0897f1e8** receives a narrowed 16-bit expression. No invocation of this dependency is shown.
- **Unmapped call destinations:** the incoming destination and numeric call sites above lack supplied opaque function identities. Their implementations, preservation, ownership effects, and termination remain **documented_only**.
- **Breakpoint facility:** receives request 4; handling and resumption are unknown.
- **Memory environment:** supplies reached storage and undeclared named objects.

No additional function identities or ownership contracts are inferred.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Independent entry reads:** destination-source value **0x12345678** and payload-source value **0xabcdef01** cause halfword **0xef01** to be written at **0x12345678**.

2. **Independent self-reference:** incoming self-reference **0x20001000** causes a 32-bit write of that value at **0x20001014**, independently of input 2.

3. **Late zero test:** if the first derived address is zero, the initial reads through zero occur before the branch that skips fixed-location initialization.

4. **Named-store width:** address truncation **0x1234** produces assignment expression **0x01234000** at **0x0897f07a**. Only if the destination is 16 bits does it retain **0x4000**; that width is not established.

5. **Captured source:** captured signed value three causes a byte read at **0x60**, followed by a 32-bit write of **0x30** there. An intervening callee change to the original source does not cause a reload.

6. **Reset composition:** source-byte contribution **0xab** and saved byte **0x34** produce expression **0xab34**, without establishing the reset destination’s width.

7. **Threshold ordering:** returned lower half zero causes a self-address write at zero before the below-20 branch. Lower half one produces address 32 and selects the larger branch.

8. **Discriminator constraints:** for signed pointer value one, the discriminator’s first operation after conversion is signed left shift by 31; no unsigned-wrap result is prescribed. For pointer value zero, the left shift can produce zero, but the following signed right shift by 32 still prevents prescribing a discriminator from the supplied evidence.

9. **Continuation truncation:** current-pointer low half **0xfffe** produces halfword **0x0002** at working address plus four.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid invocation domain, and complete calling convention.
- Named-object types, widths, signedness, and reset-associated numeric addresses.
- Signed overflow, pointer-to-signed conversions, signed-left-shift validity, negative signed right shifts, and out-of-range shift counts.
- In particular, both the discriminator’s signed left shift by 31 and its subsequent signed right shift by 32.
- Endianness, alignment requirements, mappings, permissions, device behavior, and atomicity.
- Callee effects, ownership, preservation, and termination.
- Intentional instruction modification, breakpoint handling, and successful reachability of later operations.

## Evidence

The supplied static representation and opaque dependency mapping are the behavioral evidence. Explicit access types establish explicit widths; undeclared named-object widths cannot be recovered solely from assignment expressions or working-value conversions.

The discriminator expression establishes signed conversion **before** left shift by 31, followed by signed right shift by 32, unsigned conversion, and logical right shift by 20. The separate outer-setup expression instead performs its left shift while unsigned before converting to signed.

The representation supports the stated sequencing, conditional transfers, computed call targets, and absence of a displayed normal return. Prior prose and feedback are not independent evidence.

No runtime tests, executable inspection, or dependency inspection have been performed.
