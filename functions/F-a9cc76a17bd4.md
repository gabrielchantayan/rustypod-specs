<!-- Trusted provenance: F-a9cc76a17bd4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a9cc76a17bd4

## Purpose

F-a9cc76a17bd4 performs an ordered stream of indirect invocations, transforms dependency results, reads and writes computed and fixed-location storage, and finishes with a four-operand indirect invocation.

No application-level purpose is established. Recurring offsets, address scales, and character-like constants do not establish object construction, initialization, serialization, or text output.

This contract distinguishes:

- **Established static evidence:** represented calls, access widths, value dependencies, and ordering.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **documented_only:** behavior insufficiently resolved or enumerated here to prescribe an executable replacement.

Every effect is conditional on execution reaching it. No runtime tests have been performed.

## Interface

### Entry and completion

The represented interface has no explicit parameters and no explicit return value. Its implicit inputs include:

- An incoming control-link value used to derive the first indirect target.
- Primary and secondary dependency-result channels.
- A global supplying an address-bearing value, called the **publication-base global**.
- Fixed-location storage, including an input associated with address 0x05.
- Bytes and words reached through computed addresses.
- Machine state consumed by unresolved dependencies.

The publication-base global is acquired late in the direct-memory phase, not at entry. Its retained value is called the **publication base**.

Normal completion requires all preceding calls and memory accesses to complete and the final indirect target to return. There is no explicit status result.

### Widths, arithmetic, and layout

| Value or operation | Established representation |
|---|---|
| Explicit byte access | 8 bits |
| Explicit halfword store | 16 bits |
| Explicit word access | 32 bits |
| Address-bearing arithmetic | 32-bit operands and intermediates |
| Paired dependency result | Two 32-bit components extracted from a 64-bit representation |
| Explicit computed byte load widened to a word | Zero-extension |
| Fixed-location input associated with 0x05 | Converted to an unsigned 32-bit value; source width unresolved |

Byte stores retain the low eight bits; halfword stores retain the low sixteen bits. The paired representation does not establish a language-level 64-bit callee return type.

Unsigned right shifts are zero-filling. Signed shifts and signed multiplication also appear. Signed overflow, right shifting negative values, and unsupported shift counts do not have a sufficiently established execution interpretation.

Examples involving wraparound are conditional on 32-bit bit-vector semantics. Wider intermediates must not be presumed.

Offsets describe byte-address relationships, not verified structure layouts. Byte order, alignment requirements, and the interpretation of overlapping multi-byte accesses remain unresolved.

### Preconditions

No explicit validation is represented. Successful execution requires callable targets, compatible result channels, accessible storage, suitable alignment, and validity of retained address-bearing values when used.

These are environmental requirements, not checks performed by this function. Null, low, misaligned, or otherwise invalid addresses have no represented rejection path.

## Behavior

### Indirect invocation phase

The first target depends on the incoming control-link value plus 4094 bytes. Subsequent targets generally depend on represented control-location-derived values plus the same displacement.

Invocation positions must remain distinct; they are not established as calls to one shared dependency. The stream includes calls with no explicit operands and calls with one through four operands. A call without explicit operands may still consume implicit state.

Observable operand relationships include:

- Forwarding a primary result after an eight-bit left shift.
- Forwarding low and high components of paired results separately.
- Consuming secondary result channels independently.
- Reusing retained results after many intervening calls.
- Supplying explicit zero operands, including calls with all explicit operands zero.

An early three-operand forwarding supplies the paired result’s low component shifted left by eight, its unchanged high component, and its high component shifted left by six, in that order.

Later transformations include shifts through 31 bits, masks, and extractions above bit 31 of a paired representation. Established bounds include:

| Transformation | Result range |
|---|---|
| Retain 20 low bits, then unsigned right shift by ten | 0–1023 |
| Retain 24 low bits, then unsigned right shift by twenty | 0–15 |
| Retain 24 low bits, then unsigned right shift by eight | 0–65,535 |
| Retain 17 low bits, then unsigned right shift by four | 0–8191 |

These bounds do not establish protocol fields or array indices. A left shift followed by a right shift need not recover the original value.

The complete invocation-position trace and dependency contracts remain **documented_only**. Calls are not removable merely because their results are unused or their explicit operands are zero.

### First direct-memory boundary

The following documentary roles identify retained values:

- **Shifted address:** a retained dependency-result component shifted left by 25.
- **Scaled address:** another retained result multiplied by 33,554,432.
- **Compact index:** a primary result shifted left by eight, masked to fifteen bits, then unsigned right shifted by six.
- **Boundary pair:** the last paired result before direct memory effects begin.

Under 32-bit bit-vector semantics, the compact index is a multiple of four between 0 and 508.

The boundary pair’s high component supplies a word destination base at offset 0x14; that destination receives the shifted address. A byte at shifted-address offset 0x09 then receives the high component’s low byte.

Before that byte is overwritten with the shifted address’s low byte, a word at 256 times the boundary pair’s low component plus 0x54 receives the scaled address.

The function clears the word at scaled-address offset 0x14, reads a byte at scaled-address offset 0x05, clears the word at twice the compact index plus 0x14, and stores the zero-extended byte at scaled-address offset 0x34.

The cleared word at scaled-address offset 0x14 is then read back. Another clear at twice the compact index plus 0x14 precedes two separate byte stores of the read-back value’s low byte at twice the compact index plus 0x15.

The read-back and duplicate stores remain distinct effects. Aliasing can make cached substitution or write coalescing incorrect.

### Address-derived storage

After those two byte stores, the fixed-location input associated with 0x05 is acquired and converted to an unsigned word. It supplies an address base for accesses including offsets 0x15 and 0x54.

Its source access width is **documented_only**. The conversion does not establish a byte load, truncation to eight bits, or a maximum value of 255.

In contrast, explicitly byte-typed computed loads at offset 0x05 do establish zero-extension. Such a byte can become the base of a word load at offset 0x74, permitting a low-address word access without an established array interpretation.

A word acquired at offset 0x54 through the fixed-location input’s base is retained as the **scale seed**. The function overwrites that location after acquiring the seed; later uses of the seed do not reload it.

The intervening region:

- Uses loaded words as address bases and scales them by two, four, and eight.
- Writes retained words, scaled values, and address fragments.
- Performs self-referential word stores and duplicate stores.
- Reads computed fields after earlier writes, including fresh computed byte loads.
- Writes bytes at fixed addresses 0x01 and 0x15 and a word at fixed address 0x54.

Word offsets include 0x04, 0x14, 0x44, 0x54, 0x64, and 0x74. Byte offsets include 0x05, 0x09, 0x0D, 0x11, and 0x15.

Different documentary bases do not establish disjoint storage. The complete access trace within this region remains **documented_only**; no independence or reorderability follows from this grouping.

### Constants and publication-base acquisition

A later region writes 0x3A using separate byte, halfword, and word stores at differently scaled destinations.

For subsequent effects:

- **Adjusted scale** is four times the scale seed minus 0x96.
- **Retained base** is four times the adjusted scale.
- **Byte-derived word** is a later word acquired at offset 0x74 through an explicitly loaded byte.
- **Doubled base** is twice that byte-derived word.
- **Publication payload** is four times that same byte-derived word.

These relationships retain 32-bit arithmetic limitations; they do not establish object types.

The word at retained-base offset 0x20 receives four successive values: 0x20, 0x2F, a zero-extended byte acquired from retained-base offset 0x04, and 0x20 again. The byte acquisition occurs after the 0x2F store and before the third word store.

**After the fourth word store, the function acquires the publication-base global. The fixed word store of 0x20 at address 0x60 follows that acquisition.**

Earlier overlapping writes can affect the acquired global value. The later write at 0x60 cannot retroactively alter it. No subsequent reload of the global is represented; later effects use the retained publication base even if its originating storage changes.

### Repeated publication

Each publication has four ordered effects:

1. A word at a scaled destination plus 0x54 receives the publication payload.
2. A byte at publication-base offset 0x19 receives the retained base’s low byte.
3. A byte at retained-base offset 0x11 receives the publication base’s low byte.
4. A word at publication-base offset 0x34 receives the scaled destination base.

The initial publication uses eight times the scale seed as its scaled destination base. Subsequent publications use eight times the adjusted scale.

These later publications repeat without a progressing address or collection index. Each write remains a separate represented effect. Their exact count is **documented_only** here; no buffer length or record count is asserted.

### Final acquisition boundary

After the repeated publications, the function acquires retained fields across an intervening fixed-address byte store. The ordering is part of the contract:

| Relative position | Observable effect |
|---|---|
| First | Acquire the 32-bit **fixed word** from address 0xA9 |
| Second | Acquire the 32-bit **payload field** from publication-payload offset 0x74 |
| Third | Write the byte at fixed address 0x3A |
| Fourth | Acquire the 32-bit **earlier field** from doubled-base offset 0x14 |
| Fifth | Acquire the 32-bit **retained-base field** from retained-base offset 0x04 |

The 0x3A store receives the low-byte result corresponding to eight times the adjusted scale plus 0x30.

**The last two field acquisitions occur after the 0x3A store and must observe any overlap with it.** Acquiring all four words before that store is not equivalent.

Conversely, the fixed word and payload field are retained from before the store. Overlap does not turn their later uses into fresh reads.

The earlier field supplies both a later restoration at doubled-base offset 0x54 and the final invocation’s third operand. Both uses refer to the value acquired after the 0x3A store.

### Final updates and fresh reads

Following that acquisition boundary, computed byte writes install retained address fragments and 0x6C. A word at doubled-base offset 0x54 receives the doubled base.

Fixed address 0x67 then receives the fixed word’s low byte. Three later uses of 0x67 are distinct fresh reads:

- The first follows the word load at doubled-base offset 0x44 and supplies the byte written to fixed address 0x21.
- The second follows intervening computed writes, the word load at retained-base offset 0x40, publication-base updates, and word stores at retained-base offset 0x10 and publication-base offset 0x44. It supplies the first later byte store at retained-base offset 0x09.
- The third follows that byte store, a byte store at doubled-base offset 0x09, the fixed-word store at retained-base offset 0x44, and a word load at retained-base offset 0x54. It supplies the second later byte store at retained-base offset 0x09.

Neither the initially installed byte nor one shared cached read substitutes for these acquisitions. Intervening computed writes may overlap 0x67.

Other established relationships include:

- The word acquired at doubled-base offset 0x44 supplies field writes and a right-shifted byte at doubled-base offset 0x01. The shift count is the retained base’s low eight bits.
- Publication-base offset 0x54 first receives that acquired word, then the retained base.
- The word acquired at retained-base offset 0x40 is stored at retained-base offset 0x10.
- The retained-base field acquired after the 0x3A store is stored at publication-base offset 0x74.
- The earlier field is restored to doubled-base offset 0x54.
- A word acquired at retained-base offset 0x14 supplies a later byte fragment.
- A subsequently loaded base supplies a halfword destination at twice that base, receiving 0x65.
- Words at offsets 0x64 and 0x74 of that loaded base become the final invocation’s second and fourth operands.
- A computed word destination at offset 0x54 receives 100.
- A later loaded word supplies another destination at offset 0x54, which receives the final invocation’s second operand.
- The final direct word store writes that later loaded word at twice the second operand.

Unsupported shift counts and signed-negative shifted values remain **documented_only**.

### Final invocation and return

The final target receives, in order:

1. 0x72.
2. The word acquired at offset 0x64 of the subsequently loaded base.
3. The retained earlier field acquired at doubled-base offset 0x14, **after the fixed byte store at 0x3A**.
4. The later word acquired at offset 0x74 of the same base used for operand two.

No explicit memory operation follows the invocation before the represented return. The target’s effects and return behavior are unknown.

## Side effects

Direct effects include 8-bit, 16-bit, and 32-bit writes to computed and fixed addresses. Overlap and read-after-write dependencies are permitted by the represented address relationships.

No transaction, rollback, synchronization, or restoration boundary is visible. A fault or non-return can leave preceding effects in place.

No explicit error result, bounds check, null check, retry, or recovery path is represented. Dependencies may have their own failure behavior.

No identifiable direct allocation or release operation is established. A returned word used as an address does not establish ownership transfer, permission to release storage, or lifetime stability. Unknown dependencies may allocate, release, invalidate, or mutate storage.

Thread safety and reentrancy are unresolved. Access order and duplicate effects may matter for shared, device-backed, or otherwise externally observable storage; the storage classification itself is unknown.

## Dependencies

Only F-a9cc76a17bd4 has an established identity in the supplied mapping.

| Opaque reference | Documentary role | Status |
|---|---|---|
| F-000000000001 | First incoming-control-derived target | documented_only |
| F-000000000002 | Subsequent invocation positions collectively | documented_only; not one asserted shared callee |
| F-000000000003 | Final four-operand target | documented_only |

