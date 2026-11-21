<!-- Trusted provenance: F-24caae9dbe37; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-24caae9dbe37

## Purpose

F-24caae9dbe37 performs an ordered sequence of external calls and direct memory accesses using implicit entry state, call results, additional register-derived values, and shared storage.

Established effects include:

- Reading byte selectors and using their values to select low-address byte destinations.
- Reading 8-, 16-, and 32-bit quantities from computed addresses.
- Writing words to destinations formed with several different scale factors and field offsets.
- Updating two shared words, designated **G-01** and **G-02** in this document.
- Passing a stack-relative address to an external dependency.
- Clearing a final halfword and then calling address zero.

No application-level purpose is established. Allocation, initialization, serialization, device control, and service dispatch are possible interpretations, not supported behavioral guarantees.

This contract describes the supplied static representation. It does not establish executable reachability, a portable calling convention, or the accuracy of recovered arithmetic and function boundaries.

## Interface

### Inputs and outputs

The represented interface declares no parameters and no return value. Nevertheless, behavior depends on:

- An incoming continuation-related value that selects the initial call target.
- An incoming base-like value used for the first two selector reads.
- Another incoming machine value forwarded to an early call.
- Primary and combined external-call results.
- Additional register-derived values whose exact production boundaries are unresolved.
- Absolute and computed-address memory contents.
- Shared storage sampled at specific points between calls.

There is no established conventional invocation interface. A caller cannot assume that supplying no language-level arguments supplies all required state.

There is no explicit success indicator or error return. Normal completion requires the terminal call to return.

### Widths and arithmetic

| Quantity or access | Represented width |
|---|---:|
| Selector and byte field | 8 bits |
| Halfword sample and final clear | 16 bits |
| Word, ordinary address-like value, call-result component | 32 bits |
| Combined call result | 64 bits |
| Low and high components of a combined result | 32 bits each |
| Shared fragment used in final publication | 24 bits in the concatenation |

An explicitly unsigned byte sample has range 0–255. Byte writes retain the low eight bits of the supplied quantity.

Address examples and scaling identities below use a **conditional 32-bit bit-vector interpretation**: arithmetic retains its low 32 bits. This interpretation does not resolve signed overflow or signed-left-shift behavior in the supplied representation.

Unsigned right shifts insert zeros. Masks apply before their associated shifts. They must not be replaced by superficially similar field extractions.

Splitting a represented 64-bit result into two words does not prove that the dependency returns a language-level 64-bit integer. It may represent multiple machine-register results.

### Preconditions

Completing the visible sequence requires:

- Every encountered external target to be callable and to return where later operations depend on continuation.
- Every direct access to be supported at its represented width.
- An environment supporting the required implicit machine state and shared-storage layout.

No explicit range, alignment, null-address, overflow, or dependency-success checks are represented. These are conditional execution requirements, not validated inputs.

## Behavior

### Ordering and selector semantics

The visible body is straight-line: no explicit branch or loop is represented. External calls remain observable boundaries, including calls with no displayed arguments.

Direct accesses must retain their ordering relative to intervening calls. Repeated reads are not interchangeable with cached values, and calls are not assumed to preserve shared memory.

A recurring selector operation reads an unsigned byte at a computed address and writes a byte at **17 plus that sampled byte**. Thus:

- The destination is between 17 and 272 inclusive.
- Addition of 17 is not an eight-bit wraparound operation.
- The selector access is one byte, not a pointer-sized read.
- The selector address and resulting destination are separate addresses.
- Each represented selector read obtains a fresh sample.

Direct accesses may alias one another or shared storage. No disjointness guarantee is established.

### Entry-derived processing

The initial call target depends on incoming continuation-related state. A subsequent call supplies an unsigned word used to derive:

- Its left shift by two.
- Its low 30 bits shifted right by 17.
- Its low 30 bits shifted right by 20.
- Its low 30 bits shifted right by seven.

The low byte of the left-shifted value is written using a selector read at the incoming base plus one. Following intervening calls, that selector address is read again, and the low byte of the masked, right-shifted-by-20 value is written to the newly selected destination.

