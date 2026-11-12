<!-- Trusted provenance: F-efce6c915d50; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efce6c915d50

## Purpose

The supplied static representation establishes a routine that performs mixed-width memory accesses, invokes dependencies through fixed and computed targets, and enters cyclic control flow. It does not establish an application-level purpose or a normal-return path.

Observable effects include caller-associated writes, low-memory accesses, fixed-address mutations, writes through dependency results, a halfword write at the address associated with **F-422afe36d401**, and a conditional breakpoint with immediate value 4.

**Inferred intent:** Hardware interaction or executable-state modification is plausible, but neither interpretation is an established purpose.

This contract distinguishes represented behavior from target-specific execution semantics. No runtime tests have been performed.

## Interface

### Explicit arguments

The represented interface has nine argument positions and no represented return value.

| Position | Represented form | Visible use |
|---|---|---|
| 1 | Pointer to 32-bit elements | Receives a word at byte offset 84; its first word supplies another destination address. |
| 2, 3 | 32-bit opaque values | No visible use. |
| 4 | Signed 32-bit value | Supplies a truncated halfword and participates in address arithmetic. |
| 5, 6 | 32-bit opaque values | No visible use. |
| 7 | 32-bit opaque value | Its represented local value becomes zero. |
| 8 | 32-bit opaque value | No visible use. |
| 9 | Unsigned 32-bit value | Its represented local value becomes a zero-extended byte read. |

Replacing the represented values of arguments 7 and 9 does not establish writes to caller-owned argument storage.

### Implicit inputs

The routine also consumes:

- A pointer to 16-bit elements.
- A signed 32-bit scaling input.
- A 16-bit value for a fixed-address write.
- An address used in a comparison.
- A signed 32-bit additive input.
- A working-area pointer obtained from machine state.

Their provenance, calling-convention roles, validity, and lifetimes are **documented_only**.

### Widths and execution requirements

Access widths are 8, 16, 32, and 64 bits. Represented addresses and pointer-derived arithmetic use 32-bit quantities. Selected dependency results are represented as 64-bit quantities whose lower and upper 32-bit halves are used separately. This does not establish a source-level return type or result ABI.

Explicit unsigned 32-bit arithmetic has modulo-\(2^{32}\) behavior in the representation. Signed overflow, signed left shifts, negative signed right shifts, pointer conversions, and oversized shifts require separate qualification. In particular, a signed 32-bit right shift by 32 has no portable numerical interpretation.

Each reached access requires an environment supporting its address, width, permissions, alignment, and lifetime. No validating checks are shown. Byte order, atomicity, device-memory semantics, and fault behavior remain unknown.

### Outputs and errors

Outputs are memory effects, dependency invocations, state transitions, and potentially a breakpoint. No scalar result, success status, normal completion, rollback, or error translation is established.

A fault, nonreturning dependency, or breakpoint may prevent subsequent operations.

## Behavior

### Initial accesses

The initial ordering is:

1. Write a word at argument 1’s byte offset 84. Its value is the implicit halfword pointer converted to signed 32-bit and shifted right by three.
2. Read an unsigned halfword at that implicit pointer’s byte offset 14.
3. Read argument 1’s first word as an address, then write argument 4’s low 16 bits at that address plus eight.
4. Form the previously read halfword shifted left by seven.
5. Write the implicit pointer’s low 16 bits through that pointer.

The offset-84 write precedes the first-word read. The halfword read precedes the write through its associated pointer. Aliasing can therefore affect subsequent observations.

### Derived-address setup

A signed arithmetic expression adds five to the scaling input and multiplies by `0x01000000`. Where that arithmetic has a defined execution meaning, the resulting quantity is used as follows:

1. Write a zero word at that quantity plus `0x0897F15C`.
2. Form addresses corresponding to argument 4 plus 64 and argument 4 plus 68.
3. Read the word at argument 4 plus 64 as a destination and write that destination’s low 16 bits through it.
4. Write the derived quantity at its own address plus 20.
5. Read an unsigned halfword at the sum of the two argument-derived addresses.

The final address is twice argument 4 plus 132 only where the arithmetic model supports that simplification. The representation does not settle the observable read multiplicity of the repeated destination expression in step 3.

Only afterward is argument 4 plus 64 compared with zero. This check does not guard the preceding dereference.

On the nonzero path, the following occur in order:

