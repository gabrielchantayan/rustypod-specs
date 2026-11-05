<!-- Trusted provenance: F-f51dad4c1b38; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f51dad4c1b38

## Purpose

F-f51dad4c1b38 returns a 64-bit value derived from a caller-supplied address-like argument.

The established control structure is:

- A zero argument returns zero without calling a dependency.
- A nonzero argument causes two dependency calls.
- A nonzero value in the second temporary pair causes two additional dependency calls, followed by a multiply-and-add calculation.
- The return value combines a low 32-bit word and a high 32-bit word.

**Inferred intent:** if the dependency copies eight bytes from its second argument into its first argument, the function reads a base value, a multiplier, and a multiplicand from three consecutive eight-byte fields. It returns the base unchanged when the initially read multiplier is zero; otherwise, it returns the base plus the product of the later-read multiplicand and the reread multiplier, reduced to 64 bits.

That interpretation is conditional. The dependency implementation is unavailable, so its copying behavior, failure behavior, memory access order, and other effects are not established.

## Interface

### Input

One argument is present. It is represented as a 32-bit integer in the supplied evidence but is used as an address-like value.

- Zero has an explicitly defined result.
- Nonzero values are passed to a dependency as source-like arguments.
- Additional source-like arguments are formed by adding 8 and 16 to the supplied value.
- No length, validity flag, alignment information, or ownership information accompanies the argument.

The nonzero path requires the dependency to accept the supplied address-like values for execution to complete normally. The function performs no visible validation of accessibility, alignment, object extent, or permission.

The evidence does not establish whether callers formally pass an integer, a pointer represented as an integer, or an address encoded through another interface.

### Output

The normal return is a 64-bit quantity assembled from two 32-bit words:

- One word supplies bits 0–31.
- The other supplies bits 32–63.

For a zero argument, all 64 result bits are zero.

There is no separate status result. The evidence does not establish a semantic signedness for the returned quantity. The calculation is most precisely described as a 64-bit bit pattern.

### Conditional input layout

Under the interpretation that the dependency performs an eight-byte copy, the following relative layout applies:

| Relative offset | Extent | Conditional role |
|---|---:|---|
| 0 | 8 bytes | Initial base value |
| 8 | 8 bytes | Initial zero/nonzero selector and, on the arithmetic path, a separately read multiplier |
| 16 | 8 bytes | Multiplicand, read only on the arithmetic path |

Each transferred value is treated as two 32-bit words. The recovered calculation identifies one word as low and the other as high. It does not independently establish a portable byte serialization or byte order.

A caller should not infer that an eight-byte request is performed atomically.

## Behavior

### Zero argument

When the argument equals zero, the function returns the 64-bit value zero immediately.

On this path:

- No dependency is called.
- No input memory is read by the shown function.
- No caller-provided memory is written by the shown function.
- No allocation or release is visible.

A second zero check appears later in the supplied representation, but it is unreachable after the initial early return. It does not define an additional reachable behavior.

### Initial dependency calls

For a nonzero argument, the function makes two calls to the same dependency, in this order:

1. A call with a destination-like address for the result pair, the original argument as the source-like value, and the numeric size argument 8.
2. A call with a destination-like address for a selector pair, the original argument plus 8 as the source-like value, and the numeric size argument 8.

The function does not visibly inspect a return value from either call.

After the second call returns, it checks the selector pair. The arithmetic path is selected if either its low 32-bit word or its high 32-bit word is nonzero. Equivalently, all 64 bits must be zero to avoid the additional calls.

The test is not a positivity test. A value with only the highest bit set selects the arithmetic path just as a value of one does.

### Zero-selector path

If both words of the selector pair are zero, the function makes no further dependency calls. It returns the then-current result pair.

Under ordinary eight-byte-copy behavior with no unrelated mutation, this is the eight-byte value obtained from relative offset 0.

The shown function does not request the field at relative offset 16 on this path. This is a control-flow guarantee about its dependency invocations, not a guarantee that the dependency itself accesses only the requested ranges.

### Nonzero-selector path

