<!-- Trusted provenance: F-fe620de21ddf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe620de21ddf

## Purpose

F-fe620de21ddf performs an ordered sequence of unresolved dependency invocations, followed by direct memory mutation and a final unresolved indirect transfer.

**Established static evidence:** dependency results contribute to shifted values, masks, address calculations, and later dependency arguments. Direct accesses include 8-bit, 16-bit, and 32-bit operations; overlapping fields; fixed-address storage; values loaded and subsequently used as addresses; and repeated writes to unchanged destinations.

**Inferred intent:** some stores resemble links between records, and some arithmetic resembles packed-field manipulation. Neither observation establishes an application-level operation, data structure, encoding, or ownership protocol.

The contract describes the visible interactions without assigning a semantic name such as initialization, formatting, serialization, or release. Successful completion, memory safety, and a particular final object graph are not established.

## Interface

### Explicit inputs

| Input position | Apparent width | Established contribution |
|---|---:|---|
| First | 32 bits | No explicit use appears after entry. |
| Second | 32 bits | No explicit use appears after entry. |
| Third | Unsigned 32 bits | Supplies low-byte-derived arguments during early dependency preparation. |

The third input contributes both its low eight bits shifted left by 24 and its low eight bits shifted left by five. Under 32-bit bit-vector arithmetic, the latter ranges from zero through 8,160 in multiples of 32.

The first two inputs cannot be declared irrelevant to the actual calling boundary: implicit argument consumption and the calling convention remain unresolved. No explicit input validation or rejection appears.

### Implicit inputs

Execution also depends on:

- Entry control context used to construct the first indirect target.
- Primary, secondary, and apparently paired result channels from dependencies.
- Memory reached through computed addresses.
- Fixed-address byte and word storage.
- A distinguished 32-bit stored-base slot, sampled late in the direct-access region.
- An unresolved final transfer target.

The operation is therefore not a pure function of its three explicit inputs. In particular, the stored-base slot is not sampled at entry.

### Outputs and completion

No explicit result value is established. Observable outputs comprise dependency interactions, direct memory accesses, state changes, and the final transfer with four explicit arguments.

An apparent return follows that transfer. This does not establish that the target returns, that the trailing return is reached, or that any result-register contents constitute an output contract.

### Preconditions for successful execution

Successful execution requires valid indirect targets, compatible calling conventions, and accessible memory for every executed access. Halfword and word destinations require appropriate alignment or target support for unaligned access. Low addresses must be meaningful in the execution environment.

No visible bounds check, allocation, pointer validation, or recovery mechanism establishes these conditions.

## Behavior

### Interpretation and ordering

“Established” means supported by the supplied static representation, not confirmed by execution.

The arithmetic model used for conditional examples is 32-bit bit-vector arithmetic: additions, multiplications, and left shifts retain the low 32 bits. This model is not a claim about portable signed-language semantics. Signed overflow, signed right shifts, and excessive variable shift counts require target interpretation.

Memory offsets are byte displacements. A byte store retains eight bits; a halfword store retains 16; a word store retains 32. Byte values promoted into word stores are zero-extended.

Access order matters. Captured values and later rereads are distinct inputs whenever intervening writes can affect the reread location. Repeated stores must not be collapsed merely because their final values agree.

### Dependency preparation

The operation begins with a long, straight-line sequence of indirect invocations. The first target depends on entry control context. Most later target constructions combine embedded control locations with a displacement of 4,094 bytes.

The sequence consumes primary 32-bit results, apparent 64-bit results split into halves, and secondary result channels whose interfaces are unavailable. Arguments include shifted results, masked results, zeros, and duplicated values. The third explicit input contributes early in this sequence; the continuation primarily depends on dependency outputs and constructed values.

There is no explicit branch or loop in the visible preparation. Dependencies may nevertheless branch, mutate memory, fail, or not return. Invocations with unused explicit results remain potentially observable.

Representative width-sensitive constructions include:

| Construction | Conditional bit-vector interpretation |
|---|---|
| Retain low 24 bits | Discards bits 24–31 before subsequent processing. |
| Retain low 22 bits | Range zero through 4,194,303 before scaling. |
| Retain low 20 bits, then shift right by ten | Extracts bits 10–19; range zero through 1,023. |
| Retain low 19 bits | Range zero through 524,287 before further operations. |
| Retain low 17 bits, then shift right by four | Extracts bits 4–16; range zero through 8,191. |
| Unsigned apparent 64-bit result shifted right by 36 | High 32-bit half shifted logically right by four. |
| Unsigned apparent 64-bit result shifted right by 44 | High 32-bit half shifted logically right by 12. |

An intervening left shift can discard information permanently. A subsequent right shift does not generally recover the original value. Successive fixed-width left shifts can also discard every contributing bit.

### Entry into direct memory mutation

For descriptive purposes, the direct-access region has three important derived quantities:

- The **shifted base**, formed by shifting a result left by 25.
- The **multiplication-derived base**, formed from another result multiplied by 33,554,432.
- The **small index**, formed by shifting a result left by eight, retaining the low 15 bits, and shifting right by six.

Under the bit-vector model, the small index is a multiple of four from zero through 508; its doubled value ranges from zero through 1,016 in multiples of eight. Both large bases are multiples of 33,554,432.

The first direct effects are ordered as follows:

1. A word at displacement 20 from the final preparation result’s high half receives the shifted base.
2. A byte at displacement nine from the shifted base receives the low byte of that high half.
3. A word at displacement 84 from 256 times the final result’s low half receives the multiplication-derived base.
4. The same displacement-nine byte receives the low byte of the shifted base.

The fourth store writes zero under the bit-vector model. It does not erase the preceding byte-store event from the access contract.

### Clearing, reading, and overlapping fields

The operation clears the word at displacement 20 from the multiplication-derived base, reads a byte at displacement five from that base, clears a small-index-derived word, and writes the zero-extended byte as a word at displacement 52 from the multiplication-derived base. It then reads the previously cleared displacement-20 word.

**For ordinary memory without external mutation, that word read returns zero under the bit-vector model.** This conclusion does not require a non-aliasing assumption:

- The intervening small-index-derived word store also writes zero. If it overlaps the cleared word, it cannot introduce a nonzero byte.
- The intervening word store at displacement 52 from the same multiplication-derived base cannot overlap the word at displacement 20. Their four-byte ranges remain disjoint under 32-bit modular addressing.

Device-memory behavior, read side effects, or external mutation could invalidate the ordinary-memory conclusion. Those are separate environmental uncertainties, not consequences of these intervening stores introducing nonzero data through aliasing.

Another clear of the small-index-derived word precedes two byte stores at displacement 21 from that derived base. These byte writes overlap its displacement-20 word. Under the same ordinary-memory conditions, their source is the captured zero.

The continuation reads the byte at fixed address five and writes a byte at fixed address one. Loaded bytes become zero-extended address components; loaded words become address bases. The region includes repeated self-referential stores at displacement 36, repeated byte stores at displacement nine, displacement-20 overwrites, and traversals through displacements four, 68, 84, 100, and 116.

These are access relationships, not validated object fields. In particular, a byte used to locate a word at displacement 116 is not thereby a checked index or pointer.

### Mixed-width constants and stored-base sampling

A later loaded word determines three distinct destinations receiving numeric value 58:

- A byte at four times the loaded value.
- A halfword at eight times the loaded value.
- A word at displacement 32 from four times the loaded value.

The widths and destinations differ; these are not interchangeable string writes.

An adjusted quantity is formed by subtracting 150 from a previously four-times-scaled value. Four times that adjusted quantity becomes the **adjusted base**. Its displacement-32 word receives, in order:

- Numeric value 32.
- Numeric value 47.
- A zero-extended byte read from displacement four of the adjusted base.
- Numeric value 32 again.

The byte read occurs after the writes of 32 and 47. No general non-aliasing guarantee covers other surrounding accesses.

**After the last displacement-32 store, the operation reads the distinguished stored-base slot as a 32-bit value. Only after this read does it write word value 32 at fixed address 96.** The sampled value becomes the **captured base** for the repeated-store region.

The sampling point is part of the contract. Earlier dependencies and direct writes cannot be assumed to preserve the slot’s contents. Later stores use the captured value rather than visibly reloading the slot; a later overwrite of the slot therefore does not by itself redirect those stores.

