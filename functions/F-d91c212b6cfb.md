<!-- Trusted provenance: F-d91c212b6cfb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d91c212b6cfb

## Purpose

F-d91c212b6cfb performs an ordered sequence of unresolved indirect invocations, transforms selected outputs into 32-bit fields and address-like quantities, and performs mixed-width memory mutation before a final unresolved invocation.

No application-level purpose is established. Numeric constants that coincide with character encodings do not establish text formatting, serialization, or a string-output interface. Likewise, repeated offset-based accesses do not establish an object initializer or collection algorithm.

This contract distinguishes:

- **Established static evidence:** represented access widths, data dependencies, invocation order, and memory-access order.
- **[INFERENCE]:** consequences conditional on a conventional 32-bit bit-vector execution model.
- **documented_only:** unresolved behavior for which no executable acceptance rule is established here.

The contract concerns the supplied static representation. Its accuracy and reachability in the underlying executable remain unresolved.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no explicit return value. Effective inputs include entry machine state, unresolved invocation outputs, additional machine-state outputs, and memory at fixed and computed addresses.

There is no established result buffer, scalar result, status flag, or error code. Observable effects consist of reached invocations and memory accesses. A return is represented after the final invocation, but reaching that return requires the unresolved dependency to return normally.

### Widths and layout

| Quantity | Width | Behavioral significance |
|---|---:|---|
| Byte access | 8 bits | Stores retain the low eight bits |
| Halfword store | 16 bits | Writes two bytes; distinct from byte and word stores |
| Word access | 32 bits | Values may subsequently serve as addresses |
| Selected invocation output | 64 bits | Low and high 32-bit portions are consumed separately |
| Ordinary arithmetic intermediate | Generally 32 bits | No general widening before scaling or shifting is established |

All offsets in this document are byte offsets. Important offsets include 0x04, 0x05, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x14, 0x15, 0x19, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74.

These accesses do not define a structure layout. Computed bases need not identify separate objects. For example, a word at offset 0x14 overlaps a byte at offset 0x15. Endianness, alignment requirements, and object boundaries are not established.

### Preconditions and failures

No validation protects the invocation targets or memory addresses. Successful continuation requires every reached target to be executable and every reached access to satisfy the actual runtime’s mapping, permission, alignment, and execution-mode requirements.

Computed addresses can be zero or very small. Fixed low-address accesses are also present. The representation establishes no error return, exception translation, recovery path, or rollback.

## Behavior

### 1. Ordered indirect-invocation prefix

The initial, largest region is a straight-line sequence of indirect invocations interspersed with arithmetic. No explicit data-dependent loop or branch is represented there.

The first invocation, F-unknown-01, uses a target derived from entry machine state plus displacement 0xFFE. Subsequent invocation targets use embedded address-like anchors with that displacement. Their regularity does not establish common implementation or common behavior.

F-unknown-02 identifies these remaining prefix occurrences collectively, not a single interchangeable callee. Occurrences have zero to four represented arguments. Argument dependencies include:

- Zero and other retained values.
- Earlier outputs shifted left by eight bits.
- Unmodified high 32-bit output portions.
- Shifted or masked low and high portions.
- Values retained across intervening invocations.
- Identical values supplied in multiple argument positions.
- Additional machine-state outputs with unresolved calling-convention meaning.

The third represented invocation produces a value shifted left by fourteen bits; that retained value is subsequently shifted by another six bits for argument formation. Later processing includes shifts of five through thirty-one bits.

Selected invocations produce represented 64-bit outputs whose halves are used independently. This is a static data dependency, not proof of a conventional 64-bit return-value ABI. Invocations whose outputs are not consumed can still have effects and must not be removed on that basis.

### 2. Field transformations and arithmetic

Explicit masks retain low fields of 24, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, and one bit. Masking, shifting, and retaining intermediate widths are significant. Retaining twenty-four bits before a six-bit left shift is not equivalent to retaining twenty-four bits afterward.

Unsigned right shifts are logical. Shifting an unsigned 64-bit output right by thirty-four, thirty-six, or forty-four bits yields its high word shifted logically by two, four, or twelve bits, respectively.

