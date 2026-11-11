<!-- Trusted provenance: F-55b8fcefbe4c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-55b8fcefbe4c

## Purpose

The supplied static representation establishes a routine that performs ordered, mixed-width memory accesses, invokes external execution targets, and enters processing loops with no represented reachable normal return.

An application-level purpose is not established. Low addresses, register-derived addresses, and exception-associated storage appear in the representation, but those observations do not establish a particular device, protocol, or initialization service.

This contract distinguishes:

- **Established static behavior:** accesses, calculations, and control-flow relationships directly represented.
- **Conditional behavior:** effects reached only if preceding accesses and calls complete.
- **documented_only:** behavior whose execution or semantics remain unresolved.

The sole explicit return belongs to a branch that is unreachable under the coherent 32-bit arithmetic model described below.

## Interface

### Explicit arguments

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use |
| Second | 32 bits | No explicit use |
| Third | Signed 32 bits | Multiplied by 256 to obtain the initial working offset |
| Fourth | 32 bits | No explicit use |
| Fifth | Signed 32 bits | Included in an address-derived sum before a doubled-address word store |

No return value is represented. Unused explicit arguments are not demonstrated dependencies; unknown callees inspecting residual machine state cannot be ruled out.

### Additional entry state

The representation also consumes:

- Incoming r4: a 32-bit control and address contribution.
- Incoming r5: a 32-bit pointer bit pattern.
- Incoming r6: a 16-bit value.
- Incoming r7: a 32-bit value.
- Incoming r9: an address contribution used only in the unreachable terminal branch.
- Incoming link-register contents: used to construct the optional initial call target.
- Fixed-location storage, including two separately evaluated reads of the first base global.

These are environmental dependencies, not additional established formal arguments.

### Widths and arithmetic

For arithmetic deductions, unsigned calculations and address bit patterns are treated as 32-bit values reduced modulo \(2^{32}\). Explicit byte, halfword, word, and doubleword pointer accesses have widths of 8, 16, 32, and 64 bits respectively. Pointer stores are represented as 32-bit stores.

Signed overflow and signed shifting are not thereby made well-defined source-language operations. The final shift by 32 is specifically unresolved.

Fixed-global declarations are absent. Consequently, a constant’s size, a value’s incoming width, or conversion to a 32-bit working value does **not** establish a fixed-global access width. Such widths remain unknown unless an explicit pointer access establishes them.

Endianness, alignment restrictions, mapped address ranges, access atomicity, device semantics, and concurrent modification rules are unspecified. There are no represented validity checks for derived pointers.

## Behavior

### 1. Entry and optional initialization

The routine first reads the first base global and saves its value in a 32-bit working quantity. It then forms:

- The initial working offset: third argument multiplied by 256.
- A shifted address contribution: incoming r4 shifted left by five.
- An initial destination: `0x000000A1`.

If the low 24 bits of incoming r4 are zero, initialization is skipped. The incoming r5 and r7 values, initial offset, shifted contribution, and initial destination remain in use.

Otherwise, these observable operations occur in order:

1. Assign `0x18` to interrupt-associated fixed storage.
2. Assign the incoming 16-bit r6 value to fixed storage at `0x225FB880`.
3. **Reread the first base global**, add the initial offset, and store byte `0xCE` at that address.
4. Read fixed storage at `0x0CE00026`, converting its value into the 32-bit r7-derived working value.
5. Assign zero to fixed storage at `0x0CE00000`.
6. Invoke F-7c0a18e2.

The fixed-global access widths in steps 1, 2, 4, and 5 are not established. The destination in step 3 uses the second base-global read, not the saved entry value. The preceding writes may alias that global; their effects must not be ignored when selecting the byte-store address.

F-7c0a18e2’s target is the incoming link-register value with its low two bits cleared, plus `0x8A4`. Its three represented arguments are:

- Bits 10–23 of incoming r4, right-justified.
- `0xC0`.
- Address `0x0CE00000`.

If it returns, the routine:

1. Sets the working pointer to twice the **saved entry** base value.
2. Sets the working offset to `0x0897ECA0`.
3. Reads the second fixed base global and multiplies its converted value by 32.
4. Stores halfword `0xECA0` at four times an additional register result from F-7c0a18e2.
5. Sets the initial destination to zero.
6. Stores halfword `0xECA0` at the scaled second-global value plus six.
7. Sets the r4-derived working quantity to 16.
8. Clears the shifted address contribution.

