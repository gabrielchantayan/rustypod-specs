<!-- Trusted provenance: F-8650b9fadeec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8650b9fadeec

## Purpose

F-8650b9fadeec performs a long, ordered sequence of dependency calls, address calculations, memory reads, and memory writes. Its visible effects include publishing 32-bit values into calculated memory locations, updating byte-sized fields through byte-derived addresses, modifying shared storage, and finally attempting a call to address zero.

A higher-level purpose—such as initialization, object construction, instruction processing, or device control—is not established. The repeated scaling, masking, and shifting suggest encoded values or references, but that interpretation is **inferred intent**, not a supported interface guarantee.

This contract describes the supplied static representation. It does not establish that the representation faithfully recovers an executable function, that the entire sequence is reachable, or that its final call returns.

## Interface

### Explicit inputs and output

The displayed interface contains two opaque 32-bit inputs and no declared return value.

- **First input:** No visible use.
- **Second input:** Passed as the first explicit argument to the opening dependency call.
- **Return:** No explicit result is returned. The displayed terminal return follows a call to address zero; normal completion therefore depends on that call returning.

The absence of a visible use of the first input does not establish that its incoming machine-register contents are irrelevant to dependencies. The recovered call signatures are incomplete.

### Implicit inputs

The explicit inputs are insufficient to execute the displayed behavior. Additional entry state supplies:

- A base address used for an initial 32-bit read at displacement `0x14`.
- A separate base address used for an initial byte read at displacement `1`.
- A 32-bit value stored during the opening publication sequence.
- A return-link-like value used to calculate the opening call target.
- Shared memory, including a byte at address `1` and two separately named shared storage locations.
- Values made available through secondary result-register state after dependency calls.
- Executable targets for the many indirect calls.

These are preconditions of the displayed representation, not a verified public calling convention.

### Widths and terminology

The visible operations distinguish:

| Quantity | Visible width |
|---|---:|
| Byte reads and writes | 8 bits |
| Halfword reads and writes | 16 bits |
| Ordinary words and most arithmetic values | 32 bits |
| Selected dependency results | 64 bits, split into low and high 32-bit halves |

For readability, this document uses descriptive state labels, not recovered identifiers:

- **Shared-A:** Shared storage explicitly cleared partway through the sequence and read again later.
- **Shared-B:** Shared storage assigned a dependency result near the end, then assigned a composite value at termination.
- **Selector byte:** A byte read from address `1`, or from another calculated location.
- **Selected-byte destination:** The address obtained by zero-extending a freshly read selector byte and adding `0x11`.

A selected-byte destination consequently lies between `0x11` and `0x110`, inclusive. This is an absolute byte-derived address, not an established field within a conventional heap object.

### Preconditions and errors

Every executed memory access requires an accessible location of the displayed width. Every executed call requires a valid target and a compatible machine-state contract.

No explicit validation, bounds check, error status, retry, rollback, or exception handler is visible. Null-like and low-address values are not rejected. Fault behavior, alignment requirements, and invalid-call behavior are **documented_only**.

## Behavior

### Ordering model

The displayed body is straight-line: it contains no explicit conditional branch or loop. Dependencies may nevertheless branch, fail, alter memory, or never return.

The visible order of accesses and calls matters. In particular:

- A dependency call separates many apparently repetitive stores.
- Repeated selector reads are distinct reads.
- A repeated store to a calculated destination is not necessarily a repeated store to the same address.
- Values already read and retained are distinct from values read again after dependencies execute.
- No dependency is assumed to preserve shared storage or memory reachable through calculated addresses.

The following phases describe the significant data transformations and access ordering without assigning semantics to unresolved dependencies.

### 1. Opening reads and word publication

The opening call receives the second explicit input and the result of a 32-bit read from the first implicit base plus `0x14`, logically shifted right by 19.

After several calls, a returned value selects a 32-bit destination at four times that value plus `0x14`. The incoming implicit publication value is stored there. A subsequent dependency receives the four-times-scaled value.

The next publication pattern differs: for another returned value, the function stores four times that value at sixteen times that value plus `0x14`. It then reads a word at the four-times-scaled value plus `0x14`, shifts that word right by 19, and supplies the extracted value alongside the scaled value and an explicit zero to another call.

