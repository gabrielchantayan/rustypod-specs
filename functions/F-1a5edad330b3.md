<!-- Trusted provenance: F-1a5edad330b3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1a5edad330b3

## Purpose

F-1a5edad330b3 transforms a 32-bit unsigned input into a numeric candidate and accepts that candidate only when it is strictly below a supplied unsigned bound. Rejection is represented by the 32-bit value `0xFFFFFFFF`.

For inputs representing ASCII decimal digits and letters, the transformation resembles digit-value conversion: decimal digits produce values 0–9, and letters of either case produce values 10–35. This is inferred intent, not a complete description of the accepted inputs. The function does not restrict its input to those character classes, does not enforce a conventional radix range, and can accept punctuation or much larger unsigned values.

The complete numeric transformation, including non-character inputs, is part of the established behavior.

## Interface

**Inputs**

- **Input value:** one unsigned 32-bit integer, ranging from 0 through `0xFFFFFFFF`.
- **Exclusive bound:** one unsigned 32-bit integer in the same range. Acceptance requires the transformed candidate to be less than this bound.

**Output**

One unsigned 32-bit integer:

- The transformed candidate, if it is strictly less than the exclusive bound.
- `0xFFFFFFFF` otherwise.

The bound is applied to the transformed candidate, not directly to the original input.

**Arithmetic model**

All input interpretation, comparisons, and returned values are unsigned. Subtraction that crosses below zero wraps modulo \(2^{32}\); it does not produce a signed negative value, raise an arithmetic error, or saturate at zero.

Bit positions are numbered from zero at the least-significant bit. Clearing bit 5 means clearing the bit with numeric weight 32 while leaving every other bit unchanged.

**Preconditions and representation**

The supplied body imposes no additional value preconditions. In particular, neither argument is required to fit in a byte, represent text, or describe a conventional positional-number base.

The interface is scalar. It has no buffer layout, terminator, length field, pointer argument, or ownership transfer. Machine-level calling-convention details and external serialization are not established by the supplied evidence.

## Behavior

### Candidate mapping

The following table defines the candidate before bound checking. Every interval is inclusive. Hexadecimal notation denotes numeric values, not encoded text.

| Input interval | Candidate |
|---|---|
| `0x00000000`–`0x0000002F` | Subtract 48 with 32-bit unsigned wraparound, clear bit 5 of that result, then subtract 55. |
| `0x00000030`–`0x00000039` | Input minus 48. |
| `0x0000003A`–`0x00000040` | Original input, unchanged. |
| `0x00000041`–`0x0000005F` | Input minus 55. |
| `0x00000060` | Original input, unchanged: 96. |
| `0x00000061`–`0x0000007F` | Input minus 87. |
| `0x00000080`–`0xFFFFFFFF` | Clear bit 5 of the input, then subtract 55. |

This partition is exhaustive over the unsigned 32-bit input domain.

In the first interval, the initial subtraction wraps to a large unsigned value. Clearing bit 5 does not make that value small. The subsequent subtraction therefore also produces a large unsigned candidate. These inputs are not unconditionally rejected: a sufficiently large bound can accept them.

In the last interval, clearing bit 5 leaves a value of at least 128. Subtracting 55 therefore does not underflow.

### Bound checking

A candidate is accepted exactly when:

\[
\text{candidate} < \text{exclusive bound}.
\]

Equality rejects. There is no special allowance for a zero candidate or for inputs conventionally recognized as digits.

Consequences include:

- Bound zero rejects every input.
- Bound one accepts a candidate of zero but rejects a candidate of one.
- A candidate of 35 requires a bound of at least 36.
- Bounds greater than 36 are permitted and can accept candidates outside the conventional alphanumeric digit range.
- A bound of `0xFFFFFFFF` accepts every candidate produced by the mapping.

The last consequence follows from the candidate range, not from a special-case bypass. The largest possible candidate is `0xFFFFFFA8`, which is below `0xFFFFFFFF`.

### Character-like inputs

Under ASCII interpretation:

- `0` through `9` map to 0 through 9.
- `A` through `Z` map to 10 through 35.
- `a` through `z` map to 10 through 35.

These mappings establish case equivalence for those letters. They do not establish a general text case-conversion operation, Unicode handling, locale sensitivity, or validation that the input represents a character.

Several adjacent punctuation inputs also produce useful-sized candidates:

- `[` through `_` map to 36 through 40.
- `{` through the numeric input `0x7F` map to 36 through 40.
- `:` through `@` remain numerically unchanged, yielding candidates 58 through 64.
- The grave-accent input remains 96.

Consequently, a replacement that first rejects every non-alphanumeric input would not preserve the observed behavior.

### Failure representation

