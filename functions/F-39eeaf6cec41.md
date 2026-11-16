<!-- Trusted provenance: F-39eeaf6cec41; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-39eeaf6cec41

## Purpose

F-39eeaf6cec41 accepts one opaque 32-bit input, performs an ordered sequence of indirect invocations and result transformations, updates memory through fixed and derived addresses, and reaches an unresolved terminal invocation.

No application-level purpose is established. Character-like constants and recurring offsets do not establish text generation, serialization, allocation, initialization, or object management.

This contract distinguishes:

- **Established static evidence:** behavior explicitly represented in the supplied function.
- **Inferred intent:** interpretations that are not acceptance requirements.
- **documented_only:** behavior or detail left unresolved.

All behavior is conditional on execution reaching the relevant operation. No runtime tests have been performed.

## Interface

### Explicit input and output

The explicit input is one uninterpreted 32-bit value. Its entire bit pattern is forwarded as the first explicitly represented argument to F-U001.

The represented caller does not directly dereference this input. A dependency receiving it may interpret or dereference it. No valid-value range, encoding, length, capacity, or associated object layout is established.

No explicit return value is represented. An ordinary return appears after F-U003, but completion requires that terminal invocation to return normally. There is no established success status, failure status, or returned ownership.

### Implicit inputs

Execution also depends on:

- Incoming control-link state.
- Indirect invocation targets.
- Ordinary and secondary result channels.
- Memory at fixed and derived addresses.
- The target’s arithmetic, addressing, and access semantics.

The initial target is formed by adding 0xFFE to the incoming control-link value. Subsequent targets use other represented address calculations. Their identities and complete calling conventions remain unresolved.

The explicit input alone therefore does not determine the execution trace or final memory state.

### Widths and layouts

| Quantity or access | Represented width |
|---|---:|
| Explicit input | 32 bits |
| Ordinary scalar results and saved words | 32 bits |
| Aggregate results | 64 bits |
| Byte reads and writes | 8 bits |
| Halfword stores | 16 bits |
| Word reads and writes | 32 bits |

Aggregate results are separated into low and high 32-bit portions. This does not establish whether the aggregate denotes one semantic value or multiple ABI result channels.

Byte stores retain the low eight bits. Unsigned byte reads used as word data are zero-extended. Halfword and word stores remain accesses of their represented widths; they are not interchangeable with byte stores.

Arithmetic examples below use **32-bit bit-vector arithmetic**: each 32-bit intermediate retains only its low 32 bits. This is a conditional arithmetic model, not a guarantee concerning signed overflow, pointer conversion, address wrapping, alignment, or fault behavior.

Offsets identify access locations, not validated fields. Byte and word accesses may overlap. For example, a byte at offset 0x15 lies within a word beginning at offset 0x14 on an ordinary byte-addressed mapping. Endianness determines which word bits it affects.

### Preconditions and ownership

Normal completion requires all reached invocation targets to be callable and all reached accesses to be permitted at their represented widths and required alignments.

No explicit null check, bounds check, address validation, or ownership protocol is represented. Dependencies must not be assumed to preserve globals, retain memory, release memory, or leave referenced storage accessible.

## Behavior

### Indirect invocation phase

The first represented externally observable operation is F-U001 receiving the unchanged input. A long, straight-line sequence of intermediate indirect invocations follows.

No explicit conditional branch, loop, retry, or early return is represented in the caller. Dependencies may nevertheless branch, mutate memory, fail, or not return.

Arguments include shifted ordinary results, aggregate high words, differently shifted copies of the same result, explicit zeros, masked fields, and values saved across intervening invocations. Some arguments originate in secondary result channels whose complete provenance is unresolved.

An invocation remains observable even when its ordinary result is unused. An invocation with no explicitly represented arguments is not established to lack implicit inputs or effects.

### Arithmetic boundaries

The representation contains masks selecting widths including 11 through 22 bits, as well as 24 bits. Masking, shifting, and truncation retain their represented order.

Under the conditional bit-vector model:

- A 32-bit left shift by eight discards the original high eight bits.
- Masking to 24 bits before shifting left by six differs from shifting the unrestricted word.
- Unsigned aggregate right shifts by 34, 36, and 44 correspond to logical right shifts of the aggregate’s high word by two, four, and twelve.
- Two successive 32-bit left shifts by 28 produce zero.
- A locally established zero remains zero through subsequent local shifts until reassigned.