A later word publication stores this retained scaled value at another return-derived, four-times-scaled destination plus `0x14`.

These are distinct address formulas. The sixteen-times-scaled destination must not be silently replaced with the four-times-scaled destination.

### 2. Packed-value extraction and initial selected-byte writes

A subsequent 32-bit dependency result drives several transformations:

- A left shift by two supplies call arguments and the low eight bits of an initial selected-byte write.
- Masking with `0x3fffffff`, then shifting right by 17, produces one retained extraction.
- The same masked value shifted right by 20 supplies a later selected-byte write.
- The same masked value shifted right by seven supplies another retained value used as an address source.

For the first two byte writes, the selector is read from the second implicit base plus `1`. Calls intervene, so the second selector read may produce a different destination.

One later call receives four copies of a dependency result, the two retained extractions, and an additional fixed call-site-related value. That final argument is not established as user data or an object reference.

A following dependency produces a displayed 64-bit result. Its low and high halves have separate subsequent uses. Before the high half is forwarded, the function reads a selector byte using the retained right-shift-by-seven extraction plus `1`, then clears the corresponding selected-byte destination.

That selector-based clear occurs again after intervening calls.

### 3. Byte, word, and halfword acquisition

The middle sequence alternates dependency calls, selected-byte writes, and payload reads. Significant reads occur in this order:

1. A byte at twice a retained low 32-bit result.
2. A word at a later retained result plus a top-bit-only displacement derived from that same result’s bit 19.
3. A byte at a secondary-result-derived address plus `0x0e`.
4. A byte at a retained result plus a secondary-result-derived offset.
5. A halfword at the sum of a right-shift-by-19 extraction and the previously acquired byte.

The first four payloads are passed to dependencies, often as four equal explicit arguments. The halfword is likewise supplied four times. Equal arguments do not imply that the target is a copy operation.

Between these reads, the function clears selected-byte destinations whose selector addresses are derived from:

- A retained result shifted right by nine, plus `1`.
- A retained result directly, plus `1`.
- Secondary results directly or shifted right by nine, plus `1`.
- A retained low result multiplied by `0x1000`, plus `1`.
- A top-bit-only transformation, plus `1`.
- Address `1` itself.

One call also receives four copies of an address into the current stack frame. The supplied representation does not establish the stack region’s contents, extent, or validity after return.

The displayed halfword load establishes its access width. Its extension convention at the dependency boundary is not fully established by the opaque type.

### 4. Publication after the halfword read

A displayed 64-bit result following the halfword processing is separated into two halves:

- The high half selects a four-times-scaled word destination plus `0x14`.
- The stored value is the retained right-shift-by-19 extraction.
- The low half is passed onward with the high half and an explicit zero.
- The retained extraction is published again to a later dependency-selected word destination.

The retained low half also supplies an address for two writes at displacement `0x0d`. The first stores the low byte of twice that value; after intervening calls, the second stores `4`.

Another result supplies two 32-bit reads: one at four times that result plus `0x44`, and another at a masked, right-shift-by-20 extraction plus `0x44`. These are not equivalent indexing forms.

A later store writes four times the earlier result at a destination scaled by `0x100`, plus `0x14`. The next call receives two copies of a `0x40`-scaled value and a top-bit-only argument.

### 5. Repeated link-like stores and state-field changes

The next region repeatedly alternates two observable word-store forms:

- A retained value stored at four times a newly returned value plus `0x14`.
- Four times a newly returned value stored at sixteen times that value plus `0x14`.

Calling these stores “link-like” describes their repeated relationship only; no graph or ownership model is established.

Stored payloads include scaled values and a masked extraction using `0x03ffffff` followed by a right shift of 13. Calls receive various left-shifted arguments, including shifts by 24 and 22.

The function then:

1. Clears a word at a sixteen-times-scaled base plus `0x24`.
2. Reads a word at that same scaled base plus `0x74`.
3. Uses the read word as a subsequent base.
4. Obtains a result and shifts it right by 20.
5. Stores that extraction at the subsequent base plus `0x24`.
6. Reads another word at the subsequent base plus `0x74` for a dependency argument.
7. Writes a masked extraction from another retained value through a selector read at the subsequent base plus `1`.
8. Later clears a selected-byte destination using a fresh read from that selector location.

