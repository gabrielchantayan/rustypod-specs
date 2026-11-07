<!-- Trusted provenance: F-fe6a65ef0b1e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe6a65ef0b1e

## Purpose

F-fe6a65ef0b1e performs a bounded, in-place propagation of bits among 15 adjacent words. It makes exactly three forward passes over the 14 neighboring pairs.

For a pair with at least one common set bit and unequal word values, the function propagates selected bits between the two words. Propagation proceeds first toward lower bit positions and then toward higher bit positions. The common bits captured when the pair is first examined seed both directions.

This is not an unconditional union of neighboring words. Propagation can stop at a bit position absent from both words, and pairs with no common set bits are not modified.

**Established behavior:** fixed traversal order, conditional pair processing, bounded directional propagation, and in-place word stores.

**Inferred intent:** the words may represent adjacent rows or columns of a bitmap, with shared bits providing connectivity seeds. No geometric interpretation is required by the contract or established by the supplied material.

## Interface

### Input

The function accepts one address-bearing argument identifying the first word of a contiguous, mutable region.

The required region contains:

- 15 words;
- four bytes per word;
- 60 bytes total;
- word starting offsets from 0 through 56, inclusive.

For this contract, words are treated as unsigned 32-bit bit vectors. Bit 0 is the least significant bit; bit 31 is the most significant bit. Shifts are logical: right shifts introduce zeros, and left shifts discard bits beyond bit 31.

The supplied representation uses an integer-like argument for address construction rather than an explicitly typed buffer parameter. The actual calling convention, pointer representation, and language-level rules for converting that argument to an address are not independently established.

### Output

There is no explicit return value or status indication. Results are delivered through changes to the supplied region.

### Preconditions

Normal completion requires:

- All 60 bytes to be readable.
- Every potentially modified word to be writable.
- Addresses to support the represented four-byte word accesses.
- Address calculations through the final word to identify the intended storage without invalid address arithmetic.
- Stable, ordinary memory if deterministic bit-propagation results are expected.

The function performs no explicit null, length, alignment, accessibility, or overflow validation. It receives no length argument.

### Ownership

The function accesses the supplied region in place. It does not allocate, free, retain, or return storage in the supplied body.

No ownership transfer is expressed. The caller remains responsible for the storage’s lifetime and suitability.

## Behavior

### Traversal

The function performs three passes. Every pass examines adjacent word pairs in increasing address order:

- First, words 0 and 1.
- Then, words 1 and 2.
- Continuing through words 13 and 14.

There are exactly 42 pair visits during normal completion.

Updates are immediate, not deferred until the end of a pass. A word modified as the second member of one pair is subsequently examined as the first member of the next pair using its then-current contents. Later passes likewise observe earlier modifications.

The pass count is fixed. The function does not stop early when a pass makes no changes, and it does not continue until reaching a global fixed point.

### Pair eligibility and initial reads

For each pair, call the lower-addressed word the **first word** and the higher-addressed word the **second word**.

The pair is initially examined in this order:

1. Read the first word.
2. Read the second word and compute their bitwise intersection.
3. If that intersection is nonzero, read the second word again and compare that newly read value with the initially read first word.

Directional processing occurs only if:

- The captured intersection is nonzero; and
- The initially read first word differs from the second word’s comparison read.

If the captured intersection is zero, the comparison read is skipped and the pair receives no stores.

If the compared values are equal, the pair also receives no stores.

The second read of the second word matters for externally changing storage: eligibility is not determined entirely from one snapshot of the pair. For stable memory, the condition is simply “nonempty intersection and unequal words.”

### Directional propagation rule

Each processed pair has two directional phases. Both use the original captured intersection as their seed, but the second phase operates on words already modified by the first phase.

A directional iteration uses a current propagation mask and follows this sequence:

1. Read the current first word.
2. Read the current second word.
3. Shift the propagation mask by one bit in the phase’s direction.
4. Store the first word with any second-word bits selected by the shifted mask added to it.
5. Form the next propagation mask by intersecting the shifted mask with the union of the two values read for this iteration.
6. Store the second word with any first-word bits selected by the shifted mask added to it.

Both stored word values are derived from the same two pre-store reads. In particular, the second-word store does not use the newly stored first-word value.

Equivalently, at positions selected by the shifted mask, each word acquires any bit that was present in the other word at the beginning of that iteration. At all other positions, each word retains its own previously read bits.

The next mask preserves a shifted seed position only where at least one word had that bit before the iteration. Different set bits in the mask can therefore continue or stop independently.

### Lower-bit phase

The first phase propagates toward lower bit positions.

It starts with the original captured intersection and executes at least one iteration for every eligible pair. On each iteration, the mask is logically shifted right by one bit before it is used for transfers.

The phase stops after the iteration if either:

- The next propagation mask is zero; or
- 31 iterations have completed.

A common bit at position 0 shifts out immediately. This still causes one iteration, including both stores, although those stores may preserve the existing values.

### Higher-bit phase

The second phase restarts from the original captured intersection, not from the lower-bit phase’s final mask.

It uses the words as modified by the lower-bit phase. On each iteration, the mask is shifted left by one bit within 32 bits before it is used for transfers.

The phase stops when its next mask becomes zero or after 31 iterations.

Because pair eligibility requires a nonzero original intersection, this phase also performs at least one iteration for every eligible pair. A common bit at position 31 shifts out on the first iteration and can produce two unchanged-value stores.

### Bounds and invariants

For stable ordinary memory:

- Each store preserves all bits that were set in that word’s corresponding iteration read.
- The function never intentionally clears a bit.
- A pair’s union of set bits is unchanged by its directional updates.
- No bit position absent from the entire input region can become present anywhere in the region.
- A pair with no common set bits remains untouched during that visit.
- An equal pair remains untouched during that visit.
- A qualifying pair may receive stores even when no word value changes.

Each processed pair performs at most 31 lower-bit iterations and 31 higher-bit iterations. Each iteration performs two word stores. Across all 42 pair visits, 5,208 word stores is therefore an upper bound from the loop limits, not a claim that every input attains it.

The fixed loop bounds ensure termination of the represented computation, assuming memory operations themselves complete normally.

## Side effects

### Memory reads and writes

All 15 words are read during each complete pass. There are no explicit accesses beyond the 60-byte region.

Pair initialization reads the first word before the second word. A nonzero intersection triggers the additional second-word comparison read.

Within every directional iteration, the access order is:

1. First-word read.
2. Second-word read.
3. First-word store.
4. Second-word store.

Stores write complete four-byte words, including when the stored value equals the value already present.

These are the access sequences represented by the supplied body. They are not a specification of atomicity, synchronization, memory barriers, or externally guaranteed ordering under an optimizing compilation model.

### Partial completion and concurrent changes

The function provides no transaction or rollback. If an access faults after earlier stores have completed, earlier modifications may remain. In particular, failure on a second-word store can leave only the first-word store from that iteration completed.

Concurrent writers or side-effectful memory can invalidate the stable-memory invariants. For example, a store based on an earlier read can overwrite bits added by another actor after that read.

No locking, atomic access protocol, or retry mechanism appears in the supplied body.

### Other effects

No explicit global-state access, external function call, allocation, release, logging, or error reporting appears in the supplied body.

## Dependencies

No opaque function dependencies are present in the supplied body. F-fe6a65ef0b1e makes no explicit calls to another function.

Its environmental dependencies are:

- Unsigned 32-bit bitwise operations and logical shifts.
- Four-byte word loads and stores.
- A valid address interpretation for the argument.
- Accessible storage satisfying the interface preconditions.

Endianness does not change the word-level propagation rules, but it determines how byte sequences correspond to the numeric word values used in those rules.

There is no unknown callee whose memory ownership or global-state behavior must be assumed.

