<!-- Trusted provenance: F-db6feb334b73; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db6feb334b73

## Purpose

F-db6feb334b73 performs an ordered sequence of indirect calls, derives fixed-width values from incoming execution state and call results, and mutates memory through fixed and computed addresses. It finishes with an unresolved indirect transfer carrying four visible argument words.

The supplied static representation does not establish a higher-level purpose. Repeated offsets, pointer-like values, and character-valued stores do not establish formatting, allocation, initialization, serialization, or container management.

This contract distinguishes:

- **Established static evidence:** represented operations, widths, dependencies, and ordering.
- **Inferred intent:** possible interpretations that are not acceptance requirements.
- **documented_only:** unresolved behavior that must not be replaced with an invented guarantee.

No runtime tests have been performed.

## Interface

### Entry conditions and outputs

The represented interface declares no parameters and no return value. Its behavior nevertheless depends on:

- Incoming machine state, including a 32-bit word and return-link state used in selecting the first indirect target.
- Primary and secondary result words from indirect calls.
- Memory contents at fixed and computed addresses.
- Effects of preceding calls and accesses on subsequently observed state.

A complete ordinary-call ABI is not established. Required register contents, preservation rules, executable mappings, accessible storage, alignment, and privileges are **documented_only**. A call with no displayed arguments is not necessarily input-free.

Visible outputs consist of dependency effects, ordered memory accesses and mutations, and the terminal indirect transfer. There is no established success code, error code, output-buffer interface, or ownership token.

Successful progress requires each preceding call to return appropriately and every required access to complete. The function does not visibly validate computed addresses.

### Widths and arithmetic

| Quantity | Represented treatment |
|---|---|
| Byte access | 8 bits |
| Byte used numerically | Zero-extended where explicitly represented |
| Halfword store | 16 bits |
| Word access and most arguments | 32 bits |
| Address calculations | Represented using 32-bit words |
| Paired call result | 64 bits, separated into low and high 32-bit words |
| Value narrowed to a byte | Low 8 bits |

A paired result is not established to be a scalar, pointer, or record. Only the subsequent use of its two words is established.

Arithmetic descriptions refer to retained 32-bit patterns where applicable. Left shifts discard bits outside that width; they are not unlimited-width multiplication. The representation also contains signed-typed arithmetic and shifts. Exact machine behavior for signed overflow, negative right-shift operands, and excessive variable shift counts remains **documented_only**.

### Storage layout

No allocation boundary or object type is established. Recurring byte offsets include +1, +5, +8, +9, +0x0d, +0x10, +0x11, +0x15, and +0x19. Recurring word offsets include +4, +0x10, +0x14, +0x20, +0x24, +0x34, +0x40, +0x44, +0x54, +0x64, and +0x74.

Decimal 100 and hexadecimal 0x64 designate the same offset. The literal decimal 100 also appears as a stored word value; these uses are distinct.

These offsets describe accesses, not verified record fields. Different bases may overlap. Byte stores can overlap word stores, including a byte at +0x15 within a word beginning at +0x14.

## Behavior

### Execution structure

The represented path contains three principal regions:

1. An extensive sequence of indirect calls and fixed-width transformations.
2. Direct memory mutation without further represented helper calls.
3. A terminal indirect invocation and a represented return.

No explicit conditional branch or input-controlled loop is shown. This does not establish that dependencies lack branches, terminate, or return normally.

The ordering below concerns represented accesses and calls. It does not imply atomic transactions, isolation, or a snapshot of memory.

### Indirect-call preparation

The first target is selected using incoming return-link state plus 0x0ffe. Later preparation targets use successive code-related anchors with the same displacement. These anchors select executable targets; they are not established data objects or dependency identities.

The preparation region interleaves calls with no displayed arguments, calls with explicit zero arguments, calls receiving shifted results, and calls receiving words separated from paired results. Some results are unused explicitly. Such calls still have unresolved effects and cannot be discarded.

