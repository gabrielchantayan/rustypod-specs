<!-- Trusted provenance: F-31d1bdce8a15; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-31d1bdce8a15

## Purpose

F-31d1bdce8a15 performs an ordered sequence of indirect invocations, transforms machine-width results, reads and modifies calculated-address and fixed-address storage, and ends with an unresolved indirect transfer represented as an invocation.

The supplied static representation does not establish an application-level purpose. Constants resembling character values do not establish formatting, string construction, decoding, or textual output. Repeated offsets do not establish a particular object model.

This contract distinguishes:

- **Established static evidence:** represented operations, widths, value dependencies, and ordering.
- **[INFERENCE]:** interpretations requiring additional arithmetic or machine-model assumptions.
- **documented_only:** behavior or detail not resolved by this contract.

All described effects are conditional on execution reaching them. No runtime tests have been performed.

## Interface

### Explicit inputs

| Argument position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit reference in the represented body. |
| Second | 32 bits | No explicit reference in the represented body. |
| Third | Signed 32 bits | Supplies early shifted invocation arguments; its working value is subsequently shifted left by three bits. |

The first two arguments are not proven irrelevant at the machine interface. Unresolved invocation interfaces may consume incoming machine state beyond the displayed argument positions.

### Implicit inputs

Execution also depends on:

- Incoming return-link state, used to derive the first target.
- Incoming register state used in an early six-bit left shift.
- Ordinary and secondary results of indirect invocations.
- Results represented as 64-bit quantities.
- Fixed-address storage and calculated-address storage.
- Loaded values subsequently interpreted as addresses.
- The unresolved final target.

The three explicit arguments therefore do not define a complete standalone calling contract.

### Outputs and completion

There is no explicit returned value. Observable outputs consist of indirect invocations and memory effects.

A return follows the final unresolved invocation in the representation. Normal completion requires all preceding operations to complete and the final target to return. No meaningful return-register value, success indicator, or termination guarantee is established.

### Preconditions

Execution without a target or memory-access fault requires every reached target and memory access to be valid under the actual machine environment. This includes executable targets, accessible storage, required alignment, and valid address interpretation.

No checks enforce these conditions. No buffer capacity, object type, nullability rule, or ownership convention is established for the explicit arguments.

## Behavior

### Invocation and arithmetic phase

The initial phase is a straight-line sequence of indirect invocations interleaved with arithmetic. No explicit loop or conditional branch appears.

The first target is derived from incoming return-link state plus 0xFFE bytes. Later targets are predominantly derived from embedded code-location references plus the same displacement. This does not establish that the destinations are independently callable helpers.

Results may be consumed immediately or retained across intervening invocations. Displayed argument lists include transformed values, equal argument pairs, zeros, and secondary machine results. An invocation without displayed arguments is not proven to have no machine-level inputs.

No target is assumed pure, global-preserving, memory-preserving, returning, or ownership-neutral. Repeated invocations cannot be removed or combined merely because their displayed arguments resemble one another.

Most transformations have represented 32-bit width. They include left shifts, unsigned logical right shifts, masks, scaling, and direct use of numeric results as addresses. Several represented 64-bit results are separated into low and high 32-bit portions.

Unsigned 64-bit shifts by 34, 36, and 44 bits select the high portion shifted logically right by two, four, and twelve bits respectively. These are not oversized shifts of a 32-bit operand.

The third argument contributes a three-bit left shift and later shifts of that working value by ten and 26 bits. **[INFERENCE]** Under truncating 32-bit bit-vector arithmetic, these later expressions retain the low 32 bits of the original argument shifted by 13 and 29 bits. They are not unbounded multiplication.

Masks select fields before further transformation. Masking before a shift is not generally interchangeable with masking afterward. Likewise, left shifting and subsequently right shifting does not recover discarded high bits.

Portable overflow semantics are not established for represented signed left shifts, multiplication, or address arithmetic. Numerical examples requiring truncation state that assumption explicitly.

### Initial direct accesses

The final represented 64-bit result of the invocation phase supplies two separately used portions. Earlier retained results supply additional values and bases.

The initial direct-access ordering includes:

