<!-- Trusted provenance: F-ab50dcb319e7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ab50dcb319e7

## Purpose

F-ab50dcb319e7 forwards an uninterpreted 32-bit argument into an ordered sequence of unresolved indirect calls, transforms call-derived values, performs mixed-width memory mutations, and reaches a final unresolved indirect transfer.

The supplied static material does not establish an application-level purpose. Recurring offsets and character-like constants do not establish text formatting, object initialization, or a named protocol.

This contract distinguishes:

- **Established static evidence:** represented operations, widths, value dependencies, and access ordering.
- **[INFERENCE]:** conclusions conditional on fixed-width 32-bit machine arithmetic.
- **documented_only:** behavior unresolved by the supplied material.

Observable ordering below means ordering represented by the static material. It does not establish hardware memory-ordering guarantees.

## Interface

### Explicit argument and completion

| Item | Contract |
|---|---|
| Argument | One uninterpreted 32-bit value. |
| Initial use | Forwarded unchanged as the first indirect call’s explicit argument. |
| Later use | No subsequent direct use is established; effects of the initial call may influence everything that follows. |
| Declared result | No explicit result value. |
| Completion | A return is represented after the terminal indirect transfer. Completion requires all reached calls and that transfer to return. |

The argument is not established to be a pointer, count, handle, or owned resource. No validation, null check, range check, or error-result convention is represented.

### Implicit inputs and execution requirements

Execution additionally depends on incoming link-related machine state, unresolved call targets, primary and secondary call results, global storage, and memory at fixed and computed addresses.

For uninterrupted execution, reached targets must support execution and reached memory must support the represented access widths. Some accesses involve low addresses, computed pointer-like values, overlapping locations, or potentially unaligned words. These are execution requirements, not conditions checked by the function.

No buffer extent, allocation size, alignment guarantee, or ownership annotation accompanies the interface.

## Behavior

### Call-mediated preparation

Before the direct-memory phase, the function makes a long ordered sequence of indirect calls. The initial target depends on incoming machine state. Later target expressions use embedded address-like values with a recurring displacement of 0xFFE. Concrete dependency identities remain unresolved.

Calls have zero through four represented explicit argument words. Absence of explicit arguments does not establish absence of implicit inputs or a complete calling convention.

Established data transformations include:

- Separating 64-bit results into low and high 32-bit words.
- Shifting primary and secondary result words.
- Saving values across intervening calls.
- Passing literal zeros, repeated arguments, and values that remain zero through shifts.
- Masking fields before repositioning or extraction.

For the first represented 64-bit result, the next call receives four words: the low word shifted left by 8, the high word unchanged, the high word shifted left by 6, and the low word shifted left by 14. A later single-word result is forwarded shifted left by 8.

These are word transformations, not evidence of a serialized format or memory byte order. Calls must remain ordered effect boundaries: unknown callees may mutate memory or globals, retain arguments, or fail to return. Equal-looking arguments do not make calls interchangeable.

### Widths and arithmetic

The representation contains 8-, 16-, 32-, and 64-bit quantities. Explicit byte reads through unsigned-byte accesses produce values from 0 through 255; their explicit conversion to unsigned 32-bit values zero-extends them. This rule does **not** establish the width or extension behavior of an undeclared global.

Unsigned masks select fields of the stated widths: 0xFFFFFF selects 24 bits, 0x3FFFFF selects 22, 0xFFFFF selects 20, 0x7FFFF selects 19, and 0x3FFFF selects 18. Narrower masks also occur. Unsigned right shifts zero-fill. Unsigned 64-bit right shifts by 34, 36, and 44 extract portions above the low result word.

For example:

- A 20-bit mask followed by a right shift of 10 yields at most 1023. Subsequent left shifts by 6 or 5 yield maxima of 0xFFC0 or 0x7FE0.
- A 24-bit field shifted left by 6 has maximum 0x3FFFFFC0.
- A 22-bit field shifted left by 8 has maximum 0x3FFFFF00.

**[INFERENCE]:** Under fixed-width machine arithmetic, each 32-bit operation retains only its low 32 bits. Intermediate truncation matters: consecutive left shifts by 28 and 28 do not preserve a 64-bit result.

