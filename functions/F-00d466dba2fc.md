<!-- Trusted provenance: F-00d466dba2fc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-00d466dba2fc

## Purpose

F-00d466dba2fc performs an ordered sequence of indirect calls, propagates selected returned bit patterns, reads and modifies fixed and computed memory locations, and finishes with an unresolved indirect control transfer.

The supplied static representation establishes these operations, but not their application-level purpose. Repeated offsets do not establish an object model, and character-like constants do not establish text formatting or serialization.

This contract distinguishes:

- **Established static evidence:** operations, widths, constants, and ordering visible in the supplied representation.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior for which this contract does not invent an executable rule.

No runtime tests have been performed.

## Interface

### Invocation and completion

The represented interface declares no parameters and no return value. Invocation nevertheless depends on incoming machine state:

- An incoming return-link value contributes to the first indirect call target.
- An incoming 32-bit register value supplies an early argument, shifted left by seven and duplicated in two argument positions.
- Secondary result-register contents contribute to later calls without complete dependency signatures.

There is no established application-level success result, error code, output buffer, output length, or ownership token.

The final transfer has four represented arguments. A return follows it in the supplied representation, but whether the target returns normally is **documented_only**.

### Widths and arithmetic

| Value or access | Represented width | Contract significance |
|---|---:|---|
| Byte | 8 bits | Loads used as unsigned words are zero-extended; stores retain the low eight bits |
| Halfword | 16 bits | Stores affect two bytes |
| Word | 32 bits | Loads and stores affect four bytes |
| Ordinary arithmetic values | 32 bits | Masks, shifts, scaling, offsets, and address-like calculations |
| Selected combined call results | 64 bits | Subsequently separated into low and high 32-bit portions |

Combined results do not establish a public 64-bit dependency interface. They show that two word-sized portions are available in the representation.

Explicitly unsigned right shifts are logical. A late right shift instead has a signed operand. Signed overflow, signed left shifts, pointer conversion, address wrapping, and out-of-range shift counts remain **documented_only**. Where examples use modular 32-bit arithmetic, that interpretation is stated explicitly.

### Memory requirements

Computed values are used directly as addresses or bases. Visible accesses include:

- Byte offsets 1, 5, 8, 9, 0x0d, 0x10, 0x11, 0x15, and 0x19.
- Word offsets 4, 0x10, 0x14, 0x20, 0x24, 0x34, 0x40, 0x44, 0x54, 0x64, and 0x74.
- Halfword stores at twice or eight times loaded values.
- Fixed-address byte and word storage.

These offsets do not establish a shared record definition. A byte at offset 0x15 overlaps a word beginning at offset 0x14; a byte at offset 0x10 overlaps a word beginning there. Computed regions may also alias one another or fixed storage.

Successful traversal requires all reached targets and accesses to be supported by the execution environment. The function does not validate executable targets, mapping, alignment, permissions, or object lifetimes. Zero-derived and small addresses are not rejected before use.

## Behavior

### 1. Indirect-call sequence

The first target is derived from the incoming return-link value plus 0xffe. Later targets are derived from represented code-location values plus that displacement. Their executable identities are unresolved.

The call sequence is straight-line in the supplied representation and precedes the direct-memory phase. Its observable properties include:

- Calls with no represented arguments interspersed with calls carrying one to four arguments.
- Explicit zero arguments, including repeated pairs of zeros.
- Word results retained across intervening calls.
- Combined results whose low and high words are forwarded separately.
- Secondary result-register values used without complete signatures.

A call with no represented arguments may still inspect machine registers or shared state. An ignored represented result does not make a call dispensable. Unknown targets must not be assumed to preserve globals, retain memory validity, or return normally.

### 2. Bit-pattern propagation

The early implicit input is shifted left by seven and passed twice. Another early returned word contributes its low eleven bits shifted left by ten.

A later returned word contributes several distinct fields:

- Its value shifted left by eight.
- Bits 4 through 23 shifted left by thirteen.
- Bits 19 through 23 extracted as a five-bit quantity.
- The low bit of that five-bit quantity placed at bit 16 on multiple occasions.

That five-bit quantity also participates in shifts by three and thirty-one. These transformations must not be replaced by an assumed numeric encoding.

Combined results commonly supply a low word shifted left by eight and an independently forwarded high word. Other arguments use shifts by fourteen, fifteen, sixteen, twenty-two, twenty-three, twenty-four, or twenty-five. Some transformations are applied successively to already-shifted values.

Several working values are explicitly set to zero before later shifts and masks. Under ordinary fixed-width bit-vector interpretation, those transformations remain zero. This says nothing about whether intervening dependencies preserve shared state.

Two late chains constrain address-like values under a 32-bit interpretation:

- Shifting a returned word left by eight, retaining the low fifteen bits, and logically shifting right by six produces a multiple of four from zero through 0x1fc. Twice that result is a multiple of eight from zero through 0x3f8.
- Shifting a word left by sixteen and then extracting from its low sixteen bits produces zero.

These bounds do not establish that the resulting addresses are mapped or usable.

### 3. Entry into direct-memory effects

