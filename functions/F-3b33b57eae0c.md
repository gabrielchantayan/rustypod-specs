<!-- Trusted provenance: F-3b33b57eae0c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3b33b57eae0c

## Purpose

F-3b33b57eae0c returns a 64-bit result selected from an input record, a 32-bit auxiliary argument, and two external 32-bit threshold values.

**Established behavior:** flag bits choose between an immediate result, a computed result, a fixed result, and classification of a two-word integer. The result’s upper and lower 32-bit halves have branch-dependent meanings. It is not uniformly a numeric value or a status code.

**Inferred intent:** the integer branch appears to select a representation category based on magnitude. The other branches appear to handle alternative record forms. No semantic names for the flag bits, categories, or thresholds are established.

## Interface

### Inputs

1. **Record reference.** A reference to storage addressed as 32-bit unsigned words. The following fields are relevant:

   | Byte offset | Width | Role |
   |---|---:|---|
   | 0 | 32 bits | Low integer word, or optional addend |
   | 4 | 32 bits | High integer word |
   | 24 | 32 bits | Base value for the computed-result branch |
   | 28 | 32 bits, narrowed to 16 bits | Flag source |

   Offsets assume the four-byte word layout expressed by the supplied source. Bytes between these fields have no observed role.

2. **Auxiliary argument.** A 32-bit unsigned value. Some branches copy its bit pattern into the result’s upper half. One branch condition interprets that same bit pattern as a signed 32-bit integer.

3. **External threshold state.** Two unsigned 32-bit values, designated here as **threshold A** and **threshold B**. Their values, initialization, mutability, and synchronization requirements are not supplied.

### Output

An unsigned 64-bit result. This contract describes it as an ordered pair:

**(upper 32 bits, lower 32 bits)**.

Both halves are bit patterns; an upper half copied from a negative integer’s high word is not sign-extended beyond the 64-bit result.

### Preconditions and ownership

The record reference must support every read reached by the selected branch. The source provides no null check, bounds check, alignment recovery, or memory-fault handling.

The routine does not visibly modify, retain, allocate, or release the record. Ownership transfer is not established. The supplied body contains no function calls.

The record’s physical byte order and ABI alignment requirements are not established by the source alone. The word positions and bit significance used by the computation are established.

## Behavior

### 1. Flag selection and precedence

The routine first obtains the low 16 bits of the word at byte offset 28. The upper 16 bits of that word do not influence the result.

Branch precedence is:

1. Flag bit `0x0001`.
2. Flag bit `0x0004`.
3. When `0x0004` is clear, flag bit `0x0008`.
4. Other relevant flag bits within the selected branch.

Consequently, `0x0001` overrides every other flag. When `0x0001` is clear, `0x0004` selects the integer branch regardless of `0x0008`.

### 2. Immediate branch: flag `0x0001` set

The result is:

- Upper half: auxiliary argument.
- Lower half: zero.

No other record field or external threshold is read on this path.

### 3. Non-integer branches: flags `0x0001` and `0x0004` clear

#### Fixed-result case: flag `0x0008` set

The result is:

- Upper half: auxiliary argument.
- Lower half: 7.

No record field other than the flag source is read.

#### Computed-result case: flag `0x0008` clear

The routine reads the word at byte offset 24.

If flag `0x0800` is set, it also reads the word at byte offset 0 and adds it to the word at offset 24. This addition wraps modulo \(2^{32}\).

The lower result half is then formed from:

- Twice that base or wrapped sum.
- Plus 1 if flag `0x0002` is set, otherwise plus 0.
- Plus 12.

The complete lower-half calculation wraps modulo \(2^{32}\). Overflow does not carry into the upper half.

The upper result half is:

- 2 when flag `0x0002` is set.
- 0 otherwise.

Other flag bits do not contribute to the returned value on this path. In particular, the returned upper half does not preserve the full flag field, and the auxiliary argument is unused.

