<!-- Trusted provenance: F-fbbc56edfe0d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbbc56edfe0d

## Purpose

F-fbbc56edfe0d performs an ordered sequence of unresolved indirect invocations, transforms their result bit patterns, and then reads and mutates storage through computed addresses and named locations. Its last represented invocation supplies four arguments to an unresolved indirect target.

No domain-level purpose is established. Character-like constants, repeated assignments, and self-address-like values do not establish text processing, object initialization, or linked-structure maintenance.

This contract distinguishes:

- **Established static evidence:** explicitly represented operations, widths, conversions, value dependencies, and ordering.
- **[INFERENCE]:** conclusions conditional on an assumed execution model.
- **documented_only:** unresolved behavior or details not enumerated by this contract.

Every effect is conditional on execution reaching it. An invalid access, invalid invocation target, or dependency that does not return can prevent subsequent effects.

## Interface

### Explicit inputs

| Argument | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use. |
| Second | 32 bits | No explicit use. |
| Third | Unsigned 32 bits | Supplies two early invocation arguments after transformation. |

The third input is used in two transformations:

- Left shift by 24.
- Selection of its low eight bits, followed by left shift by five.

Under 32-bit unsigned arithmetic, the first places the original low byte in bits 24–31. The second produces a value from zero through 8,160.

No input validation or explicit rejection is represented. The first two arguments are not established as pointers, lengths, or handles. Their lack of explicit use does not prove that dependencies cannot observe incoming machine state associated with them.

### Implicit inputs and storage types

Execution depends on incoming return-link state, executable targets, primary and secondary dependency results, existing storage contents, and access permissions. Some results occupy 64-bit containers whose portions are consumed separately. This does not establish a complete ABI.

Three captures require a distinction between **captured value width** and **underlying storage width**:

- Storage associated with 0x05 is read and explicitly converted to unsigned 32 bits.
- A separate named global is sampled later into a 32-bit address-like value used throughout repeated stores.
- Storage associated with 0xA9 is sampled late into a 32-bit saved value.

None of these destination widths independently establishes the source storage’s width, signedness, or extension behavior.

The same restriction applies to named assignments associated with 0x01, 0x15, 0x21, 0x3A, 0x54, 0x60, and 0x67. A narrow expression on an assignment’s right-hand side does not establish a narrow destination access.

### Outputs and completion

No explicit scalar return value is represented. Observable outputs include dependency effects, storage mutations, and a final indirect invocation with these arguments, in order:

1. The constant 0x72.
2. A saved 32-bit word loaded late from offset 0x64 of a loaded base.
3. A saved 32-bit word captured earlier from offset 0x14 of a different computed base.
4. A saved 32-bit word loaded from offset 0x74 of the base used for the second argument’s load.

The fourth argument is not the earlier capture associated with 0xA9.

A return is represented after this invocation. Whether the target returns normally or uses another continuation convention is **documented_only**.

### Preconditions and ownership

There are no represented checks for null addresses, bounds, alignment, target validity, or lifetime.

Successful progression requires every reached access and invocation target to be valid under the actual environment. Readability and writability requirements for undeclared named storage cannot be reduced to particular byte counts.

No allocation, release, retention, borrowing, or ownership-transfer discipline is established. Dependency-produced address-like values are not necessarily newly allocated or function-owned.

## Behavior

### Indirect-invocation phase

Before the first explicitly typed direct-memory store, a lengthy straight-line sequence of indirect invocations supplies zero through four explicit arguments.

The entry target derives from incoming return-link state plus 0xFFE. Subsequent targets generally derive from fixed continuation-like values plus the same displacement. This establishes target construction, not a dispatch-table abstraction or verified dependency identity.

The opening phase contains two argument-free invocations before a third produces a result that is shifted left by eight and supplied to the next invocation. Further invocations intervene before two equal, additionally shifted values are supplied together.

Later opening sites supply:

- Zero and the third input shifted left by 24.
- A dependency result shifted left by eight and the third input’s low byte shifted left by five.

The remaining sites combine shifted, masked, extracted, saved, and constant values. Some saved results survive intervening invocations before reuse. Argument-free invocations are not established as pure.

The complete site-by-site argument ledger is not enumerated here and remains **documented_only** coverage. This grouping does not authorize dropping, merging, or reordering invocations.

### Arithmetic and access-width rules

**[INFERENCE] Conditional machine model:** word arithmetic and address formation retain the low 32 bits after arithmetic. Numerical overflow examples use this model, not arbitrary-precision arithmetic.