- A 32-bit store at offset 0x14 from the high result portion.
- A byte store at offset 9 from a separately retained base, containing the low byte of the high result portion.
- A 32-bit store at offset 0x54 from the low result portion multiplied by 256.
- Another byte store to that same offset-9 destination, now containing a different retained value’s low byte.

The next accesses clear a calculated 32-bit field at offset 0x14, read a byte at offset 5 from that base, clear a separate doubled-base field at offset 0x14, and write the zero-extended byte to the first base’s 32-bit field at offset 0x34. The first cleared field is then loaded. Another clear through the doubled base follows, together with two distinct byte writes at its offset 0x15.

Neither the later load nor either repeated byte write may be replaced solely from the preceding clear. Calculated addresses may overlap.

### Fixed-address accesses within the early memory phase

A read at fixed address 0x05 occurs after those two offset-0x15 byte writes and before the byte write at fixed address 1. The value read at 0x05 is converted to an unsigned 32-bit address value and retained for subsequent calculated accesses. The declaration fixing the original load width is not included in the supplied material.

The byte written at address 1 is the low byte of the doubled base. Subsequent calculated effects include:

- A byte write at offset 0x15 from the address value obtained at 0x05.
- A byte write to another base’s offset 0x11.
- A 32-bit store through an address obtained from the earlier offset-0x14 load.
- A zero byte overwrite at that offset-0x11 destination.

A 32-bit field at offset 0x54 from the address value obtained at 0x05 is then read and retained before that field is overwritten. This captured word becomes a seed for later scaling and stores.

The fixed 32-bit write at address 0x54 has a specific position: it follows a calculated 32-bit write to the doubled base’s offset 0x14 and precedes two calculated 32-bit writes of the seed word to that same field. The fixed write contains the seed shifted left by one bit.

Later, two byte stores to the doubled base’s offset 0x15 contain the low byte of four times the seed. The byte write at fixed address 0x15 follows those two writes and precedes a calculated 32-bit store at offset 0x54 from the fourfold seed base. The fixed byte receives the low byte of the doubled base.

These fixed accesses are not interchangeable with calculated accesses. Their relative positions remain significant when calculated destinations overlap the fixed locations.

### Calculated layouts and intermediate effects

Recurring offsets describe access patterns, not proven structures.

| Offset | Represented access |
|---|---|
| 0x01 | Byte store |
| 0x04 | 32-bit load; also a byte load in one later overwrite cluster |
| 0x05 | Byte load |
| 0x08, 0x09, 0x0D | Byte stores |
| 0x10 | Byte and 32-bit stores |
| 0x11 | Byte stores |
| 0x14 | 32-bit loads and stores |
| 0x15, 0x19 | Byte stores |
| 0x20, 0x24 | 32-bit stores |
| 0x34, 0x44, 0x54 | 32-bit accesses |
| 0x40, 0x64, 0x74 | 32-bit loads, sometimes supplying later addresses |

Offset 0x15 overlaps a 32-bit field beginning at 0x14. Byte writes at offsets 0x10 and 0x11 can overlap a 32-bit field beginning at 0x10. Endianness is not independently established, so partial writes do not determine a combined numerical word without an additional assumption.

The intermediate region links scaled seed values, doubled loaded values, and pointer-derived fields. It includes self-valued stores, repeated stores, and repeated byte loads at offset 5. A loaded byte can itself become the base for a 32-bit load at offset 0x74.

A constant cluster stores numeric 0x3A at separately calculated destinations using eight-, 16-, and 32-bit widths. The eight-bit store precedes the 16-bit store, which precedes the 32-bit store.

Another destination, at offset 0x20 from an adjusted fourfold base, receives these 32-bit values in order:

1. 0x20.
2. 0x2F.
3. The zero-extended byte freshly read at that base plus 4.
4. 0x20.

The third value is a memory dependency, not a retained constant.

### Fixed global capture and repeated publication region

After the final 0x20 overwrite above, the representation reads a 32-bit value at fixed address 0x087FBBBC and retains it as the publication base. It then writes 32-bit value 0x20 at fixed address 0x60. The repeated publication stores follow both operations.

Thus the supplied representation places the global read **before**, not after, the fixed write at 0x60. Reversing them is not supported. If these locations overlap through the actual address mapping, their order affects the captured publication base.

