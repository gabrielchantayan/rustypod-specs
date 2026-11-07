<!-- Trusted provenance: F-9c6cf2acef37; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9c6cf2acef37

## Purpose

F-9c6cf2acef37 conditionally transforms the word array referenced by its first argument, using words referenced by its second argument, then may update the first argument’s stored length.

The established behavior is an in-place, low-index-first arithmetic transformation with explicit 32-bit carry propagation. Under additional representation assumptions, the arithmetic is consistent with multiword subtraction followed by removal of high zero words. Those assumptions are not established: four external factors control length interpretation, word interpretation, word storage, and length storage.

The function returns a 32-bit status value:

- **0:** the ordinary completion path, including the path where the initially evaluated second length is zero.
- **1:** a particular dependency invocation returned a nonzero 32-bit value.

A return of 1 does not imply rollback. Array writes precede the dependency invocation.

## Interface

### Arguments and record layout

The function receives two pointers, designated **destination** and **source**. Each points to a record whose visible layout contains:

| Record offset | Visible use |
|---|---|
| 0 bytes | A 32-bit stored length field |
| 4 bytes | A 32-bit field interpreted as the address of a word array |

Each array entry occupies four bytes. Arithmetic words have a 32-bit bit pattern; certain operations interpret that pattern as unsigned, while loop comparisons use signed integer interpretation.

The destination record and its array are writable. The source record and its array are read by this function, but overlapping storage can make destination writes affect later source reads.

No array capacity, allocation size, ownership flag, or independent validity field is visible.

### External representation state

Four externally stored factors participate in the operation:

- **Length-decoding factor:** multiplies a record’s stored length.
- **Word-decoding factor:** multiplies an array word before arithmetic or zero comparison.
- **Word-encoding factor:** multiplies a computed word before storage.
- **Length-encoding factor:** multiplies the final retained word count before storage.

These names describe use, not proven algebraic properties. The evidence does not establish that either encoding factor is an inverse of its corresponding decoding factor, or that any factor equals 1.

The function does not take an entry snapshot of all four factors. It reads them at their visible use sites. Repeated uses therefore cannot generally be replaced by a single initial value without additional stability assumptions.

### Preconditions for ordinary, defined use

The caller must supply readable records and valid addresses for every array access actually reached. Destination writes must target writable storage.

For conventional nonnegative-length operation, additional requirements include:

- The initially decoded destination length is at least one when the transformation branch is entered.
- The source traversal does not extend beyond readable source storage or writable destination storage.
- Length calculations and index/address calculations remain within the intended machine representation.
- Any overlapping records or arrays are acceptable to the caller under the access ordering below.

The function does not check these conditions. In particular, it does not compare source length against destination capacity.

### Result and ownership

The result is a 32-bit value, observably restricted to 0 or 1 along the shown return paths.

The function contains no direct allocation or release operation. It does not establish ownership transfer for either record or array. The ownership behavior of its dependency is unknown.

## Behavior

### 1. Initial length evaluation and branch selection

The function first evaluates and retains the product of the length-decoding factor and the destination’s stored length. This retained value supplies the later destination-tail limit and the starting point for final trimming.

It then evaluates the decoded source length for an initial nonzero test.

If that result is zero:

- No array transformation occurs.
- No dependency is invoked.
- No length field is written.
- The function returns 0.

This is not a complete no-read path: the initial destination-length computation and source-length test have already occurred.

If the initial decoded source length is nonzero, processing continues. This test is not a positivity check. A negative result also enters the transformation branch.

### 2. Source-covered word transformation

A 32-bit carry accumulator starts at zero.

The first traversal starts at index zero and continues while the index is less than the currently evaluated decoded source length, using a signed comparison. The source length and its decoding factor are consulted again for loop conditions; the traversal is not bounded solely by the initial nonzero test.

At each visited index, the observable arithmetic sequence is:

1. Read the source word and form its decoded 32-bit value.
2. Form the bitwise-complement value of that decoded source word.
3. Read the destination word and form its decoded 32-bit value.
4. Add the complemented source value and decoded destination value, retaining their low 32 bits and recording the unsigned carry-out.
5. Add the incoming carry accumulator to those low 32 bits, again recording unsigned carry-out.
6. Replace the accumulator with the sum of the two carry-out indicators.
7. Multiply the resulting low word by the word-encoding factor and store it into the destination array at the current index.

