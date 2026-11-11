<!-- Trusted provenance: F-74c302cffcd2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-74c302cffcd2

## Purpose

F-74c302cffcd2 transforms a word-array integer using a modulus descriptor, an arithmetic coefficient, and two context-provided workspaces. It normalizes the primary workspace’s length, invokes a comparison dependency, conditionally invokes an adjustment dependency, and forwards the primary descriptor to an output dependency.

**Established behavior:** the transformation uses 32-bit words, unsigned 64-bit products, explicit carry propagation, and encoded loads and stores. Its observable contract includes the order of external-value reads and the retention of selected values across subsequent arithmetic and writes.

**Inferred intent:** the squared input terms, doubled input history, and modulus products suggest a specialized modular squaring or Montgomery-style reduction. The available evidence does not establish a mathematical modular-arithmetic postcondition.

**Unknown:** representation constants, required number-theoretic invariants, dependency implementations, and ownership effects. These remain documented_only.

## Interface

### Arguments and return value

The four arguments are:

1. **Output argument:** an opaque 32-bit value, forwarded unchanged to F-c64ad9e9ad7c. This routine does not directly dereference it.
2. **Input descriptor:** an address identifying the input integer.
3. **Context:** a 32-bit address, or zero.
4. **Modulus descriptor:** an address identifying the modulus-like integer.

The return value is 32 bits:

- A zero context returns an externally supplied 32-bit value.
- On a completed nonnull-context path, successful dependency handling returns zero.
- Any nonzero result from the conditional adjustment or output dependency causes this routine to return one.
- The comparison dependency’s result selects a branch; it is not propagated as an error code.

No broader error taxonomy is established.

### Memory layouts

Integer descriptors have these directly accessed fields:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Encoded length |
| 4 | 32 bits | Address of a contiguous array of 32-bit words |

The context contains:

| Byte offset | Width | Use |
|---|---:|---|
| 0x30 | 32 bits | Primary workspace descriptor address |
| 0x34 | 32 bits | Secondary workspace descriptor address |
| 0x44 | 32 bits | Reduction coefficient |

No capacity field is consulted.

The specification names external 32-bit values by role:

- **Length multiplier:** used in deriving counts from descriptor lengths.
- **Word decoder:** multiplied by stored words to obtain arithmetic words.
- **Word encoder:** multiplied by arithmetic words before storage.
- **Length encoder:** multiplied by the normalized primary word count.
- **Null-context result:** returned on the early path.

These names do not establish their numerical values or imply that encoding and decoding are inverses.

### Arithmetic and storage preconditions

Arithmetic words are 32-bit bit patterns. Word products, sums, shifts, and encoding operations retain their low 32 bits unless explicitly widened. Product-and-carry calculations described as 64-bit use unsigned operands. A carry from a 32-bit addition is zero or one.

Count derivation uses a 32-bit signed product followed by conditional negation if negative. Ordinary valid-count behavior requires representable products and negations. The most-negative signed 32-bit value cannot be converted to a positive count by ordinary signed negation.

Let **N** denote the modulus-derived count and **M** the input-derived count. Valid normal operation requires:

- Readable input and modulus descriptor headers.
- A readable first input word even when M or N is zero.
- Readable input words for the positions actually consumed.
- Readable modulus words zero through N − 1 when N is positive.
- Primary storage for at least N + 4 words.
- Secondary storage for at least N + 1 words, including one word when N is zero.
- Valid workspace descriptor addresses and context fields.
- Representable loop bounds, indices, and address calculations.

In particular, initialization requires N + 4 to be a valid signed loop bound. No checks enforce these requirements. Independent-integer behavior also requires storage arrangements that do not let workspace writes corrupt future operands or descriptor fields.

## Behavior

### 1. Capture length values and test the context

Two distinct reads of the external length multiplier occur before the context test:

1. The first value is retained for the later input-count calculation.
2. A separate read supplies the multiplier for the modulus-length product.