These arithmetic facts do not remove surrounding invocations or memory accesses. They also do not justify substituting a primary result or zero for an unresolved secondary-channel value.

### Address and saved-value roles

The following descriptive roles distinguish captured values from later reads:

| Role | Meaning |
|---|---|
| Early lane base | An earlier aggregate low word shifted left by 25 |
| Saved shifted result | A later ordinary result shifted left by eight, retained across intervening invocations |
| Shifted base | Another ordinary result multiplied by \(2^{25}\) |
| Selector | A result shifted left by eight, masked to 15 low bits, then logically shifted right by six |
| Selector base | Twice the selector |
| Captured early destination | A word read from shifted-base offset 0x14 |
| Byte-selected base | The zero-extended byte read from fixed address 0x05 |
| Saved link | A word captured from byte-selected-base offset 0x54 before that location is overwritten |
| Adjusted base | Four times the saved link, reduced by 0x96 |
| Publication base | Four times the adjusted base |
| Linked destination | Twice a later word loaded through a byte-selected address at offset 0x74 |
| Published value | Four times that same later loaded word |
| Late state base | The word captured from fixed address 0x087FBBBC |

These roles do not establish object types, valid pointers, or disjoint storage.

Under bit-vector arithmetic, only the selector-producing result’s original low seven bits contribute. The selector ranges from zero to 508 in multiples of four. The selector base ranges from zero to 1016 in multiples of eight.

Only the low seven bits of the shifted-base-producing result contribute to the shifted base. Neither derived base is validated.

### Early direct accesses and fixed-location effects

The explicit memory phase starts with a word store of the early lane base at an aggregate-high-word-derived address plus 0x14. A byte at early-lane-base offset 0x09 is then written twice. Between those byte writes, a word store places the shifted base at an address formed by scaling an aggregate low word by 256 and adding 0x54.

Next, shifted-base offset 0x14 is cleared. A byte is read from shifted-base offset 0x05. Selector-base offset 0x14 is cleared before that saved byte is stored, zero-extended, as a word at shifted-base offset 0x34.

The captured early destination is then read from shifted-base offset 0x14. Selector-base offset 0x14 is cleared again, followed by two separate byte stores of the captured destination’s low byte at selector-base offset 0x15.

The function next reads the unsigned byte at fixed address 0x05 and writes the selector base’s low byte to fixed address 0x01. It writes the selector’s low byte at byte-selected-base offset 0x15.

The next interval has an explicit ordering requirement: **the byte write at shifted-base offset 0x11 precedes a 32-bit store of the saved shifted result at captured-early-destination offset 0x54; that word store precedes the clear of shifted-base offset 0x11. Only after the clear is the saved link read from byte-selected-base offset 0x54.**

The word-store destination uses the previously captured shifted-base-offset-0x14 value. It is not obtained by rereading that location. Moving the word store after the clear would change the access trace and can change observable behavior under overlapping mappings.

After capturing the saved link, the function overwrites byte-selected-base offset 0x54 with the captured early destination. Later calculations use the saved link, not a reread of the overwritten field.

Other fixed writes in this phase are:

- A 32-bit write of twice the saved link to fixed address 0x54.
- A later byte write of the selector base’s low byte to fixed address 0x15.

Duplicate byte and word stores are separate represented accesses. Their occurrence is not reducible to a final-state assignment.

### Linked and scaled updates

The middle memory region combines doubled, quadrupled, and eightfold address-like values with word reads at offsets 0x04, 0x44, 0x64, and 0x74. Unsigned bytes read at offset 0x05 also select later addresses.

Observable updates include byte stores at offsets 0x09, 0x0D, 0x11, and 0x15, and word stores at offsets 0x14, 0x24, 0x34, 0x44, and 0x54. Some stores use their destination base as word data.

Reads observe memory at their own positions, including applicable effects of earlier overlapping writes. Later byte reads at offset 0x05 are independent accesses, not reuse of an earlier captured byte.

One nested update reads selector-base offset 0x74 twice: once to select an offset-0x54 destination and once to obtain the word stored there. Two reads are represented; their internal evaluation order is unresolved. Replacing them with one cached read is not justified.

