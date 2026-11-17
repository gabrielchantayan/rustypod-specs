<!-- Trusted provenance: F-a6f7ec2d6e16; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a6f7ec2d6e16

## Purpose

F-a6f7ec2d6e16 performs a straight-line sequence of indirect dependency invocations, then mixed-width memory updates, followed by a terminal indirect transfer.

The supplied representation does not establish an application-level purpose or recognized data format. Repeated offsets, character-like constants, and address publications do not independently establish formatting, object initialization, or container manipulation.

This contract separates:

- **Established static evidence:** explicit accesses, conversions, value dependencies, and represented ordering.
- **Inferred interpretation:** numerical consequences under stated arithmetic assumptions, not additional application semantics.
- **documented_only:** behavior left unresolved rather than assigned an executable guarantee.

Every later effect is conditional on preceding invocations returning and preceding calculations and accesses completing. No runtime tests have been performed.

## Interface

### Explicit inputs and outputs

| Input position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use in the visible body. |
| Second | 32 bits | No explicit use in the visible body. |
| Third | Signed 32 bits | Shifted left four positions and supplied to an early preparation invocation. |

The third input is transformed numerically, not explicitly dereferenced. The absence of visible uses of the first two inputs does not establish their irrelevance to unknown dependencies or implicit calling-convention state.

The represented interface has no return value. Outputs consist of dependency interactions, memory effects, and control transfer. No success flag, error result, output length, or completion marker is established.

The terminal transfer receives four explicit 32-bit arguments. Its first argument is 0x72. A return follows the transfer in the representation; resumption of the original caller depends on the terminal target returning.

### Implicit state and preconditions

Execution depends on incoming control-link state, embedded control locations, ordinary and secondary dependency results, existing memory, named globals, and an unresolved terminal target.

Completion requires compatible executable targets, meaningful consumed result channels, valid reached memory accesses, and defined target-level arithmetic. Alignment, permissions, and address-space requirements are platform-dependent. No visible checks enforce nullness, bounds, alignment, or input validity.

### Width and layout terminology

A **byte**, **halfword**, and **word** denote 8, 16, and 32 bits. Explicitly typed pointer accesses establish these widths.

Named-global declarations are unavailable. Consequently:

- Conversion of global 0x05 to unsigned 32 bits does not establish its physical read width.
- Retention of global 0xA9 in a 32-bit value does not establish its physical read width.
- An explicitly eight-bit supplied value does not establish an eight-bit named-global write.
- The selector global supplies a retained 32-bit base, but its physical read width is unresolved.
- The publication to global 0x3A has neither an established eight-bit supplied expression nor an established eight-bit destination.

Offsets identify access sites, not a validated record layout. No object sizes, field schema, or separation between bases is established.

## Behavior

### Preparation

The first indirect target is constructed from incoming control-link state with displacement 0xFFE. Subsequent preparation targets are constructed from embedded control locations with the same recurring displacement. Different construction sites are not assumed to identify the same dependency.

Invocations have zero to four explicit arguments. Ordinary results, secondary result-channel values, and separately consumed halves of represented 64-bit results feed later invocations. Transformations include shifts, masks, low-word extraction, high-word extraction, and explicit zeros.

No explicit conditional branch or iteration appears in preparation. This does not establish termination: a dependency may block, fault, redirect control, or fail to return. A call with no explicit arguments is not necessarily input-free or side-effect-free. The complete site-by-site dependency interaction contract remains **documented_only**.

### Arithmetic model

Conditional numerical examples use 32-bit bit-pattern arithmetic: word-sized left shifts and calculated addresses truncate to 32 bits. This is an interpretation for checking value relationships, not a resolution of actual signed-overflow or address semantics.

Relevant distinctions include:

- Low-word and high-word extraction select different halves of represented 64-bit results.
- Unsigned 64-bit right shifts by 34, 36, and 44 occur before narrowing.
- Unsigned right shifts introduce zeros.
- Explicit byte and halfword destinations retain eight and sixteen bits.
- Explicit unsigned-byte reads are zero-extended when published through word destinations.
- Successive word-sized shifts retain intermediate truncation. A word shifted left 25 positions and subsequently 11 positions becomes zero under this model.

One late byte publication derives from a signed right shift whose count is the low eight bits of a calculated base. Negative operands and counts outside 0–31 require target semantics; no count-masking rule is assumed.

The named-global publication at 0x3A has a different conversion boundary: the adjusted base is converted to **char before multiplication**, then integer-promoted for multiplication by eight and addition of 0x30. There is no explicit final eight-bit narrowing of that completed arithmetic expression. Char signedness, out-of-range conversion behavior, and destination conversion remain unresolved.

