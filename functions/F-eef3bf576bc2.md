<!-- Trusted provenance: F-eef3bf576bc2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eef3bf576bc2

## Purpose

F-eef3bf576bc2 performs a byte-oriented, radix-dependent unsigned conversion and returns a 32-bit result. It optionally records a stopping address and, when its internal overflow condition is observed, invokes an error-state dependency and returns the maximum unsigned 32-bit value.

The routine directly recognizes an initial zero and conditionally recognizes a hexadecimal prefix. It delegates all subsequent character acceptance and numeric digit values to F-1a5edad330b3. Consequently, familiar integer-parsing behavior is conditional on that dependency’s contract, not established solely by this routine.

The established behavior includes:

- Default-radix selection from the initial bytes.
- Special treatment of an initial zero, including conversion-success bookkeeping without digit classification.
- Conditional consumption of an initial `0x` or `0X`.
- Accumulation using two unsigned 32-bit working values.
- Continued scanning after an overflow condition has been detected.
- An optional stopping-address write before overflow-error reporting.
- A fixed overflow return value of 4,294,967,295.

## Interface

### Inputs

**Input address**

An address from which the routine reads individual character-sized bytes. The initial byte is read unconditionally.

The routine does not check this address for nullness, validate the allocation, or receive a length. Readable storage must extend through every byte the routine attempts to inspect. A null-terminated string is a likely intended input, but termination at a zero byte depends on F-1a5edad330b3 rejecting that byte.

The routine itself does not skip whitespace or consume a leading sign.

**Optional stopping-address destination**

Either a null address or the address of one writable 32-bit slot used to hold an input address.

When non-null, this slot is written once after scanning terminates. The stored value is:

- The original input address if the routine has not recorded any accepted conversion content.
- Otherwise, the address of the byte most recently rejected by F-1a5edad330b3.

The observed representation uses a four-byte destination slot and stores an address into it. This is a 32-bit-address interface; behavior under a different pointer representation is not established.

**Requested radix**

A signed 32-bit integer. Zero requests automatic selection. There is no explicit validation of the remaining values.

After initial-byte processing, the selected radix is passed to F-1a5edad330b3 and participates in unsigned 32-bit arithmetic. Negative radices and unsupported positive radices therefore must not be assumed to produce conventional numerical conversion.

### Output

The return value is an unsigned 32-bit integer.

- If the overflow flag was never set, the return value is assembled from the two accumulator components.
- If the overflow flag was set at any point, the return value is 4,294,967,295.

This maximum value is not, by itself, an unambiguous error indication: it can also be the successful conversion of the largest representable unsigned 32-bit value.

### Preconditions

For a normally completing invocation:

- Every input byte actually read must be accessible.
- A non-null stopping-address destination must permit a four-byte write.
- F-1a5edad330b3 must eventually return a negative signed result.
- If overflow reporting is reached, F-4af36158f863 must return an address permitting the subsequent four-byte write.

These conditions are not checked locally. No local recovery behavior is established for their violation.

## Behavior

### Initial reads and radix selection

The first input byte is fetched before any digit-classification call.

If that byte is not `0`:

- A requested radix of zero becomes decimal, radix 10.
- Any other requested radix is retained.
- The first byte is offered to F-1a5edad330b3.

If the first byte is `0`, the routine immediately:

- Records that conversion content has been accepted.
- Reads the second byte.
- Omits classification and accumulation of the initial zero.

Processing then depends on the second byte and requested radix.

| Second byte | Requested radix | Result |
|---|---:|---|
| Neither `x` nor `X` | 0 | Select radix 8; classify the second byte next. |
| Neither `x` nor `X` | Nonzero | Retain the requested radix; classify the second byte next. |
| `x` or `X` | 0 or 16 | Select radix 16, read the third byte, and classify that byte next. Reset the accepted-content state to false. |
| `x` or `X` | Any other value | Retain the requested radix and the accepted-content state from the initial zero. Classify the `x` or `X` next. |

Thus a recognized hexadecimal prefix is not sufficient by itself to count as a conversion. At least one later classification must succeed. By contrast, an initial zero not followed by a recognized hexadecimal prefix is sufficient to count as a conversion, even if the next byte is immediately rejected.

An unsupported radix does not prevent the initial-zero shortcut. For example, the initial zero can establish conversion success without ever being submitted to the digit dependency.

### Classification and termination

F-1a5edad330b3 is called with the current character and selected radix.