For the remaining description:

- The **seed** is the earlier captured 32-bit offset-0x54 word.
- The **adjusted value** is four times the seed minus decimal 150.
- The **adjusted base** is four times the adjusted value.
- The **payload base** is four times a later word obtained through a byte-based offset-0x74 load.
- The **late doubled base** is twice that later word.

These are descriptive relationships, not additional runtime inputs. Arithmetic overflow remains subject to the limits stated above.

Each publication group contains, in order:

- A calculated 32-bit store of the payload base at offset 0x54.
- A byte store of the adjusted base’s low byte at publication base plus 0x19.
- A byte store of the publication base’s low byte at adjusted base plus 0x11.
- A 32-bit store at publication base plus 0x34.

The first group uses eight times the seed for its scaled destination and final stored relationship. Subsequent groups use eight times the adjusted value.

The publication base and byte values are retained, not reloaded for every group. Aliasing writes can change their original storage without changing those retained values.

This is a finite, explicitly repeated region, not an established loop. Exact multiplicity is **documented_only** in this contract; no independently verified count is claimed. Each represented write remains distinct.

### Post-publication captures and fixed byte updates

Immediately after the repeated region:

- A 32-bit word is captured from fixed address 0xA9.
- A 32-bit field is captured from payload base plus 0x74.
- A byte is written at fixed address 0x3A.
- A 32-bit field is captured from late doubled base plus 0x14.
- A 32-bit field is captured from adjusted base plus 4.

The fixed byte at 0x3A is the low-eight-bit result of eight times the adjusted value plus 0x30. Its write occurs between the payload-field capture and the other two captures. It can therefore affect those later captures if their addresses overlap it.

Calculated byte writes then target late doubled base plus 0x0D and adjusted base plus 8, 0x0D, and 9. The offset-8 byte is 0x6C. A self-valued 32-bit store at late doubled base plus 0x54 follows.

Only then is the low byte of the retained word captured from 0xA9 written at fixed address 0x67.

A 32-bit load from late doubled base plus 0x44 intervenes before the byte at 0x67 is read and written to fixed address 0x21. The 0x21 write is therefore specified as a fresh memory read from 0x67, not as another direct extraction from the retained 0xA9 word.

### Later reads of 0x67 and final transfer

Further calculated writes and loads occur before each of two additional reads of the byte at 0x67.

Before the first additional read, the function performs byte and word writes involving adjusted base, publication base, and late doubled base. These include a byte at adjusted base plus 0x10, two different word writes at publication base plus 0x54, a byte at late doubled base plus 1, a load from adjusted base plus 0x40, and word transfers involving offsets 0x74, 0x10, and 0x44.

The first additional read from 0x67 supplies a byte store at adjusted base plus 9.

Between that store and the second additional read, the function writes a byte at late doubled base plus 9, writes the retained full 0xA9 word at adjusted base plus 0x44, and loads a word from adjusted base plus 0x54. The second additional read from 0x67 supplies another byte store at adjusted base plus 9.

Each read observes memory at its own position. Neither later byte is guaranteed equal to the original low byte captured from 0xA9 or to the byte copied to 0x21. Intervening calculated writes may overlap 0x67.

The remaining effects include:

- Publishing the low byte of the captured payload field through several calculated byte destinations.
- Restoring the captured late-doubled-base offset-0x14 word into its offset-0x54 field.
- Loading adjusted base plus 0x14 before further byte and word writes.
- A 16-bit store of 0x65 through twice a newly loaded base.
- A 32-bit store of decimal 100 at offset 0x54 from another loaded base.
- Additional pointer-derived loads and 32-bit stores before the final transfer.

The final unresolved invocation has four displayed 32-bit positions:

1. Constant 0x72.
2. A recently loaded offset-0x64 word.
3. The retained word captured earlier from late doubled base plus 0x14.
4. A recently loaded offset-0x74 word.

The second and fourth values come from the base loaded late from late doubled base plus 4. The final target and its effects remain unresolved.

## Side effects

Direct accesses include eight-, 16-, and 32-bit operations. Meaningful fixed locations include 1, 0x05, 0x15, 0x21, 0x3A, 0x54, 0x60, 0x67, 0xA9, and 0x087FBBBC, with ordering described above.

