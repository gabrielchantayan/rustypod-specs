<!-- Trusted provenance: F-0df4af66fe16; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0df4af66fe16

## Purpose

F-0df4af66fe16 performs an entry-state-dependent sequence of unresolved indirect invocations, transforms call-derived values, mutates mixed-width memory locations, and finishes with an unresolved four-argument control transfer.

Its application-level purpose is **documented_only**. The supplied representation does not establish an initializer, formatter, container operation, or protocol implementation. Character-like constants and recurring offsets do not establish those meanings.

This contract distinguishes:

- **Established static evidence:** accesses, transformations, and ordering represented in the supplied material.
- **Inferred intent:** possible interpretations, not acceptance requirements.
- **Unknowns and coverage limits:** details retained as **documented_only**, rather than supplied by assumption.

Established evidence here describes the supplied representation, not independently validated executable behavior.

## Interface

### Entry, implicit inputs, and result

The represented entry has no explicit parameters or return value. Implicit inputs include:

- An incoming control-link value used to select the first indirect target.
- An incoming 32-bit machine-state value used in early shifted arguments.
- Results of unresolved invocations, sometimes represented as paired 32-bit values.
- Additional machine-state outputs not established as conventional return values.
- Mutable memory at absolute and computed addresses.
- A separately stored 32-bit address-like value captured during the late update phase.

There is no established success indicator, error code, output-buffer argument, or returned ownership token.

The final invocation receives four arguments. An apparent return follows, but whether the invocation returns normally, transfers control permanently, or follows a nonstandard convention is **documented_only**.

### Widths and layout

| Access or quantity | Width | Behavioral significance |
|---|---:|---|
| Byte | 8 bits | Stores retain the low eight bits |
| Halfword | 16 bits | Used for constants 0x003a and 0x0065 |
| Word | 32 bits | Used as data, addresses, and traversed links |
| Paired result | 64 bits | Low and high 32-bit portions are separately consumed |

Explicit numeric conversions of byte reads to words are zero-extending.

Recurring offsets include 0x01, 0x04, 0x05, 0x08, 0x09, 0x0d, 0x10, 0x11, 0x14, 0x15, 0x19, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. These are access offsets, not established field names or a complete object layout.

Endianness is unknown. Overlapping byte and word accesses therefore require an environmental byte-order definition before concrete byte-level results can be predicted.

For arithmetic deductions, this document explicitly uses a **conditional 32-bit bit-vector model**: scaling, addition, subtraction, and left shifts retain the low 32 bits; unsigned right shifts insert zeros. This model does not settle the executable meaning of signed overflow or signed shifts.

### Preconditions for reaching the end

Conditional normal completion requires callable indirect targets, returning dependencies where subsequent behavior is expected, and accessible memory at every reached address and width. Alignment, low-memory mapping, and architecture-specific arithmetic requirements must also be satisfied.

No checks establishing these conditions are represented. Zero is not treated as a sentinel before dereferencing.

## Behavior

### Unresolved invocation prefix

A long ordered prefix of indirect invocations precedes the explicit memory-update phase. The first target depends on incoming control state. Later targets are largely formed from fixed control-location-derived values with displacement 0x0ffe.

Arguments include shifted incoming state, earlier results, high or low portions of paired results, masked values, duplicated transformed values, and explicit zeros. Some results remain live across many intervening invocations.

The transformations include:

- Eight-bit left shifts followed by further 32-bit shifts.
- Masks retaining 24, 22, 21, 20, 19, 18, 17, 14, 13, or fewer bits.
- Extraction of bits 10–19 followed by left shifts of five or six.
- Unsigned extraction from full 64-bit results.
- Zero-initialized quantities subjected to further shifts.

Width is significant. Extracting a full paired result above bit 35 yields its high word shifted right by four; extraction above bit 43 yields its high word shifted right by twelve. Two successive 28-bit left shifts of a retained 32-bit quantity produce zero under the conditional model, not a 56-bit intermediate.

No result-dependent retry, explicit failure branch, or early successful return is represented. Invocations without captured results remain effects; zero arguments do not imply inactivity.