- Write a halfword at `0x0897F07A`, using the narrowed argument-derived address multiplied by 4096.
- Write word `0x6000` at `0xBF836004`.
- Read a word at `0xC1B00050`.
- Write the implicit 16-bit entry value at `0xBF836002`.

That path selects operands `0x0897F06C` and `0xC1B00000`; its word read becomes the selected setup value. Otherwise, the operands are the shifted initial halfword and argument 4 plus 68, and the earlier derived-address halfword read becomes the zero-extended selected setup value.

### Initial call and low-memory effects

Before **F-unresolved-01**, the routine captures a word from `0x0897F454`. The call’s represented explicit argument is the selected operands’ exclusive-or, interpreted as signed 32-bit and shifted right by eight.

After the call:

1. Read a byte at 32 times the captured word and zero-extend it into the represented ninth argument.
2. Write a word at that same address containing 16 times the captured word.
3. Write a zero byte at address 9.
4. Write a zero halfword at address 48.
5. Write the captured word’s low halfword at address 14.

The address multiplications originate from a signed captured value, so signed-overflow semantics remain unresolved. The byte read precedes the overlapping word write. The captured value is not reloaded after the call.

### Outer-region state and recurrence

The outer region carries four relevant values in addition to the working pointer:

| Carried state | Initial value |
|---|---|
| Unsigned word stored before the working pointer | Selected setup value |
| Quantity used for the self-address and oversized-shift writes | Zero |
| Quantity contributing minus sixteen times itself to a halfword destination | Zero |
| Control-target value | `0x0897F0C3` |

Each iteration performs these ordered accesses:

1. Write the carried unsigned word at working-pointer offset minus four.
2. Write a word self-address value at eight times the first carried quantity.
3. Write a zero word at the address obtained by shifting that quantity left by 17, interpreting the result as signed, shifting right by 32, and adding 16.
4. Write halfword `0xC470` at the carried unsigned word shifted right by four, minus sixteen times the second carried quantity, minus one if bit 3 of the unsigned word is zero, plus 38.
5. Write word `0x8000` at `0xFE238028`.
6. Replace the first carried quantity with `0x0897F228`.
7. Write word `0x225FC470` at `0x225FC470`.
8. Write halfword `0x225F` at `0x0000229F`.
9. Read a word at `0x00000022` and write its upper 16 bits, zero-extended, at working-pointer offset 84.

Step 3 has an unresolved oversized shift; its destination must not be treated as a known constant.

If the implicit comparison address differs from the working pointer plus 20, **F-unresolved-02** is invoked at the carried control-target value plus `0x2E`. Before invocation, the carried control-target value becomes `0x0897F193`. With the initial control-target value, the computed target is `0x0897F0F1`. Skipping this call leaves the control-target value unchanged.

Before entering the inner region, the second carried quantity becomes `0x0897F1F8`. The accumulator is reset to zero. A represented 64-bit quantity receives upper half `0x0897F1F8` and lower half zero; the inner derived word becomes `0x0897F22D`.

On an overflow restart described below:

- The working pointer moves back four bytes.
- The unsigned word for the next outer store becomes `0x0897F1EC`.
- The first carried quantity remains `0x0897F228`.
- The second carried quantity remains `0x0897F1F8`.
- The current control-target value is carried forward.

Consequently, a restarted outer iteration does not repeat the initial zero-quantity destinations.

### Inner region

An inner pass performs these operations:

1. Write the derived word at the current 64-bit quantity’s lower-half address plus 12.
2. Interpret that lower half as signed 32-bit and shift it right by a count taken from its low byte.
3. Determine signed subtraction overflow for subtracting one from the shifted value.
4. Read a word at twice the shifted value.
5. Write the loaded word’s low halfword at the address associated with **F-422afe36d401**.
6. Write word `0x0897F1F8` at the loaded word’s address plus four.

The overflow decision occurs after these memory effects. If true, it restarts the outer region using the recurrence above.

Otherwise:

- Write byte `0xD8` through the current 64-bit quantity’s upper-half address.
- Compute the target of **F-unresolved-03** as the carried control-target value plus `0xE56`.
- Replace the control-target value with `0x0897F1D7` before invoking that target with zero.
- Add the implicit additive input to the accumulator.
- Replace the working pointer with the zero-extended halfword read at `0x0897F216`.
- Read a byte at the call result’s lower half plus `0x0897F1F8`.

If the outer conditional call most recently set the control-target value to `0x0897F193`, the inner target is `0x0897FFE9`. If it remains the initial `0x0897F0C3`, the inner target is `0x0897FF19`.

