<!-- Trusted provenance: F-6a648ec13abe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6a648ec13abe

## Purpose

F-6a648ec13abe returns the zero-based position of the highest set bit in an unsigned integer. An input of zero returns zero as a special case. Consequently, zero and one have the same result.

For every positive input, the result is the integer part of its base-two logarithm. Equivalently, it is one less than the number of binary digits required to represent that input without leading zeroes.

This mathematical description follows from the supplied body. An intended use—such as sizing a field, selecting a bucket, or classifying a magnitude—is not established.

The operation does not count all set bits, locate the lowest set bit, or round a logarithm upward.

## Interface

**Function identity:** F-6a648ec13abe.

**Input:** One unsigned integer, passed by value.

**Output:** One signed integer, returned by value. The result is nonnegative for every valid input under the represented integer model.

**Width model:** Let \(W\) be the number of value bits in the input’s unsigned integer type. The supplied declaration does not independently establish a target ABI or an explicit numeric width. The behavioral rule is therefore stated in terms of \(W\). If the unsigned type has 32 value bits, the input range is 0 through 4,294,967,295 and the output range is 0 through 31.

The accumulator and result use the signed integer type shown in the supplied declaration. A faithful implementation must be able to represent every result from zero through \(W-1\). This holds for the conventional corresponding signed and unsigned integer types represented by the source.

**Input layout:** The interface accepts an integer value, not a byte sequence, address, array, or structured record. No byte order, alignment, buffer size, or serialized representation is specified or consumed by this function.

**Preconditions:**

- The argument must be supplied as a value of the declared unsigned integer type.
- Evaluation must use the width and unsigned bitwise semantics of that type.
- There is no nonzero-input precondition.

Conversion from an external representation, including conversion from a negative number or an out-of-range value, is outside the function’s body and outside this contract unless the calling environment specifies it.

**Ownership:** No pointer or resource handle is accepted or returned. The body contains no allocation, release, ownership transfer, or retained reference.

## Behavior

### Result rule

For an input \(x\), the returned value is:

- Zero when \(x=0\).
- The unique integer \(r\) satisfying \(2^r \leq x < 2^{r+1}\) when \(x>0\).

The powers in this definition are mathematical quantities, not required machine-integer computations. In particular, the upper comparison remains meaningful for an input whose highest bit is the top bit of the unsigned type, even though \(2^W\) is not representable in that type.

For positive inputs, the function partitions values into contiguous power-of-two intervals. Every value from \(2^r\) through \(2^{r+1}-1\), inclusive, returns \(r\), subject to the input type’s maximum value.

### Evaluation and stopping condition

The supplied body establishes an iterative evaluation using a low-bit coverage mask.

The first condition examines whether the input has any set bit above bit zero. If it does not, the function returns zero. This accounts for both zero and one without requiring a separate zero branch.

Each successful iteration extends the covered low-bit region by one position and increases the eventual result by one. The next condition then examines whether any input bit remains set outside that expanded region. Evaluation stops at the first region that covers every set bit in the input.

More precisely, before the condition associated with a result count \(r\), the coverage mask contains ones in bit positions zero through \(r\), inclusive. Its value is mathematically \(2^{r+1}-1\). The condition is true exactly when the input has a set bit at a position greater than \(r\).

For a positive input whose highest set bit is at position \(h\):

- Conditions for counts below \(h\) succeed.
- Each such success causes one count increase and one mask expansion.
- The condition at count \(h\) fails.
- The returned count is \(h\).

The same initial condition fails for zero, producing zero.

This evaluation order describes the supplied body’s arithmetic behavior. It does not prescribe instruction selection or forbid an implementation from using an equivalent operation when no additional observation is exposed.

### Width and arithmetic boundaries

The bit numbering is zero-based: the least significant bit is position zero, and the highest available position is \(W-1\).

The mask expands using an unsigned left shift by one bit and setting the lowest bit. This produces successive masks covering one, two, three, and eventually \(W\) low bits. The complement used by the condition is interpreted within the unsigned type’s width; it does not introduce an unbounded collection of high bits.

When the mask covers all \(W\) bits, no input bit can remain outside it. The condition must therefore fail. No subsequent mask expansion is evaluated.

Consequences under this width model:

- The loop executes at most \(W-1\) times.
- It evaluates its stopping condition at most \(W\) times.
- It never needs a shift count equal to or greater than the type width.
- It returns at most \(W-1\).
- The maximum unsigned input terminates normally.

