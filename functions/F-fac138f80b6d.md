<!-- Trusted provenance: F-fac138f80b6d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fac138f80b6d

## Purpose

F-fac138f80b6d performs a count-limited transfer from a source address to a destination address and returns the original destination address on normal completion.

The visible behavior combines an optional aligned, four-byte transfer phase with a byte-at-a-time phase. The byte phase stops after storing a zero-valued byte or after exhausting the remaining count. When it stores a zero-valued byte, it invokes an unresolved dependency with the next destination address and the remaining count.

**Inferred intent:** the structure resembles a bounded string copy with a possible padding operation after a terminator.

**Established limitation:** neither the value controlling the aligned-phase stopping test nor the implementation of the post-terminator dependency is supplied. Consequently, this contract does not establish conventional bounded-string-copy semantics, guaranteed zero padding, or correct terminator detection in the aligned phase.

## Interface

### Inputs

1. **Destination address**
   - Identifies the starting location for writes.
   - The displayed interface uses a pointer to unsigned words, but the implementation also performs single-byte writes.
   - This address is preserved as the normal return value.

2. **Source address**
   - Identifies the starting location for reads.
   - The displayed interface uses a pointer to unsigned words.
   - Both transfer phases perform word-sized source reads. In the byte phase, successive reads begin one byte apart.

3. **Count**
   - A signed integer controlling the number of destination bytes transferred directly.
   - A positive count permits transfer.
   - A zero or negative count reaches the return condition without a source read or destination write, subject to the arithmetic qualification below.
   - It is not an unsigned length: negative values must not be reinterpreted as large positive requests.

### Width and representation model

The source structure indicates a 32-bit address and integer model:

- An unsigned word occupies four bytes.
- The count and temporary signed arithmetic use 32-bit integers.
- Addresses are converted to integer representations for alignment tests and one-byte address advancement.
- Alignment is tested against a four-byte boundary.

These widths are the working model for the behavior described here. They are not independent confirmation of the original binary’s ABI, endianness, alignment-fault rules, or compiler arithmetic semantics.

The byte phase narrows a loaded word to a character-sized value and stores one byte. In the intended eight-bit-byte model, this selects the low eight bits of the loaded word. Which addressed source byte supplies those bits depends on endianness.

### Output

On every visible normal return, the result is the original destination address.

There is:

- No returned byte count.
- No error status.
- No visible replacement of the return value with the post-terminator dependency’s result.

A fault or failure that prevents normal return is not converted into a documented error result.

### Access preconditions

For defined, successful execution:

- Every executed word-sized source read must be valid.
- Every executed destination write must be valid at its actual width.
- Address advancement must remain representable and within valid storage.
- Any invoked dependency’s preconditions must also hold; those preconditions are not supplied.

Destination capacity equal to the positive count covers the direct writes visible in this function. It does not establish sufficient capacity for the unresolved dependency.

Source accessibility cannot be specified merely as “the number of bytes copied.” The byte phase performs a full word read even when only one destination byte remains.

## Behavior

### Entry and shared-state read

The function first reads one global integer value used by the aligned-phase stopping test. This read precedes the alignment decision and the count-based early return.

Therefore, even a zero-count or negative-count invocation visibly includes a global read. Whether that read has any special hardware or concurrency significance is unknown.

The original destination address is retained for the return value. Separate advancing source and destination positions are used during transfer.

### Selection of the aligned phase

The aligned phase is eligible only when both starting addresses are divisible by four.

If either starting address is not four-byte aligned, the function proceeds directly to the byte phase. It does not first transfer bytes to obtain alignment.

If both addresses are aligned, the function considers a word transfer only while the remaining signed count is greater than three. Counts of one, two, or three therefore proceed directly to the byte phase even when both addresses are aligned.

### Aligned word transfers

For each eligible aligned iteration, the observable order is:

1. Read one unsigned word from the current source address.
2. Evaluate the stopping test using that word and the global value captured at entry.
3. If the test is nonzero, leave the aligned phase without storing the word or advancing either address.
4. Otherwise, store the entire word at the current destination address.
5. Advance both addresses by four bytes.
6. Reduce the remaining count by four.

