<!-- Trusted provenance: F-af3f99fa44cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af3f99fa44cc

## Purpose

F-af3f99fa44cc adds a source pair of integers into a destination pair, then repeatedly adjusts the destination while its second component meets an inclusive threshold. The destination supplies initial operands and receives all results. There is no returned value or separate status.

The adjustment amount and threshold are captured from two successive evaluations of a shared expression. Both captures are stored as `int`, but the shared object’s declaration is absent. Its type, signedness, and width are therefore not established. In particular, the threshold is obtained by negating the shared expression **before** converting the result to `int`; it is not necessarily the negation of the already converted adjustment amount.

Under additional type and conversion assumptions, a stable negative shared value produces carry-like normalization: whole positive thresholds are removed from the second component and counted in the first. That application-level purpose is **inferred intent**, not an unconditional contract.

All behavior below is derived from static evidence. No runtime tests have been performed.

## Interface

### Explicit inputs and output

| Input | Required storage | Role |
|---|---|---|
| Destination reference | Two consecutive readable and writable `int` elements | Initial destination pair and result storage |
| Source reference | Two consecutive readable `int` elements | Contributions to the destination |

Each reference identifies the first element of its pair. The second element is one `int` element later. The evidence does not establish a concrete byte offset.

Normal completion returns no value. Results are communicated through destination storage only.

Source and destination are not required by the supplied body to be disjoint. Identical references and partial overlap are neither checked nor compensated for. Valid overlap affects the result according to the live access order described below.

### Implicit input and captured quantities

A shared expression is evaluated twice. Define:

- \(A\): the result of converting the **earlier evaluation** to `int`.
- \(T\): the result of applying unary negation to the **later evaluation**, under that expression’s type and applicable promotion rules, and then converting the negated result to `int`.

These definitions distinguish two different operations:

| Capture | Required interpretation |
|---|---|
| Adjustment amount \(A\) | Evaluate shared expression, then convert to `int` |
| Threshold \(T\) | Evaluate shared expression, apply unary negation in the applicable expression type, then convert to `int` |

The shared object is not established to have type `int`, or even a particular integer width or signedness. Assignment into `int` storage does not establish the type of the expression being assigned.

The shared object’s definition, initialization, lifetime, ownership, permitted values, and access discipline are unavailable.

### Widths, arithmetic, and conversions

The pair elements and both captured quantities have type `int`. Its concrete width, representation, byte size, alignment, and limits are not established. Neither a 32-bit range nor four-byte spacing may be assumed.

The destination additions and per-adjustment arithmetic operate in `int`. There is no explicit widening, saturation, overflow check, or deliberately wrapping arithmetic. A defined execution requires every reached signed-`int` arithmetic result to be representable.

The capture expressions require separate analysis:

- Conversion to `int` may be value-preserving when the input is representable.
- A wider or unsigned input may require a conversion whose result or failure behavior depends on the applicable language and implementation rules.
- For an integer expression, integer promotions apply before unary negation.
- If the promoted operand is signed, negation must satisfy that signed type’s representability rules.
- If the promoted operand is unsigned, negation is modular in that unsigned type; it is not signed arithmetic overflow.
- An unsigned object of sufficiently small rank may promote to `int`. Its negation must therefore not automatically be classified as unsigned modular negation.

There is no universal requirement that the original shared value have a representable signed negation. That requirement applies only when unary negation actually occurs in a signed integer type.

### Storage and ownership preconditions

Every accessed element must be live, correctly aligned, within accessible storage, and valid for the required read or write. The function does not check these conditions.

Any overlap with shared storage must also be valid under the surrounding object and access rules. The missing shared declaration prevents unconditional acceptance of such aliases.

The body contains no allocation, release, ownership transfer, or retention of either reference.

## Behavior

### Initial updates and capture order

The established semantic order is:

1. The destination’s first component is replaced by its sum with the source’s first component.
2. The shared expression is evaluated and converted to `int` to capture \(A\).
3. The shared expression is evaluated again; unary negation is applied before conversion to `int` to capture \(T\).
4. The destination’s second component is replaced by its sum with the source’s second component.
5. Adjustment eligibility is evaluated using the updated second component and captured threshold.

The source pair is not frozen at entry. A first-component destination write can change storage subsequently read as the source’s second component.

No relative ordering between the two operand reads within either initial addition is established. The ordering between the completed updates and captures is established.

These are semantic access constraints, not a promise about physical memory transactions, atomic operations, or compiled instruction order.

### Adjustment behavior

Adjustment occurs whenever the destination’s current second component is **greater than or equal to \(T\)**. Equality requires adjustment.

Each adjustment:

- Adds \(A\) to the destination’s second component.
- Then increases the destination’s first component by one.

The second-component write precedes the first-component write. Only after both operations does the next eligibility evaluation occur.

Normal completion requires the second component to be strictly less than \(T\). There is no explicit iteration limit, alternative exit, or error branch if adjustments do not move toward that condition.

Let \(P\) and \(Q\) denote the destination components after their respective initial additions. For a defined execution completing after \(N\) adjustments, the result is mathematically