An explicitly zero indicator is compared with the earlier overflow indicator. On the represented non-overflow path they are equal. The resulting transition reads a halfword at the call result’s lower half plus the derived word, treats the zero-extended halfword as a destination, and writes the accumulator there.

The first inner pass starts with lower half zero. Under ordinary represented arithmetic, its shifted value is zero and subtracting one does not overflow. The overflow restart remains a conditional control-flow description, not a claim that this initial pass reaches it.

### Later cycle: opening and small destination

The later cycle initially carries:

- Opening-call argument zero.
- Current address `0x0897F1F8`.
- Word intermediate 8.
- The byte read following **F-unresolved-03** as its byte source.
- The working pointer and accumulator established above.

Before **F-unresolved-04**, the low eight bits of the byte source are saved. The call receives the opening-call argument.

Its represented 64-bit result supplies a retained upper-half pointer and a destination formed from 32 times its signed lower half. Signed multiplication overflow remains unresolved. The destination receives its own address as a word before an unsigned comparison with 20.

For a destination below 20, the larger mutation block is skipped. The current address and word intermediate remain unchanged. The common-tail argument is zero, and the common-tail target is `0x0897F80F`.

### Later cycle: larger mutation block

For a destination not below 20, the routine:

1. Writes the current address at destination offset 88.
2. Writes the working pointer’s low halfword at current-address offset 14.
3. Replaces the word intermediate with the previous current address shifted left by two.
4. Reads a separately identified reset-source byte.
5. Writes a reset-associated halfword combining that byte in bits 15–8 with the saved byte in bits 7–0.
6. Invokes **F-unresolved-05** with the unsigned working-pointer value shifted right by two.
7. Invokes **F-unresolved-06**, obtaining a new represented 64-bit result.

The reset-source read follows the first two stores. No supplied address mapping establishes that it reads the reset destination’s upper byte.

The newer result supplies a lower-half address and a newer upper-half pointer. The routine then writes, in order:

- The saved byte at twice the newer lower-half address.
- The newer upper-half pointer’s low byte at newer-lower-half address plus eight.
- The retained opening-call upper-half pointer through twice the previous accumulator.
- The complete 64-bit result through its lower-half address.

The accumulator becomes four times the retained pointer. The current address becomes the working-pointer value shifted left by eight. Three consecutive words are written through the retained pointer: the word intermediate, the new current address, and the working-pointer value.

The newer upper-half pointer is not substituted for the retained pointer in those three stores.

A branch quantity is formed by shifting the working-pointer value left by 31, interpreting it as signed for a right shift by 32, then interpreting that result as unsigned and shifting right by 20. Its numerical value remains **documented_only**.

If that quantity is zero:

- Write the low 16 bits of the word intermediate multiplied by 256 to the reset destination, without another reset-source read.
- Add the word intermediate to the accumulator.
- Write the accumulator through the newer upper-half pointer.
- Select common-tail argument zero and target `0x0897F83B`.

Otherwise:

- Write the branch quantity at word-intermediate-times-256 plus 32.
- Select that multiplied value, interpreted as signed and shifted right by 29, as the common-tail argument.
- Execute a breakpoint with immediate 4.
- If execution continues, replace the current address with the newer upper-half pointer interpreted as signed and shifted right by 18.
- Select common-tail target `0x08791770`.

### Later cycle: common tail

**F-unresolved-07** is invoked at the path-selected target with the selected argument. The target distinctions above are established computed-dispatch selections, not unknown callee behavior.

The tail then:

1. Writes the result’s low halfword through its lower-half address.
2. Writes a word self-address value at 64 times the current address.
3. Captures a word through the current working pointer.
4. Writes a halfword at current-address offset four containing the low 16 bits of the old working-pointer value plus four.
5. Replaces the working pointer with the word-sized pointer read at the result’s upper-half address plus 80.
6. Invokes **F-unresolved-08**, then **F-unresolved-09**, without represented explicit arguments.
7. Sets the byte source to the current word intermediate.
8. Invokes **F-unresolved-10** with the captured word.
9. Sets the next opening-call argument to the captured unsigned word shifted right by three, and the next word intermediate to the captured word.

The capture precedes the potentially aliasing halfword store and survives subsequent calls as a represented carried value. The next saved byte comes from the previous word intermediate, not necessarily the newly captured word. Control returns to the opening call.

## Side effects

Writes are nontransactional. Earlier mutations may remain observable when later execution faults or stops.