These references distinguish roles, not verified callee identities.

Dependencies may consume implicit state, modify globals, invalidate retained addresses, allocate or release memory, fail, or never return. No preservation or ownership rule is assumed.

The publication-base global and fixed-location storage are memory dependencies, not function dependencies.

## Acceptance examples

These are conditional static deductions, not executed tests.

### Result transformations

A forwarded primary result of 0x12 supplies 0x1200 after an eight-bit left shift. Under 32-bit bit-vector semantics, 0x01000000 supplies zero under the same transformation.

An early paired result with low component 3 and high component 5 supplies the described three operands as 768, 5, and 320.

### Compact-index boundary

Under bit-vector semantics, the compact index is a multiple of four from 0 through 508. Twice the index plus 0x14 ranges from 0x14 through 0x40C. This is not a memory-validity guarantee.

### Input-width distinction

An explicit computed byte load returning 0xE1 supplies 225 when widened.

Acceptance must not restrict the fixed-location input associated with 0x05 to 255 merely because it is converted to an unsigned word.

### Publication-base retention

If the final store at retained-base offset 0x20 changes overlapping publication-base-global storage, the following global acquisition observes that change.

If the later word store at 0x60 changes the global again, publication addresses still use the previously acquired value.

### Intervening store at 0x3A

Suppose the byte at 0x3A lies within the word at doubled-base offset 0x14. The earlier-field acquisition must occur after the 0x3A store and observe the resulting word. Its exact numeric value depends on byte order and the other bytes.

That post-store word supplies the final invocation’s third operand and the later restoration at doubled-base offset 0x54. A pre-store snapshot is unacceptable.

The same post-store requirement applies independently if 0x3A overlaps the word at retained-base offset 0x04.

If 0x3A instead overlaps the fixed word or payload field, their retained values still come from their earlier acquisitions. Reloading them after the store changes the represented dependency.

### Fresh reads at 0x67

Suppose the first fresh read observes 0xA9, an intervening overlapping write changes 0x67 to 0x3C before the second read, and another changes it to 0xE7 before the third.

The fixed byte at 0x21 receives 0xA9. The two later byte stores at retained-base offset 0x09 receive 0x3C and 0xE7 respectively. This assumes those writes produce the stated bytes without prescribing byte order.

### Overwrites and completion

All four word stores at retained-base offset 0x20 remain separate effects despite the final value being 0x20.

If the final target returns normally, the function returns without an explicit result. If it does not return, no alternate completion path is established.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose, object model, and semantic meaning of constants.
- Dependency identities, prototypes, contracts, and complete invocation-position trace.
- Complete enumeration of the intervening address-derived access trace.
- Exact repeated-publication count.
- Calling-convention interpretation of secondary result channels.
- Signed overflow, signed shifts, and exceptional shift counts.
- Source width and declaration of the input associated with address 0x05.
- Byte order, alignment, permissions, and memory classification.
- Ownership, lifetime, invalidation, concurrency, and reentrancy.
- Fidelity of fixed low-address storage to the execution environment.
- Final target resolution, effects, and return behavior.

The placement of the 0x3A store between the second and third retained-field acquisitions is established, not an uncertainty. Numeric consequences of overlap may remain unresolved because byte order and memory behavior are unknown.

These limitations do not justify adding validation, caching fresh reads, omitting repeated writes, inventing object types, or assuming dependency state preservation.

## Evidence

Evidence is limited to the supplied static body and function-identity mapping. Prior wording, reviewer assertions, embedded comments, and warnings are not independent behavioral authority.

Static support includes explicit access widths, conversions, result-component extraction, masks, retained values, duplicate accesses, and ordering.

The body establishes these boundaries:

- Publication-base-global acquisition follows the last store at retained-base offset 0x20 and precedes the fixed word store at 0x60.
- The fixed word and payload field are acquired before the byte store at 0x3A.
- The earlier field and retained-base field are acquired after that store.
- The earlier field remains the final invocation’s third operand.
- Three separate later reads of 0x67 supply distinct byte stores.

The unsigned conversion of the input associated with 0x05 does not establish its source access width.

No implementation access, dependency inspection, runtime execution, smoke run, or runtime tests have been performed. No independently verified provenance or snapshot hash is asserted.
