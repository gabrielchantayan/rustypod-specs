<!-- Trusted provenance: F-37adf808f9dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-37adf808f9dc

## Purpose

F-37adf808f9dc classifies an integer selector into one of three result values:

- Selector **1** produces **2**.
- Selector **2** produces **1**.
- Every other selector produces **0**.

The first argument does not participate in this classification. The supplied function body contains no argument-dependent memory access, external call, or persistent state update.

This is a static behavioral contract. No runtime tests have been performed.

## Interface

### Inputs

| Position | Contract name | Representation | Behavioral role |
|---|---|---|---|
| First | Unused value | An opaque 32-bit value in the supplied recovered interface | Accepted but not inspected by the function body. |
| Second | Selector | An integer represented as `int` in the supplied recovered interface | Compared with 0, 1, and 2 to select the result. |

The contract names above are descriptive labels, not implementation identifiers.

The first argument has no established semantic type beyond its four-byte representation. In particular, there is no evidence that it denotes a pointer, handle, object, flags field, or enumeration. If a caller supplies an address-shaped value, the function does not dereference it.

The selector is presented as a signed integer. A 32-bit selector is consistent with the recovered interface, but the supplied material does not independently establish the target integer-width convention or machine-level argument-passing rules. The classification applies to the selector value received by the function; conversion from a wider caller-side value is outside the established contract.

### Output

The result is a 32-bit value. Its complete established value set is:

| Result | Condition |
|---|---|
| 0 | Selector is neither 1 nor 2. |
| 1 | Selector is 2. |
| 2 | Selector is 1. |

All three results are exactly representable as either signed or unsigned 32-bit values. Return signedness therefore does not affect the numerical interpretation of any established result.

There are no output buffers, returned allocations, or returned references.

### Preconditions

No value-specific precondition appears in the supplied body. In particular:

- The selector need not be limited to 0, 1, and 2.
- The unused value need not refer to accessible memory.
- No initialized object or external state is consulted.

The caller must still satisfy the actual calling convention. Register assignments, stack layout, argument extension rules, and binary interoperability requirements are not established by the supplied material.

## Behavior

### Result selection

The function implements a total classification over the integer values representable by its selector interface.

A selector equal to 1 produces 2. A selector equal to 2 produces 1. All remaining selectors produce 0, including zero, negative values, and positive values greater than 2.

The function does not reduce values modulo a range, interpret individual bits, or normalize unsupported values into either distinguished selector. Equality with the complete received selector value determines the result.

The unused value cannot change the result. Holding the selector fixed while changing only the first argument leaves the established result unchanged.

### Evaluation order

The supplied body establishes the following logical comparison order:

1. Zero is distinguished from nonzero selectors.
2. For a nonzero selector, equality with 1 is considered first.
3. Only a nonzero selector not equal to 1 reaches the comparison with 2.
4. A matching distinguished selector returns its corresponding result; otherwise the result is 0.

This comparison order is not a promise about physical argument loads or processor instructions. The supplied representation does not establish whether a compiler retains the branches, reloads an argument, or uses an equivalent instruction sequence.

No explicit memory read or write intervenes between the selector comparisons. There is no external call whose behavior could alter state during selection.

### Arithmetic and boundaries

No addition, subtraction, multiplication, division, shifting, narrowing, or index calculation occurs in the supplied body. The function selects between constant results.

Consequently, the established computation has no arithmetic overflow or underflow case. Extreme representable selector values are handled by the same equality rules as other values.

If the selector interface is confirmed to be signed 32-bit, both −2,147,483,648 and 2,147,483,647 produce 0. Those concrete boundary examples are conditional on that width confirmation.

### Errors

The supplied body contains no separate error path, exception operation, diagnostic, or failure-reporting mechanism.

Result 0 is the ordinary result for every selector outside the two distinguished values. Calling it an “error,” “invalid value,” or “unsupported operation” would require caller-side evidence that is not supplied.

## Side effects

The supplied body contains no explicit:

- Memory dereference or buffer access.
- Global or static state access.
- Allocation, deallocation, or ownership transfer.
- Object mutation.
- File, device, network, or diagnostic operation.
- Callback or other function invocation.

The function neither acquires nor releases ownership of anything represented by the first argument. It also does not validate that argument. A successful return therefore establishes nothing about whether an address or handle supplied there is valid.

The only established caller-visible output is the return value. Ordinary execution mechanics, such as stack use and register changes permitted by the calling convention, are outside the source-level side-effect description.

For defined calls, the result depends only on the received selector. No external state dependency appears in the supplied body.

## Dependencies

**No function dependencies are present in the supplied body.**

There are no opaque dependency IDs to enumerate beyond F-37adf808f9dc itself. Its established classification does not rely on an unknown callee, lookup table, allocator, or stateful service.

Caller behavior and binary integration remain outside the available evidence. This contract does not assume that callers validate selectors, assign meanings to the results, or preserve particular machine state.

## Acceptance examples

These are conditional acceptance criteria derived from static inspection, not observed test results.

| Received selector | First argument | Required result | Basis |
|---|---|---|---|
| 0 | Any admissible 32-bit value | 0 | Zero reaches the default result. |
| 1 | Any admissible 32-bit value | 2 | First distinguished selector. |
| 2 | Any admissible 32-bit value | 1 | Second distinguished selector. |
| 3 | Any admissible 32-bit value | 0 | Nonzero and unequal to both distinguished selectors. |
| −1 | Any admissible 32-bit value | 0 | Negative values do not match 1 or 2. |

Additional acceptance conditions:

- **First-argument independence:** For selector 1, changing the first argument from an all-zero 32-bit value to an all-one 32-bit value must leave the result at 2. The same independence applies to every selector.
- **No address validation:** If the first argument contains an address-shaped bit pattern, the function must not access memory through it. This condition does not waive calling-convention requirements.
- **No low-bit classification:** A representable selector such as 257 must produce 0, not 2, despite sharing its low byte with 1.
- **Boundary behavior:** If the selector is signed 32-bit, its minimum and maximum representable values must each produce 0.
- **Repeated calls:** Repeated defined calls receiving the same selector must produce the same result, irrespective of previous selectors. No retained state appears in the supplied body.

No acceptance condition requires particular branch instructions, execution timing, or physical load counts.

## Uncertainties

### Established static evidence

The supplied body establishes the two distinguished selector mappings, the default result, first-argument non-use, and the absence of explicit calls and memory operations. Its recovered type notation explicitly identifies the first argument and return value as four-byte quantities.

### Inferred intent

The mapping may be intended to exchange two codes while mapping all other codes to zero. That interpretation is plausible but not required for conformance. No domain-specific meaning is established for any input or result.

The function is not a general exchange operation over all integers: values outside 1 and 2 are discarded into the common result 0.

### documented_only

The following remain unestablished:

- The selector’s independently verified ABI width.
- The precise calling convention and machine-level representation of arguments and return values.
- The domain meanings of the selector and result codes.
- Why the interface includes an unused first argument.
- Whether callers restrict the selector domain.
- The final compiled instruction sequence and timing characteristics.

These unknowns do not change the established mapping for received selector values.

## Evidence

The sole evidence is the supplied recovered function definition. It contains one unused argument, integer equality checks, three constant return outcomes, and no calls or explicit memory accesses.

No caller definitions, machine-code listing, ABI documentation, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples specify expected behavior from static evidence; they are not claims of executed verification.
