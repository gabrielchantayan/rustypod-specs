<!-- Trusted provenance: F-6d33ecb17b5a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6d33ecb17b5a

## Purpose

F-6d33ecb17b5a compares two explicitly counted sequences of unsigned 16-bit elements and returns a three-way ordering result. Comparison proceeds from the beginning of each sequence. The first unequal pair determines the result; if all inspected pairs are equal, exhaustion of either count determines the result.

A flag optionally makes the 26 ASCII lowercase letter values compare as their uppercase equivalents. This transformation affects comparison values only; it does not modify either sequence.

**Established static behavior:** this is an elementwise comparison with count-based termination and an optional, narrowly defined case transformation.

**Inferred intent:** the function is suitable for comparing counted sequences containing 16-bit character values. The available evidence does not establish a particular text encoding, Unicode collation, or locale-sensitive behavior.

## Interface

The function accepts five inputs, in this order:

| Position | Input | Established representation | Meaning |
|---|---|---|---|
| 1 | First sequence address | Pointer to unsigned 16-bit elements | Beginning of the first sequence |
| 2 | First count | Signed 32-bit integer | Count tested for exhaustion before reading the first sequence |
| 3 | Second sequence address | Pointer to unsigned 16-bit elements | Beginning of the second sequence |
| 4 | Second count | Signed 32-bit integer | Count tested for exhaustion before reading the second sequence |
| 5 | Flags | Unsigned 32-bit integer | Bit 0 enables ASCII lowercase-to-uppercase comparison |

Each sequence consists of successive 16-bit elements, with no per-element metadata. Advancing an element pointer advances by one 16-bit element, or two bytes in an eight-bit-byte address model. Byte order and alignment requirements are not established by the supplied evidence.

The return value is a 32-bit quantity with exactly three normal results:

| Return bits | Signed interpretation | Meaning |
|---|---|---|
| `0xFFFFFFFF` | −1 | First sequence compares less than the second |
| `0x00000000` | 0 | Sequences compare equal |
| `0x00000001` | +1 | First sequence compares greater than the second |

The negative result is the all-ones 32-bit pattern. Consumers interpreting the result as unsigned must not treat it as a negative integer without conversion.

### Preconditions and supported domain

For ordinary counted-sequence use:

- Both counts are nonnegative and represent numbers of elements, not bytes.
- Every element actually inspected is readable through its corresponding address.
- Pointer advancement needed to reach inspected elements is valid under the execution environment.
- Sequence storage remains readable during the comparison.

The function contains no explicit validation of these conditions.

A zero count prevents dereferencing its sequence address. More strongly, if either count is zero on entry, neither sequence is read: the count checks alone determine the result. Consequently, a null address paired with a zero count does not itself cause an explicit dereference in this function.

Negative counts are not rejected. Their consequences are described under **Uncertainties**; callers should not interpret them as empty sequences.

## Behavior

### Count checks precede element reads

At the beginning of every comparison round, the first count is examined before the second count determines whether element comparison can proceed.

The outcomes are:

1. If the first count is zero and the second count is nonzero, return `0xFFFFFFFF`.
2. If both counts are zero, return zero.
3. If the first count is nonzero and the second count is zero, return one.
4. Only when both counts are nonzero are sequence elements read.

These are exact zero checks. A negative count is nonzero for these decisions.

For nonnegative counts, these rules give the usual prefix ordering: after all shared leading elements have compared equal, the shorter sequence compares less than the longer sequence.

### Element acquisition and comparison

When both counts are nonzero, the function obtains the current unsigned 16-bit element from the first sequence, followed by the current unsigned 16-bit element from the second sequence in the supplied operation ordering.

Each element is widened without sign extension to a 32-bit unsigned comparison value. Therefore, every initially loaded value lies between 0 and 65,535 inclusive.

The available evidence establishes this ordering at the supplied representation level. It does not establish externally observable processor transaction ordering, volatile-memory semantics, or behavior under concurrent modification.

### Optional ASCII transformation

If flags bit 0 is clear, both loaded values are compared unchanged.

If flags bit 0 is set, each value in the inclusive range `0x0061` through `0x007A` is replaced, for comparison only, by the value 32 less. Thus:

- `0x0061` compares as `0x0041`.
- `0x007A` compares as `0x005A`.
- Values outside that range remain unchanged.

The range decision uses unsigned 32-bit arithmetic. Subtracting `0x61` from a smaller value wraps in that unsigned width, producing a value outside the accepted range. It does not cause values below `0x0061` to be folded accidentally.

Flags bits 1 through 31 have no effect on the supplied behavior. Setting those bits neither enables additional transformations nor changes the result when bit 0 is unchanged.

This is not general Unicode case folding. It does not combine elements, decode surrogate pairs, normalize text, or transform non-ASCII lowercase letters. An element with value zero is ordinary data, not a terminator.

### Ordering decision

After the optional transformation:

- If the first value is greater than the second, return one.
- If the first value is less than the second, return `0xFFFFFFFF`.
- If the values are equal, continue with the next pair.

Although the ordering comparisons interpret the widened values as signed 32-bit integers, all possible comparison values remain between 0 and 65,535. Signed and unsigned ordering therefore agree for these values.

On an unequal pair, the function returns immediately. It does not inspect later elements, and it does not advance the sequence positions or decrement the counts for that mismatching pair before returning.

### Progress after equality

After an equal pair:

- Each count is decreased by one.
- Each sequence position advances by one 16-bit element.
- The function returns to the count checks before attempting another read.

These changes affect the function’s working state only. The caller’s pointer variables and count variables are not updated through output parameters.

For nonnegative counts, at most the smaller initial count’s worth of element pairs is read. If all of those pairs compare equal, the next count checks return either equality or the prefix-ordering result without another element read.

## Side effects

The supplied body performs sequence reads but contains no sequence writes, explicit global accesses, allocation, deallocation, or dependency calls.

In particular:

- ASCII folding does not modify the input elements.
- The return value is the only explicit output.
- No caller-visible cursor or remaining-count state is written.
- No ownership transfer or release operation is present.
- Overlapping sequences, including identical addresses, are not explicitly rejected. Read-only overlap does not itself introduce a write conflict.

The function does not validate addresses or provide a recoverable error result for unreadable storage. An invalid access may fault or otherwise follow the execution environment’s invalid-memory behavior; the available evidence does not define that outcome.

The static read ordering is first element, then second element, followed by comparison. This is not a promise that the function is appropriate for memory-mapped devices, volatile storage, or concurrently changing buffers.

## Dependencies

No called dependencies appear in the supplied body. There are therefore no additional opaque function IDs to enumerate.

Execution nevertheless depends on the surrounding environment for readable memory, valid pointer operations, and the interpretation of invalid accesses. Those environmental requirements are not separate function calls, and their failure behavior is not specified here.

No assumptions about an unknown callee preserving global state or managing ownership are needed: no such callee is shown.

## Acceptance examples

These examples are conditional static acceptance criteria, not runtime observations. Element lists denote numeric 16-bit values.

### Empty inputs

- First count 0, second count 0: return `0x00000000`, without reading either address.
- First count 0, second count 3: return `0xFFFFFFFF`, without reading either address.
- First count 2, second count 0: return `0x00000001`, without reading either address.

The last two cases do not require readable sequence storage because the comparison never reaches an element read.

### Exact comparison and prefixes

With bit 0 clear:

- `[0x0041]` versus `[0x0042]`, both counts 1: return `0xFFFFFFFF`.
- `[0x0042]` versus `[0x0041]`, both counts 1: return one.
- `[0x0041, 0x0042]` versus the same two values, both counts 2: return zero.
- `[0x0041]`, count 1, versus `[0x0041, 0x0042]`, count 2: return `0xFFFFFFFF` after reading only the first pair.
- `[0x0042]`, count 1, versus `[0x0041, 0xFFFF]`, count 2: return one. The first mismatch takes precedence over sequence length, and the second element is not read.

### Case transformation and its boundaries

- `[0x0061]` versus `[0x0041]`, both counts 1, flags zero: return one.
- The same inputs with flags bit 0 set: return zero.
- `[0x007A]` versus `[0x005A]`, both counts 1, bit 0 set: return zero.
- `[0x0060]` versus `[0x0040]`, both counts 1, bit 0 set: return one; `0x0060` is below the folding range.
- `[0x007B]` versus `[0x005B]`, both counts 1, bit 0 set: return one; `0x007B` is above the folding range.
- `[0x00E1]` versus `[0x00C1]`, both counts 1, bit 0 set: return one; non-ASCII case relationships do not affect comparison.

For fixed inputs, flags zero and `0xFFFFFFFE` must produce the same result. Flags one and `0xFFFFFFFF` must likewise produce the same result.

### Zero and high-valued elements

- `[0x0000, 0x0042]` versus `[0x0000, 0x0041]`, both counts 2: return one. The zero-valued first pair does not terminate comparison.
- `[0xFFFF]` versus `[0x0001]`, both counts 1: return one. The high-valued element is not sign-extended from 16 bits.

### Early return and read extent

If both counts are positive and the first pair differs, only that pair is required for the comparison. Later elements do not affect the result and are not explicitly read.

If all compared elements are equal and the counts are equal, exactly that many pairs are read before the function returns zero.

## Uncertainties

The following are **documented_only**, not inferred guarantees:

- **Negative counts:** they pass the nonzero checks and can cause element reads. A zero first count with a negative second count returns `0xFFFFFFFF`; a negative first count with a zero second count returns one. When both counts remain nonzero, equal pairs cause further decrements and pointer advances. The supplied representation does not establish a portable contract for eventual signed-count overflow or invalid pointer progression.
- **ABI details:** calling convention, pointer width, register assignment, and return-register extension are not established.
- **Storage representation:** byte order and required alignment for 16-bit reads are not established.
- **Invalid memory:** no defined error return, exception translation, or recovery mechanism is shown.
- **Concurrent mutation:** no synchronization or stable-snapshot guarantee is shown.
- **Text semantics:** character encoding, locale, normalization, and broader case equivalence are not established.

For the supported nonnegative-count domain, count decrement does not overflow: an inspected count is positive before it is reduced by one.

## Evidence

This contract is based solely on static inspection of the supplied function representation. No runtime tests have been performed.

Direct evidence establishes the count-check precedence, unsigned 16-bit reads, 32-bit widening, bit-0-controlled ASCII transformation, three return patterns, early return on mismatch, and advancement only after equality. The supplied body also directly supports the absence of explicit writes and dependency calls.

The characterization as a counted-sequence comparator is supported by those operations. Character-string intent remains an inference; encoding and environmental guarantees remain unknown.