The modulus product is interpreted as signed and conditionally negated to obtain N. The retained first multiplier is not substituted for the second read.

Only after this count derivation does the routine test the context.

If the context is zero, it reads and returns the null-context result. It does not read the input descriptor, obtain workspaces, or invoke dependencies. A readable modulus length and valid count arithmetic are therefore required even on the null-context path.

The relative evaluation order of independent operands within the modulus-product expression is not separately established.

### 2. Obtain and initialize the workspaces

For a nonzero context, the routine obtains the primary descriptor address from offset 0x30 and then the secondary descriptor address from offset 0x34. These descriptor addresses are retained.

It writes literal zero, without applying the word encoder, to primary words zero through N + 3, in increasing index order. The primary data-array address is obtained through the descriptor for the individual accesses; it is not established as a single retained data pointer.

Next, it multiplies the input descriptor’s encoded length by the **first retained length-multiplier value**, conditionally negating a negative product to obtain M. This calculation occurs after primary initialization.

It then reads input word zero, multiplies it by a word-decoder read, and retains the resulting 32-bit value for reduction-factor selection throughout the outer loop. This read occurs even if M or N is zero.

Finally, it writes literal zero to secondary word zero. Neither workspace length is changed at this stage.

### 3. Process the word positions

The outer loop visits positions zero through N − 1 in increasing order.

At each position:

- If the position is below M, the routine reads that input word and decodes it using a word-decoder read.
- Otherwise, the current arithmetic input word is zero, without reading that input position.
- A retained iteration factor is one for the first iteration and two for subsequent iterations.

The reduction factor is the low 32 bits of:

- the context coefficient, read during this iteration; multiplied by
- the sum of decoded primary word zero and the product of the current input word, the retained initial input word, and the iteration factor.

The primary-zero decoding uses a word-decoder read. The computed reduction factor is retained for the remainder of the iteration.

#### Earlier positions

For each position below the current outer position, in increasing order, two carry chains are advanced. Both incoming carries start at zero for each outer iteration.

First, a 64-bit total combines:

- current input word multiplied by the decoded secondary word;
- decoded primary word at that position;
- incoming first carry.

Its high half becomes the next first carry.

Second, another 64-bit total combines:

- reduction factor multiplied by the decoded modulus word;
- the first total’s low half;
- incoming second carry.

Its high half becomes the next second carry.

At positions greater than zero, the second total’s low half is encoded using a word-encoder read and stored to the preceding primary position. At position zero, that low half is discarded.

Each described decoding uses the external word decoder at that access; there is no iteration-wide decoder snapshot.

#### Current position

Three calculations follow at the current outer position:

1. A 64-bit total combines the current input word’s square, decoded primary word, and first carry.
2. A 64-bit total combines the reduction factor times the decoded modulus word, the preceding low half, and second carry.
3. A 64-bit total combines the current input word times the decoded secondary word and the preceding low half.

If the current position is positive, the third total’s low half is encoded and stored to the preceding primary position. At position zero it is discarded.

The high halves from the first and third totals are added as 32-bit quantities. Their low sum and carry-out form a two-part carry state. The second total’s high half supplies a separate modulus-related carry.

#### Following positions and mandatory external reads

The following-position loop considers indices from the current position plus one through N − 1.

**Every evaluation of this loop’s condition first reads the word encoder into a retained value, then reads the word decoder into another retained value, and only then evaluates the index comparison.** This includes the final unsuccessful condition check.

Consequently:

- These two reads occur even when the loop body executes zero times.
- Each later condition evaluation replaces the retained pair.
- After loop termination, the pair comes from the final unsuccessful check.
- The loop body’s arithmetic does not replace its ordinary external reads with this retained pair.

For each position whose condition succeeds:

1. Decode the primary word using a fresh decoder read and add it to the low carry state.
2. Retain that addition’s low sum.
3. Replace the low carry state with the previous upper carry state plus the addition’s carry-out; reset the upper carry state to zero.
4. Form a 64-bit total from the reduction factor times the decoded modulus word, the retained low sum, and the modulus-related carry.
5. Retain its high half as the next modulus-related carry.
6. Encode its low half using a fresh encoder read and store it to the preceding primary position.

