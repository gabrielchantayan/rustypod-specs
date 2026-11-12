<!-- Trusted provenance: F-a38ce80eadab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a38ce80eadab

## Purpose

F-a38ce80eadab specifies a represented execution path containing 54 dependency invocations, ten coprocessor-directed operations, and a final 16-bit memory store. The supplied representation ends at an invalid-instruction boundary, not an ordinary return.

No application-level purpose is established. In particular, the coprocessor operations do not establish hardware initialization, cache maintenance, or a device protocol.

This contract distinguishes three kinds of claims:

- **Established static evidence:** explicit operand relationships, operation order, result-width usage, and the final store.
- **Inferred machine interpretation:** arithmetic deductions using 32-bit words and 64-bit result containers.
- **Unknown behavior:** dependency effects, execution validity, architectural interpretation, and termination details not established by the supplied material.

The supplied representation warns of invalid instruction data. This is therefore a contract for the represented path, conditional on that path being executable, rather than proof that an actual execution follows it.

## Interface

### Explicit inputs

| Position | Width and interpretation | Represented use |
|---|---|---|
| First | 32-bit value | Passed unchanged to the first dependency invocation. |
| Second | 32-bit value | No explicit use in this body. |
| Third | Signed 32-bit interpretation | Supplies separate left-shift expressions by 22 and 19 bits. |

The absence of an explicit use of the second input does not establish that dependencies cannot observe its incoming register value.

### Ambient inputs

The represented interface also requires state outside the three explicit inputs:

| Identifier | Represented value |
|---|---|
| A-L | Incoming link-state value used to derive the first indirect target. |
| A-G | Incoming 32-bit general-purpose value, used in left shifts by 14 and 8. |
| A-X | Additional 32-bit register value passed as the second explicit operand of occurrence 05. Its producer is unresolved. |
| A-C0 | 32-bit ambient coprocessor-related operand shared as the final operand of all ten coprocessor operations. |
| A-C1 | Distinct 32-bit ambient coprocessor-related operand used after occurrence 19. |
| A-C2 | Distinct 32-bit ambient coprocessor-related operand used by the first operation after occurrence 21. |
| A-C3 | Distinct 32-bit ambient coprocessor-related operand used after occurrence 29. |
| A-C4 | Distinct 32-bit ambient coprocessor-related operand used between occurrences 32 and 33. |
| A-C5 | 32-bit ambient coprocessor-related operand shared by operations after occurrences 35, 40, and 41. |
| A-C6 | Distinct 32-bit ambient coprocessor-related operand used after occurrence 37. |

“Distinct” identifies different represented input bindings; it does not require numerically unequal values.

A-C0 also supplies the **penultimate** operand of the second operation after occurrence 21 and the second operation after occurrence 40. Thus those two operations each receive A-C0 twice.

These identifiers preserve explicit operand relationships. They do not establish what the operands mean architecturally, whether they designate hardware registers, or whether dependencies preserve underlying coprocessor state.

### Output and termination

No ordinary return value is represented.

The final explicit ordinary-memory operation stores the 16-bit value `0x00CD` at a derived numeric address. An invalid-instruction termination marker follows. The marker does not establish an exception class, exit status, recovery path, or hardware halt mechanism.

### Arithmetic conventions

For the arithmetic relationships below:

- A 32-bit left shift retains the low 32 bits of the shifted bit pattern.
- A 64-bit result has a low half consisting of bits 0–31 and a high half consisting of bits 32–63.
- Right shifts of unsigned values are logical.
- The right shift used in occurrence 17 is represented with a signed 32-bit interpretation. Arithmetic sign extension is discussed conditionally rather than assumed universally.

These conventions express the apparent machine-value relationships. They do not claim that every corresponding signed expression has defined behavior under high-level-language arithmetic rules.

## Behavior

### Dependency occurrence identifiers

F-U01 through F-U54 identify dependency **occurrences**, not established function identities. F-U12 is omitted because occurrence 12 has the supplied identity F-3de949054bdd.

Different F-U identifiers may resolve to the same target. “None shown” means no explicit arguments are represented; it does not establish an empty machine-level argument list.

“Result 08,” for example, denotes the value retained from occurrence 08. This is document-local notation, not an implementation identifier.

The following order is unconditional within the represented body. There are no represented branches or loops. Later operations nevertheless require preceding calls and machine operations to permit continuation.

