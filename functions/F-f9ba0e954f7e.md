<!-- Trusted provenance: F-f9ba0e954f7e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f9ba0e954f7e

## Purpose

F-f9ba0e954f7e describes a call-heavy execution prefix that interleaves indirect dependency invocations with byte and 32-bit memory accesses. Visible operations include extracting bit fields, scaling returned values, selecting low numerical addresses through byte reads, and storing words at calculated addresses.

No higher-level purpose is established. The evidence does not identify a serializer, allocator, collection operation, or device protocol.

This contract distinguishes:

- **Established static evidence:** accesses, argument expressions, and ordering explicitly present in the supplied representation.
- **Conditional consequences:** results that follow under an explicitly stated arithmetic or execution assumption.
- **documented_only:** behavior not resolved by that representation.

The supplied material warns about invalid instruction data and overlapping global storage. Its visible endpoint is an invalid-instruction marker, not a demonstrated normal return. This contract therefore covers the represented execution prefix conditionally; it does not establish that the entire prefix is executable.

## Interface

### Explicit arguments

The represented interface accepts four 32-bit arguments.

| Argument | Visible use |
|---|---|
| First argument | Supplied to the first indirect invocation. |
| Second argument | No explicit use in the visible body. |
| Third argument | Shifted left by 26 bits and supplied to an indirect invocation alongside a saved selector byte. |
| Fourth argument | Shifted left by 10 bits and supplied as the third argument of an invocation whose first two arguments are identical. |

The second argument’s lack of explicit use does not establish behavioral independence from its incoming value. Actual calling conventions and implicit machine-state inputs remain unresolved.

### Additional execution inputs and preconditions

Execution also depends on:

- An incoming return-address-like value used to calculate the first indirect target.
- An incoming address-bearing value used for an early byte read.
- An incoming byte-sized value used for the first byte write.
- Another incoming 32-bit value used to calculate a later selector address.
- Represented secondary register results associated with dependency calls.
- Two reads of the same represented global word location.
- Contents and accessibility of calculated memory addresses.
- Validity and behavior of indirect call targets.

Consequently, the four explicit arguments alone are insufficient to establish valid execution. Continuing through the visible prefix requires every preceding dependency to return and every preceding memory access to succeed. Neither condition is checked visibly.

### Outputs and completion

No explicit return value, success status, or normal-return path is demonstrated. Observable outputs include memory writes and dependency invocations.

Whether the final invalid-instruction marker denotes a real processor fault, incorrect decoding, or an unavailable continuation is **documented_only**.

### Widths and layouts

A byte is 8 bits; a word is 32 bits. The represented paired result is 64 bits and is separated into low and high 32-bit halves.

Selector bytes are unsigned and zero-extended. A selector-directed byte write uses the selector’s numerical value plus 17 as its destination. Such destinations range from address 17 through address 272 inclusive.

A common word destination is four times an index plus 20. Other word destinations use 16 times an index plus 20, or an index multiplied by \(2^{28}\) plus 20. These are literal address calculations, not established record layouts or object-relative offsets.

Byte writes retain only the low eight bits. Right shifts of unsigned words are logical. Some arithmetic uses signed-looking operands; its overflow semantics are unresolved. Modular simplifications below apply only under 32-bit machine arithmetic.

## Behavior

### Ordering and retained values

The visible body has no explicit loop or conditional branch. Its order is linear, subject to calls returning and accesses succeeding. Intervening dependency calls are observable and cannot be discarded or assumed harmless.

The following descriptive names distinguish retained values:

- **First result:** an early returned word obtained through an exceptional indirect-target displacement.
- **Second result:** a later returned word obtained through another exceptional displacement.
- **First global sample:** the first word read from the represented global location after the first word store.
- **Second global sample:** the next read of that location, whose value is multiplied by four.
- **Scaled global value:** four times the second global sample.
- **Middle result:** the returned word later supplied twice to F-674e75a953e8.
- **Paired result:** a later represented 64-bit result.