The modulus decoding also uses a fresh decoder read.

#### Top-word calculations and writes

After the final unsuccessful following-position check, the routine decodes primary word N using a **fresh** decoder read, not the retained decoder.

It adds that word to the low carry state. The low part of this addition is then combined with the modulus-related carry:

- Primary word N − 1 receives the encoded low part of the combined sum.
- Primary word N receives the encoding of the outstanding upper carry plus the carry-outs from these two additions.

The encoded value destined for primary word N is computed first, using an encoder read. The routine then computes and writes primary word N − 1 using another encoder read, and finally writes the previously computed value to primary word N.

These operations do not overwrite the encoder and decoder retained by the final following-position condition check. All top-word operations are inside a positive-count iteration; no N − 1 destination is used when N is zero.

#### Secondary-history stores using the retained pair

Only after the primary top-word calculations and writes does the routine update secondary history.

Using the **encoder and decoder retained by the final unsuccessful following-position check**, it:

1. Reads the existing secondary word at the current position and decodes it with that retained decoder.
2. Takes the bitwise OR of that decoded value and the current input word shifted left one bit, discarding overflow.
3. Encodes the OR result with the retained encoder and writes it to the current secondary position.
4. Encodes the current input word’s high bit, represented as zero or one, with the same retained encoder and writes it to the next secondary position.

The second store does not require a decoder operation. The high-bit value is derived through signed right shift and negation in the supplied representation.

The iteration factor is then set to two, and the next outer position is selected.

### 4. Normalize the primary length

Normalization starts at primary index N.

While the index is positive, the routine decodes the primary word at that index using a decoder read. A zero result decrements the index; a nonzero result stops the scan. At index zero, short-circuit evaluation prevents another word probe.

The routine writes the primary length as the low 32 bits of the length encoder multiplied by one plus the final index.

Thus:

- Normalization never selects fewer than one word on valid-count paths.
- Primary word zero is not tested to distinguish zero from a one-word value.
- When N is zero, no normalization word probe occurs and the length becomes the length encoder.
- Normalization does not clear words above the selected length.

### 5. Invoke dependencies and normalize their status

The routine calls F-61e238108dfa with the retained primary descriptor address and the modulus descriptor address.

Its 32-bit return is interpreted as signed:

- Negative: skip F-8a9589a9b71f.
- Zero or positive: call F-8a9589a9b71f with those same descriptor addresses.

If the conditional dependency returns any nonzero 32-bit value, this routine returns one immediately.

Otherwise, it calls F-c64ad9e9ad7c with the original output argument and retained primary descriptor address. A zero return becomes final zero; any nonzero return becomes final one.

There is no direct revalidation, re-normalization, or explicit rollback between these calls.

## Side effects

### Direct writes

For a valid nonnull-context execution, this routine directly writes:

- Primary words zero through N + 3 during initialization.
- Primary words zero through N during positive-count arithmetic.
- The primary descriptor’s length during normalization.
- Secondary word zero during initialization.
- Secondary words zero through N across positive-count iterations.

It does not directly assign the secondary length, input length, or modulus length. Aliased storage can nevertheless make a write affect another logical object.

There is no direct allocation or release operation in the body. The absence of such operations does not establish dependency ownership behavior.

### Ordering, retention, and failure

The phase order is initialization, arithmetic and history updates, normalization, comparison, optional adjustment, and output.

Retained values include the first length multiplier, the derived counts, the initial decoded input word, the workspace descriptor addresses, each iteration’s reduction factor, and the final condition-check encoder/decoder pair used for that iteration’s secondary stores.

Other external values and data-array addresses are read at their individual access sites. No synchronization is present. The specification does not impose left-to-right ordering on independent operands inside a single expression.

