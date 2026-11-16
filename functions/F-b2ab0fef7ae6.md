<!-- Trusted provenance: F-b2ab0fef7ae6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b2ab0fef7ae6

## Purpose

F-b2ab0fef7ae6 performs an ordered sequence of indirect dependency invocations, width-sensitive transformations, and direct memory mutations, ending with an unresolved indirect invocation.

No application-level purpose is established. The evidence does not identify the operation as formatting, serialization, construction, list maintenance, or initialization. Character-like constants and recurring offsets do not establish those interpretations.

This contract distinguishes:

- **Established static evidence:** behavior explicitly represented in the supplied body.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior or detail not specified by this contract.

All requirements are conditional on the supplied static representation accurately describing execution. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32 bits | No explicit body-level consumption is represented. |
| Second input | 32 bits | No explicit body-level consumption is represented. |
| Third input | Signed 32-bit integer | Its bit pattern is shifted left by six for an early dependency argument. |

The first two inputs are not established to be semantically irrelevant: early indirect invocations have incomplete interfaces and may observe incoming machine state.

No accepted range, units, pointer interpretation, length interpretation, or validation rule is established for the third input.

### Implicit inputs and prerequisites

Execution additionally depends on incoming return-link state, dependency results, secondary register outputs, existing memory, an address-bearing global word, low-address storage, and an unresolved terminal target.

Every reached invocation target and memory access must be valid under the actual execution environment for execution to continue normally. No visible checks enforce target validity, mapping, bounds, alignment, or lifetime. Small computed integers are used directly as addresses; no separate allocation base is necessarily added.

The three explicit inputs therefore do not determine the observable behavior by themselves.

### Outputs and ownership

There is no represented explicit return value. Outputs consist of invocation boundaries and memory effects.

The terminal invocation has four represented arguments: 0x72, a late word loaded from an offset-0x64 field, a retained earlier offset-0x14 word, and a late offset-0x74 word. Their relationships are specified below. If the terminal invocation returns normally, the enclosing function reaches its represented return.

No explicit allocation, release, retain, or ownership transfer is identified. Dependencies are not established to avoid such actions. Dependency-derived addresses must not be presumed borrowed, owned, stable, or releasable.

## Behavior

### Dependency-driven preparation

Preparation is a long, straight-line sequence of indirect invocations. The first target depends on incoming return-link state plus 0xFFE. Later targets are represented using successive code-location values plus the same displacement. This does not establish common dependency identity.

No explicit loop, conditional branch, retry, or status test is represented in preparation. Dependencies may independently branch, mutate state, fail, or not return.

Observable argument transformations include:

- Left shifts of primary 32-bit results.
- Independent use of secondary register outputs.
- Separation of represented 64-bit results into low and high 32-bit portions.
- Passing a result alongside shifted versions of that result.
- Masking before shifting.
- Explicit zero arguments.
- Retention and later reuse of results across intervening invocations.

An unused primary result does not make an invocation dispensable. An invocation without represented explicit arguments is not established to have no implicit inputs. Preparation must not be treated as one pure arithmetic expression.

The complete invocation-site catalogue and every preparation argument dependency are **documented_only** here. The established invocation boundaries and transformations must not be replaced with an inferred high-level operation.

### Arithmetic and widths

Byte, halfword, and word accesses are respectively 8, 16, and 32 bits. Narrow stores retain the low eight or sixteen bits. Explicit unsigned-byte reads widen to values from 0 through 255.

For represented 64-bit results, low and high portions mean the least and most significant 32 bits. Logical full-container shifts by 34, 36, and 44 before narrowing correspond to logical shifts of the high portion by 2, 4, and 12. They are not oversized shifts of a 32-bit operand.

Masks, narrowing, and intermediate shifts remain significant. A left shift followed by a right shift does not recover discarded bits. Consecutive left shifts must not be interpreted as unlimited-precision multiplication.

Under the **conditional ordinary-word model**, left shifts, multiplication, subtraction, and address arithmetic retain the appropriate 32-bit result. Signed-looking expressions do not independently establish language-level overflow behavior. Instruction-level overflow and unusual variable-shift behavior remain documented_only.

Several retained values are explicitly zeroed and later shifted. Their derived arguments remain zero under ordinary word semantics; this establishes no preservation guarantee for globals or unspecified machine state.

### Entry into direct memory mutation