Represented transformations include left shifts, unsigned right shifts, masks, subtraction, and multiplication by powers of two. Some extractions operate on full 64-bit values, including positions above bit 31.

Under the conditional model:

- Left shift by eight discards the original high byte.
- Left shift by 25 retains contributions only from the original low seven bits.
- Left shift by 30 retains contributions only from the original low two bits.
- Two successive left shifts by 28 produce zero.
- Explicit unsigned right shifts insert zeros.

Signed-looking operations do not establish portable signed-overflow semantics.

One late byte destination receives a signed-looking right-shift result. The shift count is the low eight bits of a computed address-like value. Negative-operand behavior and counts of at least 32 are **documented_only**; no machine count-masking rule is established.

For **explicitly typed pointer accesses**, byte, halfword, and word widths establish one-, two-, and four-byte accesses respectively. Explicit unsigned byte reads converted to words are zero-extended. These rules must not be extended to undeclared named storage merely because its consumer or assigned expression has a known width.

### Initial direct-memory effects

The last dependency result before direct mutation supplies distinct low and high words. Earlier results supply additional bases.

The initial effects retain this order:

- A word store at offset 0x14 through the high-result-derived base.
- A byte store at offset nine through another derived base.
- A word store at offset 0x54 through the low result scaled by 256.
- Another byte store at the same offset-nine destination.

The second byte store remains separate even when it replaces the first.

A base formed by scaling an earlier result by 2²⁵ then has its offset-0x14 word cleared. Its offset-five byte is read before another scaled destination’s offset-0x14 word is cleared. The unsigned byte is zero-extended and stored as a word at offset 0x34 of the first base.

A subsequent offset-0x14 word read remains a read. Replacing it with zero requires assumptions about memory behavior and interference that are not established. Duplicate clears and equal byte stores also remain distinct effects.

### Early named-storage sampling and assignments

The read associated with 0x05 occurs before the assignment associated with 0x01. Its unsigned 32-bit conversion is saved as an address base for an explicitly typed offset-0x15 byte store and offset-0x54 word read and write.

For an integral source, conversion to unsigned 32 bits produces the source value modulo 2³². The representation does not establish which value the underlying storage read produces. In particular, it does not establish a byte load, zero-extension from eight bits, or a 0–255 range.

The assignment associated with 0x01 receives an explicitly one-byte-converted doubled value. It does not copy the 0x05 capture. Its destination storage width remains unresolved.

Further computed effects intervene before:

- An assignment associated with 0x54 receives an earlier saved word shifted left by one.
- An assignment associated with 0x15 receives an explicitly one-byte-converted earlier doubled base.

The established ordering is:

**0x05 sampling → 0x01 assignment → 0x54 assignment → 0x15 assignment → later 0x60 assignment.**

This is an ordering constraint, not adjacency. None of these named assignments independently establishes a byte- or word-width write.

### Computed-address propagation and layout

Loaded words and explicitly unsigned loaded bytes become address bases. Values written through typed pointers include zeros, scaled dependency results, saved words, truncated addresses, and self-address-like values.

| Relative offset | Explicitly typed access categories |
|---|---|
| 0x01, 0x08, 0x09, 0x0D, 0x11, 0x15, 0x19 | Byte writes |
| 0x04 | Byte or word reads |
| 0x05 | Unsigned byte reads |
| 0x10 | Byte and word writes |
| 0x14 | Word reads and writes |
| 0x20, 0x24, 0x34 | Word writes |
| 0x40 | Word read |
| 0x44, 0x54, 0x74 | Word reads and writes |
| 0x64 | Word reads |

Different bases appear throughout; this is not a validated object layout. Computed-base offset-five byte reads do not establish the type of the distinct named source associated with 0x05.

The propagation region retains repeated self-reference stores, repeated byte stores, and reads whose results become further destinations. A nested operation reads the same offset-0x74 location twice, once for the value and once to derive a destination. Their relative evaluation order is **documented_only**.

A typed byte destination at offset 0x15 can overlap a word beginning at offset 0x14. The numerical result depends on endianness. For undeclared named storage, even the access footprint needed to determine overlap is unresolved.

### Constant-writing and repeated-store regions

A typed-pointer region writes 0x3A at three widths: byte, halfword, and word, through differently scaled destinations. The word destination is at offset 0x20 of one computed base.

Another base is formed by subtracting 0x96 from an earlier scaled value and multiplying by four. Its offset-0x20 word receives, in order:

- 0x20.
- 0x2F.
- An explicitly unsigned byte read from its offset four, zero-extended to a word.
- 0x20 again.