The result shifted right by 20 is at most `0xfff`. Its later right shift by 21 is therefore necessarily zero under the displayed unsigned widths. This remains true regardless of intervening shared-memory changes, because the extraction is retained as a value.

### 6. Byte-field reads and further publication

Three byte reads occur at displacement `0x15` from dependency-supplied bases. Dependencies intervene between them. The third byte is zero-extended and published into a 32-bit destination selected through secondary result state.

This region also:

- Repeatedly clears selected-byte destinations using a right-shift-by-20 result scaled by `0x400`.
- Extracts bits using mask `0x1fffffff` and a right shift by 18.
- Publishes that extraction shifted left by 13.
- Writes the low byte of a later result shifted left by two.
- Writes another masked, right-shift-by-20 extraction through a selector read at address `1`.
- Clears destinations selected through a masked extraction shifted right by six.

Further calls and stores publish dependency results and scaled references. A value originating in secondary result state undergoes successive left shifts by three, two, and two; under a 32-bit bit-vector interpretation, its final retained form is a left shift by seven with high bits discarded.

### 7. Shared-A reset and zero-derived operations

Shared-A is explicitly assigned zero. Its exact storage width and overlap with smaller named storage are not established independently of the supplied representation.

Subsequent operations publish zero words, create additional scaled stores, and read words at calculated addresses plus `0x14`.

A later retained unsigned value is explicitly set to zero. Until it is reassigned, this makes several expressions constant:

- Its four-times-scaled selector address plus `1` is address `1`.
- Its masked right shifts are zero.
- A later extraction shifted right by 22 is zero.
- After a right shift by one, a subsequent right shift by 23 is also zero.

Calls still occur between the corresponding accesses. Constant arithmetic does not make fresh reads from address `1` constant.

A separately visible dependency, **F-U001**, is called with two equal arguments from a dependency result. That result is subsequently used for a byte read at displacement `1`. The acquired byte later supplies an address for another byte read at displacement `1`.

No preservation or transformation guarantee for F-U001 is established.

### 8. Late address-field and chained-word operations

The later sequence includes:

- A selected-byte write containing a dependency-returned byte shifted left by two and narrowed back to eight bits.
- A direct byte clear at a scaled address plus `9`.
- A direct byte write at another base plus `9`, using the low byte of that base.
- A word read at that latter base plus `0x34`, supplied to a dependency alongside the halves of a displayed 64-bit result.

Word publication then precedes a chain of word reads involving displacement `0x14`, a right shift by 22, another displacement-`0x14` read, and a right shift by one. Stores and calls are interleaved; these accesses cannot be treated as one uninterrupted traversal.

Shared-A is read again. Its earlier explicit clear does not establish that this later read is zero: intervening dependencies and overlapping memory may have changed it.

However, a retained unsigned 32-bit read shifted right by 23 and then again by 23 is necessarily zero. Several subsequent zero payloads and low-address selector accesses follow from that width constraint.

### 9. Shared-B publication and terminal sequence

Near the end, the function captures the byte at address `1`, clears the selected-byte destination derived from that captured byte, and retains the captured value across further calls.

It then assigns a dependency result to Shared-B before forwarding that result.

Later operations include:

- Reading a word at the captured byte value plus `0x74` and writing its low byte at the captured value plus `1`.
- Repeating displacement-`0x74` word reads and displacement-`1` byte writes on dependency-derived scaled bases.
- Retaining a word read from one such base and publishing it to later word destinations.
- Storing a scaled dependency result at a retained address with its bottom two bits cleared, plus `0x14`.
- Publishing zero to another word destination.
- Reading Shared-A again, then reading a word at that captured value plus `0x14`.
- Writing low-byte and right-shift-by-22-derived portions of the read word through selected-byte destinations.

The final visible actions are ordered:

1. Clear one byte through a pointer supplied by secondary result state.
2. Assign Shared-B a composite 32-bit value: an opaque 24-bit memory-derived component in the upper bits and the final dependency’s byte result in the low eight bits.
3. Clear a halfword at an address obtained by masking a retained result with `0x3fffffff` and shifting right by seven.
4. Attempt a call to address zero with three explicit zero arguments.
5. Return only if that call returns.