The stopping test combines:

- The loaded word minus the captured global value.
- The bitwise complement of the loaded word.
- The captured global value shifted left by seven bits.

The three results are combined by bitwise AND; a nonzero result stops the aligned phase.

The global value is not supplied. Thus, the contract establishes the test’s role and inputs, but not which word contents trigger it. In particular, the visible material alone does not prove that every word containing a zero byte triggers the test, or that every word without a zero byte passes it.

A word accepted by this test is copied in full. The function does not inspect or individually store its four component bytes before that write.

If the test stops the aligned phase, the source address remains at the beginning of the word just examined. The byte phase then performs another word-sized read at that same source address. These are distinct reads; they need not observe identical contents if the memory changes between them.

### Byte phase and exhaustion

The byte phase is entered after the aligned phase ends or when that phase is ineligible.

At the start of each byte-phase iteration, a decremented remaining-count value is calculated. The function then tests the current, not-yet-decremented signed count:

- If the current count is less than one, it returns the original destination address.
- Otherwise, it performs the next byte transfer.

Consequently, ordinary zero and negative counts cause no source access and no destination write. However, the decrement is visibly calculated before the test. For the minimum 32-bit signed value, subtracting one is outside the signed range. The supplied representation does not establish whether this corresponds to wrapping machine arithmetic or an operation subject to language-level overflow restrictions. That boundary is documented_only.

For a positive remaining count, the transfer order is:

1. Read a full unsigned word at the current source address.
2. Store the narrowed character-sized value as one byte at the current destination address.
3. Advance the destination address by one byte.
4. Advance the source address by one byte.
5. Adopt the decremented count.
6. Test whether the transferred character-sized value is zero.

The zero test uses the value already loaded. It does not reread the destination or source.

If the value is nonzero, the function repeats the byte phase. If that transfer exhausted the count, the next iteration returns without another source read.

### Zero-valued byte and dependency invocation

If a byte-phase transfer stores zero, the function leaves that loop and invokes **F-000000000001**, the opaque designation used here for the unresolved external dependency.

Its visible arguments are:

1. The destination address immediately after the zero byte just stored.
2. The remaining count after accounting for that byte.

This invocation occurs even when the zero byte used the final permitted position and the remaining count is zero.

On this path, the remaining count passed to the dependency is nonnegative: a byte transfer begins only with a positive count and consumes one position.

After the dependency returns normally, F-fac138f80b6d returns the original destination address. The dependency’s return value does not affect that visible result.

No claim is made that the dependency:

- Writes zeros.
- Writes exactly the remaining count.
- Leaves memory unchanged when the count is zero.
- Preserves globals.
- Preserves source storage.
- Allocates, retains, or releases memory.
- Returns normally.

### Overlap and changing memory

The function does not detect overlapping source and destination ranges and does not select a backward-copy strategy.

Its direct accesses proceed forward, but at different widths:

- The aligned phase loads a whole word before storing that word.
- The byte phase loads a whole word before storing one byte.
- Later loads can observe effects of earlier destination writes when the ranges overlap.

Therefore, overlap is not established to have snapshot-copy or move-style semantics.

Likewise, repeated reads of overlapping source words do not promise a consistent snapshot when another actor or memory-mapped device changes their contents.

## Side effects

### Directly established effects

The function can:

- Read the global stopping-test value once at entry.
- Read four-byte source words.
- Write four-byte destination words in the aligned phase.
- Write single destination bytes in the byte phase.
- Invoke the unresolved dependency after a zero byte is stored.

For a positive count, the direct transfer writes consume no more than that count. The unresolved dependency is outside this direct-write bound.

No explicit allocation, deallocation, ownership transfer, or global write appears in the visible body. This observation does not constrain the dependency’s effects.

### Access boundaries and faults

The byte phase may read beyond the logical transfer boundary because a one-byte write is preceded by a four-byte read.

For example, a one-byte request can still require four readable bytes at the initial source address. A terminator near the end of an accessible region does not, by itself, make the access safe.

