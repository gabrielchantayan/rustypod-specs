<!-- Trusted provenance: F-dddf5f925f3c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dddf5f925f3c

## Purpose

F-dddf5f925f3c performs an ordered sequence of indirect dependency invocations, transforms dependency-produced values, mutates memory through computed addresses, and ends with an unresolved indirect transfer carrying four operands.

The supplied body establishes represented value dependencies, explicitly typed memory accesses, and sequencing. It does not establish an application-level purpose. Character-like constants do not prove text construction; repeated offsets and self-references do not establish a particular data structure or ownership model.

This contract distinguishes:

- **Established static evidence:** operations and relationships directly supported by the supplied body, conditional on that representation accurately describing execution.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **documented_only:** unresolved properties for which no behavior is invented.

No runtime tests have been performed.

## Interface

### Inputs

The represented interface has no explicit parameters. Implicit inputs include:

- Incoming control state used to derive the first indirect target.
- Dependency results and additional result-register state.
- Values read through computed addresses.
- Named shared objects associated with addresses including 0x05, 0x67, and 0xA9.
- A separate named shared object whose value supplies a publication base.
- The unresolved final transfer target.

Some dependency results occupy represented 64-bit containers and are consumed through their lower and upper 32-bit portions. This supports the extraction relationships, not a verified dependency signature or a single logical 64-bit return object.

### Outputs and completion

No scalar return value is represented. Outputs consist of dependency invocations, shared-object accesses, explicitly typed memory accesses and mutations, and the final indirect transfer.

A return follows that transfer in the supplied body. Normal completion therefore requires the transfer to return and every preceding reached operation to complete. No success status, termination guarantee, or caller-state preservation contract is established.

### Widths and layout

| Represented width | Established use |
|---|---|
| 8 bits | Explicit computed-address byte accesses and explicit one-byte value conversions |
| 16 bits | Explicit computed-address stores of 0x003A and 0x0065 |
| 32 bits | Local arithmetic values, explicitly typed word accesses, address-bearing values, and terminal operands |
| 64 bits | Dependency-result containers used for extraction |

Offsets include 0x01, 0x04, 0x05, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x14, 0x15, 0x19, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74. They are byte offsets, not element indices. Their recurrence does not establish one universal record layout.

**Named-object qualification:** declarations for the named globals are absent. Their names, addresses, destination locals, and assignment expressions do not independently establish their storage widths or signedness. This applies to objects at 0x05, 0xA9, and 0x67, other fixed-address named objects, and the separate shared publication-base source.

In particular:

- Converting the object at 0x05 to unsigned 32 bits establishes the converted value’s type, not an unsigned byte load.
- Capturing the object at 0xA9 in a 32-bit local establishes the retained value’s width, not the source load width.
- Assigning a one-byte-converted value to the object at 0x67 does not establish a byte-wide store.
- Reading that object into an explicitly byte-wide computed destination does not establish a byte-wide source read.

Endianness, alignment requirements, address mapping, and device-memory semantics remain documented_only.

### Preconditions

Normal execution requires each reached indirect target to be executable under the applicable calling convention and every reached memory access to be valid at its actual width.

No visible validation rejects null, low, unaligned, overlapping, or arithmetic-derived addresses. Returned and loaded address-bearing values have no established validity, lifetime, or ownership guarantee.

## Behavior

### Invocation phase and arithmetic

Before the direct-memory mutation region, the function executes a long straight-line sequence of indirect invocations. The first target depends on incoming control state plus 0xFFE. Subsequent represented targets use embedded control-location values with the same displacement.

Invocation order remains significant even where results are ignored or operands are zero. Identical operands do not make separate invocations interchangeable. An invocation with no explicit argument expressions does not establish absence of machine-level inputs.

Transformations include:

- Lower and upper 32-bit extraction from represented 64-bit results.
- Left shifts of signed and unsigned 32-bit representations, frequently by eight bits.
- Masking before shifting, including low-order fields of 1, 11–15, 17–22, and 24 bits.
- Reuse of captured values across intervening invocations.
- Explicit zero operands alongside opaque result-register values.

Unsigned right shifts are zero-filling. Right shifts of unsigned 64-bit containers by 34, 36, or 44 extract the upper portion shifted right by two, four, or twelve bits respectively; they are not oversized shifts of 32-bit operands.