Early arguments include the incoming word shifted left by 13 and shifted right logically by 11. A subsequently obtained result supplies both a left-shift-by-8 argument and a low-24-bit field shifted right by 4 and then left by 13.

That saved result later supplies a five-bit field by retaining its low 24 bits and shifting right by 19. This field contributes arguments shifted left by 3 and by 31. Its least significant bit separately contributes a value at bit position 16.

Further transformations use masks retaining 13, 15, 16, 17, 18, or 24 bits, together with shifts ranging from 2 through 31. Several arguments depend on saved earlier results rather than the most recent call result. Intervening calls do not eliminate those dependencies.

Important fixed-width consequences include:

- Retaining 24 bits and shifting right by 8 yields 0 through 65,535.
- Retaining 24 bits and shifting right by 19 yields 0 through 31.
- Retaining 18 bits and shifting right by 11 yields 0 through 127.
- Retaining 17 bits and shifting right by 4 yields 0 through 8,191.
- Shifting a 32-bit word left by 8 clears its low byte and discards its original high byte.
- Successive left shifts retain only bits surviving both operations.

Several derivations are explicitly seeded with zero. Their shifted forms remain zero in the represented dataflow, but the receiving calls remain observable dependencies.

Near the transition to direct memory access, a result is shifted left by 8, masked to 15 bits, and shifted right by 6. This produces a multiple of four from 0 through 508. Doubling it produces a multiple of eight from 0 through 1,016. The doubled value is used as an address base, not merely as a count.

### First direct memory effects

The following descriptive labels identify values without assigning object semantics:

- **Shifted base:** a paired-result low word shifted left by 25.
- **Scaled base:** a separately obtained result multiplied by 2²⁵.
- **Compact index:** the multiple-of-four value from 0 through 508.
- **Compact base:** twice the compact index.
- **Saved shifted word:** another saved result shifted left by 8.
- **Final pair:** the last paired result before direct memory mutation.

The first direct word store places the shifted base at the final pair’s high word plus 0x14. The byte at shifted base +9 then receives the low byte of that high word. A word store places the scaled base at 256 times the final pair’s low word plus 0x54. Only afterward does shifted base +9 receive the low byte of the shifted base.

The latter byte is zero under the represented 32-bit shift. Both writes remain part of the access sequence.

The function then clears the word at scaled base +0x14 and reads a byte from scaled base +5. It clears compact base +0x14, writes the zero-extended byte to scaled base +0x34, and reads the word at scaled base +0x14. Another clear of compact base +0x14 precedes two byte writes of the read-back word’s low byte to compact base +0x15.

The read-back word is not unconditionally zero. That conclusion requires storage behavior and non-overlap conditions sufficient to prevent intervening writes or external effects from changing it.

### Linked-address mutation

The next region reads the byte at absolute address 5 and uses its zero-extended value as a base. After this read, absolute byte address 1 receives the low byte of the compact base.

The ensuing accesses combine byte stores at +9, +0x0d, +0x11, and +0x15 with word stores at +0x14, +0x24, +0x34, +0x44, and +0x54. Bases come from earlier words, doubled or quadrupled values, eightfold values, and words loaded through offsets including +4, +0x64, and +0x74.

A byte read from a computed base +5 is also used as an address base for a word read at +0x74. Neither this use nor self-referential stores establish valid heap links.

Observable replacement patterns include:

- A byte at scaled base +0x11 receives the saved shifted word’s low byte and is subsequently cleared.
- A word is captured from the byte-derived base +0x54 before that location is replaced.
- Repeated stores write the same captured value at a computed +0x24 location and its low byte at another base +9.
- Compact base +0x14 receives a shifted value and then repeated replacements with a captured word.
- Absolute word address 0x54 receives twice a captured value.
- Absolute byte address 0x15 receives the low byte of the compact base.

Later accesses propagate quadrupled and eightfold versions of the captured word. They also read through +0x74, double a loaded value, and use byte-derived bases for additional +0x74 reads. Stores can replace earlier association values before subsequent reads occur.