### Initial dependency sequence

| Occurrence | Dependency | Explicit operands and retained result |
|---:|---|---|
| 01 | F-U01 | First input. Target derived from A-L plus `0xEB6`. |
| 02 | F-U02 | A-G shifted left 14. |
| 03 | F-U03 | None shown. |
| 04 | F-U04 | None shown. |
| 05 | F-U05 | A-G shifted left 8; A-X; zero. |
| 06 | F-U06 | None shown. |
| 07 | F-U07 | None shown. |
| 08 | F-U08 | None shown; retain a 32-bit result. |
| 09 | F-U09 | Result 08; third input shifted left 22; result 08 shifted left 5. |
| 10 | F-U10 | None shown; retain a 32-bit result. |
| 11 | F-U11 | Result 10. |
| 12 | F-3de949054bdd | None shown. |
| 13 | F-U13 | None shown; retain a 32-bit result. |
| 14 | F-U14 | Result 13. |
| 15 | F-U15 | None shown; retain a 32-bit result. |
| 16 | F-U16 | Result 15. |
| 17 | F-U17 | Third input shifted left 19; signed right shift by 5 of the retained word “result 08 shifted left 5.” Retain a 32-bit result. |
| 18 | F-U18 | Result 17. |
| 19 | F-U19 | None shown; retain a 64-bit result. |
| 20 | F-U20 | High half of result 19 shifted left 1. |
| 21 | F-U21 | None shown; retain a 32-bit result. |

A coprocessor operation occurs between occurrences 19 and 20. Two more occur between occurrences 21 and 22. Their complete operands appear in the coprocessor schedule below.

The second operand of occurrence 17 must not generally be replaced with result 08. The initial left shift loses five high bits. Under arithmetic-right-shift semantics, the subsequent shift sign-extends the remaining 27-bit value.

### Middle dependency sequence

| Occurrence | Dependency | Explicit operands and retained result |
|---:|---|---|
| 22 | F-U22 | None shown; retain a 32-bit result. |
| 23 | F-U23 | Zero; zero; result 22 shifted left 27; zero. |
| 24 | F-U24 | None shown. |
| 25 | F-U25 | None shown. |
| 26 | F-U26 | None shown. |
| 27 | F-U27 | None shown. |
| 28 | F-U28 | None shown. |
| 29 | F-U29 | None shown; retain a 32-bit result. |
| 30 | F-U30 | None shown. |
| 31 | F-U31 | None shown. |
| 32 | F-U32 | None shown. |
| 33 | F-U33 | The retained result-22-derived word shifted left another 24; zero. |
| 34 | F-U34 | None shown. |
| 35 | F-U35 | None shown; retain a 32-bit result. |
| 36 | F-U36 | None shown. |
| 37 | F-U37 | Zero; zero; zero. Retain a 32-bit result. |
| 38 | F-U38 | Result 37 shifted left 1; zero; result 37 shifted left 4; zero. |
| 39 | F-U39 | None shown. |
| 40 | F-U40 | None shown. |

The result-22-derived word is the 32-bit value obtained by shifting result 22 left 27. Its only potentially set bits are positions 27–31. Consequently:

- Shifting that word left another 24 yields zero. This supplies both the coprocessor data between occurrences 32 and 33 and occurrence 33’s first operand.
- Masking that word with `0x007FFFFF` yields zero. Shifting the masked value left 4 therefore gives occurrence 38’s zero second operand.

A separate retained value is explicitly set to zero after occurrence 22. Its shifts by 22 and 11 supply the first two operands of occurrence 37; the third operand is an explicit zero. No represented assignment changes that retained value before occurrence 37.

After occurrence 40, the first of two coprocessor operations occurs. Between those two operations, an intermediate value is formed by taking the low two bits of “result 37 shifted left 4” and shifting them left 16. The low two bits are necessarily zero, so this intermediate and its later shifted forms are zero.

These reductions concern represented local dataflow. They do not depend on dependencies returning zero or preserving globals.

### Final dependency sequence

