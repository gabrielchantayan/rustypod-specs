<!-- Trusted provenance: F-0da7d1f433ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0da7d1f433ba

## Purpose

F-0da7d1f433ba reads an integer field at byte offset 236 from an address supplied by its caller. It returns zero when the value read equals signed negative one; otherwise, it returns that value unchanged.

The observable transformation is therefore a sentinel normalization, not a general validity check. Negative values other than negative one remain negative. Positive values are not bounded, clamped, or otherwise transformed. A stored zero and a stored negative one both produce a zero result.

The function does not write the normalized value back to the field.

**Established static evidence:** one integer load, comparison against negative one, conditional substitution of zero, and return of the resulting integer.

**Inferred intent:** negative one may represent an absent, uninitialized, or otherwise exceptional field value that callers should observe as zero. The supplied material does not establish which meaning applies.

## Interface

### Input

The function accepts one integer-typed argument. Its value is used as an address base, rather than as the integer value to normalize.

The field address is obtained by adding 236 bytes to that base. The addition precedes conversion of the resulting address into an integer-access location. Consequently, the offset is a byte displacement; it is not an index scaled by the size of an integer.

The supplied representation does not establish whether the base denotes:

- The start of a larger object.
- An interior address within an allocation.
- A location in a mapped or externally managed region.
- Some other readable address arrangement.

Only the relationship between the supplied base and the accessed field is established.

### Output

The function returns an integer with the same declared integer type as the field load.

For a successfully read value:

| Value read from the field | Returned value |
|---|---:|
| Negative one | Zero |
| Any other representable integer | The value read, unchanged |

There is no separate status result. In particular, zero does not distinguish a stored zero from the normalized negative-one sentinel.

### Width and representation

The supplied representation uses an unqualified integer type for the argument, field load, and return value. It does not independently provide an ABI, an integer-size declaration, an endianness specification, or a pointer-width declaration.

Under a 32-bit integer interpretation:

- The field occupies four bytes beginning at offset 236.
- Those bytes occupy offsets 236 through 239 relative to the supplied base.
- The loaded and returned values range from −2,147,483,648 through 2,147,483,647.
- On a two’s-complement target, the negative-one sentinel has the 32-bit pattern with every bit set.

These are conditional layout and representation consequences, not independently verified platform facts. The established operation is comparison with the integer value negative one, not a separately specified byte-pattern comparison.

### Preconditions

Successful ordinary execution requires that:

- The address calculation produce a usable address under the actual target’s address and integer rules.
- The field location permit an integer-width read.
- Any alignment requirements of that read be satisfied.
- The underlying storage remain accessible for the read.

If the base is the start of a conventional contiguous object and integers are four bytes wide, the object must provide readable storage through offset 239. The function itself does not require or inspect any particular layout before offset 236.

No null check, bounds check, alignment check, or address-overflow check is shown. Invalid inputs do not have a defined function-level error result in the supplied evidence.

## Behavior

### Field selection

The function selects exactly one field using the fixed byte displacement of 236 from the supplied base. It does not obtain the displacement from another field, an argument, a table, or a dependency.

The base location itself is not dereferenced to obtain a second pointer. The supplied argument directly participates in forming the field address.

### Read and normalization

The field is read once in the supplied representation. The value obtained by that read is then tested against negative one.

If it equals negative one, the returned value is zero. Otherwise, the returned value is the value already obtained from the field. There is no second field read on either branch.

This distinction matters if the field can change independently: the shown behavior derives the result from the captured read value, rather than re-reading storage after the comparison. The material does not, however, establish atomicity, synchronization, or language-level guarantees for concurrent mutation.

### Boundary behavior

Only exact equality with negative one triggers normalization.

Accordingly:

- Zero remains zero.
- Positive one remains positive one.
- Negative two remains negative two.
- The minimum representable signed integer remains unchanged unless that value is negative one, which it is not for ordinary supported integer widths.
- The maximum representable signed integer remains unchanged.

No arithmetic is performed on the loaded value. Thus, the normalization itself does not introduce increment, decrement, negation, multiplication, or magnitude-overflow cases. Address formation is the separate arithmetic operation whose target-specific overflow behavior remains unspecified.

