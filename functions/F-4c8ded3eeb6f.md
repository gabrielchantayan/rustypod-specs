<!-- Trusted provenance: F-4c8ded3eeb6f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4c8ded3eeb6f

## Purpose

F-4c8ded3eeb6f conditionally performs a sequence of arithmetic checks, up to four memory reads, and up to four memory writes. The supplied static representation does not establish an application-level purpose.

The maximum visible memory-access sequence consists of two 32-bit reads, a signed 16-bit read, two 32-bit writes, another signed 16-bit read, and two further 32-bit writes. Incoming flags and additional register state determine which accesses are reached.

This contract is **documented_only**. The representation reports invalid instruction data and truncated control flow, and every visible path ends at an invalid-instruction marker. It does not establish a valid executable function boundary, a complete implementation, or normal return behavior.

## Interface

### Declared inputs and result

| Input | Represented type | Visible role |
|---|---|---|
| First argument | Unsigned 32-bit word | Its incoming value is not consumed. Reached arithmetic stages replace its working value. |
| Second argument | Unsigned 32-bit word | Its incoming value does not supply any visible store. Before a store can occur, its working value is replaced by an indirectly loaded word. |
| Third argument | Pointer to a signed 16-bit value | Its converted address contributes to initial arithmetic. A later stage reads the original address and forms an address one byte beyond it. |
| Fourth argument | Unsigned 32-bit word | Acts as a byte displacement and participates in an address-equality check. |

No return value is declared, and no normal return is visible.

The represented operations use 32-bit unsigned words, 32-bit signed integers, and signed 16-bit halfwords. Address conversions pass through 32-bit integers. These observations describe the supplied representation; they do not independently establish the actual ABI, pointer width, or integer-to-address mapping.

### Additional entry state

Execution also depends on:

- An incoming zero flag.
- An incoming carry flag, interpreted numerically as zero or one.
- Eight additional unsigned 32-bit register values.
- An incoming link-register address.
- A stack-related address.

The eight additional words are identified here by ordinal labels, not as declared arguments:

| Hidden word | Visible uses |
|---|---|
| First | First subtraction; logical right shift by one; later equality with the second hidden word. |
| Second | Second subtraction; later equality with the first hidden word. |
| Third | Increment by 36; later equality with the fourth hidden word. |
| Fourth | Third subtraction; later equality with the updated third hidden word. |
| Fifth | Equality with the updated sixth hidden word. |
| Sixth | Increment by 8; equality with the fifth hidden word. |
| Seventh | Second subtraction; final equality with the updated eighth hidden word. |
| Eighth | Increment by 10; final equality with the seventh hidden word. |

The representation does not establish whether working-register changes are observable by a caller, exception handler, or other execution context.

### Memory and ownership preconditions

Accessibility requirements are path-dependent. An address used only by an unreached operation need not be accessible for the earlier visible operations to complete.

A complete, non-faulting maximum-length path requires:

- A readable 32-bit word at the incoming link-register address.
- A readable 32-bit word at the address derived by narrowing that first word to a character-sized value and adding four.
- A readable signed 16-bit value at the original third-argument address.
- Two writable 32-bit locations at offsets 13 and 17 bytes from the incoming link-register address.
- A readable signed 16-bit value at the later displacement-derived address.
- Two writable 32-bit locations derived from the final signed-halfword value.

The contract establishes no allocation, ownership transfer, lifetime extension, alignment guarantee, or permission guarantee. A 13-byte displacement does not preserve word alignment when the original address is word-aligned.

## Behavior

### Arithmetic and conversion rules

Unsigned 32-bit arithmetic is interpreted modulo \(2^{32}\). Unsigned comparisons use the range zero through \(2^{32}-1\). A Boolean complement in this contract means one for false and zero for true.

Not every address-related expression is purely unsigned. The representation includes pointer-to-signed-integer conversions and signed arithmetic. Consequently, formulas using wrapping 32-bit addresses below are conditional on the target providing the corresponding bit-pattern behavior. They do not establish portable behavior for signed overflow or integer-to-pointer conversion.

