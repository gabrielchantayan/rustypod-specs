<!-- Trusted provenance: F-2a82e70cfea2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2a82e70cfea2

## Purpose

The available static evidence describes a routine that performs an ordered series of indirect calls, computed-address word stores, computed-address byte stores, and an unconditional repeating call-and-store sequence.

A higher-level purpose is not established. In particular, the evidence does not justify describing this routine as an allocator, serializer, initializer, device driver, or decoder.

The visible routine has no normal return path. If execution reaches the final loop and every subsequent call and memory access completes, the loop repeats indefinitely. Earlier faults, non-returning dependencies, or control transfers inside dependencies remain possible.

This contract describes the supplied representation, not independently verified executable behavior. The evidence includes warnings about invalid instruction data and removed unreachable regions, so apparent signatures, boundaries, and control flow require qualification.

## Interface

### Explicit inputs

Three explicit arguments are shown:

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits, interpretation unspecified | Forwarded to the first indirect call. |
| Second argument | 32 bits, represented as signed | Used both as an address-like value and as a scaled index; forwarded to the first call. |
| Third argument | 32-bit unsigned | Shifted right by seven, then by another seven before being passed to a later call. |

The third argument therefore contributes its original value shifted logically right by fourteen. Its low fourteen bits do not affect that particular argument.

### Additional entry state

The representation also consumes machine state not described by the three-argument interface:

- An incoming address-like value, used to read a word at offset 20.
- An incoming unsigned 32-bit value, used in address calculations, shifts, and dependency arguments.
- An incoming 32-bit value, stored by the first visible operation.
- An incoming control-flow value, used to calculate the first indirect call target.
- A secondary result register exposed after a later dependency call.

These are hidden inputs, not established globals or ordinary formal parameters. A caller specification limited to the three explicit arguments is insufficient to determine the observed behavior.

### Outputs and completion

The displayed return type supplies no result value. No normal return is visible.

Observable outputs consist of:

- Writes to computed memory locations.
- Ordered calls and their explicit arguments.
- Any effects performed by dependencies.
- Potential faults or non-return caused by calls or accesses.

There is no explicit status code, error object, rollback operation, or cleanup path.

### Arithmetic and memory model

Word loads and stores are represented as 32-bit operations. Byte loads and stores are represented as 8-bit operations. Right shifts of the unsigned quantities are logical.

For describing target-sized arithmetic below, “low 32 bits” means reduction modulo \(2^{32}\). This is a conditional machine-level interpretation, not a claim that the displayed signed expressions define portable arithmetic. Signed multiplication and left shifts can overflow under a source-language interpretation; the supplied representation does not resolve that discrepancy.

Endianness, alignment requirements, address-space permissions, and behavior on invalid addresses are not established.

## Behavior

### Reading conventions

The following descriptive values distinguish data lifetimes without assigning application meaning:

- **Entry word**: the word read from the hidden incoming address plus 20.
- **Entry unsigned value**: the hidden unsigned 32-bit input.
- **Quarter-scale result**: four times a particular dependency result.
- **Sixteen-scale address**: sixteen times a particular dependency result.
- **Selector address**: a later computed address from which an 8-bit selector is read.
- **Packed result**: a dependency result subsequently used in masked bit extractions.

Each reference to a saved result means the value retained in the supplied representation. It does not imply that a dependency preserves unrelated registers, globals, or memory.

### Dependency notation

The opaque IDs F-U01 through F-U36 below identify unresolved call sequences in encounter order. Suffixes distinguish individual call positions, not proven distinct implementations.

A **five-call sequence** F-Un has these observable calls, in order:

1. F-Un.1 receives the arguments stated for that sequence.
2. F-Un.2 has no explicit arguments shown.
3. F-Un.3 has no explicit arguments shown.
4. F-Un.4 has no explicit arguments shown.
5. F-Un.5 has no explicit arguments shown and supplies the result used afterward.

