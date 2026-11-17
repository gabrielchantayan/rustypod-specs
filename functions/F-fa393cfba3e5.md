<!-- Trusted provenance: F-fa393cfba3e5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fa393cfba3e5

## Purpose

F-fa393cfba3e5 performs an ordered sequence of indirect calls, transforms captured results, modifies memory through fixed and calculated addresses, and finishes with an unresolved four-argument indirect transfer.

The supplied static representation does not establish an application-level purpose. In particular, it does not establish initialization, text formatting, container maintenance, or a communication protocol. Character-like constants and recurring offsets are insufficient to establish those interpretations.

This contract distinguishes:

- **Established static evidence:** represented calls, value relationships, access widths, and ordering.
- **Inferred intent:** possible interpretations that are not compatibility requirements.
- **documented_only:** unresolved behavior for which no executable guarantee is supplied.

All behavior below is conditional on execution reaching the relevant operation. No runtime tests have been performed.

## Interface

### Inputs

The represented interface has no explicit parameters. Its behavior nevertheless depends on implicit execution state:

- An incoming control-link value, used to calculate the first indirect destination by adding 0xffe.
- An incoming 32-bit register value used in early argument transformations.
- Primary and secondary results associated with indirect calls.
- Captured 64-bit composite results, subsequently separated or shifted.
- Bytes and 32-bit words read from fixed and calculated addresses.
- An unresolved terminal destination.

The complete calling convention, required entry context, and validity of secondary register results are **documented_only**. An ordinary parameterless invocation is not established to supply sufficient state.

### Outputs and completion

No explicit scalar return value, success indicator, returned allocation, or output-buffer interface is established. Observable outputs are the indirect transfers and memory-access history.

The terminal transfer receives four arguments:

| Position | Established value |
|---|---|
| First | 0x72 |
| Second | A late 32-bit word loaded at offset 0x64 from a memory-derived base |
| Third | A previously captured 32-bit word from offset 0x14 of a doubled memory-derived base |
| Fourth | A late 32-bit word loaded at offset 0x74 from the same base used for the second argument |

The second argument is also used in direct memory mutations before dispatch. The third argument remains the earlier captured value; it is not a fresh read at dispatch.

A return is represented after the terminal transfer. Whether the actual destination returns, and whether control then reaches the original caller, remain **documented_only**.

### Widths, layout, and arithmetic

| Quantity or access | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access | 32 bits |
| Composite call result | 64 bits |
| Address-bearing integer quantities | 32 bits |

Offsets are byte offsets. Unsigned byte loads used as word values are zero-extended. Byte and halfword stores retain only the low eight and sixteen bits, respectively.

Storage is not divided into independent logical fields merely because different offsets are used. For example, a word beginning at offset 0x14 overlaps a byte at offset 0x15.

Unsigned 32-bit transformations must retain their width boundaries. Shifting a 32-bit value left and then right does not restore bits discarded by the first operation. Shifting a 64-bit composite result before narrowing is distinct from shifting its lower word.

Some address calculations and transformations are represented as signed arithmetic. Machine-level behavior for signed overflow, negative operands, and overflowing signed left shifts is **documented_only**. This contract neither substitutes unbounded arithmetic nor promises portable wraparound for those expressions.

Endianness, alignment requirements, address-space mapping, and device-memory behavior are unresolved.

### Preconditions and ownership

Uninterrupted execution requires every reached indirect destination to be executable and every reached memory access to satisfy the target’s access and alignment requirements. No visible validation enforces these conditions.

Small, zero-derived, unaligned, and memory-derived addresses are not rejected by any represented local branch.

No allocation ownership, borrowing discipline, lifetime guarantee, or release obligation is established. A word used as an address is not thereby proven to identify an allocated object.

## Behavior

### Indirect-call phase

The function begins with the control-link-derived transfer and continues through a long straight-line sequence of indirect calls. Most later destinations are calculated from embedded address-like values with the same added displacement.

The represented order of calls is significant. Calls with unused primary results remain part of the behavior: they may affect memory, external state, secondary results, or subsequent control flow. Similar argument patterns do not establish that two destinations identify the same dependency.

