<!-- Trusted provenance: F-be4aec3be4dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be4aec3be4dc

## Purpose

F-be4aec3be4dc scans a byte sequence from a caller-supplied cursor, accumulates a decimal-weighted integer, and advances the caller’s cursor to the first byte rejected by its scanning condition.

The established stopping rule is broader than ordinary decimal parsing: **every byte from 0 through 57 inclusive is accepted; bytes from 58 through 255 stop the scan.** Decimal digits occupy only the upper portion of the accepted range. A zero byte does not terminate scanning.

Decimal-number parsing is inferred intent, not a complete description of the established behavior. The function performs no explicit validation of an initial digit, sign, string terminator, buffer length, or arithmetic range.

## Interface

### Input and output cursor

The sole argument identifies a caller-owned cursor slot. The supplied representation accesses this slot as a four-byte value, interprets its contents as the starting byte address, and ultimately replaces its contents with the stopping byte address.

The following are required for a defined, successful invocation:

- The cursor slot is readable at entry and writable when scanning finishes.
- Its initial contents identify readable byte-addressed storage.
- Every byte examined before stopping, including the stopping byte, is readable.
- The pointer representation and four-byte cursor representation are compatible in the execution environment.
- Address progression remains valid for the full scan.

There is no length argument and no explicit null-pointer check. The function does not establish these preconditions itself.

The slot remains owned by the caller. It is updated in place rather than replaced by newly allocated storage.

### Return value

The result is an integer accumulator initialized to zero.

The supplied representation uses a signed integer accumulator and return type, an unsigned integer conversion in the update expression, and a four-byte cursor slot. A conventional 32-bit interpretation gives:

- Eight-bit unsigned input bytes.
- A 32-bit signed accumulator and return value.
- A 32-bit unsigned arithmetic contribution during each accumulator update.
- A four-byte stored cursor representation.

The explicit four-byte slot and byte-sized reads are established by the supplied representation. The exact language implementation rules governing plain integer widths and out-of-range signed conversions are not independently supplied. Arithmetic claims requiring those rules are qualified below.

### Output conventions

There is no separate status result. The return value does not distinguish among:

- An empty accepted prefix.
- A prefix of zero digits.
- A longer accepted sequence whose arithmetic result is zero.
- Other accepted byte sequences that happen to produce zero under the applicable arithmetic rules.

The final cursor identifies the first rejected byte; the rejected byte is not consumed.

## Behavior

### Starting state and termination rule

The function first initializes its accumulator to zero and obtains the starting address from the caller’s cursor slot.

At each current address, it reads one unsigned byte and tests whether that byte’s numeric value, less 48, is strictly below 10.

For an eight-bit unsigned byte promoted to an ordinary integer capable of representing 0 through 255, the subtraction ranges from −48 through 207. Consequently:

- Values 0 through 57 satisfy the condition.
- Values 58 through 255 fail the condition.

There is no lower-bound test against 48. In particular, whitespace, punctuation below the digit range, and zero bytes can be accepted.

If the first byte is rejected, no accumulator update occurs. The function writes the starting address back to the cursor slot and returns zero. The write still occurs even though the cursor value does not change.

### Accumulation

For each accepted position, the function reads the byte again for use in the accumulator update.

The mathematical intent of the update is:

**new accumulator = 10 × previous accumulator + byte value − 48**

This equation describes the intended numerical transformation, but it does not by itself define every overflowing or out-of-range execution.

For ordinary decimal digits, while all relevant arithmetic remains representable, each accepted digit extends the accumulated decimal value. Leading zero digits are consumed and do not change the accumulated value.

Accepted bytes below 48 contribute negative mathematical digit values. They are not skipped and are not handled as separators or signs. Their presence can therefore alter the accumulator rather than merely alter where parsing starts.

After updating the accumulator, the function advances its working cursor by one byte and repeats the test.

### Arithmetic boundaries

The accumulator update mixes signed multiplication with unsigned arithmetic:

- Multiplication of the previous signed accumulator by ten occurs before its combination with the unsigned byte contribution.
- Conversion to unsigned arithmetic does not retroactively make an overflowing signed multiplication well-defined.
- Under a 32-bit unsigned interpretation, the unsigned addition and subtraction are modulo 2³².
- Assigning an unsigned result outside the signed accumulator’s representable range back to that accumulator requires implementation-specific interpretation.

Accordingly, the source does **not** justify an unconditional contract of signed decimal arithmetic, saturation, checked overflow, or modulo-2³² parsing.

For a 32-bit signed accumulator, multiplying by ten is representable only when the preceding accumulator lies from −214,748,364 through 214,748,364 inclusive. Even when that multiplication is representable, the subsequent unsigned calculation can produce a value outside the signed result range.

No explicit overflow detection, range error, or recovery behavior is present. Results beyond the established arithmetic domain are documented_only unless the execution environment’s integer semantics are supplied.

### Completion and cursor publication

When the condition rejects a byte, the function writes the address of that byte into the caller’s cursor slot and returns the current accumulator.

The caller-visible cursor is not advanced incrementally during the scan. There is one explicit final cursor-slot write after a rejecting byte has been found.

A zero byte is accepted, so a conventional zero-terminated string is not sufficient by itself to bound the scan. Unless a byte of at least 58 is eventually encountered, scanning continues beyond that terminator.

## Side effects

### Reads

The source-level access order is:

1. Read the initial cursor value from the caller’s slot.
2. Read the current byte to decide whether scanning continues.
3. For an accepted position, read that position again to obtain the accumulation contribution.
4. Advance to the next byte and repeat.
5. On rejection, stop without a second accumulation read of that position.

Thus, for a stable sequence containing an accepted prefix of length *n*, there are nominally two byte-read evaluations for each accepted position and one for the first rejected position.

These are source-level evaluations, not a guarantee of distinct physical memory transactions. The supplied accesses have no explicit volatile qualification or synchronization. Compiler transformations, concurrent mutation, and memory-mapped device behavior are not specified.

If memory changes between the test read and the accumulation read, the source does not require the second value to equal the first. Stable readable input is the safe precondition for the ordinary parsing interpretation.

### Writes

The only explicit external write is the final replacement of the caller’s cursor value with the stopping address.

The scanned bytes are not directly modified. If the cursor slot overlaps the scanned storage, however, the final cursor write can modify bytes in that storage. No non-aliasing requirement or overlap handling is expressed.

### Ownership and failure effects

There is no visible allocation, deallocation, ownership transfer, or external call. The function does not release the cursor slot or scanned storage.

There is no explicit error-return path. Invalid memory, invalid address progression, or unsupported arithmetic can prevent normal completion. No cursor update or return value is promised when execution fails before the final write.

The function does not provide bounded termination: readable storage alone is insufficient unless a rejecting byte is eventually reached.

## Dependencies

F-be4aec3be4dc has no visible function-call dependencies.

Its behavior depends on execution-environment properties rather than identified callees:

- Representation and validity of the stored cursor address.
- Readability and stability of the traversed byte storage.
- Writability of the cursor slot.
- Integer widths, signed-overflow treatment, and unsigned-to-signed conversion rules.
- Validity of successive one-byte address advances.

No dependency contract can be used to infer additional validation, memory ownership, or global-state preservation.

## Acceptance examples

These examples assume stable readable storage, a valid writable cursor slot, and compatible pointer representation. Decimal arithmetic examples also assume the conventional 32-bit interpretation described above and no out-of-range intermediate.

1. **Immediate rejection**

   The initial byte is 58, the value conventionally associated with a colon.

   Expected result: zero. The cursor still identifies that byte. The slot receives a write of the unchanged address, and the byte is not consumed.

2. **Ordinary decimal prefix**

   The sequence begins with byte values 49, 50, 51, then 58.

   Expected result: 123. The cursor advances three bytes and identifies the rejecting value 58.

3. **Leading zero digits**

   The sequence begins with 48, 48, 55, then 65.

   Expected result: 7. All three digit bytes are consumed; the cursor identifies 65.

4. **Zero-valued accepted prefix**

   The sequence begins with 48, 48, then 255.

   Expected result: zero. The cursor advances two bytes. This distinguishes the operation from immediate rejection only through the cursor output, not through the returned integer.

5. **Below-digit punctuation**

   The sequence begins with 47, then 58.

   Established scanning behavior: 47 is accepted, and the cursor advances to 58.

   The mathematical accumulator value is −1. A returned signed value of −1 is conditional on an implementation that converts the resulting maximum 32-bit unsigned value to −1. That signed conversion is not independently established here.

6. **Zero byte followed by a rejecting byte**

   The sequence begins with 0, then 58.

   Established scanning behavior: the zero byte is consumed, and the cursor advances to 58. The function does not stop at the zero byte.

   The mathematical accumulator value is −48; the exact signed return requires the applicable unsigned-to-signed conversion rule.

7. **Largest positive 32-bit signed decimal value**

   The bytes encode decimal 2147483647 and are followed by a rejecting byte.

   Under the conventional 32-bit interpretation, the arithmetic remains representable and the result is 2,147,483,647. The cursor identifies the following rejecting byte.

8. **Decimal value beyond that signed range**

   The bytes encode decimal 2147483648 and are followed by a rejecting byte.

   The final signed multiplication is representable, but the update produces an unsigned value outside the signed result range. A particular negative return must not be required without the environment’s conversion rule.

9. **No rejecting byte within readable storage**

   Every readable byte encountered is at most 57.

   No successful termination is guaranteed. A zero byte does not resolve this case. No specific return, fault type, or final cursor value is established.

## Uncertainties

The following remain documented_only:

- Exact target integer and pointer implementation rules beyond the sizes indicated by the supplied representation.
- The signed result of assigning an out-of-range unsigned accumulator update.
- Behavior after signed multiplication overflow.
- Physical read counts or ordering beyond the source-level evaluations.
- Concurrent modification, memory-mapped input, and unsynchronized observer behavior.
- Fault type, diagnostics, or recovery for inaccessible memory.
- Any caller-enforced length, terminator, digit-only, or arithmetic-range guarantees.

The likely intent is decimal-prefix conversion with cursor advancement. The missing lower bound is established behavior, not a justified basis for silently restricting accepted input to decimal digits.

## Evidence

This contract is based solely on the supplied static function representation.

Established evidence includes accumulator initialization, the byte acceptance expression, the separate accumulation read, one-byte cursor progression, the final cursor-slot write, and return of the accumulator. No external call, explicit bounds check, allocation, release, or error branch is visible.

The accepted range of 0 through 57 follows from the unsigned-byte range and the signed subtraction used by the condition. Decimal parsing intent is inferred from the factor of ten and the subtraction of 48.

No runtime tests have been performed. The acceptance examples are conditional contract examples derived from static evidence, not reported execution results.
