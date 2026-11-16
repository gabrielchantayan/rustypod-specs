<!-- Trusted provenance: F-ed93e47f39af; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed93e47f39af

## Purpose

F-ed93e47f39af combines an ordered sequence of indirect dependency invocations with fixed-width arithmetic and a subsequent direct-memory mutation stage. Values returned by dependencies, secondary machine-register results, ambient entry state, and fixed-address storage contribute to later arguments, addresses, and stored values. A final indirect invocation receives four words after the memory updates.

The supplied representation does not establish an application-level purpose. Repeated offsets do not establish a conventional object layout; character-like constants do not establish text output.

This contract distinguishes:

- **Established static evidence:** represented operations, widths, value dependencies, and ordering.
- **[INFERENCE]:** interpretations or mathematical consequences conditional on explicitly stated machine semantics.
- **documented_only:** unresolved behavior or details not independently enumerated here.

The ordering requirements below describe the represented sequence conservatively. They do not establish that every intermediate non-volatile store is externally observable at runtime.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First argument | 32 bits | Forwarded unchanged to the entry dependency invocation. |
| Second argument | 32 bits, represented as signed | Forwarded unchanged and also supplied after a left shift by six positions. |

The entry invocation has three explicitly represented argument words: the first input, the second input, and the shifted second input.

Neither input has an established encoding, length, ownership classification, or valid-value enumeration. A pointer interpretation of either argument is **documented_only**.

### Ambient inputs

The effective input state also includes:

- Incoming control state used to compute the first indirect target.
- An incoming machine-register value used after a left shift by five.
- Secondary register results associated with dependency invocations.
- Fixed-address storage and memory reached through computed addresses.
- Dependency effects that may influence subsequently read memory.

Complete calling-convention requirements and register-preservation rules are **documented_only**. An invocation with no explicitly represented arguments need not be independent of ambient state.

### Outputs and completion

No explicit return value, success flag, error code, output length, or returned allocation is established.

The final indirect invocation receives, in order:

1. Word constant `0x72`.
2. A late word loaded from a reached base at offset `0x64`.
3. A word saved earlier from another base at offset `0x14`.
4. A late word loaded from the same reached base as the second argument, at offset `0x74`.

The saved third argument is not a late reload. Subsequent replacement of its originating field does not change that retained argument.

A return is represented after the final invocation. The underlying transfer mechanism and whether it returns normally are **documented_only**.

### Preconditions

Completion requires valid executed indirect targets and accessible memory for every executed access. No visible checks establish nullness, alignment, object extent, permission, or validity of loaded address-producing words.

Fixed low addresses participate directly. Ordinary user-process memory assumptions therefore do not establish a valid execution environment.

## Behavior

### Dependency stage

An extensive straight-line sequence of indirect invocations precedes the direct-memory stage. The representation shows no explicit conditional branch, retry, or data-dependent loop within F-ed93e47f39af itself. Dependency bodies may branch, fail, or fail to return.

The stage combines one-word results, paired results represented as 64 bits, secondary register outputs, selected explicit zeros, masks, shifts, and retained earlier values. Later invocations sometimes use values saved across several intervening invocations.

Dependencies cannot be treated as pure, interchangeable, or globally preserving. Retained values remain retained values; explicit memory reads remain reads at their represented positions.

The complete intermediate invocation trace and all argument transformations are **documented_only** rather than independently enumerated in this contract.

### Widths and arithmetic

The visible memory widths are 8-bit bytes, 16-bit halfwords, and 32-bit words. Paired results provide numeric low and high 32-bit halves; this does not establish memory byte order.

Represented transformations include constant left shifts, unsigned logical right shifts, masks, power-of-two scaling, and subtraction of `0x96`. Byte stores retain only the low eight bits. Byte values explicitly read as unsigned are zero-extended when stored as words.

Under a 32-bit bit-vector interpretation, arithmetic results used as words discard bits beyond bit 31. For example, a left shift by 25 retains only the original low seven bits and produces a zero low byte.

Some expressions are represented as signed arithmetic. Portable signed-overflow behavior, signed-right-shift behavior, and all instruction-level shift rules are not established. Modular conclusions here are conditional on the stated bit-vector interpretation.