For a 32-bit unsigned input with its highest bit set, the body performs 31 successful iterations and then returns 31. The arithmetic does not require treating that input as a signed value.

### Errors and exceptional inputs

The body defines no error result, exception path, diagnostic, or retry behavior. Zero is an ordinary input, not an error.

Every representable unsigned value has a result. There is no sentinel input within that domain.

## Side effects

The supplied body contains no access to caller-provided memory, global state, external devices, files, or other persistent state. It makes no calls.

Its explicit state changes are confined to two internal scalar values: a result count and a coverage mask. The input value is not modified.

The source-level evaluation order is:

1. Establish the initial count and coverage mask.
2. Evaluate whether any input bit lies outside the mask.
3. On a successful condition, increase the count, then expand the mask, then evaluate the next condition.
4. On a failed condition, return the current count.

These internal updates create no caller-visible write in the supplied interface. Argument passing, register use, stack storage, and the return-value location are ABI details not established by the body.

The number of iterations depends on the highest set-bit position. The body therefore does not establish a constant-time execution property. Actual timing and generated machine behavior are unknown.

## Dependencies

No dependency function IDs are present. F-6a648ec13abe invokes no other function in the supplied body.

The operation relies only on integer initialization, comparison, bitwise operations, unsigned shifting, signed increment, loop control, and returning a scalar.

There is no unknown callee whose effects on globals, memory ownership, or resource lifetime must be assumed. Compiler-generated support operations, if any, are not established by the supplied evidence.

## Acceptance examples

The following are acceptance criteria derived from static inspection. They are not reports of executed tests.

### Width-independent examples

These examples apply whenever the input type can represent the stated value:

| Input | Required result | Reason |
|---:|---:|---|
| 0 | 0 | No set bit lies outside the initial low-bit mask. |
| 1 | 0 | The highest set bit is at position zero. |
| 2 | 1 | The highest set bit is at position one. |
| 3 | 1 | Additional lower set bits do not change the highest position. |
| 4 | 2 | An exact power of two returns its exponent. |
| 7 | 2 | All values from 4 through 7 share the same highest set bit. |
| 8 | 3 | Crossing a power-of-two boundary increases the result. |
| 9 | 3 | The result is not a population count or an upward-rounded logarithm. |
| 16 | 4 | The result identifies the highest bit, not the lowest one. |

For any representable \(2^k\) with \(k \geq 1\):

- \(2^k-1\) must return \(k-1\).
- \(2^k\) must return \(k\).
- \(2^k+1\), when representable, must return \(k\).

For two positive inputs with the same highest set-bit position, changing any lower bits must leave the result unchanged.

### Conditional 32-bit examples

If the unsigned input type has exactly 32 value bits:

| Input | Required result |
|---:|---:|
| 65,535 | 15 |
| 65,536 | 16 |
| 2,147,483,647 | 30 |
| 2,147,483,648 | 31 |
| 4,294,967,295 | 31 |

The final example also requires termination without an error or an additional iteration beyond full-width mask coverage.

## Uncertainties

- **documented_only — Exact numeric widths:** The supplied declaration identifies unsigned and signed integer types but provides no independent ABI or type-width definition. The 32-bit cases above are conditional, not proof of the target width.
- **documented_only — Calling convention:** Register assignment, stack layout, symbol linkage, and binary return-value representation are unspecified.
- **documented_only — Runtime characteristics:** No execution measurements, instruction trace, or runtime tests are available. Source-level iteration counts do not prove a particular machine-code timing profile.
- **documented_only — Intended application:** The result is consistent with a floor-logarithm or highest-set-bit helper. No caller evidence establishes its application or any narrower expected input range.
- **documented_only — External conversions:** Behavior before entry, including parsing, narrowing, or signed-to-unsigned conversion, is not shown.

These unknowns do not prevent a complete value-level contract parameterized by the unsigned type’s width.

## Evidence

**Established static evidence:** The supplied function body initializes a zero result count and a one-bit coverage mask, repeatedly checks for input bits outside that mask, increases the count while expanding low-bit coverage, and returns the count. It contains no function calls or external-memory accesses.

**Derived behavior:** The mask sequence covers progressively higher bit positions. The first failed condition identifies the highest set-bit position, with zero returning the initial count. The result bounds and termination argument follow from eventual full-width coverage.

**Inferred intent:** Describing the operation as a highest-set-bit or floor-base-two-logarithm helper is a mathematical characterization, not evidence of its role in a larger system.

**Verification status:** Static analysis only. No runtime tests have been performed.