Calls in this region receive scaled results, duplicated arguments, and an incoming machine value. One call receives four copies of one result, the two masked extractions by 17 and seven, and an address-derived control value.

A combined result supplies a new low-word working value and a high component. Before a subsequent call receives the high component twice and an earlier result, a fresh selector based on the earlier right-shifted-by-seven quantity selects a byte to clear.

### Early sampling and selected clears

Further calls separate repeated selected-byte clears. Selector addresses use saved values, right-shifted values, absolute address one, and additional register-derived values.

Distinct data samples include:

- An unsigned byte at twice a saved word.
- A word at a saved word plus a high-bit contribution.
- A byte at an additional-result base plus 14.
- An unsigned byte at a saved base plus a register-derived displacement.
- A halfword at the sum of a right-shifted word and the previously sampled unsigned byte.

The high-bit contribution is formed by an unsigned right shift by 19 followed by a left shift by 31. Under the conditional 32-bit interpretation, it is zero or `0x80000000`, determined by original bit 19.

Several sampled quantities are forwarded four times in one argument list. Where a byte or halfword sample has no established signedness, its extension into the dependency’s argument convention remains unresolved.

One dependency receives the same stack-relative address in four argument positions. The address is represented as 84 bytes above a stack reference. The visible body does not establish initialization of that region, its extent, permitted access, or whether retention is safe.

### Initial scaled stores and byte fields

After the halfword sample and another combined result:

1. A saved unsigned right-shifted quantity is stored as a word at four times the high component plus 20.
2. Calls intervene.
3. The same saved quantity is stored at four times a different call result plus 20.

The combined result’s low component is retained as a base for two byte writes at offset 13. The first stores the low byte of twice that base. The second stores four after intervening calls.

Two separate word reads use:

- Four times a saved unsigned result, plus 68.
- The low 30 bits of that result shifted right by 20, plus 68.

The second address formula has no implicit multiplication by four.

Subsequent stores include a destination at 256 times a call result plus 20, receiving four times an earlier unsigned result, and a destination at four times another result plus 20, receiving a masked extraction.

A recurring group stores four times a result at sixteen times that same result plus 20. A later call-selected destination at four times another result plus 20 receives the saved scaled value. These formulas do not establish allocation, copying, linking, or reference-counting semantics.

### Offset-36 and offset-116 processing

A later group clears a word at offset 36 from a sixteen-times-scaled result, then reads a word at offset 116 from that same scaled base. The sampled word becomes a new base-like quantity.

After calls:

- A call-produced unsigned word is shifted right by 20.
- That extraction is written at offset 36 from the new base.
- A word at offset 116 from the new base is passed to a dependency.
- A selected byte receives the low byte of another masked extraction.
- A later selector reload selects a byte to clear.

The extraction already shifted right by 20 is subsequently shifted right by 21 for arguments. For a 32-bit unsigned input, this further extraction is necessarily zero.

Three unsigned bytes are sampled separately at offset 21 from additional register-derived bases, with substantial intervening call sequences. The third sample is later stored, zero-extended, as a word at a scaled destination.

Another additional result is masked to 29 bits and shifted right by 18. Later left shifts and selector addresses use this extraction without establishing its application-level meaning.

### Shared reset and bounded values

G-01 is explicitly written with a zero word in the middle portion. Later reads follow external calls and therefore need not observe zero. No preservation guarantee is available for those dependencies.

Some preserved working values, unlike shared storage, are explicitly set to zero and remain zero in the represented dataflow. Their uses produce selector reads at absolute address one and zero-valued arguments.

A separately designated dependency receives two equal arguments. The value supplied in both positions comes from the preceding call. That preceding result is retained for a later unsigned byte read at result plus one; the separately designated dependency’s own return value is not visibly assigned to that retained quantity.

The sampled unsigned byte later supplies an address-like value for another byte read at sample plus one. That second address lies between one and 256 inclusive.

Other unsigned bounds establish necessarily-zero values:

- Masking to 30 bits and shifting right by 19 produces at most 2047.
- Applying that masked extraction by 19 again produces zero.
- Shifting the bounded result right by 22 produces zero.
- Shifting it right by one and then by 23 also produces zero.

These deductions apply to preserved working values, not to fresh memory samples after calls.

### Late byte fields and word-address chains

A byte is cleared at a destination formed from a previously four-times-scaled result multiplied by 8192, plus nine.

Later, another base-like result receives its own low byte at offset nine. A word at offset 52 from that base is passed with components of a combined result.

Subsequent word reads use distinct address chains:

- An additional-result base plus 20.
- A previously sampled word shifted right by 22, plus 20.
- A saved sampled word plus 20.
- A masked and shifted additional result plus 20.

Scaled stores and calls separate these samples. They must not be collapsed into one stable object traversal.

An earlier G-01 snapshot is shifted right by 23, then that extraction is shifted right by 23 again. The second result is necessarily zero for an unsigned 32-bit snapshot, so a selector based on it reads address one. This earlier snapshot is distinct from the final G-01 snapshot described below.

### G-02 publication and repeated field samples

A call-produced word is written to G-02 before being passed to another dependency.

During this region, an unsigned byte from address one is saved as a base-like value. Its sampling point precedes the G-02 publication and later calls. A subsequent operation reads a full word at that saved byte value plus 116, then writes the word’s low byte at the saved byte value plus one.

Related operations use four-times-scaled call results as bases. They read words at offset 116 and write low bytes at offset one. One such base is revisited after calls: its second word sample is a fresh read, not reuse of the first.

Another word sampled at offset 116 is retained both for its low-byte write and for later word stores at scaled destinations.

Further effects include:

- A scaled call result stored at a saved value masked with `0xfffffffc`, plus 20.
- A zero word stored at another scaled destination.
- A combined result whose low component is shifted left by two and stored at four times its high component plus 20.

### Final G-01 snapshot: timing and lifetime

Immediately after that combined-result-derived store, **G-01 is read once into a preserved 32-bit snapshot**.

The following order is established:

1. The combined result’s scaled low component has already been stored through the high-component-selected destination.
2. G-01 is sampled.
3. Five external calls occur. The first receives the scaled low component, the high component, and two further copies of the scaled low component. The following four calls display no arguments.
4. The last of those calls supplies another destination selector.
5. The saved scaled low component is stored as a word at four times that selector plus 20.
6. Only then is a word read at **the saved G-01 snapshot plus 20**.

This final dereference does **not** reload G-01. If any intervening call changes G-01, the address still uses the earlier snapshot. Likewise, if the intervening scaled store aliases G-01, that write does not replace the snapshot.

The pointed-to word itself is sampled only at the later dereference. Intervening calls or the preceding store may therefore change the word that is ultimately read at the saved address.

After further calls, the sampled word supplies selected-byte writes: first its low byte, then the low byte of its unsigned right shift by 22. These writes use different selector-address formulas.

### Terminal sequence

The final calls produce a byte-sized result and an additional address-like result. The visible direct effects then occur in this order:

1. A zero byte is written through the additional address-like result.
2. G-02 receives a word formed by concatenating a designated 24-bit shared fragment above the byte-sized result.
3. A zero halfword is written at the address formed by masking a saved low-word result to 30 bits and shifting right by seven.
4. Address zero is called with three zero arguments.

The concatenation’s upper 24 bits are not established to be G-02’s previous upper bits. The fragment’s physical access width and sampling details remain unresolved.

The final halfword address is at most `0x007fffff`. The calculation does not ensure two-byte alignment.

The represented return follows the call to address zero. Neither successful return nor a guaranteed fault at that target is established.

## Side effects

Direct effects comprise the described byte, halfword, and word accesses, including the G-01 reset and both G-02 publications. Dependencies may introduce additional effects.

An overlap warning establishes that some shared symbols overlap smaller symbols at the same address. Shared assignments must not be modeled as updates to necessarily disjoint abstract variables.

No ownership transfer is established:

- Returned addresses are not proven newly allocated.
- Forwarded addresses are not proven borrowed, consumed, or released.
- Clearing storage does not establish deallocation.
- The exposed stack-relative address is not proven safe to retain.
- No explicit release operation is identifiable.

There is no visible rollback, cleanup handler, explicit failure result, or recovery branch. If a call fails to return or an access faults, earlier effects have already occurred unless the execution environment provides additional semantics.

## Dependencies

Only F-24caae9dbe37 has an established supplied function identity. The following opaque IDs distinguish unresolved dependency roles within this contract:

| Opaque ID | Role |
|---|---|
| F-U001 | Initial target selected from incoming continuation-related state |
| F-U002 | Multiple later targets selected from embedded control locations and offsets |
| F-U003 | Separately designated dependency receiving two equal arguments |
| F-U004 | Terminal target at address zero |

F-U002 denotes a family of unresolved targets, not one known dependency. Repeated offsets at different control locations do not establish identical targets.

Dependency identities, argument conventions, register preservation, memory effects, ownership behavior, and termination are **documented_only**. Calls displaying no arguments are not proven independent of implicit machine state.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Selector range.** A selector sample of 255 selects address 272 for the byte write, not address 16.

2. **Selector reload.** If intervening calls change a selector from two to seven, surrounding writes can target addresses 19 and 24. Reusing the first sample is not equivalent.

3. **Byte truncation.** An early unsigned result of `0x00000041`, shifted left by two, yields `0x00000104`; its byte write stores four.

4. **Mask before shift.** For `0xffffffff`, masking to 30 bits and shifting right by 20 produces 1023. Its byte write stores 255.

5. **Shared reset.** If a dependency changes G-01 after its zero-word reset, a subsequent G-01 sample observes the storage at that later sampling point, not an assumed persistent zero.

6. **Final snapshot survives calls.** Suppose the final G-01 sample is `0x1000`, and an intervening dependency changes G-01 to `0x2000`. After the intervening scaled store, the specified word read is still at `0x1014`, not `0x2014`.

7. **Snapshot does not freeze pointed-to memory.** Under the preceding conditions, if an intervening effect changes the word at `0x1014`, the later dereference samples the changed word. Only the base snapshot was preserved.

8. **Full-width sample and repetition.** Reading a word at base plus 116 before writing its low byte requires the word-width access. Revisiting that base after calls requires another word read, even if an earlier low byte is available.

9. **Final publication.** If the final byte result is `0x5a`, G-02’s low byte becomes `0x5a`; its upper 24 bits come from the designated shared fragment.

10. **Terminal failure.** If address zero is not callable, normal completion is not required. Failure there does not erase preceding writes.

## Uncertainties

The following remain **documented_only**:

- Architecture, byte order, alignment rules, and address-space mapping.
- Accuracy of represented function boundaries and call targets.
- Complete calling conventions and production points for additional register results.
- Signed overflow and shift behavior where arithmetic is signed-looking.
- Exact shared-storage addresses and overlap relationships.
- Physical access width and sampling mechanics of the final 24-bit fragment.
- Whether low addresses identify ordinary memory, devices, aliases, vectors, or invalid storage.
- Whether dependencies allocate, release, retain pointers, change shared storage, fault, or fail to return.
- Whether address zero is intentionally callable.

Repeated scales and offsets suggest structured state processing. That is inferred intent only; no object schema, protocol, or ownership model is established.

## Evidence

Established static evidence comes from the supplied representation: explicit access widths, masks, shifts, call arguments, dataflow, ordered shared accesses, overlap warning, and terminal call.

The final G-01 snapshot distinction follows from an explicit shared read before the intervening call sequence, preservation of that sampled value, and its later use as the base of the word read after another scaled store.

Bounded ranges and necessarily-zero extractions are mathematical consequences of the represented unsigned widths, conditional on their accuracy.

The publishing system’s trusted, host-generated envelope supplies snapshot provenance and a snapshot hash. That supplied provenance is distinct from independent verification by this document’s author; no hash is reproduced here.

No runtime tests, executable inspection, dependency inspection, or independent provenance verification have been performed.