The following descriptive roles identify values, not object types:

| Role | Meaning |
|---|---|
| Final primary word | Low 32 bits of the last preparation result. |
| Final secondary word | High 32 bits of that result. |
| Shifted address | An earlier dependency-derived word shifted left by 25. |
| Scaled root | Another earlier result multiplied by 2²⁵. |
| Small field | A result shifted left by eight, masked to 15 bits, then logically shifted right by six. |
| Small base | Twice the small field. |

Under ordinary 32-bit semantics, the small field is a multiple of four no greater than 0x1FC. The small base is a multiple of eight no greater than 0x3F8. These are nevertheless used as addresses.

The initial write ordering is established: an offset-0x14 word through the final secondary word receives the shifted address; the shifted-address-plus-0x09 byte receives the final secondary word’s low byte; an offset-0x54 word through 256 times the final primary word receives the scaled root; then the same offset-0x09 byte receives the shifted address’s low byte. That last byte is zero under ordinary word semantics. Both byte writes remain observable boundaries.

The scaled-root offset-0x14 word is then zeroed. Its offset-0x05 byte is read before the small-base offset-0x14 word is zeroed and before the byte is stored, zero-extended, at scaled-root offset 0x34. The scaled-root offset-0x14 word is subsequently re-read, rather than replaced by a presumed zero. Another small-base offset-0x14 zero store follows, then two byte stores of the reloaded word’s low byte at small-base offset 0x15.

### Interdependent fields and early globals

A low-address value at 0x05 is read and widened to an unsigned word; its underlying storage width is not established. This read precedes the byte write at global location 0x01, which receives the small base’s low byte. Computed writes then use the loaded low-address value as a base, including offsets 0x15 and 0x54.

A word at that base’s offset 0x54 is captured before that field is replaced. Call the captured word the **captured link**. Later calculations form two, four, and eight times this word.

Other established effects include:

- An offset-0x74 load through the scaled root before writes through its loaded value.
- An operation reading small-base offset 0x74 both for a destination base and for the stored word. Independent ordering between those two reads is unresolved.
- Repeated self-referential stores, including a captured-link offset-0x24 field receiving the captured link itself.
- Repeated overwrites of small-base offset 0x14 and several offset-0x09 and offset-0x15 bytes.

The global word at 0x54 receives twice the captured link after an earlier small-base offset-0x14 write and before subsequent stores of the captured link to that field. Later, two small-base offset-0x15 byte stores precede the global byte write at 0x15, which receives the small base’s low byte. That global write precedes subsequent computed offset-0x54 and offset-0x34 stores.

Further mutations use offsets 0x04, 0x05, 0x14, 0x44, 0x54, 0x64, and 0x74. An unsigned byte read becomes an address base for a word read at offset 0x74. Thus this word read is based on a zero-extended byte, not an established pointer-sized field.

The complete intermediate access trace is documented_only here. No stable object schema or disjointness of these fields is established.

### Constants and repeated stores

For the remaining behavior, define these descriptive roles:

- **Branch word:** the later word obtained through the byte-based offset-0x74 lookup.
- **Doubled branch base:** twice the branch word.
- **Fourfold branch base:** four times the branch word.
- **Adjusted value:** four times the captured link minus 0x96.
- **Adjusted base:** four times the adjusted value.
- **Global base snapshot:** the address-bearing global word loaded before the repeated-store region.

All scaling is subject to the arithmetic qualifications above. No lower-bound check precedes subtraction of 0x96.

The constant 0x3A is stored separately as a byte at four times the branch word, a halfword at eight times that word, and a word at fourfold-branch-base offset 0x20. This is not an established string.

Adjusted-base offset 0x20 receives, in order, the words 0x20, 0x2F, a zero-extended byte read from adjusted-base offset 0x04, and 0x20. The byte read and intermediate writes remain significant.

**The address-bearing global word is loaded before the global word store of 0x20 at 0x60.** The loaded base is retained for subsequent accesses; the representation does not reload it for each repeated group. A computed write that later aliases its storage does not retroactively change this snapshot.

The repeated region has four ordered effects per group: an offset-0x54 word store through a scaled base, an offset-0x19 byte store through the global base snapshot, an adjusted-base offset-0x11 byte store, and a global-base-snapshot offset-0x34 word store. The byte values are respectively the adjusted base’s low byte and the snapshot’s low byte.

