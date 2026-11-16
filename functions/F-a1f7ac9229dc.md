<!-- Trusted provenance: F-a1f7ac9229dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a1f7ac9229dc

## Purpose

F-a1f7ac9229dc performs an ordered sequence of indirect dependency invocations, derives address-like values from their results, mutates memory through computed and fixed addresses, and invokes an unresolved final target.

The supplied representation establishes those effects, not an application-level purpose. Repeated offsets and character-like constants do not establish text formatting, object initialization, list management, or a protocol.

This contract distinguishes:

- **Established static evidence:** represented accesses, widths, conversions, saved-value relationships, and ordering.
- **[INFERENCE]:** conclusions requiring an explicitly stated arithmetic or memory model.
- **documented_only:** unresolved semantics or trace details not exhaustively specified here.

Summarized phases do not authorize removing or reordering represented accesses.

## Interface

### Inputs and outputs

The represented interface has no explicit parameters and no defined scalar return value. Effective inputs include an incoming execution-context word, an incoming continuation-related value, dependency results and secondary-result words, memory reached through computed addresses, fixed-location globals, and the final invocation target.

Observable outputs are dependency invocations and memory effects. A return is represented after the final invocation, conditional on that invocation returning normally.

No buffer extent, conventional object handle, status output, or error parameter is established.

### Widths and arithmetic

| Entity | Represented width or qualification |
|---|---|
| Explicit byte access | 8 bits |
| Explicit halfword store | 16 bits |
| Explicit word access | 32 bits |
| Combined dependency result | 64 bits |
| Numeric lower and upper halves | 32 bits each |
| Conversion of the global at 0x05 | Unsigned 32-bit result; underlying storage type unresolved |

Byte and halfword stores retain the low eight and sixteen bits, respectively. Explicit unsigned byte loads produce values from 0 through 255. Splitting a combined result by numeric significance does not establish memory byte order.

The global read at **0x05 is not established as an unsigned byte load**. Its declaration, accessed extent, signedness, and extension or truncation behavior are unavailable. Its unsigned conversion result is not necessarily restricted to 0–255.

**[INFERENCE]** Under a 32-bit bit-vector model, word arithmetic retains the low 32 bits. Some represented operations are signed, so universal wraparound, signed-overflow behavior, and signed-shift behavior are not established.

Preparation includes left-shift distances as small as one and as large as thirty-one; this does not assert that every intervening distance occurs. Unsigned right shifts by 34, 36, and 44 operate on whole 64-bit results, not 32-bit operands.

### Preconditions and errors

Every reached target must be callable under the actual execution-state convention. Every reached access must satisfy mapping, permission, alignment, and width requirements.

No null, bounds, alignment, dependency-success, lifetime, or object-extent validation is visible. Small and zero-derived addresses are not rejected.

No clean error result, rollback, recovery path, or exception translation is established. Faults and non-returning dependencies may interrupt execution after earlier effects.

## Behavior

### Preparation and retained values

Preparation is a statically ordered sequence of indirect invocations, with no represented retry loop or conditional shorter route. The first target is derived from the incoming continuation-related value plus 4094; its represented argument is the context word shifted left by five. Subsequent targets predominantly use fixed continuation-like bases plus the same displacement.

Arguments include shifted primary and upper result words, saved earlier results, secondary-result words, explicit zeros, repeated values, and masks. Common relationships include lower result words shifted left by eight, context shifts by five and eleven, and upper words shifted left by six. Later masks select individual bits and low fields of widths from 11 through 24 bits.

Whole-result unsigned right shifts by 34, 36, and 44 select the upper word shifted right by two, four, and twelve, respectively. Ignored results and zero arguments do not make their invocations dispensable: dependencies may change state.

The complete preparation trace is **documented_only** in this contract.

The following descriptive labels identify retained relationships, not source identifiers or verified object types:

| Label | Meaning |
|---|---|
| Packed base | A saved lower dependency-result word shifted left by 25 |
| Object seed | A different saved dependency result |
| Scaled object base | Object seed multiplied by \(2^{25}\) |
| Payload | Another saved result shifted left by eight |
| Compact index | A later result shifted left by eight, masked to fifteen low bits, then logically shifted right by six |
| Small base | Twice the compact index |
| Final preparation pair | Lower and upper words of the final combined preparation result |
| Fixed-read base | Unsigned 32-bit conversion of the global read at 0x05 |

