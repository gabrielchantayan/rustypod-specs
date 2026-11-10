<!-- Trusted provenance: F-aa38feacc6a4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa38feacc6a4

## Purpose

The supplied static representation establishes an initialization sequence followed by indefinitely repeated dependency calls, address-derived memory accesses, conditional re-entry, and a software breakpoint. It does not establish a reliable application-level purpose.

No normal return path is represented. Execution may continue through the represented loops, fault during an access, stop or transfer control inside a dependency, or enter breakpoint handling.

**Inferred intent:** the fixed-address assignment and breakpoint may indicate low-level execution-state manipulation. This interpretation is not established; no destination is identified as a device register, allocated object, or ordinary writable storage.

This contract is **documented_only**. It describes static evidence and conditional consequences. No runtime tests have been performed.

## Interface

### Explicit inputs

The represented interface has five argument positions:

| Position | Represented type | Visible role |
|---|---|---|
| First | Pointer to 32-bit unsigned words | Initialization destination and numerical source for address arithmetic. |
| Second | Opaque 32-bit value | No visible use. |
| Third | Unsigned 32-bit value | Initialization data, mask input, address source, and initial processing state. |
| Fourth | Opaque 32-bit value | No visible use. |
| Fifth | Signed 32-bit value | Participates in an initial signed address calculation. |

Three incoming machine-state values are additional effective inputs:

- A pointer to 32-bit words, subsequently replaced by pointers read from memory.
- A signed 32-bit address value used by the first halfword write.
- A 32-bit value used for initialization and masking.

Their origins and calling-convention status are unresolved. Their presence does not establish that they contain valid addresses or initialized values.

### Outputs and memory model

No explicit return value or normal completion is represented. There is no visible error code, validation failure result, rollback, or cleanup path.

Explicitly typed accesses include 8-, 16-, 32-, and 64-bit accesses. **Two global assignments have unresolved destination widths:** the assignment of `0x20000000` to storage associated with `0x20000040`, and the assignment of a captured byte value to reset-associated global storage. Neither assigned expression establishes the destination’s storage width.

The representation uses 32-bit words and address conversions consistent with a 32-bit address model. The low and high halves of a 64-bit result mean its numerical low and high 32 bits; this does not establish memory byte order.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). Signed arithmetic performed before conversion to unsigned does not acquire wrapping semantics merely because its result is later stored in an unsigned word.

Every reached access requires suitable mapping, permissions, and alignment. No checks establish these conditions. Several paths attempt an access at address zero.

### State terminology

The following labels describe processing state, not additional interface parameters:

| State | Initial value |
|---|---|
| Current pointer | Incoming implicit word pointer. |
| Working word | Eight times the original third argument, minus two, modulo \(2^{32}\). |
| Step word | Original third argument times `0x02000000`, modulo \(2^{32}\). |
| Accumulator | Bitwise intersection of the incoming initialization value and original third argument. |
| Active destination | Address formed from the original third argument times `0x01000000`, modulo \(2^{32}\). |
| Secondary destination | Initially the active destination. |
| Payload | Original third argument shifted left by 28 as an unsigned 32-bit word. |
| First-call argument | Signed 32-bit value determined by the processing-entry path. |

## Behavior

### Initialization

The following effects occur in order, assuming each preceding operation completes:

| Order | Destination | Operation |
|---|---|---|
| 1 | Incoming signed address plus 6 | Write the original third argument’s low 16 bits using a 16-bit access. |
| 2 | Twice the signed-converted first-argument address, plus 48 | Write the doubled address narrowed to 16 bits. |
| 3 | Global storage associated with `0x20000040` | Assign `0x20000000`; destination width is unresolved. |
| 4 | First-argument address | Write zero using an 8-bit access. |
| 5 | First-argument address | Write the incoming initialization value using a 32-bit access. |
| 6 | First-argument address plus 40 bytes | Write 64 using a 32-bit access. |
| 7 | First-argument address plus 44 bytes | Write the numerical first-argument address using a 32-bit access. |

The byte write and the overlapping word write are separate represented accesses. The global assignment between the initial halfword writes and those accesses must remain in that order, but cannot be characterized as a 32-bit store. Its declaration, destination conversion, and storage footprint are unavailable.

