<!-- Trusted provenance: F-b9ad687f6e4d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9ad687f6e4d

## Purpose

F-b9ad687f6e4d scans an optional leading minus sign, an optional hexadecimal prefix, and a table-defined sequence of bytes. It accumulates a value with signed 16-bit wraparound semantics and returns that value sign-extended to the integer return width. When requested, it also reports the address of the first byte not accepted by the selected classification table.

The apparent intent is conversion of textual octal, decimal, or hexadecimal values. That intent is inferred from the radix choices and prefix handling. The actual accepted bytes and their numeric contributions depend on external tables whose contents are not supplied. The contract therefore does not assume conventional digit recognition.

The function does not report overflow, distinguish “no digits” from a converted zero, or require consumption of the whole input.

## Interface

The interface has four arguments:

| Argument | Role |
|---|---|
| Input address | Address of a readable byte sequence, or null. Bytes are treated as unsigned 8-bit values. |
| Stop-address destination | Optional address of a writable 32-bit slot receiving the stopping input address. Null suppresses this write. |
| Requested radix | Integer selecting octal when equal to 8, hexadecimal when equal to 16, and decimal otherwise. |
| Auxiliary unsigned value | Replaced by the first input byte whenever the input address is nonnull. It has no effect on any observed result. |

The return value is a signed integer carrying a sign-extended 16-bit result. The supplied representation indicates a 32-bit integer and address model: the stop-address destination is written as a four-byte quantity. This is a target-specific interface description, not a portable pointer-size guarantee.

### Preconditions

For a defined, successful execution:

- Every input byte actually examined must be readable.
- If the hexadecimal-prefix check reaches its second byte, that byte must also be readable.
- The selected classification-table entries must be readable.
- For each accepted byte, the corresponding conversion-table entry must be readable.
- A nonnull stop-address destination must support the observed four-byte address store.
- The relevant global table-address slots must be readable.

The function has no length argument and performs no bounds checks. A zero byte stops scanning only if the selected classification table rejects it. Consequently, a conventional null-terminated string is sufficient only under an appropriate table configuration.

No allocation, ownership transfer, or release is established.

## Behavior

### 1. Initial input handling

If the input address is null, the function returns zero. It does not read input, consult the table-address slots, or write the stop-address destination.

For a nonnull input address, it reads the first byte and uses that byte in place of the auxiliary unsigned argument.

If that first byte is zero, the function immediately returns zero. The stop-address destination is left untouched, even when nonnull. This differs from the later case in which scanning stops at a rejected byte: that later path does write the stopping address.

The first byte is also retained as the sole basis for deciding whether a minus sign was present. Later input contents do not change that decision.

### 2. Initial radix and classification-table selection

For a nonempty input:

- Requested radix 8 selects radix 8 and the octal classification-table address.
- Requested radix 16 selects radix 16 and the hexadecimal classification-table address.
- Every other requested value selects radix 10 and the decimal classification-table address.

There is no error for unsupported radix values. Values such as 0, 2, 10, negative integers, or other non-8/non-16 integers all follow the decimal-selection path.

The supplied representation shows the global address-slot reads in this order:

1. Read the octal classification-table address.
2. If the requested radix is not 8, read the hexadecimal classification-table address.
3. If it is also not 16, read the decimal classification-table address and use radix 10.

These reads occur before sign and prefix processing. Some loaded addresses are therefore unused. Reading an address slot is distinct from reading a byte through that address.

### 3. Leading minus handling

If the original first byte is `-`, with byte value 45, advance the scanning address by one byte.

This is the only explicit sign handling. There is no explicit leading-plus handling and no explicit whitespace skipping.

A plus sign, whitespace, or an additional minus sign encountered afterward is subject to ordinary table classification. It must not be assumed to be rejected without the table contents.

The minus sign is consumed even if no subsequent byte is accepted as a digit.

### 4. Hexadecimal-prefix handling

At the current scanning address, including after a consumed minus sign, check for the two-byte prefix `0x` or `0X`.