The packed base and scaled object base have different origins.

**[INFERENCE]** Under 32-bit bit-vector arithmetic, the compact index is a multiple of four from 0 through 508; the small base is a multiple of eight from 0 through 1016. These ranges do not establish valid mappings.

### Opening writes and the cleared-word capture

The direct-memory phase begins without validation. Its opening history:

- Stores the packed base as a word at final upper result plus 20.
- Writes the final upper result’s low byte at packed base plus 9.
- Stores the scaled object base as a word at 256 times the final lower result plus 84.
- Overwrites packed base plus 9 with the packed base’s low byte.

Next, the scaled object base’s offset-20 word is cleared. An unsigned byte is captured from that base plus 5. The small base’s offset-20 word is cleared, and the captured byte is stored, zero-extended to a word, at scaled object base plus 52. Only then is the scaled object base’s offset-20 word captured.

**[INFERENCE—ordinary memory]** If these accesses complete in ordinary byte-addressed memory with stable, non-mirrored address mapping, ordinary read/write semantics, and no external mutation, the captured offset-20 word is **zero**:

- The intervening offset-5 read does not modify memory.
- The intervening small-base word store writes only zero bits, including any overlapping bytes.
- The same scaled base’s offset-52 word occupies a different byte range from its offset-20 word.

Ordinary overlap therefore supplies no mechanism for introducing nonzero bits into the cleared word. No dependency invocation intervenes in this interval.

Memory-mapped access effects, mirrored mappings between distinct address ranges, and external mutation remain **documented_only possibilities**, not established explanations of actual behavior. Without the ordinary-memory assumptions, the contract retains the captured value rather than replacing its load with a constant.

After capture, the small base’s offset-20 word is cleared again. Two byte stores at small base plus 21 receive the captured word’s low byte. Under the ordinary-memory conditions above, both stored bytes are zero.

### Fixed-location boundary and propagation of the capture

The next established boundary is:

1. Read the global associated with **0x05**, converting its value to unsigned 32 bits and retaining the fixed-read base.
2. Write the small base’s low byte to **0x01**.
3. Write the compact index’s low byte at fixed-read base plus 21.

The global’s read width remains unresolved, but its position before the 0x01 write is established.

The payload’s low byte is then written at scaled object base plus 17. The payload is stored as a word at captured offset-20 value plus 84, and scaled object base plus 17 is cleared.

**[INFERENCE—ordinary memory]** Because the captured value is zero under the conditions above, that payload word store targets **0x54**. This conclusion concerns its destination at that point, not the eventual contents of 0x54.

A word captured from fixed-read base plus 84 becomes the **saved link**. That field is then overwritten with the captured offset-20 value—zero under the same ordinary-memory conditions. These accesses use the retained fixed-read base, not a fresh read of 0x05. The saved link itself is not thereby established as zero; its load precedes the overwrite.

### Link-derived mutations and fixed writes

Subsequent accesses propagate the compact index, payload, object-seed shifts, and loaded address-like words through fields at offsets including 9, 20, 36, 52, 68, 84, and 116.

An offset-116 word loaded through the scaled object base supplies an offset-84 destination. Another mutation uses offset-116 reads through the small base to populate an offset-84 field. Repeated self-address stores occur at saved link plus 36, interleaved with low-byte stores through another loaded base.

A later fixed-location boundary has this order:

| Order | Destination and width | Value |
|---:|---|---|
| 1 | Small base plus 20, word | Object seed shifted left by 26 |
| 2 | **0x54**, word | Saved link shifted left by one |
| 3 | Small base plus 20, word, twice | Saved link |
| 4 | Small base plus 21, byte, twice | Low byte of four times the saved link |
| 5 | **0x15**, byte | Small base’s low byte |

Computed destinations may overlap fixed locations.

The **fourfold link** and **eightfold link** are four and eight times the retained saved link. A word loaded from small base plus 116 is doubled. Separately, an unsigned byte loaded from fourfold link plus 5 supplies the base for an offset-116 word load. The resulting values participate in further word and byte mutations.