[INFERENCE] Under 32-bit bit-vector arithmetic, left shifts discard bits above bit 31. Two successive twenty-eight-bit left shifts therefore produce zero, rather than a widened fifty-six-bit shift. Similarly, successive left shifts by eight, eighteen, and eight bits produce zero under that model.

The representation also contains signed-looking shifts and multiplication. Modulo-2³² interpretations of those operations are conditional, not established language-level overflow guarantees.

Near the prefix’s end, an output is shifted left by eight bits, masked to fifteen bits, and shifted logically right by six. Call the resulting bounded component **the narrow component**.

[INFERENCE] With 32-bit bit-vector shifts, the narrow component equals four times the original output’s low seven bits. It ranges from zero to 508 in multiples of four. Its double ranges from zero to 1016 in multiples of eight. These bounds establish no memory validity.

### 3. Initial direct-memory mutation

For this region:

- **Final pair** means the last represented 64-bit prefix output.
- **Large base** means an earlier output multiplied by 2²⁵.
- **Tag base** means another retained output shifted left by twenty-five bits.
- **Narrow base** means twice the narrow component.
- **Saved word** means a retained invocation output shifted left by eight bits.

These are dependency labels, not object types.

The initial stores publish the tag base as a word at final-pair-high offset 0x14, then publish the low byte of final-pair-high at tag-base offset 0x09. A word store at offset 0x54 from the final-pair-low portion scaled by 256 publishes the large base. The tag-base byte is then overwritten with the tag base’s own low byte.

The clear-and-read sequence is alias-sensitive. A word at large-base offset 0x14 is cleared before reading a byte at large-base offset 0x05. A word at narrow-base offset 0x14 is then cleared before the previously read byte is zero-extended and stored as a word at large-base offset 0x34.

The word at large-base offset 0x14 is subsequently read. Narrow-base offset 0x14 is cleared again, and two consecutive byte stores at narrow-base offset 0x15 publish the low byte of that read word.

Neither the read word nor the byte obtained after the earlier clear can be simplified using presumed object separation. Intervening accesses may overlap their locations.

### 4. Cross-linked mutation

The next region reads the byte at fixed address 0x05, zero-extends it, and uses it as a base. This read precedes a byte write at fixed address 0x01. Further writes mix the narrow component, saved word, cleared-region results, and words obtained through offsets 0x54 and 0x74.

A word obtained from the byte-derived base at offset 0x54 is retained before that location is overwritten. Call this retained word **the cross-link word**. Its doubled, fourfold, and eightfold forms become later values and addresses.

Established mutation characteristics include:

- Word links at offsets 0x04, 0x44, 0x54, 0x64, and 0x74 are read or replaced.
- Byte stores at offsets 0x09, 0x0D, 0x11, and 0x15 interleave with word stores.
- Some word stores publish their own computed base.
- A word at fixed address 0x54 receives twice the cross-link word.
- A byte is written at fixed address 0x15.
- Several identical stores occur more than once.
- Two represented word reads at the same offset-0x74 address participate in one assignment: one determines its destination and the other supplies its value. Their relative operand-evaluation order is unresolved.
- Repeated reads at offset 0x05 are fresh byte reads, zero-extended before use, not necessarily copies of an earlier byte.

No list, tree, reference count, ownership graph, or synchronization discipline follows from this access pattern.

### 5. Constant stores and retained-base publication

A later word read through a zero-extended byte and offset 0x74 supplies a link value. Call it **the publication link**. Its double remains important in the final region; its fourfold form is published repeatedly.

The constant-store group writes numeric 0x3A through three widths: a byte at four times the publication link, a halfword at eight times that link, and a word at offset 0x20 from four times that link.

For the repeated publication region, define:

- **Adjusted value:** four times the cross-link word, minus 0x96.
- **Publication base:** four times the adjusted value.
- **Retained base:** a word read from a separate fixed global location.

At publication-base offset 0x20, ordered word stores supply 0x20, then 0x2F, then a zero-extended fresh byte from publication-base offset 0x04, then 0x20 again. The byte read remains represented even though its widened value is overwritten.