The accumulator and working word are initialized. A signed quantity is then formed from the signed-converted current-pointer address, plus the fifth argument, plus 8. A 32-bit write stores that quantity at an address equal to twice the quantity.

The initial signed address additions and multiplications require representable results. Conversion of an address outside the signed 32-bit range requires target-specific interpretation. No unconditional wrapping result is established for these operations.

Initialization then establishes the active destination and performs:

1. A 32-bit write of the original third argument at the first-argument address plus 4 bytes.
2. An 8-bit write of the working word’s low byte at the active destination plus 20 bytes.

The initial step word, payload, and secondary destination are then established.

### Re-entry writes

Initial processing entry and one later branch execute this sequence:

1. Write zero at the active destination using an 8-bit access.
2. Write the working word’s low 16 bits at the active destination using a 16-bit access.
3. Write the payload at the active destination using a 32-bit access.
4. Add the step word to the accumulator modulo \(2^{32}\).
5. Write the updated accumulator at the secondary destination using a 32-bit access.

Initially, both destinations coincide. Later re-entry may use different destinations.

After these writes, the first-call argument is set to zero.

### Processing-cycle entry

The first-call argument is not reset on every processing cycle:

- Initial entry through the re-entry writes supplies zero.
- Later entry through those writes supplies zero.
- Restart following the scaled-result comparison supplies zero, without executing the re-entry writes.
- Continuation after the breakpoint retains a payload-derived argument instead.

At each cycle’s start, the low byte of the current step word is captured. This captured byte is retained for later uses even though the step word is subsequently replaced.

The selected first-call dependency is invoked with the current first-call argument. Its consumed result is 64 bits.

### First result and pointer replacement

The following accesses and calls occur in order:

1. Write the result’s low 16 bits using a 16-bit access at the address represented by its low 32 bits.
2. Multiply the working word by 64 using unsigned 32-bit arithmetic. Write that product using a 32-bit access at an address equal to the product.
3. Read a 32-bit word through the current pointer. This becomes the new step word.
4. Write a 16-bit value at the working word plus 4. The value is the old current-pointer address narrowed to 16 bits, plus 4, with the stored result narrowed to 16 bits.
5. Read a replacement pointer from the address formed by converting the first result’s high half to signed 32 bits and adding 80 bytes.
6. Replace the current pointer with that pointer.
7. Invoke F-000000000005, then F-000000000006, with no explicit arguments.
8. Invoke F-000000000007 with the new step word.
9. Invoke F-000000000008 with the step word logically shifted right by 3; consume its 64-bit result.

The multiplication by 64 does not update the working word. The halfword destination in step 4 therefore uses the unscaled working word.

The old pointer supplies both the step-word read and the following halfword’s value. Pointer replacement occurs before the three intervening dependencies.

The replacement-pointer address calculation has unresolved behavior if signed conversion or addition lacks an established target result.

### Scaled-result write and comparison

The high half of F-000000000008’s result becomes the active destination.

Separately, the result is converted to signed 32 bits, multiplied by 32 as signed arithmetic, and assigned to an unsigned 32-bit word. If the converted signed value is \(s\), multiplication is representable only for:

\[
-67{,}108{,}864 \leq s \leq 67{,}108{,}863.
\]

An out-of-range signed conversion is target-dependent. Signed multiplication overflow is not defined by the later unsigned assignment.

When these operations have established results, a 32-bit write stores the resulting unsigned word at an address equal to that word. The write precedes the comparison.

The word is then compared unsigned with 20:

- **Below 20:** processing restarts without re-entry writes. The next first-call argument is zero.
- **At least 20:** execution follows the extended path.

Under representable signed multiplication, zero is the only resulting unsigned multiple of 32 below 20. Overflowing cases cannot be classified by assuming modulo-\(2^{32}\) multiplication.

### Extended path

The extended path first performs:

1. A 32-bit write of the working word at the scaled address plus 88.
2. A 16-bit write of the narrowed current-pointer address at the working word plus 14.
3. An update of the step word to the working word shifted left by 2 using unsigned arithmetic.
4. An assignment of the captured byte value to reset-associated global storage.

