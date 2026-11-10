<!-- Trusted provenance: F-e51f64be89ac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e51f64be89ac

## Purpose

F-e51f64be89ac describes an ordered sequence of memory accesses, global-state updates, address-derived writes, and conditional control flow ending at invalid-instruction boundaries. The supplied static evidence establishes neither a conventional service nor a successful return.

The contract below records observable accesses and their dependencies without assigning a higher-level purpose. Apparent initialization, self-linking, or state destruction is **inferred intent**, not required behavior.

Three evidence categories apply:

- **Established static evidence:** explicit access widths, arithmetic expressions, captures, repeated reads and writes, and represented control-flow order.
- **Conditional interpretation:** consequences requiring stated arithmetic or memory-model assumptions.
- **documented_only:** unresolved behavior, including instruction validity, some storage widths, oversized shifts, and actual fault handling.

Every described operation is conditional on execution reaching it and earlier operations completing. Invalid-instruction warnings prevent treating the representation as proof of an executable end-to-end path.

## Interface

### Explicit inputs and output

| Argument | Represented type | Established use |
|---|---|---|
| First | Unsigned 32-bit integer | Address formation, bitwise operations, unsigned subtraction, index derivation |
| Second | Signed 32-bit integer | Address displacement and source of a truncated 16-bit write |
| Third | Opaque 32-bit value | No explicit use |
| Fourth | Signed 32-bit integer interpreted as an address | 32-bit write at byte offset 8 and 32-bit read at byte offset 40 |

No return value or normal return is represented. There is no established success indicator or recoverable error result.

The third argument’s lack of explicit use does not prove machine-level independence from its incoming register or stack storage.

### Ambient inputs

Execution additionally depends on:

- An incoming register-derived address base with no preceding assignment in the representation.
- A pointer supplied in a stack slot at byte displacement 592 from the represented stack reference.
- Initial global values and subsequently reread global storage.
- Numeric addresses associated with F-e47d7b2e9c88 and F-8afdaab91693.
- Memory reached through arguments, global values, pointer values, and derived addresses.
- The target’s arithmetic, pointer-conversion, alignment, endianness, and fault semantics.

No validation, admissible range, allocation size, or ownership requirement is supplied.

### Width and layout conventions

Explicit memory accesses are 8, 16, or 32 bits. All offsets below are byte offsets unless stated otherwise.

Unsigned 32-bit subtraction and multiplication have modulo-\(2^{32}\) results in the represented arithmetic. This does not resolve signed overflow, signed left shifts, oversized shifts, or the eventual machine interpretation of an integer as an address.

Global storage is described by role or numeric address, not assumed to be independent objects. The supplied overlap warning makes independence unsafe. Where no access width is established, this contract leaves the physical width **documented_only**.

## Behavior

### 1. Capture global state and perform initial linked accesses

Two 32-bit global values are captured before the first explicit write: a field-access base and an address offset. Subsequent references to these captures use their captured values, not fresh reads of their original global storage.

The represented access order is:

1. Write the second argument’s low 16 bits at the incoming register base plus the second argument.
2. Read 32 bits at the captured address offset plus the bitwise OR of the first argument with `0x88`.
3. Read one byte at the incoming register base plus that same OR-derived component.
4. Write the 32-bit result from step 2 at the fourth argument plus 8.
5. Write the fourth argument’s 32-bit value at the first argument plus `0x40`.
6. Write the result from step 2 at the captured field base plus `0x74`.
7. Read 32 bits at the fourth argument plus `0x28`.
8. Read 32 bits at the captured field base plus `0x5C`.
9. Write the captured field base’s low byte at the address read in step 2, plus 3.
10. Write the byte from step 3 at the captured field base plus `0x10`.
11. Write the low byte of the step-8 value at the numeric address equal to the step-3 byte plus `0x1C`.

The OR operation sets selected bits; it is not addition. The destination in step 11 is numerically between `0x1C` and `0x11B`, inclusive.

Earlier writes may overlap later read locations. These operations therefore cannot be treated as independent field initialization.