“No explicit arguments shown” does not establish that no register or memory inputs exist.

### Initial operations

The first visible memory operation stores the hidden incoming 32-bit value at four times the second argument, plus 20. This occurs before the entry-word read.

The routine then reads the entry word from the hidden incoming address plus 20.

The initial call sequence is:

| Opaque ID | Explicit arguments or result use |
|---|---|
| F-U01.1 | First and second arguments. Its target also depends on incoming control-flow state. |
| F-U01.2 | No explicit arguments; result forwarded to F-U01.3. |
| F-U01.3 | The result of F-U01.2. |
| F-U01.4 | No explicit arguments; result forwarded to F-U01.5. |
| F-U01.5 | The result of F-U01.4. |
| F-U01.6–F-U01.9 | Four successive calls without explicit arguments; the last supplies a result. |

After F-U01.9 returns, the routine writes the entry word shifted logically right by eight to the second argument plus 20.

The two destinations derived from the second argument are different expressions. They must not be conflated into one object field. They can alias for particular inputs.

### Word-store and call sequence

The following operations occur in table order. A store listed before a call sequence precedes all calls in that sequence. Intermediate calls must not be removed merely because their results are unused: their side effects and control-flow behavior are unknown.

| Stage | Ordered behavior |
|---|---|
| F-U02 | Run a five-call sequence whose first argument is four times the final F-U01 result. Save the final result. |
| F-U03 | Store four times the F-U02 result at sixteen times that result, plus 20. Run a five-call sequence whose first argument is the entry unsigned value shifted right by eight. |
| F-U04 | Store the saved four-times F-U02 result at that same four-times result plus 20. Run a five-call sequence whose first argument is the F-U03 result shifted left by two. |
| F-U05 | Store four times the F-U04 result at sixteen times that result, plus 20. Retain both scaled values. Call F-U05.1 with the sixteen-scale address; F-U05.2 without explicit arguments; F-U05.3 with its predecessor’s result; F-U05.4 without explicit arguments; and F-U05.5 with its predecessor’s result and the entry unsigned value shifted right by eight. Four more calls without explicit arguments follow; the last supplies the stage result. |
| F-U06 | Repeat the word store made immediately before F-U05, using the retained address and value rather than recomputing them from memory. Run a five-call sequence beginning with four times the F-U05 result. |
| F-U07 | Store four times the F-U06 result at sixteen times that result, plus 20. Shift the third explicit argument right by seven. Call F-U07.1 with the saved quarter-scale result and eight times the entry unsigned value. F-U07.2 has no explicit arguments; F-U07.3 receives its result. F-U07.4 has no explicit arguments; F-U07.5 receives its result and the entry unsigned value shifted right by eight. Four calls without explicit arguments follow; the last supplies a result. |
| F-U08 | Store the retained four-times F-U06 result at eight times the entry unsigned value, plus 20. Run a five-call sequence beginning with four times the F-U07 result. |
| F-U09 | Store four times the F-U08 result at sixteen times that result, plus 20. Run a five-call sequence beginning with four arguments: eight times the entry unsigned value, that same value again, the entry unsigned value shifted right by eight, and the original third argument shifted right by fourteen. |
| F-U10 | Store eight times the entry unsigned value at eight times that input, plus 20. Run a five-call sequence beginning with the F-U09 result shifted left by two. After the last call, consume the secondary result register, rather than its ordinary displayed return value. |
| F-U11 | Multiply that secondary result by \(2^{24}\). Store it at the secondary result multiplied by \(2^{26}\), plus 20. Run a five-call sequence beginning with the saved \(2^{24}\)-scaled value. |
| F-U12 | Store the retained \(2^{24}\)-scaled value at four times the F-U11 result, plus 20. Run a five-call sequence beginning with four times the F-U11 result. |
| F-U13 | Store four times the F-U12 result at sixteen times that result, plus 20. Call F-U13.1 with the saved quarter-scale result; F-U13.2 without explicit arguments; F-U13.3 with its result; F-U13.4 without explicit arguments; F-U13.5 with its result; then four calls without explicit arguments. |
| F-U14 | Store the retained four-times F-U12 result at four times the F-U13 result, plus 20. Run a five-call sequence beginning with four times the F-U13 result. |
| F-U15 | Store four times the F-U14 result at sixteen times that result, plus 20. Retain the quarter-scale result for later masking. Call F-U15.1 with that result; F-U15.2 without explicit arguments; F-U15.3 with its result; F-U15.4 without explicit arguments; F-U15.5 with its result; then four calls without explicit arguments. |
| F-U16 | Store the entry unsigned value shifted right by sixteen at four times the F-U15 result, plus 20. Run a five-call sequence beginning with four times the F-U15 result. |
| F-U17 | Store four times the F-U16 result at sixteen times that result, plus 20. Form the selector address by clearing bit 31 of the retained four-times F-U14 result and shifting right by seven. Run a five-call sequence beginning with three arguments: sixteen times the F-U16 result, the same value again, and the entry unsigned value shifted right by fifteen. |
| F-U18 | Store the retained sixteen-times F-U16 result at that same value plus 20. Run a five-call sequence beginning with the F-U17 result shifted left by two. Retain its final result as the first packed result. |