One address-producing transformation shifts a word left by eight, masks with `0x7fff`, and logically shifts right by six.

**[INFERENCE, 32-bit bit-vector semantics]:** its result ranges from 0 to 504 in multiples of four. Twice that result ranges from 0 to 1008 in multiples of eight. These bounds establish neither accessibility nor alignment requirements of the environment.

### Initial direct-memory effects

The final paired dependency result contributes both a high-word address and a low-word scaled address. The initial represented write order is:

- A word write at the high result word plus `0x14`.
- A byte write of that high word’s low byte to a retained computed base plus `9`.
- A word write of another retained scaled value at 256 times the low result word plus `0x54`.
- A second byte write to the same base plus `9`, using the retained value shifted left by 25.

Under the bit-vector interpretation, the last byte is zero. The earlier byte assignment nevertheless remains part of the represented sequence.

Next, a word at another computed base plus `0x14` is cleared, a byte at that base plus `5` is read, a word at the small doubled base plus `0x14` is cleared, and the zero-extended byte is stored at the first base plus `0x34`. The first base’s `0x14` word is then read again. Intervening accesses may alias that word; substituting the earlier zero is not justified.

Further operations include another clearing of the small base’s `0x14` word and consecutive identical byte stores at its `0x15` offset.

### Field relationships and aliasing

Recurring offsets describe access relationships, not verified structure definitions.

| Offsets | Established access characteristics |
|---|---|
| `0x01`, `0x05`, `0x08`, `0x09`, `0x0d`, `0x15`, `0x19` | Byte accesses, including selectors, constants, and truncated retained values. |
| `0x04` | Byte or word reads, depending on the base and operation. |
| `0x10`, `0x11` | Overlapping byte and word updates. |
| `0x14` | Word clearing, saving, and repeated replacement. |
| `0x20`, `0x24`, `0x34` | Word updates, including constants, scaled values, and self-reference-like values. |
| `0x40`, `0x44`, `0x54`, `0x64`, `0x74` | Word reads or replacements; several read values subsequently produce addresses. |

A word at offset `0x14` overlaps a byte at `0x15`; a word at `0x10` overlaps a byte at `0x11`. Different computed bases may also alias one another or fixed-address storage.

A compound operation contains two represented reads of an offset-`0x74` word, using the results as an address contribution and a stored value. Their exact intra-expression evaluation order is **documented_only**. Unconditionally replacing them with one cached read is not justified.

The intervening region includes self-reference-like word stores, repeated byte stores, repeated word replacements, and address derivation through bytes followed by offset-`0x74` word reads. Its complete direct-memory trace beyond the effects expressly described here is **documented_only**.

### Three-width constant writes and subsequent replacements

A word reached through a byte-selected offset-`0x74` read supplies a loaded value used for three consecutive memory writes:

| Relative destination | Width | Numeric value |
|---|---:|---:|
| Four times the loaded value | 8 bits | `0x3a` |
| Eight times the loaded value | 16 bits | `0x003a` |
| Four times the loaded value plus `0x20` | 32 bits | `0x0000003a` |

These writes occur in byte, halfword, then word order. No memory access intervenes between them in the supplied representation. An address adjustment occurs between the first and second writes.

The following replacement sequence uses a different computed base. For reference, call the earlier retained word used in its derivation the **anchor**. The **adjusted value** is four times the anchor minus `0x96`; the **replacement base** is four times the adjusted value. These names describe retained numeric values, not allocations.

The replacement base’s offset-`0x20` word receives, in order:

1. `0x20`.
2. `0x2f`.
3. A zero-extended byte freshly read from the replacement base plus `4`.
4. `0x20`.

The byte read occurs after the first two word stores. However, the byte at base plus `4` and the word at base plus `0x20` occupy disjoint ranges under ordinary flat-address memory semantics, including 32-bit address wrapping. Those immediately preceding stores do not alter that byte through ordinary overlap. Dependence would require additional address-space or device behavior, which is not established.

### Fixed-base capture and repeated writes