A word at fourfold link plus 100 supplies a destination receiving a byte at offset 13 and a self-address word at offset 20. A word captured from fourfold link plus 4 supplies later offset-5 unsigned byte reads. Those repeated reads remain distinct; intervening mutations may change their results.

The exhaustive computed-address trace and evaluation ordering within compound expressions are **documented_only** here.

### Constant stores, fixed-base capture, and repetition

A later unsigned-byte-derived offset-116 word load supplies the **constant seed**. Numeric 58 is stored in this order:

| Destination | Width |
|---|---:|
| Four times the constant seed | 8 bits |
| Eight times the constant seed | 16 bits |
| Four times the constant seed plus 32 | 32 bits |

The **adjusted seed** is the fourfold link minus 150. The **adjusted base** is four times the adjusted seed. Its offset-32 word receives numeric 32, numeric 47, a zero-extended unsigned byte read from adjusted base plus 4, and numeric 32 again, in that order.

After the final offset-32 store, a separate fixed-location word is captured as the **fixed base**. Only then does **0x60** receive numeric 32 as a word. Later operations retain that captured base; independent mapping of its storage is not established.

The **fourfold constant** is four times the constant seed. The first repeated group has this ordered effect:

| Order | Destination and width | Value |
|---:|---|---|
| 1 | Eightfold link plus 84, word | Fourfold constant |
| 2 | Fixed base plus 25, byte | Adjusted base’s low byte |
| 3 | Adjusted base plus 17, byte | Fixed base’s low byte |
| 4 | Fixed base plus 52, word | Eightfold link |

Subsequent groups retain this four-store order and the byte values. Their first destination is eight times the adjusted seed plus 84; their fourth value is eight times the adjusted seed. Their first value remains the fourfold constant.

This is finite static repetition, not polling or convergence. Exact multiplicity is not enumerated here and is **documented_only** coverage. Repetition may not be collapsed solely because values repeat.

### Final mixed-width mutations and dispatch

After repetition, a word is captured from **0xA9**, followed by a word from fourfold constant plus 116. Address **0x3A** receives the low byte of eight times the adjusted seed plus 48.

Words are then captured from twice the constant seed plus 20 and adjusted base plus 4 before later overwrites. Final effects include:

- Numeric 108 as a byte at adjusted base plus 8.
- Truncated retained bases at offsets 9 and 13.
- A self-address word at twice the constant seed plus 84.
- The low byte of the 0xA9 capture at **0x67**.
- A word capture from twice the constant seed plus 68.
- A read of 0x67 for the byte store to **0x21**.
- Further reads of 0x67 for adjusted-base offset-9 stores, separated by mutations.

Later reads of 0x67 cannot universally reuse its initial stored value.

The offset-68 capture supplies a byte store at adjusted base plus 16, a word at fixed base plus 84, and a shifted byte at twice the constant seed plus 1. The word at fixed base plus 84 is subsequently overwritten with the adjusted base. The shift count is the adjusted base’s low eight bits; counts of 32 or greater and signed right-shift behavior remain unresolved.

A word captured from adjusted base plus 64 is stored at adjusted base plus 16. The earlier adjusted-base offset-4 capture is stored at fixed base plus 116. Further byte stores are interleaved with word-field mutations, including storing the 0xA9 capture at adjusted base plus 68.

A base loaded from twice the constant seed plus 4 supplies a halfword destination at twice that loaded base, receiving numeric 101. Another loaded destination receives numeric 100 in its offset-84 word.

Near completion, a saved offset-100 word determines a destination at twice that word, receiving another saved word. The final indirect invocation has four represented arguments:

1. Numeric 114.
2. The saved offset-100 word from the later loaded base.
3. The earlier capture from twice the constant seed plus 20.
4. A later offset-116 word from the later loaded base.

The fourth argument is distinct from the earlier fourfold-constant offset-116 capture. Target identity and continuation semantics remain unresolved.

## Side effects

Direct writes span 8, 16, and 32 bits. Dependencies may add memory, global-state, external, and control-flow effects.

No identified allocation or release establishes ownership. Address-like stores do not establish acquisition, transfer, retention, or release. Unknown dependencies are not assumed to preserve globals, memory, or returned-address validity.