The explicit entry-word read is not repeated. Changes made by dependencies to its original memory location do not change the already retained word in this representation.

### Selector-mediated byte writes

The next destination is determined through a byte-sized indirection:

- Read one byte at the selector address plus one.
- Add 17 to that byte’s unsigned value.
- Store one byte at the resulting address.

Consequently, the destination is between 17 and 272 inclusive. The loaded byte is not a full-width pointer.

The first such store writes the low eight bits of the first packed result shifted left by two.

Next:

1. F-U19.1 receives three identical arguments, each the first packed result shifted left by two.
2. F-U19.2 has no explicit arguments.
3. F-U19.3 receives the preceding result.
4. Four calls without explicit arguments follow; the last supplies a result.

The routine then performs a fresh selector-byte read and stores the low eight bits of the first packed result masked with \(0x3fffffff\), then shifted right by twenty.

F-U20 is a five-call sequence beginning with the final F-U19 result shifted left by two. Its final result becomes the second packed result.

After F-U20:

1. Store the low eight bits of the second packed result shifted left by two at the entry unsigned value shifted right by twenty-three, plus 28.
2. Read the selector byte again and store zero at its derived destination.

These are separate ordered writes. Aliasing can make the second overwrite the first.

F-U21.1 receives the entry unsigned value shifted right by twenty-three. F-U21.2 has no explicit arguments. F-U21.3 receives the second packed result masked with \(0x3fffffff\), then shifted right by six. Four calls without explicit arguments follow.

Another fresh selector-byte read and zero store follow F-U21.

F-U22 is a five-call sequence beginning with the final F-U21 result shifted left by two.

### Transition to the final loop

Mask the F-U22 result with \(0x3fffffff\), then shift right by twenty-five. The resulting value is between zero and 31.

A different selector source is now used: 1,024 times the retained second packed result shifted left by two, plus one. The routine reads a byte there and stores zero at that byte’s unsigned value plus 17.

F-U23.1 receives twice the zero-to-31 value. F-U23.2 has no explicit arguments and supplies the initial loop-carried result.

Each loop iteration then:

1. Runs the five-call sequence F-U24. Its first call receives the loop-carried result and the current bounded value shifted right by eight.
2. Reads a byte from the newer selector source and stores zero at its derived destination.
3. Runs the five-call sequence F-U25, beginning with the F-U24 result shifted left by two.
4. Sets the next loop-carried result to the F-U25 result shifted left by two.
5. Sets the next bounded-value state to the F-U25 result shifted left by twenty-six.

The loop has no visible exit condition.

The second argument to F-U24.1 is zero on the first iteration. Under 32-bit truncating shift arithmetic, later values are not necessarily zero: after the shift by eight, only bits 18 through 23 can remain set.

