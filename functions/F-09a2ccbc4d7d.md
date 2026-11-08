<!-- Trusted provenance: F-09a2ccbc4d7d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-09a2ccbc4d7d

## Purpose

F-09a2ccbc4d7d searches a counted byte region for a counted pattern and returns the **one-based starting position** of the first successful candidate. Zero means that no candidate succeeded, including when the supplied pattern count is zero.

Comparison is either byte-exact or ASCII-case-insensitive, selected by the lowest option bit. Zero bytes are ordinary data.

**Established behavior:** the outer search uses the full search count, but each candidate’s comparison extent depends on eight-bit reductions of the counts. Consequently, this function does not implement conventional arbitrary-length substring matching. A pattern count greater than 256 can describe a short effective comparison, and increasing the search count can prevent a previously successful candidate from matching.

**Inferred intent:** a bounded byte-pattern search with optional ASCII case folding. This inferred purpose does not override the count-dependent behavior specified below.

## Interface

### Inputs and result

| Input | Meaning | Representation |
|---|---|---|
| Search address | Beginning of the region containing candidate starts | Byte-addressed pointer |
| Search count | Number of candidate starts available to the outer search | Unsigned 32-bit count |
| Pattern address | Beginning of the comparison pattern | Byte-addressed pointer |
| Pattern count | Enables searching and determines the effective comparison length | Unsigned 32-bit count |
| Options | Selects comparison mode | Unsigned 32-bit value; only bit 0 matters |

The result is a signed integer. Under the supplied 32-bit integer interpretation, it is a signed 32-bit result. Zero denotes failure. A successful result is the candidate’s one-based starting position, subject to the address-conversion restrictions below.

The count calculations in this contract use eight-bit bytes and unsigned 32-bit quantities. Exact target ABI guarantees, particularly pointer-to-signed-integer conversion, remain **documented_only**.

### Storage layout

Both inputs refer to flat byte sequences. There are no headers, terminators, alignment-sensitive fields, or implicit length values.

The regions may overlap. The function neither writes them nor requires them to be disjoint.

The supplied counts do not prove that the corresponding storage exists. Conversely, a large pattern count does not necessarily cause that many pattern bytes to be read: the effective comparison length is at most 256.

### Preconditions

For execution without invalid memory access or invalid address arithmetic:

- Every byte actually read must be accessible.
- Every address formed on the executed path must be valid under the surrounding execution model, including addresses formed without subsequently being dereferenced.
- Successful-result address conversions and signed subtraction must be defined if a portable numerical result is required.

No pointer validation, storage-size validation, or recovery from invalid memory is provided.

Stable bytes are required for the simplified matching conditions and acceptance examples below. The read schedule is specified separately because the function does not create a snapshot of either region.

## Behavior

### Comparison mode

Only option bit 0 affects behavior. All other option bits are ignored.

With bit 0 clear, bytes compare exactly.

With bit 0 set, ASCII lowercase bytes `0x61` through `0x7A` compare as their uppercase counterparts `0x41` through `0x5A`. All other byte values remain unchanged. In particular:

- Zero bytes retain their ordinary value.
- Bytes at or above `0x80` are not case-folded.
- No locale, Unicode, multibyte encoding, or character-boundary processing applies.

The lowercase range check uses unsigned arithmetic. Values below `0x61` are not included by a negative intermediate result.

### Empty inputs and the retained first pattern byte

A zero pattern count returns zero without reading either region.

For every nonzero pattern count, the first pattern byte is read and its comparison value is retained before the search count determines whether any candidate is available.

Therefore, a zero search count with a nonzero pattern count:

- Requires the first pattern byte to be readable.
- Reads no search byte.
- Returns zero.

The first pattern comparison value remains the same throughout the call. Later candidates do not reread the first pattern byte. Subsequent pattern bytes, when needed, are read separately for each candidate extension.

### Candidate selection

Candidate starts are considered in increasing byte order, beginning at the search address. The full search count bounds the number of starts considered; it is not reduced to eight bits.

A candidate whose first search byte differs from the retained first pattern comparison value cannot succeed. A candidate with an equal first byte is eligible for extension under the length and capacity rules below.

An unsuccessful candidate never skips subsequent starts. The next candidate begins exactly one byte later, even if the unsuccessful comparison inspected several following bytes. Consequently, the same search byte can be read during one candidate’s extension and again during a later candidate.

The first successful candidate terminates the search. Exhausting all candidate starts returns zero.

### Effective pattern length

For any nonzero pattern count, the effective pattern length is:

**1 + ((pattern count − 1) modulo 256).**

This is the number of equal byte pairs required for success. It is always between 1 and 256.

| Supplied pattern count | Effective pattern length |
|---:|---:|
| 1 | 1 |
| 2 | 2 |
| 255 | 255 |
| 256 | 256 |
| 257 | 1 |
| 258 | 2 |
| 511 | 255 |
| 512 | 256 |

A nonzero multiple of 256 requires 256 equal pairs; it does not behave as an empty pattern.

The original pattern count determines this same effective length independently for every candidate. Bytes beyond that length do not participate in comparison.

### Candidate capacity

A candidate’s comparison capacity depends on how many outer-search positions remain after its first byte.

**Candidate capacity = 1 + (remaining search count modulo 256).**

Equivalently, for a candidate at one-based position \(s\) within an initial search count \(N\):

\[
\text{capacity}=1+((N-s)\bmod 256),\qquad 1\leq s\leq N.
\]

Capacity is always between 1 and 256. It is not the full number of bytes remaining in the declared search region.

For stable bytes and valid execution, a candidate succeeds exactly when:

- Its effective pattern length does not exceed its capacity; and
- All byte pairs through the effective pattern length compare equal in the selected mode.

These conditions are evaluated in candidate order, so the return identifies the earliest candidate satisfying both.

A candidate can fail despite ample accessible storage and hundreds of positions remaining. For example, when exactly 256 positions remain after the first byte, capacity is only one. Conversely, an effective pattern length of one succeeds on first-byte equality even when capacity is one.

Increasing the search count is not monotonic with respect to success: it changes the reduced capacity at each existing candidate.

### Read extent and ordering

For a nonempty pattern, the retained initial pattern byte is read before any search byte.

At each candidate, the first search byte is read before any extension bytes for that candidate. An initial mismatch causes no additional pattern-byte read.

During extension, each subsequent search byte is read before its corresponding pattern byte. Comparison stops at the first unequal pair, upon completion of the effective pattern length, or when the candidate’s reduced capacity prevents another pair.

Thus:

- No later pair is read after a mismatch.
- No pair beyond the effective pattern length is read.
- No pair beyond the candidate capacity is read.
- An effective pattern length of one requires no second byte from either region.
- Insufficient capacity does not cause a speculative read of the next pair.

Address formation has additional preconditions beyond these read bounds. The address following a candidate’s first search byte is formed before that byte is read. Similarly, an extension search-byte access is preceded by formation of its following address.

For a candidate whose first byte matches, the pattern address advances after every compared pair, including the final successful pair or a mismatching extension pair, before the stopping decision. The resulting address is not necessarily dereferenced. Such address formation must still be valid under the applicable pointer rules.

### Result calculation

Success uses the address immediately following the candidate’s **first** search byte, not the address following the complete match.

The function converts that address and the original search address to signed integers and subtracts them. When those operations are defined and represent the expected address difference:

- The first candidate returns 1.
- The second candidate returns 2.
- A candidate at zero-based offset \(k\) returns \(k+1\).

This is neither a pointer result nor a zero-based index.

A small relative offset alone does not establish that the computation is portable: the two absolute address conversions and their signed subtraction also matter. No wraparound rule, saturation behavior, or alternative large-offset encoding is established.

## Side effects

The function reads input bytes and returns a signed integer. Its bookkeeping is local.

There are no explicit:

- Writes to either supplied region.
- Global-state accesses.
- Function calls.
- Allocations or releases.
- Ownership transfers.
- Error callbacks, status objects, or diagnostic outputs.

The caller retains responsibility for the lifetime and accessibility of both regions.

Read order can matter if the memory changes during the call. Only the first pattern byte’s comparison value is retained across candidates; other relevant bytes are read when their comparisons occur. The function supplies no synchronization or coherent-snapshot guarantee.

Zero is not an invalid-input status. It conflates an empty pattern, an empty search, ordinary comparison failure, and rejection caused by reduced candidate capacity.

Invalid storage or invalid address arithmetic is not converted into a defined error result. The surrounding execution environment determines whether such execution faults or otherwise lacks defined behavior.

## Dependencies

No function dependencies appear in the supplied body. There are therefore no dependency IDs to enumerate and no unknown callee effects to assume.

The function performs byte normalization, scanning, count reduction, and result calculation directly. Its behavior still depends on the environment’s:

- Byte and integer representations.
- Pointer validity rules.
- Pointer-to-signed-integer conversions.
- Signed subtraction semantics.
- Handling of invalid memory accesses.