### 2. Derive an index and update a halfword region

The following quantities are derived from the first argument:

- A 32-bit unsigned result after subtracting `0x38`.
- A 32-bit unsigned difference after subtracting `0xE0`.
- A halfword-region address obtained by multiplying that difference by two, modulo \(2^{32}\).
- A masked difference obtained by clearing bit 31 of the difference.
- An index obtained by shifting the masked difference right by one.

The index ranges from zero through `0x3FFFFFFF`.

A separate destination is derived from the previously captured 32-bit read at the fourth argument plus `0x28`. Its expression shifts that signed value left by two and then right by 32. The supplied representation does not establish a reliable result for that expression; the resulting destination is **documented_only**.

Subject to that unresolved calculation, the memory effects occur in this order:

1. Write the index’s low byte at the captured offset-40 value plus `0x0897E8B8`.
2. Assign `0xB8` to a global location of unstated physical width.
3. Write the low byte of the first-argument-minus-`0x38` result at the first argument minus `0x32`.
4. Write that full 32-bit subtraction result to the unresolved destination.
5. Write `0xFFFF` as 16 bits at halfword-region offsets 4 and 6, in that order.
6. Write `0xB8` as one byte at the unresolved destination plus 4.
7. Write `0xFFFF` as 16 bits at halfword-region offset 2.
8. Assign the halfword-region address’s low 16 bits to a global location.
9. Read 16 bits at halfword-region offset zero, zero-extend the result into an address, and write 32 bits there. The stored value is the first-argument-minus-`0x38` result masked with `0x897E8B80`.

No explicit initialization of halfword-region offset zero precedes its read. Other writes can affect it only through aliasing. The zero-extended destination in step 9 is numerically within `0x0000`–`0xFFFF`; accessibility is not established.

### 3. Use the address of F-e47d7b2e9c88

F-e47d7b2e9c88 is not called. Its address is used as scalar data and as a memory destination.

The representation derives:

- A base from the dependency address interpreted as signed 32-bit and shifted right by 32.
- A sign-related 16-bit value from the same signed interpretation shifted right by 31 and truncated to 16 bits.

The first derivation is unresolved. The second depends on signed-right-shift semantics.

Before testing the address’s sign, the function writes the sign-related halfword at the derived base plus 4.

If the dependency address, interpreted as signed 32-bit, is negative, the represented path reaches an invalid-instruction boundary immediately afterward. All preceding completed accesses remain before that boundary; this is not preflight validation.

Otherwise, the represented sequence is:

1. Write the sign-related halfword at the derived base plus `0x12`.
2. Write the dependency address as 32 bits at that address itself.
3. Read a signed 16-bit value at the derived base plus the earlier index.
4. Repeat the halfword write at offset `0x12`.
5. Repeat the 32-bit self-address write.
6. Sign-extend the read halfword to 32 bits, shift it right by 16, and write the result at the derived base plus 8.

The index in step 3 is a byte displacement, not a halfword-element index.

Under arithmetic signed-right-shift semantics, step 6 stores zero for a nonnegative halfword and `0xFFFFFFFF` for a negative halfword. The repeated writes remain observable operations; neither aliasing nor device-backed behavior is excluded.

### 4. Update low-address storage and the stack-supplied region

After the nonnegative branch:

1. A global location is assigned zero.
2. A small base is selected as `0x58` plus bit 2 of the dependency address, retaining that bit’s value of zero or four. The base is therefore `0x58` or `0x5C`.
3. The unresolved derived base from section 3 is written as 32 bits at the small base plus 4.
4. The masked difference shifted right by 17 is assigned as a 16-bit value to another global.
5. The index’s low byte, zero-extended to 16 bits, is assigned to another global.
6. The full index is written as 32 bits at four times the small base plus `0x10`.
7. The full index is assigned to another global.

The 32-bit destinations in steps 3 and 6 are respectively `0x5C` or `0x60`, and `0x170` or `0x180`.

The stack-supplied pointer is then used as follows:

1. Write the small base as 16 bits at the pointer.
2. Overwrite that location with 32 bits obtained by shifting the unresolved derived base left by eight.
3. Write the same shifted value as 32 bits at pointer offset 20.
4. Overwrite the pointer’s first 32 bits with the pointer’s numeric value.
5. Read 32 bits at twice the pointer’s numeric value.
6. Write that read value’s low 16 bits at the original pointer.
7. Treat the full read value as an address and write that value to itself as 32 bits.

The next 32-bit write uses the **address of the stack slot**, not the pointer stored in it. Its destination is that slot address multiplied by `0x20`, plus `0x18`. Its value is the stack-supplied pointer interpreted as signed 32-bit and shifted right by 31.

An overlapping global update then reads its byte-sized constituent at that point and concatenates a 16-bit value of 5 above that byte. The resulting numeric composition is `0x000500` plus the byte. The physical destination width is not established.

Finally, the pointer’s numeric value is written as 32 bits at four times that value plus `0x18`.

### 5. Perform final global and address-derived updates

A signed 32-bit global value is captured. The numeric address of F-8afdaab91693 is also captured; no call occurs.

The subsequent operations are:

1. Assign four times the stack-supplied pointer’s numeric value to the 32-bit global at `0x0897EA80`.
2. Read that global for the destination of a 32-bit store and read it for the value being stored.
3. Read that global again, add 4, and write the earlier index as 32 bits at that destination.
4. Multiply the separately captured signed global value by four. Write that product as 32 bits at the address given by that same product.
5. Form another destination by shifting the captured signed global value left by six, shifting that result right by 17, and adding the result to the captured value. Write the address of F-8afdaab91693 shifted left by two as 32 bits at this destination.
6. Convert the captured global value to unsigned 32-bit, extract its high 16 bits, and assign them to a 16-bit reset-halfword global.
7. Assign zero to another global.
8. At eight times the captured F-8afdaab91693 address, write 32-bit zero at offset zero, then 32-bit zero at offset 4.
9. At offset 16 from that same eight-times-address base, write one byte equal to the base’s low byte plus eight, truncated to eight bits.

Step 2 does **not** establish an evaluation order between its destination-producing read and value-producing read. Neither read may be replaced contractually with the value assigned in step 1. Step 3 likewise requires a fresh read after the preceding store. Intervening memory effects are not assumed to preserve this global.

Step 4 uses four times the **captured global value**, not four times the stack pointer. These are distinct address expressions that may alias; equality is not guaranteed.

Step 5’s complete arithmetic is material to its destination. Signed left shift, overflow, and signed-right-shift semantics remain unresolved. No portable numeric result is imposed.

The final assignments occur in this order:

| Storage identified by numeric address or role | Assigned value |
|---|---|
| `0x00000086` | `0x897EA700` |
| `0xEA700000` | `0x66` |
| `0x0897EB26` | `0xEAEC` |
| `0x12FD4C34` | `0x1A` |
| `0x897EA65C` | That storage’s own address, `0x897EA65C` |
| Overlapping reset aggregate | Current reset-halfword value concatenated above low halfword `0xEA70` |
| `0x12FD4CB8` | `0x5C` |

For assignments without an explicit access width, the physical width remains **documented_only**.

The reset-aggregate update reads the reset-halfword global **at that point**. It does not reuse the high halfword previously extracted in step 6. Intervening writes and overlapping storage may have changed it. If its current value is \(h\), the concatenated 32-bit value is \(h \times 65536 + \texttt{0xEA70}\).

The represented path then reaches another invalid-instruction boundary. No normal return follows.

## Side effects

The established effects are memory mutations across argument-relative, register-relative, global-derived, low-numeric, dependency-address-derived, stack-derived, and fixed-address storage.

There is no shown allocation, release, ownership transfer, reference counting, locking, rollback, or output-object construction. Writing a pointer does not establish ownership or extend any object’s lifetime.

No shown check establishes that a destination is mapped, writable, aligned, or within an object. A fault may interrupt the sequence after partial mutation. The representation contains no rollback.