### Values entering direct-memory processing

The following semantic roles describe retained value relationships without assigning object semantics:

| Role | Derivation |
|---|---|
| Shifted quantity | A retained preparation-result word shifted left 25 positions. |
| Large base | A separate retained result multiplied by 2²⁵. |
| Byte-scaled quantity | Another retained result shifted left eight positions. |
| Compact value | A result shifted left eight positions, masked to its low fifteen bits, then shifted unsigned right six positions. |
| Doubled compact value | Twice the compact value. |
| Final preparation pair | Low and high words of the last represented 64-bit preparation result. |

Under the bit-pattern model, the compact value is four times its producer’s low seven bits: a multiple of four from 0 through 508. Its doubled value ranges from 0 through 1016. Neither zero nor other low addresses are visibly rejected.

### Opening effects and global 0x05

The opening effects publish the shifted quantity through a word destination at offset 0x14 from the final preparation pair’s high word. A byte at shifted-quantity offset 9 then receives the low byte of that high word.

Before that byte location is written again, a word at offset 0x54 from 256 times the final pair’s low word receives the large base. The repeated offset-9 byte publication supplies the shifted quantity’s low byte, which is zero under the bit-pattern model.

The large base’s offset-0x14 word is cleared before an explicit unsigned-byte read at its offset 5. That byte is retained. The doubled compact value’s offset-0x14 word is cleared, and the retained byte is published as a zero-extended word at large-base offset 0x34.

The large base’s offset-0x14 word is then read and retained. After another clear of the doubled-compact destination, that retained word supplies two consecutive byte writes at doubled-compact offset 0x15.

Only after both writes is global 0x05 read and converted to unsigned 32 bits. The retained converted value is not established to lie within 0–255.

That read precedes publication to global 0x01 of the doubled compact value’s low eight bits. Subsequently, the retained converted-global value selects an offset-0x15 byte destination receiving the compact value’s low byte. Global destination widths and conversions remain unresolved.

### Retained links and fixed-global interleavings

The byte-scaled quantity is supplied to a byte destination at large-base offset 0x11 and a word destination at offset 0x54 from the earlier retained large-base offset-0x14 word. The offset-0x11 byte is subsequently cleared.

A word at converted-global-base offset 0x54 is read before replacement with that earlier retained word. The old word is the **retained link**. Later calculations use the retained value rather than reloading the replaced location.

Further processing publishes the compact value at doubled-compact offset 0x34 and follows a word at large-base offset 0x74. The reached base receives offset-0x54 word updates and repeated offset-9 byte updates. The retained link is also published twice into its own offset-0x24 word, with intervening effects preserved.

A nested assignment contains two word-read expressions for doubled-compact offset 0x74. One supplies the published value; the other selects an offset-0x54 destination. Both evaluations precede that assignment’s store. Their relative order is unresolved; the assignment’s own store cannot occur between them.

The fixed-global publication at 0x54 is interleaved with doubled-compact offset-0x14 updates:

- The calculated word receives the large-base producer shifted left 26 positions.
- Global 0x54 receives twice the retained link.
- The calculated word receives the retained link twice.

The supplied expression for global 0x54 is word-sized, but its physical write width and destination conversion are unknown.

Two subsequent byte writes at doubled-compact offset 0x15 both supply the low byte of four times the retained link. **After both**, global 0x15 receives the doubled compact value’s low eight bits. Further calculated word publications then include the byte-scaled quantity at offset 0x54 from four times the retained link, and four times the retained link at doubled-compact offset 0x34.

### Calculated propagation and constants

The retained link and its multiples select further accesses:

- A word reached through doubled-compact offset 0x74 is doubled and propagated through offset-0x44 destinations.
- An unsigned byte at offset 5 from four times the retained link selects a low-address offset-0x74 word read.
- Self-address words are published at calculated offset-0x14 locations.
- An offset-0x64 word selects another destination for a retained byte and a self-address publication.
- A word read at offset 4 selects later offset-5 byte reads.
- Those byte reads are distinct accesses with intervening writes; they are not an entry-time snapshot.
- Byte-derived values published through word destinations are zero-extended.

A later loaded word selects three ordered publications of 0x3A: a byte at four times the word, a halfword at eight times it, and a word at offset 0x20 from its fourfold address. These explicit destinations establish widths independently of the later named global at 0x3A.

For the next phase:

- The **adjusted base** is four times the retained link minus 150.
- The **publication base** is four times the adjusted base.