Afterward, a separate named global is captured into a 32-bit address-like value. This sampling precedes the assignment of 0x20 to named storage associated with 0x60. The latter assignment’s destination width is unresolved.

The captured base is reused throughout the repeated-store region; no per-group global reload is represented. Mutation of its backing storage does not replace the saved base.

Each repeated group retains four typed-pointer effects in this order:

1. Word store at offset 0x54 of the group’s principal scaled destination.
2. Byte store at offset 0x19 of the captured global base.
3. Byte store at offset 0x11 of the constant-sequence base.
4. Word store at offset 0x34 of the captured global base.

The initial group uses an earlier saved word scaled by eight as its principal destination. Later groups use the post-subtraction value scaled by eight. Byte values derive from saved low-eight-bit conversions of the two bases, not fresh global reads.

Exact group multiplicity is not enumerated here and remains **documented_only** coverage. A single group is not established as equivalent to the repeated sequence.

### Late captures and distinct named-storage samples

After the repeated stores, storage associated with 0xA9 is captured into a **32-bit saved value**. This establishes neither a four-byte read nor the source’s signedness.

An explicitly typed word at computed offset 0x74 is captured next. Both captures precede the assignment associated with 0x3A.

The 0x3A assignment’s expression uses an eight-bit character conversion of the post-subtraction value, multiplies that interpreted value by eight, and adds 0x30. Character signedness and the named destination’s conversion are unresolved. Under ordinary integral character interpretation, the expression’s low eight bits agree with eight times the post-subtraction value plus 0x30; this does not establish a byte-width write or the complete stored value.

Only afterward are an explicitly typed offset-0x14 word from a doubled-link-derived base and an offset-four word from the constant-sequence base captured. The former supplies the final invocation’s third argument.

Further typed stores include 0x6C at offset eight of the constant-sequence base and a self-address-like word at offset 0x54 of the doubled-link-derived base.

The low eight bits of the saved 0xA9 capture are then supplied to the assignment associated with 0x67. The expression’s one-byte conversion does not establish the named destination’s storage width.

There are three distinct subsequent samples of storage associated with 0x67:

1. After a computed offset-0x44 word read, the first supplies the assignment associated with 0x21. No explicit byte-width source read or destination write is established for this named-to-named assignment.
2. After further mutations and an offset-0x40 word read, the second supplies an explicitly typed byte destination at computed offset nine. The consumer imposes a one-byte conversion; it does not determine source load width.
3. After another computed offset-nine store, an offset-0x44 word store using the saved 0xA9 capture, and an offset-0x54 word read, the third supplies another explicitly typed offset-nine byte destination.

These are fresh samples at their represented positions. Intervening accesses may overlap the named storage, and the samples need not agree. Reusing the original 0xA9 capture or a single 0x67 sample is not established as equivalent.

### Final mutations and invocation

Remaining effects propagate saved words and perform further typed byte writes at offsets nine and 0x11.

A late loaded base is doubled for a halfword write of 0x0065. Its offset-0x64 word is then captured for the final invocation’s second argument. Another computed destination receives the word value 100.

The late base’s offset-0x74 word is captured for the fourth argument before additional reads and stores. Subsequent effects include an offset-0x54 word store and a word store through twice the saved second-argument value.

The final invocation uses saved captures, not final rereads of their original locations.

## Side effects

Direct effects include mixed-width typed-pointer accesses, named-storage reads and assignments with unresolved storage widths, and unresolved indirect control transfers.

| Named location association | Established operation; storage width unresolved |
|---|---|
| 0x05 | Read followed by explicit unsigned 32-bit conversion and saved address use |
| 0x01 | Assignment supplied with an explicitly one-byte-converted doubled value |
| 0x15 | Assignment supplied with an explicitly one-byte-converted doubled base |
| 0x21 | Assignment supplied by the first later 0x67 sample |
| 0x3A | Assignment supplied by character-conversion, multiplication, and addition |
| 0x54 | Assignment supplied by an earlier saved value shifted left by one |
| 0x60 | Assignment supplied with 0x20 |
| 0xA9 | Read captured into a 32-bit saved value |
| 0x67 | Assignment supplied with the saved 0xA9 capture’s low eight bits; three later distinct reads |

This inventory specifies neither byte footprints nor source signedness. It does not replace the ordering constraints.

Low-address associations do not establish ordinary global objects, disjoint storage, permissions, alignment, or device behavior. Possible overlap must not be dismissed using guessed named-storage widths.