After the first byte-phase advancement, subsequent source reads may be unaligned. Whether such reads succeed, fault, or have special device behavior depends on the execution environment.

No visible validation or recovery handles:

- Null or inaccessible pointers.
- Insufficient destination space.
- Insufficient readable source extent.
- Unsupported unaligned reads.
- Address overflow.
- Faults during a read or write.
- Dependency failure.

Any writes completed before a later fault are not visibly rolled back.

## Dependencies

### Entry global value

An unidentified global integer supplies the aligned-phase stopping-test parameter.

Established properties:

- It is read before the transfer decisions.
- The captured value is reused throughout the aligned phase.
- Its numeric value is absent from the supplied evidence.

Unknown properties include mutability, initialization, special access semantics, and its intended bit pattern.

### F-000000000001

This is a document-local opaque ID for the unresolved external call. No authoritative dependency mapping or implementation was supplied for it.

Established call contract:

- Called only after the byte phase stores a zero-valued byte.
- Receives the next destination address and the remaining count.
- Can be called with a zero remaining count.
- Its return value is unused.

Its memory effects, ownership rules, global effects, error behavior, and termination behavior are documented_only.

No other function calls are visible in the supplied body.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

### Zero count

Given valid execution of the preliminary arithmetic and global read, a count of zero produces:

- The entry global read.
- No source read.
- No destination write.
- No dependency invocation.
- Return of the original destination address.

### Ordinary negative count

For a count such as negative one, with representable decrement arithmetic:

- The global is read.
- No transfer occurs.
- The dependency is not called.
- The original destination address is returned.

This example does not resolve the minimum signed-count boundary.

### One-byte request without a zero value

Given one remaining byte and a source word whose narrowed byte is nonzero:

- A full source word is read.
- One destination byte is written.
- No dependency call occurs.
- The function returns the original destination address on the next exhaustion check.

Only one writable destination byte is needed for the direct transfer, but a full readable source word is needed.

### Zero in the final permitted position

Given one remaining byte and a source word whose narrowed byte is zero:

- One zero byte is stored.
- F-000000000001 is called with the next destination address and count zero.
- Normal dependency return is followed by return of the original destination address.

The zero-count call must not be omitted from the contract.

### Aligned accepted word

Given aligned starting addresses, count four, and a loaded word for which the stopping test is zero:

- One source word is read.
- That word is stored in full.
- The remaining count becomes zero.
- The function returns without calling the dependency.

This remains the established behavior even if someone interprets the word as containing a string terminator; acceptance depends on the actual stopping test.

### Aligned rejected word

Given aligned starting addresses, count at least four, and a loaded word for which the stopping test is nonzero:

- The aligned phase reads but does not store that word.
- Neither address advances during that attempt.
- The byte phase reads a word again at the same source address.
- Subsequent behavior depends on that second read’s narrowed value.

### Unaligned starting address

If either starting address is unaligned, no aligned word store is attempted. The byte phase still uses word-sized source reads; it is not a byte-read fallback.

## Uncertainties

The following remain documented_only:

- The global value and the resulting exact stopping-test predicate.
- Whether the intended predicate reliably detects zero bytes.
- Endianness and the addressed-byte meaning of word-to-character narrowing.
- Exact ABI widths beyond the working 32-bit model.
- Signed-shift behavior for the global value.
- Minimum signed-count decrement behavior.
- Unaligned word-read support and fault behavior.
- Pointer-conversion and address-overflow semantics.
- All undocumented effects and requirements of F-000000000001.
- Safe overlap conditions beyond the stated access ordering.
- Concurrency and memory-mapped-I/O behavior.

String-copy intent is inferred from structure, not a substitute for these missing facts. Zero-padding behavior is not established.

## Evidence

This contract is based solely on the supplied static function representation.

Established observations include the entry global read, four-byte alignment gate, signed count comparisons, word-sized source reads in both phases, distinct write widths, forward address advancement, post-store zero test, conditional dependency call, and original-destination return.

No dependency implementation, global initializer, ABI documentation, caller contract, or runtime trace was supplied. No runtime tests have been performed.