The initial variable left shift uses the low eight bits of its own input as the shift count. That count can range from 0 through 255. Behavior for counts of 32 or greater is unresolved; no masking, saturation, or other processor-specific rule is assumed.

Each later variable right shift, if reached, has a zero count. Its controlling gate requires the preceding working result—the source of the count—to be zero.

The character-sized conversion used for the second read is independently significant. Character width, plain-character signedness, and out-of-range signed narrowing behavior are not established. Eight-bit signed characters alone do not establish low-eight-bit truncation followed by two’s-complement interpretation. Under applicable language rules, an out-of-range conversion to a signed character type can have an implementation-defined result or raise an implementation-defined signal.

### Initial arithmetic gates

If the incoming zero flag is clear, no visible memory access occurs. Execution proceeds to the terminal invalid-instruction marker.

If the incoming zero flag is set, the converted third-argument address, the incoming carry bit, and `0x085740e8` contribute to an initial word. That word is shifted left by its own low-eight-bit count. Call the resulting word the shifted operand.

The first carry-like state is true when either:

- The shifted operand is less than the first hidden word; or
- The unsigned difference between the first hidden word and the shifted operand is less than the incoming carry bit.

The first gate succeeds only when the first hidden word, minus the shifted operand, minus the Boolean complement of the incoming carry bit, is zero.

If that gate succeeds, a second carry-like state is true when either:

- The seventh hidden word is less than the second hidden word; or
- The unsigned difference between the second and seventh hidden words is less than the first carry-like state.

The second gate succeeds only when the second hidden word, minus the seventh hidden word, minus the Boolean complement of the first carry-like state, is zero.

If the second gate succeeds, the third hidden word increases by 36. The first hidden word is also logically shifted right by one for the next calculation.

The third carry-like state is true when either:

- That shifted value is less than the fourth hidden word; or
- The unsigned difference between the fourth hidden word and that shifted value is less than the second carry-like state.

The third gate succeeds only when the fourth hidden word, minus that shifted value, minus the Boolean complement of the second carry-like state, is zero.

A failed gate prevents every subsequent memory access. The increment by 36 has already occurred if the third gate is evaluated, even when that gate fails.

These carry-like relationships are those represented statically. They are not interchangeable with assumed conventional carry or borrow rules.

### Initial reads and address gate

When all three initial gates succeed:

- The sixth hidden word increases by 8.
- The eighth hidden word increases by 10.
- A destination cursor is formed 13 bytes beyond the incoming link-register address.

The first two memory reads then occur in this order:

1. Read a 32-bit word at the original link-register address.
2. Narrow that word to a character-sized value, add four, interpret the result as an address, and read a 32-bit word there.

The second loaded word becomes the value used for the second store of each reached write pair.

The second read is not a read at the full first loaded word plus four. Narrowing precedes the addition. If that conversion raises a signal or otherwise prevents continued execution, the first read has already occurred, but the second read and all later accesses are not established.

After those reads, an arithmetic gate combines:

- `0x042ba0a0`;
- The stack-related address converted to an unsigned word and logically shifted right by one;
- The destination cursor; and
- Two occurrences of the Boolean complement of the third carry-like state.

Under uniform 32-bit bit-pattern arithmetic, the Boolean terms cancel. The resulting expression is:

\[
\texttt{0x042ba0a0}
-
(\text{converted stack-related address} \mathbin{\text{ logical right shift }} 1)
+
\text{destination cursor}.
\]

The next stage requires that result to be zero. Signed conversions and signed intermediate arithmetic qualify this simplification.

Both initial reads precede this gate. Failure does not undo those reads or the hidden-word increments.

### First halfword and first write pair

If the address gate succeeds:

- A working third-argument address is formed one byte beyond the original address.
- A signed 16-bit value is read at the original third-argument address.
- The value is sign-extended and represented as an unsigned 32-bit word.
- A displaced cursor is formed by subtracting the fourth argument from the destination cursor as a byte displacement.

The first write pair occurs in order:

1. Write the sign-extended halfword as a 32-bit word at the destination cursor.
2. Write the earlier indirectly loaded word four bytes later.

Under the apparent address model, the destinations are the incoming link-register address plus 13 and plus 17 bytes.

Only after both writes is the first halfword tested for zero. A nonzero value prevents later gates but does not suppress or undo this write pair.

### Gates following the first write pair

If the first halfword is zero, the following conditions must succeed in order:

1. The indirectly loaded word is zero.
2. The fourth argument equals the one-byte-advanced third-argument address converted to an unsigned word.
3. The second hidden word equals the first hidden word.
4. The fourth hidden word equals the third hidden word after its increment by 36.
5. The sixth hidden word after its increment by 8 equals the fifth hidden word.
6. The eighth hidden word after its increment by 10 equals the seventh hidden word.

The represented exclusive-or and variable-shift expressions reduce to these equality conditions because every reached shift count is zero.

Failure of any condition prevents the final halfword read and final write pair. The first write pair remains completed.

### Final halfword and final write pair

If all preceding gates succeed, a signed 16-bit value is read 13 bytes beyond the displaced cursor.

Under wrapping 32-bit address arithmetic, its address is:

\[
\text{incoming link-register address} + 26 - \text{fourth argument}.
\]

This read occurs after the first write pair. If its byte range overlaps those writes, its value depends on the resulting memory bytes and target byte order, not necessarily on entry memory.

The final halfword is sign-extended to a signed 32-bit value. The final writes occur in order:

1. Write that sign-extended value as a 32-bit word at the address represented by the value minus `0x08574200`.
2. Write the earlier indirectly loaded word at the address represented by the value minus `0x085741fc`.

The destinations are four bytes apart. Both subtractions remain within the signed 32-bit range for every signed 16-bit input. Their conversion to usable addresses remains target-dependent.

Reaching this stage requires the indirectly loaded word to be zero. Thus, on every path reaching the final pair, the second word of both write pairs is zero.

### Termination and errors

Every visible path that completes its reached operations proceeds to an invalid-instruction marker. No normal return, status value, error code, recovery branch, or exception classification is established.

A failed conversion or unsupported memory access may prevent later operations. The representation provides no rollback of completed writes. Actual handling of invalid instructions, access faults, conversion signals, and unaligned operations remains unresolved.

## Side effects

The maximum visible memory-access sequence is:

| Order | Access |
|---|---|
| 1 | Read 32 bits at the incoming link-register address. |
| 2 | Read 32 bits at the character-conversion-derived address. |
| 3 | Read a signed 16-bit value at the original third-argument address. |
| 4 | Write the sign-extended first halfword as 32 bits. |
| 5 | Write the indirectly loaded word four bytes later. |
| 6 | Read a signed 16-bit value at the displacement-derived address. |
| 7 | Write the sign-extended final halfword at its value-derived address. |
| 8 | Write the indirectly loaded word four bytes later. |

This is the order represented statically, not an independently verified hardware-ordering guarantee. Overlapping locations are not excluded.

Working-state changes include the conditional hidden-word increments by 36, 8, and 10, replacements of argument working values, and address derivations. No ABI-level preservation or final observable register state is established.

No allocation, release, ownership acquisition, or ownership transfer is visible. No atomicity, synchronization, transaction, volatile-access, or concurrency guarantee is established.

## Dependencies

No ordinary helper call with a separately identified opaque function ID is visible.

F-4c8ded3eeb6f depends on ambient flags and register state, accessible memory, arithmetic and conversion semantics, shift behavior, alignment handling, byte order, and terminal invalid-instruction handling.

The terminal marker does not establish a callable helper contract. Whether execution traps, stops, transfers control, or reflects incorrectly interpreted instruction data remains **documented_only**. No unknown dependency is assumed to preserve global state or acquire or release memory.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Unless stated otherwise, they assume defined target semantics for reached arithmetic and successful reached accesses.

### Incoming zero flag is clear

No visible memory read or write occurs, regardless of the four declared argument values. Execution reaches the terminal marker without an established normal return.