### Completion and errors

After a successful read and comparison, the function returns normally with the selected integer result.

There is no explicit error branch, fallback for unreadable memory, retry, exception handler, or diagnostic operation. A failed or invalid memory access must not be documented as returning zero merely because zero is the sentinel’s normalized result. Any resulting fault or other platform behavior lies outside the established normal-return contract.

## Side effects

### Observable memory access

The source-level access order is:

1. Form the field address using the supplied base and the fixed byte offset.
2. Read one integer from that address.
3. Select and return the result based on the value read.

No target-memory write appears in the supplied body. In particular, reading negative one does not replace the stored field with zero.

No access to another object field, global variable, external resource, or indirect dependency is shown. This describes the explicit operations in the supplied representation; it does not establish instrumentation behavior or machine-level access guarantees absent from that representation.

### Ownership and lifetime

The function performs no allocation, deallocation, ownership transfer, or explicit resource acquisition.

The caller remains responsible for providing storage that is valid for the required read. The function does not retain the supplied address in any shown state, and no persistent state change is shown.

A successful call does not establish that the memory is writable, owned by the caller, or safe for later access. It establishes only that this invocation obtained a value through the shown read.

## Dependencies

No called functions are present in the supplied body. There are therefore no opaque function dependencies to enumerate and no callee preservation, allocation, or release behavior to assume.

The behavior depends on execution-environment properties that are not functions:

- Integer width and signed representation.
- Address and pointer representation.
- The rules governing the address calculation.
- Integer-load size, alignment, and byte order.
- Accessibility and stability of the referenced storage.

These properties must be supplied by the target platform or surrounding contract where needed.

## Acceptance examples

The following are conditional acceptance criteria derived from static evidence, not observations from executed tests.

### Stored sentinel

Given a valid base whose integer field at byte offset 236 contains negative one, the function must return zero.

The field must not be rewritten to zero by this function. A caller inspecting the same unchanged storage afterward would still find negative one, provided no external actor modified it.

### Stored zero

Given valid readable storage with zero in the selected field, the function must return zero without changing that storage.

The result is intentionally indistinguishable from the sentinel case through the return value alone.

### Other negative value

Given valid readable storage with negative two in the selected field, the function must return negative two.

An implementation that maps every negative value to zero would not satisfy the established behavior.

### Positive value

Given valid readable storage with 237 in the selected field, the function must return 237. The value is not treated as another offset, dereferenced, or validated against an external range.

### Integer boundary

On a target with 32-bit signed integers, a selected field containing −2,147,483,648 must produce that same result. A field containing 2,147,483,647 must likewise produce that same result.

### Field selection

Given a valid layout in which an adjacent field contains negative one but the integer beginning at byte offset 236 contains seven, the function must return seven. Values in adjacent fields do not participate in the shown decision.

### Unreadable address

If the calculated field location is unreadable, no particular return value is an acceptance requirement established by this material. In particular, zero is not a documented invalid-address response.

## Uncertainties

The following remain **documented_only**:

- The exact integer and pointer widths, target ABI, endianness, and signed representation.
- Whether the integer-typed input reflects the original interface or an address represented through an integer type.
- The enclosing object’s type, total size, field name, and semantic role.
- The intended meaning of the negative-one sentinel.
- Address-overflow behavior and whether any caller-side validation exists.
- Alignment requirements and handling of unaligned reads.
- Whether the field can change concurrently, whether a load is atomic, and what synchronization callers must provide.
- Behavior for invalid, inaccessible, or otherwise unsupported addresses.

These unknowns do not justify adding validation, retries, broader normalization, or ownership behavior to the function’s established contract.

## Evidence

The sole evidence is the supplied function body. It establishes the fixed offset of 236 bytes, one integer read, an equality comparison against negative one, replacement of that result with zero only on equality, and an integer return.

The supplied body contains no function calls, explicit memory writes, allocation operations, or explicit error handling.

No runtime tests have been performed. Acceptance examples describe required behavior conditional on valid storage and applicable platform properties; they are not test results.