The exact prefix invocation catalogue is outside this document’s enumerated coverage. No permission to reorder or remove invocations follows from that coverage limit.

### Values used to describe explicit memory effects

The following are descriptive contract labels, not source identifiers or semantic object types:

| Label | Definition |
|---|---|
| Anchor | A prior call result scaled by 2²⁵ |
| Aligned value | A different prior result shifted left by eight |
| Shifted value | An earlier paired result’s low word shifted left by 25 |
| Selector | An eight-bit-shifted result, masked with 0x7fff, then shifted right by six |
| Small base | Twice the selector |
| Final pair | The last paired result before explicit stores |
| Saved word | The word captured from a byte-derived base plus 0x54 before replacement |
| Displaced value | Four times the saved word, minus 0x96 |
| Computed base | Four times the displaced value |
| Traversed value | A later word reached through a byte-derived base plus 0x74 |
| Doubled destination | Twice the traversed value |
| Scaled link | Four times the traversed value |
| Captured address | The separately stored word loaded before the repeated region |

Arithmetic definitions use the conditional model. Labels do not imply disjoint memory.

The selector ranges over multiples of four from 0 through 508. Consequently, the small base ranges over multiples of eight from 0 through 1016. The aligned and shifted values have zero low bytes under that model.

### Opening mutations and low-address state

The opening updates first place the shifted value in a word at the final pair’s high-word address plus 0x14. A byte at shifted value plus 0x09 then receives the high word’s low byte. Next, a word at 256 times the final pair’s low word plus 0x54 receives the anchor. Only afterward is the byte at shifted value plus 0x09 overwritten with the shifted value’s own low byte—zero under the conditional model.

The anchor’s word at 0x14 is then cleared. Its byte at 0x05 is read before the small base’s word at 0x14 is cleared. That captured byte is zero-extended into the anchor’s word at 0x34. The anchor’s word at 0x14 is subsequently read; the small base’s word at 0x14 is cleared again, followed by two byte stores of the read word’s low byte at small base plus 0x15.

These accesses are ordered observations, not merely final-state assignments. Replacing a read with an earlier written constant requires memory-behavior and overlap guarantees absent here.

A byte read from absolute address 0x05 supplies another base by zero extension. Absolute address 0x01 receives the small base’s low byte. Subsequent effects include:

- A selector byte at the byte-derived base plus 0x15.
- The aligned value’s low byte at anchor plus 0x11, followed later by a clear of that byte.
- An aligned-value word store through the previously read anchor word, at offset 0x54.
- Capture of the saved word from the byte-derived base plus 0x54 before that location is replaced.
- Updates at small base plus 0x34.
- Traversals through offset 0x74, followed by word stores at offset 0x54.
- Repeated stores of the saved word at its own offset 0x24 and repeated low-byte stores through another destination at offset 0x09.

A word at absolute address 0x54 receives twice the saved word. A byte at absolute address 0x15 also receives a computed low byte.

Further updates use doubled, fourfold, and eightfold values as both addresses and data. They include word accesses at offsets 0x04, 0x14, 0x34, 0x44, 0x54, 0x64, and 0x74, and byte accesses at 0x05, 0x09, 0x0d, and 0x15. Byte-derived bases are not widened object pointers by assumption. Repeated reads through matching offsets are not guaranteed to return matching values.

No list, tree, reference-count, or ownership semantics are established for these linked updates.

### Constant writes and address capture

The constant 0x3a is stored as:

- A byte at four times the traversed value.
- A halfword at eight times that value.
- A word at scaled link plus 0x20.

These are separate effects with different widths.

At computed base plus 0x20, successive word stores contain 0x20, 0x2f, a zero-extended byte read from computed base plus 0x04, and finally 0x20 again. The byte read follows the first two stores; it is not an entry-time snapshot.

**After the final 0x20 store at computed base plus 0x20, the separately stored address word is read and captured.** That read precedes both the word store of 0x20 at absolute address 0x60 and the first scaled-destination store.

