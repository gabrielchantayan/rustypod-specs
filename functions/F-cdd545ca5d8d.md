<!-- Trusted provenance: F-cdd545ca5d8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cdd545ca5d8d

## Purpose

F-cdd545ca5d8d returns an integer computed from two descriptors. Each descriptor exposes four integer fields. The computation has two paths:

- When the descriptors’ fourth fields compare equal, return the difference between their first fields, shifted right by two bits.
- When their fourth fields differ, obtain an integer factor from F-dc364b41daca and combine it with three shifted differences drawn from the descriptors.

The field arrangement and four-byte scaling suggest a distance computation over segmented storage. That interpretation is **inferred intent**, not an established interface guarantee. The supplied evidence does not establish that any field is a valid address, that either descriptor denotes an iterator, or that the result measures a particular application-level unit.

## Interface

### Inputs and layout

The function accepts two pointers to integer storage, described here as the **first descriptor** and **second descriptor**. Four zero-based slots are relevant:

| Slot | First descriptor | Second descriptor |
|---|---|---|
| 0 | Used in both paths | Used in both paths |
| 1 | Used only when slot 3 differs | Not read directly |
| 2 | Not read directly | Used only when slot 3 differs |
| 3 | Used for branch selection and, on the differing-slot path, further computation | Used for branch selection and, on the differing-slot path, further computation |

These are integer slots, not established pointer-valued fields. Slot spacing is the size of the source integer type. A four-byte slot width, and consequently a sixteen-byte descriptor extent, would follow from a 32-bit integer target; that target width is not independently established by the supplied fragment.

The arithmetic’s explicit right shift by two represents bit scaling by four. It does not itself establish the width of an integer slot.

### Result

The result has the source integer type. There is no separate success flag, error object, or distinguished failure value in the supplied body.

Let:

- \(A_0\), \(A_1\), and \(A_3\) denote values read from the first descriptor’s corresponding slots.
- \(B_0\), \(B_2\), and \(B_3\) denote values read from the second descriptor’s corresponding slots.
- \(S(x)\) mean a right shift of the integer value \(x\) by two bits.
- \(K\) denote the integer returned by F-dc364b41daca.

For stable descriptor contents, the equal-slot path returns:

\[
S(A_0-B_0).
\]

For stable descriptor contents, the differing-slot path returns:

\[
\bigl(S(A_3-B_3)-1\bigr)K
+S(A_0-A_1)
+S(B_2-B_0).
\]

These formulas describe the selected arithmetic, not a promise that all fields are captured in a single snapshot. In particular, the dependency call precedes the descriptor reads used in the differing-slot return expression.

### Preconditions and validity

Both descriptor pointers must permit the reads actually performed. The function contains no null check, bounds check, alignment check, or field-consistency validation.

At minimum:

- Both slot-3 fields must be readable for branch selection.
- If those fields compare equal, both slot-0 fields must also be readable.
- If they differ, the additional reads of first-descriptor slot 1 and second-descriptor slot 2 must be valid, as must the repeated slot-3 reads.

The descriptors need not be distinct according to the visible implementation. Aliasing is not rejected.

Arithmetic must remain within the defined domain of the applicable integer model for a defined source-level result. The supplied evidence does not establish a portable wraparound contract for signed overflow.

## Behavior

### Branch selection

The function first compares slot 3 of the two descriptors.

The relative order of the two reads used by that comparison is not specified by the supplied expression. Both comparison operands are evaluated before the selected branch proceeds.

The branch decision uses the values obtained for that comparison. It is not recomputed after a dependency call.

### Equal-slot path

When the compared slot-3 values are equal:

1. The function obtains the two slot-0 values.
2. It subtracts the second descriptor’s value from the first descriptor’s value.
3. It shifts that difference right by two bits.
4. It returns the shifted result.

There is no direct call to F-dc364b41daca on this path. Slots 1 and 2 are not directly read on this path.

The relative order of the two slot-0 reads is not established. There is no explicit validation that their difference is a multiple of four.

### Differing-slot path

When the compared slot-3 values differ:

1. The function reads slot 3 of the first descriptor again to obtain the argument for F-dc364b41daca.
2. It invokes F-dc364b41daca once with that integer argument.
3. If that call returns normally, its integer result supplies the factor \(K\).
4. The function evaluates the three-term return expression using descriptor values read after the call.

The return expression combines:

- A slot-3 difference, shifted right by two, then reduced by one, then multiplied by \(K\).
- First-descriptor slot 0 minus first-descriptor slot 1, shifted right by two.
- Second-descriptor slot 2 minus second-descriptor slot 0, shifted right by two.

The subtraction of one occurs **after** shifting the slot-3 difference. Each descriptor difference is shifted separately. The behavior is not equivalent in general to summing unshifted differences and shifting once.

The dependency’s return value is not checked. Zero and negative values participate in the same arithmetic as positive values.

### Reads across the dependency boundary

The differing-slot path does not retain the comparison’s slot-3 values for the final calculation. It rereads slot 3 for the dependency argument and reads descriptor fields again after the dependency returns.

Accordingly:

- The dependency argument need not equal the value observed during branch selection if the relevant storage changes between those reads.
- Changes made by the dependency to descriptor storage can affect the returned arithmetic.
- A dependency-induced change that makes the slot-3 fields equal does not switch execution to the equal-slot path.
- If the dependency invalidates storage needed by subsequent reads, the supplied function provides no protection.