Visible argument relationships include:

- Left-shifted primary 32-bit results.
- Separate lower and upper words of composite results.
- Shifted secondary register results.
- A captured value supplied in multiple argument positions.
- Explicit zeros.
- Masked fields shifted left or logically right.
- Earlier captured results retained across intervening calls.

A call with no displayed arguments is not proof that the actual dependency consumes no incoming register state.

There is no represented conditional branch selecting among the calls. Reaching a later call nevertheless requires earlier dependencies to return sufficiently for execution to continue.

### Arithmetic boundaries

Masks and operation widths are behavioral requirements, not optional simplifications. The prefix contains masks ranging from a single bit through 24 bits, including 11-, 12-, 13-, 14-, 16-, 17-, 18-, 19-, 20-, 21-, and 22-bit fields.

Representative established bounds include:

- A value masked to 24 bits and shifted logically right by eight lies between zero and 65,535.
- A value masked to 20 bits and shifted logically right by ten lies between zero and 1,023.
- A value masked to 19 bits and shifted left by two is at most 0x1ffffc.
- A 64-bit result shifted right by 34, 36, or 44 bits draws from its upper portion, not its lower word.

One late signed right shift uses the low eight bits of an address-derived quantity as its count. The supplied representation does not establish behavior for counts outside 0–31 or the machine rule for negative shifted operands. These cases remain **documented_only**.

### Initial direct-memory mutations

The direct-memory phase uses several captured roles:

- **Encoded base:** a returned value multiplied by 2²⁵.
- **Earlier calculated base:** a preceding result shifted left by 25 bits.
- **Small index:** the result of an eight-bit left shift, masked to its low 15 bits, then shifted logically right by six.
- **Index base:** twice the small index.
- **Shifted payload:** a preceding returned value after an eight-bit left shift.

These are descriptive roles, not object types. The mask establishes a small-index bound of 511 and an index-base bound of 1,022. The bounds do not establish valid mapping or word alignment.

The opening mutations first publish the earlier calculated base as a word at offset 0x14 from the upper word of the last composite result. A byte at offset nine from the earlier calculated base then receives the low byte of that composite result’s upper word. Between this byte store and the next store to the same byte destination, the encoded base is written as a word at offset 0x54 from an address obtained by scaling the composite result’s lower word by 256.

The second store to the offset-nine byte destination takes its value from the low byte of the captured earlier calculated base. The two byte stores therefore have different value sources. Their stored bytes are **not guaranteed to differ**, and the second store is not guaranteed to change the stored contents.

Subsequent operations clear encoded-base offset 0x14, capture a byte at encoded-base offset five, and clear index-base offset 0x14. The captured byte is published as a zero-extended word at encoded-base offset 0x34. A word is then read from encoded-base offset 0x14, index-base offset 0x14 is cleared again, and the low byte of the captured word is written twice consecutively at index-base offset 0x15.

The captured word must not universally be assumed zero. The intervening index-base store can alias its location.

### Memory-derived links and scaling

A byte read from absolute address 0x05 is zero-extended and used as an address base. The function also writes a byte to absolute address 0x01.

The word at offset 0x54 of the byte-derived base is captured before replacement. This captured word supplies later scaling and addressing. An earlier captured word replaces it, while additional stores publish the small index and shifted payload through other bases.

The phase includes these observable relationships:

- Words loaded from offset 0x74 become bases for offset-0x54 writes.
- A captured base is stored at its own offset 0x24, with repeated stores to that destination.
- Byte destinations receive low bytes of captured bases.
- Index-base offset 0x14 receives successive word values, including consecutive identical stores.
- Absolute address 0x54 receives a word derived by doubling the captured link value.
- Absolute address 0x15 receives a byte derived from the index base.
- The captured link value is multiplied by four and eight for subsequent addresses and stored values.
- Words at offsets 0x14, 0x34, 0x44, 0x54, 0x64, and 0x74 are read or replaced through these calculated bases.