Repeated reads must observe memory at their respective positions. The representation does not establish that a repeated load returns the same value, even when its address expression is unchanged.

### Literal-valued stores and association-base capture

A later loaded word supplies three literal-valued destinations:

- Its quadrupled value receives a byte store of 0x3a.
- Its eightfold value receives a halfword store of 0x3a.
- Its quadrupled value plus 0x20 receives a word store of 0x3a.

A different base is formed by subtracting 0x96 from an earlier quadrupled value, then multiplying the adjusted value by four. At this base +0x20, the represented word replacements occur in this order: 0x20, 0x2f, the zero-extended byte read from the base +4, and 0x20 again.

The ordering at the boundary to association writes is specifically:

1. The final computed +0x20 word store writes 0x20.
2. The global word supplying the association base is loaded and captured.
3. Absolute word address 0x60 receives 0x20.
4. The association-write run begins.

The global load therefore occurs **before**, not after, the absolute-address store. Unknown aliasing or access semantics do not permit reversing these events. The association run uses the captured global value; it is not represented as reloading the global after the write to 0x60.

The literals resemble punctuation and spacing in some encodings. Text production remains inferred intent only: no encoding, string boundary, or output stream is established.

### Repeated association writes

The association run repeatedly performs four stores in a fixed relative order:

- A word at an eightfold computed base +0x54 receives the quadrupled literal-source value.
- A byte at the captured global-supplied base +0x19 receives the low byte of the quadrupled adjusted base.
- A byte at the quadrupled adjusted base +0x11 receives the low byte of the captured global-supplied base.
- A word at the captured global-supplied base +0x34 receives the eightfold computed base.

The first group uses the eightfold unadjusted earlier value. Subsequent groups use the value adjusted by subtraction of 0x96 before scaling.

This is explicit repetition in the supplied representation, not an established traversal of distinct elements. Replacing the run with one final assignment would discard represented accesses. Exact repetition multiplicity remains **documented_only** in this contract.

The captured association base is reused throughout. There is no intervening represented helper call, but this does not establish isolation from external memory changes or make overlapping stores independent.

### Final mutation and transfer

After the association run, the function captures a word from absolute address 0xa9 and another from the quadrupled literal-source base +0x74. It then writes a narrowed computed value to absolute byte address 0x3a before capturing further words from a doubled loaded base +0x14 and the adjusted quadrupled base +4.

The remaining region includes:

- A byte store of 0x6c at the adjusted quadrupled base +8.
- Byte stores of captured low-byte values at computed +0x0d and +9 destinations.
- A self-valued word store at a computed +0x54 destination.
- A byte store to absolute address 0x67, followed later by a read of that location for the store to absolute address 0x21.
- A variable right shift with a count taken from the low byte of the adjusted quadrupled base.
- Replacement of computed words at +0x10, +0x44, +0x54, and +0x74.
- Further reads of absolute byte address 0x67 for computed byte stores.

The absolute-byte reads are not established to be interchangeable with a cached earlier value. Intervening overlapping stores or external access behavior could matter.

A word freshly loaded through +4 supplies the base for a halfword store of 0x65 at twice that word. A later word loaded through +0x64 is retained for both the final writes and terminal invocation. Another computed +0x54 destination receives decimal 100.

Near the end, a word is captured from the freshly loaded base +0x74. A separate word is then read through +0x64 of the doubled loaded base. Subsequent stores include a byte at that doubled base +9, a word through the latter loaded value +0x54, and a final word store through twice the retained +0x64 value.

The terminal indirect invocation receives four visible 32-bit words:

1. 0x72.
2. The retained word loaded through offset +0x64.
3. The earlier captured word from the doubled loaded base +0x14.
4. The later captured word from the freshly loaded base +0x74.

Target identity and calling convention remain unresolved. A return is represented afterward, conditional on control returning from the terminal target.

## Side effects