### Repeated-store region

The repeated region contains ordered groups with four effects:

- A word at displacement 84 from a scaled base receives a previously constructed four-times-scaled value.
- A byte at displacement 25 from the captured base receives the low byte of the adjusted base.
- A byte at displacement 17 from the adjusted base receives the low byte of the captured base.
- A word at displacement 52 from the captured base receives a scaled value.

The first group uses an earlier eight-times-scaled quantity for its relevant address and final word. Subsequent groups use eight times the adjusted quantity. Many subsequent groups repeat identical address and value expressions.

The visible repetition does not establish traversal over distinct elements. Exact multiplicity is **documented_only** in this contract; no collection size is inferred. Individual stores remain ordered effects even where destinations and values repeat.

### Final-region fixed-storage ordering

The final mutation region distinguishes an early captured word from later fixed-address byte rereads.

The 32-bit word at fixed address **169 is captured first**. A subsequent word read at displacement 116 from the previously constructed four-times-scaled base occurs before the byte write at fixed address **58**. That byte write uses the adjusted quantity scaled by eight, plus 48, retaining the low eight bits.

Further computed-address reads capture a word at displacement 20 from a doubled loaded base and a word at displacement four from the adjusted base. Several computed-address stores then precede the write of the captured address-169 word’s low byte to fixed address **103**.

After that write:

- A computed-address word read at displacement 68 occurs before address 103 is read as a byte for the write to fixed address **33**.
- Address 103 is read again later to supply a byte store at displacement nine from the adjusted base.
- After additional stores and a computed-address word read at displacement 84, address 103 is read once more for another byte store at that same displacement-nine destination.

The full word captured from address 169 is also used in a computed-address word store at displacement 68 from the adjusted base. It is not replaced by a later address-103 byte read.

Conversely, none of the three address-103 byte reads may be replaced unconditionally with the low byte captured from address 169. Intervening computed-address writes may alias address 103. Even the first read, used for address 33, follows an intervening computed-address word read; this distinction also matters if reads have device effects.

### Remaining mutation and final transfer

The final region captures words before overwriting some of their source locations, writes byte fragments at displacements one, eight, nine, 13, 16, and 17, and restores an earlier captured displacement-20 word into a displacement-84 destination.

A byte at displacement one receives a value derived from a signed right shift. The count is the low eight bits of the adjusted base. Counts at or above 32 remain **documented_only** without target shift semantics.

Later effects include:

- A halfword store of 101 at twice a loaded word.
- A byte store at displacement 17 from another loaded base.
- A word store of 100 at displacement 84 from that base.
- Further word reads through displacements four, 100, and 116.
- A word store through a doubled loaded value.

The final unresolved transfer receives four explicit arguments:

| Position | Established value |
|---|---|
| First | Numeric value 114. |
| Second | The word captured by a late displacement-100 read from the base whose doubled address received halfword value 101. |
| Third | The earlier captured displacement-20 word, also used to restore a displacement-84 location. |
| Fourth | The word captured by the late displacement-116 read from that same base. |

The fourth argument is not defined as the earlier word captured from fixed address 169. Equality would require additional evidence.

## Side effects

### Access widths and layout

The representation exposes byte displacements, not a verified structure layout.

| Width | Representative displacements or destinations |
|---|---|
| 8 bits | 1, 5, 8, 9, 13, 16, 17, 21, 25 |
| 16 bits | Twice-loaded-value and eight-times-loaded-value destinations |
| 32 bits | 4, 16, 20, 32, 36, 52, 64, 68, 84, 100, 116 |

Overlaps include byte displacement 21 within a word beginning at displacement 20, and byte displacements 16 and 17 within a word beginning at displacement 16. Byte order is unresolved, so partial overwrites generally do not determine a unique numeric word value.

Meaningful fixed-address effects include:

| Address | Visible access |
|---:|---|
| 1 | Byte write. |
| 5 | Byte read. |
| 21 | Byte write. |
| 33 | Byte write sourced by a later read of address 103. |
| 58 | Byte write after the address-169 capture and following displacement-116 read. |
| 84 | Word writes, including consequences of the ordinary-memory zero result. |
| 96 | Word write of 32 after stored-base sampling. |
| 103 | Byte write followed by three distinct byte reads in the final region. |
| 169 | Early final-region 32-bit word capture. |