The captured address, not a refreshed global value, supplies the repeated region’s address and low-byte data. Moving its capture earlier, delaying it past the absolute 0x60 write, or reloading it for each group is not behaviorally equivalent without additional guarantees.

### Repeated update region

The first group writes the scaled link as a word at eight times the saved word plus 0x54. It then writes, in order:

- The computed base’s low byte at captured address plus 0x19.
- The captured address’s low byte at computed base plus 0x11.
- Eight times the saved word as a word at captured address plus 0x34.

Following groups instead use eight times the displaced value for both the scaled destination and the final stored word. Within each group, the scaled-destination word store precedes the two byte stores, which precede the word store at captured address plus 0x34.

The represented quantities do not advance a cursor. Stores may overlap each other, the captured address’s storage, or later read locations. Such overlap does not trigger a reload of the captured address.

Exact group multiplicity is **documented_only** in this document: no numerical count is asserted. This is a coverage limit, not evidence that identical-looking writes may be collapsed. Their observability depends on memory mapping and observation behavior.

### Post-repetition capture order

The transition out of the repeated region has this explicit ordering:

1. A word is captured from absolute address 0xa9.
2. A word is captured from scaled link plus 0x74.
3. A byte at absolute address 0x3a receives the low-byte result of eight times the displaced value plus 0x30.
4. A word is captured from doubled destination plus 0x14.
5. A word is captured from computed base plus 0x04.

The 0x3a write is therefore **between** the first two captures and the final two captures. If it overlaps either later word read, that read observes memory after the byte write, subject to the environment’s memory semantics. Treating all four words as a common pre-mutation snapshot is not accepted.

Subsequent stores place the computed base’s low byte at doubled destination plus 0x0d, literal byte 0x6c at computed base plus 0x08, and the captured address’s low byte at computed base plus 0x0d and plus 0x09. A word at doubled destination plus 0x54 receives the doubled destination itself.

### Live reads of absolute address 0x67

Absolute address 0x67 first receives the low byte of the captured word from 0xa9. The next relevant operation is a **word read from doubled destination plus 0x44**. Only after that read is absolute address 0x67 read as a byte for the store to absolute address 0x21.

Thus, the 0x21 store consumes a live byte read, not directly the earlier captured word’s low byte.

Further mutations use the derived word as data at computed base plus 0x10 and captured address plus 0x54. A byte at doubled destination plus 0x01 receives a signed-right-shift-derived byte. The shift count is the low eight bits of the computed base; no five-bit count reduction is established. Oversized counts and signed-shift semantics remain unresolved.

The word at captured address plus 0x54 is then replaced with the computed base. A word read from computed base plus 0x40 supplies a later word store at computed base plus 0x10. Other intervening stores update captured address plus 0x74 and plus 0x44.

There are then **two separate additional byte reads of absolute address 0x67**:

- The first supplies a byte store at computed base plus 0x09.
- Afterward, a byte is stored at doubled destination plus 0x09, the captured 0xa9 word is stored at computed base plus 0x44, and a word is read from computed base plus 0x54.
- Only then does another byte read of 0x67 supply the second store at computed base plus 0x09.

All three reads of 0x67—the one supplying 0x21 and the two supplying computed base plus 0x09—are distinct live observations. Neither agreement with the initial byte write nor agreement with each other is guaranteed. Computed writes may alias 0x67, and ordinary stable-memory behavior is not established.

### Final mutations and transfer

The word read from computed base plus 0x54 becomes another destination. The low byte of the previously captured scaled-link word is written at that destination plus 0x0d and at doubled destination plus 0x11.

The captured doubled-destination word from offset 0x14 is restored at doubled destination plus 0x54. A fresh word read from computed base plus 0x14 occurs before subsequent byte and word updates using the destination obtained through computed base plus 0x54.

Near the end, a word read from doubled destination plus 0x04 supplies a traversal base. A halfword 0x0065 is stored at twice that base before its word at offset 0x64 is read. The traversed destination receives a byte derived from the fresh computed-base word and a word value of 100 at offset 0x54.