These names do not imply pointer types, ownership, or preservation guarantees from dependencies.

### Initial selector operations

The first indirect invocation receives the first explicit argument. Several further calls follow, including one whose returned word is later forwarded after multiplication by four.

Before that forwarding invocation, the routine reads a byte at the incoming address-bearing value plus one. It writes the incoming byte-sized value to the selected address, meaning the byte read plus 17. The selector is retained for a later call.

After additional calls, an invocation receives the third explicit argument shifted left by 26 and the retained selector byte.

A subsequent invocation produces the first result. Another receives the first result twice and the fourth explicit argument shifted left by 10.

After another call, a selector byte is read at the incoming unsigned 32-bit value logically shifted right by nine, plus one. Zero is written to that selector’s destination.

### First-result field and first word store

A represented secondary register result is passed twice to a call, with zero as the third argument.

Following further calls, the routine reads a selector at the first result logically shifted right by 12, plus one. It writes the low byte of the represented secondary result shifted right by 22 to the selected destination.

A newly returned word is forwarded after multiplication by four. Additional calls culminate in an invocation receiving the selector and zero. A later call produces the second result.

The second result is then passed twice to another invocation, with the first result shifted left by 10 as the third argument. After a further call, the second result is stored as a word at four times a represented secondary result plus 20.

### Two distinct global reads

Immediately after that word store, the represented global location is read twice, in this order:

1. The first read produces the first global sample.
2. The second read produces the value multiplied by four to obtain the scaled global value.

No dependency call intervenes between these reads. They nevertheless remain distinct memory-read expressions. Storage semantics and asynchronous stability are not established, so equality of the samples must not be assumed.

Their subsequent uses differ:

- The first global sample supplies a call argument formed by masking to 30 bits and logically shifting right by 10.
- The second global sample supplies the scaled global value used in selector addresses and subsequent shifted-address calculations.

After the reads, a call receives the secondary result used to address the preceding word store. Further calls obtain an index. The second result shifted right by 12 is stored at four times that index plus 20.

### Global-derived selector updates

A later returned word supplies two byte-write values.

For the first update, the selector is read at the scaled global value plus one. The selected destination receives the low byte of four times the returned word.

A following invocation receives the first global sample masked to 30 bits and shifted right by 10.

After intervening calls, the selector location at the scaled global value plus one is read again. The newly selected destination receives the low byte of the same returned word masked to 30 bits and shifted right by 20.

The selector reads are separate observations. The second destination need not equal the first, even though their selector-address calculation is retained.

### Shifted global-derived addresses

The scaled global value is shifted left by 26. After intervening calls, two word-read expressions at that shifted value plus 28 supply an invocation’s two arguments. Their relative evaluation order is not established, and they do not constitute a guaranteed shared snapshot.

The shifted value is also multiplied by 1024. This product is used for:

- A selector read at product plus one, followed by a zero byte write.
- An invocation receiving zero, a represented secondary result, and the product.
- Another selector read at product plus one, followed by a byte write using the secondary result shifted right by 22.
- A later invocation receiving the product shifted left by 26 and a newly returned word multiplied by four.

Under 32-bit modular arithmetic, shifting left by 26 and then multiplying by 1024 yields zero for every starting word. Under that interpretation, the product-based selector address is one. This does not authorize adding an object base or replacing the represented arithmetic.

### Direct dependency and middle-result updates

A subsequent indirect invocation produces the middle result. F-674e75a953e8 is then invoked with two identical arguments, both equal to that result. No ordinary return value from this direct dependency is visibly consumed.

After another indirect call, the middle result is stored at four times a represented secondary result plus 20.

The following invocation receives that secondary result and its 30-bit-masked value shifted right by 10. Further calls obtain a new index, and the secondary result is stored at four times that index plus 20.

A later returned word drives two byte updates:

- A selector read at eight times the middle result plus one selects a destination receiving the low byte of four times the returned word.
- After calls, a selector read at 64 times that returned word plus one selects a destination receiving its 30-bit-masked value shifted right by 20, truncated to a byte.

Further calls receive returned values multiplied by four and retained selectors. Another invocation receives the middle result shifted left by 10 twice.

After a further call, the routine reads a selector at address one and writes zero to the selected destination. Here the address follows from an explicitly zeroed retained quantity, independently of the earlier conditional modular collapse.

### Secondary-result selector and paired result

Another represented secondary result is passed twice to an invocation, together with 16 times that result.

After additional calls, a selector is read at 16 times that secondary result plus one. Its destination receives the low byte of the secondary result shifted right by 22.

Further calls forward newly returned words multiplied by four and the retained selector. A subsequent invocation receives two zeros. The next represented call yields the paired result.

The paired result’s high half selects a word address at four times that half plus 20, where zero is stored. After intervening calls, a newly returned index selects a word destination receiving the high half.

The low half drives two later word writes:

- Its product with \(2^{26}\) is stored at its product with \(2^{28}\), plus 20.
- After dependency calls, its value masked to 29 bits and shifted right by nine is stored at four times a newly returned index plus 20.

These are separate writes with intervening calls.

### Late scaled-index word updates

Five late groups begin with a newly returned index. Each computes four times that index and stores that scaled value at 16 times the index plus 20. The groups then differ:

| Group | Operations following that initial store |
|---|---|
| First | A call receives two separate reads from the scaled value plus 20, transformed by right shift 12 and left shift two, with the scaled value between them as the second argument. Later, the scaled value is copied to a newly selected word destination. |
| Second | A call receives the scaled value twice and the paired result’s low half shifted left by 26. Later, the scaled value is copied to a newly selected word destination. |
| Third | A call receives the scaled value. Later, that value is copied to a newly selected word destination. |
| Fourth | A word is read at the scaled value plus 20 and retained before subsequent calls. Its logical right shift by 12 is later stored at a newly selected word destination. |
| Fifth | A three-argument invocation combines values from the fourth and fifth groups, followed later by a copy of the fifth group’s scaled value to a newly selected word destination. |

Each “newly selected word destination” in this table means four times a newly returned index plus 20. Indices from different groups are not asserted equal.

For the fifth group’s three-argument invocation:

- The first argument is the low six bits of four times the fourth group’s destination index, shifted left by 14.
- The second is the fifth group’s scaled value.
- The third is the fourth group’s destination index shifted left by 30.

The first group’s two same-address reads have unresolved relative evaluation order. In contrast, the fourth group explicitly retains one read value across later calls.

### Final selector writes

A late returned word is multiplied by four. Its low byte is written through a selector read at the fourth group’s destination index multiplied by \(2^{28}\), plus one.

A following invocation receives that product twice and the returned word shifted left by four. Another returned word is forwarded unchanged to a subsequent invocation.

After further calls, the same selector-address expression is read again. The newly selected destination receives the low byte of the earlier returned word masked to 30 bits and shifted right by 20.

The remaining visible calls forward newly returned words multiplied by four and the retained selector. The representation then ends at the invalid-instruction marker.

## Side effects

Visible effects include byte reads and writes, word reads and writes, two ordered global-word reads, and dependency invocations.

The global reads are not one atomic snapshot. The first saved sample and the second sample’s scaled derivative have separate downstream uses. Later changes to global storage do not retroactively change those retained values, but may have other unresolved effects.

Calculated addresses may alias each other or global storage. A word store may overlap selector storage or byte-write destinations. No non-aliasing guarantee, alignment guarantee, byte order, or ordinary-memory interpretation is established.

There is no visible allocation, release, reference-count adjustment, ownership transfer, or borrowing discipline. Dependencies may affect resources, but no such contract may be inferred from their invocation.

No explicit validation, error result, retry, rollback, or cleanup path is visible. A fault or nonreturning call can leave earlier writes observable. Atomicity and recovery are **documented_only**.