Publication-base offset 0x20 receives ordered word publications of 0x20, 0x2F, a zero-extended byte read from publication-base offset 4, and 0x20 again. Final memory alone does not capture the intermediate publications or intervening read.

### Repeated publication phase

A separate selector global is read into a retained 32-bit base. Global 0x60 is then assigned 0x20; its physical write width is unresolved.

A finite run of publication groups follows without intervening calls. Each group orders:

1. A word publication at a scaled base’s offset 0x54.
2. A byte publication at selector offset 0x19.
3. A byte publication at publication-base offset 0x11.
4. A word publication at selector offset 0x34.

The first word value is four times the late-loaded word used for the preceding three 0x3A publications. The two byte values are the low bytes of the publication base and selector, respectively.

The first group uses eight times the retained link for its scaled destination base and final word value. Subsequent groups use eight times the adjusted base instead.

The selector is retained, not repeatedly read from its global. Aliasing writes do not change that retained value. The exact repetition count is **documented_only** in this contract. The representation contains finite repeated statements, not an unbounded loop; reducing them to one group is not established as equivalent for write-observable memory.

### Global 0x3A: conversion before arithmetic

After the repeated phase, global 0xA9 supplies a retained 32-bit value, and the fourfold late-loaded base’s offset-0x74 word is retained. The publication to global 0x3A follows these reads and precedes the final phase’s offset-0x14 and publication-base offset-4 retained reads.

The supplied value is formed by converting the adjusted base to char, promoting that converted value, multiplying by eight, and adding 48. In the represented 32-bit integer setting, all eight-bit char values fit the promoted arithmetic type. The multiplication and addition are therefore not eight-bit arithmetic.

For adjusted base 34, the char conversion preserves 34 regardless of char signedness, and the completed expression is **320**, before destination conversion. Calling that expression a byte value of 0x40 is unsupported.

If the destination were an unsigned eight-bit object, destination conversion would produce 0x40. If its declared type could retain 320 unchanged, it could receive 320. Neither declaration is available, so neither stored result is unconditional.

Char signedness and conversion behavior matter for adjusted bases outside the common representable range. They remain **documented_only**; conversion is not replaced by an assumed unsigned low-byte interpretation.

### Final changes and terminal arguments

The final phase retains an offset-0x14 word from the doubled late-loaded link and a word at publication-base offset 4 before further updates. It publishes 0x6C through an explicit byte destination at publication-base offset 8.

Established transitions include:

- The doubled link’s offset-0x54 word receives its own base, then later the retained offset-0x14 word.
- Selector offset 0x54 receives a loaded word, then the publication base.
- Global 0x67 receives the low eight bits of the retained global-0xA9 value.
- A subsequent read of global 0x67 supplies global 0x21.
- Later reads of global 0x67 supply explicit publication-base offset-9 byte stores. These are not unconditional copies of the initially retained value.
- Publication-base offset 0x10 first receives a byte and later a word read from publication-base offset 0x40.
- The signed-shift-derived byte is stored at doubled-link offset 1.
- A newly loaded word selects a halfword publication of 0x65 at twice its value.
- A derived offset-0x54 word receives 100.
- The last word store targets twice a retained late-loaded word.

The terminal transfer uses:

| Argument | Source |
|---|---|
| First | 0x72. |
| Second | Retained word from offset 0x64 of the base selected through doubled-link offset 4. |
| Third | Earlier retained doubled-link offset-0x14 word, also used to restore its offset-0x54 destination. |
| Fourth | A later word read at offset 0x74 from the same base supplying the second argument. |

The second argument is not reloaded after intervening writes. The fourth argument is read before the final memory updates, not necessarily at transfer time. The offset-0x74 value retained immediately after the repeated phase is a different source.

## Side effects

### Memory and ownership

The function directly reads and modifies fixed and calculated locations. Explicit pointer accesses establish widths; missing named-global declarations do not.

No disjointness guarantee separates globals, calculated bases, or overlapping offset ranges. Reads observe state at their represented evaluation points, subject to platform semantics. Retained values remain retained even if their original locations are overwritten.

Ownership, allocation provenance, sharing, object boundaries, lifetimes, and access validity remain **documented_only**. Self-address publication does not establish ownership or container membership.

No allocation or release is explicitly represented in the direct-memory phase. Unknown dependencies may nevertheless allocate, retain, release, or mutate storage. Neither argument passing nor address publication independently establishes ownership transfer.

### Errors and observability

No transaction, rollback, synchronization, atomicity, retry, or error translation is represented. Faults and nonreturning dependencies can prevent later effects; completed effects are not visibly undone.