After the replacement sequence, a word is read from an additional fixed storage location. Call the retained result the **fixed-storage base**. That read precedes the word assignment of `0x20` at address `0x60`.

A repeated four-write pattern follows. Its values are retained, not freshly reloaded for each group:

- A word write at a scaled address plus `0x54`, containing four times the loaded value used by the three-width constant writes.
- A byte write at the fixed-storage base plus `0x19`, containing the replacement base’s low byte.
- A byte write at the replacement base plus `0x11`, containing the fixed-storage base’s low byte.
- A word write at the fixed-storage base plus `0x34`, containing a scaled address value.

The initial group uses eight times the anchor for both the first write’s base and the fourth write’s value. Subsequent groups use eight times the adjusted value in those positions.

There is no visible advancement through a buffer. Exact repetition count is **documented_only**. Reducing the groups to final values alone is not established as equivalent under unresolved memory semantics.

### Late fixed reads, saved values, and fresh byte reads

After the repeated region, the function reads a 32-bit word at fixed address `0xa9`. It then reads a word at offset `0x74` from four times the earlier loaded value, before assigning the byte at fixed address `0x3a`.

The byte assigned at `0x3a` is eight times the adjusted value plus `0x30`, truncated to eight bits. The earlier word read from `0xa9` remains retained across this assignment.

Let the **late doubled base** mean twice the loaded value used by the three-width constant writes. Before subsequent mutations, the function saves its offset-`0x14` word and the replacement base’s offset-`4` word. It then performs several byte updates, writes `0x6c` at the replacement base plus `8`, and stores the late doubled base into its own offset-`0x54` word.

The fixed byte at `0x67` is assigned the low byte of the saved `0xa9` word. Three later accesses read `0x67` freshly:

- **First read:** a computed-memory word read at the late doubled base plus `0x44` occurs after the assignment to `0x67` and before the fresh read used to assign fixed address `0x21`.
- **Second read:** later, a byte store to the replacement base plus `9` obtains its value by reading `0x67` again. Before this read, intervening operations include a variable-shift byte store, fixed-storage-base field replacements, a word read at replacement base plus `0x40`, and a word copy to replacement base plus `0x10`.
- **Third read:** another byte store to replacement base plus `9` reads `0x67` separately. Between the second and third reads, the function writes a byte at late doubled base plus `9`, writes the retained `0xa9` word at replacement base plus `0x44`, and reads a word at replacement base plus `0x54`.

Unknown aliasing means these three reads cannot be specified as unconditional reuse of the originally assigned byte. The computed word read before the first fresh read must also retain its position.

The variable shift uses the signed-represented word read from late doubled base plus `0x44`, with a count equal to the replacement base’s low eight bits. The result is truncated into the byte at late doubled base plus `1`. Counts of at least 32 and signed-right-shift details remain **documented_only**.

### Final updates and dispatch

The late region also:

- Copies a saved word from replacement base plus `0x40` into its offset-`0x10` word, replacing an earlier byte update there.
- Replaces fields at offsets `0x44`, `0x54`, and `0x74`.
- Restores the saved late-doubled-base offset-`0x14` word into that base’s offset-`0x54` word.
- Reloads the replacement base’s offset-`0x14` word for a later truncated byte store.
- Replaces the replacement base’s offset-`9` byte after the two fresh-`0x67` stores.

A word loaded from late doubled base plus `4` supplies a further reached base. A halfword `0x0065` is written at twice that reached base before its offset-`0x64` word is read. Another reached location receives word constant `100` at offset `0x54`.

The reached base’s offset-`0x74` word is read after those mutations. A subsequent word read from late doubled base plus `0x64` supplies both a destination base for a field update and the value written at twice the word that becomes the final invocation’s second argument.

The final invocation uses the retained values described under Interface. No direct cleanup or rollback follows in the representation.

## Side effects

### Fixed-storage ordering

The following is the meaningful represented ordering of explicit fixed-storage activity. Computed accesses occur between many entries and can overlap fixed storage.