The fourth operation has **unresolved destination width**. The assigned value is the byte captured at cycle entry, not the low byte of the newly updated step word. Its source width does not establish an 8-bit destination access or storage footprint.

F-000000000009 is then called with the current-pointer address logically shifted right by 2. F-00000000000a follows with no explicit arguments and supplies a 64-bit result.

That result establishes the secondary destination from its high half and a data destination from its low half. The following writes occur in order:

1. Write the captured byte using an 8-bit access at twice the signed-converted data-destination address.
2. Write the low byte of the result’s high half using an 8-bit access at the data destination plus 8 bytes.
3. Write the active-destination address using a 32-bit access at twice the accumulator, calculated with unsigned arithmetic.
4. Write the complete result using a 64-bit access at the data destination.

The first address calculation uses signed multiplication; conversion and overflow remain qualified. The 64-bit access’s atomicity and physical decomposition are unknown.

The accumulator is then assigned the signed-converted active-destination address shifted left by 2. The working word is assigned the signed-converted current-pointer address shifted left by 8. These signed shifts are not unconditionally equivalent to unsigned wrapping shifts.

Subject to usable results, three consecutive 32-bit writes occur at active-destination byte offsets 0, 4, and 8, storing:

- The step word.
- The updated working word.
- The current-pointer address.

The payload becomes the step word multiplied by 256 modulo \(2^{32}\).

### Re-entry branch and breakpoint

A branch quantity is formed by:

- Converting the current-pointer address to signed 32 bits.
- Applying a signed left shift by 31 and signed right shift by 32.
- Converting to unsigned and logically shifting right by 20.

A shift count of 32 on a represented 32-bit operand has no portable defined interpretation. The signed left shift also has validity constraints. Neither branch can be ruled out without additional target evidence.

The active destination is set to zero before the branch.

**If the branch quantity is zero**, execution returns to the re-entry writes. The first access is an 8-bit zero write at address zero. The first-call argument is reset to zero after those writes.

**If the branch quantity is nonzero**, a 32-bit write stores it at the payload plus 32. The next first-call target is selected, and the first-call argument becomes the payload converted to signed 32 bits and right-shifted by 29. Software breakpoint number 4 is executed.

Conversion of a high-bit-set payload and right shifting a negative signed value require target interpretation.

If execution resumes at the represented continuation, the working word becomes the secondary-destination address converted to signed 32 bits and right-shifted by 18. Execution then enters another processing cycle directly.

There is no intervening zero reset or re-entry write sequence. The next first-call dependency receives the payload-derived argument.

## Side effects

The visible body modifies memory reached through explicit and implicit inputs, arithmetic-derived addresses, dependency results, and two global assignments.

The fixed-address global assignment supplies `0x20000000`; the reset-associated assignment supplies the captured byte value. Their destination widths and affected byte ranges are not established. They must not be treated as proven 32-bit and 8-bit stores respectively.

Explicitly typed narrow and overlapping accesses remain distinct. Aliasing can cause earlier writes to change later reads. The supplied overlapping-global warning also prevents assuming global storage is disjoint from other destinations.

No allocation, deallocation, reference-count operation, or ownership transfer is visible. Passing an address-derived integer does not establish that a dependency acquires or releases memory.

Dependencies and breakpoint handling may modify memory, globals, or machine state, or fail to return. No preservation or ownership contract is available for them.

The represented ordering does not establish atomicity, hardware bus ordering, synchronization, barriers, or rollback.

## Dependencies

These opaque IDs identify call roles, not verified external contracts.

| ID | Role and consumed interface |
|---|---|
| F-000000000001 | First call on initial processing entry; signed 32-bit zero argument, 64-bit result consumed. |
| F-000000000002 | First call after scaled-result restart; signed 32-bit zero argument, 64-bit result consumed. |
| F-000000000003 | First call after extended-path re-entry writes; signed 32-bit zero argument, 64-bit result consumed. |
| F-000000000004 | First call after breakpoint continuation; payload-derived signed argument, 64-bit result consumed. |
| F-000000000005 | First call after pointer replacement; no explicit arguments, result unused. |
| F-000000000006 | Second call after pointer replacement; no explicit arguments, result unused. |
| F-000000000007 | Receives the current 32-bit step word; result unused. |
| F-000000000008 | Receives the step word logically shifted right by 3; 64-bit result consumed. |
| F-000000000009 | Receives the current-pointer address logically shifted right by 2; result unused. |
| F-00000000000a | No explicit arguments; supplies the extended-path 64-bit result. |
| F-00000000000b | Breakpoint mechanism invoked with number 4; continuation unresolved. |