The first group uses eight times the captured link for its scaled base and offset-0x34 value. Subsequent groups use eight times the adjusted value. Their offset-0x54 stores carry the fourfold branch base.

Exact repetition multiplicity is documented_only in this contract. Repetition is established and must not be collapsed into one final-state update where count or ordering is observable. **Inferred intent:** the pattern may resemble publication of fields; that interpretation supplies no additional guarantees.

### Late global ordering and retained reads

After the repeated region, the global word at 0xA9 is read and retained as the **captured global word**. Next, a word at fourfold-branch-base offset 0x74 is read and retained as the **captured branch field**.

**Both reads precede the global byte write at 0x3A.** That byte receives the low eight bits of eight times the adjusted value plus 0x30.

Only after this byte write are the doubled-branch-base offset-0x14 word and adjusted-base offset-0x04 word read and retained. Call them the **retained restoration word** and **retained field word**.

These boundaries are alias-sensitive. A computed offset-0x74 read can observe preceding repeated writes; the 0x3A byte write can affect either subsequent retained read if their byte ranges overlap. The two retained reads must not be moved before that write.

Subsequent computed stores include the adjusted base’s low byte at doubled-branch-base offset 0x0D; 0x6C at adjusted-base offset 0x08; the snapshot’s low byte at adjusted-base offsets 0x0D and 0x09; and a self-referential doubled-branch-base offset-0x54 word store.

The global byte at 0x67 is then initialized from the captured global word’s low byte. This uses the retained word, not a fresh read of 0xA9.

### Fresh reads of global byte 0x67

The initialization of 0x67 and its later uses are distinct accesses:

- After initializing 0x67, a word is read from doubled-branch-base offset 0x44. Only then is 0x67 freshly read as a byte and copied to global byte location 0x21.
- A later adjusted-base offset-0x09 byte store obtains its value from another fresh read of 0x67.
- A second later adjusted-base offset-0x09 byte store obtains its value from a third fresh read of 0x67.

The two offset-0x09 stores are not a repeated use of one cached byte. Between the 0x21 copy and the first such store are computed byte and word writes, an adjusted-base offset-0x40 read, and further computed field writes.

Between the first and second fresh-read stores, doubled-branch-base offset 0x09 is written, adjusted-base offset 0x44 receives the captured global word, and adjusted-base offset 0x54 is read to obtain a new destination base.

Any intervening computed write may alias 0x67. Therefore, the 0x21 copy and the two later offset-0x09 stores need not contain the byte originally derived from 0xA9, nor need they match each other.

### Remaining mutations and terminal handoff

The late offset-0x44 word read supplies a byte stored at adjusted-base offset 0x10 and a word temporarily stored at global-base-snapshot offset 0x54. It also supplies a signed-looking right-shift result narrowed into doubled-branch-base offset 0x01; the shift count is the adjusted base’s low eight bits. No range check is shown. The snapshot’s offset-0x54 word is subsequently replaced by the adjusted base.

The adjusted-base offset-0x40 read supplies a later word store at offset 0x10 of that same base. Before that store, snapshot offset 0x74 receives the retained field word. Snapshot offset 0x44 subsequently receives the doubled branch base.

The base loaded from adjusted-base offset 0x54 is used for later writes. Its offset 0x0D receives the captured branch field’s low byte. Doubled-branch-base offset 0x54 receives the retained restoration word. An adjusted-base offset-0x14 word is then freshly captured before additional byte and offset-0x44 updates.

A word read from doubled-branch-base offset 0x04 supplies another base. Twice that base receives the halfword 0x65. Its offset 0x64 supplies the **terminal link word**, and its offset 0x74 later supplies the **terminal field word**. Between those reads, the previously loaded destination base receives a byte derived from the captured adjusted-base offset-0x14 word and the word 100 at offset 0x54.

Finally, a fresh doubled-branch-base offset-0x64 read supplies a destination whose offset 0x54 receives the terminal link word. Twice the terminal link word receives that destination as a word.

The terminal invocation follows, with represented arguments 0x72, terminal link word, retained restoration word, and terminal field word. No visible cleanup, rollback, or error translation intervenes.

## Side effects

### Access layout

| Width | Computed-base offsets represented |
|---|---|
| 8 bits | 0x01, 0x05, 0x08, 0x09, 0x0D, 0x10, 0x11, 0x15, 0x19 |
| 16 bits | Scaled addresses without a field offset |
| 32 bits | 0x04, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, 0x74 |

