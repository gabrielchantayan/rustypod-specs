<!-- Trusted provenance: F-5b23ade9b3da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5b23ade9b3da

## Purpose

F-5b23ade9b3da conditionally computes a 16-bit result from a descriptor containing a 16-bit count and a reference to an array of 32-bit words. It first invokes F-8f8b4462085f. A nonzero dependency result causes an immediate return of zero without any descriptor access by F-5b23ade9b3da itself.

If the dependency returns zero, the function examines the word selected by the descriptor’s count. It scans that word from its most significant bit toward its least significant bit. For a stable, nonzero selected word and a positive count, the result is the array position of that word’s highest set bit, expressed as a one-based bit length, reduced modulo 65,536.

**Inferred intent:** the descriptor represents a multiword unsigned integer, with its most significant active word selected by the count. Under that interpretation, the function computes the integer’s bit length. The representation’s normalization and the meaning of the dependency’s result are not established.

The implementation does not handle a stable zero selected word by returning zero. Instead, its scan fails to terminate.

## Interface

### Input

The function accepts one pointer to a descriptor. The visible descriptor fields are:

| Byte offset | Width | Interpretation |
|---|---:|---|
| 0 | 16 bits | Unsigned word count |
| 2 | 16 bits | Not directly accessed |
| 4 | 32 bits | Address-bearing field used as the word-array base |

This layout follows from the supplied typed accesses: the input points to 16-bit units, and the base field is accessed two such units after the descriptor start.

No complete descriptor size, allocation boundary, alignment guarantee, or ownership declaration is supplied. Only these accessed fields are established. The four-byte base-field interpretation reflects the source’s apparent 32-bit address model; behavior on a different data model is not established.

For a count \(n\) and base address \(B\), the selected word starts at:

\[
B + 4n - 4
\]

For positive \(n\), this is word index \(n-1\), counting from zero. Each selected-word access reads 32 bits.

### Output

The declared result is an unsigned integer. Under the apparent 32-bit model, its return width is 32 bits, but every terminating path produces a value between 0 and 65,535 inclusive.

The result is:

- Zero if F-8f8b4462085f returns nonzero.
- Otherwise, a masked arithmetic result determined by the scan and a count read after the scan.

Zero is not an unambiguous indication of an empty value or an error. It can also result from the final 16-bit reduction.

### Preconditions for ordinary, terminating behavior

If the dependency returns zero, ordinary defined execution requires:

- Readable descriptor storage for the count and base fields.
- A usable selected-word address.
- Access alignment and memory mappings compatible with the typed reads.
- A selected word that allows the scan to terminate.
- No unsupported concurrent modification of the accessed objects.

For the conventional bit-length interpretation, additional conditions are needed: the count is positive, the selected word is nonzero, and the relevant descriptor and word contents remain stable throughout the scan and final count read.

These conditions are not validated by the function.

## Behavior

### Dependency gate

The function initializes private scan state and calls F-8f8b4462085f once, without explicit arguments.

If its result is nonzero, F-5b23ade9b3da returns zero immediately. There are no direct descriptor or selected-word reads on this branch.

If its result is zero, execution proceeds to the scan. Descriptor values used by the scan are therefore obtained after the dependency has returned. Any dependency-induced changes to that storage can affect the result.

The meaning of a nonzero dependency result is unknown. Describing it specifically as an error, an empty-input indication, or a validation failure would exceed the evidence.

### Word selection and scanning

The scan begins with:

- A 32-bit mask containing only bit 31.
- A scan-position value of 32.

Each condition evaluation reads the descriptor’s base field and count, calculates the selected address, and reads the 32-bit word there. It tests that word against the current mask.

If the tested bit is set, the scan stops without decrementing the scan position for that iteration.

If the tested bit is clear:

- The scan-position value is decremented and reduced to its low 16 bits.
- The mask is shifted right by one bit.
- The condition is evaluated again using fresh descriptor and selected-word reads.