**After a dependency returns failure, this routine performs no explicit rollback.** The resulting memory state and surviving effects depend on the invoked dependencies. A dependency may overwrite or restore workspace contents, reverse effects, or release storage. Preservation of earlier writes is not guaranteed.

## Dependencies

| Opaque ID | Invocation condition | Arguments | Return treatment |
|---|---|---|---|
| F-61e238108dfa | Every nonnull-context execution reaching the call stage | Primary descriptor, modulus descriptor | Signed negative skips adjustment; otherwise invokes it |
| F-8a9589a9b71f | Comparison result is nonnegative | Primary descriptor, modulus descriptor | Zero continues; nonzero returns one |
| F-c64ad9e9ad7c | Comparison is negative, or adjustment returns zero | Original output argument, primary descriptor | Zero returns zero; nonzero returns one |

Comparison, arithmetic adjustment, and output transfer are inferred roles, not established dependency contracts.

No dependency is assumed to preserve globals, descriptors, storage, or ownership. Later calls receive the retained addresses regardless of changes made by earlier dependencies. Safe use therefore depends on dependency behavior not established here.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

### Distinct length-multiplier reads

Suppose the first multiplier read yields two, the separate modulus multiplier read yields three, the modulus length is one, and the input length is one. With valid arithmetic and storage, N is three and M is two. The later input-count calculation uses the retained two, not another multiplier read.

### Null context

With a readable modulus header, valid count arithmetic, and a zero context:

- Both initial length-multiplier reads occur.
- Modulus-count derivation precedes the context branch.
- The null-context result is returned.
- No input descriptor or workspace is accessed, and no dependency is called.

The result need not be zero or one.

### Zero modulus-derived count

With N = 0 and otherwise valid storage:

- Four primary words are cleared.
- Input word zero is still read and decoded.
- Secondary word zero is cleared.
- No outer iteration or following-position condition executes.
- Primary length becomes the length encoder.
- Dependency processing still occurs.

### Final condition snapshot

For the last outer iteration of any positive N, there are no following-position body executions. Nevertheless, the condition reads encoder first and decoder second before failing its comparison.

That pair is retained through the subsequent fresh top-word decoder and encoder reads and the primary writes. It supplies the encoding and decoding values for the secondary-history update.

### Single-word identity representation

Assume N = M = 1, all multipliers and encoders remain one, the coefficient is zero, input word zero is three, and storage is valid and nonoverlapping.

Immediately before dependency calls:

- Primary words zero and one are zero.
- Primary length is one.
- Secondary words zero and one are six and zero.

The discarded low square value is nine. This is not an ordinary unreduced-square contract.

### Conditional dependency failure

If comparison is nonnegative and F-8a9589a9b71f returns two:

- The output dependency is not called.
- This routine returns one, not two.
- This routine performs no explicit rollback.
- Final workspace contents and surviving effects depend on what the invoked dependencies did.

### Negative comparison and successful output

If comparison returns a signed negative value and the output dependency returns zero, adjustment is skipped and the final result is zero. No preservation claim follows for workspace contents.

## Uncertainties

The following remain **documented_only**:

- External constant values and their representation semantics.
- The exact mathematical result and required modulus/coefficient invariants.
- Whether adjustment establishes a canonical residue.
- Supported aliasing arrangements.
- Dependency mutation, lifetime, ownership, allocation, release, and error contracts.
- Behavior under signed overflow, invalid addresses, or excessive counts.
- Guarantees under concurrent modification of external values or memory.
- Platform semantics required for the signed-shift high-bit extraction.

## Evidence

**Established static evidence:** the supplied body supports the layouts, widths, distinct initial multiplier reads, retained input-count multiplier, per-condition encoder-then-decoder reads, final-check snapshots, intervening top-word operations, secondary-store snapshot use, arithmetic phases, normalization, call order, and return rules described above.

**Inferred intent:** encoded-word modular squaring/reduction using caller-provided scratch storage.

**Unknown:** external implementations, representation invariants, mathematical guarantees, and ownership behavior.

No runtime tests have been performed.
