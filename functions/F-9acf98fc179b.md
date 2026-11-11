<!-- Trusted provenance: F-9acf98fc179b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9acf98fc179b

## Purpose

F-9acf98fc179b prepares a decimal-looking field and submits its characters individually to F-bcb8529decd8. Its inputs comprise an opaque output context, a two-word numeric payload, a field width, a precision request, and formatting flags.

**Established static behavior:** the function normalizes precision, invokes numeric dependencies, extracts bytes through indexed table reads, calculates padding, and requests character emissions in a defined order. It also reads global operands at specific dependency-call boundaries.

**Inferred intent:** fixed-point numeric formatting with optional sign, alignment, and zero padding. The available representation does not establish a complete numerical contract. In particular, dependency semantics, some argument lists, and mutable global data remain unresolved. Conventional decimal formatting must not be substituted for the visible behavior.

## Interface

The function accepts nine 32-bit argument words and has no explicit return value.

| Position | Contract at this boundary |
|---|---|
| 1–4 | Opaque output-context words, copied locally and supplied with every character-emission request. |
| 5–6 | Two-word numeric payload, supplied to numeric dependencies. |
| 7 | Field-width word, participating in 32-bit padding arithmetic. |
| 8 | Precision-request word, initially interpreted as signed for the negative-value test. |
| 9 | Formatting-flags word. |

The payload occupies 64 bits across two words. Its mathematical type, encoding, and external word-order convention are not established independently of the unavailable dependencies.

The recognized flags are:

| Mask | Established effect |
|---|---|
| 0x01 | Changes positive leading padding into trailing spaces. |
| 0x02 | Selects a plus sign. |
| 0x04 | Selects a space sign if 0x02 is absent. |
| 0x08 | Enables decimal-point and stored fractional-byte emission when normalized precision is zero. |
| 0x10 | Selects leading zero padding instead of leading spaces. |

Other flag bits have no direct effect in the visible body.

### Preconditions and ownership

The output context must be acceptable to F-bcb8529decd8. This function does not validate it, accept an explicit output capacity, or check output status.

Numeric dependencies must accept the arguments and intermediates presented along the selected path. Each digit-extraction result used as a table index must identify readable memory relative to the selected table base. There is no index validation.

Temporary digit bytes are held in local buffers. The output dependency receives character values, not pointers to these buffers. No explicit allocation, release, or ownership transfer appears. Dependencies may nevertheless have ownership or lifetime effects; their contracts are unavailable.

## Behavior

### Entry processing and early global reads

A precision word with its sign bit set is replaced by six. Consequently, words from 0x80000000 through 0xffffffff take the default-precision path, not the large-positive-precision path.

Before numeric conversion, the function makes two calls to F-11ed8242c40c:

1. It reads two global operand locations and supplies their values alongside the original payload.
2. After the first call returns, it reads those same two locations again and supplies the newly read values alongside the original payload.

These are separate reads, not reuse of a captured operand pair. The first call may modify either location, directly or indirectly. Equality of the operands passed to the two calls is therefore not established. The relative evaluation order of the two reads within either individual call is not established.

Neither call’s returned value is visibly consumed.

The visible local sign state remains affirmative. The conditional payload sign-bit change is not taken, and the function’s own sign selection never chooses a minus sign:

- Flag 0x02 selects `+`.
- Otherwise, flag 0x04 selects a space.
- Otherwise, no sign is selected.

This does not establish that negative numeric payloads are processed correctly or safely by the dependencies.

### Precision selection and numeric preparation

The first call to F-baa332e06e32 receives the original two-word payload and produces a 32-bit integer-like quantity.

After that call, the function records whether the normalized precision exceeds eight and whether it equals nine. Precision greater than nine is then capped at nine.

Let **P** denote the capped precision. It is always between zero and nine. A separate adjustment condition is true when the pre-cap normalized request is zero through nine, and false when that request is ten through 2,147,483,647. A negative original request, already replaced by six, takes the true branch.

The remaining preparation follows this dependency order:

1. F-b3a54d881220 receives the first integer-like quantity and returns a two-word result.
2. F-387e686c467e receives that result and the original payload.
3. F-a6af171adb23 receives P.
4. F-c4b34006a13c receives the precision-derived result and the preceding arithmetic result.
5. F-baa332e06e32 is called again, producing a second 32-bit integer-like quantity. Its arguments are not explicitly represented.
6. F-b3a54d881220 is called again. Its arguments are not explicitly represented.
7. F-387e686c467e receives the latter two-word result and the saved result from F-c4b34006a13c.
8. The function reads a **different pair of global operand locations** and passes those values, together with the preceding arithmetic result, to the first F-c6bfc9d0fb36 call.
9. If the precision adjustment condition is true, the second integer-like quantity is increased by one.
10. F-a6af171adb23 receives P again.
11. F-b3a54d881220 receives the second integer-like quantity.
12. F-c6bfc9d0fb36 is called again; its arguments are not explicitly represented.
13. If the adjustment condition is true, the first integer-like quantity is increased by one, F-a6af171adb23 receives P a third time, and another F-baa332e06e32 result is subtracted from the second integer-like quantity.

