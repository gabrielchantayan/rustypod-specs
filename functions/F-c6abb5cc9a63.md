<!-- Trusted provenance: F-c6abb5cc9a63; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c6abb5cc9a63

## Purpose

F-c6abb5cc9a63 exposes a long, straight-line sequence of byte writes, word writes, memory reads, and calls to mostly unresolved indirect dependencies. Its application-level purpose is not established.

The supplied representation ends at an explicit bad-instruction termination marker and carries a warning that instruction data disrupted control-flow recovery. Consequently, this contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Conditional interpretation:** results obtained under the apparent 32-bit machine-arithmetic model, provided execution reaches the relevant operation.
- **Documented-only unknowns:** actual instruction validity, indirect dependency behavior, calling convention details, and continuation beyond the recovered boundary.

The byte and word operations must not be interpreted as a known object model, serialization format, allocator protocol, or device interface without additional evidence.

## Interface

### Explicit input and result

The visible interface accepts one unsigned 32-bit input, referred to here as the **input word**.

No explicit result is returned. This does not establish that execution returns normally: the visible sequence terminates at a bad-instruction marker rather than an ordinary return.

### Implicit execution inputs

The sequence also depends on:

- An incoming machine-register value used as an address and as an unsigned arithmetic operand. This contract calls it the **entry base**.
- An incoming return-link value used to derive the first indirect call target.
- Secondary register outputs associated with certain calls. Their producers and ABI meaning are not fully established.
- Values returned by unresolved indirect dependencies.
- Memory contents at computed addresses.

These are genuine dependencies of the displayed behavior, not additional parameters whose calling convention has been established.

### Widths and address calculations

The visible representation uses:

- Unsigned 8-bit reads.
- 8-bit writes that retain only the low eight bits of the supplied value.
- 32-bit reads and writes.
- Mostly 32-bit arithmetic operands and call results.
- One 64-bit call result, separated into low and high 32-bit halves.

The arithmetic descriptions below use the apparent 32-bit machine interpretation: additions, multiplications, and left shifts retain the low 32 bits. This interpretation is necessary to describe the apparent machine behavior, but is not independently verified against instructions or execution.

Unsigned right shifts insert zero bits. In particular, right shifts of the input word and explicitly unsigned values are not sign-extending.

### Memory access forms

Three recurring forms are important:

- **Selector-byte update:** read one byte at a computed source address, zero-extend that byte, add 17, and write one byte at the resulting address.
- **Indexed word update:** write four bytes at an address formed as four times an index plus 20.
- **Scaled word update:** write four bytes at an address formed as sixteen times an index plus 20.

A selector-byte update targets an address between 17 and 272 inclusive. The byte is not shown as a full-width pointer loaded from memory. Interpreting it as one would change the contract.

Endianness, address-space mappings, alignment requirements, and memory-access fault behavior are unknown.

## Behavior

### Ordering conventions

The stages below follow the displayed order. Each later stage is conditional on all preceding operations and calls completing sufficiently for execution to continue.

A **five-call chain** means five successive indirect calls:

1. The first receives the explicitly described arguments.
2. Three intervening calls have no explicit arguments in the supplied representation.
3. The fifth also has no explicit arguments and, where stated, supplies a captured result.

“No explicit arguments” does not establish an absence of register inputs, memory inputs, or side effects. Every call remains an observable dependency boundary.

Unless specifically described otherwise, indirect targets are formed by adding 4094 to a displayed code-position or code-label value. The first call instead uses the incoming return-link value plus 4094. One late call uses an offset of 4076.

### 1. Initial input-byte updates

The function first reads the byte at entry base plus one. It writes the low eight bits of the input word to the address selected by that byte plus 17.

It then performs a five-call chain whose first call receives the entry base logically shifted right by 12. The fifth call supplies a captured word.

After that chain, it rereads the byte at entry base plus one. This is a new read, not reuse of the first selector. The newly selected destination receives bits 22 through 29 of the input word.

A second five-call chain begins with four times the first captured word and supplies another captured word.

Next come two indirect calls: the first receives four times that second result and the selector retained from the second entry-base read; the second has no explicit arguments.

The selector is retained across these calls, but memory at its source or destination is not assumed unchanged.

### 2. Shift-derived accesses and the first zero-selector phase

An intermediate is formed by shifting the saved entry base left by 26 bits. Two separate 32-bit reads are shown at that intermediate plus 28. Their values are supplied as the two explicit arguments to an indirect call, followed by another call with no explicit arguments.

The two reads have no established relative evaluation order. They must not be assumed to produce equal values if memory can change between accesses.

A second intermediate is formed by multiplying the first intermediate by 1024. Under 32-bit modular arithmetic, this second intermediate is zero: the combined scaling shifts the original value beyond the available width.

The function then reads a selector byte at that second intermediate plus one and writes zero to the selected address plus 17. Under the same arithmetic model, the selector source is address one.