### 4. Integer branch: flag `0x0001` clear and `0x0004` set

The routine reads the high integer word at offset 4, followed by the low integer word at offset 0. Together these form a 64-bit two-word integer.

#### Auxiliary-dependent shortcut

Before magnitude classification, the routine checks all of the following:

- The auxiliary argument, interpreted as signed 32-bit, is greater than 3.
- The original high integer word is zero.
- The original low integer word is either zero or one.

If all conditions hold:

- Integer zero returns **(0, 8)**.
- Integer one returns **(0, 9)**.

The signed comparison matters: auxiliary values from `0x80000000` through `0xffffffff` do not satisfy this condition, despite being large unsigned values.

No threshold is read on this shortcut.

#### Magnitude formation

If the shortcut does not apply, classification uses an unsigned two-word magnitude:

- If the original high word’s sign bit is clear, both words are unchanged.
- If that sign bit is set, the two-word value is negated using 32-bit modular word arithmetic, including the carry relationship between the low and high words.

The magnitude of the minimum signed 64-bit value is therefore represented as the unsigned bit pattern `0x8000000000000000`; it is not rejected as overflow.

For every classification result below, the returned upper half is the **original high integer word**, not the magnitude’s high word and not a normalized sign indicator.

#### Category selection

The following conditions are evaluated in order. A condition that selects a category ends processing.

1. **Category 1:** magnitude high word is zero and magnitude low word is at most 127.

2. **Category 2:** magnitude high word is zero and magnitude low word is at most threshold A.

   Threshold A is read here only when the first condition did not select category 1 and the magnitude high word is zero.

3. **Category 3:** magnitude high word is zero and magnitude low word is at most threshold B.

   This condition is reached only after category 2 was not selected. Threshold B is not read when the magnitude high word is nonzero.

4. **Category 4:** magnitude high word is zero and magnitude low word is at most `0x7fffffff`.

5. **Category 5 or 6:** all remaining magnitudes use the final comparison described below.

For categories 1 through 6, the lower result half is the category number.

The thresholds are not assumed to be ordered. Earlier successful conditions take precedence even if a threshold exceeds a later fixed boundary.

#### Final category comparison

This stage performs two distinct reads of threshold A:

1. It records whether the first observed threshold A is less than or equal to the magnitude high word.
2. It records whether the magnitude high word equals the second observed threshold A.

If the equality observation is true, the first recorded comparison is replaced with whether the magnitude low word equals `0xffffffff`.

Category selection then follows these rules:

- Category 5 if the resulting comparison is false.
- Category 5 also if the equality observation is true and the magnitude low word is `0xffffffff`.
- Category 6 otherwise.

**Conditional simplification:** if threshold A remains unchanged across these two reads, this stage returns category 5 exactly when the magnitude high word is at most threshold A, and category 6 when it is greater. At equality, every possible low word yields category 5, including `0xffffffff`.

The source does not establish a single threshold snapshot across the invocation.

## Side effects

### Reads

The source-level read order is:

- Flag source first on every path.
- Computed-result branch: offset 24, then offset 0 only if `0x0800` is set.
- Integer branch: offset 4, then offset 0.
- Threshold reads only when reached during classification.

In the integer branch:

- Category 1 and the auxiliary-dependent shortcut require no threshold reads.
- A magnitude with nonzero high word bypasses both low-word threshold comparisons.
- A zero-high-word magnitude may read threshold A, then threshold B, depending on earlier results.
- The final comparison reads threshold A twice, in sequence.

These are source-level dependencies and access order. Exact machine load widths, speculative accesses, atomicity, and ordering against concurrent activity are not established.

### Writes and state changes

The supplied body contains no writes to the record or external threshold state. Its visible result is the returned 64-bit value.

There is no explicit error return, exception generation, diagnostic output, allocation, deallocation, or callback. Invalid storage is outside the established preconditions; no recovery behavior is specified.