The additional register result is not established as an ordinary function return value.

### 2. Common writes

The routine next performs:

1. A halfword store at the shifted contribution plus `0x30`, containing that contribution’s low 16 bits.
2. A word store at the r4-derived working quantity multiplied by \(2^{25}\), plus `0x40`, containing that scaled quantity.
3. A zero-byte store at the initial destination.
4. A word store of the r7-derived working value at the same destination.

On the skipped-initialization path, the halfword value is necessarily zero, but its address is **not** necessarily `0x30`. For example, incoming r4 equal to `0x01000000` produces shifted contribution `0x20000000` and a zero-halfword store at `0x20000030`. The word store in step 2 is nevertheless zero at `0x40`.

After initialization, steps 1 and 2 instead write zero at `0x30` and word `0x20000000` at `0x20000040`.

The byte and word stores at the initial destination are distinct ordered accesses, even though they overlap.

### 3. Unreachable terminal branch

A subsequent branch requires both:

- Bit 31 set in the working r4-derived quantity multiplied by \(2^{25}\).
- A nonzero result when that quantity is multiplied by \(2^{26}\).

Neither established entry path satisfies this condition. On the skipped path, the low 24 bits are zero, making the first scaled value zero. On the initialized path, the quantity is 16 and the first scaled value is `0x20000000`, whose top bit is clear.

The branch is therefore **documented_only**. Its represented effects, if entered outside these established conditions, are:

1. Store a zero byte at the \(2^{26}\)-scaled address.
2. Add the r7-derived value to that address to obtain a halfword-pointer address.
3. Store the low 16 bits of the initial destination at the working pointer plus two.
4. Store the new halfword-pointer address as a word at itself plus `0x40`.
5. Store its low 16 bits at itself.
6. Form an address from the r4-derived quantity multiplied by \(2^{30}\), plus `0x11`, and store that address as a word through itself.
7. Intersect the working offset with the halfword-pointer address. Shift that intersection left once, interpret it as signed 32 bits, and arithmetically shift right by ten to obtain another address.
8. Perform two consecutive identical halfword stores at that address, each containing its low 16 bits.
9. Store halfword `0x0050` at the unshifted intersection plus two.
10. Assign `0xF430` to fixed storage at `0xFD`, assign `0x0897F430` to exception-associated storage, and clear another exception-associated object.
11. Read a further fixed-global value as an address and store word `0x0897F430` through it.
12. Transfer to F-e4319b70 at incoming r9 plus `0x0897F314`.

The fixed-global widths in steps 10 and 11 are unknown. F-e4319b70 receives four represented arguments: the address read in step 11 plus 16, multiplied by \(2^{24}\); zero; `0x5000`; and `0x56`.

An explicit return follows the transfer, but whether that indirect transfer is genuinely a returning call is unresolved.

### 4. Reachable loop preparation

On the ordinary path, the routine:

1. Stores word `0x40` at the initial destination plus `0x28`.
2. Stores the initial destination as a word at itself plus `0x2C`.
3. Initializes an accumulator to the bitwise intersection of the r7-derived value and working offset.
4. Initializes a secondary quantity to eight times the working offset minus two.
5. Adds the working pointer, fifth argument, and eight; stores that sum as a word at twice the sum.
6. Forms a primary destination from the working offset multiplied by \(2^{24}\).
7. Stores the working offset at the initial destination plus four.
8. Stores the secondary quantity’s low byte at the primary destination plus `0x14`.
9. Multiplies the working offset by \(2^{25}\), clears a separate scaled working value, and makes a second destination equal to the primary destination.
10. Selects initial loop-call target base `0x08BFED9A`.

On the skipped path, both offset multiplications in steps 6 and 9 produce zero. After initialization, they produce `0xA0000000` and `0x40000000`. The byte in step 8 is `0xFE` on either path.

### 5. Loop-entry write cluster

The cluster performs, in order:

1. Store a zero byte through the primary destination.
2. Store the secondary quantity’s low halfword through that destination.
3. Store the scaled working value as a word through that destination.
4. Add the working offset to the accumulator.
5. Store the updated accumulator through the second destination.

The destinations coincide initially, but need not coincide later.