Captured scalar values and fresh global reads have different contracts. Captures retain their represented values despite subsequent aliasing writes; explicitly repeated reads observe storage again. No unshown preservation guarantee applies.

## Dependencies

| Opaque ID | Established relationship | Unresolved properties |
|---|---|---|
| F-e47d7b2e9c88 | Address used for shifts, sign testing, repeated self-address stores, and low-address selection | Actual address width, relocation, writability, intended role |
| F-8afdaab91693 | Address used in shifted values and multiplied-address destinations | Actual address width, relocation, validity of derived accesses |

Neither dependency is visibly invoked. Their callable input/output contracts, global preservation, and memory ownership behavior are not established or relied upon.

The invalid-instruction boundaries are not established recoverable dependency calls.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **OR rather than addition.** A first argument of zero produces an OR-derived component of `0x88`. A first argument of `0x88` produces the same component, not `0x110`.

2. **Truncation does not change displacement.** With a second argument of negative one, the initial 16-bit write stores `0xFFFF`. Its address expression still uses the signed displacement negative one.

3. **Unsigned index boundary.** With a first argument of `0xE0`, the masked difference and index are zero. With `0xDF`, subtraction wraps to `0xFFFFFFFF`, masking gives `0x7FFFFFFF`, and the index is `0x3FFFFFFF`.

4. **Distinct final self-address expressions.** If the stack pointer is numerically `0x2000` and the separately captured signed global is `0x1000`, the value initially assigned to the scaled-pointer global is `0x8000`. The later captured-global self-address store targets `0x4000` and stores `0x4000`. The intervening stores still use actual reads of the scaled-pointer global, not an assumed cached `0x8000`.

5. **Complete signed-shift destination.** If the captured signed global is `0x1000`, and ordinary exact signed arithmetic applies to these values, shifting left by six gives `0x40000`; shifting that result right by 17 gives 2. The destination is therefore `0x1002`.

6. **Fresh reset-halfword read.** If the earlier extraction writes `0x1234`, but the reset-halfword’s current value is `0xABCD` when the final concatenation occurs, that concatenation produces `0xABCDEA70`, not `0x1234EA70`.

7. **Negative-address branch after mutation.** If F-e47d7b2e9c88’s address has bit 31 set, and preceding accesses complete under resolved arithmetic semantics, the halfword write at the derived base plus 4 precedes the invalid-instruction boundary. The branch’s later self-address writes are not reached.

8. **Partial overwrite.** At the stack-supplied pointer, the later 16-bit store changes only the addressed two bytes of the preceding 32-bit pointer-value store. Which numeric halfword remains untouched depends on endianness.

9. **Failure preserves prior effects absent external recovery.** If the initial write succeeds and the next read faults, no represented operation reverses that write.

## Uncertainties

- **documented_only — executable validity:** invalid-instruction warnings prevent proof of an executable full path.
- **documented_only — arithmetic:** oversized shifts, signed overflow, negative signed left shifts, and signed-right-shift behavior are unresolved.
- **documented_only — ambient state:** the incoming register base and stack-supplied pointer are not explained by the explicit interface.
- **documented_only — memory model:** endianness, alignment, mapping, device behavior, concurrency, and fault recovery are unspecified.
- **documented_only — global layout:** overlap is reported, but complete storage extents and some write widths are absent.
- **documented_only — operand evaluation:** no relative order is established between the two global reads forming the destination and value of the section-5 self-address store.
- **documented_only — intent and completion:** no high-level purpose, successful outcome, or recoverable failure protocol is established.

## Evidence

This contract uses only the supplied static representation: explicit types and accesses, arithmetic widths, constants, access ordering, captures versus repeated global reads, dependency-address references, concatenations, the overlap warning, and invalid-instruction boundaries.

The disputed final operations are distinguished directly: repeated reads of the scaled-pointer global; a separate self-address store derived from the captured signed global; the full left-six/right-seventeen destination calculation; and a fresh reset-halfword read for the final concatenation.

No implementation access, independent instruction analysis, memory map, dependency implementation, or runtime observation was available. No runtime tests have been performed.
