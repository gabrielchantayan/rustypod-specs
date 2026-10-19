<!-- Trusted provenance: F-7c8a9c6c9d7f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7c8a9c6c9d7f

## Purpose

F-7c8a9c6c9d7f conditionally retrieves one 32-bit value from an address calculated using a caller-supplied base and signed selector. Selectors greater than 12 are directed to F-a817d8d58c10, a dependency annotated as not returning.

The visible selector check is an upper-bound check only. Negative selectors are not rejected. Consequently, this function must not be described as providing complete bounds validation for an array.

A plausible intended use is access to a four-byte field within one of thirteen eight-byte records beginning 96 bytes after a containing object’s base. That interpretation is inferred intent, not an established object layout or allocation guarantee. The established behavior is the address calculation and four-byte retrieval.

## Interface

**Inputs**

- **Base:** An integer-valued input used as the base of the eventual memory address. It is not visibly validated, dereferenced independently, or converted through a checked pointer API.
- **Selector:** A signed integer-valued input compared against 12 and, when the comparison permits retrieval, multiplied by an eight-byte stride.

The recovered interface represents both inputs as signed integers. A 32-bit interpretation is consistent with that interface, but the supplied material does not independently establish the target ABI or machine arithmetic rules. In particular, the width of the returned value is more firmly established than the pointer width or the behavior of overflowing address arithmetic.

**Output**

On normal completion, the function returns the four-byte value read at the selected address. The return value is an untyped 32-bit quantity: its interpretation as a signed integer, unsigned integer, pointer, floating-point value, or packed fields is not established.

There is no separate success indicator or recoverable error return visible in the function.

**Address layout**

For arithmetic that is representable under the applicable execution model, the selected byte address is:

**base + 96 + 8 × selector**

The operation reads four bytes beginning at that address. Eight bytes separate adjacent selected addresses, but only four bytes are read by this function. No meaning is established for the remaining four bytes in each apparent eight-byte slot.

**Preconditions for normal completion**

- The selector must not be greater than 12.
- The address calculation must produce a usable address under the actual target’s arithmetic and addressing rules.
- The four-byte region at that address must be readable.
- Any target-specific alignment requirements must be satisfied.

These are conditions for a successful read, not validations performed by the function. A negative selector can satisfy the visible comparison while violating the caller’s object boundaries.

## Behavior

### Selector decision

The first visible decision compares the selector, as a signed value, with 12.

A selector greater than 12 causes a call to F-a817d8d58c10. No explicit arguments are supplied to that dependency in the recovered interface. The dependency is annotated as not returning. Under that annotation, this path has no normal return from F-7c8a9c6c9d7f and does not reach the four-byte retrieval.

The exact reason for the dependency call is not established. It may implement a fatal condition or another non-returning transfer, but the supplied material does not identify termination behavior, diagnostics, exception behavior, or an error classification.

Selectors equal to or below 12 bypass the dependency call and proceed to retrieval. In particular:

- Zero is accepted by the visible comparison.
- Twelve is accepted.
- Thirteen is not accepted.
- Every negative selector is accepted by this comparison.

The function does not visibly clamp, normalize, reinterpret as unsigned, or substitute a default selector.

### Address calculation and retrieval

On the retrieval path, the function derives an address from the supplied base, a fixed displacement of 96 bytes, and a selector-dependent displacement of eight bytes per selector unit.

The selected addresses for ordinary representable cases include:

| Selector | Displacement from base |
|---:|---:|
| −13 | −8 bytes |
| −12 | 0 bytes |
| −1 | 88 bytes |
| 0 | 96 bytes |
| 1 | 104 bytes |
| 12 | 192 bytes |

Negative selectors therefore move the read backward relative to the apparent start of the record area. Some remain after the base, some reach the base itself, and others precede it. The function does not distinguish these cases.

The selected operation reads a four-byte value and returns it without a visible transformation. There is no visible sign extension into a larger return type, byte swapping, masking, decoding, or second lookup.

The four-byte width does not establish a particular bus transaction, atomic access guarantee, or concurrency model. Those properties require target-specific evidence.

### Observable ordering

For the visible body, the relevant ordering is:

1. The selector comparison determines whether the dependency call is taken.
2. On the permitted retrieval path, the selected address is used for a four-byte read.
3. That read’s value becomes the normal return value.

No data read through the supplied base is shown before the selector decision. No memory write is shown before or after the retrieval. This describes the supplied body, not speculative processor activity or undisclosed behavior inside the dependency.

### Arithmetic boundaries

The stride is eight bytes; it is not the four-byte return width. The fixed displacement is 96 bytes.