The carry accumulator is therefore not an ordinary signed overflow flag. It is derived from two explicit unsigned 32-bit carry calculations.

For a valid execution starting with carry zero, its value remains 0 or 1. The implementation nevertheless computes it as the sum of two carry indicators rather than as a single comparison.

A destination write occurs before the next index is processed. Source/destination overlap can consequently affect later reads. The source array is not copied or snapshotted.

### 3. Destination-tail transformation

After the source-covered traversal, the decoded source length is evaluated again to select the starting index of a second traversal. It is not necessarily the number of iterations actually completed by the first traversal if relevant storage or external state changed.

The second traversal continues while its index is below the initially retained decoded destination length.

Before each condition comparison, including the final unsuccessful comparison, the function reads the word-decoding factor into a retained temporary value. This detail matters because that retained value is later used for trimming.

For each visited destination-tail word:

- Read and decode the destination word.
- Subtract one from its 32-bit bit pattern.
- Add the incoming carry accumulator to that result, retaining the low 32 bits.
- Set the next carry to the sum of:
  - one if the decoded destination word was nonzero, otherwise zero; and
  - the unsigned carry-out from adding the incoming carry to the decremented value.
- Encode and store the resulting destination word.

The destination-tail traversal does not stop early merely because carry has reached a particular value.

Under stable factors and ordinary lengths, this traversal propagates the arithmetic through destination words above the source-covered portion.

### 4. Low-word completion and dependency branch

After both traversals, the function reads and decodes destination word zero.

If that decoded value is not the 32-bit all-ones pattern, the function:

- Reads destination word zero again.
- Decodes it using the factor read for that expression.
- Adds one.
- Encodes and stores the result back to word zero.

The second read is observable. It is not specified as reuse of the value tested by the preceding condition.

If the tested decoded value is the all-ones pattern, the function invokes **F-4f8dc40081ad**, passing the destination record pointer.

The dependency’s 32-bit return value controls continuation:

- **Zero:** continue to trimming.
- **Any nonzero value:** return 1 immediately.

The visible comparisons treat that return value as unsigned. Both a return value of 1 and every value greater than 1 select the immediate return of 1.

No direct length update follows a nonzero dependency return. Earlier array writes remain observable, and the dependency may itself have modified the record or other state.

### 5. High-word trimming and length update

If processing reaches trimming, the starting bound is the destination length retained at entry, not a freshly decoded destination length.

The comparison multiplier is the word-decoding factor retained by the final condition evaluation of the destination-tail traversal. It is not necessarily the current external factor. In particular, the dependency may have changed external state without changing this retained multiplier.

Trimming works from the highest candidate word toward index zero:

- Decrement the candidate index before examining the corresponding word.
- Stop descending when the candidate index is below one.
- Otherwise, continue descending while the candidate word multiplied by the retained comparison multiplier is zero.
- Perform a final zero comparison at the stopping index.
- Retain zero words if that final comparison is zero; otherwise retain one more than the stopping index.

The function then multiplies the retained count by the length-encoding factor read at that stage and stores the result in the destination record’s length field. It returns 0.

Trimming is based on the multiplied comparison result, not necessarily on whether the raw stored word equals zero. Without constraints on the retained multiplier, a nonzero raw word can compare as zero.

No discarded array words are cleared. Only the stored length changes.

## Side effects

### Direct writes

The visible direct writes are:

1. Destination words in increasing index order during the source-covered traversal.
2. Destination words in increasing index order during the tail traversal.
3. Potentially destination word zero during low-word completion.
4. The destination stored length, if trimming completes.

The function does not directly write the source record or source array as separately designated objects. Aliasing can nevertheless make any destination write change storage later accessed through the source.

### Aliasing and mutable state

No non-aliasing contract is demonstrated.

Consequences include:

- Identical arrays cause each first-traversal source word to be read before the corresponding destination word is overwritten.
- Shifted overlap can expose a prior destination write as a later source value.
- Overlap with record fields can alter subsequent length or address reads.
- Array-address fields are used at individual access sites rather than being established as immutable entry snapshots.

The dependency receives the destination record itself. Its effects on that record, its array, external factors, allocation ownership, or unrelated state are not established. Continuation after the call must not assume preservation beyond the behavior explicitly retained within this function.

### Error behavior

There is no visible argument validation, capacity check, bounds-error result, or arithmetic-error result.