The global reads in step 8 occur after the preceding arithmetic calls, not at entry. Their values are neither fixed constants nor guaranteed entry-state values: any intervening dependency may modify the relevant storage. These locations are distinct from the pair read for F-11ed8242c40c. The representation does not establish a within-pair read order.

The increments depend on the recorded precision condition, not on a visibly inspected result from F-c6bfc9d0fb36. Neither of its returned values is visibly tested.

The two resulting 32-bit quantities drive the integral and fractional extraction stages. Scaling, rounding, and carry handling are plausible intent, but exact arithmetic semantics and some argument associations remain documented_only.

### Integral extraction and table reads

The integral stage retains a pointer to the first indexed position of a character table. Retaining that pointer does **not** snapshot the table contents.

Each iteration:

- Calls F-870c4fcf7295 with the current integral quantity and divisor ten.
- Uses the low 32 bits of its paired result as the next quantity.
- Uses the high 32 bits, interpreted as an index, to read one byte through the retained table-base pointer.
- Copies the byte into a local buffer.

The dependency call precedes the table-byte read on every iteration. If the dependency changes table memory before returning, the following read observes the memory then available, not necessarily its entry-state contents. Later iterations also read memory afresh.

The stage runs at least once. It stops when the next quantity is zero or after twenty iterations.

If twenty iterations occur, the retained count is reduced to nineteen, and the twentieth stored byte is replaced by a zero terminator. Subsequent emission therefore uses at most nineteen integral bytes. No truncation error is reported.

Retained bytes are emitted in reverse extraction order. Ordinary most-significant-first decimal text follows only if the dependency behaves as quotient-and-remainder division and the relevant table reads yield conventional digit bytes.

### Fractional extraction and table reads

The fractional stage also calls F-870c4fcf7295 before each indexed table-byte read. The low result word becomes the next quantity, and the high result word selects the byte relative to the corresponding table base.

Every selected byte is copied into a separate local buffer. The table is read on each iteration; this stage does not use a snapshot either. Dependencies may modify table contents between reads, including between the integral and fractional stages.

The iteration count is:

- One when P is zero.
- Exactly P when P is one through nine.

A zero next quantity does not terminate fractional extraction early.

A zero terminator follows the stored bytes, but subsequent output uses counts rather than terminator scanning. One through nine fractional bytes are retained.

The function also computes an additional fractional-zero count as P minus the stored fractional-byte count, clamped to zero. This count is zero on every visible reachable path.

### Padding calculation

Let:

- **W** be the field-width word.
- **I** be the retained integral-byte count.
- **P** be the capped precision.
- **S** be one if a sign is selected, otherwise zero.

The preliminary padding expression is:

**W − I − P − S − 1**

The final subtraction reserves a position even when no decimal point will be emitted.

The subtraction chain involves 32-bit unsigned operands. Its width must not be silently expanded into unbounded mathematical arithmetic. The resulting 32-bit value is used as a signed padding count; a negative count is replaced by zero. Extreme-width behavior depends on the target’s unsigned-to-signed conversion convention, which is not independently established beyond the supplied representation.

Flag 0x01 negates the resulting nonnegative count. A positive count requests leading padding; a negative count requests trailing spaces. Zero requests neither.

### Emission order

All numeric preparation and all table-byte extraction finish before the first call to F-bcb8529decd8.

With positive padding and without flag 0x10, the function requests the calculated number of spaces before any selected sign.

With positive padding and flag 0x10:

- Without a selected sign, it requests exactly the calculated number of zeros.
- With a selected sign, it requests the sign first, clears the pending sign, and requests one fewer zero than the calculated padding count.

The second case consumes one padding unit for a sign that has already been counted separately in the width expression.

After leading padding, emission proceeds in this order:

1. Any still-pending sign.
2. Retained integral bytes, in reverse extraction order.
3. The conditional decimal point and retained fractional bytes.
4. Additional fractional zeros, whose reachable count is zero.
5. Trailing spaces for a negative padding count.

For P greater than zero, the function requests a decimal point followed by exactly P fractional bytes.

For P equal to zero:

- Without flag 0x08, neither the decimal point nor the extracted fractional byte is emitted.
- With flag 0x08, both the decimal point and the one extracted fractional byte are emitted.

Digit emissions use the bytes already copied into local buffers. They do not reread the character table. A table modification during output therefore does not, by itself, change those buffered bytes.

No output return value is checked. Normal return from each output call permits the next selected emission to proceed.

## Side effects

Established direct effects are global-operand reads, indexed table-byte reads, local-buffer writes, and dependency invocations, including character-emission requests.

No explicit write to caller-owned memory or global storage appears in the function body. That does not establish an absence of external changes: every dependency may modify memory or global state according to its unknown contract.

Relevant observable ordering includes:

- The first early operand pair is read before the first F-11ed8242c40c call.
- The same locations are reread after that call and before the second call.
- The later operand pair is read only after the intervening numeric dependencies.
- Each extraction dependency call occurs before its corresponding table-byte read.
- Every table-byte read and local copy precedes output.

