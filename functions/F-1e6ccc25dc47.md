<!-- Trusted provenance: F-1e6ccc25dc47; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1e6ccc25dc47

## Purpose

The supplied static representation establishes a sequence of computed-address reads, writes, and two possible invalid-instruction endpoints. It does not establish an application-level purpose, a successful-return path, or an ordinary error-reporting interface.

The routine derives addresses and data from its second and fourth arguments, additional entry state, dependency-associated expressions, and later memory reads. It performs writes before testing its only displayed branch condition. If that condition permits continuation, further writes occur before the final invalid-instruction endpoint.

**Evidence boundary:** the representation reports invalid instruction data and overlapping named storage. It also contains shifts by 32 of values presented as 32-bit integers. The contract therefore distinguishes established expression structure and access order from unresolved execution semantics. It does not claim that the representation is a valid executable reconstruction.

## Interface

### Explicit arguments

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits, otherwise untyped | No visible use. |
| Second | Signed 32-bit integer | Arithmetic source, stored word, and basis for a halfword-access address. |
| Third | 32 bits, otherwise untyped | No visible use. |
| Fourth | Signed 32-bit integer | Basis for an initial byte-write address and an unresolved pointer expression. |

No validation, ownership, or configuration meaning is established for the unused arguments.

### Additional inputs

The represented body also depends on:

- An unsigned 32-bit register-associated entry value.
- A pointer associated with an entry-stack slot at displacement `0x250`.
- A named integer read only on the continuing path.
- Three separate evaluations of an expression associated with F-e47d7b2e9c88.
- Two separate evaluations of an expression associated with F-8afdaab91693.
- Named-storage reads performed after earlier writes, including a byte read, scaled-pointer reads, and a final upper-halfword read.

The initialization, calling-convention roles, and storage provenance of these inputs are not established.

### Output and termination

The represented return type carries no value. Neither displayed path returns normally:

- The negative branch reaches an invalid-instruction endpoint after earlier writes.
- The continuing branch reaches another invalid-instruction endpoint after the remaining operations.

These endpoints are not established process exits, portable exceptions, error codes, or recoverable failures. An earlier memory fault could prevent either endpoint from being reached.

### Widths and arithmetic

Explicit indirect accesses use 8-bit bytes, 16-bit halfwords, and 32-bit words. Pointer expressions are presented in a 32-bit integer model.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). Narrowing retains the corresponding low bits. Signed overflow, signed left shifts, and shifts by 32 remain unresolved where relevant. Arithmetic-right-shift interpretations are stated conditionally rather than assumed universally.

Named destinations lack independent declarations in the excerpt. A narrowed or concatenated expression establishes the expression’s width, not necessarily the destination’s physical write width.

No general address-validity, bounds, alignment, or permission checks are shown. Reaching an operation requires all preceding operations to complete.

## Behavior

The order below is the order represented by the supplied body. It does not establish hardware bus ordering, atomicity, or behavior after a fault. Addresses can alias unless supporting evidence excludes that possibility.

### Initial derivation

The second argument produces three related quantities:

- An unsigned difference from `0xA8`.
- That difference with bit 31 cleared.
- An index equal to the masked difference shifted right by one.

The masked difference ranges from zero through `0x7FFFFFFF`; the index ranges from zero through `0x3FFFFFFF`.

The halfword-access base is twice the **unmasked** difference, with unsigned 32-bit wrapping. The unmasked and masked intermediate differences can differ. Nevertheless, doubling either produces exactly the same wrapped address: clearing bit 31 removes either zero or `0x80000000`, whose doubled contribution is zero modulo \(2^{32}\).

A separate word-write pointer is derived from the fourth argument by a left shift of two followed by a right shift of 32. Its result is unresolved. It must not be replaced with zero, the original argument, or a target-specific masked-shift result.

### Initial memory effects

The following accesses occur in order:

1. A byte write stores the index’s low byte at the fourth argument plus `0x0897E8B8`.
2. A named destination receives the expression `0xB8`.
3. A byte write stores the register-associated value’s low byte at that value plus six.
4. A word write through the unresolved pointer stores the second argument.
5. Halfword writes store `0xFFFF` at offsets four and six from the halfword-access base, in that order.
6. A byte write stores `0xB8` at offset four from the unresolved pointer.
7. A halfword write stores `0xFFFF` at offset two from the halfword-access base.
8. A named destination receives the low 16 bits of that base.
9. An unsigned halfword is read at the base itself. Its zero-extended value becomes the destination address for a word write containing the register-associated value masked with `0x897E8B80`.

The three explicit halfword writes do not directly initialize the halfword at offset zero. Other earlier writes could affect that read through aliasing.

The representation does not establish whether these addresses designate allocated objects, mapped RAM, device registers, or inaccessible storage.

### First dependency-associated expression and branch

There are **three distinct evaluations** associated with F-e47d7b2e9c88, in this order:

1. A 32-bit evaluation is captured for the later sign test, self-addressed word writes, and selector calculation.
2. A separate evaluation, interpreted as signed, is shifted right by 32 to derive a base.
3. A third evaluation, interpreted as signed, is shifted right by 31 and narrowed to a halfword.

The second evaluation’s base derivation is unresolved because of the shift width. Under conventional signed arithmetic-right-shift semantics, the third evaluation produces halfword `0x0000` when nonnegative and `0xFFFF` when negative.

No memory write is displayed between these three evaluations. Nevertheless, their equality is not established by the excerpt: the associated symbol’s meaning and evaluation semantics are unresolved. Equality is conditional on immutable-address semantics or other evidence ensuring stable values.

The routine writes the third evaluation’s derived halfword at base offset four. It then tests the **first evaluation’s** sign.

If that first value is negative, execution reaches the first invalid-instruction endpoint. The preceding write is not rolled back.

If the first value is nonnegative, the routine performs:

1. A halfword write of the third evaluation’s derived halfword at base offset `0x12`.
2. A word write of the first value to the address numerically equal to that first value.
3. A signed-halfword read at the base plus the index, using the index as a byte displacement.
4. A repeat of the halfword write at offset `0x12`.
5. A repeat of the self-addressed word write.
6. A word write at base offset eight containing the signed-halfword read, sign-extended and shifted right by 16.

Under conventional arithmetic-right-shift semantics, the last word is zero for a nonnegative halfword and `0xFFFFFFFF` for a negative halfword.

The repeated writes remain observable operations. The sign of the first dependency evaluation does not establish the sign-derived halfword unless the first and third evaluations are known to agree.

### Selector and entry-stack-pointer operations

On the continuing path, a named destination first receives zero.

A selector is then formed from bit 2 of the **first** F-e47d7b2e9c88 evaluation: the result is `0x58` when that bit is clear and `0x5C` when set.

The following operations occur in order:

1. A word write stores the unresolved base at address `0x5C` or `0x60`, respectively.
2. A named destination receives the masked difference shifted right by 17 and narrowed to 16 bits.
3. Another named destination receives a 16-bit expression containing the index’s low byte.
4. A word write stores the index at address `0x170` or `0x180`, respectively.
5. A further named destination receives the index.

The entry-stack pointer then determines these ordered accesses:

- A halfword write of the selector at offset zero.
- A word write of the unresolved base shifted left by eight at offset zero.
- A word write of the same shifted expression at offset 20.
- A word write of the pointer’s numeric value at offset zero.
- A word read at twice the pointer’s numeric value.
- A halfword write of that read word’s low 16 bits at pointer offset zero.
- A word write of that read word to the address numerically equal to the word.

The left-shifted signed expression has unresolved overflow semantics. The overlapping writes at pointer offset zero are separate operations, not a guaranteed final record layout.