These are access offsets, not a recovered structure definition. Byte and word accesses at offset 0x10 have different extents.

Direct low-address accesses include byte writes at 0x01, 0x15, 0x21, 0x3A, and 0x67; three late byte reads at 0x67; word writes at 0x54 and 0x60; and a word read at 0xA9. The separate read at 0x05 has unresolved storage width.

### Aliasing, errors, and concurrency

Computed addresses may overlap each other and global storage. No named global is established to occupy a disjoint allocation. Final values consequently depend on aliasing, access widths, byte order, and memory-region behavior.

No volatile, atomic, synchronization, or transactional semantics are established. Repeated accesses remain part of the represented behavior without implying any particular hardware observability model.

No explicit null check, bounds check, alignment check, error return, exception handler, or rollback is represented. Fault handling is documented_only. Previously completed effects are not visibly undone if a later access or dependency fails.

## Dependencies

The supplied mapping identifies only F-b2ab0fef7ae6. No opaque function IDs are available for its indirect dependencies; their identities remain **documented_only**, rather than receiving invented IDs.

Established dependency categories include producers of 32-bit results, represented 64-bit results, and secondary register outputs; consumers of transformed arguments and constants; incompletely specified early invocations; and the unresolved terminal invocation.

No dependency is established to preserve globals, preserve pointed-to storage, avoid allocation, release memory, or return normally. Similar invocation patterns do not establish identical targets.

## Acceptance examples

These are conditional specification examples, not executed tests.

1. **Third input:** With input 3 and ordinary 32-bit shifting, the early shifted argument is 192. Dependency results remain unconstrained.

2. **Masked extraction:** Selecting the low 24 bits of 0xAB123456 and shifting left by six produces 0x048D1580. The original high byte does not participate.

3. **Full-container shift:** A represented 64-bit result with high word 0x80000010 yields 0x08000001 after logical full-container shifting by 36 and narrowing.

4. **Initial overwrite:** If reached locations are valid and the final secondary word ends in 0xAB, the initial shifted-address-plus-0x09 stores carry 0xAB and then zero under ordinary word semantics. A final zero alone does not establish the required ordering.

5. **Global snapshot:** If a later write changes the address-bearing global’s storage, subsequent repeated groups still use its earlier captured value. Reloading it after the 0x60 store is not equivalent.

6. **Late retained reads:** If the 0x3A write overlaps a subsequent retained read, that read occurs after the write. The 0xA9 and fourfold-branch-base offset-0x74 reads occur before it.

7. **Fresh global bytes:** Suppose ordinary readable storage at 0x67 contains 0x42 at the 0x21-copy boundary, but an intervening aliasing write changes it to 0x99 before a later offset-0x09 store. That later store uses 0x99, not the originally captured global word’s low byte.

8. **Independent later reads:** If an intervening write changes 0x67 between the two fresh-read offset-0x09 stores, those stores may differ. Caching the first byte for both would omit a required read.

9. **Invalid address and handoff:** An unmapped computed destination has no established function-defined error code. If all effects complete, replacing the terminal invocation with an ordinary return omits an established invocation boundary.

## Uncertainties

The following remain **documented_only**:

- Application purpose, data model, and object layouts.
- Dependency identities, complete interfaces, and preservation guarantees.
- Full calling convention and interpretation of secondary outputs.
- Complete preparation catalogue and intermediate access trace not enumerated here.
- Exact repeated-store multiplicity.
- Storage width of the low-address 0x05 read.
- Address mapping, byte order, alignment, and memory-region semantics.
- Signed-overflow and architecture-specific variable-shift behavior.
- Ownership, lifetime, reentrancy, concurrency, and failure recovery.
- Whether the terminal transfer is actually a call and returns normally.

These unknowns do not weaken the explicitly established global ordering or fresh-read boundaries.

## Evidence

This contract uses only the supplied static body. Prior prose and reviewer feedback are not independent evidence. No executable inspection, dependency-body inspection, implementation validation, or runtime testing has been performed.

Static evidence establishes the widths and transformations described above, direct access boundaries, repeated mutations, global-load-before-0x60 ordering, late reads on either side of the 0x3A write, separate reads of 0x67, and terminal argument relationships. Successful execution, ownership policies, and application-level meaning remain unestablished.