The visible side effects are indirect dependency invocations, byte/halfword/word accesses, fixed-address mutations, computed-address mutations, and the terminal control transfer.

There is no established atomicity, rollback, locking, isolation, or concurrency guarantee. Endianness affects interactions between overlapping widths. Alignment, device-memory behavior, and access-fault semantics are **documented_only**.

No allocation or release operation is explicitly identified. No ownership transfer, retention, or release obligation can be assigned to an unresolved dependency. The absence of an identified release does not establish a leak.

There is no visible local error recovery. An unsuccessful access or a dependency that does not return may prevent later effects. Specific fault classes and propagation rules are unresolved.

## Dependencies

F-db6feb334b73 is the only resolved opaque function identity in the supplied mapping. No separate dependency ID can be assigned reliably.

| Dependency class | Established interaction | Unresolved behavior |
|---|---|---|
| Initial indirect target | Selected from incoming return-link state using displacement 0x0ffe | Identity, ABI, required state, effects |
| Preparation targets | Ordered invocations with transformed words, explicit zeros, and paired-result components | Identity, preservation, memory effects, termination |
| Terminal indirect target | Four visible argument words, beginning with 0x72 | Target selection, argument meanings, return behavior |

No dependency is assumed to preserve globals, preserve unmentioned machine state, allocate storage, release storage, retain pointers, or return normally.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Low-24-bit extraction.** For a relevant result of 0xab123456, retaining 24 bits and shifting right by 8 yields 0x1234. The high byte does not contribute.

2. **Five-bit field.** Low 24 bits of 0xf80000 yield 31 after shifting right by 19. Its least significant bit contributes 0x00010000 where placed at bit position 16.

3. **Compact-base boundary.** Relevant result bits with low seven bits equal to 0x7f yield compact index 508 and compact base 1,016. This derivation cannot produce an arbitrary 15-bit address base.

4. **Overwritten byte.** If shifted base +9 is observable, it first receives the final pair’s high-word low byte and subsequently zero. Recording only zero loses a required event.

5. **Literal replacement.** Under ordinary non-overlapping storage, the computed +0x20 destination ends at 0x20. Its earlier 0x20, 0x2f, and byte-derived word replacements remain ordered events.

6. **Global capture precedes absolute store.** An access trace must place the association-base global load between the final computed +0x20 store and the word store to absolute address 0x60. Moving the load after the absolute store is unacceptable even if final memory happens to match.

7. **Captured association base.** If the global load supplies a particular base, subsequent association stores use that captured value. They do not obtain a replacement base by reloading the global after the write to 0x60.

8. **Aliasing affects later reads.** When a computed store overlaps a subsequently read byte or word, the later read observes storage at that point. Independent-object or snapshot assumptions are not acceptable.

9. **Terminal completion.** If all preceding operations complete and the terminal target returns normally, the represented return is reached without a declared result. A non-returning terminal target does not establish successful function completion.

## Uncertainties

The following remain **documented_only**:

- Higher-level purpose and object model.
- Complete entry ABI, implicit inputs, and preservation rules.
- Identities and contracts of all indirect dependencies.
- Validity and meaning of fixed low-address accesses.
- Endianness, alignment, access faults, and external memory semantics.
- Machine behavior for signed overflow and variable shifts outside the ordinary word-width range.
- Exact multiplicity of the long repeated association-write run.
- Runtime reachability, termination, ownership, and concurrency safety.
- Whether the terminal transfer is accurately represented as a call followed by return.

These uncertainties do not permit inventing address validation, collapsing repeated stores, reversing established accesses, assigning object types, or assuming dependency preservation.

## Evidence

The evidence is the supplied static representation and opaque identity mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

The representation establishes the displayed widths, constants, data dependencies, indirect invocations, and access ordering. In particular, it places the association-base global load after the final computed +0x20 store and before the absolute word store to 0x60.

It does not establish valid runtime mappings, successful execution, dependency semantics, or intended application behavior. No runtime execution or tests have been performed.