Further byte reads at offset five supply unsigned bases for word reads at offset 0x74. Such bases may be within the first 256 address values. No null or small-address handling is represented.

These relationships may suggest linking structures, but no list, tree, descriptor, or record abstraction is established. Repeated reads remain reads; repeated writes remain writes. Captured values must not be substituted for later reads without establishing equivalence under aliasing and target memory behavior.

### Constant stores and repeated publication

A later group stores 0x3a through three access widths:

- An 8-bit store at four times a memory-derived value.
- A 16-bit store at eight times that value.
- A 32-bit store at offset 0x20 from four times that value.

A separate adjusted base is formed by subtracting 0x96 from a calculated base and scaling the result by four. Its offset-0x20 word destination receives 0x20, then 0x2f, then a zero-extended byte read from offset four of that adjusted base, and finally 0x20 again. The byte read occurs after the first two word stores and before its value is published.

**The fixed-address word at 0x087fbbbc supplying the loaded base is read next, before the word store of 0x20 to absolute address 0x60.** The loaded base is a captured result of that earlier read. Neither unknown memory behavior nor a final-state interpretation permits reversing these accesses or replacing the captured value with a post-store reread.

After the absolute-address store, a long contiguous publication run repeatedly relates:

- An offset-0x54 word destination at a scaled address.
- An offset-0x19 byte destination relative to the captured loaded base.
- An offset-0x11 byte destination relative to the adjusted base.
- An offset-0x34 word destination relative to the captured loaded base.

Within each complete recurring group, the offset-0x54 word store precedes the two byte stores, and the offset-0x34 word store follows them. The byte values are low-byte captures of the associated bases.

The initial group uses the original link-derived eightfold scale for its scaled destination and final word value. Following groups use the adjusted-base-related eightfold scale. The later groups repeat the same represented addresses and values rather than advancing through distinct elements. They contain no intervening calls or explicit reads.

The exact repetition cardinality is not enumerated in this contract and remains **documented_only** here. No guessed count or single final-state-equivalent group is authorized.

### Late memory effects and dispatch

After the publication run, the function captures a word from absolute address 0xa9 and another word from offset 0x74 of a scaled base. It then writes a byte at absolute address 0x3a, derived from the low-byte arithmetic of an eightfold scaled quantity plus 0x30.

Before subsequent mutations, it captures the word later used as the terminal third argument from offset 0x14 of a doubled memory-derived base, and another word from offset four of the adjusted base.

Late mutations include:

- A byte value of 0x6c at adjusted-base offset eight.
- Byte stores at offsets nine, 0x0d, 0x10, and 0x11 through calculated bases.
- A self-base word store at offset 0x54, later replaced by the earlier captured offset-0x14 word.
- Copying the low byte of the captured absolute-0xa9 word to absolute address 0x67, then copying the byte read from 0x67 to absolute address 0x21.
- Reading an offset-0x44 word before publishing its low byte and a variable-count shifted form.
- Two stores to the loaded-base offset-0x54 destination, first using a captured word and later the adjusted base.
- Capturing an adjusted-base offset-0x40 word before publishing it at offset 0x10.
- Capturing an adjusted-base offset-0x54 word and using it as another address base.
- Repeated reads of absolute address 0x67 for later byte stores.
- A 16-bit store of 0x65 at twice a memory-derived value.
- A 32-bit store of decimal 100 at offset 0x54 of another memory-derived base.

Near dispatch, offset-0x64 and offset-0x74 words are captured from the base used for terminal arguments two and four. Further stores publish linked values through offset 0x54 and through twice the second argument’s value before the terminal transfer.

All overlapping byte, halfword, and word effects depend on access order and target byte layout.

## Side effects

Established fixed-address writes include absolute addresses 0x01, 0x15, 0x21, 0x3a, 0x54, 0x60, and 0x67. Fixed-address reads include an unsigned byte at 0x05 and words at 0xa9 and 0x087fbbbc, together with later byte reads at 0x67.

Calculated-address writes may overlap these locations or each other. Storage names do not establish disjoint global variables; address and access width determine overlap.