No external case-conversion service, allocator, locale state, or ownership protocol is involved.

## Acceptance examples

These are **conditional static expectations**, not runtime test results. Unless stated otherwise, bytes remain stable, all executed accesses and address formations are valid, and successful-result conversions and subtraction yield the ordinary representable offset.

1. **Empty pattern does not match.**  
   A pattern count of zero returns zero for any search count and options. Neither region is read.

2. **Empty search still reads a nonempty pattern’s first byte.**  
   Search count zero and pattern count three return zero after reading the first pattern byte. No search byte is read. An inaccessible first pattern byte is not made safe by the empty search.

3. **Result is one-based.**  
   Searching the four bytes `xxAB` for the two-byte pattern `AB`, using exact comparison, returns 3.

4. **Failed extensions do not skip overlapping candidates.**  
   Searching `aaab` for `aab`, with counts four and three, returns 2. The first candidate fails at its third pair; the candidate one byte later remains eligible.

5. **Only option bit 0 matters.**  
   Searching `aB` for `Ab`, with both counts two, returns zero with options 0 or 2, and returns 1 with options 1 or 3.

6. **Embedded zero is ordinary data.**  
   Searching bytes `0x11, 0x00, 0x22` for bytes `0x00, 0x22`, with counts three and two, returns 2.

7. **Non-ASCII bytes are not folded.**  
   A one-byte search containing `0xE1` does not match a one-byte pattern containing `0xC1`, even with option bit 0 set.

8. **A large pattern count can require only one byte.**  
   With pattern count 257 and a positive search count, equality of the first search and first pattern bytes succeeds at position 1. No second pattern byte is read.

9. **A nonzero multiple of 256 is not empty.**  
   With both counts 256 and identical 256-byte regions, the first candidate has capacity 256 and succeeds at position 1.

10. **Reduced capacity can reject an available match.**  
    Let the search contain 257 bytes: `A`, then `B`, then 255 copies of `x`. Searching for `AB` with pattern count two returns zero. At the first candidate, 256 positions remain, giving capacity one. The second pair is not read for that candidate, and no later candidate begins with `A`.

11. **Increasing the search count can restore capacity.**  
    Let the search contain 258 bytes: `A`, then `B`, then 256 copies of `x`. Searching for `AB` with pattern count two returns 1. At the first candidate, 257 positions remain, giving capacity two.

12. **The outer scan extends beyond 256 positions.**  
    Searching 256 copies of `x` followed by `Q`, with search count 257 and the one-byte pattern `Q`, returns 257.

13. **Insufficient capacity is not a partial success.**  
    Searching the single byte `A` for `AB`, with counts one and two, returns zero. No second search or pattern byte is read, although the pattern address advances after the equal first pair.

14. **A mismatch at the required final pair still fails.**  
    Searching `AC` for `AB`, with both counts two and exact comparison, returns zero. Reaching the effective pattern length does not override the unequal final pair.

## Uncertainties

The following remain **documented_only**:

- Exact target ABI guarantees beyond the byte and integer model used above.
- Pointer-to-signed-integer conversion behavior for the addresses supplied by actual callers.
- Successful-result behavior when either address conversion or signed subtraction lacks the required representability.
- Whether callers impose narrower count or address restrictions.
- Whether the eight-bit reductions are intentional compatibility behavior or a defect.
- Outcomes under concurrent mutation, memory-mapped input, or other read-sensitive storage. The access ordering is established; a stable logical match result in those environments is not.
- The platform’s response to invalid accesses or invalid pointer arithmetic.

No claim is made that malformed inputs return zero, that arbitrary counts are safe, or that the routine implements conventional substring semantics.

## Evidence

The supplied function body provides static evidence for:

- Selection of comparison mode by the lowest option bit.
- Immediate failure for a zero pattern count.
- The initial pattern-byte read before testing search availability.
- Retention of that first comparison value throughout the call.
- Full-width traversal of candidate starts.
- Eight-bit reduction of pattern progress and candidate continuation capacity.
- ASCII-only lowercase normalization.
- Search-before-pattern read order for extension pairs.
- One-byte advancement between unsuccessful candidate starts.
- Success calculation from the address following the candidate’s first byte.
- Absence of explicit storage writes, calls, allocation, release, and ownership operations.

The effective-length formula, capacity formula, stable-input success criterion, and acceptance expectations are deductions from those static facts. The description of intended use is inference, not an additional guarantee.

No runtime tests have been performed.
