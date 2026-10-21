<!-- Trusted provenance: F-6f91c5ea8ecb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6f91c5ea8ecb

## Purpose

F-6f91c5ea8ecb updates three integer fields in a caller-supplied record. It uses an extent supplied directly by the caller, or obtains an extent from F-870c4fcf7295 when the supplied value is zero.

The established transformation produces:

- A first displacement equal to the negated, integer-divided half of a record value.
- A second displacement equal to half the difference between the selected extent and an earlier record value.
- A stored copy of the selected extent.

The arithmetic suggests centering or positioning, but that interpretation is **inferred intent**, not an established requirement. The evidence does not identify coordinates, units, a rendering system, or the meanings of the record’s other fields.

## Interface

### Inputs

The function receives two arguments:

1. **Record reference.** A pointer to storage addressed as consecutive signed-integer elements. Elements are numbered from zero in this contract.
2. **Requested extent.** A signed integer. Zero selects the dependency-assisted path; every nonzero value, including a negative value, selects the direct path.

The relevant record elements are:

| Element | Established role |
|---|---|
| 0 | Value read to calculate the first displacement |
| 1 | Value captured at entry and used in extent selection and the second displacement |
| 5 | Destination for the first displacement |
| 6 | Destination for the second displacement |
| 7 | Destination for the selected extent |

The function does not directly access elements 2, 3, or 4.

### Width and layout

The supplied representation uses signed `int` types for both arguments’ integer components, the captured value, the dependency result, and the record elements. It does not independently establish the target’s integer width, byte order, alignment requirements, or machine-level overflow behavior.

Let **W** denote the signed-integer width and **S** its storage size in bytes. Element *n* begins *n × S* bytes from the record base. A 32-bit, four-byte integer model would place the accessed elements at byte offsets 0, 4, 20, 24, and 28, but that concrete model remains **documented_only** pending target evidence.

The constants 2, 10, and 12 are exact. The evidence shows no explicit wider intermediate arithmetic.

### Output and preconditions

There is no return value. Results are communicated through record elements 5, 6, and 7.

For an ordinary successful execution:

- Elements 0 and 1 must support the required reads.
- Elements 5, 6, and 7 must support the required writes.
- The record must remain valid when execution resumes after any dependency call.
- The relevant arithmetic must have defined semantics under the actual execution model.

No pointer, bounds, sign, or arithmetic-range validation is shown.

## Behavior

### Initial capture and extent selection

The first explicit record access reads element 1. That value is captured before any dependency call or output write.

If the requested extent is nonzero, it becomes the selected extent. F-870c4fcf7295 is not called on this path.

If the requested extent is zero, the function:

- Forms the product of the captured element-1 value and 12.
- Calls F-870c4fcf7295 with that product as its first argument and 10 as its second argument.
- Uses the returned signed integer as the selected extent.

Zero is therefore an input selector, not an invalid value subject to rejection. A dependency result of zero is accepted as the selected extent; it does not trigger another call. Negative requested extents and negative dependency results are likewise used without explicit rejection or clamping.

### Result computation and ordered stores

After extent selection, the function reads record element 0. On the dependency-assisted path, this read occurs after F-870c4fcf7295 returns.

It then performs three explicit stores in this order:

1. **Element 5:** the negation of the quotient obtained by dividing the newly read element-0 value by 2.
2. **Element 6:** the quotient obtained by subtracting the initially captured element-1 value from the selected extent, then dividing that difference by 2.
3. **Element 7:** the selected extent.

The first calculation divides before negating. The second subtracts before dividing. These operation boundaries matter for overflow analysis and for identifying which intermediate values must be representable.

The function does not reread element 1 after the dependency call. Consequently, a dependency-induced change to element 1 does not replace the captured value used for the second displacement. Conversely, because element 0 is read later, a dependency-induced change to element 0 can affect the first displacement.

### Integer division and arithmetic boundaries

Under modern C-style signed division, division by 2 truncates toward zero. Thus an input of 5 produces a first displacement of −2, while an input of −5 produces 2. A difference of −5 produces a second displacement of −2, not −3.

That rounding rule is the appropriate conditional interpretation of the supplied representation; the exact target instruction semantics are not independently provided.

Potential arithmetic-range issues are concentrated in:

- Multiplication of the captured value by 12 on the zero-request path.
- Subtraction of the captured value from the selected extent.

There is no visible saturation, checked arithmetic, or explicit widening for either operation. This contract does not assign wraparound results when signed arithmetic exceeds its representable range.