Mixed-width overlap matters: offset-21 bytes can overlap offset-20 words, and offset-16 words can overwrite bytes at offsets 16 and 17. Endianness is unresolved.

That general aliasing risk does **not** invalidate the opening zero-capture conclusion under ordinary-memory conditions. Its specific intervening writes cannot introduce nonzero bits into the cleared field. Nonordinary mappings or access effects require separate evidence.

No synchronization, atomicity, transaction, or rollback is established. Represented access order does not supply hardware visibility guarantees.

## Dependencies

- **F-opaque-001:** documentation label for unresolved preparation targets; not an assertion that all invocations reach one function.
- **F-opaque-002:** documentation label for the unresolved final target.

Calling conventions, secondary-result rules, preservation guarantees, ownership obligations, lifetimes, errors, and non-returning behavior remain **documented_only**.

## Acceptance examples

These are conditional review scenarios, not executed tests.

### Cleared word with ordinary overlap

Assume ordinary byte-addressed memory, stable non-mirrored mapping, no external mutation, and successful accesses. Even if the small-base zero store overlaps some or all of the scaled-object offset-20 word, the subsequent capture must be zero. A nonzero offset-5 byte, stored as a word at scaled-object offset 52, does not change that conclusion.

The following two small-base offset-21 byte stores therefore each write zero. The later payload word store uses destination 0x54, and the fixed-read-base offset-84 overwrite writes zero.

### Nonordinary memory

If a device read at scaled-object offset 5 changes the offset-20 field, or distinct address ranges mirror the same storage, the ordinary-memory proof is inapplicable. Such behavior is **documented_only**, not established. The represented offset-20 load and propagation of its actual result must remain accounted for.

### Compact-index range

Under 32-bit bit-vector arithmetic, a compact index of 508 is reachable; 511 is not. Treating it as an arbitrary nine-bit index changes reachable addresses.

### Fixed read and conversion

If the converted 0x05 value is 0x000000FE, the fixed-read base is 254. With small base 0x128, the next 0x01 byte store receives 0x28, and the following indexed byte store targets 254 plus 21.

This conditions on the converted value, not an octet at 0x05. A hypothetical signed eight-bit value of −2 would convert to 0xFFFFFFFE; an unsigned eight-bit value of 254 would convert to 254. Neither declaration is supplied. The read precedes the 0x01 write in either case.

### Captures and ordering

The fixed-base capture remains between the final adjusted-base offset-32 store and the 0x60 word store. The offset-32 store history remains 32, 47, the captured byte zero-extended, then 32.

A retained capture survives later memory overwrites as a saved value. Conversely, explicit later reads of 0x67 must reflect memory at their respective read points.

### Interrupted execution

A fault has no established rollback obligation. A non-returning preparation dependency prevents later represented operations; a non-returning final dependency does not establish normal return.

## Uncertainties

The following remain **documented_only**:

- Application purpose, object types, and field meanings.
- Dependency identities, execution-state convention, errors, ownership, and lifetimes.
- The 0x05 global’s declaration, read width, signedness, and conversion details.
- Complete preparation trace, repeated-group multiplicity, and exhaustive summarized access order.
- Compound-expression evaluation ordering.
- Mapping, permissions, alignment, endianness, and general aliasing.
- Memory-mapped access effects, mirrored mappings, or external mutation that could defeat the conditional zero-capture result.
- Signed overflow, signed right shifts, and large variable-shift behavior.
- Final target identity and continuation semantics.

Recurring offsets suggest structured mutation **[INFERENCE]**, but do not establish structure definitions or invariants.

## Evidence

This contract uses static inspection of the supplied representation. Embedded comments, the previous draft, and reviewer feedback are not independent behavioral authority.

The opening zero-capture conclusion follows conditionally from the clear, intervening byte read, intervening zero word store, distinct same-base offset-52 word store, and subsequent offset-20 load. Ordinary overlapping stores do not explain a nonzero capture.

Explicit accesses support the stated widths. The 0x05 expression establishes an unsigned 32-bit conversion without establishing its underlying read width. Fixed-location ordering and the fixed-base capture boundary are separately represented.

No tools, implementation access beyond the supplied representation, runtime execution, or tests were used. Acceptance examples describe conditional obligations, not observed outcomes.