Next, a word write stores the pointer’s sign indicator at an address formed from the **address of its entry-stack slot**, multiplied by `0x20`, plus `0x18`. This destination depends on the slot address, not on the pointer stored in the slot.

A named aggregate then receives a 24-bit concatenation: the 16-bit value `0x0005` above a **freshly read named byte**. This byte read occurs here, after the preceding indirect writes. Although an earlier named assignment used the index’s low byte, the concatenation must not be specified as reusing a cached copy of that index byte. Overlap may affect the byte actually read.

A subsequent word write stores the entry-stack pointer at an address formed from four times its numeric value plus `0x18`.

### Later reads and scaled-pointer storage

Only now is the separate named integer read and captured. Its value is not an entry-time snapshot; earlier writes may have changed its storage.

The next dependency and memory operations have this order:

1. A first expression associated with F-8afdaab91693 is evaluated and captured.
2. A named scaled-pointer destination receives four times the entry-stack pointer.
3. A second expression associated with F-8afdaab91693 is evaluated and shifted left by two.
4. The named scaled-pointer storage is read again to form a self-addressed word write.
5. That named storage is read again to form the destination four bytes beyond it, where the index is written.

The two dependency evaluations are separated by a memory write. They must not be collapsed into one captured dependency value. If the expression denotes mutable storage overlapping the named destination, the two evaluations can differ. Equality requires immutable-address semantics or other stability evidence.

The self-addressed write obtains both its destination basis and stored value from named scaled-pointer storage, rather than exclusively from a protected computed local. The excerpt does not establish an intra-expression ordering for those reads. The following index write uses a subsequent named-storage read, which may observe a change caused by the preceding write.

The continuing path then:

- Writes four times the captured later integer as a word to the address numerically equal to that product.
- Writes the **second** F-8afdaab91693 evaluation shifted left by two as a word at an address formed by adding the captured integer to its own left-shift-by-six, arithmetic-right-shift-by-17 expression.
- Assigns the captured integer’s upper 16 bits to named upper-halfword storage.
- Assigns zero to another named destination.

Signed multiplication, addition, and left-shift overflow in these expressions are unresolved.

### Final indirect and named writes

The **first** captured F-8afdaab91693 evaluation, multiplied by eight, provides the next indirect base. At that base, the routine writes:

1. A zero word at offset zero.
2. A zero word at offset four.
3. A byte at offset 16 containing the base’s low byte plus eight, narrowed to a byte.

Final named-destination assignments then occur in this order:

| Order | Assigned expression |
|---:|---|
| 1 | `0x897EA700` |
| 2 | `0x66` |
| 3 | `0xEAEC` |
| 4 | `0x1A` |
| 5 | The destination location’s own address |
| 6 | A 32-bit concatenation of a freshly read named upper halfword above `0xEA70` |
| 7 | `0x5C` |

The sixth assignment **reads the named upper-halfword storage at this point**. It does not directly reuse the upper halfword of the captured later integer. The intervening named and indirect writes may overlap that storage and change the value read.

The concatenation expression is 32 bits wide. The excerpt does not independently establish its named destination’s width; the same destination-width limitation applies to the other final named assignments.

Execution then reaches the final invalid-instruction endpoint.

## Side effects

The routine performs extensive indirect and named-storage writes, including low-address writes, repeated writes, and overlapping writes. Read values can determine later addresses and data.

Established constraints:

- No allocation, deallocation, reference-count operation, ownership transfer, or cleanup is shown.
- No ownership of accessed storage is established.
- Named-storage overlap is explicitly warned about; exact overlap relationships are not supplied.
- Later reads must be respected as reads at their displayed positions, not replaced with assumed earlier values.
- Separate dependency-associated evaluations are not guaranteed equal.
- Earlier writes are not rolled back at either displayed terminal event.
- No synchronization, atomicity, or fault-recovery mechanism is shown.

Memory mapping, byte order, alignment rules, device effects, and actual fault behavior remain unresolved.

## Dependencies