Explicit one-byte conversions retain eight bits of represented data. Explicit unsigned-byte reads widened to unsigned 32 bits produce values from 0 through 255. That rule does **not** extend to named objects with missing declarations.

Signed shifts, additions, and multiplications occur. Portable overflow behavior is not established, and assigning a signed expression to an unsigned destination does not retroactively make the expression unsigned. Signed overflow, signed right shifts, address wraparound, and out-of-range variable shifts remain documented_only where target semantics are needed.

There is no visible result-based error branch, retry loop, or conditional early exit. Dependencies can nevertheless fail, redirect control, or not return.

### Opening memory mutations

The last dependency result supplies lower and upper 32-bit portions used with earlier retained values.

The opening sequence includes:

- A 32-bit store at the upper result portion plus 0x14.
- A byte store at a saved shifted-value base plus 0x09, receiving the upper portion’s low byte.
- A 32-bit store at an address derived from the lower portion multiplied by 256, plus 0x54.
- A second byte store at the same saved base plus 0x09, receiving a retained shifted value’s low byte.

Both byte stores are represented effects; the intervening word store prevents treating them as an isolated final-value assignment.

Initialization then includes a 32-bit zero store at another base plus 0x14, an explicit unsigned-byte read at that base plus 0x05, zero stores through a twice-scaled base, and a 32-bit store of the widened byte at offset 0x34.

A subsequent 32-bit read at offset 0x14 observes memory at that point. Earlier writes through other address expressions may alias it; its value cannot automatically be replaced with zero.

### Capture from the named object at 0x05

Two consecutive byte stores first write the same truncated value to a computed base plus 0x15. The named object at 0x05 is then evaluated and converted to unsigned 32 bits. An assignment to the named object at 0x01 follows.

The captured unsigned value subsequently supplies destinations at offsets 0x15 and 0x54. A 32-bit word at captured base plus 0x54 is read before that location is overwritten.

The capture must remain after the consecutive stores and before the assignment at 0x01. Without established non-aliasing and access semantics, moving it across those operations is unjustified. Later changes to its source do not replace the retained value.

The source object’s width and signedness are documented_only. The captured base is not established to lie between 0 and 255. A negative signed narrow source, if that were its declaration, would convert to a large unsigned 32-bit value rather than zero-extend its raw byte representation.

### Address-bearing updates

The mutation region derives addresses and data using multiplication by two, four, and eight, left shifts, and subtraction of decimal 150.

Effects include self-references at offsets 0x14, 0x24, and 0x54; repeated overwrites at offsets 0x14, 0x44, and 0x54; and byte-truncated publications at offsets 0x09, 0x0D, 0x11, and 0x15.

Explicit unsigned-byte reads at computed bases plus 0x05 supply bases for 32-bit reads at offset 0x74. Repeated byte reads are separate observations. One expression also contains two reads of offset 0x74; those are not one retained snapshot. Evaluation order between sibling reads within that expression is unresolved.

No non-aliasing, structural, or ownership invariant follows from these pointer-like relationships.

### Constant stores and publication-base capture

One region stores 0x3A through three **destination expressions that may alias**:

| Destination expression | Store width |
|---|---:|
| Four times a loaded base | 8 bits |
| Eight times that base | 16 bits |
| Four times that base, plus 0x20 | 32 bits |

These are three ordered stores, not necessarily three distinct addresses. If the base is zero, the first two destinations are both zero.

A subsequent computed destination at offset 0x20 receives four successive 32-bit values: 0x20, 0x2F, a zero-extended explicit byte read from the same base plus 0x04, and 0x20 again. The byte read follows the first two word stores and precedes the store of its widened value.

For the remaining behavior:

- **Adjusted base** means the earlier four-times-scaled value after subtraction of 150.
- **Publication base** means four times the adjusted base.
- **Captured shared base** means the 32-bit local value obtained from the separate named shared source.
- **Terminal record base** means the previously computed twice-scaled base used by the late record accesses.

These are discussion terms, not claims about allocated objects.

The shared-source capture precedes the assignment of 0x20 to the named object at 0x60. Subsequent publication uses the captured value, not a fresh evaluation after that assignment. The source load width and the 0x60 destination width remain documented_only.

### Repeated publication