Some signed-looking shifts, products, and address calculations can overflow. Their executable semantics remain **documented_only**; the contract does not treat them as arbitrary-precision arithmetic or claim portable signed-overflow behavior.

### Values used in the memory phase

The following descriptive labels identify saved values, not object types:

| Label | Definition |
|---|---|
| Packed base | A saved single-word call result multiplied by 2²⁵. |
| High-scale token | The low word of a different, earlier 64-bit result shifted left by 25. |
| Shifted token | A later single-word call result shifted left by 8. |
| Small index | A saved result shifted left by 8, masked with 0x7FFF, then shifted right by 6. |
| Small base | Twice the small index. |
| Terminal pair | The final 64-bit result before direct stores. |
| Global-derived base | The unsigned 32-bit conversion of the global associated with address 0x05. Its underlying type, read width, and extension behavior are unresolved. |
| Prior word | The 32-bit value read from global-derived base plus 0x54 before that location is replaced. |

**[INFERENCE]:** With 32-bit shifts, the small index ranges from 0 through 508 in multiples of four; the small base ranges from 0 through 1016 in multiples of eight. The preceding left shift makes the mask’s nominal range unattainable in full.

The global-derived base is not established to lie between 0 and 255. It must not be conflated with the explicit unsigned-byte read at packed base plus 0x05.

### Opening mutations and low-address state

The opening stores link terminal-result words, the high-scale token, and the packed base. A word at terminal high word plus 0x14 receives the high-scale token. A byte at high-scale token plus 0x09 first receives the terminal high word’s low byte. A word at 256 times terminal low word plus 0x54 then receives the packed base, before the token-relative byte is overwritten with the high-scale token’s low byte.

Next, packed base plus 0x14 is cleared; an explicit unsigned byte is read at packed base plus 0x05; small base plus 0x14 is cleared; and the saved byte is zero-extended into the word at packed base plus 0x34. The word at packed base plus 0x14 is then read and saved. Small base plus 0x14 is cleared again, followed by two byte stores of the saved word’s low byte at small base plus 0x15.

The subsequent low-address and base-dependent changes have these ordering constraints:

- The global associated with 0x05 is read and converted to unsigned 32 bits **before** the byte at fixed address 0x01 receives the small base’s low byte.
- After that fixed byte write, global-derived base plus 0x15 receives the small index’s low byte.
- Packed base plus 0x11 receives the shifted token’s low byte; the shifted token is stored as a word at the previously saved word plus 0x54; then packed base plus 0x11 is cleared.
- The prior word is read at global-derived base plus 0x54 before that location receives the earlier saved packed-relative word.

No universal substitution of zero is valid for the packed-relative word read merely because an earlier store cleared that location. Intervening accesses can alias it.

### Linked and scaled mutations

The next region uses loads through offsets 0x74, 0x64, and 0x04 to derive further destinations. It includes self-reference word stores at offsets 0x24 and 0x14, repeated byte stores at offset 0x09, and word replacements at offset 0x54.

One expression reads small base plus 0x74 twice while constructing a self-reference store through the loaded values. The two reads are not established to be indivisible or equal. Their relative expression-evaluation order is not independently resolved.

Two fixed-address mutations occur within this region:

| Fixed mutation | Ordering and value |
|---|---|
| Word beginning at 0x54 | Receives the prior word shifted left by one. This follows a store at small base plus 0x14 derived from the packed-base source result shifted left by 26, and precedes two stores of the prior word at small base plus 0x14. |
| Byte at 0x15 | Receives the small base’s low byte. This follows two byte stores at small base plus 0x15 of the low byte of four times the prior word, and precedes the following scaled word-store region. |

These fixed writes can alias computed destinations; they are not independent metadata.

Further accesses use four times and eight times the prior word, twice values loaded through offset 0x74, and zero-extended bytes used as bases for later word loads. Updates include:

- Word stores at offsets 0x14, 0x34, 0x44, and 0x54.
- Byte stores at offsets 0x09, 0x0D, and 0x15.
- Fresh byte reads at offset 0x05, including repeated reads separated by writes.
- A word read at offset 0x64 of a computed base.

Recurring offsets do not establish a uniform structure layout.

### Literal stores and repeated groups