Small selectors such as −13 through 13 permit straightforward displacement calculations. Extreme negative selectors are more problematic: multiplication by eight, addition of the fixed displacement, or addition to the base may exceed a signed integer range or cross an address-space boundary.

The supplied material does not establish whether such cases wrap, trap, become invalid under a source-language model, or follow another target-specific rule. A behavioral implementation must not silently choose a portable overflow policy and present it as recovered behavior. Extreme arithmetic cases remain documented_only pending machine-level evidence.

## Side effects

On the normal retrieval path, the visible memory effect is one four-byte read. The body contains no explicit memory write, allocation, release, reference-count update, or persistent state update.

The base is used for addressing. There is no visible ownership transfer, and this function itself does not release the addressed storage. The caller must provide a region that remains readable for the access.

These observations do not establish that F-a817d8d58c10 preserves global state, preserves ownership, or avoids releasing memory. Its implementation is unavailable. Its possible effects must remain separate from the effects visible in this body.

If the calculated address is unreadable or improperly aligned for the target, the function provides no visible recovery mechanism. Whether execution faults, traps, or behaves otherwise is not established. A zero base is not explicitly rejected; whether it leads to a readable selected address depends on the selector and environment.

## Dependencies

### F-a817d8d58c10

- **Invocation condition:** The signed selector is greater than 12.
- **Explicit arguments:** None shown.
- **Return behavior:** Annotated as non-returning.
- **Visible result use:** None.
- **Unknown effects:** Global mutations, diagnostics, memory ownership changes, resource handling, process termination, and other control transfers.

The annotation supports treating the greater-than-12 path as having no normal completion. It does not identify how that outcome is produced.

If the dependency unexpectedly returns, the supplied body places the retrieval after the conditional call. However, the annotation and the absence of machine-level control-flow evidence prevent a guaranteed contract for that situation. Dependency return is therefore documented_only, not a supported recovery path.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

1. **First apparent record:** Given selector 0, valid address arithmetic, and readable storage at base plus 96, the function returns the four-byte value at that address. It does not call F-a817d8d58c10.

2. **Highest accepted nonnegative selector:** Given selector 12 and readable storage at base plus 192, the function returns that location’s four-byte value. Equality with the upper bound must not invoke the dependency.

3. **First rejected nonnegative selector:** Given selector 13, the function calls F-a817d8d58c10. Under the non-returning annotation, no normal result is produced and the retrieval is not reached.

4. **Negative selector:** Given selector −1 and readable storage at base plus 88, the function retrieves that location’s value. Rejecting the selector merely because it is negative would differ from the visible behavior.

5. **Read at the base:** Given selector −12 and valid, readable storage at the base, the function returns the four-byte value beginning there. It does not require retrieval to remain within the apparent record area beginning at displacement 96.

6. **Stride distinction:** Given readable, distinct four-byte values at base plus 96, base plus 100, and base plus 104, selector 1 selects the value at base plus 104. A four-byte indexing stride would be incorrect.

7. **Unreadable selected address:** Given a selector that passes the comparison but selects unreadable memory, no successful return is promised. A fabricated zero or sentinel result would add behavior absent from the supplied evidence.

8. **Extreme arithmetic:** Given an extreme negative selector or a base near an address-space boundary, no wrapping-address result is specified here. Acceptance requires additional target arithmetic evidence.

## Uncertainties

The following remain documented_only:

- Exact target ABI, pointer width, and integer-overflow semantics.
- Endianness and interpretation of the returned four-byte quantity.
- Alignment, atomicity, and concurrent-access guarantees.
- Actual allocation boundaries and whether an eight-byte record abstraction exists.
- Whether callers are expected to restrict selectors to 0 through 12.
- The behavior and state effects of F-a817d8d58c10 beyond its non-returning annotation.
- Fault behavior for invalid addresses.
- Behavior if the annotated dependency unexpectedly returns.

None of these unknowns justifies adding lower-bound validation, checked arithmetic, default values, diagnostics, or ownership handling to the recovered contract.

## Evidence

**Established static evidence:** The supplied body contains a signed greater-than-12 decision, a conditional call to F-a817d8d58c10, a non-returning annotation for that dependency, and a four-byte retrieval using a 96-byte fixed displacement and eight-byte selector stride. No explicit writes or ownership operations appear in the body.

**Inferred intent:** Thirteen apparent slots, each eight bytes wide, with a selected four-byte field beginning 96 bytes after a containing base. This interpretation is not required to explain the retrieval and is not evidence of valid allocation bounds.

**Verification status:** Static analysis only. No runtime tests, dependency inspection, ABI verification, or machine-level arithmetic validation have been performed.