After this cluster, the first processing-call argument is set to zero. Short-path continuation also resets that argument to zero, but does **not** replay this cluster.

### 6. Processing pass and call-selection state

Each pass first captures the working offset’s low byte. It then:

1. Calls F-29d6a04b at the selected target base plus `0xA1E`, using the current argument state.
2. Splits its 64-bit result into lower and upper 32-bit components.
3. Stores the lower component’s low halfword at the address given by that lower component.
4. Stores a word at the secondary quantity multiplied by 64, containing that same scaled quantity.
5. Reads a word through the current working pointer; this becomes the working offset.
6. Stores the low 16 bits of the old working-pointer address plus four at the secondary quantity plus four.
7. Reads a replacement 32-bit working pointer from the upper result component plus `0x50`.
8. Calls F-b854c31d, then F-903ae725, each without explicit arguments.
9. Calls F-6a1f92c8 with the word read in step 5.
10. Calls F-d20e7849 with that word logically shifted right by three.
11. Uses F-d20e7849’s upper result component as a destination pointer and multiplies its lower component by 32.
12. Stores that scaled lower component as a word at its own address.
13. Compares it with 20 using an unsigned comparison.

The first loop call’s selection and argument transitions are:

| Entry or continuation | Target base before adding `0xA1E` | Argument |
|---|---:|---|
| Initial processing pass | `0x08BFED9A` | Zero |
| Short-path continuation | `0x0897EDF1` | Zero |
| Extended path returning through the write cluster | `0x0897EE1D` | Zero |
| Resumed-breakpoint continuation | `0x08790D52` | Signed arithmetic right shift of the current scaled working value by 29 |

The intermediate constant `0x0897EDDF` participates in another call’s target construction. It is not the next F-29d6a04b argument on any represented loop back edge.

If the scaled result is below 20, execution takes the short-path continuation immediately. A 32-bit multiple of 32 is below 20 only when it is zero. This does not require the unscaled lower result component to have been zero.

### 7. Extended processing and breakpoint

Otherwise, the routine:

1. Stores the secondary quantity as a word at the scaled result plus `0x58`.
2. Stores the replacement working pointer’s low halfword at the secondary quantity plus `0x0E`.
3. Sets the working offset to four times the secondary quantity.
4. Assigns an exception-associated global a concatenation of its separately represented upper-three-byte component and the byte captured at pass entry.
5. Calls F-f8130d6a with the working pointer logically shifted right by two.
6. Calls F-4b9e207c without explicit arguments.

The upper-three-byte component in step 4 is read **at that update**, after the preceding calls. It is not preserved from an earlier observation. The global’s physical read/write widths remain unknown.

Let the latest F-4b9e207c result have lower and upper 32-bit components. The following effects are ordered:

1. Store the captured byte at twice the lower-component address.
2. Store the upper component’s low byte at the lower-component address plus eight.
3. Store F-d20e7849’s earlier upper component as a word at twice the current accumulator.
4. Store the complete 64-bit F-4b9e207c result at its lower-component address.
5. Set the accumulator to four times F-d20e7849’s upper component.
6. Set the secondary quantity to the working pointer shifted left by eight.
7. Write three consecutive words through F-d20e7849’s upper-component pointer: the current working offset, new secondary quantity, and working pointer.
8. Set the scaled working value to the working offset multiplied by 256.
9. Evaluate a derived value using a signed left shift by 31, a signed right shift by 32, conversion to unsigned, and a logical right shift by 20.
10. Set the primary destination to zero. The second destination remains F-4b9e207c’s upper component.

Step 9 is **documented_only**: the oversized shift has no established result.

Conditionally, if that expression yields zero, execution returns to the loop-entry write cluster. The target base is `0x0897EE1D`; the ensuing call argument is reset to zero.

If it yields nonzero:

1. Store the derived value as a word at the scaled working value plus `0x20`.
2. Change the first loop-call target base to `0x08790D52`.
3. Set its argument state to the signed arithmetic right shift of the scaled working value by 29.
4. Execute software breakpoint number four.
5. If execution resumes, set the secondary quantity to the signed arithmetic right shift of the latest F-4b9e207c upper component by 18.
6. Continue directly in the inner processing loop.

Breakpoint continuation does not replay the write cluster, add to the accumulator there, or reset the call argument to zero.