| Occurrence | Dependency | Explicit operands and retained result |
|---:|---|---|
| 41 | F-U41 | Zero; retain a 32-bit result. |
| 42 | F-U42 | Result 37 shifted left 12; result 37 shifted left 16; result 41 shifted left 9. |
| 43 | F-U43 | None shown; retain a 64-bit result. |
| 44 | F-U44 | Low half of result 43; high half of result 43; low half shifted left 1. |
| 45 | F-U45 | Zero; retain a 32-bit result. |
| 46 | F-U46 | Result 45; retain a 32-bit result. |
| 47 | F-U47 | Result 46. |
| 48 | F-U48 | None shown. |
| 49 | F-U49 | None shown; retain a 64-bit result. |
| 50 | F-U50 | Low half of result 49; high half of result 49; high half shifted left 11. Retain a 64-bit result. |
| 51 | F-U51 | Low half of result 50. |
| 52 | F-U52 | None shown; retain a 32-bit result. |
| 53 | F-U53 | Result 52; zero. |
| 54 | F-U54 | None shown. |

The zero operands at occurrences 45 and 53 derive from the masked intermediate formed between the two coprocessor operations after occurrence 40. They do not depend on dependency return values.

The coprocessor operation after occurrence 41 precedes occurrence 42.

### Complete coprocessor schedule

Each row specifies the selector tuple, data operand, and **both trailing ambient operands**, in represented argument order.

| Operation | Position | Selector tuple | 32-bit data | Penultimate operand | Final operand |
|---:|---|---|---|---|---|
| 1 | After 19, before 20 | `(0, 6, 7)` | Low half of result 19 | A-C1 | A-C0 |
| 2 | After 21, first | `(0, 3, 7)` | Result 21 | A-C2 | A-C0 |
| 3 | After 21, second | `(0, 3, 7)` | Result 21 | A-C0 | A-C0 |
| 4 | After 29, before 30 | `(0, 1, 7)` | Result 29 | A-C3 | A-C0 |
| 5 | After 32, before 33 | `(0, 5, 7)` | Result 22 shifted left 27, then left 24; necessarily zero | A-C4 | A-C0 |
| 6 | After 35, before 36 | `(0, 6, 7)` | Result 35 | A-C5 | A-C0 |
| 7 | After 37, before 38 | `(0, 5, 7)` | Result 37 shifted left 1 | A-C6 | A-C0 |
| 8 | After 40, first | `(0, 7, 7)` | Zero | A-C5 | A-C0 |
| 9 | After 40, second | `(0, 0, 7)` | Zero | A-C0 | A-C0 |
| 10 | After 41, before 42 | `(0, 2, 7)` | Result 37 shifted left 12 | A-C5 | A-C0 |

Operations 2 and 3 remain separate even though their selector tuples and data operands match. Their penultimate input bindings differ. Operations 8 and 9 likewise remain separate; their selector tuples and penultimate bindings differ.

A-C5 is reused in operations 6, 8, and 10. A-C0 is the final operand in every row and also the penultimate operand in operations 3 and 9. A-C1, A-C2, A-C3, A-C4, and A-C6 each occupy the penultimate position in one operation.

### Final memory write

After occurrence 50, the represented body retains an unsigned value obtained by shifting its 64-bit result right 41 bits and converting to 32 bits. This value has at most 23 significant bits. Its subsequent signed 32-bit interpretation is therefore nonnegative.

After occurrence 54, shifting that retained value right another 11 bits obtains the top 12 bits of result 50. Let that unsigned integer be \(n\). The final destination is:

\[
n+\left\lfloor n/4\right\rfloor,\qquad 0\le n\le4095.
\]

The destination range is therefore 0 through 5118 inclusive. The addition cannot overflow a 32-bit value.

The function stores `0x00CD` as a 16-bit value at that address, then reaches the invalid-instruction boundary. There is no explicit base address, allocation, bounds check, null check, or alignment adjustment.

## Side effects

The represented body contains:

- **54 dependency invocation occurrences**, including F-3de949054bdd.
- **Ten coprocessor-directed operations**, with the complete operand schedule above.
- **One explicit ordinary data-memory store**, of width 16 bits.
- **No explicit ordinary data-memory load**.

Dependency execution may add memory reads, writes, allocation, release, blocking, exceptions, or hardware effects. No bounds on those effects are established.

### Memory layout and ownership

The store width and numeric value are established; byte order is not. The ordering of the bytes `0x00` and `0xCD` remains unknown.

The destination need not be even. A top-12-bit value of 1 produces address 1. Whether an unaligned 16-bit store succeeds, traps, or behaves otherwise is unresolved.