For stable data, if the selected word’s highest set bit is at zero-based position \(h\), where \(0 \leq h \leq 31\), the scan stops with scan position \(h+1\).

The function does not inspect earlier words to compensate for a zero selected word. It neither reduces the count nor searches backward through the array.

### Repeated reads and ordering

The selected address is not fixed once at scan entry. Both the count and base field are reread on every condition evaluation. Thus, in an environment where such changes are meaningful, later iterations can select a different address.

Established ordering is limited to the following:

1. The dependency call completes before direct descriptor accesses begin.
2. Each condition obtains the descriptor values needed to calculate an address before reading the word at that address.
3. A failed bit test precedes the scan-state update and the next condition.
4. A successful bit test precedes the final count read used for the return calculation.

The supplied expression does not establish a total evaluation order between the base-field read and the count read within one condition. No atomic snapshot of those fields is established.

Ordinary unsynchronized concurrent writes are not granted defined behavior by this contract. Repeated source-level reads do not imply atomicity, synchronization, or guaranteed device-memory semantics.

### Final calculation

After the scan terminates, the function reads the descriptor count again. This read is separate from the count read that selected the successful word.

Let:

- \(s\) be the scan position when the scan stops.
- \(n_f\) be the count obtained for the final calculation.

The returned result is:

\[
\left(s + 32(n_f-1)\right) \bmod 65{,}536
\]

The final result is the low 16 bits, not a saturating value and not an overflow error.

With stable data, positive count \(n\), and a nonzero selected word with highest set bit \(h\), this simplifies to:

\[
\left(32(n-1)+h+1\right) \bmod 65{,}536
\]

Only the selected word is inspected. Lower words do not affect the result.

### Stable zero selected word

For a stable selected word equal to zero, every bit test fails.

After 32 failed tests:

- The mask becomes zero.
- The scan position becomes zero.

Further tests against a zero mask always fail, regardless of the word’s value, provided the reads themselves continue normally. The mask remains zero, and the scan position cycles modulo 65,536.

Therefore, if no successful test occurs before the mask becomes zero, the scan cannot subsequently terminate through its normal exit condition. It continues rereading the descriptor and selected word indefinitely unless an external event interrupts execution or an access fails.

### Zero count

A zero count is not rejected. Address calculation selects the word four bytes before the stored base address, subject to the target’s address arithmetic.

If that address is readable and the selected word is nonzero, the function can terminate. With a stable zero count and terminating scan position \(s\), the result is:

\[
(s-32) \bmod 65{,}536
\]

This behavior is not evidence that zero is a supported descriptor count. It is an unchecked consequence of the arithmetic.

## Side effects

F-5b23ade9b3da performs no explicit writes to the descriptor, word storage, or other externally supplied storage. Its visible assignments update private scan state.

It performs no explicit allocation, release, ownership transfer, or resource acquisition. These observations do not constrain F-8f8b4462085f.

Observable effects can include:

- The dependency call and any effects it causes.
- Descriptor reads and repeated 32-bit selected-word reads.
- Access faults if required memory is inaccessible or incompatible with the target’s access requirements.
- Nontermination if the scan never observes a set bit before its mask becomes zero.

There is no explicit exception handling, error-reporting channel, bounds check, null check, or timeout.

An input pointer that would be unusable for descriptor access is not directly dereferenced on the dependency’s nonzero branch. This does not establish that such an input is safe for the entire call: the dependency’s effects and requirements remain unknown.

## Dependencies

### F-8f8b4462085f

**Established interface at this call site:**

- Called once.
- Receives no explicit arguments.
- Returns a value compared with zero.

**Established influence:**

- Nonzero: F-5b23ade9b3da returns zero.
- Zero: F-5b23ade9b3da begins descriptor-driven scanning.

**Unknowns — documented_only:**