A five-call chain follows. Its first call receives:

- Zero.
- An unresolved secondary-register value.
- The second intermediate.

The fifth call supplies a captured word.

The selector source at the second intermediate plus one is read again. Its selected destination receives the low byte of the unresolved secondary-register value shifted right by 22. The provenance and stability of that secondary-register value are not established.

Another five-call chain starts with four times the captured word and supplies a further result.

An indirect call then receives the second intermediate shifted left by 26 and four times that further result. A following no-explicit-argument call supplies a word that is retained across several later stages. This contract calls it the **retained word**.

### 3. Direct dependency and linked word writes

F-674e75a953e8 is called with the retained word as both explicit arguments. No explicit result from that call is used.

After one further indirect call with no explicit arguments, the retained word is written at an indexed word address determined by another unresolved secondary-register value.

A five-call chain begins with that secondary value and the value obtained by masking it with 0x3fffffff and shifting right by 10. The returned index selects an indexed word destination, which receives the secondary value.

Another five-call chain starts with four times that index and supplies a new word. This contract calls it the **middle word**.

The function reads a selector byte at eight times the retained word plus one. The selected destination receives the low byte of four times the middle word.

### 4. Middle-word byte updates

A five-call chain starts with four times the middle word and returns a captured word.

Next, a selector byte is read at 64 times the middle word plus one. The selected destination receives the low byte of:

- The middle word masked with 0x3fffffff.
- Then logically shifted right by 20.

A five-call chain starts with four times the captured word and supplies another result.

Two indirect calls follow. The first receives four times that result and the selector just read; the second has no explicit arguments.

Another pair follows. Its first call receives the retained word shifted left by 10 as both explicit arguments. Its second call has no explicit arguments.

The function then reads the byte at address one and writes zero to the destination selected by that byte plus 17. This address calculation uses a displayed value explicitly set to zero before the relevant multiplication; it is not evidence that a preceding dependency returned zero.

### 5. Secondary-output phase and split result

A five-call chain starts with three arguments derived from a further unresolved secondary-register value:

- That value.
- The same value again.
- Sixteen times that value.

The chain returns a captured word.

A selector byte is read at sixteen times the secondary value plus one. Its selected destination receives the low byte of the secondary value shifted right by 22.

A five-call chain starts with four times the captured word and returns another word.

Two indirect calls follow: first with the retained selector and four times that returned word, then with no explicit arguments.

Another pair follows. The first receives zero twice. The second supplies a displayed 64-bit result. This contract calls its halves the **split low word** and **split high word**.

Zero is written to the indexed word destination selected by the split high word.

A five-call chain begins with the split high word and returns an index. The function writes the split high word to the indexed word destination selected by that index.

Another five-call chain begins with four times that index. Its results are not explicitly captured.

### 6. Split-low-derived word writes

The function writes the split low word multiplied by 0x04000000 to the address formed from the split low word multiplied by 0x10000000, plus 20.

A five-call chain starts with the value just written and returns an index. The indexed word destination selected by that index receives:

- The split low word masked with 0x1fffffff.
- Then logically shifted right by nine.

A further five-call chain begins with four times that index and returns another index.

That returned index initiates the repeated scaled-write behavior described next.

### 7. Repeated scaled and indexed word transfers

Five successive returned indices undergo the same initial operation: four times the index is written to sixteen times that index plus 20.

The intervening behavior differs and is significant:

1. **First index:** two separate word reads occur at four times the index plus 20. An indirect call receives one read shifted right by 12, four times the index, and the other read shifted left by two. Their relative read order is not established. Four further calls complete a five-call chain; its returned index selects an indexed destination that receives four times the first index.

2. **Second index:** after the scaled write, a five-call chain begins with four times this index twice, followed by the split low word shifted left by 26. The returned index selects an indexed destination receiving four times the second index.

3. **Third index:** after the scaled write, a five-call chain begins with four times this index. Its returned index selects an indexed destination receiving four times the third index.

4. **Fourth index:** after the scaled write, one word is read at four times this index plus 20 and retained. A five-call chain begins with four times the index. Its returned index selects an indexed destination receiving the retained word logically shifted right by 12.

5. **Fifth index:** after the scaled write, a five-call chain begins with three arguments:
   - The preceding indexed destination’s base, masked with 0x3f and shifted left by 14.
   - Four times the fifth index.
   - The preceding destination index shifted left by 30.

   The returned index selects an indexed destination receiving four times the fifth index.

Between these numbered steps, a five-call chain starts with four times the just-used destination index and supplies the next index.

After the fifth transfer, one more such chain supplies the final word used by the byte-update tail.

These operations do not establish distinct source and destination objects. Address wrapping and index relationships can make accesses overlap.

### 8. Final byte-update tail and termination boundary