Represented ordering constrains this static contract. It does not prove that every repeated access survives optimization or is independently observable on a particular platform. Diagnostics, concurrency, reentrancy, recovery, and device-memory behavior remain unresolved.

## Dependencies

Opaque IDs describe unresolved roles, not resolved external identities:

| ID | Role |
|---|---|
| F-unknown-01 | Initial incoming-control-derived invocation. |
| F-unknown-02 | Family of subsequent preparation invocation sites. |
| F-unknown-03 | Terminal indirect transfer. |

F-unknown-02 does not assert a shared target across invocation sites.

Targets, complete prototypes, implicit inputs, secondary-result semantics, memory effects, errors, and return behavior remain **documented_only**. No dependency is assumed to preserve globals or to own or release referenced memory.

## Acceptance examples

These are conditional static checks, not executed tests.

### Input and compact-value arithmetic

For third-input bit pattern 3, the shifted contribution is 0x30. For 0x10000000, it is zero under 32-bit truncation.

If the compact producer’s low seven bits are 127, the compact and doubled values are 508 and 1016. If those bits are zero, both values are zero; no visible rejection follows.

### Global conversion is not a byte-read guarantee

If global 0x05 supplies unsigned value 0x00001200 after its unresolved declared-type conversion, the subsequent offset-0x15 byte destination is 0x00001215 under the address model.

That read precedes publication to global 0x01. Treating the retained base as zero solely because the global is identified by address 0x05 is unsupported.

### Fixed-global interleavings

For compact value 12 and retained link 0x1000, the doubled compact value is 0x18.

Global 0x54 receives supplied value 0x2000 between the earlier calculated offset-0x14 publication and the later publication of 0x1000 there.

The two later calculated offset-0x15 byte writes target 0x2D and supply zero. After both, global 0x15 receives eight-bit supplied value 0x18. These facts do not establish either global’s physical write width.

### Char conversion and destination conversion

For retained link 46, the adjusted base is 34. Its char conversion preserves 34, and the promoted arithmetic expression supplied to global 0x3A is 320, or 0x140.

Acceptance must not require a supplied expression of 0x40. That stored result is conditional on a destination conversion that narrows to unsigned eight bits. A destination capable of retaining 320 need not discard the upper bits.

### Aliasing and retained reads

A cleared word reads as zero only if intervening effects do not change its bytes. Overlapping byte writes may change later word reads according to byte order.

A retained link continues to supply calculations after its original offset-0x54 location is overwritten.

For stable ordinary memory containing 0x2000 at doubled-compact offset 0x74, the nested assignment’s two reads both yield 0x2000, and its store targets 0x2054 with that value. The store follows both operand reads; their relative order remains unresolved.

### Interrupted execution

If a preparation dependency does not return, subsequent direct-memory effects are not required.

If the terminal transfer is reached, its first explicit argument is 0x72 and the remaining arguments use the retained sources specified above. Successful return is not guaranteed.

## Uncertainties

The following remain **documented_only**:

- Application purpose, data format, and object interpretation.
- Complete dependency-site contract and calling convention.
- Indirect targets, secondary-result semantics, and termination.
- Named-global declarations, physical widths, signedness, and conversions.
- Char signedness and out-of-range conversion before the global-0x3A arithmetic.
- The stored result of global 0x3A after its unknown destination conversion.
- Signed overflow, exceptional shifts, address wrapping, alignment, and byte order.
- Memory validity, ownership, aliasing, sharing, lifetime, and special-memory semantics.
- Relative ordering of nested operand reads and independent runtime observability.
- Exact repeated-publication count within this contract.
- Terminal control-flow role, errors, and return behavior.
- Runtime faults, concurrency, timing, and device effects.

Unknown widths do not erase established conversion placement or ordering. Unknown aliasing does not justify assuming disjoint memory.

## Evidence

This contract derives from static inspection of the supplied representation. Prior prose, feedback, comments, and warnings are not independent behavioral proof.

Explicit access types support the stated byte, halfword, and word constraints. Named-global expressions support supplied-value relationships and ordering, but absent declarations prevent assigning physical access widths.

For global 0x3A, the conversion applies to the adjusted base before multiplication and addition. Integer promotion follows that conversion; no explicit narrowing applies to the completed arithmetic expression. This supports 320 as the pre-destination-conversion value for adjusted base 34 while leaving the stored value unresolved.

The representation also supports the preparation structure, calculated accesses, fixed-global interleavings, repeated publications, retained reads, and terminal argument relationships described above.

No target resolution, instruction inspection, execution, or runtime testing has been performed.