\[
(P+N,\;Q+NA).
\]

This relationship does not authorize replacing the individual operations with bulk arithmetic. Every reached addition must satisfy its own representability requirements.

### Captures remain fixed

Once captured, \(A\) and \(T\) remain fixed throughout the adjustment phase.

Destination writes may change the shared object through a valid alias. Such writes do not refresh either captured quantity.

The two shared evaluations occur separately and in order. No call or explicit intervening write appears between them, but the evidence does not establish every possible external access mechanism. A supported way for the evaluations to observe different values remains unresolved.

Even if both evaluations observe the same numerical value, \(T=-A\) does **not** follow without type and conversion conditions.

### Conditions establishing a negative adjustment and positive threshold

Suppose both evaluations obtain the same negative integer value \(s\), and all of the following hold:

- Conversion of \(s\) to `int` preserves its value.
- Unary negation, after any integer promotions, produces the mathematical value \(-s\) without an arithmetic violation.
- Conversion of that negated result to `int` preserves its value.

Then \(A=s\), and \(T=-s\) is a positive representable `int`. Write this threshold as \(B\).

Under valid access and representable destination arithmetic:

- If \(Q<B\), no adjustment occurs.
- If \(Q=B\), one adjustment makes the second component zero.
- If \(Q>B\), each adjustment removes \(B\) until the second component is below \(B\).
- Whenever adjustment occurs, the final second component lies between zero and \(B-1\), inclusive.
- The first component increases by the number of thresholds removed.

For nonnegative \(Q\), this is a quotient-and-remainder relationship with positive divisor \(B\). The first component must accommodate every required increase.

A negative \(Q\) is already below \(B\) and is left unchanged. No borrowing or compensating decrease of the first component occurs.

These conclusions apply, for example, when the shared object is a stable `int` with a negative value other than a value whose negation is unrepresentable. They do not apply merely because an unknown shared expression is described as “negative.”

### Zero, positive values, and converted captures

For an integer-typed shared expression whose two evaluations both yield zero, integer promotions, negation, and conversion to `int` preserve zero. Thus \(A=0\) and \(T=0\).

- If \(Q<0\), the function completes without adjustment.
- If \(Q\geq0\), the second component never changes, so eligibility remains true. Repeated first-component increases cannot continue indefinitely within a finite signed range.

For a stable positive integer value \(s\), the corresponding positive-value analysis requires value-preserving conversion of \(s\), valid unary negation producing \(-s\), and value-preserving conversion of \(-s\). Under those assumptions, \(A=s>0\) and \(T=-s<0\).

- If \(Q<T\), the function completes without adjustment.
- Otherwise, every adjustment increases the second component, so it cannot become less than \(T\) through representable arithmetic. Continued adjustment eventually reaches an unrepresentable addition.

For unknown shared types or non-value-preserving conversions, classify behavior from the actual captured `int` values \(A\) and \(T\), not from the sign of the original shared value.

In particular, an unsigned shared expression that remains unsigned after promotion undergoes modular negation before the threshold conversion. A wider expression can also require narrowing at either capture. The resulting \(A\), \(T\), and any conversion consequences cannot be recovered without the missing type and execution-model information.

### Errors and partial state

There is no explicit validation, error report, exception-handling contract, or rollback.

Invalid storage access and unrepresentable destination `int` arithmetic have no specified result under this contract. Capture conversion behavior that depends on missing implementation rules remains unresolved; it is not replaced with an assumed wrap or rejection policy.

Earlier destination writes are not guaranteed recoverable output after an invalid execution. For example, a representable second-component adjustment does not establish a valid partial result if the following first-component increase is unrepresentable.

## Side effects

The only direct writes in the body target the destination’s two elements. Source or shared storage may nevertheless change through valid aliases.

| Storage relationship | Consequence |
|---|---|
| Identical source and destination references | Each initial addition doubles that component’s then-current value |
| Destination first component overlaps source second component | The first write changes a later source contribution |
| Shared storage validly aliases destination first component | The first addition can change the expression evaluated for both captures |
| Shared storage validly aliases destination second component | Both captures precede that component’s initial addition |
| Later destination writes affect shared storage | Shared storage changes, but \(A\) and \(T\) remain fixed |

The shared-alias cases require compatible declarations and a valid object arrangement. Neither is established for actual callers.

The source reference is not directly used as a write destination. This does not imply that overlapping source storage remains unchanged.

No external call, allocation, deallocation, input/output, logging, locking, or synchronization appears. The pair update is not atomic or transactional. Thread safety is not established.

## Dependencies

There are no called-function dependencies in the supplied body. No additional opaque function IDs are required, and no callee behavior is assumed.

| Dependency | Established role | Unresolved contract |
|---|---|---|
| Caller-provided pair storage | Supplies operands and receives destination updates | Validity, extent, overlap, and shared-storage relationships |
| Shared expression | Supplies the two capture inputs | Declaration, type, signedness, width, values, lifetime, ownership, and access discipline |
| Integer and conversion model | Determines arithmetic, promotions, negation, and conversion outcomes | Concrete limits and implementation-dependent conversion behavior |
| Object-access model | Determines validity of accesses and proposed aliases | Surrounding declarations and target constraints |