A finite straight-line publication region repeatedly performs explicitly typed accesses involving:

- A byte store of the publication base’s low byte at captured shared base plus 0x19.
- A byte store of the captured shared base’s low byte at publication base plus 0x11.
- A 32-bit store of an eight-times-scaled value at captured shared base plus 0x34.
- A 32-bit store of a saved four-times-scaled value at an eight-times-scaled destination plus 0x54.

The initial group uses an eight-times-scaled destination and publication value derived from the unadjusted value. Later groups use the adjusted value. It is not one uniform group from the beginning.

All represented repetitions remain part of the access behavior. This contract does not enumerate their exact count; that count is documented_only here. A final-state-only replacement is not justified where aliasing or externally observable accesses matter.

### Final shared-object observations

After repeated publication, the named object at 0xA9 is evaluated into a retained 32-bit value. A separate, explicitly 32-bit read at a saved scaled base plus 0x74 follows. Only afterward is the named object at 0x3A assigned an arithmetic expression derived from the adjusted base, a character-sized conversion, multiplication by eight, and addition of 0x30.

Neither the 0xA9 source width nor the 0x3A destination width is established by these expressions.

Further accesses include an explicit byte store of 0x6C at publication base plus 0x08 and a 32-bit self-reference store at terminal record base plus 0x54. The retained 0xA9-derived value is then converted to one byte and assigned to the named object at 0x67. The receiving object’s storage width and any further destination conversion remain unresolved.

The following are separate source evaluations, not reuse of the originally assigned value:

1. After the assignment at 0x67, an explicit 32-bit read at terminal record base plus 0x44 captures a word.
2. A fresh evaluation of the object at 0x67 supplies an assignment to the named object at 0x21.
3. After intervening computed accesses, another evaluation of 0x67 supplies an explicit byte store at publication base plus 0x09.
4. A byte store at terminal record base plus 0x09, a 32-bit store of the retained 0xA9-derived value at publication base plus 0x44, and a 32-bit read at publication base plus 0x54 intervene.
5. A third evaluation of 0x67 supplies another explicit byte store at publication base plus 0x09.

The three source evaluations have unresolved access widths and conversions. Their values need not be equal. The assignment at 0x21 also has unresolved destination width.

### Terminal mutations and transfer

The word captured from terminal record base plus 0x44 contributes to a byte store at publication base plus 0x10 and a 32-bit store at captured shared base plus 0x54. A later store overwrites the latter with the publication base.

That captured word also supplies a right-shifted byte store at terminal record base plus 0x01. The shift count is the publication base’s low eight bits, not a byte read through that address. Counts outside the supported range receive no invented masking rule.

A 32-bit read at publication base plus 0x40 supplies a 32-bit store at publication base plus 0x10, overlapping the earlier byte destination. A previously captured word is written at captured shared base plus 0x74 before that copy.

The value read from publication base plus 0x54 becomes a destination base. Further mutations restore a previously captured word at terminal record base plus 0x54 and publish bytes derived from the saved offset-0x74 value.

Near completion:

- A 32-bit read at terminal record base plus 0x04 supplies a loaded base.
- A 16-bit store writes 0x0065 at twice that loaded base.
- A 32-bit read at loaded base plus 0x64 captures a late value.
- A computed destination plus 0x54 receives decimal 100 as a 32-bit store.
- A 32-bit read at loaded base plus 0x74 captures another terminal operand.
- A separate 32-bit read at terminal record base plus 0x64 supplies another base.
- That base plus 0x54 receives the late value, and twice the late value receives the other base through a 32-bit store.

The final indirect transfer receives, in order:

1. 0x72.
2. The late value from loaded base plus 0x64.
3. The earlier word captured from terminal record base plus 0x14.
4. The late word from loaded base plus 0x74.

No success, error, or ownership interpretation is established for these operands.

## Side effects

Named fixed-address effects include evaluations at 0x05 and 0xA9; assignments at 0x01, 0x15, 0x21, 0x3A, 0x54, 0x60, and 0x67; and three subsequent evaluations at 0x67. Their physical access widths are documented_only because their declarations are unavailable. Explicit narrow conversions constrain values, not necessarily named-object storage accesses.

Computed accesses have the explicit widths described above. They may overlap one another or named-object storage. Different destination expressions do not imply distinct addresses.