The function retains the output-context word values for reuse. It does not guarantee that objects reachable through those words, global operands, or table contents remain unchanged.

There is no explicit error return, rollback, retry, or recovery path. Dependency-specific failures, traps, exceptions, and nonlocal exits are unresolved. The absence of a status check must not be interpreted as successful delivery of every requested character.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-11ed8242c40c | Called twice with the original payload. Each call obtains its other two operands from fresh reads of the same two global locations. Returned values are not visibly consumed. |
| F-baa332e06e32 | Produces 32-bit integer-like quantities. Arguments are incompletely represented for later calls. |
| F-b3a54d881220 | Produces two-word intermediates; some calls have explicit integer-like inputs and one has incompletely represented arguments. |
| F-387e686c467e | Receives two pairs of words during numeric preparation. Exact arithmetic is unknown. |
| F-a6af171adb23 | Receives capped precision two or three times. |
| F-c4b34006a13c | Receives a precision-derived pair and an earlier arithmetic-result pair. |
| F-c6bfc9d0fb36 | Called twice. The first call receives an arithmetic result and a freshly read, separate global operand pair; the second argument list is incomplete. |
| F-870c4fcf7295 | Returns the next extraction quantity and a table index from a quantity and divisor ten. Each call precedes an indexed table-byte read. |
| F-bcb8529decd8 | Receives the retained four-word context and one character value per requested emission. |

These entries describe boundary usage, not guarantees of purity, arithmetic meaning, memory preservation, or ownership.

## Acceptance examples

These are conditional deductions, not runtime observations. Unless stated otherwise, assume successful, normally returning dependencies and table reads that produce integral text `12` and the fractional byte `3`.

1. **Ordinary fractional output:** precision one, width zero, and no flags request `12.3`.

2. **Leading spaces:** precision one, width eight, and no flags calculate four padding units and request four spaces followed by `12.3`.

3. **Trailing spaces:** the preceding case with flag 0x01 requests `12.3` followed by four spaces.

4. **Unsigned zero padding:** precision one, width eight, and flag 0x10 request `000012.3`.

5. **Sign with zero padding:** precision one, width eight, and flags 0x02 and 0x10 calculate three padding units but request `+0012.3`. Only two padding zeros follow the sign; the total is seven characters.

6. **Sign precedence:** flags 0x02 and 0x04 together select `+`, not a space.

7. **Zero precision without forced output:** width eight, precision zero, and no flags calculate five padding units and request five spaces followed by `12`. The un-emitted decimal-point position leaves seven characters.

8. **Zero precision with flag 0x08:** the preceding case requests five spaces followed by `12.3`, including the otherwise suppressed fractional byte.

9. **Negative precision word:** 0xffffffff becomes precision six and takes the enabled adjustment branch.

10. **Precision cap:** requests nine and ten both retain nine fractional bytes, but only nine takes the increment-and-adjust branch. Identical numeric text is not required.

11. **Early global mutation:** if the first F-11ed8242c40c call changes either early operand location, the second call receives values read after that change. Reusing the first captured operand pair would not match the visible reads.

12. **Later global mutation:** if a preceding arithmetic dependency changes a later operand location, the first F-c6bfc9d0fb36 call receives its subsequently read value, not an entry-time snapshot.

13. **Table mutation:** if an extraction call changes the table byte selected by its returned index, that iteration copies the byte present after the call. A later table change does not retroactively replace the buffered byte.

14. **Output failure status:** if an output call reports failure through its return value but returns normally, subsequent emissions continue because that value is not tested.

15. **Integral limit:** if extraction reaches twenty iterations, only the first nineteen extracted integral bytes remain eligible for reversed emission; the twentieth is overwritten by the terminator.

## Uncertainties

The following remain **documented_only**:

- Exact payload representation, signedness, special values, and external word ordering.
- Numeric meanings of the arithmetic dependencies and incompletely represented arguments.
- Exact rounding, conversion, carry, and out-of-range behavior.
- Values read from either global operand pair, including changes made by dependencies.
- Relative read order within an individual global operand pair.
- Complete table contents, table mutability, valid index range, and behavior for invalid indices.
- Dependency behavior for negative extraction quantities.
- Numeric-intermediate overflow semantics and extreme unsigned-to-signed conversion behavior.
- Dependency effects on global storage, overlapping storage, ownership, and failure handling.

The supplied representation warns about overlapping global symbols. It does not justify assuming all global data regions are independent.

The formatter-like structure suggests decimal scaling and rounding, but does not establish a standard formatting-library contract.

## Evidence

This contract derives solely from the supplied static function representation and opaque dependency mapping. Established evidence includes argument widths, reachable local sign state, precision conditions, dependency order, separate global-read occurrences, table-read placement, local byte copies, extraction counts, padding operations, and emission order.

The supplied representation identifies removed unreachable regions. Their omitted behavior is not incorporated into this contract.

No implementation was executed. No runtime tests or dependency inspections were performed. Acceptance examples express conditional static consequences, not observed results.