A later loaded link value determines a **doubled link base**, twice that value, and a **scaled link**, four times that value. The scaled link receives byte 0x3A; twice the scaled link receives halfword 0x003A; and scaled link plus 0x20 receives word 0x0000003A.

A separate **adjusted scalar** is four times the prior word minus 0x96. Four times this scalar forms the **working base**. At working base plus 0x20, four word stores occur in order:

1. 0x00000020.
2. 0x0000002F.
3. The zero-extension of a freshly read byte at working base plus 0x04.
4. 0x00000020 again.

After these stores, a stored 32-bit base is read once. **That read precedes the fixed word write of 0x00000020 beginning at address 0x60.** The saved base, rather than a reload after the fixed write, drives the repeated region.

Each repeated group contains, in represented order, a scaled-destination word store of the scaled link, a byte store at stored base plus 0x19 of the working base’s low byte, a byte store at working base plus 0x11 of the stored base’s low byte, and a word store at stored base plus 0x34.

The first group differs: its scaled destination uses eight times the prior word, and its final word contains eight times the prior word. Subsequent groups use eight times the adjusted scalar for those two roles.

Distinct represented repetitions must not be collapsed into one assignment. An independently verified exact group count is **documented_only**.

### Final phase and fresh global reads

After the repeated region, a 32-bit word beginning at fixed address 0xA9 is read and saved. A word at scaled link plus 0x74 is then read and saved. **After those reads**, the byte at fixed address 0x3A receives the low eight bits of eight times the adjusted scalar plus 0x30. This uses the scaled scalar, not the low-address destination as a source value.

Words at doubled link base plus 0x14 and working base plus 0x04 are saved next. Subsequent mutations include byte stores at offsets 0x08, 0x09, and 0x0D, including literal 0x6C at working base plus 0x08, and a self-reference word store at doubled link base plus 0x54.

The byte at fixed address 0x67 receives the low byte of the saved full word from 0xA9. Three later uses are **fresh reads of the byte at 0x67**, not uses of that saved word:

| Reload | Destination and intervening activity |
|---|---|
| First | Supplies the byte write at fixed address 0x21, after a word read at doubled link base plus 0x44. |
| Second | Supplies a byte write at working base plus 0x09, after multiple computed writes, a read at working base plus 0x40, and a word write overlapping an earlier byte write at working base plus 0x10. |
| Third | Supplies another byte write at working base plus 0x09, after a byte write at doubled link base plus 0x09, a full-word store at working base plus 0x44, and a word read at working base plus 0x54. |

The intervening full-word store at working base plus 0x44 uses the **saved full 32-bit word from 0xA9**. It does not reconstruct that word from a reloaded byte. That destination is subsequently overwritten with another saved word.

Computed writes can alias 0x67. Without non-aliasing and stability guarantees, the three reloaded bytes need not equal each other or the byte originally stored there.

Another byte write, at doubled link base plus 0x01, derives from a signed-looking right shift of a loaded word. The count is the working base’s low eight bits and can exceed 31. Shift behavior for such counts, and signed right-shift behavior, remain **documented_only**.

### Terminal mutations and transfer

The final updates include a halfword store of 0x0065 at twice a newly loaded base, a word store of decimal 100 at another computed base plus 0x54, and word loads through offsets 0x64 and 0x74. A further offset-0x64 load determines a destination whose offset 0x54 receives the earlier offset-0x64 result. A word at twice that earlier result receives the later loaded base.

The terminal indirect transfer receives four represented argument words:

- 0x72.
- The earlier result loaded through offset 0x64.
- The saved word read at doubled link base plus 0x14 near the start of the final phase.
- The later word loaded through offset 0x74 in the terminal mutation group.

The final argument is not the earlier saved scaled-link-relative word. Target identity, calling convention, and returning behavior remain unresolved.

## Side effects

The footprint includes unresolved callee effects and all represented direct accesses.