| Opaque ID | Established relationship |
|---|---|
| F-e47d7b2e9c88 | Three separate expression evaluations supply, respectively, a captured word, an unresolved base derivation, and a sign-derived halfword. No invocation is shown. |
| F-8afdaab91693 | Two separate evaluations occur around an intervening named write. The first supplies the later eightfold address basis; the second supplies the left-shifted stored word. No invocation is shown. |

The excerpt does not establish whether these expressions denote immutable function addresses, mutable data, or artifacts of incorrect instruction interpretation.

No dependency calling contract is established. No assumption is made that either dependency preserves globals, owns memory, or releases memory.

## Acceptance examples

These are conditional static criteria, not observed executions. Each requires preceding operations to complete and any relevant unresolved arithmetic to admit an interpretation.

1. **Second argument `0xA8`.**  
   Both differences, the index, and the doubled base are zero. The first byte-write value is zero. Halfword writes target addresses four, six, and two, followed by a halfword read at zero.

2. **Second argument `0xA7`.**  
   The unsigned difference is `0xFFFFFFFF`, the masked difference is `0x7FFFFFFF`, and the index is `0x3FFFFFFF`. Doubling either difference modulo \(2^{32}\) gives `0xFFFFFFFE`. The first byte-write value is `0xFF`.

3. **First F-e47d7b2e9c88 evaluation negative.**  
   The base-offset-four halfword write occurs before termination. Its data comes from the third evaluation, not necessarily from the first value’s sign. The continuing-path writes do not occur.

4. **First evaluation nonnegative; third evaluation negative.**  
   If the separate evaluations can differ, continuation is permitted while the sign-derived halfword is `0xFFFF` under conventional arithmetic-right-shift semantics. Nonnegative branch selection alone does not require zero halfword data.

5. **Selector bit clear or set.**  
   A clear bit 2 in the first evaluation selects `0x58`, with low-address word destinations `0x5C` and `0x170`. A set bit selects `0x5C`, with destinations `0x60` and `0x180`.

6. **Negative signed-halfword read.**  
   Under conventional sign extension and arithmetic right shift, the subsequent base-offset-eight word is `0xFFFFFFFF`; for a nonnegative halfword it is zero.

7. **Dependency expression denotes an immutable address.**  
   Stable immutable-address semantics can justify equality of its separate evaluations. Without that premise, the contract retains them separately, particularly across the intervening scaled-pointer assignment.

8. **Scaled-pointer storage changes after assignment.**  
   Subsequent indirect destinations follow the values read from that named storage at their use points, not an assumed cached fourfold pointer.

9. **Upper-halfword storage is overwritten before final concatenation.**  
   The final concatenation uses the upper-halfword value read after those writes. It need not match the captured integer’s original upper halfword.

## Uncertainties

The following remain **documented_only**:

- Application-level intent and correctness of instruction boundaries.
- Semantics of shifts by 32 and problematic signed arithmetic.
- Provenance and initialization of extra entry state.
- Meaning, stability, and equality of dependency-associated evaluations.
- Named destination widths and exact storage-overlap relationships.
- Intra-expression evaluation order not explicitly established by the representation.
- Byte order, alignment, memory permissions, and device semantics.
- Whether faults interrupt the displayed sequence.
- Any normal completion behavior outside the supplied representation.

An initialization, corruption, or resource-management interpretation is not an established acceptance requirement.

## Evidence

Established static evidence is limited to the supplied representation: argument widths, explicit access widths, arithmetic expressions, separate dependency references, named-storage read points, displayed operation order, branch condition, and invalid-instruction endpoints.

The modular equivalence of the two doubled differences follows from the stated unsigned 32-bit arithmetic model. Conditional sign-extension results require the stated arithmetic-right-shift interpretation.

Invalid-instruction and overlapping-storage warnings limit executable reconstruction. No runtime tests, dependency execution, independent implementation access, or binary validation have been performed.