## Dependencies

### F-674e75a953e8

One direct invocation is visible.

- Explicit inputs: two identical 32-bit values equal to the middle result.
- Position: after obtaining the middle result and before the subsequent middle-result word store.
- Ordinary return-value use: none visible.
- Memory effects, global effects, ownership behavior, failure modes, and register preservation: **documented_only**.

Later reuse of the middle result does not establish the lifetime or validity of memory it might designate.

### Unresolved indirect dependencies

All other represented invocations are indirect. No opaque identities are supplied for their targets.

Targets are calculated from incoming execution state or represented code-relative values. Repeated target-displacement patterns have exceptions; they do not establish dependency equivalence.

The shown argument lists establish visible forwarding expressions, not complete ABI signatures. No indirect dependency may be assumed to preserve globals, memory, resources, or undocumented machine state.

## Acceptance examples

These are conditional static consequences, not observed runtime results.

1. **Distinct global samples.** If the first global read produces hexadecimal 00000400 and the second produces hexadecimal 00000020, the first-sample field argument is 1. The scaled global value is hexadecimal 00000080, so its selector address is 129. Deriving both uses from either single sample would not match the representation.

2. **Equal samples remain two reads.** If both global reads return 8, the saved first value is 8 and the scaled second value is 32. Equal contents do not permit describing the operation as one read.

3. **Unsigned selector.** If a selector is 255, the byte-write destination is address 272, not an address calculated by sign-extending negative one.

4. **Byte truncation.** If a returned word is 255, a byte write using four times that word stores 252, not the full product 1020.

5. **First-result selector address.** If the first result is hexadecimal 12345000, shifting right by 12 yields hexadecimal 00012345. The corresponding selector address is hexadecimal 00012346.

6. **Repeated selector observations.** If one selector read returns 10 and the later read at the same location returns 20, the corresponding destinations are 27 and 37. Reusing the first selector would change behavior.

7. **Direct-call arguments.** If the middle result is 7 and execution reaches F-674e75a953e8, both explicit arguments are 7.

8. **Conditional modular collapse.** Under 32-bit modular arithmetic, any word shifted left by 26 and then multiplied by 1024 becomes zero. The resulting selector address is one.

9. **Explicit zero-derived address.** The later selector operation based on an explicitly zeroed quantity reads address one without needing the modular-collapse assumption.

10. **Completion boundary.** Reaching the final calls does not establish success or a normal return. The supplied prefix ends at an invalid-instruction marker.

## Uncertainties

The following remain **documented_only**:

- Correctness of the represented instruction decoding and linear control flow.
- Actual ABI, implicit incoming state, and secondary-result conventions.
- Identities and contracts of indirect dependencies.
- Exact overflow behavior of signed-looking arithmetic.
- Accessibility, alignment requirements, byte order, and device semantics of calculated addresses.
- Relative evaluation order of separate memory reads within a call’s arguments.
- Global-storage layout, including overlapping symbols.
- Stability of the global between its two consecutive reads, despite no intervening call.
- Concurrency and asynchronous memory changes.
- Ownership, lifetime, and preservation guarantees across dependencies.
- Behavior beyond the final invalid-instruction marker.

An incorrectly represented instruction stream is a possible explanation for the unusual arithmetic and calls, not an established correction.

## Evidence

The supplied static representation supports the four-word interface, visible access widths, ordered memory operations, distinct consecutive global-read expressions, value forwarding, direct invocation of F-674e75a953e8, and invalid-instruction endpoint.

In particular, the first global read is retained independently; the second is multiplied by four. The first feeds the masked/right-shift-by-10 argument, while the second feeds scaled selector and shifted-address calculations. There is no intervening dependency invocation, but no established guarantee that the two reads return equal values.

No runtime tests, dependency inspections, memory-map checks, or independent instruction validation have been performed. Inferred intent remains unresolved; the acceptance examples describe conditional consequences of static evidence only.
