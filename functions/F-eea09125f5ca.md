<!-- Trusted provenance: F-eea09125f5ca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eea09125f5ca

## Purpose

F-eea09125f5ca performs a fixed sequence of opaque calls, ten coprocessor-transfer operations, integer transformations, and a final 16-bit memory write. The supplied representation ends at a bad-instruction termination marker rather than an established normal return.

No application-level purpose is established. A processor-configuration or device-control interpretation is **inferred intent**, not part of the behavioral guarantee.

This contract distinguishes:

- **Established static evidence:** represented operation order, operand dependencies, widths, and the final store.
- **Conditional deductions:** arithmetic consequences under a 32-bit machine-word interpretation.
- **documented_only unknowns:** unresolved dependency behavior, architectural interpretation, execution validity, and termination beyond the represented endpoint.

All descriptions of later operations are conditional on earlier operations completing and returning as represented. No runtime tests have been performed.

## Interface

### Explicit input

The sole explicit input is an opaque 32-bit word. It is forwarded unchanged to F-indirect-01.

No interpretation as a pointer, count, enumeration, string, or structure is established. No validation precedes its forwarding.

### Implicit inputs

The represented computation also depends on:

- Two incoming 32-bit general-register words, designated **incoming word A** and **incoming word B**.
- An incoming link/control value used to form the first indirect-call target.
- Seven incoming opaque 32-bit coprocessor-related operands.
- Callable destinations for the remaining indirect calls.
- Any memory, globals, registers, or processor state accessed by dependencies.

For the transfer descriptions, the seven coprocessor-related operands receive descriptive labels:

| Label | Identifying use in the represented sequence |
|---|---|
| Control A | Fifth operand of transfer T1 |
| Control B | Sixth operand of every transfer; also fifth operand of T3 and T9 |
| Control C | Fifth operand of T2 |
| Control D | Fifth operand of T4 |
| Control E | Fifth operand of T5 |
| Control F | Fifth operand of T6, T8, and T10 |
| Control G | Fifth operand of T7 |

These labels identify distinct incoming operand dependencies, not established architectural register names. Distinct labels do not require distinct numeric values.

The incoming general-register words and control operands are not established as formal arguments under a known calling convention.

### Outputs and termination

No public return value or normal-return path is established.

Potential externally observable effects include dependency effects, coprocessor operations, and a final 16-bit store of numeric value 205. Execution can fail or cease before that store. After the store, the representation ends at a bad-instruction marker; the actual processor outcome there is unknown.

### Widths and arithmetic

For conditional arithmetic deductions:

- A word has 32 bits.
- A double word has 64 bits.
- A halfword has 16 bits.
- A double word’s low and high words are bits 0–31 and 32–63.
- Word left shifts retain the low 32 bits.
- Unsigned right shifts insert zeros.
- Signed right shifts are treated as arithmetic right shifts where explicitly stated.

These are machine-bit-pattern interpretations, not claims about portable signed-language arithmetic. Several represented left shifts have signed operands; their portable language-level behavior is not established.

No memory byte order follows from the low/high-word terminology.

## Behavior

### Ordering and call identities

The represented body has no conditional branch or loop. Calls and coprocessor transfers occur in a fixed order, followed by the final store.

F-indirect-01 through F-indirect-49 identify **49 indirect-call occurrences**, not 49 proven distinct functions. Together with F-3de949054bdd, the body contains **50 calls**.

“No explicit arguments” below means that none are shown in the supplied representation. It does not establish that a dependency consumes no machine state.

The first indirect target is formed from the incoming link/control value plus 228. Its validity, address-width behavior, and execution-state interpretation are unknown. The remaining indirect targets are represented using fixed numeric or code-location-relative expressions; their implementations are unresolved.

### Initial calls

The following calls occur in order:

| Dependency | Represented arguments and result use |
|---|---|
| F-indirect-01 | Receives the explicit input unchanged. |
| F-indirect-02 | No explicit arguments; result not consumed. |
| F-indirect-03 | No explicit arguments; result not consumed. |
| F-indirect-04 | Returns a word designated **seed**. |
| F-indirect-05 | Receives seed, incoming word A shifted left by 8, and seed shifted left by 5. |
| F-indirect-06 | Returns an opaque word. |
| F-indirect-07 | Receives F-indirect-06’s result. |
| F-3de949054bdd | No explicit arguments; result not consumed. |
| F-indirect-08 | Returns an opaque word. |
| F-indirect-09 | Receives F-indirect-08’s result. |
| F-indirect-10 | Returns an opaque word. |
| F-indirect-11 | Receives F-indirect-10’s result. |
| F-indirect-12 | Receives incoming word B shifted left by 13, then the signed right shift by 5 of the previously formed seed-left-shifted-by-5 word. |
| F-indirect-13 | Receives F-indirect-12’s result. |
| F-indirect-14 | Returns a double word. |

Under 32-bit arithmetic-right-shift semantics, F-indirect-12’s second argument is not generally seed. The initial left shift discards seed’s upper five bits. The subsequent right shift sign-extends original bit 26.

### First transfers and selector processing

After F-indirect-14 returns:

1. Transfer T1 receives its low word.
2. F-indirect-15 receives its high word shifted left by 1.
3. F-indirect-16 returns a word.
4. Transfer T2 receives that word.
5. Transfer T3 receives the same word, but with a different fifth control operand.
6. F-indirect-17 returns a word designated **selector**.

The function forms **expanded selector** by shifting selector left by 27. It then performs:

| Dependency | Represented arguments and result use |
|---|---|
| F-indirect-18 | Receives zero, zero, expanded selector, zero. |
| F-indirect-19 | No explicit arguments; result not consumed. |
| F-indirect-20 | No explicit arguments; result not consumed. |
| F-indirect-21 | No explicit arguments; result not consumed. |
| F-indirect-22 | No explicit arguments; result not consumed. |
| F-indirect-23 | No explicit arguments; result not consumed. |
| F-indirect-24 | Returns a word supplied immediately to transfer T4. |
| F-indirect-25 | No explicit arguments; result not consumed. |
| F-indirect-26 | No explicit arguments; result not consumed. |
| F-indirect-27 | No explicit arguments; result not consumed. |

Transfer T5 then receives expanded selector shifted left by 24. F-indirect-28 receives that same shifted value followed by zero.

Under the 32-bit model, expanded selector has possible set bits only in positions 27–31. Its further left shift by 24 is therefore zero. This arithmetic simplification does not eliminate T5 or F-indirect-28.

### Payload processing

The next operations are:

1. F-indirect-29 is called without explicit arguments.
2. F-indirect-30 returns a word.
3. Transfer T6 receives that word.
4. F-indirect-31 is called without explicit arguments.
5. F-indirect-32 receives three zero words and returns a word designated **payload**.

F-indirect-32’s first two arguments arise from left shifts by 22 and 11 of an intermediate previously assigned zero. Its third argument is literal zero.

Transfer T7 receives payload shifted left by 1. The function forms **scaled payload** by shifting payload left by 4, then calls F-indirect-33 with:

1. Payload shifted left by 1.
2. Expanded selector masked to its low 23 bits, then shifted left by 4.
3. Scaled payload.
4. Zero.

The second argument is zero under the 32-bit model because expanded selector has no set bits in its low 23 positions.

F-indirect-34 and F-indirect-35 follow, each without explicit arguments. Then:

1. Transfer T8 receives zero.
2. A **derived temporary** is formed from scaled payload’s low two bits, shifted left by 16.
3. Transfer T9 receives zero.
4. F-indirect-36 receives zero and returns an **auxiliary result**.
5. Transfer T10 receives scaled payload shifted left by 8.
6. F-indirect-37 receives scaled payload shifted left by 8, scaled payload shifted left by 12, and auxiliary result shifted left by 9.

Scaled payload has four low zero bits under the word model. Consequently, the derived temporary is zero.

### Coprocessor-transfer operands

There are exactly **ten** represented transfers. Each has six ordered operands. The table records all six: three leading selector/control fields, a 32-bit data operand, and two trailing incoming control operands.

The field positions are descriptive. Their architectural meanings, admissible values, privilege requirements, and effects are not established.