With conventional signed integers and truncation toward zero, halving any representable value makes the subsequent negation representable. The first displacement therefore does not have the same overflow exposure as directly negating an unhalved minimum signed value.

## Side effects

The function explicitly writes only record elements 5, 6, and 7, in that order. It does not explicitly write element 0 or element 1.

On the nonzero-request path, the supplied body shows no external call and no explicit global access. On the zero-request path, F-870c4fcf7295 may introduce additional effects. Its implementation and contract are unavailable, so this specification does not assume that it preserves globals, leaves the record unchanged, or preserves the record’s lifetime through other references.

The three stores are sequential, not an established atomic update. No synchronization, rollback, or transactional guarantee is visible. If execution fails during an access, arithmetic operation, or dependency call, there is no shown recovery path that completes or reverses earlier effects.

The record is caller-supplied storage. This function contains no explicit allocation, release, or ownership-transfer operation. That observation does not establish the dependency’s memory-management behavior.

## Dependencies

### F-870c4fcf7295

**Call condition:** exactly when the requested extent equals zero.

**Arguments:**

1. The initially captured element-1 value multiplied by 12.
2. The integer constant 10.

**Result use:** the returned signed integer becomes the selected extent for both the element-6 calculation and the element-7 store.

**Established ordering:** the initial element-1 read precedes the call; the element-0 read and all three explicit output stores follow a normal return from the call.

**Unknown contract:** the dependency’s purpose, accepted ranges, return-value meaning, failure signaling, global effects, ownership effects, and termination behavior are **documented_only**. No particular mathematical operation may be attributed to it from these arguments alone.

If the dependency does not return normally, the supplied body establishes no execution of the subsequent output stores.

## Acceptance examples

These are conditional acceptance criteria, not observed test results. Numerical examples assume representable intermediate arithmetic, truncation toward zero, valid storage, and no relevant concurrent modification.

### Direct extent with even values

Given element 0 equal to 8, element 1 equal to 6, and requested extent 20:

- The dependency is not called.
- Element 5 is written with −4.
- Element 6 is then written with 7.
- Element 7 is then written with 20.

### Odd values and negative differences

Given element 0 equal to 5, element 1 equal to 10, and requested extent 5:

- Element 5 becomes −2.
- Element 6 becomes −2 because the difference is −5.
- Element 7 becomes 5.

An implementation that rounds the second displacement downward to −3 would not satisfy this conditional division model.

### Negative nonzero request

Given element 0 equal to −5, element 1 equal to 3, and requested extent −4:

- The dependency is not called.
- Elements 5, 6, and 7 receive 2, −3, and −4, respectively.

Negative input does not select a special error path.

### Dependency-assisted selection

Given requested extent zero and initial element 1 equal to 4:

- The dependency receives 48 and 10.
- If it returns 18 and element 0 is 9 when subsequently read, the stored results are −4, 7, and 18.

If the dependency instead returns zero, element 7 receives zero without a second dependency call.

### Dependency-visible record changes

Suppose initial element 1 is 4. During a normally returning dependency call, suppose element 1 changes to 100, element 0 changes to 12, and the dependency returns 18.

Provided the record remains valid:

- Element 5 becomes −6, using the later element-0 value.
- Element 6 becomes 7, using the captured value 4 rather than 100.
- Element 7 becomes 18.

This criterion distinguishes the established read ordering from a contract that takes one simultaneous snapshot.

## Uncertainties

The following remain **documented_only**:

- Exact integer width, storage size, byte order, and alignment.
- Actual target behavior for signed overflow and exceptional execution.
- The complete record layout and field meanings.
- The dependency’s implementation, side effects, and error conventions.
- Concurrency guarantees and visibility of intermediate stores.
- Whether callers supply negative values or guarantee bounded arithmetic.
- Any domain interpretation such as coordinates, centering, or dimensions.

No error return, diagnostic, explicit exception handling, or fallback behavior is established. Their absence from this body does not prove that invalid inputs execute safely.

## Evidence

**Established static evidence:** one supplied function body, its signed-integer interface, its conditional dependency call, its arithmetic expressions, and its explicit access order.

**Inferred intent:** the two displacement calculations resemble centering operations. No behavioral requirement depends on that interpretation.

**Unestablished runtime properties:** target representation, dependency behavior, failure outcomes, and concurrency behavior.

No runtime tests have been performed. The acceptance examples specify expected behavior under their stated assumptions; they are not execution evidence.