### An initial subtraction gate fails

If the first or second gate fails, neither initial word read occurs.

If only the third gate fails, memory is still untouched, but the third hidden word has already increased by 36.

### Address gate fails

If all three initial gates succeed and the later address gate fails:

- Two word reads have occurred.
- All three hidden-word increments have occurred.
- No halfword read or write is reached.

This example additionally assumes that the intervening character conversion completes.

### First halfword is negative one

If the address gate succeeds and the first signed halfword is −1, the first write pair stores `0xffffffff`, followed by the indirectly loaded word. No final halfword read or final write pair occurs.

The nonzero halfword does not suppress the first pair.

### First halfword is zero and the indirect word is nonzero

The first pair stores zero followed by the nonzero indirect word. The next gate fails. Three reads and two writes have occurred.

### Character narrowing affects the indirect address

Suppose the first loaded unsigned word is 128, and characters are eight bits wide.

If the character conversion has unsigned-character semantics, the narrowed value is 128. Adding four gives the integer address value 132.

If plain characters are signed, those assumptions alone do **not** establish the narrowed result: 128 is outside the signed eight-bit range. Under applicable language rules, the conversion result is implementation-defined or an implementation-defined signal may be raised.

Only with the additional assumption that narrowing retains the low eight bits and interprets them as a two’s-complement signed value does the narrowed value become −128. Under that explicit assumption, adding four gives −124. Converting −124 into an address remains a separate target-dependent operation.

The same qualification applies to other loaded words whose low byte is `0x80`; the low byte alone does not establish signed narrowing behavior. None of these cases establishes address accessibility.

### All gates succeed and the final halfword is zero

The complete path performs four reads and four writes.

Under a 32-bit pointer-bit-pattern interpretation, the final destinations are `0xf7a8be00` and `0xf7a8be04`. Both final stored words are zero. Writable mappings at those addresses are an additional environmental precondition.

### Final read overlaps the first pair

If the final halfword range overlaps a completed first-pair write, the read occurs afterward. Its value must be determined from the resulting bytes under the target’s byte order. Entry contents alone are insufficient.

## Uncertainties

The following remain **documented_only**:

- Validity of the entry boundary, instruction interpretation, and truncated body.
- Application-level purpose.
- Calling convention and availability of additional entry state.
- Behavior for initial shift counts of 32 or greater.
- Character width and plain-character signedness.
- Character narrowing behavior, including implementation-defined results or signals for out-of-range signed conversions.
- Pointer-to-integer conversions, signed address arithmetic, and signed-overflow cases.
- Actual address width and integer-to-pointer mapping.
- Alignment requirements and unaligned-access behavior.
- Endianness and byte-level overlap results.
- Memory permissions, ownership, and object lifetimes.
- Register, flag, and memory state observable after a fault or terminal event.
- Meaning and handling of the invalid-instruction marker.
- Concurrency and hardware-level ordering.

No unestablished conversion, preservation, memory-management, or recovery behavior is assumed.

## Evidence

**Established static evidence:** The supplied representation declares four arguments, consumes additional flags and register values, conditionally updates three hidden words, and contains four possible reads and four possible writes. The reads comprise two word reads and two signed-halfword reads. The indirect address uses character narrowing before adding four. The representation reports invalid instruction data and ends with truncated control flow at an invalid-instruction marker.

**Derived static consequences:** Reached later variable shifts have zero counts. The working third-argument address advances by one byte. The first write pair precedes the later equality gates and final halfword read. Reaching the final write pair requires the indirectly loaded word to be zero. The two final destination subtractions fit within signed 32-bit range.

**Conditional consequences:** Wrapping-address formulas require compatible target conversion and arithmetic semantics. The signed-character example yielding −128 and then −124 additionally requires low-eight-bit truncation with two’s-complement signed interpretation; signedness and width alone are insufficient.

**Inferred intent:** None adopted. The evidence does not establish a copy routine, validator, decoder, dispatcher, or other application-level operation.

**Runtime evidence:** None. No runtime tests have been performed.