Bound failure has one representation: `0xFFFFFFFF`. The body exposes no separate status output, diagnostic, exception, retry, or error category.

The candidate mapping itself never produces `0xFFFFFFFF`. Within this function’s contract, the sentinel therefore unambiguously represents rejection rather than an accepted candidate. How a caller interprets or propagates that sentinel is not established.

## Side effects

The supplied body operates only on its scalar arguments and returns a scalar result.

There are no explicit:

- Memory dereferences or externally visible memory writes.
- Global-state reads or writes.
- Calls to other functions.
- Allocations, releases, or ownership changes.
- I/O operations or persistent-state updates.

Intermediate arithmetic changes only the function’s working value. It does not modify the caller’s argument storage through any interface shown here.

There is no observable sequence of external reads or writes to preserve. The relevant ordering constraint is numeric: the candidate must be determined before the exclusive-bound decision. In particular, wrapped arithmetic for low-valued inputs cannot be replaced by an earlier rejection if that would prevent acceptance under a sufficiently large bound.

These side-effect statements describe the supplied body. They do not assert anything about instrumentation, wrappers, or a surrounding execution environment that was not provided.

## Dependencies

No dependency calls appear in the supplied body. No additional opaque function IDs are needed to describe its behavior.

The result depends only on the two argument values and the stated unsigned 32-bit arithmetic model. There is no evidence of a lookup table, external character-classification service, allocator, or mutable configuration.

Accordingly, no assumption about an unknown callee preserving globals or managing memory is necessary.

## Acceptance examples

The following are contract-derived examples, not runtime observations.

| Input | Exclusive bound | Required result | Reason |
|---|---:|---:|---|
| `0x30` (`0`) | 1 | 0 | Candidate zero is below one. |
| `0x30` (`0`) | 0 | `0xFFFFFFFF` | Zero is not below zero. |
| `0x39` (`9`) | 10 | 9 | Highest decimal digit fits. |
| `0x39` (`9`) | 9 | `0xFFFFFFFF` | Equality rejects. |
| `0x41` (`A`) | 11 | 10 | Uppercase letter maps to ten. |
| `0x61` (`a`) | 11 | 10 | Lowercase letter has the same mapping. |
| `0x5A` (`Z`) | 36 | 35 | Candidate fits a bound of 36. |
| `0x7A` (`z`) | 35 | `0xFFFFFFFF` | Candidate equals the bound. |
| `0x5B` (`[`) | 37 | 36 | Punctuation is not independently rejected. |
| `0x7B` (`{`) | 37 | 36 | The corresponding bit-5 variant also maps to 36. |
| `0x3A` (`:`) | 59 | 58 | This input remains unchanged. |
| `0x40` (`@`) | 65 | 64 | Candidate 64 is accepted only above 64. |
| `0x60` | 97 | 96 | This input also remains unchanged. |
| `0x80` | 74 | 73 | Inputs are not truncated to an ASCII domain. |
| `0xA0` | 74 | 73 | Clearing bit 5 gives the same candidate as `0x80`. |
| `0x00` | `0xFFFFFF9A` | `0xFFFFFF99` | Wrapped subtraction can yield an accepted large candidate. |
| `0x00` | `0xFFFFFF99` | `0xFFFFFFFF` | The same candidate rejects at equality. |
| `0xFFFFFFFF` | `0xFFFFFFFF` | `0xFFFFFFA8` | The maximum input does not automatically reject. |

A conforming implementation must preserve these conditional results even if its intended callers normally supply only alphanumeric inputs and small bounds.

## Uncertainties

**Inferred intent:** The decimal and alphabetic mappings suggest conversion of a character-like value into a digit under a caller-supplied radix. The evidence does not establish that interpretation as an input restriction.

**Documented only:** The permitted input domain at actual call sites, expected bound range, caller error handling, and any surrounding parsing protocol are unknown.

**Documented only:** The supplied evidence does not establish whether unusual punctuation and large-value acceptance were deliberate compatibility behavior or unintended consequences. Their origin does not change the mapping established here.

**Documented only:** Timing, machine calling convention, concurrency guarantees outside the scalar body, and execution-environment behavior are unspecified.

No runtime tests have been performed. The examples and numeric boundaries above are static deductions.

## Evidence

Established static evidence consists of the supplied function body: two unsigned scalar inputs, conditional arithmetic and bit masking, an unsigned exclusive-bound comparison, and a scalar return.

The complete-domain mapping follows from partitioning the input ranges at the arithmetic and masking boundaries. The examples follow by evaluating that mapping and applying the strict bound comparison.

The supplied body contains no dependency calls or explicit external-state access. No caller implementation, external documentation, or runtime trace was supplied to establish broader application behavior.