| Transfer | Position in execution | Field 1 | Field 2 | Field 3 | Data operand | Field 5 | Field 6 |
|---|---|---:|---:|---:|---|---|---|
| T1 | After F-indirect-14, before F-indirect-15 | 0 | 6 | 7 | Low word of F-indirect-14’s result | Control A | Control B |
| T2 | After F-indirect-16 | 0 | 3 | 7 | F-indirect-16’s result | Control C | Control B |
| T3 | Immediately after T2 | 0 | 3 | 7 | The same F-indirect-16 result | Control B | Control B |
| T4 | Immediately after F-indirect-24 | 0 | 1 | 7 | F-indirect-24’s result | Control D | Control B |
| T5 | After F-indirect-27, before F-indirect-28 | 0 | 5 | 7 | Expanded selector shifted left by 24 | Control E | Control B |
| T6 | After F-indirect-30, before F-indirect-31 | 0 | 6 | 7 | F-indirect-30’s result | Control F | Control B |
| T7 | After F-indirect-32, before F-indirect-33 | 0 | 5 | 7 | Payload shifted left by 1 | Control G | Control B |
| T8 | After F-indirect-35 | 0 | 7 | 7 | Zero | Control F | Control B |
| T9 | After formation of the derived temporary, before F-indirect-36 | 0 | 0 | 7 | Zero | Control B | Control B |
| T10 | After F-indirect-36, before F-indirect-37 | 0 | 2 | 7 | Scaled payload shifted left by 8 | Control F | Control B |

T2 and T3 are separate operations despite having identical transferred data and leading fields. Their fifth operands differ in provenance.

Repeated control labels record repeated dependencies on the same incoming operand in the representation. They do not establish that intervening callees preserve physical coprocessor registers, globals, or other architectural state.

### Final calls

After F-indirect-37, execution continues:

| Dependency | Represented arguments and result use |
|---|---|
| F-indirect-38 | Returns a double word. |
| F-indirect-39 | Receives that result’s low word, high word, and low word shifted left by 1. |
| F-indirect-40 | Receives the derived temporary shifted left by 8; returns a word. |
| F-indirect-41 | Receives F-indirect-40’s result; returns a word. |
| F-indirect-42 | Receives F-indirect-41’s result. |
| F-indirect-43 | No explicit arguments; result not consumed. |
| F-indirect-44 | Returns a double word. |
| F-indirect-45 | Receives that result’s low word, high word, and high word shifted left by 11; returns a double word. |
| F-indirect-46 | Receives the low word of F-indirect-45’s result. |
| F-indirect-47 | Returns a word. |
| F-indirect-48 | Receives F-indirect-47’s result and the derived temporary shifted left by 25. |
| F-indirect-49 | No explicit arguments; result not consumed. |

The double-word extraction needed for the final address occurs after F-indirect-45 and before F-indirect-46. The temporary’s left shift by 25 also occurs before F-indirect-46.

Under the word model, the temporary-derived arguments to F-indirect-40 and F-indirect-48 are both zero.

No result validation, status interpretation, or conditional error handling is represented.

### Final address and store

The final address derives from F-indirect-45’s double-word result, not from the last call.

The function extracts bits 41–63 using an unsigned right shift by 41, retaining the result as a 32-bit unsigned word. That word has at most 23 significant bits, so interpreting it as signed leaves it nonnegative.

It then shifts that signed interpretation right by 11 and adds the result to its own unsigned right shift by 2.

Equivalently, let \(q\) be the unsigned value of bits 52–63 of F-indirect-45’s result. Then:

\[
0 \le q \le 4095,\qquad
\text{address}=q+\left\lfloor q/4\right\rfloor.
\]

The generated addresses range from 0 through 5118 inclusive, but do not include every integer in that interval. The address arithmetic does not overflow a 32-bit word.

After F-indirect-49 returns, the function attempts one 16-bit store of numeric value 205 at that address. No mapping, permission, null-address, or alignment check is represented.

The address may be odd. Store success, fault behavior, and byte order remain unknown. The representation ends after this store at the bad-instruction marker.

## Side effects

### Direct effects

The represented direct-effect inventory is:

- **49 indirect-call occurrences**.
- **One call to F-3de949054bdd**, making **50 calls total**.
- **Ten coprocessor-transfer occurrences**, T1 through T10.
- **One final 16-bit store**, if reached and completed.

The bad-instruction endpoint is evidence of truncated control-flow interpretation, not proof of a particular runtime trap, halt, or exception.

Transfers remain distinct observable operations even when their data operands simplify to zero. No equivalence to omitting, combining, or reordering them is established.

### Memory, ownership, and preservation