Recurring offsets suggest linked state as **inferred intent** only. No record layout, link validity, or structural invariant is established.

### Constants and late-state capture

The numeric constant 0x3A is stored at three destinations with distinct widths:

- An 8-bit store at four times a loaded word.
- A 16-bit store at eight times that word.
- A 32-bit store at four times that word plus 0x20.

Publication-base offset 0x20 then receives four word writes: 0x20, 0x2F, a zero-extended byte read from publication-base offset 0x04, and 0x20 again. The byte read occurs after the second write and supplies the third write; it need not reflect entry-time memory.

After the fourth write, the function captures the late state base from fixed address 0x087FBBBC. It subsequently writes the word 0x20 to fixed address 0x60. The fixed store does not cause a reload of the captured base.

### Repeated write region

Each represented group in this region has four ordered access roles:

1. A word store of the published value at a scaled destination plus 0x54.
2. A byte store of the publication base’s low byte at late-state-base offset 0x19.
3. A byte store of the late state base’s low byte at publication-base offset 0x11.
4. A word store of the scaled destination base at late-state-base offset 0x34.

The first group uses eight times the saved link as its scaled destination. Subsequent groups use eight times the adjusted base. The first group therefore differs both in its offset-0x54 destination and its offset-0x34 data.

Subsequent groups do not advance a cursor. No array extension or character appending is established. Exact multiplicity is **documented_only** here; the repeated occurrences are not optional or permission to collapse the region into final values.

### Final fixed reads and captures

Immediately after the repeated region, the represented access order is:

- A word read from fixed address 0xA9.
- A word read from published-value offset 0x74.
- A byte write to fixed address 0x3A.
- A word read from linked-destination offset 0x14.
- A word read from publication-base offset 0x04.

The fixed-0x3A byte receives the low eight bits of eight times the adjusted base plus 0x30. This address is distinct from the earlier numeric constant 0x3A.

The two words read after that fixed byte write are retained for later stores and terminal arguments. They are not entry-time snapshots.

Further writes update linked-destination offset 0x0D and publication-base offsets 0x08, 0x0D, and 0x09. Offset 0x08 receives 0x6C. A word store places the linked destination itself at linked-destination offset 0x54.

### Fixed byte 0x67 and independent rereads

The low byte of the previously captured fixed-0xA9 word is written to fixed address 0x67. A word is then read from linked-destination offset 0x44, followed by a fresh byte read from fixed 0x67 for a copy to fixed address 0x21.

Fixed 0x67 is read twice more, each time supplying a byte store at publication-base offset 0x09. These reads are separated by other accesses. Before the third read, those accesses include a word write at publication-base offset 0x44 and a word read at publication-base offset 0x54.

All three fixed-0x67 reads are independent. An intervening derived access may overlap that address. Neither later copy is guaranteed to reproduce the initially assigned byte or the byte copied to fixed 0x21.

The word captured from linked-destination offset 0x44 is also right-shifted by the publication base’s low eight bits. The result is truncated to one byte and written at linked-destination offset 0x01. Signed right-shift behavior and counts of 32 or more require target-specific semantics.

Other effects include a word copy from publication-base offset 0x40 to offset 0x10, a store of the saved publication-base-offset-0x04 word at late-state-base offset 0x74, and updates to late-state-base offsets 0x44 and 0x54.

### Terminal updates and invocation

The final region uses the saved published-value-offset-0x74 word as byte data and restores the saved linked-destination-offset-0x14 word to linked-destination offset 0x54. It also captures a word from publication-base offset 0x14.

A word read from linked-destination offset 0x04 selects another base. A 16-bit 0x65 is stored at twice that base, and its offset-0x64 word is captured. After intervening updates, including a word store of 100 through another derived destination plus 0x54, its offset-0x74 word is captured.

A separate read from linked-destination offset 0x64 selects a further destination. Final word stores use these saved values, including a store through twice the first captured offset-0x64 word.

F-U003 receives four explicitly represented arguments:

1. 0x72.
2. The offset-0x64 word captured through the linked-destination-offset-0x04-selected base.
3. The saved linked-destination-offset-0x14 word.
4. The later offset-0x74 word captured through that same selected base.