A selector byte is read at 0x10000000 times the destination index from the fourth transfer, plus one. The selected destination receives the low byte of four times the final word.

An indirect call whose target uses the exceptional offset of 4076 receives:

- Four times the final word.
- That same value again.
- Sixteen times the final word.

The next no-explicit-argument call returns a word. That word is passed to another indirect call. Four additional calls follow, with the last supplying a captured result.

The selector source used at the beginning of this tail is read again. The newly selected destination receives the low byte of the final word masked with 0x3fffffff and shifted right by 20.

A five-call chain begins with four times the captured result and returns another word.

Two final indirect calls follow. The first receives four times that word and the latest selector; the second has no explicit arguments.

The supplied representation then terminates at a bad-instruction marker. No successful return, cleanup, status result, or subsequent behavior is established.

## Side effects

The function explicitly modifies memory through byte and 32-bit writes. It also performs calls whose additional effects are unknown.

Important constraints:

- Selector sources are reread at several points; caching them across dependency calls changes the displayed behavior.
- Distinct selector reads can choose different destinations.
- Reads and writes may alias one another.
- Low numeric addresses are explicit consequences of the represented calculations, not grounds for assuming the accesses are harmless.
- No allocation, ownership transfer, reference-count operation, or release is explicitly established.
- No dependency may be assumed to preserve global memory, registers outside an established ABI, selector contents, or resource ownership.
- No explicit bounds checks, null checks, error-status checks, retry behavior, or rollback appear.
- Partial writes can precede a later fault or non-returning call. No transactional guarantee is supported.

## Dependencies

### F-674e75a953e8

Established interface at this call site:

- Receives the retained 32-bit word twice.
- Occurs before the first secondary-output-indexed word write.
- Has no explicitly consumed return value.

Its memory effects, register effects, failure behavior, ownership rules, and termination behavior are unknown.

### F-unresolved-indirect

This opaque identifier denotes the unresolved dependency class, not one established shared function.

Its members include numerous distinct call sites, including the return-link-derived first target and the late exceptional-offset target. Their identities, calling conventions, argument completeness, result conventions, and memory effects remain documented-only unknowns.

Some results are consumed as indices or arithmetic operands. One result is displayed as 64 bits. Other values are obtained through secondary-register outputs rather than established explicit returns.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

1. **Initial truncation.** If the input word is 0x12345678 and the first selector read yields five, the first explicit write stores 0x78 at address 22.

2. **Fresh selector after calls.** With that same input, if the second entry-base selector read yields nine, the second input-derived write stores 0x48 at address 26. It must not reuse the first selector solely because the source address is unchanged.

3. **Selector range.** A selector value of 255 selects address 272. It does not select a destination based on a wider pointer load.

4. **Combined scaling.** Under 32-bit modular arithmetic, shifting any entry base left by 26 and multiplying the result by 1024 produces zero. The following selector source is therefore address one. This does not make the preceding word-read address zero: that address uses the earlier intermediate plus 28.

5. **Independent argument reads.** If a changing memory source yields different words for the two reads used in one call, the contract does not require equal arguments or permit replacing both reads with one cached value. Their relative evaluation order remains unknown.

6. **Zero split-result write.** If the split high word is three, the zero write immediately following result separation targets address 32. Subsequent dependencies may overwrite it.

7. **Masked extraction.** If the split low word is 0xffffffff, its masked-right-shifted value written in the split-low phase is 0x000fffff, not a sign-extended value.

8. **Conditional reachability.** If an early indirect dependency does not return, no later explicit write is required to occur. The visible sequence does not provide a recovery path.

## Uncertainties

The following remain **documented_only**:

- Whether the recovered sequence corresponds to executable instructions throughout.
- The true entry ABI and origin of implicit register inputs.
- Exact producers of unresolved secondary-register values.
- Whether the displayed 64-bit result is an ordinary ABI result or an analysis artifact.
- The complete set of indirect dependency inputs and effects.
- Target address width, endianness, memory mapping, alignment, and fault semantics.
- Whether low addresses identify ordinary memory, mapped hardware, or invalid locations.
- Whether signed-looking operations in the representation accurately model wrapping machine instructions.
- Whether execution reaches the terminal marker, and what the underlying machine does there.
- Any business-level meaning of the byte selectors, indices, or repeated transfers.

The repeated patterns suggest structured manipulation, but no particular higher-level intent is established.

## Evidence

This contract is based solely on the supplied static representation.

Evidence includes the explicit unsigned input width, byte and word access widths, arithmetic constants, call ordering, repeated selector reads, direct call to F-674e75a953e8, split result use, and bad-instruction termination boundary.

The warnings reduce confidence in treating the representation as a complete executable specification. Arithmetic simplifications and acceptance examples are conditional on the stated 32-bit model.

No runtime tests, instruction-level validation, dependency inspection, or memory-layout verification have been performed.