| Access class | Established locations |
|---|---|
| Explicit unsigned-byte reads | Computed offsets 0x04 and 0x05. |
| Fixed byte reads | Three final-phase reloads at 0x67. |
| Unresolved-width global read | Global associated with 0x05, converted to unsigned 32 bits. |
| Byte writes | Computed offsets 0x01, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x15, 0x19; fixed addresses 0x01, 0x15, 0x21, 0x3A, 0x67; scaled locations. |
| Halfword writes | Scaled locations receiving 0x003A and 0x0065. |
| Word reads | Computed offsets 0x04, 0x14, 0x40, 0x44, 0x54, 0x64, 0x74; fixed word at 0xA9; stored-base storage. |
| Word writes | Computed offsets 0x10, 0x14, 0x20, 0x24, 0x34, 0x44, 0x54, 0x74; fixed words at 0x54 and 0x60; scaled locations. |

Offsets are byte offsets; decimal 100 is 0x64. Overlapping byte and word accesses are not independent fields.

No allocation, release, ownership transfer, borrowing rule, or lifetime extension is established. Unknown callees must not be assumed to preserve globals or own or release memory.

No rollback, synchronization, transaction, or failure cleanup is represented. Earlier effects can remain after a fault or nonreturning call. Device semantics, atomicity, and concurrent-observer guarantees are **documented_only**.

## Dependencies

The supplied opaque identity map identifies only F-ab50dcb319e7. It supplies no distinct opaque IDs for callees; no dependency identities are invented here.

| Unresolved boundary | Established contract |
|---|---|
| Initial indirect call | Receives the explicit 32-bit argument unchanged. |
| Intermediate calls | Ordered effect boundaries with represented word transformations and saved-result dependencies. |
| Terminal transfer | Receives the four words specified above. |

Implementations, implicit inputs, secondary-result conventions, global effects, resource behavior, and failure behavior remain unresolved.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Forwarding:** Given input 0x89ABCDEF and a reachable initial call, the first explicit argument is exactly 0x89ABCDEF.
- **Result transformation:** Given first 64-bit result low word 1 and high word 2, the next four arguments are 0x100, 2, 0x80, and 0x4000.
- **Small-index boundary:** Under fixed-width shifts, a relevant saved result with low seven bits 0x7F produces small index 508 and small base 1016, not index 511.
- **Explicit byte extension:** If the explicit unsigned-byte read at packed base plus 0x05 yields 0xE5, its corresponding word store contains 0x000000E5. This example does not constrain the undeclared global at fixed address 0x05.
- **Global-derived base:** If conversion of that global yields 0x1234, later base-relative accesses use 0x1234; no unsupported byte narrowing to 0x34 is permitted.
- **Fixed mutations:** Under nonoverflowing arithmetic, prior word 3 produces word 6 at fixed 0x54; small base 8 produces byte 8 at fixed 0x15; adjusted scalar 5 produces byte 0x58 at fixed 0x3A. Their phase ordering must remain intact.
- **Stored-base snapshot:** If the fixed write at 0x60 aliases stored-base storage, subsequent repeated groups still use the base saved before that write.
- **Reload distinction:** If the three reads at 0x67 yield 0x11, 0x22, and 0x33, those values respectively supply the 0x21 write and the two later working-base-relative byte writes. The intervening full-word store still uses the saved word from 0xA9.
- **Overwrite trace:** With stable ordinary memory, the four working-base-plus-0x20 stores finish at 0x20, but the intervening stores and fresh byte read remain represented accesses.
- **Inaccessible destination:** No successful completion, structured error return, or rollback is promised.

## Uncertainties

The following remain **documented_only**:

- Domain purpose and argument meaning.
- Accuracy of the representation relative to an executable.
- Callee identities, contracts, and complete calling conventions.
- The underlying type, read width, and extension behavior of the global at 0x05.
- Signed-overflow, address-wrap, and variable-shift semantics.
- Endianness, alignment support, mappings, aliases, and concurrency.
- Independently verified repetition count and complete call-by-call trace.
- Whether the final transfer returns.

These limitations prevent treating this document as a complete executable replacement specification.

## Evidence

The supplied static material is the sole evidence. It establishes call-dependent transformations, mixed-width accesses, the ordered fixed-address mutations, the stored-base read before the 0x60 write, and the distinction between fresh byte reads at 0x67 and the saved full word from 0xA9.

The global at 0x05 lacks a supplied declaration; its unsigned 32-bit conversion alone does not establish a byte load or zero-extension.

No implementation access, dependency inspection, or runtime tests have been performed. Acceptance examples are derived conditions, not observed results. No snapshot hash or independent executable provenance is asserted.