Its signed result controls scanning:

- Any nonnegative result is accepted as a numeric contribution.
- Any negative result terminates scanning.

The routine does not independently verify that an accepted contribution is less than the radix, fits in a byte, or corresponds to an ordinary digit.

There is no separate local stopping rule for a zero byte. If the dependency accepts a zero byte, scanning continues past it.

The first negative result is retained for the possible overflow-reporting call. Its exact value is not otherwise interpreted locally.

### Accumulation and integer widths

The routine uses two unsigned 32-bit working quantities:

- A low component, reduced to its bottom 16 bits after each accepted character.
- A high component, updated using the carry extracted from the low-component calculation.

Both begin at zero.

For each accepted numeric contribution, the mathematical update is:

1. Compute an intermediate low value as the previous low component multiplied by the selected radix, plus the accepted contribution, modulo \(2^{32}\).
2. Update the high component as its previous value multiplied by the selected radix, plus the upper 16 bits of that intermediate low value, modulo \(2^{32}\).
3. Record that conversion content has been accepted.
4. Read the next input byte.
5. If the updated high component exceeds 65,535, set the overflow flag.
6. Retain only the bottom 16 bits of the intermediate low value.

The selected radix participates through its unsigned 32-bit representation. The accepted contribution is nonnegative as a signed 32-bit value and is then incorporated into the unsigned calculation.

The arithmetic is not an arbitrary-precision computation. In particular, the intermediate low calculation can itself wrap before its upper 16 bits are extracted. The high calculation can also wrap.

For ordinary positive radices and conventional digit values, this arrangement supports a 32-bit conversion and detects growth beyond the representable range. For unconstrained radix and dependency-result combinations, the contract is the arithmetic above, not a general promise to detect every mathematical overflow.

### Overflow handling

The overflow flag is sticky. Once set, later arithmetic cannot clear it.

Detection does not terminate scanning. The routine continues classifying accepted characters and reading subsequent bytes until the dependency returns a negative result.

This has two observable consequences:

- The stopping address identifies the eventual rejected byte, not necessarily the character at which overflow first became detectable.
- An overflowed input can still cause further reads and dependency calls, including reads outside available storage if the caller’s readable-input precondition is not met.

The next input byte is read before the high-component overflow check for the preceding accepted contribution. A fault or interruption during that read can therefore prevent completion of that iteration’s local overflow update.

### Stopping-address publication

After classification terminates, the optional stopping-address destination is processed before any overflow-reporting call.

If accepted content was recorded, the routine writes the address of the rejected byte. Otherwise, it writes the original input address.

In particular:

- An ordinary first-byte rejection reports the original address.
- An initial zero followed by an immediately rejected second byte reports the second-byte address.
- A recognized hexadecimal prefix followed immediately by a rejected third byte reports the original address, not the prefix’s end.

A null destination suppresses this write but does not suppress scanning, overflow detection, error reporting, or the returned numeric result.

### Final return and error-state write

If no overflow was recorded, the return value is formed by bitwise OR of:

- The final low component.
- The final high component shifted left by 16 bits within 32-bit arithmetic.

If overflow was recorded:

1. The optional stopping-address write has already occurred.
2. F-4af36158f863 is called with the negative result that ended classification.
3. The routine writes the 32-bit value 2 through the address returned by that dependency.
4. The routine returns 4,294,967,295.

There is no local check for a null or invalid returned error-state address.

The non-overflow path does not invoke F-4af36158f863 and does not directly clear or initialize an error-state location.

## Side effects

### Direct memory accesses

The routine directly performs:

- Sequential byte reads from the input.
- At most one four-byte write to the optional stopping-address destination.
- On overflow, one four-byte write of value 2 through the address returned by F-4af36158f863.

It performs no direct input writes, allocation, or deallocation.

### Observable ordering

The important ordering constraints are:

- Prefix-related byte reads occur before the first classification call.
- Each classification call precedes accumulation for that character.
- Each accepted character causes the following byte to be read before the local overflow check.
- The next classification call occurs only after that iteration’s local arithmetic and bookkeeping finish.
- The stopping-address write follows the terminating classification call.
- The stopping-address write precedes the overflow-reporting dependency call.
- The error-state write follows that dependency’s return.

Input reads are not a snapshot. Dependencies may affect bytes that have not yet been read. A byte already fetched for a classification call is not explicitly reread before that call.

### Aliasing and ownership