No rollback, synchronization, transaction boundary, or explicit error-reporting path is represented. Completed effects are not visibly undone if execution stops. Dependencies may introduce further mutations or resource releases; preservation of unmentioned memory is not guaranteed.

## Dependencies

Only F-fbbc56edfe0d has an established identity in the supplied mapping.

| Opaque dependency ID | Role | Unresolved properties |
|---|---|---|
| F-unresolved-01 | Entry invocation derived from return-link state | Identity, ABI, effects, completion |
| F-unresolved-02 | Subsequent ordered computed-target invocations | Identities, complete argument ledger, results, effects |
| F-unresolved-03 | Final four-argument invocation | Target, ABI, effects, return behavior |

These IDs classify roles without asserting distinct executable targets.

No dependency is assumed to preserve globals or secondary machine results, retain or release ownership, or return normally.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Input transformations.** Under 32-bit arithmetic, third inputs 0x000000AB and 0x123456AB both produce 0xAB000000 and 0x00001560 at the two input-transformation sites. Whole-execution equivalence does not follow.

2. **Low-byte boundary.** Third input 0x00000100 produces zero at both sites. Input 0x000000FF produces 0xFF000000 and 0x00001FE0. No explicit range rejection is represented.

3. **0x05 source interpretation.** If its source is unsigned eight-bit 0xFF, conversion gives 0x000000FF. If it is signed eight-bit −1, unsigned 32-bit conversion gives 0xFFFFFFFF. An unsigned 16-bit source 0x1234 instead gives 0x00001234. None of these source declarations is established.

4. **Named read versus capture width.** Capturing the 0xA9-associated source into 32 bits does not require a four-byte load. Any acceptance condition concerning its accessed byte range requires an independently established storage declaration.

5. **Narrow assignment expression.** The one-byte expression supplied to the 0x67 assignment does not establish that adjacent storage remains untouched. That requires the named destination’s actual width and conversion rules.

6. **Early ordering.** Sampling associated with 0x05 precedes the 0x01 assignment; the 0x54 assignment precedes the 0x15 assignment; all precede the 0x60 assignment. Intervening computed accesses remain significant.

7. **Global capture.** If backing global storage changes during repeated stores, later groups still use the previously captured base. Entry-time sampling and per-group reloads do not satisfy this requirement.

8. **Late capture timing.** If the 0x3A assignment overlaps a later offset-0x14 capture source, that later read observes the resulting memory state under the applicable memory model. The earlier 0xA9 capture remains saved.

9. **Three 0x67 samples.** If intervening effects change the sampled source values, the three consumers may receive different results. The second and third consumers convert their respective samples for typed byte stores; the first remains a named-storage assignment with unresolved destination width.

10. **Explicit widths.** The typed-pointer stores of 0x3A affect one, two, and four bytes. These widths do not establish widths for named assignments elsewhere.

11. **Interrupted progression.** If F-unresolved-02 does not return, subsequent direct mutations and F-unresolved-03 need not occur. No local recovery is established.

12. **Final arguments.** F-unresolved-03 receives 0x72, the late offset-0x64 capture, the saved offset-0x14 capture, and the late offset-0x74 capture, in that order.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, complete ABI, and externally meaningful postcondition.
- Dependency identities, effects, completion, and ownership behavior.
- Complete invocation ledger and exact repeated-store multiplicity as enumerated guarantees.
- Widths, signedness, extension, and assignment conversions of undeclared named storage, including the separately sampled global base.
- Address validity, alignment, permissions, endianness, and actual overlap.
- Signed-overflow, character-signedness, and variable-shift semantics.
- Relative evaluation order of the nested duplicate offset-0x74 reads.
- Concurrency, device-memory behavior, allocation, release, and reentrancy.
- Whether the final indirect invocation returns.

Unresolved storage declarations do not erase established sampling points, explicit consumer conversions, saved-value reuse, or operation order.

## Evidence

This contract is based on the supplied static representation, not embedded comments, prior prose, or reviewer assertions as independent authority.

Established evidence includes explicit argument widths, transformations, computed invocations, typed-pointer access widths, capture timing, repeated mutations, three separately represented 0x67 samples, and the final four supplied arguments.

For undeclared named storage, the representation establishes assignments and consumer widths or conversions—not necessarily storage access widths. Specifically, the 0xA9 capture establishes a 32-bit saved value, while the 0x67 samples establish distinct reads and subsequent consumers. Neither establishes byte footprints without declarations.

The modular arithmetic model is labeled **[INFERENCE]**. No runtime tests, execution traces, tool-based inspection, or independent machine-level verification have been performed.