## Side effects

The routine directly performs word stores, byte stores, one initial word read, and repeated selector-byte reads. Dependency calls can add arbitrary effects not described here.

Observable ordering matters:

- The first word store precedes the initial word read.
- Calls separate many apparently repeated stores.
- Each selector-mediated write performs a new selector read.
- The two consecutive byte stores following F-U20 occur without an intervening call.
- The final loop repeats its zero store even when the computed destination remains unchanged.

No volatility, atomicity, locking, memory barrier, or concurrency guarantee is established. Device-memory interpretations are possible but unproven.

No ownership transfer, allocation, or release is directly established. Dependency arguments must not be treated as owned pointers merely because they resemble addresses. Likewise, no dependency is known to preserve global state or release memory.

## Dependencies

Only F-2a82e70cfea2 has a supplied resolved opaque identity. F-U01 through F-U25 are documentation-local opaque identifiers for unresolved call sequences. They do not establish unique target functions or a complete ABI.

The dependency notation reserves no semantics beyond the explicit arguments, result uses, and order stated above. Multiple call positions may resolve to the same target. A dependency may fault, not return, modify memory, or affect later selector reads.

The secondary result consumed after F-U10 is especially uncertain: the evidence does not supply a declared multi-result interface or explain how that register value is produced.

## Acceptance examples

These examples are conditional static obligations, not executed tests.

1. **Initial ordering:** If all addresses are valid, the hidden incoming value is stored before the entry word is read. If those locations alias, the load can observe the just-stored value.

2. **Saved-word shift:** If the loaded entry word is \(0x12345678\), the later word store to the second argument plus 20 writes \(0x00123456\), provided execution reaches it.

3. **Third-argument truncation:** Original third-argument values \(0x00003fff\) and zero both contribute zero to F-U09.1’s fourth explicit argument. \(0x00004000\) contributes one.

4. **Selector rereading:** If one selector read returns 3, that write targets address 20. If a dependency changes the selector byte to 9, a later write targets address 26. Reusing the earlier selector would not match the visible behavior.

5. **Byte extraction:** If the first packed result is \(0x12345678\), the first selector-mediated store writes \(0xe0\). The later masked-and-shifted store writes \(0x23\).

6. **First loop argument:** Regardless of the F-U22 result, the first bounded value is at most 31. Its right shift by eight is therefore zero.

7. **Later loop argument:** Under 32-bit truncating arithmetic, if F-U25 returns one, the next bounded-value state is \(0x04000000\), and the next F-U24.1 second argument is \(0x00040000\).

8. **Nontermination:** If every operation continues successfully, observing one loop iteration does not establish completion. The visible loop repeats without returning.

## Uncertainties

The following are **documented_only**:

- Executable accuracy of the supplied representation, given its instruction-data and reachability warnings.
- Actual calling convention, hidden-input provenance, and secondary-result ABI.
- Target arithmetic behavior where represented signed operations overflow.
- Alignment, endianness, memory mapping, and fault behavior.
- Whether computed addresses identify ordinary memory, device registers, or invalid locations.
- Resolved dependency identities and all unshown dependency effects.
- Allocation, ownership, release, and synchronization semantics.
- Any behavior outside the displayed reachable representation.

Repeated scaling and offset-20 stores suggest a structural pattern, but no object layout or domain interpretation is established. The only supported layout facts are the access widths and address expressions described above.

## Evidence

This contract is based solely on the supplied static representation and its accompanying warnings.

Established static observations include the explicit argument widths, word and byte access forms, unsigned masks and shifts, call ordering, result forwarding, repeated selector reads, and unconditional final loop.

The modulo-\(2^{32}\) arithmetic model is explicitly conditional where signed-expression reconstruction leaves ambiguity. Higher-level intent remains unknown.

No implementation access, runtime execution, smoke run, or runtime tests have been performed.