## Acceptance examples

These examples are **conditional static deductions**, not runtime observations. Unless stated otherwise, pair storage is valid and disjoint, shared storage is independent, and every reached destination operation is representable.

### Value-preserving negative captures

Assume both shared evaluations yield −10, conversion to `int` preserves −10, unary negation validly produces 10, and conversion of 10 preserves that value. Thus \(A=-10\) and \(T=10\).

| Initial destination | Source | Final destination | Required behavior |
|---|---|---|---|
| (4, 3) | (2, 5) | (6, 8) | Below threshold; no adjustment |
| (4, 3) | (2, 7) | (7, 0) | Equality requires adjustment |
| (4, 20) | (2, 7) | (8, 7) | Two adjustments |
| (4, −8) | (2, 1) | (6, −7) | No borrowing for a negative second component |

Leaving the equality case at (6, 10) violates the inclusive boundary.

### Identical references and partial overlap

With identical references to (3, 12) and captures \(A=-10\), \(T=10\), the initial additions produce (6, 24), followed by a final pair of **(8, 4)**.

For three consecutive elements initially (1, 2, 3), let the source occupy the first two and the destination occupy the last two. With the same captures, the final storage is **(1, 3, 6)**. The middle element becomes 3 before it contributes to the destination’s second addition. An outcome of (1, 3, 5) incorrectly assumes an entry-time source snapshot.

### Shared storage affected by destination writes

Conditionally assume the shared object is a valid `int` alias of the destination’s first component. Destination (−10, 3) and disjoint source (2, 5) change that component to −8 before capture. The captures are \(A=-8\), \(T=8\), and the final destination is **(−7, 0)**. The subsequent shared value −7 does not replace either capture.

If instead a valid shared `int` object aliases the second destination component, destination (1, −10) and source (2, 25) capture \(A=-10\), \(T=10\) before the second addition changes shared storage to 15. The final destination is **(4, 5)**.

### Zero and positive captures

With integer shared evaluations of zero, destination (1, −3), and source (2, 1), the result is **(3, −2)** without adjustment. A second-component sum of zero instead enters adjustment and cannot complete through entirely representable arithmetic.

Assume a stable shared value of 3 is captured and negated with value-preserving conversions, giving \(A=3\), \(T=-3\). Destination (1, −5) and source (2, 1) produce **(3, −4)** without adjustment. A second-component sum of −3 requires adjustment and cannot reach the exit condition through representable arithmetic.

### Unsigned and wider shared expressions

Conditionally assume a 32-bit unsigned shared expression remains unsigned after promotion, a 32-bit `int`, and both shared evaluations yield 10. Unary negation produces the unsigned value \(2^{32}-10\). The threshold is the conversion of that value to `int`, **not an already established −10**. Without the conversion rule, no concrete threshold or final pair is accepted.

Conditionally assume a wider signed shared expression yields 4,294,967,296 and negation is representable in that wider type, while `int` is 32 bits. Both captures require out-of-range conversion to `int`. Signed negation itself is not the failing representability boundary; the capture outcomes remain dependent on the conversion rules.

### Arithmetic boundaries

If the destination’s first component is the greatest representable `int` and the source contributes one, no wrapping or explicit error result is specified.

If unary negation occurs in a signed integer type and its operand has no representable negation, no defined final result is specified—even if the later second-component sum would not require adjustment.

That signed-negation example does not apply to modular unsigned negation.

## Uncertainties

The following remain **documented_only**:

- Concrete `int` width, limits, representation, alignment, byte size, pointer width, and byte order.
- The shared expression’s declaration, type, signedness, width, and applicable promotions.
- Results or consequences of capture conversions that are not value-preserving.
- Shared-object initialization, lifetime, ownership, permitted range, and actual aliases.
- Whether the two shared evaluations can differ through a supported external mechanism.
- Caller invariants ensuring valid storage and representable arithmetic.
- Actual use of identical references, partial overlap, or shared-storage aliases.
- Supported concurrency, signals, or special-memory behavior.
- Physical memory transactions and compiled instruction ordering.
- Application-level intent and behavior outside the defined execution domain.

These unknowns do not establish signed shared storage, \(T=-A\), disjointness, wrapping destination arithmetic, borrowing, recovery, or ownership transfer.

## Evidence

**Established static evidence:** two references to consecutive `int` elements; an initial first-component update; two ordered shared-expression captures into `int`, with unary negation preceding the second conversion; an initial second-component update; an inclusive adjustment comparison; ordered second-then-first adjustment writes; no returned value; and no called functions.

**Conditional deductions:** carry-like normalization follows when captures produce a negative amount and its positive opposite under valid arithmetic. The acceptance examples state the storage, type, and conversion assumptions needed for their results.

**Inferred intent:** redistribution resembling carry normalization under a suitable negative shared value.

**Unknown:** the shared declaration, target model, conversion rules, caller behavior, and external access mechanisms are absent.

No runtime tests have been performed.