The prefix check:

- Reads the current byte.
- Reads the following byte only when the current byte is `0`, with byte value 48.
- Recognizes a following `x` or `X`, with byte value 120 or 88.

When recognized, the function:

- Changes the effective radix to 16, regardless of the requested radix.
- Advances the scanning address by two bytes.
- Reads the hexadecimal classification-table address again and uses that address for scanning.

The prefix is consumed without requiring a subsequent accepted byte. There is no corresponding explicit octal-prefix rule.

### 5. Table-defined scanning and accumulation

The accumulator begins at zero.

For each candidate byte, the function reads that byte and tests one bit in the selected classification table:

- The byte’s upper five bits select a classification-table byte, using an index from 0 through 31.
- The byte’s lower three bits select a bit within that table byte.
- A zero bit ends scanning.
- A nonzero bit accepts the byte.

This layout represents membership for all 256 possible unsigned byte values using 32 bytes of bit data. The actual membership is unknown.

For each accepted byte, the function advances the scanning address by one byte, then obtains an unsigned 8-bit numeric contribution from a separate conversion table indexed by the accepted byte’s full value. The conversion-table base address is read from its global slot for each accepted byte.

The accumulator transition is:

\[
a_{\text{next}} =
\operatorname{signed16}\!\left(r \times a_{\text{current}} + d\right)
\]

where:

- \(r\) is the effective radix, always 8, 10, or 16;
- \(d\) is the unsigned conversion-table entry, from 0 through 255;
- \(\operatorname{signed16}\) retains the low 16 bits and interprets them as a signed two’s-complement value.

Thus each accepted byte updates an accumulator in the range −32768 through 32767. There is no saturation, overflow flag, or range error.

Membership and conversion are independent. No observed check requires the contribution to be smaller than the radix. An accepted byte may contribute any value representable by its conversion-table entry.

The selected classification-table address is retained throughout scanning. Its contents are read as scanning proceeds. The conversion-table address, by contrast, is fetched again for each accepted byte; the supplied representation does not establish a single conversion-table snapshot for the entire call.

### 6. Stop-address reporting and return

Scanning ends at the first byte whose membership bit is zero. That byte is examined but is not included in the accumulator, and the stopping address points to it.

If the stop-address destination is nonnull, the function writes that stopping address to the destination. This happens even if no byte was accepted after sign or prefix processing.

After the optional address store:

- If the original first byte was not a minus sign, return the signed 16-bit accumulator.
- If the original first byte was a minus sign, negate the accumulator, narrow the result to 16 bits, and return its signed interpretation.

On the represented two’s-complement target, negating −32768 therefore produces −32768 after narrowing. The final return is not a wider, unbounded negation.

No input or table reads are shown after the stop-address write.

## Side effects

The only explicit memory write is the optional four-byte stop-address store on the scanning path.

The function does not explicitly modify:

- The input sequence.
- Classification-table contents.
- Conversion-table contents.
- Global table-address slots.

It performs no visible allocation, deallocation, I/O, error-state update, or function call.

The stop-address destination is caller-provided storage. Nothing establishes that the returned address owns memory, transfers ownership, or extends the lifetime of the input.

No non-aliasing requirement is shown. If the destination overlaps readable input, a table, or a table-address slot, the terminal store may alter that storage. Because it occurs after scanning and table lookup have finished, it cannot change the already-computed result within this invocation. Whether such overlapping use is valid under the original language-level object model is not established.

Unreadable memory, an unwritable destination, invalid table addresses, and scanning beyond available input have no explicit recovery path. A defined error return for these conditions is not established.

## Dependencies

No called-function dependency is present in the supplied evidence. No additional opaque function IDs can be assigned from that evidence.

The function depends on five external data roles:

| Data role | Observed use |
|---|---|
| Octal classification-table address slot | Read during initial table selection. |
| Hexadecimal classification-table address slot | Read during applicable initial selection and again upon a recognized hexadecimal prefix. |
| Decimal classification-table address slot | Read when the requested radix is neither 8 nor 16. |
| Classification-table bytes | Determine which input bytes are accepted. |
| Conversion-table address slot and entries | Supply an unsigned byte contribution for each accepted input byte. |