Indirect dependencies may have additional effects. Nothing supplied establishes that they preserve globals, avoid allocation, retain memory, release memory, or leave external state unchanged.

There is no visible transaction, rollback, lock, or synchronization boundary. Faults or nonreturning dependencies may leave a partially executed access history.

No local error-reporting branch is represented. Recovery from inaccessible memory, invalid destinations, alignment failures, arithmetic hazards, or dependency failures is **documented_only**.

## Dependencies

F-fa393cfba3e5 is the only resolved opaque function ID supplied. No opaque IDs are available for the indirect dependencies; assigning identities would be unsupported.

| Dependency class | Established role | Unresolved contract |
|---|---|---|
| Entry-state-derived destination | First transfer | Identity, ABI, effects, return behavior |
| Embedded-value-derived destinations | Ordered call prefix | Identity, arguments beyond those represented, result meaning |
| Secondary-result producers | Supply later register-derived values | Validity and preservation rules |
| Terminal destination | Receives four arguments | Identity, ownership effects, return behavior |

No dependency is established to be pure, infallible, or memory-preserving.

## Acceptance examples

These are conditional compatibility criteria, not executed tests.

### Unsigned propagation and truncation

If an offset-five unsigned byte read returns 0xfe, the corresponding zero-extended word publication contains 254. A negative sign-extended word is incompatible.

If a byte store takes its value from a captured word of 0x12345678, it writes 0x78, not four bytes.

### Equal bytes from distinct sources

If the last composite result’s upper word and the captured earlier calculated base both have low byte 0x00, both opening offset-nine byte stores write 0x00. The second access remains required even though it does not change ordinary memory contents. Requiring those stored bytes to differ is incompatible.

### Fixed-base read before absolute publication

If the fixed-address read at 0x087fbbbc returns a base, subsequent publication groups use that captured base even if the later word store at 0x60 affects what a fresh fixed-address read would return. The fixed read must precede the absolute store; rereading afterward is not equivalent.

### Overlapping storage

If index-base offset 0x14 is ordinary writable memory, a zero word store there followed by a byte store of 0xab at offset 0x15 modifies part of the same word. The resulting word depends on endianness. Independent logical-field treatment is incompatible.

### Access history

Where accesses are externally observable, two identical consecutive byte writes remain two writes. Likewise, the offset-0x20 sequence 0x20, 0x2f, captured byte, 0x20 cannot be collapsed into its final value.

### Small addresses and terminal transfer

If the small index is zero, its doubled base is zero, and the represented offset-0x14 and offset-0x15 accesses are not skipped.

If terminal dispatch is reached, its first argument is 0x72 and its third argument is the earlier captured word. Normal return remains conditional on the unresolved destination returning.

## Uncertainties

The following remain **documented_only**:

- Application purpose and intended data model.
- Complete entry ABI and required incoming execution state.
- Indirect dependency identities and contracts.
- Secondary register-result validity and preservation.
- Signed overflow, signed shifts, and exceptional shift counts.
- Endianness, alignment, mapping, permissions, and device-memory semantics.
- Ownership, allocation, retention, and release behavior.
- Exact call-prefix enumeration and repeated-publication cardinality within this contract.
- Fault handling and partial-execution outcomes.
- Whether the terminal transfer returns normally.

These limitations do not weaken established access widths, value sources, or read-before-write ordering.

## Evidence

This contract uses only the supplied static representation of F-fa393cfba3e5. Prior prose, comments, and review feedback are not independent proof of behavior.

The represented operations establish the implicit-state-dependent entry, indirect-call prefix, arithmetic widths and masks, overlapping memory accesses, captured-value relationships, repeated stores, and final four-argument transfer.

In particular, the representation places the fixed-base word read before the absolute-0x60 word store, and supplies the two opening offset-nine byte stores from distinct captured quantities without an inequality condition.

Linking or formatting intent is inference, not an acceptance requirement. No dependency implementation, machine-instruction validation, runtime observation, or test result was supplied or produced.