| Position | Activity |
|---|---|
| Early direct-memory region | Read storage at `0x05`; its declaration width is unavailable. |
| Immediately after that read | Assign the byte at `0x01`. |
| Before later replacements of the small base’s `0x14` word | Assign a word at `0x54`. |
| Before the three-width constant writes | Assign the byte at `0x15`. |
| After the offset-`0x20` replacement sequence | Read the additional fixed word used as the fixed-storage base. |
| After that capture, before repeated groups | Assign word `0x20` at `0x60`. |
| After repeated groups | Read a word at `0xa9`. |
| After a subsequent computed offset-`0x74` word read | Assign the byte at `0x3a`. |
| During late updates | Assign the byte at `0x67`. |
| After the computed offset-`0x44` word read | Read `0x67` and assign the byte at `0x21`. |
| Later, with mutations between them | Read `0x67` twice more, once for each separate replacement-base-plus-`9` byte store. |

This ordering does not imply separate storage for overlapping symbols or immunity from computed writes.

### Ownership, observation, and errors

No direct allocation, release, reference-count protocol, or ownership transfer is established. Dependencies may allocate, retain, consume, or release resources; their contracts are unknown.

No atomicity, locking, transaction boundary, rollback, or concurrency guarantee is shown. Non-volatile stores do not establish that a hardware or external observer sees every represented intermediate value.

No explicit error-handling path is visible. An invalid target or inaccessible address may prevent completion; the failure mechanism and resulting machine state are **documented_only**. Earlier effects need not be undone.

## Dependencies

F-ed93e47f39af is the only verified opaque function identity supplied. No verified opaque IDs are available for its indirect dependencies.

Dependency identities, semantic contracts, preservation guarantees, ownership behavior, failure conventions, and termination are **documented_only**. The established boundary facts are limited to represented invocation ordering, explicit argument words, result widths, and subsequent uses.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Entry arguments.** If the second input is `1` and the entry target is valid, its second argument is `1` and its third is `64`; the first input is unchanged.

2. **Word truncation.** Under 32-bit bit-vector semantics, shifting `0x04000000` left by six produces zero, not the unbounded integer `0x100000000`.

3. **Small base.** A relevant input word of `0x7f` produces `504` through the shift-mask-shift transformation and `1008` after doubling. Accessibility remains a separate precondition.

4. **Three widths.** If the loaded value is `16`, the constant-write region assigns byte `0x3a` at address `64`, halfword `0x003a` at `128`, and word `0x0000003a` at `96`, in that order, assuming valid arithmetic and accessible destinations.

5. **Replacement sequence versus observation.** In ordinary accessible memory without interfering writes or special behavior, the offset-`0x20` replacement sequence leaves `0x20`. The represented earlier stores remain part of this conservative contract; universal runtime visibility of each intermediate store is not required or established.

6. **Disjoint byte read.** Under ordinary flat-address semantics, the stores at replacement base plus `0x20` do not overwrite the byte at replacement base plus `4`.

7. **Fresh fixed byte.** If an intervening computed write changes `0x67` after its assignment but before one of its later reads, that read uses the then-current byte. The three read results need not match.

8. **Saved final argument.** If the saved offset-`0x14` word is subsequently overwritten in memory, the final invocation’s third argument remains the retained earlier word.

9. **Invalid dependency.** An invalid indirect target establishes neither successful completion nor a standardized error result.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object model, and complete calling convention.
- Dependency identities and behavior.
- Full intermediate invocation trace and unenumerated direct-memory effects.
- Exact repeated-group count.
- Memory byte order, alignment tolerance, permissions, and address-space behavior.
- Signed overflow, address arithmetic outside the conditional bit-vector model, and variable-shift semantics.
- Device-backed behavior, asynchronous interference, and runtime visibility of intermediate stores.
- Failure mechanisms and whether the final transfer returns normally.

**[INFERENCE]:** repeated self-reference-like and cross-reference-like stores may install relationships between memory regions. This interpretation is not an acceptance requirement.

## Evidence

This contract derives from the supplied static representation: explicit widths, numeric transformations, retained values, argument positions, fixed addresses, and represented memory-access order.

Comments, prior prose, and reviewer feedback are not independent evidence. No dependency bodies, external implementation, binary execution trace, or runtime observations were supplied.

No runtime tests have been performed. The acceptance examples are conditional reasoning checks only.