Apart from the final store, no direct ordinary data-memory read or write is explicitly represented. This does not exclude instruction fetches, address-resolution effects, or arbitrary memory operations inside dependencies.

No allocation, release, ownership transfer, borrowing convention, reference counting, or cleanup obligation is established. Words forwarded between dependencies must not be classified as pointers or resources without additional evidence.

The final address has no established allocation provenance. Its small numeric range does not prove that it denotes RAM, unmapped memory, or a device register.

No dependency is assumed to preserve globals, memory, processor state, coprocessor state, or physical registers. The stated reuse of values follows the supplied dataflow, not an independently established preservation contract.

## Dependencies

F-3de949054bdd is the only dependency with a supplied stable opaque identity. Its behavior and side effects are unknown.

F-indirect-01 through F-indirect-49 identify unresolved call occurrences. Their identities do not establish distinct targets, valid entry points, complete argument lists, correct return conventions, purity, ownership, or error semantics.

The ten coprocessor operations are architecture-dependent dependencies. Their complete represented operand lists and order are documented above; their architectural interpretation remains unknown.

Reaching the final store requires every earlier call to return and every earlier transfer to complete as represented. No recovery path is shown for nonreturning calls, invalid destinations, transfer faults, or store faults.

## Acceptance examples

These are conditional static acceptance cases, not observed executions.

1. **Input forwarding.** With explicit input 0x12345678, F-indirect-01 receives 0x12345678 unchanged, provided its destination is callable.

2. **Seed truncation and sign extension.** Under 32-bit arithmetic-right-shift semantics, seed 0x04000000 becomes 0x80000000 after the left shift by 5, then 0xFC000000 after the signed right shift by 5. F-indirect-12 therefore does not receive the original seed as its second argument.

3. **Consecutive transfer distinction.** If F-indirect-16 returns 0x12345678, T2 and T3 both transfer that data with leading fields 0, 3, 7. T2 uses Control C as its fifth operand; T3 uses Control B. Both use Control B as their sixth operand. Neither operation may be omitted merely because the data matches.

4. **Selector truncation.** With selector equal to 1, F-indirect-18’s third argument is 0x08000000. T5’s data and F-indirect-28’s first argument are zero. F-indirect-33’s second argument is also zero.

5. **Payload scaling.** With payload 0x12345678, scaled payload is 0x23456780. T10 receives 0x45678000. The derived temporary is zero, making the temporary-derived arguments to F-indirect-40 and F-indirect-48 zero.

6. **Minimum address.** If bits 52–63 of F-indirect-45’s result are zero, the attempted final store targets address zero. Success is not guaranteed.

7. **Unaligned address.** If those bits encode 1, the address is 1. No alignment adjustment occurs; an architecture-dependent alignment fault is possible.

8. **Maximum address and width.** If those bits encode 4095, the address is 5118. The attempted write remains 16 bits wide and contains numeric value 205.

9. **Interrupted execution.** If F-indirect-24 does not return, T4 and all later operations are not reached through the represented path. No fallback or cleanup path is established.

## Uncertainties

The following remain **documented_only**:

- Correctness of the instruction interpretation, which explicitly reports bad instruction data.
- Actual indirect targets and complete calling conventions.
- Address arithmetic and execution-state rules for indirect destinations.
- Correspondence between represented signed shifts and real machine operations.
- Whether the double-word results accurately model the target ABI.
- Architectural meanings and validity of all coprocessor selector/control operands.
- Whether incoming control operands identify registers, encode selectors, or have another architecture-specific interpretation.
- Effects of dependencies on memory, globals, resources, and processor state.
- Memory byte order, address mapping, permissions, and alignment behavior.
- Whether the final store completes.
- Actual execution beyond the represented endpoint.
- Application-level purpose, success criteria, and error-reporting conventions.

No retry, rollback, exception handler, cleanup procedure, or normal-return guarantee is established.

## Evidence

This contract is based solely on the supplied static representation: the explicit input, incoming-state dependencies, 50-call sequence, ten complete coprocessor operand lists, integer dataflow, final 16-bit store, and bad-instruction endpoint.

Zero-valued intermediates, sign-extension effects, and the final address formula are conditional deductions from the represented widths and machine-word arithmetic assumptions.

No implementation or dependency was executed or independently inspected. No runtime tests have been performed.