No transaction, rollback, lock, or restoration boundary is visible. Partial execution can leave preceding effects in place.

No identifiable allocation or release occurs in the direct-memory region. Dependencies may allocate, release, retain, or transfer memory; none of those effects, or preservation of shared state, may be assumed.

Invalid accesses or targets can prevent completion. Platform-specific fault behavior, reentrancy, and thread safety remain documented_only.

## Dependencies

Only the subject has a resolved supplied opaque identity. These opaque discussion labels do not assert resolved target identities:

| Opaque ID | Role | Unresolved properties |
|---|---|---|
| F-U001 | Initial indirect invocation | Target, signature, effects, return behavior |
| F-U002 | Population of later invocations before direct mutation | Target identities, conventions, result-register provenance, effects |
| F-U003 | Final indirect transfer | Dispatch, signature, ownership effects, whether it returns |

F-U002 denotes a population, not one identified function. No dependency is assumed pure, memory-owning, memory-releasing, or globally preserving.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Result extraction.** For a represented 64-bit result of 0x1234567800000001, upper-portion extraction yields 0x12345678. Shifting the lower portion left eight yields 0x00000100 where the represented signed operation is valid.

2. **Mask before shift.** Selecting the low 24 bits of unsigned 0xFF000001 and shifting left six yields 0x00000040.

3. **Unknown declaration at 0x05.** If the source is an unsigned eight-bit object holding 0xE3, conversion yields 0x000000E3. If instead it is a signed eight-bit object holding −29, conversion to unsigned 32 bits yields 0xFFFFFFE3. The body does not select either declaration. Both cases retain the established capture timing.

4. **Aliasing constant destinations.** If the shared scale base is zero and the accesses are valid, the 8-bit and 16-bit 0x3A stores both target address zero, in that order. The 32-bit store targets 0x20. Rejecting this case because destinations must be distinct would add an unsupported precondition.

5. **Captured shared base.** If the assignment at 0x60 affects the shared-base source through permitted overlap, publication still uses the value captured beforehand.

6. **Retained 0xA9-derived value.** Later writes do not cause the retained 32-bit value to be reloaded. Its later explicit 32-bit store does not prove that its original source access was 32 bits wide.

7. **Independent 0x67 evaluations.** If successive evaluations yield 0x41 and 0x52 after a permitted intervening change, the corresponding explicit byte destinations receive those respective low-eight-bit values. This does not establish source access width or the width of the assignment at 0x21.

8. **No rollback or invented shift semantics.** An unsupported variable shift has no specified substitute behavior. If F-U003 does not return, normal completion is not guaranteed and preceding writes have no visible rollback.

## Uncertainties

The following remain documented_only:

- Application-level purpose and valid caller scenarios.
- Accuracy of the supplied static representation.
- Named-global declarations, storage widths, signedness, conversions, and volatility.
- Dependency identities, conventions, effects, and additional result-register provenance.
- Signed overflow, signed right shifts, variable-shift boundaries, and address wraparound.
- Endianness, alignment, mapping, device-memory behavior, and complete alias relationships.
- Evaluation order between sibling reads not separately sequenced.
- Exact publication repetition count not enumerated here.
- Allocation, release, retention, ownership, and lifetime effects of dependencies.
- Concurrency, termination, and final-transfer dispatch behavior.

**Inferred intent:** repeated offsets and self-references suggest structured-state manipulation; character-like constants suggest possible formatting data. Neither interpretation is an acceptance requirement.

## Evidence

The contract uses the supplied body’s represented invocations, local widths, explicit access types, conversions, arithmetic, retained values, and terminal operands. Prior prose, embedded comments, and warnings are not independent proof.

Explicit computed-address access types support their stated widths. Missing named-global declarations prevent equivalent width and signedness claims for those objects. Conversion to unsigned 32 bits at 0x05 is not evidence of zero extension from an unsigned byte. Capture into a 32-bit local at 0xA9 is not evidence of a 32-bit load. The assignments and subsequent evaluations at 0x67 do not establish that object’s width.

The three scaled 0x3A destination expressions preserve three stores and their respective widths without establishing address inequality. Capture timing and separate source evaluations are supported independently of the unresolved named-object types.

No dependency bodies, executable-semantics reference, runtime trace, tools, or runtime tests were used.
