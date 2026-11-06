<!-- Trusted provenance: F-bd6e0c2e1c9d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd6e0c2e1c9d

## Purpose

F-bd6e0c2e1c9d classifies a supplied 32-bit value against two stored bounds. It returns a fixed result below the first bound or outside a qualifying interval. Inside a nondegenerate interval, it constructs a scaled displacement and delegates the result calculation to F-1e0edc6e0bc9.

The apparent intent is a percentage-like position within an interval: zero below the lower bound, one hundred above the upper bound, and a scaled ratio between them. That intent is not a complete established contract. The supplied evidence does not define F-1e0edc6e0bc9, so division semantics, rounding, and the actual results of delegated calculations remain unknown.

The bounds and supplied value are compared as signed 32-bit integers, despite their unsigned storage. This distinction is observable whenever a word has its most significant bit set.

## Interface

### Inputs

The function accepts:

- A pointer to a sequence of 32-bit words. The first word supplies the first bound; the next word supplies the second bound.
- A 32-bit value supplied directly as an argument.

For this contract, the first bound is called the **lower bound**, the second the **upper bound**, and the supplied value the **position**. These names describe their apparent roles; the function does not require or enforce correctly ordered bounds.

The visible layout is:

| Offset from the supplied pointer | Width | Meaning |
|---|---:|---|
| 0 bytes | 32 bits | Lower-bound bit pattern |
| 4 bytes | 32 bits | Upper-bound bit pattern |

No length, allocation size, ownership flag, or context pointer is supplied.

The first word must be readable on every execution. The second word must be readable only on executions that pass the initial comparison. The evidence does not establish alignment rules, a null-pointer convention, or handling for inaccessible memory.

### Interpretation and widths

The behavioral arithmetic uses the following model:

- Stored bounds and the position are 32-bit words.
- Comparisons interpret each word as a signed, two’s-complement 32-bit integer.
- Word-level subtraction and negation retain 32-bit results.
- A two-word quantity has a low 32-bit word and a high 32-bit word.
- The scaled displacement is represented with 64 bits.

Thus, the bit pattern `0xFFFFFFFF` is interpreted as −1 during comparisons, not as 4,294,967,295.

The supplied source representation supports this width model through its word operations and 32-bit splitting of wider arithmetic. A separate ABI or type-definition document was not supplied.

### Output

The function returns a signed integer result represented here as 32 bits.

There are three possible result sources:

1. Literal zero when the position is signed-less-than the lower bound.
2. Literal one hundred when the initial comparison passes but the interval-calculation condition fails.
3. The result returned by F-1e0edc6e0bc9 on the interval-calculation path.

There is no separate status channel or visible error object.

## Behavior

### Ordered bound access and classification

The function first reads the lower bound.

If the signed position is less than the signed lower bound, it returns zero immediately. It does not read the upper bound and does not call the dependency on this path.

Otherwise, it reads the upper bound. It enters the calculation path only when both of these conditions hold:

- The signed position is less than or equal to the signed upper bound.
- The two bound words are unequal.

If either condition fails, the function returns one hundred without calling the dependency.

The equality check compares the complete bound words. For 32-bit two’s-complement values, this is also equality of their signed interpretations.

These rules have several consequences:

- A position equal to the lower bound does not take the immediate-zero branch.
- A position equal to the upper bound can enter the calculation path.
- Equal bounds never enter the calculation path.
- Reversed bounds never enter the calculation path.
- The upper bound is read even when the eventual outcome is the fixed result one hundred.
- A below-lower-bound result takes precedence over any interpretation of the upper bound.

### Calculation-path invariants

Let the signed numerical values of the lower bound, upper bound, and position be \(L\), \(U\), and \(P\).

Entering the calculation path establishes:

\[
L \le P \le U,\qquad L \ne U.
\]

Consequently:

\[
L < U,
\]

\[
0 \le P-L \le U-L,
\]

and

\[
1 \le U-L \le 2^{32}-1.
\]

These are consequences of the actual branch conditions, not additional caller preconditions.

The maximum mathematical difference between two signed 32-bit values is 4,294,967,295. Therefore, a valid interval span can exceed the positive range of a signed 32-bit integer even though both endpoints individually fit that type.

### Dependency operands

On the calculation path, the function constructs:

\[
N = 100(P-L)
\]

and

\[
D = U-L.
\]

These are mathematical differences between signed endpoint values, not differences between their unsigned numerical interpretations.

The visible word arithmetic accounts for sign extension and borrow when constructing those quantities. In particular, a wrapped low-word subtraction is not by itself the complete signed difference.

The dependency receives four 32-bit argument words, in this order:

1. The low word of \(N\).
2. The high word of \(N\).
3. The low word of \(D\).
4. The high word of \(D\).

On every reachable calculation path:

\[
0 \le N \le 429{,}496{,}729{,}500
\]

and

\[
1 \le D \le 4{,}294{,}967{,}295.
\]

Accordingly:

- The numerator fits in 64 bits.
- Its high word is between 0 and 99 inclusive.
- The denominator’s high word is zero.
- The denominator’s low word is nonzero, though it may have its most significant bit set.

A dependency implementation that treated only the denominator’s low word as a signed 32-bit divisor would not necessarily implement the apparent intended ratio. The caller supplies a two-word quantity; the dependency’s interpretation remains a separate requirement.

### Sign handling and returned result

The visible implementation includes magnitude normalization and a possible result-sign correction based on the high words of the constructed operands.

Under the calculation-path invariants, both complete operands are nonnegative, and the denominator is positive. The negative-operand normalization paths are therefore not selected, and the final sign correction does not negate the dependency result.

The established externally relevant behavior is that the function calls F-1e0edc6e0bc9 with the operand words described above and returns its result unchanged, if the call returns normally.