**The retained-base global word is read after those stores and before the word store of 0x20 at fixed address 0x60.** The retained base is therefore the value obtained before that fixed-address store. Reversing these accesses is not established as equivalent, even if ordinary disjoint memory would sometimes give the same values.

The first publication occurrence writes four times the publication link at offset 0x54 from eight times the cross-link word. It then publishes the publication base’s low byte at retained-base offset 0x19, the retained base’s low byte at publication-base offset 0x11, and eight times the cross-link word as a word at retained-base offset 0x34.

Later repeated occurrences use eight times the adjusted value for both the offset-0x54 destination base and the value stored at retained-base offset 0x34. Their two intervening byte stores retain the same dependencies.

Thus, the first occurrence differs from later occurrences in both that destination dependency and the final word value. No counter advancement or destination progression is represented. Exact repetition multiplicity is documented_only here; the sequence must not be treated as one idempotent update or an array-fill contract.

### 6. Final mutation and transfer

For this section, **doubled link base** means twice the publication link. The retained base and publication base keep their meanings above.

The opening memory order is:

1. Read a word at fixed address 0xA9.
2. Read a word at offset 0x74 from four times the publication link.
3. Write a computed byte at fixed address 0x3A.
4. Read and retain a word at doubled-link-base offset 0x14.
5. Read and retain a word at publication-base offset 0x04.

The byte written at 0x3A is the low-byte result of eight times the adjusted value plus 0x30. This expression does not establish decimal conversion or textual meaning.

The retained offset-0x14 and offset-0x04 reads occur **after** the fixed-address byte write. They may therefore observe its effect if their accessed ranges overlap 0x3A. The earlier offset-0x74 word has already been captured.

Subsequent byte stores publish retained low bytes at doubled-link-base offset 0x0D and publication-base offsets 0x0D and 0x09; publication-base offset 0x08 receives 0x6C. A word at doubled-link-base offset 0x54 receives that base itself.

The low byte of the retained fixed-address-0xA9 word is then written at fixed address 0x67. **This publication is followed by three distinct live reads of byte address 0x67:**

- After reading the word at doubled-link-base offset 0x44, a byte read at 0x67 supplies the write to fixed address 0x21.
- After intervening byte and word mutations, another byte read at 0x67 supplies a store at publication-base offset 0x09.
- After further mutations and a word read at publication-base offset 0x54, a third byte read at 0x67 supplies another store at publication-base offset 0x09.

These are not established as copies of the initially published byte. Intervening stores can overlap 0x67.

Between these reads, the function writes the offset-0x44 result’s low byte at publication-base offset 0x10, publishes that result at retained-base offset 0x54, and stores a right-shift-derived byte at doubled-link-base offset 0x01. The shift count is the publication base’s low byte. Signed right-shift behavior and counts outside the supported word-width range remain documented_only; no explicit five-bit count reduction is established.

Other mutations replace retained-base offset 0x54 with the publication base, publish the previously retained offset-0x04 word at retained-base offset 0x74, replace publication-base offset 0x10 with a word freshly read at its offset 0x40, and write the doubled link base at retained-base offset 0x44.

After the second live 0x67 read supplies its byte store, doubled-link-base offset 0x09 receives the publication base’s low byte. Publication-base offset 0x44 receives the retained 0xA9 word. Those writes precede the third live 0x67 read and can affect its value through aliasing.

Later effects include restoring the retained offset-0x14 word at doubled-link-base offset 0x54, publishing low bytes from retained words, and replacing publication-base offset 0x44 with the freshly loaded offset-0x54 link.

The closing link-following region obtains a word from doubled-link-base offset 0x04. A halfword at twice that word receives 0x65. A word read at its offset 0x64 is retained for the final invocation. Another destination receives decimal 100 at offset 0x54. A word at the same link’s offset 0x74 is then retained separately.

Finally, a fresh word from doubled-link-base offset 0x64 determines a destination whose offset 0x54 receives the retained offset-0x64 value. A word store through twice that retained value publishes the fresh destination link.

F-unknown-03 receives four represented arguments:

- 0x72.
- The retained word obtained through the closing offset-0x64 link.
- The word retained at doubled-link-base offset 0x14 immediately after the fixed-address-0x3A write.
- The word obtained through the closing offset-0x74 link.

The earlier publication-region offset-0x74 read is a different retained value and does not supply the fourth argument.

## Side effects

Fixed-address accesses include byte locations 0x01, 0x05, 0x15, 0x21, 0x3A, and 0x67, and word locations 0x54, 0x60, and 0xA9, plus the retained-base global word. Address 0x67 is both written and read repeatedly.

Effects are incremental. No atomic commit, rollback, cleanup, or restoration on failure is established. Earlier effects remain unconditionally unrolled back by this body if execution stops later.

No explicit allocation or release operation is identifiable. Ownership, lifetime, and dependency memory effects remain unresolved. Repeated stores and overlapping accesses must not be discarded solely because final bytes might match in ordinary memory.

## Dependencies

| Opaque ID | Role | Established information |
|---|---|---|
| F-unknown-01 | Initial invocation | Target depends on entry machine state |
| F-unknown-02 | Remaining prefix occurrences | Ordered indirect invocations with varying argument and output dependencies |
| F-unknown-03 | Final invocation | Four represented arguments beginning with 0x72 |

These IDs designate dependency roles, not verified distinct implementations. No assumption is made that dependencies preserve globals, preserve address mappings, allocate or release memory, transfer ownership, return normally, or report failures through a particular mechanism.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Width-sensitive forwarding

Given low output word 0x01000001, retaining twenty-four bits before a six-bit left shift yields 0x00000040. Preserving the discarded high bit is inconsistent with that transformation.

### Narrow-component boundaries

Under 32-bit bit-vector shifts, an original output with low seven bits all set produces narrow component 508 and narrow base 1016. Zero low seven bits produce zero. Neither establishes mapped memory.

### Retained-base access order

Given a trace that reaches publication, the retained-base global read precedes the word store of 0x20 at address 0x60. A trace containing the reverse order is not accepted merely because both executions eventually publish identical bytes.

### Final opening alias

Given an accessed retained-word range overlapping address 0x3A, its post-write read must use memory after the computed byte store. The opening offset-0x74 read precedes that store and cannot be moved after it on an assumption of separate objects.

### Live 0x67 values

Given 0x67 contains 0x2A at the second live read, that read supplies 0x2A to publication-base offset 0x09. If an intervening store changes 0x67 to 0x80 before the third live read, the later store uses 0x80, not the initially published byte.

### Repetition and interruption

Tracing memory must distinguish the consecutive duplicate byte stores at narrow-base offset 0x15. Final-state equality alone is insufficient.

If a dependency does not return, later operations need not occur. If direct mutation faults, no rollback is established.

### Final argument retention

At F-unknown-03, the first argument is 0x72. The retained offset-0x14 argument is not replaced by a fresh read, and the fourth argument uses the closing offset-0x74 read rather than the earlier publication-region read.

## Uncertainties

The following remain documented_only:

- Application purpose, valid callers, and actual reachability.
- Accuracy of the supplied static representation.
- Architecture, endianness, alignment, address interpretation, and memory mappings.
- Signed overflow, signed right shifts, and out-of-range shift counts.
- Calling conventions and additional machine-state outputs.
- Dependency identities, effects, ownership behavior, and return guarantees.
- Object extents, aliasing restrictions, lifetimes, concurrency, and device-memory behavior.
- Exact multiplicity of the repeated publication region.
- External fault handling and normal-return behavior.

[INFERENCE] Regular invocation targets, low addresses, and extensive repeated stores warrant caution about higher-level interpretation. They do not justify inventing an application algorithm or deleting represented effects.

## Evidence

This contract derives from the supplied static representation and dependency identity mapping. Embedded comments, prior prose, and reviewer statements are not independent behavioral authority.

Static evidence establishes mixed-width accesses, explicit arithmetic dependencies, ordered invocations, the global read before the 0x60 store, the final opening read/read/write/read/read order, and three live reads of byte address 0x67 after its publication.

No implementation access, runtime execution, tests, dependency inspection, or observed memory trace has been performed. Acceptance examples express conditional review criteria only.