The allocation, initialization, mutability, sharing, and lifetime of these data are unknown. No assumptions about ownership or preservation by other components are made.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Table-independent cases

1. **Null input:** For any requested radix, auxiliary value, and valid stop-address destination, return zero and leave the destination unchanged.

2. **Initially empty input:** When the input points to a zero byte, return zero and leave the destination unchanged. Classification tables are not consulted.

3. **Unsupported radix:** For a nonempty input without a recognized hexadecimal prefix, a requested radix of 2 selects the decimal classification path. It does not select binary conversion or return a radix error.

4. **Prefix overrides radix:** An input beginning with `0X` selects radix 16 and begins classification after those two bytes, even when the requested radix was 8.

5. **No length guarantee:** If the classification table accepts every byte encountered, the function has no intrinsic stopping condition based on input length.

### Cases conditional on conventional numeric tables

For the following examples, assume the selected tables accept the displayed digits, map them to conventional numeric values, and reject the indicated terminator.

| Input and condition | Required result |
|---|---|
| `123!`, decimal selection, `!` rejected | Return 123; report the address of `!`. |
| `17!`, requested radix 8, `!` rejected | Return 15; report the address of `!`. |
| `0x10!`, any requested radix, `!` rejected | Return 16; report the address of `!`. |
| `-0x10!`, `!` rejected | Return −16; report the address of `!`. |
| `32768!`, decimal selection, `!` rejected | Return −32768 because the accumulated value is narrowed to 16 bits. |
| `65536!`, decimal selection, `!` rejected | Return zero; report the address of `!`. |
| `-32768!`, decimal selection, `!` rejected | Return −32768 after final 16-bit negation. |

### Empty converted sequence

These cases depend only on rejection of the byte at the post-sign or post-prefix position:

- For `-` followed by a rejected zero byte, return zero and report the address of that zero byte, one byte after the original input.
- For `0x` followed by a rejected zero byte, return zero and report the address two bytes after the original input.
- For `-0X` followed by a rejected zero byte, return zero and report the address three bytes after the original input.
- For a nonempty input whose first byte is neither specially consumed nor accepted, return zero and report the original input address.

Unlike an initially empty input, these paths write a nonnull stop-address destination.

## Uncertainties

The following are **documented_only**:

- **Actual accepted alphabet:** Classification-table contents are absent. Conventional digits, whitespace rejection, plus-sign rejection, and zero-byte rejection during scanning are not established.
- **Actual numeric mappings:** Conversion-table contents are absent. Conventional digit values are inferred intent, not unconditional behavior.
- **Original source-language arithmetic guarantees:** The reconstructed expression uses a multiply-and-shift form to retain and sign-extend 16 bits. Its intermediate multiplication can exceed signed 32-bit range. The modular contract above describes the represented target behavior; it is not a claim that the reconstructed expression has portable signed-C semantics.
- **Narrowing rules on another target:** The observed signed 16-bit behavior depends on the represented target’s narrowing and signed-shift semantics.
- **Concurrent mutation:** No synchronization is shown. Behavior under concurrent changes to input, tables, or global address slots is not established.
- **Memory and calling-convention details:** Full ABI requirements, alignment constraints, accessible memory extents, and fault behavior are not supplied.
- **Termination:** No upper bound on consumed bytes is established.

## Evidence

The supplied static body establishes the null and initial-zero exits, radix-selection branches, leading-minus consumption, case-insensitive hexadecimal prefix, bitset membership layout, byte-indexed contribution lookup, 16-bit accumulator narrowing, optional stopping-address store, and final sign handling.

The numeric-parser purpose and conventional table meanings are inferred from structure and constants. They are not substitutes for the missing table contents.

No runtime tests have been performed. The acceptance examples state conditional expected behavior, not observed execution results.