The caller does not visibly clamp that result to the range zero through one hundred. It also does not independently reject a negative or otherwise unexpected dependency result.

### Equal and reversed bounds

For equal bounds:

- A position below their common signed value returns zero.
- A position equal to or above that value returns one hundred.

For reversed bounds, where \(L>U\):

- A position below \(L\) returns zero, regardless of its relationship to \(U\).
- A position at or above \(L\) returns one hundred.

No dependency call occurs in either case. Equal bounds avoid the calculation path rather than relying on the dependency to handle a zero denominator.

## Side effects

The function’s own visible memory operations are limited to reading the bound words in the order described above. It contains no explicit write through the input pointer and no explicit allocation, release, or ownership transfer.

On the calculation path, the dependency receives only four scalar words, not the input pointer. This does not prove that the dependency cannot access shared state or the same memory through another reference.

No guarantee is established that F-1e0edc6e0bc9:

- Preserves global or thread-local state.
- Avoids memory allocation or release.
- Leaves the bound storage unchanged through other aliases.
- Returns normally.
- Avoids externally visible effects.

There is no visible reread of either bound after the dependency call. If the dependency changes the original bound storage through another reference, the function’s already-constructed operands are not recomputed.

The two bound reads are not shown as an atomic snapshot. No synchronization or concurrent-update protocol is established. Behavior under concurrent mutation is documented_only.

Invalid-memory behavior is also documented_only. The function provides no visible pointer validation or recovery mechanism; the evidence does not establish a particular fault, exception, or diagnostic.

## Dependencies

### F-1e0edc6e0bc9

This is the only visible called dependency.

**Invocation condition:** The position is within the signed closed interval defined by distinct bounds.

**Argument layout:** Four 32-bit words, ordered as numerator low, numerator high, denominator low, denominator high.

**Numerator:** One hundred times the signed displacement from the lower bound.

**Denominator:** The positive signed distance between the bounds, represented as a two-word nonnegative quantity.

**Result use:** Its returned integer becomes the caller’s result without a reachable sign change or visible range check.

**Established call count:** Exactly one invocation on each calculation-path execution; none on either fixed-result path.

**Unknown semantics:** The supplied evidence does not establish whether the dependency performs division, what rounding rule it uses, whether its result has additional constraints, or what errors and side effects it may produce.

The percentage interpretation would be consistent with a dependency that computes an integer quotient from the two supplied multiword operands. That interpretation remains inferred intent, not established behavior.

## Acceptance examples

The following examples are static acceptance conditions. They have not been executed.

### Fixed-result cases

| Lower bound | Upper bound | Position | Required observable behavior |
|---:|---:|---:|---|
| 10 | Any unread value | 9 | Read only the lower bound; return 0; no dependency call. |
| 10 | 20 | 21 | Read both bounds; return 100; no dependency call. |
| 10 | 10 | 9 | Read only the lower bound; return 0. |
| 10 | 10 | 10 | Read both bounds; return 100; no dependency call. |
| 20 | 10 | 15 | Return 0 after the first bound read. |
| 20 | 10 | 20 | Read both bounds; return 100; no dependency call. |

The “unread value” case does not require the second word to contain any particular value. It also does not establish that an invalid overall pointer or invalid first-word access is acceptable.

### Delegated calculations

For lower bound 10, upper bound 20, and position 15:

- The calculation path is taken.
- The numerator is 500.
- The denominator is 10.
- The dependency arguments are 500, 0, 10, 0.
- The returned value must equal the dependency’s returned value.

If the dependency is separately established to return the integer quotient, the result is 50. Without that dependency contract, 50 is not independently guaranteed.

For lower bound 10, upper bound 20, and position 10:

- The dependency arguments are 0, 0, 10, 0.
- The caller does not bypass the dependency merely because the displacement is zero.

For the same bounds and position 20:

- The dependency arguments are 1000, 0, 10, 0.
- The caller does not directly return the literal one hundred on this endpoint path.

### Signed interpretation

For a lower-bound word of `0xFFFFFFFF`, upper-bound word of 1, and position word of 0:

- The signed bounds are −1 and 1.
- The position is inside the interval.
- The dependency arguments are 100, 0, 2, 0.

For lower bound 0 and position word `0xFFFFFFFF`:

- The position is interpreted as −1.
- The function returns zero without reading the upper bound.

### Maximum span

For lower bound −2,147,483,648, upper bound 2,147,483,647, and position equal to the upper bound:

- The denominator is 4,294,967,295.
- The numerator is 429,496,729,500.
- The argument words are `0xFFFFFF9C`, 99, `0xFFFFFFFF`, 0.

This example requires preservation of the full multiword values. Interpreting either low word alone as the complete signed operand would lose the established arithmetic meaning.

## Uncertainties

The following remain documented_only:

- The dependency’s arithmetic operation, rounding, result constraints, failure behavior, and side effects.
- The complete calling convention and independently verified platform type widths.
- Alignment requirements and behavior for invalid pointers.
- Concurrency guarantees for reading the two bound words.
- Any external invariant requiring ordered bounds or restricting values to a smaller range.
- Whether a broader application assigns special meaning to returned values outside zero through one hundred.

The percentage-like purpose is inferred from scaling by one hundred, the boundary results, and the multiword operand construction. It does not justify assuming that delegated results are percentages.

## Evidence

Established static evidence consists of the supplied function body: its two ordered bound reads, signed comparisons, fixed-result branches, multiword operand construction, single dependency call, and result handling.

The operand ranges and the unreachability of negative-operand correction on the calculation path follow mathematically from those branch conditions under the stated 32-bit model.

No dependency implementation, external interface documentation, runtime observation, or test result was supplied. No runtime tests have been performed.