No textual, status, context, or ownership meaning is established.

## Side effects

Direct effects comprise byte reads and writes, halfword stores, and word reads and writes through fixed and derived addresses.

| Fixed address | Explicitly represented access |
|---|---|
| 0x05 | Byte read |
| 0x01, 0x15, 0x3A, 0x67, 0x21 | Byte writes |
| 0x67 | Three separate byte reads |
| 0x54, 0x60 | Word writes |
| 0x087FBBBC, 0xA9 | Word reads |

Derived accesses may overlap these locations or each other.

The early offset-0x11 byte write, intervening offset-0x54 word store, and subsequent offset-0x11 clear are three distinct effects in that order.

No transaction, rollback, locking, or explicit recovery path is represented. If execution terminates after earlier effects, no represented recovery restores them.

Dependencies may perform additional I/O or memory-management effects. None is established, and no global-preservation guarantee is assumed.

## Dependencies

The dependency labels below are document-local opaque boundary identifiers.

| Opaque ID | Boundary |
|---|---|
| F-U001 | Initial indirect invocation receiving the explicit input |
| F-U002 | Intermediate unresolved invocation boundaries collectively |
| F-U003 | Terminal unresolved invocation receiving four represented arguments |

F-U002 denotes multiple boundaries, not one established shared target. Target equality, complete signatures, ABI, secondary-channel provenance, preservation, ownership, and retention remain **documented_only**.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

### Input preservation and truncation

For explicit input 0xFEDCBA98, F-U001 receives that exact 32-bit pattern. Under bit-vector arithmetic, shifting a later 0x12345678 result left by eight yields 0x34567800.

### Aggregate extraction

For an aggregate whose high word is 0x80000004, an unsigned right shift of the complete aggregate by 34 yields 0x20000001. Arithmetic shifting of a signed high word is not an equivalent substitution.

### Selector boundaries

All contributing selector bits clear produce selector base zero. All set produce 1016. Neither result establishes a valid address.

### Intervening word-store ordering

For an execution reaching the early offset-0x11 interval, an access trace places the saved-shifted-result word store after the byte write and before its clear. Its address uses the captured early destination plus 0x54.

If the word store overlaps the byte location under the actual mapping, the clear remains later. If that word store faults and the fault terminates execution, the later clear and saved-link read have not yet occurred.

### Late-base capture

The repeated region uses the base captured from fixed 0x087FBBBC before the fixed-0x60 store. It does not reload the base merely because a subsequent write occurs.

### Final capture ordering

If fixed address 0x3A overlaps either later capture location, the linked-destination-offset-0x14 and publication-base-offset-0x04 reads occur after the byte update and observe its applicable effects.

### Independent fixed-byte copies

If an intervening access changes fixed 0x67, a later copy to publication-base offset 0x09 uses the byte read at that later point, not necessarily the byte previously copied to fixed 0x21.

### Partial completion

If an early write succeeds and a later access terminates execution, no represented rollback undoes the early write. If F-U003 is reached, its first represented argument is 0x72; ordinary completion remains conditional on its return.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid input domain, and semantic layouts.
- Accuracy and completeness of the supplied static representation.
- Indirect-target identities, ABI, and secondary-channel provenance.
- Target address width, overflow, pointer conversion, endianness, alignment, and fault delivery.
- Signed right shifts and large shift counts.
- Memory-map validity, overlap, device behavior, and asynchronous mutation.
- Internal evaluation order of the nested pair of offset-0x74 reads.
- Exact repeated-group multiplicity not enumerated here.
- Dependency allocation, release, retention, ownership, and preservation behavior.
- Whether the terminal operation corresponds to an ordinary returning invocation or another control transfer.

Linked-state and publication-like interpretations are inferred intent only. They impose no additional structural requirements.

## Evidence

The supplied static function representation supports input forwarding, represented widths, indirect activity, arithmetic transformations, explicit memory accesses, captured-value reuse, and terminal argument dependencies.

It specifically establishes that the early saved-shifted-result word store lies between the shifted-base-offset-0x11 byte write and clear, with the saved-link read following the clear.

The prior draft, comments, and reviewer feedback are not independent authority. No tools, external implementation inspection, or runtime tests were used. Acceptance examples describe conditional requirements rather than observed executions.