The four first-call roles arise from different target constructions. Their validity, distinctness, and semantic relationship are not established.

## Acceptance examples

These are conditional static consequences, not executed tests. Each assumes preceding operations complete and necessary target interpretations are supplied.

1. **Initial global-assignment ordering.**  
   The assignment of `0x20000000` to storage associated with `0x20000040` follows the two initial halfword writes and precedes the byte-zero write through the first argument. Acceptance requires this assignment value and ordering, not a four-byte storage footprint.

2. **Original third argument is zero.**  
   The initial working word is `0xFFFFFFFE`. Active destination, step word, payload, and accumulator are zero. The preliminary byte write targets address 20 and stores `0xFE`. Re-entry starts at address zero.

3. **Original third argument is one.**  
   The initial working word is 6, active destination is `0x01000000`, step word is `0x02000000`, and payload is `0x10000000`. The first cycle’s self-address write stores 384 at address 384. The later halfword write targets address 10, not 388.

4. **F-000000000008 returns zero.**  
   Signed conversion and multiplication produce zero. A 32-bit zero write at address zero precedes restart. If it completes, the next first-call argument is zero and no re-entry writes intervene.

5. **F-000000000008 returns one.**  
   Signed multiplication produces 32 without overflow. A 32-bit write stores 32 at address 32, then the extended path is selected.

6. **F-000000000008 returns `0x0000000008000000`.**  
   Conversion is representable, but multiplication by 32 exceeds signed 32-bit range. No zero product or restart is established. Those outcomes require an additional target-specific wrapping assumption.

7. **Captured byte differs from the updated step word’s low byte.**  
   If the captured byte is `0xAB` and the updated step word’s low byte is zero, the reset-associated global assignment supplies `0xAB`. A one-byte destination write is not established.

8. **Breakpoint path with payload `0x20000000`.**  
   Conditional on reaching the nonzero branch and resuming, the payload-derived argument is 1. After the continuation’s working-word update completes, the next first call receives 1, not zero.

9. **Zero branch quantity or nonresuming breakpoint.**  
   A zero quantity selects re-entry beginning with an 8-bit write at address zero. If breakpoint handling instead never resumes, no subsequent working-word update or processing cycle is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose and accuracy of the represented interface.
- Origins and validity of implicit inputs.
- Destination declarations, widths, conversions, and storage footprints for both global assignments.
- Calculated call-target identity and executability.
- Dependency side effects, preservation, ownership, failure, and termination.
- Memory mapping, alignment, permissions, byte order, and atomicity.
- Pointer-to-integer and out-of-range signed conversions.
- Overflowing signed additions and multiplications.
- Signed left shifts and right shifts of negative values.
- The right shift by the represented operand width.
- Breakpoint handling and continuation.
- Aliasing and overlapping-global effects.
- Whether initialization can finish.

Missing checks do not establish successful execution or any particular failure mechanism.

## Evidence

Established static evidence includes explicitly typed access widths, offsets, assignment expressions, arithmetic conversions, call order, result-half extraction, loop nesting, and breakpoint placement.

The two global assignments lack supplied declarations or destination types. Their right-hand-side values establish what is assigned, not the width of storage written. Their ordering remains visible.

The zero assignment outside the inner processing loop establishes the entry-dependent first-call argument. Signed multiplication before unsigned assignment establishes the scaled-result overflow qualification. The multiplication by 64 does not assign back to the working word.

Warnings about omitted unreachable material and overlapping globals limit completeness and disjointness assumptions; they do not establish additional behavior.

Intent, target-specific outcomes, dependency contracts, and successful execution remain unverified. No runtime tests have been performed.