No buffer object, capacity, lifetime, ownership transfer, or release obligation is represented. The numeric destination is not evidence of allocated or writable memory.

### State changes and errors

No dependency is known to preserve globals, ambient machine state, or coprocessor state. No dependency is known to own, retain, or release memory.

There are no represented error-result checks or recovery branches. An invalid indirect target, nonreturning dependency, coprocessor exception, or inaccessible destination can prevent later events. The final store is not guaranteed merely because the function was entered.

## Dependencies

F-3de949054bdd is the only dependency with a supplied cross-document identity. It is invoked at occurrence 12, with no explicit arguments shown. Its result, state effects, ownership behavior, and termination behavior remain unknown.

The other 53 occurrences are unresolved indirect invocations. Their occurrence identifiers do not imply distinct targets. Most targets are derived from represented address constants or address anchors; the first additionally depends on A-L.

Executable mapping, instruction validity, and compatible calling conventions are prerequisites for an actual trace matching this contract. Target resolution and address-overflow behavior are not independently established.

The representation uses results 19, 43, 49, and 50 as 64-bit containers, with subsequent extraction of 32-bit halves. Actual return-register assignment and ABI details remain unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Each assumes the represented path is executable and earlier operations permit continuation.

1. **Unused explicit input:** Changing only the second explicit input does not change an explicit operand expression in this body. It need not preserve overall behavior if a dependency observes the incoming register.

2. **Lost high bits:** If result 08 is `0x08000000`, shifting it left 5 in 32 bits gives zero. Occurrence 09 receives zero as its third operand, and occurrence 17 receives zero as its second operand.

3. **Signed-shift distinction:** If result 08 is `0x04000000`, its left shift by 5 gives `0x80000000`. Under arithmetic-right-shift semantics, occurrence 17’s second operand is `0xFC000000`, not the original result.

4. **Two operations after occurrence 21:** If A-C2 differs numerically from A-C0, operations 2 and 3 receive different penultimate values. Both receive result 21 as data and A-C0 as the final operand. If A-C2 equals A-C0, two operation occurrences are still represented.

5. **Shared ambient inputs:** Changing only A-C5 changes the penultimate operand expression of operations 6, 8, and 10. Changing only A-C0 changes every final operand and additionally both trailing operands of operations 3 and 9. These are operand relationships, not guarantees about downstream effects.

6. **Unconditional derived zeros:** Every 32-bit result 22 gives zero for operation 5’s data, occurrence 33’s first operand, and occurrence 38’s second operand. Occurrence 37 receives three zeros independently of results 22–36.

7. **Final address boundaries and alignment:** Top-12-bit values 0, 1, and 4095 produce addresses 0, 1, and 5118 respectively. A successful store still requires accessible memory and acceptable alignment behavior.

8. **Low-bit independence:** Changing only the low 52 bits of result 50 leaves the final destination unchanged. It can change the low-half operand passed to occurrence 51 and therefore does not imply unchanged dependency effects.

9. **Early termination:** If F-3de949054bdd does not return, occurrences 13–54, all later coprocessor operations, and the final store are not required to occur.

## Uncertainties

The following remain **documented_only**:

- Whether the represented path corresponds to valid executable instructions.
- Actual indirect targets, target aliasing, address arithmetic, and instruction-set requirements.
- Complete calling conventions, implicit inputs, and register-preservation rules.
- The producer and timing of A-X.
- Architectural signed-right-shift behavior at occurrence 17 and the validity of recovered signed-arithmetic notation.
- The meanings, privilege requirements, and effects of coprocessor operations and their ambient operands.
- Dependency mutations, ownership behavior, errors, and termination.
- Byte order, address mapping, alignment handling, and final-store accessibility.
- The actual mechanism and outcome of the terminal invalid instruction.

The ambient-operand identities and reuse documented above are established relationships, not uncertainties. Their architectural meanings remain unresolved.

## Evidence

The supplied static representation establishes the dependency occurrence order, explicit operand relationships, ten coprocessor-operation occurrences, their trailing ambient-operand identities and reuse, retained result widths, and the final 16-bit store. It also contains an invalid-instruction warning and terminal boundary.

The zero reductions, shift combinations, final destination formula, and destination bounds are arithmetic deductions under the stated machine-width interpretation. They are not observations of execution.

No runtime tests have been performed.