The return expression does not establish a left-to-right order for its descriptor reads. Its arithmetic grouping is established; an exact chronological order among those reads is not.

### Integer arithmetic and edge cases

For nonnegative representable differences, shifting right by two discards the low two bits and is equivalent to taking the floor after division by four.

For negative signed values, right-shift behavior depends on the applicable source or target semantics. An arithmetic right shift would round toward negative infinity, not toward zero. The supplied fragment does not independently establish that negative-shift rule.

Potential signed-overflow points include the differences, the decrement of the shifted slot-3 difference, the multiplication by the dependency result, and the additions forming the result. No widening operation or explicit overflow handling is visible. A 32-bit wrapping interpretation must not be assumed solely from this fragment.

## Side effects

The supplied body contains:

- Reads from the two descriptors.
- A conditional call to F-dc364b41daca.
- An integer return.

It contains no direct descriptor writes, explicit allocation, explicit deallocation, ownership transfer, logging, or error reporting.

This is not evidence that the function is globally side-effect-free. On the differing-slot path, effects of F-dc364b41daca are part of the observable execution. Its behavior may include state changes not shown here.

Passing an integer field to that dependency does not establish whether the dependency can reach either descriptor through shared state. No guarantee of global preservation, storage preservation, or memory ownership follows from the call signature.

The supplied body has no explicit recovery path if a read is invalid or the dependency does not return normally.

## Dependencies

### F-dc364b41daca

**Invocation condition:** The two slot-3 values obtained during branch selection differ.

**Visible argument:** One integer, obtained by rereading slot 3 of the first descriptor after branch selection.

**Visible result:** One integer used as a multiplicative factor in the differing-slot calculation.

**Invocation count:** One direct invocation on the differing-slot path; none on the equal-slot path.

**Established sequencing:** The dependency completes before evaluation of the differing-slot return expression.

**Unknown contract:** Valid argument range, semantic meaning, side effects, ownership behavior, failure modes, and guarantees about accessible memory are documented_only. The caller neither validates the result nor substitutes an alternative when it is zero or negative.

## Acceptance examples

These are conditional acceptance cases derived from the supplied expressions. They are not runtime observations. Unless stated otherwise, descriptor contents remain stable, required reads are valid, all arithmetic is representable, and the dependency returns normally.

### Equal slot 3, positive difference

If both slot-3 values are 100, first-descriptor slot 0 is 1016, and second-descriptor slot 0 is 1000:

- F-dc364b41daca is not called.
- The result is \(16\) shifted right by two, or **4**.
- Slots 1 and 2 do not contribute.

### Equal slot 3, non-multiple-of-four difference

If slot 3 compares equal and the slot-0 difference is 11, the result is **2**. The function does not reject the unaligned difference or require exact divisibility by four.

If instead the difference is −11, the result is **−3 under arithmetic-right-shift semantics**. That negative result is conditional on the shift model.

### Differing slot 3, nonzero factor

Suppose:

- First descriptor: slot 0 = 1012, slot 1 = 1000, slot 3 = 108.
- Second descriptor: slot 0 = 2000, slot 2 = 2020, slot 3 = 100.
- F-dc364b41daca returns 16 without changing the descriptors.

The dependency receives 108. The result is:

\[
((8\mathbin{\text{ shifted right by }}2)-1)\cdot16
+(12\mathbin{\text{ shifted right by }}2)
+(20\mathbin{\text{ shifted right by }}2)
=24.
\]

### Differing slot 3, zero factor

With the same descriptors, if the dependency returns zero, the result is **8**. The dependency is still invoked, and the two within-descriptor differences still contribute.

### Dependency changes a field

Use the nonzero-factor example, but let the dependency change first-descriptor slot 3 from 108 to 112 and return 16, leaving the other relevant fields unchanged.

The dependency argument is still 108. The subsequent slot-3 difference is 12, and the result is **40**. This demonstrates that the final calculation uses post-call reads rather than a preserved comparison snapshot.

### Identical descriptor pointers

If both inputs point to the same stable, valid descriptor, slot 3 compares equal and the two slot-0 values are equal. The result is **0**, with no dependency call.

## Uncertainties

The following remain **documented_only**:

- Exact integer width, alignment, and binary layout on the original target.
- Whether descriptor fields represent addresses, offsets, identifiers, or another quantity.
- The intended unit and valid range of the result.
- Negative signed-right-shift semantics and machine behavior for overflowing arithmetic.
- Whether callers enforce multiples of four, ordering constraints, or descriptor consistency.
- The complete contract and side effects of F-dc364b41daca.
- Concurrency, asynchronous mutation, or volatile-access requirements.
- Any higher-level ownership or lifetime guarantees.

The segmented-distance interpretation is inferred intent only. It must not supply missing validation, symmetry, or ownership guarantees.

## Evidence

The supplied static fragment establishes the field accesses, branch condition, conditional dependency call, arithmetic grouping, lack of explicit writes, and lack of explicit validation.

The ordering claims follow only where the supplied control flow establishes sequencing: branch selection precedes the dependency call, and that call precedes evaluation of the differing-slot return expression. No stronger ordering among arithmetic operand reads is asserted.

No implementation outside the supplied fragment was inspected. No runtime tests were performed. The acceptance examples are mathematical consequences under their stated conditions, not measured results.