If either selector word is nonzero, the function makes two more calls, in this order:

1. A call targeting the same temporary pair previously used for the selector, with the original argument plus 16 as the source-like value and size argument 8.
2. A call targeting a separate temporary pair, with the original argument plus 8 as the source-like value and size argument 8.

The first of these replaces the selector pair with the multiplicand pair under the copying interpretation. The second obtains a fresh multiplier pair.

The first read of relative offset 8 determines whether this path is entered. The later read of relative offset 8 supplies the multiplier used in the calculation. These observations are distinct and must not be treated as one cached read when externally visible mutations are possible.

After the fourth call returns, the function combines the result pair, multiplicand pair, and multiplier pair.

### Arithmetic

Let:

- \(B_L\) and \(B_H\) be the low and high words of the result pair.
- \(X_L\) and \(X_H\) be the low and high words of the multiplicand pair.
- \(Y_L\) and \(Y_H\) be the low and high words of the freshly read multiplier pair.

All words have width 32 bits.

The low-word multiplication uses the full 64-bit unsigned product:

\[
P = X_L \times Y_L
\]

The returned low word is:

\[
R_L = \bigl((P \bmod 2^{32}) + B_L\bigr) \bmod 2^{32}
\]

The calculation records whether that low-word addition generates a carry out of bit 31. Let that carry be \(c\), either zero or one.

The returned high word is the low 32 bits of:

\[
X_HY_L + X_LY_H
+ \left\lfloor P / 2^{32}\right\rfloor
+ B_H + c
\]

This establishes the intended word-level multiply-and-add behavior:

\[
R = (B + X Y) \bmod 2^{64}
\]

where each 64-bit operand is formed from its corresponding high and low words.

The high-by-high product does not contribute to the low 64 bits and is not included in the shown expression. No saturation, overflow error, or extra carry result is returned.

Some high-word operations are rendered with signed integer types in the supplied representation. The modulo interpretation describes the recovered word-level calculation; it is not a claim that translating those expressions into signed arithmetic in another language would be defined for every input.

## Side effects

### Established effects

The function creates local temporary storage and passes addresses of that storage to the dependency.

Its visible dependency-call counts are:

| Condition | Calls |
|---|---:|
| Argument is zero | 0 |
| Argument is nonzero and selector pair is zero | 2 |
| Argument is nonzero and selector pair is nonzero | 4 |

No explicit global access, allocation, deallocation, persistent state update, or direct write through the input argument is shown.

### Effects not established

The dependency may have effects beyond populating the supplied temporary storage. Its implementation is not supplied. Consequently, this contract does not assert that it:

- Leaves globals unchanged.
- Leaves the source region unchanged.
- Accesses only eight bytes.
- Preserves temporary values outside its designated destination.
- Returns normally for invalid input.
- Allocates no memory.
- Retains no supplied addresses.
- Releases no memory.
- Performs an atomic transfer.

The wrapper provides no visible cleanup or recovery mechanism if a dependency call fails to return normally.

### Ownership and lifetime

The shown function contains no explicit transfer of ownership and no explicit release operation.

Under the copying interpretation, the input is borrowed for the duration of the calls. However, the dependency’s retention and ownership rules remain unknown. In particular, passing local destination addresses does not by itself prove that the dependency refrains from retaining them.

## Dependencies

### F-dep-01

**Identity status:** document-local opaque function ID for the single external dependency in the supplied evidence. No independently verified dependency identifier was provided.

Every visible invocation has three arguments:

1. A destination-like address referring to a local pair of 32-bit words.
2. An address-like value derived from the input argument.
3. The numeric value 8.

The invocation sequence is:

| Order | Destination role | Source-like offset | Condition |
|---:|---|---:|---|
| 1 | Result pair | 0 | Nonzero input |
| 2 | Selector pair | 8 | Nonzero input |
| 3 | Selector pair reused as multiplicand | 16 | Selector pair was nonzero |
| 4 | Multiplier pair | 8 | Selector pair was nonzero |

The function consumes local words after these calls without a visible initialization or success check for each destination. Therefore, the expected data behavior depends on F-dep-01 populating the relevant words adequately.