## Acceptance examples

These are conditional acceptance cases derived from static analysis, not executed tests. Values denote unsigned 32-bit words. Unless stated otherwise, all unmentioned words are zero, storage is stable, and all accesses are valid.

### All-zero region

**Input:** all 15 words are zero.

**Required result:** all words remain zero. Each pair fails the nonzero-intersection condition, so the function performs no stores. All three passes still occur.

### Equal nonzero words

**Input:** every word is `0x00000005`.

**Required result:** the region remains unchanged and receives no stores. Every pair has a nonzero intersection but fails the inequality condition.

### Unequal but disjoint neighboring words

**Input:** the first two words are `0x00000001` and `0x00000002`.

**Required result:** both retain their original values. Their intersection is zero, so neither directional phase runs for that pair.

### Propagation toward lower bits

**Input:** the first two words are `0x00000010` and `0x0000001C`.

**Required result:** both become `0x0000001C`.

The common bit at position 4 seeds propagation. The second word supplies set bits at positions 3 and 2 to the first word on successive lower-bit iterations. Absence at position 1 stops that chain.

### A gap stops propagation

**Input:** the first two words are `0x00000010` and `0x00000014`.

**Required result:** the first remains `0x00000010`; the second remains `0x00000014`.

The lower-bit shift from the shared position 4 reaches position 3, which is absent from both words. Propagation stops there and does not jump across the gap to copy position 2. The higher-bit direction also finds no continuation.

Because this pair qualifies, unchanged output does not imply absence of stores.

### Propagation toward higher bits

**Input:** the first two words are `0x00000002` and `0x0000000E`.

**Required result:** both become `0x0000000E`.

The lower-bit phase cannot continue through position 0. The higher-bit phase independently restarts from the original common bit at position 1 and propagates through positions 2 and 3.

### Earlier updates affect a later pair

**Input:** the first three words are `0x00000008`, `0x0000000C`, and `0x00000004`.

**Required result:** all three become `0x0000000C`.

The first pair becomes equal to `0x0000000C`. The next pair then uses the updated middle word and transfers the higher adjacent bit into the third word during the same pass.

### Boundary shifts do not wrap

**Input:** the first two words are `0x80000000` and `0x80000001`.

**Required result:** both remain unchanged.

The lower-bit phase encounters an absent position immediately below the shared high bit. The higher-bit phase shifts the high bit out of the 32-bit word; it does not wrap it to bit 0.

## Uncertainties

- **documented_only — ABI and widths:** The four-byte stride and unsigned-word operations support the 32-bit contract above. The supplied material does not independently establish the machine ABI or pointer width. A wider-word interpretation would not be interchangeable with this contract.
- **documented_only — Byte order:** No endianness is specified. Byte-oriented callers must use the actual environment’s word representation.
- **documented_only — Invalid input:** Null addresses, short buffers, unsupported alignment, inaccessible memory, and invalid address arithmetic have no explicit handling. A particular failure mode is not established.
- **documented_only — Concurrency:** No race-free or atomic behavior is established. Acceptance examples assume no external mutation.
- **documented_only — Application meaning:** Bitmap connectivity is a plausible interpretation, not a proven input-format requirement.
- **documented_only — Global closure:** Three passes are established. No guarantee is established that the result is a global propagation fixed point or that another invocation cannot make further changes.
- **documented_only — Runtime behavior:** No execution, instrumentation, or runtime tests have been performed.

## Evidence

The supplied function body is the sole evidence source.

Static evidence establishes the 14-pair forward traversal, three-pass bound, four-byte spacing, intersection-and-inequality gate, separate directional seeds, 31-iteration directional limits, full-word store order, and absence of explicit external calls.

The behavioral invariants and acceptance examples are deductions from those operations under the stated stable-memory and 32-bit assumptions. They are not runtime observations.

No prior document, comment, presumed application intent, or unknown dependency behavior is used to fill the documented uncertainties.