A return of 1 specifically reflects a nonzero dependency result; it is not a general report of invalid inputs or subtraction underflow.

The behavior of invalid addresses, unavailable storage, arithmetic outside the target’s supported interpretation, or invalid negative indices is not established as a recoverable error path.

## Dependencies

### F-4f8dc40081ad

**Invocation condition:** the decoded destination word zero, after both arithmetic traversals, equals the all-ones 32-bit pattern.

**Argument:** the destination record pointer.

**Return interpretation:** zero permits trimming; every nonzero 32-bit result makes F-9c6cf2acef37 return 1.

**Documented-only unknowns:**

- The dependency’s intended operation.
- Whether it modifies the length field, address field, array, or external representation factors.
- Whether it allocates, releases, or replaces memory.
- Whether it preserves any global state.
- The meaning of its individual nonzero results.

No other function invocation is visible.

## Acceptance examples

These examples are conditional contracts derived from static arithmetic, not observed executions.

### Initially zero source length

Suppose the initial decoded source length is zero and both initial record reads are valid.

Expected behavior:

- Return 0.
- Make no direct array or length writes.
- Do not invoke F-4f8dc40081ad.

The destination’s stored length need not itself be zero for this path.

### Single-word subtraction under identity representation

Assume all four factors equal 1 and remain stable, both decoded lengths are one, arrays and records are disjoint, and all accesses are valid.

With destination word 9 and source word 4:

- The first arithmetic traversal stores 4.
- Low-word completion changes it to 5.
- Trimming retains one word.
- The function stores destination length 1 and returns 0.

With both words equal to 4:

- The first traversal produces the all-ones pattern.
- F-4f8dc40081ad is invoked.
- Subsequent behavior depends on its return and side effects.

Thus the equality case cannot be fully specified as ordinary subtraction without the dependency contract.

### Borrow propagation across two words

Under stable identity factors, a destination length of two, a source length of one, and disjoint storage, let destination words be `[0, 1]` and the source word be `[1]`, listed low word first.

Expected direct behavior:

- The source-covered traversal stores `0xFFFFFFFE` in destination word zero.
- The tail traversal stores zero in destination word one.
- Low-word completion changes word zero to `0xFFFFFFFF`.
- The dependency is not invoked, because its branch decision occurred before that increment.
- Trimming reduces the stored destination length to one.
- Return 0.

This example distinguishes the pre-increment dependency test from the final low-word value.

### Nonzero dependency result

Assume execution reaches the dependency and it returns a nonzero value.

Expected behavior:

- Return 1.
- Do not perform this function’s trimming or final length write.
- Do not restore earlier word values.
- Preserve no stronger claim about destination state than the dependency’s separate contract permits.

### Zero dependency result with modified state

Assume the dependency returns zero after changing the destination array address or external factors.

Expected continuation:

- Use the destination address field subsequently read for trimming accesses.
- Use the pre-call retained comparison multiplier for trimming comparisons.
- Use the length-encoding factor read for the final length store.

A contract that assumes all three come from one unchanged representation snapshot would be too strong.

## Uncertainties

- **Inferred intent:** multiword subtraction and normalization under a compatible encoding scheme.
- **Not established:** signedness of the represented mathematical quantity, underflow semantics, or required relation between input lengths.
- **Not established:** actual values, inverses, or stability guarantees for the four external factors.
- **Documented-only:** negative decoded lengths and a nonpositive initial destination length on the active branch. Such inputs can skip arithmetic loops yet still reach word-zero or negative-index accesses.
- **Documented-only:** exact behavior when signed multiplication or signed index arithmetic exceeds its representable range. Explicit carry operations establish 32-bit unsigned addition behavior, but do not supply a complete language-level overflow contract for every other expression.
- **Not established:** concurrent modification, volatile-access requirements, or ordering of operands within individual multiplication expressions.
- **Not established:** dependency ownership and state-preservation guarantees.

## Evidence

This contract rests solely on the supplied static function body and dependency mapping.

Established evidence includes the four-byte record-field and array strides, explicit 32-bit carry operations, signed traversal comparisons, repeated source-bound evaluation, retained destination bound, conditional dependency invocation, return paths, and final length write.

The subtraction interpretation is an inference contingent on representation assumptions. Dependency internals, caller invariants, external-factor values, and allocation behavior are unavailable.

No runtime tests have been performed.