A word is then read from traversal base plus 0x74, followed by a word read from doubled destination plus 0x64. After another byte update at doubled destination plus 0x09, a word store through the latter destination at offset 0x54 uses the captured offset-0x64 word. The final explicit word store places that destination word at twice the offset-0x64 word’s value.

The final unresolved invocation receives, in order:

1. 0x72.
2. The word captured from traversal base plus 0x64.
3. The earlier word captured from doubled destination plus 0x14, after the 0x3a write.
4. The word captured from traversal base plus 0x74.

No explicit memory access follows that invocation.

## Side effects

Direct absolute accesses include byte reads or writes at 0x01, 0x05, 0x15, 0x21, 0x3a, and 0x67; word writes at 0x54 and 0x60; and a word read at 0xa9. Computed accesses may overlap these locations.

There is no established transaction, atomicity guarantee, synchronization, rollback, or non-aliasing invariant. Represented ordering does not itself establish hardware barriers or cross-thread visibility.

Ownership and lifetime are unknown. No explicit allocation or release primitive is identifiable in the memory-update phase, but dependencies may allocate, retain, transfer, mutate, or release resources. They are not assumed to preserve globals.

No bounds checks, null checks, error reports, cleanup path, or recovery path are represented. A fault or nonreturning dependency can leave earlier effects in place.

## Dependencies

| Opaque ID | Role | Established boundary |
|---|---|---|
| F-U01 | Ordered family of unresolved prefix invocations | Partially represented argument transformations and result consumption; identities, signatures, and effects unknown |
| F-U02 | Final unresolved invocation | Four argument positions established; target and return behavior unknown |

These labels do not establish that F-U01 is one function or that F-U02 differs from every prefix target. Only F-0df4af66fe16 has a supplied resolved identity.

## Acceptance examples

These are conditional static obligations, not executed tests.

- **Paired extraction:** If a pair’s high word is 0x12345678, unsigned extraction above bit 35 yields 0x01234567.
- **Retained width:** Two successive 28-bit left shifts of a 32-bit quantity yield zero under the conditional model.
- **Small-base boundary:** A relevant result of 0x7f produces selector 508 and small base 1016; 0x80 produces zero. Neither proves address validity.
- **Overwritten byte:** A final-pair high word ending in 0xab causes an initial 0xab byte store at shifted value plus 0x09, followed by zero under the conditional model. Both remain represented effects.
- **Address capture timing:** If the final computed-base-plus-0x20 store changes the address word’s storage through aliasing, its captured value reflects that store. A later aliasing 0x60 write does not refresh the capture.
- **Post-repetition interleaving:** If absolute address 0x3a lies within either later captured word, acceptance uses the post-write word value, with byte placement determined by endianness.
- **Live 0x67 values:** If the three byte reads return 0x11, 0x22, and 0x33, their consuming stores receive those respective values. An intervening read-induced or aliasing effect must not be erased by caching.
- **Invalid access:** An unusable target or inaccessible address has no specified successful completion result and no represented rollback.

## Uncertainties

The following remain **documented_only**:

- Application purpose, semantic types, and intended object invariants.
- Authenticity and completeness of the represented control flow.
- Dependency identities, calling conventions, and additional machine-state outputs.
- Endianness, alignment, low-memory mapping, and access-induced effects.
- Executable signed-overflow and signed-shift semantics.
- Exact prefix catalogue and repeated-group multiplicity within this document’s coverage.
- Relative evaluation order of multiple reads embedded within a single represented expression where not separately established.
- Ownership, lifetime, concurrency, and final-transfer return behavior.

Packed-value construction and structured-state updates are possible inferred intent only. They do not justify replacing widths, live reads, captured values, or interleaved effects with higher-level assumptions.

## Evidence

The contract derives solely from static examination of the supplied representation. Embedded comments, prior prose, annotations, and reviewer statements are not independent authority.

The representation establishes the specific capture timing before the 0x60 write, the 0x3a write between post-repetition word captures, and three distinct later reads of absolute address 0x67. These obligations follow from the represented accesses themselves.

No tools, implementation access, dependency inspection, independent binary validation, or runtime tests have been used. Acceptance examples are deductions under stated conditions, not observed execution results.