## Side effects

The routine can write fixed storage, low addresses, and addresses derived from incoming state or dependency results. No bounded output buffer is established.

Overlapping accesses remain distinct, with their represented widths and order. Aliasing may cause earlier writes to affect later reads, including the base-global reread and exception-storage update.

No allocation, release, ownership transfer, or reference-count operation is established. Dependency-produced pointers do not imply ownership. Unknown callees may modify globals or other memory; preservation is not assumed.

No error result, validation, rollback, cleanup, or fault recovery is represented. Invalid accesses, nonreturning dependencies, or an unresumed breakpoint can prevent subsequent effects.

## Dependencies

These opaque IDs identify documentary execution roles, not verified external implementations.

| ID | Established role |
|---|---|
| F-7c0a18e2 | Optional indirect call with three arguments and an additional consumed register result |
| F-e4319b70 | Uncertain indirect transfer in the unreachable terminal branch |
| F-29d6a04b | State-selected loop call returning consumed lower and upper 32-bit components |
| F-b854c31d | First subsequent call without explicit arguments |
| F-903ae725 | Second subsequent call without explicit arguments |
| F-6a1f92c8 | Receives the word loaded through the working pointer |
| F-d20e7849 | Receives that word shifted right by three; returns consumed address/value components |
| F-f8130d6a | Receives the replacement pointer bit pattern shifted right by two |
| F-4b9e207c | Returns components used in ordered byte, word, and 64-bit stores |

Distinct target selections for F-29d6a04b need not identify the same implementation. Calling conventions, memory effects, register preservation, ownership behavior, and termination are unknown.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Skipped initialization with a nonzero shifted address.** Incoming r4 equal to `0x01000000` skips initialization. The common writes include zero halfword at `0x20000030`, zero word at `0x40`, then zero byte and incoming r7 word at `0xA1`.

2. **Two base observations.** If initialization runs and an intervening fixed-storage write changes the first base global from A to B, the `0xCE` byte-store address uses B plus the initial offset. If the initial call returns, the working pointer is nevertheless twice saved A.

3. **Initialization result addressing.** With incoming r4 equal to one, F-7c0a18e2 receives zero as its first argument. After return, halfword `0xECA0` is stored through four times its additional register result; the initial destination becomes zero.

4. **Wrapped short-path result.** If F-d20e7849’s lower component is `0x08000000`, multiplication by 32 yields zero. The routine writes word zero at address zero before continuing with target base `0x0897EDF1` and argument zero.

5. **Extended ordered stores.** If the latest F-4b9e207c components are lower `0x1000` and upper `0x12345678`, the routine first writes the captured byte at `0x2000`, then byte `0x78` at `0x1008`, then the earlier upper-component word at twice the accumulator, then the full 64-bit result at `0x1000`.

6. **Conditional breakpoint continuation.** If the unresolved expression yields nonzero and execution resumes after breakpoint four, the next first loop call uses base `0x08790D52` and the arithmetic-shift-derived argument. No loop-entry writes intervene.

7. **Access-sensitive destination.** A device-backed primary destination can observe byte, halfword, and word writes in that order. Replacing them with only the final word write does not preserve the represented behavior.

## Uncertainties

- **documented_only — global layout:** Fixed-global access widths, physical overlap, and exception-associated object mappings are undeclared.
- **documented_only — arithmetic:** Signed overflow and signed shifts require operational confirmation; the right shift by 32 is unresolved.
- **documented_only — execution environment:** Mapping, permissions, alignment, endianness, atomicity, and breakpoint handling are unknown.
- **documented_only — dependencies:** Their implementations, preservation rules, ownership effects, and termination are unavailable.
- **documented_only — terminal transfer:** Its treatment as a returning call is uncertain, and its branch is unreachable under the stated arithmetic model.
- **Inferred intent only:** Storage associations and access patterns do not establish an application-level responsibility.

## Evidence

This contract uses only the supplied static representation: explicit pointer-access widths, separate global evaluations, constants, arithmetic, call ordering, branch conditions, and loop edges.

Terminal-branch unreachability, retained high-bit address contributions, and the zero-only below-20 scaled comparison are static arithmetic deductions.

No implementation inspection beyond the supplied representation, dependency inspection, dynamic traces, or hardware observations were available. No runtime tests have been performed.