- The dependency’s purpose and complete contract.
- Whether it reads or changes globals.
- Whether it accesses, changes, allocates, or releases memory.
- Whether it can affect the descriptor or selected words through indirect state.
- Whether it always returns normally.

The absence of an explicit pointer argument does not establish independence from the input storage. No preservation or ownership guarantee is assumed.

## Acceptance examples

These are conditional examples derived from static evidence, not runtime test results. Except where specified otherwise, assume a 32-bit address model, ordinary readable memory, stable descriptor and word contents, and a dependency result of zero.

| Condition | Required behavior |
|---|---|
| Dependency returns any nonzero value | Return zero; perform no direct descriptor reads. |
| Count is 1; selected word is hexadecimal `80000000` | Stop on the first test and return 32. |
| Count is 1; selected word is hexadecimal `00000001` | Stop on bit 0 and return 1. |
| Count is 1; selected word is hexadecimal `00000100` | Return 9. |
| Count is 2; word index 1 is hexadecimal `00000001` | Return 33, regardless of word index 0’s contents. |
| Count is 2; word index 1 is hexadecimal `80000000` | Return 64. |
| Count is 2; word index 1 is zero, while word index 0 is nonzero | Do not search word index 0; fail to terminate normally while the selected word remains zero. |
| Count is 2,048; selected word is hexadecimal `80000000` | Return zero because the unreduced result is 65,536. |
| Count is 65,535; selected word is hexadecimal `80000000` | Return 65,504 after 16-bit reduction. |
| Count is zero; the word immediately before the base is readable and equals hexadecimal `80000000` | Return zero. |
| Count is zero; that preceding word equals hexadecimal `00000001` | Return 65,505. |
| The dependency changes the descriptor and then returns zero | Use post-call descriptor reads; do not rely on descriptor values from before the call. |

Where the execution environment permits externally changing storage, a count change between the successful bit test and the final count read can make the returned value combine a scan position from one selection with a count from another. No consistency check is present. This is an observation about the access structure, not a guarantee of defined behavior for unsynchronized mutation.

## Uncertainties

### Inferred intent

A multiword-integer bit-length operation is consistent with the selected-word formula and scan direction. This interpretation requires a positive count and a nonzero highest active word.

The dependency may be a guard for such preconditions, but its body is unavailable. Its call alone does not establish that zero counts or zero selected words are impossible.

### Unknowns — documented_only

The supplied evidence does not establish:

- The complete descriptor type or ownership model.
- Word-storage allocation size or bounds.
- Byte order.
- Required alignment or unaligned-access behavior.
- Whether the stored base is always a valid pointer representation.
- The precise platform rules for address wraparound and signed address-field conversion.
- Whether callers guarantee normalization.
- Whether accessed storage can legally change during execution.
- Whether a dependency-induced release can invalidate later accesses.
- The actual machine-level load count after compilation.

Under the apparent 32-bit arithmetic model, the selected-address expression uses a 32-bit unsigned offset and can wrap at the address-width boundary. That does not make a wrapped address valid.

The final count arithmetic has no large intermediate overflow for a 16-bit count under this model: the count-minus-one range is −1 through 65,534, and multiplying it by 32 fits a signed 32-bit integer. The final low-16-bit reduction is nevertheless intentional in the visible expression and discards higher result bits.

## Evidence

**Established static evidence:** one supplied function body showing the input’s typed field accesses, a single call to F-8f8b4462085f, the dependency-result branch, repeated selected-word tests, a right-shifting 32-bit mask, a 16-bit-reduced scan position, and a final result masked to 16 bits.

**Derived consequences:** the positive-count bit-length formula, repeated address selection, separate final count read, modulo-65,536 results, unchecked zero-count addressing, and failure to terminate after the mask becomes zero follow from those operations.

**Inferred intent:** interpreting the descriptor as a normalized multiword unsigned integer.

No runtime tests have been performed. No dependency implementation, caller contract, allocation evidence, or platform documentation was supplied.