## Dependencies

No opaque function dependencies are present in the supplied body.

The routine depends on two external data values:

| Data dependency | Width | Use |
|---|---:|---|
| Threshold A | 32 bits, unsigned | Category 2 low-word limit; later high-word comparison |
| Threshold B | 32 bits, unsigned | Category 3 low-word limit |

Their numerical values are **documented_only** unknowns. No particular integer encoding, storage capacity, or bit-width category can be assigned to them from this evidence.

The reuse of threshold A in two different comparison positions is established behavior, not evidence that the two positions have an independently known semantic relationship.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They have not been executed. Results are written as **(upper, lower)**.

1. **Highest-priority flag**

   With low flags `0x000d` and auxiliary argument `0x12345678`, the result is **(`0x12345678`, 0)**. The integer and fixed-result flags do not override `0x0001`.

2. **Fixed non-integer result**

   With low flags `0x0008` and auxiliary argument 23, the result is **(23, 7)**.

3. **Computed result without the optional addend**

   With low flags zero and the offset-24 word equal to 5, the result is **(0, 22)**.

   Changing the low flags to `0x0002` produces **(2, 23)**.

4. **Computed result with modular addition**

   With low flags `0x0800`, the offset-24 word equal to `0xffffffff`, and the offset-0 word equal to 2, their wrapped sum is 1. The result is **(0, 14)**.

5. **Integer shortcut**

   With low flags `0x0004`, integer value one, and auxiliary argument 4, the result is **(0, 9)**.

   With the same record and auxiliary argument `0x80000000`, the shortcut does not apply. Magnitude classification returns **(0, 1)**.

6. **Small negative integer**

   With low flags `0x0004`, original high word `0xffffffff`, and original low word `0xffffffff`, the integer represents negative one. Its magnitude is one, and the result is **(`0xffffffff`, 1)**.

7. **Threshold precedence**

   With integer magnitude 128 and no applicable shortcut:

   - If threshold A is at least 128, category 2 is selected.
   - If threshold A is below 128 and threshold B is at least 128, category 3 is selected.
   - If both are below 128, category 4 is selected.

   These outcomes assume the stated values at the corresponding reads.

8. **Final-stage equality boundary**

   Suppose the magnitude high word is nonzero and equals a stable threshold A. Both a low word of zero and a low word of `0xffffffff` select category 5.

   If that high word instead exceeds the stable threshold A, category 6 is selected.

9. **Minimum signed 64-bit input**

   With original words **(`0x80000000`, 0)**, magnitude formation preserves that unsigned magnitude bit pattern. The returned upper half remains `0x80000000`. The lower category depends on the final threshold-A observations.

## Uncertainties

- **documented_only — Semantic meaning:** flag names, category meanings, record type, and auxiliary-argument purpose are unknown.
- **documented_only — Threshold values:** neither threshold’s value nor initialization is supplied.
- **documented_only — Concurrent changes:** no synchronization or stable-snapshot guarantee is established for the record or thresholds. The stable-threshold simplification is conditional.
- **documented_only — Platform details:** physical endianness, alignment requirements, exact generated memory accesses, and invalid-access behavior are not established.
- **Inferred intent:** magnitude-based representation selection is plausible, but does not justify assigning concrete encoding names to categories 1–6.
- **Established limit:** absence of explicit writes or calls in this body does not establish broader system purity, memory ownership policy, or concurrency safety.

## Evidence

This contract is based solely on static inspection of the supplied function body.

The evidence establishes branch precedence, referenced word offsets, narrowing of the flag source, signed interpretation of the auxiliary shortcut comparison, 32-bit modular arithmetic, two-word magnitude formation, result-half construction, conditional threshold reads, and absence of explicit writes or calls.

No external declarations, callers, threshold initializers, binary-level access details, or runtime observations were supplied. No runtime tests have been performed.