**Inferred intent:** an eight-byte transfer from the second argument to the first.

**Unknown:** actual transfer semantics, byte order, alignment restrictions, partial-transfer behavior, overlap rules, fault handling, return convention, synchronization, and unrelated state changes.

## Acceptance examples

These are conditional acceptance cases derived from the supplied static evidence. No runtime tests have been performed.

Except for the zero-input case, numeric examples assume that F-dep-01 copies exactly eight bytes into the designated destination, that the field encoding matches the recovered low/high word pairing, and that there are no unrelated mutations.

### Zero input

Given an argument of zero:

- The result is `0x0000000000000000`.
- F-dep-01 is not invoked.

### Zero selector suppresses the arithmetic path

Given:

- Base: `0x1122334455667788`.
- Initial selector at relative offset 8: zero.

Expected:

- Exactly two dependency calls.
- No invocation with source-like offset 16.
- Result: `0x1122334455667788`.

### Ordinary multiplication and addition

Given stable fields:

- Base: 5.
- Multiplier: 3.
- Multiplicand: 7.

Expected:

- Four dependency calls.
- Result: 26.

### Carry from the low result word

Given stable fields:

- Base: `0x00000000FFFFFFFF`.
- Multiplier: 1.
- Multiplicand: 1.

Expected result:

`0x0000000100000000`

This exercises carry propagation from the low-word addition into the high word.

### High half of the low-word product

Given stable fields:

- Base: zero.
- Multiplier: `0x00000000FFFFFFFF`.
- Multiplicand: `0x00000000FFFFFFFF`.

Expected result:

`0xFFFFFFFE00000001`

This exercises both halves of the unsigned 32-by-32 product.

### Cross-word contribution

Given stable fields:

- Base: zero.
- Multiplier: 2.
- Multiplicand: `0x0000000100000000`.

Expected result:

`0x0000000200000000`

A high-only nonzero selector likewise takes the arithmetic path; the selector test examines both words.

### Overflow beyond 64 bits

Given stable fields:

- Base: zero.
- Multiplier: `0x0000000100000000`.
- Multiplicand: `0x0000000100000000`.

Expected result:

`0x0000000000000000`

The mathematical product is \(2^{64}\); only its low 64 bits are returned.

### Selector changes between reads

Given:

- Base: 10.
- The first offset-8 transfer supplies 1.
- The offset-16 transfer supplies 7.
- The second offset-8 transfer supplies 4.

Expected result:

38.

The first value selects the branch; the later value is the multiplier. If the later transfer instead supplies zero, the result is 10, but all four calls still occur.

## Uncertainties

The following remain **documented_only**:

- Whether F-dep-01 is a copy operation at all.
- Whether each requested eight-byte transfer fully initializes its destination.
- Physical byte order and portable field serialization.
- Pointer representation and the treatment of negative input integers.
- Address arithmetic near representational boundaries.
- Alignment and minimum accessible object size.
- Fault, exception, and partial-transfer behavior.
- Dependency effects on globals, source memory, ownership, and local storage.
- Coherence or synchronization between the two observations of relative offset 8.
- Exact source-language signed-overflow semantics for high-word expressions.

For an ordinary copying implementation, successful execution requires accessible source storage at offsets 0 and 8, plus offset 16 when the initial selector is nonzero. This is a conditional operational precondition, not evidence of validation by this function.

## Evidence

The established findings come solely from static inspection of the supplied function representation:

- An initial zero check returns a zero 64-bit result.
- The nonzero path contains two unconditional dependency calls with size argument 8.
- Both words of the second temporary pair participate in the branch condition.
- The branch contains two additional calls, accessing offset 16 before rereading offset 8.
- The calculation includes an unsigned 32-by-32 product, its upper half, two cross-word products, the original result words, and a low-word carry.
- The final return concatenates a high 32-bit word and a low 32-bit word.

The base-plus-product interpretation depends on the stated dependency assumptions. No dependency implementation, caller contract, runtime trace, or runtime test result was supplied.