The following descriptive roles distinguish retained values without asserting structure types:

| Role | Derivation |
|---|---|
| Shifted base | An earlier call result multiplied by 0x02000000 |
| Compact extraction | The bounded value from zero through 0x1fc described above |
| Compact base | Twice the compact extraction |
| Packed base | An earlier combined result’s low word shifted left by twenty-five |
| Saved shifted word | A separate earlier result shifted left by eight |

After the final combined result, the packed-base value is written as a word at the final high word plus 0x14. The high word’s low byte is then written at packed base plus 9. Next, the shifted base is stored at the final low word scaled by 0x100, plus 0x54. Only afterward is packed base plus 9 overwritten with the packed-base value’s low byte.

The shifted-base word at offset 0x14 is cleared before a byte is read at its offset 5. The compact-base word at offset 0x14 is then cleared, and the captured byte is zero-extended into the shifted-base word at offset 0x34. A fresh word read follows at shifted base plus 0x14.

That read must not be replaced by an assumed zero: an intervening access uses another computed base, and aliasing and memory-observation behavior are unresolved. The compact-base word is cleared again before two identical byte writes at compact base plus 0x15, each using the captured word’s low byte.

A byte read from fixed address 5 is zero-extended and used as another base. Fixed address 1 receives the compact base’s low byte. Subsequent effects include byte writes through the byte-derived base and the shifted base, a word write through the captured word at offset 0x54, and a read of the byte-derived base’s offset-0x54 word before that field is overwritten. The saved value from this read is the **major base** used below.

### 4. Linked and scaled updates

The next region combines the major base, compact base, loaded links, and scales of two, four, and eight.

Words at offset 0x74 supply further bases. One expression uses the compact-base offset-0x74 word both as a destination base and as a stored value; the representation contains two reads of that field. They must not be silently reduced to one read without a memory-semantics justification.

Repeated writes include:

- The major base written at its own offset 0x24.
- Low-byte portions of the major base written through a loaded link at offset 9.
- Consecutive word writes at compact base plus 0x14.
- Consecutive byte writes at compact base plus 0x15.
- Successive replacements of words at scaled-base offsets 0x44 and 0x54.

Fixed address 0x54 receives twice the major base, and fixed address 0x15 receives the compact base’s low byte.

Further links arise from word offsets 4 and 0x64. A byte at offset 5 of a computed region is zero-extended and used as the base for a word read at offset 0x74. That byte is read again later, after intervening writes; the second read is not necessarily interchangeable with the first.

Values stored in this region include four and eight times the major base, twice a loaded link, self-referential base values, and low-byte portions of those values. Repeated offsets are access locations, not independently validated fields. Aliasing and overlapping widths can connect earlier stores to later reads.

### 5. Literal writes and repeated publication

A loaded link supplies three literal destinations:

- One byte containing 0x3a at four times the link.
- One halfword containing 0x3a at eight times the link.
- One word containing 0x3a at four times the link plus 0x20.

The subtraction used below occurs between the byte and halfword stores, but introduces no additional memory access.

For this region:

- **Adjusted intermediate** means four times the major base, minus 0x96.
- **Adjusted base** means four times the adjusted intermediate.
- **Publication value** means four times the loaded link.
- **Publication base** means the word captured from fixed address 0x087fbbbc.

At adjusted base plus 0x20, four word stores occur in order: 0x20, 0x2f, a zero-extended byte read from adjusted base plus 4, and 0x20. The byte read occurs after the 0x2f store and before the store of its captured value.

**The publication base is then loaded from fixed address 0x087fbbbc before the word containing 0x20 is written to fixed address 0x60. The repeated-publication stores begin only after that fixed-address write.**

This read-before-write ordering is part of the contract. The publication base is a captured value, not a fresh read after the write to 0x60. Neither unresolved aliasing nor unknown memory-observation behavior justifies reversing these accesses.

The publication region consists of finite, explicitly repeated groups. Within each group, the memory effects retain this order:

| Position | Effect |
|---|---|
| First | Word store of the publication value at a scaled destination plus 0x54 |
| Second | Byte store of the adjusted base’s low byte at publication base plus 0x19 |
| Third | Byte store of the publication base’s low byte at adjusted base plus 0x11 |
| Fourth | Word store of the scaled destination at publication base plus 0x34 |

The initial scaled destination is eight times the major base. Subsequent groups use eight times the adjusted intermediate. The captured publication base is reused throughout; no reload of its fixed source is represented within these groups.

The exact repetition count is **documented_only** in this contract. The finite repeated trace is established; a single coalesced final-state update is not established as equivalent.

### 6. Final rearrangement and transfer

After publication, a word is captured from fixed address 0xa9, then another from publication value plus 0x74. Fixed byte address 0x3a receives the low-byte result of eight times the adjusted intermediate plus 0x30. Additional words are captured from a doubled-link region at offset 0x14 and from adjusted base plus 4 before later stores.

Late effects include:

- Byte 0x6c at adjusted base plus 8.
- Low-byte portions of retained bases at offsets 9 and 0x0d.
- The low byte of the fixed-address word copied to fixed byte address 0x67, then read from there for further writes, including fixed address 0x21.
- A loaded word written at publication base plus 0x54, followed later by the adjusted base at that same field.
- A byte at doubled-link base plus 1 derived from a signed right shift whose count is the adjusted base’s low byte.
- A word read at adjusted base plus 0x40 copied to adjusted base plus 0x10, after an earlier byte store at that destination.
- The word captured from adjusted base plus 4 stored at publication base plus 0x74.
- A previously captured offset-0x14 word restored at doubled-link base plus 0x54.
- A halfword containing 0x65 stored at twice a newly loaded base.
- A word containing decimal 100 stored through another loaded base at offset 0x54.
- A final word store at twice a late loaded value.

Repeated reads of fixed byte address 0x67 remain reads at their represented positions, not an assumed immutable constant.

The final unresolved transfer carries four represented arguments, in order:

1. 0x72.
2. A word loaded from offset 0x64 of a late base.
3. The word captured earlier from doubled-link base plus 0x14.
4. A late word read from offset 0x74 of that late base.

No subsequent cleanup, rollback, or error translation is represented.

## Side effects

Observable effects include the indirect calls, memory reads and writes, fixed-storage changes, and final transfer.

No allocation or release operation is explicitly identified. Unknown dependencies may allocate, release, mutate shared state, or transfer control. The function establishes no ownership or lifetime contract for pointer-like values.

Effects are not transactional. If a dependency does not return or an access faults, earlier effects may remain visible. No compensating action is shown.

No locking or synchronization is represented. Atomicity, thread safety, reentrancy, interrupt safety, and device-memory behavior are **documented_only**. Access width alone does not establish atomicity.

Repeated writes and read-after-write order remain relevant whenever storage is aliased or observed. Final memory contents alone are insufficient to characterize the represented behavior.

## Dependencies

F-00d466dba2fc is the only function with a supplied opaque identity. No opaque IDs are supplied for the indirect targets or final transfer target. Their identities remain **documented_only**; this contract does not invent dependency IDs or merge call sites into one service.

Missing dependency contracts include:

- Input and result-register conventions.
- Meaning of combined results and secondary register outputs.
- Preservation of registers and shared state.
- Memory ownership and lifetime effects.
- Normal-return, nonreturning, fault, and error behavior.

## Acceptance examples

These examples are conditional static checks, not executed tests.

### Eleven-bit field

For an early returned word of 0x00000fff, retaining eleven low bits and shifting left by ten produces 0x001ffc00. Bits above bit 10 do not contribute.

### Twenty-bit field from a twenty-four-bit mask

For the relevant returned word 0x00ffffff, selecting bits 4 through 23 and shifting left by thirteen gives the mathematical value 0x1ffffe000, or 0xffffe000 after 32-bit reduction. The later bits-19-through-23 extraction is 31.

### Compact-base boundary

If the relevant late result’s low seven bits are all set, the bounded extraction is 0x1fc and the compact base is 0x3f8. Its offset-0x14 word address is 0x40c. Mapped writable memory at that address remains a separate precondition.

### Fixed-storage ordering

If memory accesses are trace-observed, the publication transition must show the word read at 0x087fbbbc before the word write of 0x20 at 0x60, followed by the first publication-group store.

A trace placing the write to 0x60 before that read does not satisfy this ordering, even if final ordinary-memory contents happen to match.

### Captured publication base

If the fixed-source read yields a particular publication base, subsequent publication groups use that captured value. If later accesses affect its source storage, those changes do not by themselves cause a reload.

### Literal-store trace

For write-observed storage at adjusted base plus 0x20, the word-store sequence is 0x20, 0x2f, the captured unsigned byte value, and 0x20. Emitting only the final value does not preserve the represented trace.

### Invalid access or target

If a reached address is invalid or a target is non-executable, no clean error result is promised. Earlier effects may already be visible; the exact fault mechanism is **documented_only**.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid invocation environment.
- Executable accuracy of the supplied representation.
- Dependency identities, calling conventions, and state effects.
- Signed overflow, signed shifts, address arithmetic, and pointer-width rules.
- Endianness, alignment, and unaligned-access behavior.
- Right-shift behavior when the low-byte count exceeds 31.
- Aliasing among computed and fixed regions.
- Whether low addresses denote ordinary memory, device storage, or inaccurate reconstruction.
- Exact publication-group multiplicity in this contract.
- Termination and normal return from the final transfer.

Linked-record manipulation and text emission are possible inferred intentions, not established requirements.

## Evidence

This contract uses only the supplied static representation for F-00d466dba2fc. Prior prose and reviewer feedback are not independent behavioral evidence.

The publication transition explicitly captures the fixed-storage word before assigning 0x20 to fixed address 0x60; the subsequent stores follow that assignment. This establishes the read-before-write requirement without assuming disjoint storage or unobserved memory.

Other established evidence consists of represented widths, arithmetic transformations, constants, call ordering, memory-access relationships, and final argument order.

No dependency bodies, executable inspection, runtime traces, or runtime tests were available. The contract records static behavior and its unresolved boundaries; it does not claim complete executable equivalence.
