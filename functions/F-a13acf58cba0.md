<!-- Trusted provenance: F-a13acf58cba0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a13acf58cba0

## Purpose

F-a13acf58cba0 converts a descriptor-selected sequence of up to four bytes into an unsigned integer. Bytes are interpreted in most-significant-byte-first order. One accepted descriptor kind returns the accumulated value directly; the other returns its arithmetic negation in the unsigned result width.

The function also validates the descriptor kind and byte count. It returns an all-ones error value for an unsupported kind or a count greater than four. A null descriptor or a validated descriptor with a null payload address returns zero.

The result does not distinguish every successful conversion from an error. In particular, the all-ones result is also obtainable from accepted inputs.

These behaviors are established by static inspection of the supplied representation. No runtime tests have been performed.

## Interface

### Input

The function accepts one nullable descriptor pointer.

Under the 32-bit word and address interpretation used for this contract, a non-null descriptor has the following layout:

| Byte offset | Width | Meaning |
|---|---:|---|
| 0 | 32 bits | Unsigned payload byte count |
| 4 | 32 bits | Kind selector |
| 8 | 32 bits | Payload address; zero means no payload |

The accepted kind selectors are:

- Decimal **10**, hexadecimal **0x0000000A**: return the accumulated magnitude.
- Decimal **266**, hexadecimal **0x0000010A**: return the unsigned negation of the accumulated magnitude.

The count is valid only when it is between zero and four, inclusive.

The descriptor fields are native words in the supplied representation. Their byte order is not specified by the conversion logic. In contrast, the payload byte order is explicitly most-significant-byte first and does not depend on native word byte order.

### Output

The result is a 32-bit unsigned value:

- **0** for a null descriptor.
- **0xFFFFFFFF** for an unsupported selector or a count greater than four.
- **0** when validation succeeds but the payload address is zero.
- Otherwise, the converted magnitude or its unsigned negation, according to the selector.

No separate status output, returned length, or error object is present.

### Preconditions

For defined, stable-input behavior:

- Every descriptor field reached by execution must be readable with the alignment and access requirements of the target.
- A nonzero payload address used with a positive count must identify at least that many readable bytes.
- Descriptor fields and payload bytes must remain stable during execution.
- Address arithmetic for the accessed payload range must identify valid memory.

A zero count requires no readable payload bytes, even when the payload address is nonzero. The payload-address field itself is still read after successful validation.

The 32-bit interpretation is the working model of the supplied representation. The original target’s type definitions, pointer representation, and alignment rules were not independently supplied.

## Behavior

### Null descriptor

A null descriptor produces zero without reading descriptor fields or payload memory.

This is an accepted early-return case, not the all-ones validation failure.

### Validation

For a non-null descriptor, the selector is examined first.

If the selector is neither 10 nor 266, the function returns **0xFFFFFFFF**. The supplied control flow does not require a count read, payload-address read, or payload-byte read on this path.

For an accepted selector, the count is examined next. A count greater than four produces **0xFFFFFFFF**, without consulting the payload address or reading payload bytes.

The count comparison is unsigned. Thus, a count word with all bits set is greater than four; it is not treated as a negative count that bypasses validation.

Validation occurs before the null-payload check. Consequently, a zero payload address does not make an otherwise invalid descriptor acceptable.

### Null payload address

After successful validation, the function examines the payload address.

If that address is zero, the function returns zero regardless of the accepted selector and regardless of whether the validated count is zero, one, two, three, or four.

This behavior does not report a missing-payload error. In particular, a positive count paired with a zero payload address returns zero without attempting a payload read.

### Byte conversion

For a nonzero payload address, the function reads the selected bytes in increasing address order, starting at the payload address.

The first byte read becomes the most significant byte of the selected sequence. Each subsequent byte extends the magnitude by one byte on the right.

Examples of the byte-order rule:

- The single byte **12** represents hexadecimal **0x12**.
- The two bytes **12 34** represent **0x1234**.
- The three bytes **12 34 56** represent **0x123456**.
- The four bytes **12 34 56 78** represent **0x12345678**.

All byte values in these examples are hexadecimal. Each byte contributes an unsigned value from zero through 255; there is no sign extension.

With stable valid input, the function reads exactly the count’s number of payload bytes. It does not read a terminator, skip leading zeros, reverse bytes, or inspect bytes beyond that count.

A zero count produces an empty accumulation whose value is zero. No payload byte is accessed.

### Selector-dependent result

For selector 10, the accumulated magnitude is returned unchanged.

For selector 266, the accumulated magnitude is negated modulo **2³²**. This is unsigned arithmetic, not a separately returned signed integer and not a sign-bit interpretation of the payload.

Consequences include:

- A magnitude of zero remains zero.
- A magnitude of one becomes **0xFFFFFFFF**.
- A magnitude of **0x80000000** remains **0x80000000**.
- A magnitude of **0xFFFFFFFF** becomes one.

No saturation, range exception, or signed-overflow diagnostic is shown.

At most four bytes are accumulated. Under the 32-bit model, every accepted magnitude fits in the result width. Four-byte accumulation retains all 32 payload bits.

### Error ambiguity

The validation failure value **0xFFFFFFFF** is not reserved exclusively for errors.

It is also the successful result of:

- Selector 10 with the four payload bytes **FF FF FF FF**.
- Selector 266 with a magnitude of one.

A caller cannot determine validation success from the returned integer alone. Any distinction must come from information outside this return value.

## Side effects

The supplied function contains no explicit writes to the descriptor, payload, or global state. It shows no allocation, deallocation, ownership transfer, logging, callback, or external function call.

Descriptor and payload storage remain caller-provided. Nothing in the supplied behavior releases or retains that storage.

The meaningful source-level read progression is:

1. Examine the selector for a non-null descriptor.
2. Examine the count only if the selector is accepted.
3. Examine the payload address only if the count is accepted.
4. For a nonzero payload address, test the loop bound and read payload bytes in increasing index order.
5. Apply the selector-dependent result transformation after accumulation.

The selector used after accumulation is the value obtained at the initial selector read. The supplied representation refers to the descriptor’s count again during loop-bound checks and to its payload address during payload access. It does not establish that all descriptor fields are captured as one immutable snapshot.

These observations describe the supplied source-level access structure. They do not prove exact machine-load counts or hardware ordering. No volatile, atomic, locking, or memory-barrier guarantees are shown. Concurrent changes and memory-mapped-device behavior therefore fall outside the stable-input contract.

Invalid non-null memory is not converted into the all-ones error result by any shown check. If a required descriptor or payload access is invalid, the supplied representation provides no defined recovery behavior.

## Dependencies

No called-function dependencies are present in the supplied body.

F-a13acf58cba0 depends on readable caller-provided memory and the target’s integer, byte, pointer, and native-word access semantics. No external dependency is available whose memory ownership or global-state behavior needs to be assumed.

The supplied mapping identifies only F-a13acf58cba0 itself; it does not establish additional opaque dependency IDs.

## Acceptance examples

The following are conditional acceptance cases derived from static evidence, not executed tests. They assume the 32-bit model and stable readable storage wherever an access is required. Payload bytes are written in hexadecimal.

| Input condition | Required result | Required payload access |
|---|---:|---|
| Null descriptor | **0x00000000** | None |
| Selector 11 | **0xFFFFFFFF** | None; count and payload address need not be read |
| Selector 10, count 5 | **0xFFFFFFFF** | None; payload address need not be read |
| Selector 266, count **0xFFFFFFFF** | **0xFFFFFFFF** | None |
| Selector 10, count 3, zero payload address | **0x00000000** | None |
| Selector 266, count 4, zero payload address | **0x00000000** | None |
| Selector 10, count 0, nonzero payload address | **0x00000000** | No payload bytes |
| Selector 266, count 0, nonzero payload address | **0x00000000** | No payload bytes |
| Selector 10, count 1, payload **80** | **0x00000080** | One byte |
| Selector 10, count 2, payload **12 34** | **0x00001234** | Two bytes, ascending addresses |
| Selector 10, count 3, payload **00 12 34** | **0x00001234** | All three bytes, including the leading zero |
| Selector 10, count 4, payload **12 34 56 78** | **0x12345678** | Four bytes |
| Selector 266, count 2, payload **12 34** | **0xFFFFEDCC** | Two bytes |
| Selector 266, count 1, payload **01** | **0xFFFFFFFF** | One byte; successful conversion |
| Selector 10, count 4, payload **FF FF FF FF** | **0xFFFFFFFF** | Four bytes; successful conversion |
| Selector 266, count 4, payload **FF FF FF FF** | **0x00000001** | Four bytes |
| Selector 266, count 4, payload **80 00 00 00** | **0x80000000** | Four bytes |

Additional boundary conditions:

- An unsupported selector paired with count zero and a zero payload address still returns **0xFFFFFFFF**.
- An accepted selector paired with count five and a zero payload address still returns **0xFFFFFFFF**.
- A nonzero payload address with count zero need not point to readable payload bytes, because no byte is accessed.
- With a positive count and a nonzero but unreadable payload address, no normal result is specified by the supplied evidence.

## Uncertainties

### Established static evidence

The supplied representation establishes the two accepted selectors, the four-byte maximum, validation precedence, null-input behavior, byte accumulation order, optional unsigned negation, and absence of explicit writes or calls.

### Inferred intent

The two selectors appear to distinguish a magnitude from its negated form. Their broader format meaning is not established.

The all-ones early return appears intended as an error indication. That interpretation does not eliminate its collision with valid results.

### documented_only

The following remain unverified:

- Original target ABI, exact native type definitions, word endianness, and alignment requirements.
- Machine-level load ordering and whether repeated source-level field references become repeated memory reads.
- Behavior under concurrent descriptor or payload mutation.
- Handling of inaccessible addresses, address arithmetic outside valid storage, and hardware memory faults.
- Whether surrounding callers impose narrower magnitude limits or distinguish failure using separate metadata.
- Whether the permissive zero result for a missing payload is intentional format semantics or merely existing behavior.

None of these unknowns supplies an additional validation rule or recovery guarantee.

## Evidence

This contract is based exclusively on static inspection of the supplied function representation and its provided opaque identity mapping.

The representation contains three descriptor-word positions, selector and count guards, a null-payload guard, bounded byte accumulation, and selector-dependent unsigned negation. These support the behavior described above under the stated width model.

No implementation execution, runtime tests, caller inspection, dependency inspection, or independent ABI verification has been performed.