Calculated bases are not proven distinct from one another or from fixed storage. Low-address accesses must not be ignored, reclassified as ordinary application globals, or identified as device accesses without further evidence.

No allocation, release, retain, or ownership transfer is positively identified. Unresolved targets may nevertheless perform such operations. No lifetime guarantee follows merely because a value is retained and later used as an address.

No explicit validation, error result, handler, rollback, retry, or cleanup path appears. A failure may occur after earlier effects; transactionality is not established.

Atomicity, synchronization, thread safety, reentrancy, and memory attributes are **documented_only**.

## Dependencies

The only supplied opaque function identity is **F-31d1bdce8a15** itself. No distinct dependency has a resolved opaque ID.

The indirect destinations and final transfer remain unidentified dependencies. Assigning library meanings or fabricating additional resolved IDs would exceed the evidence. The subject’s identity alone does not establish recursion.

Required dependency information includes machine argument/result conventions, secondary results, memory and global effects, ownership effects, lifetime effects, and return or fault behavior. None may be assumed from a displayed invocation alone.

## Acceptance examples

These are conditional static examples, not executed tests.

### Fixed-read capture

If the read at 0x05 yields an address value, the subsequent byte write at address 1 must not be moved before that read. Later accesses through the captured value use the capture, not a fresh read of 0x05.

### Global capture before 0x60

If the read at 0x087FBBBC yields publication base 0x2000, that retained base supplies the following publication groups. The 32-bit 0x20 write at 0x60 occurs after the capture and before the groups.

### Retained word versus fresh byte

Suppose the captured word at 0xA9 has low byte 0xAB. The explicit fixed write to 0x67 publishes 0xAB. If a later calculated store changes 0x67 to 0xCD before one of its later reads, the corresponding byte store at adjusted base plus 9 must receive 0xCD, not cached 0xAB.

### Independent later reads

If 0x67 contains 0x11 at the first later read and an intervening aliased write changes it to 0x22, the two calculated offset-9 stores receive 0x11 and 0x22 respectively. Combining the reads is not equivalent.

### Byte widening

A reached byte load yielding 0xE1 supplies unsigned 32-bit value 225 when widened. It does not supply a sign-extended negative value.

### Ordered overwrites

For observable storage, the offset-0x20 write trace retains 0x20, 0x2F, the freshly widened byte, and 0x20. Keeping only the final write is insufficient.

### Width-sensitive arithmetic

Under truncating 32-bit arithmetic, a left shift of 0x12345678 by eight followed by a logical right shift by eight yields 0x00345678. Discarded bits are not recovered.

### Partial execution

If an indirect dependency does not return, subsequent operations need not occur. Earlier effects are not required to be undone. A zero third argument does not eliminate invocations or fixed-address effects.

## Uncertainties

The following remain **documented_only**:

- Application intent, valid input domain, and complete entry-state requirements.
- Completeness of represented invocation interfaces.
- Whether represented paired results form one scalar or separate machine results.
- Original storage width of the fixed read at 0x05.
- Endianness, alignment rules, memory attributes, and address mapping.
- Signed overflow and address-conversion behavior.
- Behavior of the late right shift whose count comes from an address’s low byte, including out-of-range counts.
- Object identity, validity, and lifetime.
- Exact multiplicity of the explicitly repeated publication region.
- Unresolved dependency effects and ownership rules.
- Final target and return behavior.

These uncertainties do not make the explicit fixed accesses or fresh reads of 0x67 unresolved dependency semantics. Their represented positions and value dependencies are established.

## Evidence

Evidence consists solely of the supplied static representation for F-31d1bdce8a15. Comments, symbolic names, prior prose, and reviewer feedback are not independent behavioral authority.

The representation supports the described widths, invocation dependencies, fixed and calculated accesses, capture points, repeated stores, fresh reads of 0x67, and unresolved final transfer. In particular, it places the read at 0x087FBBBC before the fixed write at 0x60.

No implementation access, instruction-level validation, runtime execution, smoke run, or test results were available. Acceptance examples are conditional deductions from static evidence, not observed execution.