No ownership transfer is established for any supplied address. The routine does not directly release memory.

No disjointness check exists between the input, stopping-address destination, and returned error-state address. If they overlap, the writes occur in the order described above. In particular, an error-state write can overwrite storage written earlier as the stopping-address destination.

No preservation, allocation, or release guarantees are established for either dependency.

## Dependencies

### F-1a5edad330b3

**Observed role:** Classifies one character under the selected radix and returns a signed integer.

**Locally enforced interpretation:**

- Negative: terminate scanning.
- Nonnegative: accept the result as the numeric contribution.

**Unknown contract:**

- Recognized characters and supported radices.
- Whether zero bytes, signs, whitespace, or non-ASCII bytes are rejected.
- Whether accepted values are bounded by the radix.
- Behavior for negative or otherwise unsupported radices.
- Global-state changes, memory writes, ownership effects, and exceptional behavior.

The exact signedness and extension of non-ASCII input character values at the call boundary are not established sufficiently to specify their classification.

### F-4af36158f863

**Observed role:** On the overflow path, receives the terminating negative classification result and returns an address through which value 2 is written.

The meaning of that location and the meaning of value 2 are not established here. Calling it an error-state accessor describes its observed use, not a proven external API identity.

Its side effects, storage duration, ownership behavior, and dependence on the negative argument remain unknown.

## Acceptance examples

These are conditional acceptance examples, not executed tests.

For the ordinary numeric examples, assume F-1a5edad330b3 implements conventional digit values, accepts only digits valid for the selected radix, rejects a zero terminator and punctuation, and has no relevant interfering side effects. Offsets are zero-based byte offsets from the input address.

| Input bytes | Requested radix | Expected result | Expected stopping offset |
|---|---:|---:|---:|
| `123!` | 0 | 123 | 3 |
| `077!` | 0 | 63 | 3 |
| `09!` | 0 | 0 | 1 |
| `0x2a!` | 0 | 42 | 4 |
| `0X2A!` | 16 | 42 | 4 |
| `0x2a!` | 10 | 0 | 1 |
| `0!` | 0 | 0 | 1 |
| `0x!` | 0 | 0 | 0 |
| `!` | 10 | 0 | 0 |
| `-1!` | 10 | 0 | 0 |
| ` 1!` | 10 | 0 | 0 |

The `0x!` case illustrates the reset of accepted-content state after recognizing a hexadecimal prefix. The `0!` case illustrates the opposite rule: an initial zero alone establishes accepted content.

Under the same assumptions:

- `4294967295!` in radix 10 returns 4,294,967,295, stops at offset 10, and does not take the local overflow-reporting path.
- `4294967296!` in radix 10 returns the same value, stops at offset 10, invokes F-4af36158f863 after publishing that stopping address, and writes 2 through the returned address.
- `42949672960!` in radix 10 continues through the remaining digit after overflow becomes detectable, then stops at offset 11.
- Replacing a valid stopping-address destination with null leaves the specified return and overflow behavior unchanged, assuming dependencies do not introduce pointer-dependent interference.

A dependency-boundary example does not require conventional digit classification: if the requested radix is 1, the input starts with `0!`, and the dependency rejects `!`, the initial-zero shortcut still produces result zero and stopping offset 1.

## Uncertainties

The following are **documented_only**:

- Exact digit-recognition rules and valid radix range.
- Treatment of non-ASCII character bytes.
- Whether zero-terminated input is sufficient to guarantee termination.
- The external meaning and lifetime of the overflow-reporting destination.
- Whether error-state value 2 has a standardized meaning.
- Dependency effects on globals, input storage, other memory, or ownership.
- Behavior when reads or writes are inaccessible or dependencies fail to return normally.
- Portability beyond the observed 32-bit arithmetic and address-slot representation.

**Inferred intent:** An unsigned integer parser with radix autodetection and saturating overflow reporting.

**Established limitation:** That intent does not supply missing validation, whitespace handling, sign handling, dependency guarantees, or general overflow guarantees for arbitrary radix/result combinations.

## Evidence

The supplied static body establishes the initial-byte decisions, dependency-call conditions, accumulator widths and updates, sticky overflow flag, stopping-address selection, write ordering, and final return selection.

The supplied dependency mapping establishes the opaque IDs used here. No dependency implementation or external API contract was supplied.

No runtime tests have been performed. All acceptance examples are conditional predictions from the static evidence, not observed execution results.