## Side effects

The function directly writes 8-, 16-, and 32-bit memory locations and two shared storage locations. Dependencies may cause additional effects not described here.

Ownership is **documented_only**:

- No visible operation establishes allocation, retention, transfer, or release.
- A returned address is not proven newly allocated or caller-owned.
- Publishing a value is not proven to transfer ownership.
- Clearing a byte is not proven to release an object.
- The stack-address argument is not proven safe for a dependency to retain.

Aliasing is permitted by the available evidence. A selected-byte write may affect memory later read as a selector, payload, or wider word. Overlapping shared symbols are explicitly warned about in the supplied material; distinct labels do not prove disjoint storage.

No transactional guarantee exists. If execution stops, prior writes and dependency effects may remain observable.

## Dependencies

Only F-8650b9fadeec has a supplied resolved opaque identity.

**F-U001** is a document-local opaque label for the separately visible two-argument dependency described above. It is not a resolved identity.

All other dependency identities are **documented_only**. The evidence shows numerous indirect calls, mostly using targets calculated from fixed call-site-related values, with several different offsets. The opening target additionally depends on incoming return-link-like state.

The visible calls have varied explicit argument lists and apparent results: bytes, words, split 64-bit values, and secondary register outputs. The evidence does not establish:

- Complete signatures or calling conventions.
- Whether different call sites reach distinct functions.
- Global or register preservation.
- Memory ownership.
- Error or non-return behavior.

The terminal target is address zero. It must not be reclassified as a known service or ordinary successful dependency.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Initial extraction:** If the opening 32-bit read yields `0xffffffff`, the second explicit argument of the opening call is `0x1fff`.

2. **Selected-byte addressing:** If a selector read yields `0xff`, the byte destination is `0x110`. A later selector read yielding `0x02` selects `0x13`, even when both reads use the same source address.

3. **Narrowing:** If a selected-byte payload originates from the 32-bit value `0x41` shifted left by two, the stored byte is `0x04` under the displayed bit-vector interpretation.

4. **Top-bit displacement:** In the word-read stage using a value’s right-shift-by-19 extraction followed by a left shift by 31, the displacement is either zero or `0x80000000`, determined by the original bit 19. It is not the full extracted value multiplied without truncation.

5. **Guaranteed zero:** A 32-bit unsigned result shifted right by 20 and then right by 21 supplies zero, including when the original result is `0xffffffff`.

6. **Fresh shared read:** A dependency changing Shared-A after its explicit clear may affect the later Shared-A read. Assuming that read remains zero is not acceptable.

7. **Preserved access width:** A terminal halfword clear requires a 16-bit write of zero. Substituting a byte clear is not equivalent when the adjacent byte is nonzero.

8. **Terminal control flow:** If address zero is not callable, earlier stores are not rolled back by any visible mechanism. Successful normal return must not be reported.

## Uncertainties

The following remain **documented_only**:

- Machine architecture, endianness, alignment rules, and fault semantics.
- Reliability of the recovered function boundary and call signatures.
- Exact ABI meaning of incoming implicit state and secondary outputs.
- Shared-storage widths and overlap boundaries.
- Dependency identities, effects, ownership rules, and preservation guarantees.
- Whether calculated low-address accesses are meaningful in the intended environment.
- Whether the terminal call can execute or return.

Unsigned shifts support the stated range deductions. Some displayed calculations use signed-looking types; treating all such scaling and left shifts as wrapping 32-bit bit-vector operations is an interpretation, not proof of language-level signed arithmetic semantics.

## Evidence

**Established static evidence:** Two explicit 32-bit inputs; no declared result; ordered calls and memory effects; byte, halfword, word, and split-result widths; masks, shifts, scaled addressing, shared-storage assignments, stack-address exposure, and the terminal zero-address call.

**Inferred intent:** Encoded references, link-like publications, and state preparation suggested by repeated patterns. These interpretations impose no additional behavior.

**Unknown:** Executable fidelity, complete dependency contracts, ownership, fault handling, reachability, and normal completion.

No runtime tests have been performed. No implementation access or external validation was used.