Affected storage includes caller-associated memory, implicit working areas, low memory, fixed addresses, result-derived storage, the reset-associated destination, and the address associated with **F-422afe36d401**. Differently sized accesses may overlap.

No allocation, release, ownership transfer, lifetime extension, atomic read-modify-write, synchronization barrier, or instruction-cache synchronization guarantee is directly established. Dependencies are not assumed to preserve globals, mappings, or storage validity.

## Dependencies

**F-422afe36d401** identifies an address receiving a halfword write, not a visible invocation.

**F-unresolved-01** through **F-unresolved-10** identify the call sites described above. They do not establish distinct or stable callee identities.

Additional established target selections include:

| Call site | Represented target |
|---|---|
| **F-unresolved-04** | `0x07F755F6` |
| **F-unresolved-05** | `0x0897EE9F` |
| **F-unresolved-08** | `0x0897F191` |
| **F-unresolved-09** | `0x0897FDDD` |
| **F-unresolved-10** | `0x0897F5E1` |

**F-unresolved-06** uses an address-associated base plus `0xFFE`; the supplied representation does not independently establish that base’s numerical address.

Known target arithmetic does not resolve calling conventions, complete argument sets, return conventions, ownership effects, failures, or machine-state preservation. These remain **documented_only** for every dependency.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial truncation.** Argument 4 equal to `0x12345678` supplies halfword `0x5678` at the address read from argument 1’s first word, plus eight.

2. **Late check.** If argument 4 plus 64 resolves to zero, the earlier dereference still precedes the zero comparison.

3. **Initial outer state.** If the selected setup value is `0x100`, the first outer store writes `0x100` before the working pointer. The self-address write targets zero. The halfword destination is `0x35`: `0x10 − 1 + 0x26`.

4. **Conditional restart.** If overflow causes a restart, the next preceding-word store contains `0x0897F1EC`, and the self-address destination is eight times `0x0897F228`, namely `0x44BF9140`. Its quantities are not reset to zero.

5. **Dispatch update.** On the initial outer conditional-call path, the target is `0x0897F0F1`, while the carried control-target value has already become `0x0897F193`. A subsequent inner call therefore targets `0x0897FFE9`.

6. **Skipped outer dispatch.** If the outer call is skipped while the control-target value remains initial, the inner call targets `0x0897FF19` and updates the control-target value to `0x0897F1D7` before invocation.

7. **Reset composition.** A reset-source byte of `0xA5` and saved byte of `0x3C` produce halfword `0xA53C`. This does not establish preservation of the destination’s previous upper byte.

8. **Distinct result pointers.** If the opening result’s upper half is P and the newer result’s upper half is Q, the three consecutive word stores target P; the zero-quantity accumulator store targets Q.

9. **Tail selection.** The small-destination, larger-block zero-quantity, and continuing breakpoint paths select targets `0x0897F80F`, `0x0897F83B`, and `0x08791770`, respectively.

10. **Captured and delayed values.** A captured word `0x12345678` supplies next opening argument `0x02468ACF`. If the preceding word intermediate was `0x11223344`, the next saved byte is `0x44`, regardless of the captured word’s low byte.

## Uncertainties

The following remain **documented_only**:

- Actual instruction set, byte order, address model, alignment, and calling convention.
- Function-boundary fidelity and implicit-input provenance.
- Signed-overflow, pointer-conversion, and oversized-shift semantics.
- Original machine overflow and branch-condition behavior.
- Exact read multiplicity for repeated expressions.
- Dependency contracts, result conventions, ownership, and state preservation.
- The reset-source byte’s address relationship to the reset destination.
- Memory-map classifications, access side effects, and breakpoint handling.
- Consequences of writing at the dependency-associated address.
- Any return behavior absent from the supplied representation.

These unknowns do not erase the established widths, ordering, carried-state assignments, numerical dispatch selections, or conditional transitions.

## Evidence

**Established static evidence:** The supplied representation contains nine explicit argument positions, implicit inputs, ordered mixed-width accesses, outer-state initialization and overflow recurrence, computed-target updates before invocation, path-specific common-tail targets, distinct result-pointer provenance, captured values, and cyclic control flow.

**Inferred intent:** Hardware interaction and executable-state modification are possible interpretations only.

**Documented_only:** Target-specific execution semantics, dependency behavior, ownership, unresolved aliasing, reconstruction fidelity, and application purpose.

Evidence is limited to the supplied static representation. No runtime tests, smoke runs, or instruction-level validation have been performed.