The distinguished stored-base slot is an additional 32-bit storage input. Its sampling order is established independently of whether its storage overlaps another accessed range.

### Ownership and concurrency

No allocation, release, reference counting, lifetime extension, or ownership transfer is established. Dependency contracts are unavailable; they cannot be assumed to preserve globals, retain addresses safely, or acquire or release memory.

No lock, atomic operation, transaction, or rollback is visible. Thread safety, reentrancy, and consistency under concurrent observation are **documented_only**.

### Errors and interruption

No explicit error result, diagnostic, recovery branch, or rollback appears. Invalid targets and inaccessible memory are not converted into a defined error response. An interruption may leave all preceding effects in place.

## Dependencies

No resolved identities are supplied for the indirect dependencies. These opaque IDs identify documentary roles only:

| Opaque ID | Role | Contract status |
|---|---|---|
| F-unknown-01 | Entry-context-derived invocation | Target, implicit arguments, effects, and completion unresolved. |
| F-unknown-02 | Ordered family of subsequent invocations | Members need not be the same function or share behavior. |
| F-unknown-03 | Final four-argument transfer | Identity, effects, return behavior, and outputs unresolved. |

No preservation, purity, ownership, or successful-return guarantee is assigned to any dependency.

## Acceptance examples

These examples are conditional analytical requirements, not executed tests.

1. **Low-byte input contribution.** With identical relevant dependency behavior, third-input values 1 and 257 produce the same explicit derived forms: 16,777,216 and 32. Whole-execution equivalence is not established.

2. **Small-index boundary.** If the contributing result’s low seven bits are 127, the small index is 508 and its doubled value is 1,016.

3. **Overwritten byte event.** If the initial shifted-base displacement-nine store writes 171, that event precedes the later zero store. Checking only the final zero omits an established effect.

4. **Zero despite possible aliasing.** With ordinary memory, 32-bit bit-vector arithmetic, and no external mutation, the multiplication-derived displacement-20 read must return zero. This remains true if the intervening small-index word clear overlaps it. The displacement-52 word store cannot overlap that displacement-20 word.

5. **Late stored-base sampling.** If earlier effects change the stored-base slot from one address to another, repeated stores use the value present after the last displacement-32 constant store. Sampling before that store, or after the fixed-address-96 write, does not satisfy the specified ordering.

6. **Captured word versus reread byte.** Suppose address 169 yields a word with low byte 171. Address 103 later receives 171. Without intervening mutation or read side effects, its first reread supplies 171 to address 33. Later rereads are separate events and need not remain 171 if intervening writes overlap address 103.

7. **Mixed-width constants.** Valid destinations receive numeric 58 through one byte store, one halfword store, and one word store at their distinct computed addresses. Three identical byte writes are not equivalent.

8. **Unavailable low memory or nonreturning target.** Inaccessible required low addresses prevent any completion guarantee. A nonreturning F-unknown-03 likewise does not violate a promised normal return, because no such promise is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose, record types, and intended invariants.
- Actual calling convention and implicit use of explicit inputs.
- Dependency identities, side effects, memory ownership, and completion.
- Reliability of apparent paired results and secondary result channels.
- Target signed arithmetic, address arithmetic, and variable-shift semantics.
- Byte order, alignment requirements, and device-memory behavior.
- Executable validity and meaning of low-address accesses.
- Storage boundaries and alias relationships, including the stored-base slot.
- Exact multiplicity of the repeated-store region.
- Final target identity, meaningful outputs, and normal completion.

These uncertainties do not negate the ordinary-memory zero conclusion or permit moving the established stored-base sampling point.

## Evidence

The evidence is the supplied static representation. Comments, annotations, apparent types, and prior descriptions are not treated as authoritative behavior.

The strongest evidence concerns access widths, constants, masks, argument construction, and visible ordering. In particular, the representation distinguishes the zero-preserving intervening stores, the late 32-bit stored-base read, the early address-169 word capture, and the subsequent separate address-103 byte reads.

No tools were used and no runtime tests were performed. Acceptance examples describe conditional behavior, not observed execution.
