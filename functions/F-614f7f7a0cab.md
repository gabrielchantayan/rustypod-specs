<!-- Trusted provenance: F-614f7f7a0cab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-614f7f7a0cab

## Purpose

F-614f7f7a0cab performs an ordered sequence of indirect dependency invocations, transforms dependency results into fields and address-like values, and then updates shared storage through direct reads and writes. A final indirect invocation receives a constant and three previously captured words.

**Established static evidence:** The supplied representation exposes three 32-bit inputs, no explicit return value, indirect invocations, and direct 8-, 16-, and 32-bit memory accesses. Later addresses and stored values depend on earlier dependency results and memory captures.

**[INFERENCE] Intent:** Repeated masks, shifts, scaling factors, and offsets suggest packed-state manipulation and state propagation. They do not establish an application, device, protocol, or object model. Constants resembling character values do not establish text output.

This contract describes static evidence, not verified machine execution. Dependency semantics and target-specific arithmetic remain unresolved where identified below.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32 bits | No explicit reference in the body. |
| Second input | 32 bits | No explicit reference in the body. |
| Third input | Signed 32 bits | Left-shifted by two positions and supplied to F-U002 and F-U003. |

Lack of an explicit reference does not establish that the first two inputs are reserved or irrelevant to the actual calling convention.

The third input has no visible sign, range, overflow, or alignment validation. Its transformed value is an invocation argument; no length, capacity, or allocation meaning is established.

### Implicit inputs and preconditions

Execution depends on:

- Entry control-link state and valid indirect targets.
- Primary, represented 64-bit, and secondary dependency result channels.
- Existing shared storage, including a shared-root slot.
- Storage at low numerical addresses and at addresses derived from returned, loaded, scaled, or narrowed values.
- The environment’s integer, address, alignment, and memory-access rules.

Successful completion requires support for every executed invocation and memory access. Ordinary process memory with inaccessible low addresses may not satisfy these requirements. No validated object extent, buffer size, or ownership-bearing handle is established.

### Outputs and completion

No explicit return value is established. Observable outputs include dependency invocations, direct memory effects, and the final invocation of F-U005.

F-U005 receives, in order:

1. `0x72`.
2. A captured 32-bit word from offset decimal `100` of a late-loaded base.
3. A 32-bit word captured earlier from the doubled propagation base at offset `0x14`.
4. A subsequently captured 32-bit word from offset `0x74` of that late-loaded base.

These are saved values, not fresh reads at the invocation boundary. The representation ends with a return after F-U005. Whether that target returns, transfers control elsewhere, terminates execution, or fails is **documented_only**.

## Behavior

### Dependency-driven phase

The first indirect target, F-U001, depends on entry control-link state. Its primary result, left-shifted by eight positions, and the third input, left-shifted by two, become the arguments to F-U002.

F-U002 supplies a represented 64-bit result. F-U003 receives its lower 32-bit portion shifted left by eight, its upper 32-bit portion, and the transformed third input.

The subsequent invocation sequence is long and explicitly ordered. Established argument relationships include:

- Zero-valued arguments.
- Primary results shifted left by eight.
- Separately consumed upper and lower portions of represented 64-bit results.
- Identical transformed values in multiple argument positions.
- Saved results reused after intervening invocations.
- Secondary result-channel values with unavailable production contracts.
- Masked fields subsequently shifted, scaled, or narrowed.

No conditional branch, retry loop, validation path, or early error return is visible in this sequence. A dependency may nevertheless fail or not return.

An invocation with no displayed arguments is not established to have no implicit inputs or side effects. An unused displayed result does not make its invocation dispensable. Similar argument patterns do not establish identical targets.

F-U004 is the last invocation before the direct-memory region. No further dependency invocation is visible until F-U005.

### Widths, extraction, and arithmetic

The representation distinguishes 8-, 16-, 32-, and 64-bit quantities. Numerical separation of a represented 64-bit result into upper and lower words does not establish a memory layout or calling convention.

Unsigned extraction after right-shifting a 64-bit result by 34, 36, or 44 positions corresponds to logically right-shifting its upper word by 2, 4, or 12 positions.

Representative field operations have these effects:

| Transformation | Established field relationship |
|---|---|
| Retain 24 low bits, then shift right by 8 | Select bits 8–23. |
| Retain 24 low bits, then shift right by 19 | Select bits 19–23. |
| Retain 20 low bits, then shift right by 10 | Select bits 10–19. |
| Retain 17 low bits, then shift right by 4 | Select bits 4–16. |
| Shift a 32-bit value left by 8, retain 15 low bits, then shift right by 6 | Under 32-bit bit-vector semantics, place the original low seven bits in positions 2–8. |

Operation order matters: a later right shift does not restore bits lost through an earlier fixed-width left shift. Successive left shifts may therefore yield zero regardless of the original value.

Many left shifts and multiplications have signed operands. Modulo-32-bit results are conditional on bit-vector semantics, not established portable signed-overflow behavior. Integer-to-address conversion and address wraparound are likewise unresolved.

Byte stores retain an eight-bit result; halfword stores affect 16 bits. Explicit unsigned byte loads widen to values from 0 through 255.

A specific late byte calculation **right-shifts a signed 32-bit loaded value by the field base’s low byte**, then narrows the result for storage. The supplied representation does not independently establish sign-extending behavior for negative operands. Signed right-shift behavior for negative values and handling of counts outside 0–31 are **documented_only**. Neither arithmetic shifting nor count masking is an unconditional requirement.

### Opening direct-memory effects

The opening accesses distinguish three sources: the upper word of F-U004’s result, its lower word scaled by 256, and an independently saved scaled base. No equality or disjointness between their derived destinations is guaranteed.

The opening effects preserve this order:

- A word store at offset `0x14` from the upper-word-derived base.
- A byte write at offset `9` from an earlier transformed base.
- A word store at offset `0x54` from the lower word scaled by 256.
- A second write to the same offset-`9` byte, from a different value source.
- A word clear at offset `0x14` from the saved scaled base.
- An unsigned byte capture at offset `5` from that saved base.
- A word clear at offset `0x14` from twice the narrow selector defined by the final row of the field table.
- A word store of the captured, zero-extended byte at offset `0x34` from the saved scaled base.

The cleared saved-base offset-`0x14` word is subsequently reloaded. It is zero only if intervening overlapping effects and environmental memory semantics leave it unchanged. Another selector-derived word clear and two identical selector-derived byte writes remain distinct represented accesses.

### Low-address capture and early propagation

Storage at absolute address `5` is read and converted to an unsigned 32-bit base **before** the byte write at absolute address `1`. The underlying declaration of the address-`5` storage is unavailable; the conversion does not establish its read width.

The address-`1` byte receives the narrowed value of twice the selector. Later accesses use the already captured address-`5`-derived base, without an implied reread.

That captured base controls an offset-`0x15` byte write and a later offset-`0x54` word capture followed by an overwrite. Between the byte write and word capture, other derived writes occur, including a byte write and then a zero-byte write at saved scaled base plus `0x11`. The word capture observes storage at its actual position in this ordering.

The captured offset-`0x54` word is called the **anchor** below. Overwriting its source does not replace the saved anchor.

Early propagation then follows words at offsets including `4`, `0x64`, and `0x74`, and unsigned byte loads that become further bases. It writes word fields at offsets including `0x14`, `0x24`, `0x34`, `0x44`, and `0x54`. Some stores place a base value into a field addressed from that same base; others propagate low bytes of captured address-like values.

This region includes a word write beginning at absolute address `0x54` and a byte write at absolute address `0x15`. Repeated derived loads remain accesses to current storage, not permanent aliases for earlier values.

### Shared-root capture and recurring updates

The following descriptive roles identify captured or calculated values, not validated objects:

| Role | Relationship |
|---|---|
| Anchor | Word captured through the address-`5`-derived base at offset `0x54`. |
| Adjusted base | Four times the anchor minus decimal `150`. |
| Field base | Four times the adjusted base. |
| Propagation word | Word captured through the preceding byte-and-word lookup chain. |
| Doubled propagation base | Twice the propagation word. |
| Fourfold propagation base | Four times the propagation word. |

These relationships carry the arithmetic qualifications above.

Before the root-relative recurring updates, the propagation-word-derived region receives a byte value `0x3A` at four times the propagation word, a halfword value `0x3A` at eight times that word, and a word value `0x3A` at four times that word plus `0x20`.

The word at field base plus `0x20` receives four ordered writes: `0x20`, `0x2F`, an unsigned byte loaded from field base plus `4`, and `0x20` again. The byte load occurs after the first two writes.

The shared root is captured as a 32-bit word **before** the word write of `0x20` at absolute address `0x60`. Later root-relative accesses use the saved root; neither that absolute write nor a possible alias implicitly refreshes it.

The recurring update region preserves a four-access group:

- A word write at an eightfold-scaled base plus `0x54`, storing the fourfold propagation base.
- A byte write at captured root plus `0x19`, using the field base’s low byte.
- A byte write at field base plus `0x11`, using the captured root’s low byte.
- A word write at captured root plus `0x34`, storing the eightfold-scaled base.

The initial group uses eight times the anchor. Subsequent groups use eight times the adjusted base. No root reload is visible within the region.

These repeated effects are separate represented events. Their exact repetition cardinality is not enumerated by this contract; this omission does not authorize coalescing them.

### Late captures and address-`0x67` reads

The late region begins with a **32-bit capture from absolute address `0xA9`**, before:

- A word capture at fourfold propagation base plus `0x74`.
- The byte write at absolute address `0x3A`.
- A word capture at doubled propagation base plus `0x14`.
- A word capture at field base plus `4`.

The address-`0x3A` byte derives from eight times the adjusted base plus `0x30`, narrowed for storage. It does not refresh the earlier address-`0xA9` capture.

The doubled-propagation offset-`0x14` capture remains saved for a later store and F-U005’s third argument.

Before writing absolute address `0x67`, the body writes bytes at doubled propagation base plus `0x0D` and field base plus `8`, `0x0D`, and `9`. The offset-`8` byte is `0x6C`. It also stores the doubled propagation base into its own offset-`0x54` word.

Absolute address `0x67` then receives the low byte of the saved address-`0xA9` word. That same saved full word later supplies a word write at field base plus `0x44`.

Three subsequent explicit reads of address-`0x67` storage have separate consumers:

| Read | Position and consumer |
|---|---|
| First | After a word load at doubled propagation base plus `0x44`; supplies the absolute address-`0x21` byte write. |
| Second | After derived writes and the field-base offset-`0x40` word capture; supplies a byte write at field base plus `9`. |
| Third | After further derived writes and the field-base offset-`0x54` word capture; again supplies a byte write at field base plus `9`. |

Between the first and second reads, the body writes the loaded propagation value’s low byte at field base plus `0x10`, stores that loaded word at captured root plus `0x54`, performs the signed-value right-shift calculation described above, and stores its narrowed result at doubled propagation base plus `1`. It then overwrites captured root plus `0x54` with the field base.

The intervening region also captures field base plus `0x40`, updates captured root plus `0x74` from the saved field-base offset-`4` word, writes the offset-`0x40` capture at field base plus `0x10`, and writes the doubled propagation base at captured root plus `0x44`.

Between the second and third reads of `0x67`, the body writes a byte at doubled propagation base plus `9`, stores the saved address-`0xA9` word at field base plus `0x44`, and captures a word from field base plus `0x54`.

Derived writes may alias `0x67`, so the three reads may differ. The saved address-`0xA9` word remains its original captured value even if its source storage changes.

### Tail propagation and final invocation

The field-base offset-`0x54` capture becomes another destination base. Its offset-`0x0D` byte receives the low byte of the earlier fourfold-propagation offset-`0x74` capture.

The saved doubled-propagation offset-`0x14` word is stored at doubled propagation base plus `0x54`. Field base plus `0x14` is then captured. Subsequent writes update doubled propagation base plus `0x11` and overwrite field base plus `9` and `0x44`; those locations need not retain their earlier values.

The final loaded base comes from doubled propagation base plus `4`. The following ordering is established:

- A halfword store of `0x65` at twice that loaded base.
- A word capture at loaded base plus decimal `100`.
- A byte write at the propagated destination plus `0x11`, derived from the saved field-base offset-`0x14` capture.
- A word store of decimal `100` at that destination plus `0x54`.
- A word capture at loaded base plus `0x74`.
- A further destination-base capture at doubled propagation base plus decimal `100`.
- A byte write at doubled propagation base plus `9`, using the loaded base’s low byte.
- A word store of the saved offset-decimal-`100` capture at the further destination plus `0x54`.
- A word store of that further destination at twice the saved offset-decimal-`100` capture.

F-U005 then receives the four arguments specified under Interface.

## Side effects

### Memory visibility and layout

No atomicity or transactional behavior is established. Intermediate states, duplicate writes, and partial completion may be observable.

Offsets identify individual accesses, not a validated structure layout. For example, a word access at offset `0x74` spans through `0x77` but establishes neither an object size nor accessible capacity.

Distinct address expressions may overlap. Byte order affects how byte writes alter later overlapping word reads. Alignment requirements are unresolved, particularly for low-address and narrow-value-derived accesses.

Source-level access ordering is established where described. Physical bus transactions, compiler treatment of non-volatile storage, and externally observable access cardinality require environmental information.

### Ownership and errors

No dependency contract establishes allocation, release, retention, or ownership transfer. Unknown dependencies prevent an allocation-free or release-free guarantee.

An address-like returned or loaded value does not prove ownership, validity, lifetime, or extent.

No visible validation error, status return, rollback, exception handler, or cleanup path is established. A failing invocation or access may prevent subsequent effects; earlier writes are not visibly undone. Failure reporting, synchronization, reentrancy, and thread safety are **documented_only**.

## Dependencies

The opaque IDs identify invocation sites, not proven distinct callees.

| ID | Established boundary behavior |
|---|---|
| F-U001 | Target depends on entry control-link state; primary result is consumed. |
| F-U002 | Receives the shifted first result and shifted third input; supplies a represented 64-bit result. |
| F-U003 | Receives the transformed lower word, upper word, and shifted third input. |
| F-U004 | Last invocation before direct memory effects; supplies a represented 64-bit result. |
| F-U005 | Final invocation with `0x72` and three saved 32-bit values. |

Remaining target identities and complete contracts are **documented_only**. No global preservation, memory ownership, or lifetime guarantee is inferred from any invocation.

## Acceptance examples

These examples are conditional static consequences, not runtime observations.

### Argument propagation and fixed-width narrowing

If the third input is `3` and F-U001 returns `0x12`, non-overflowing arithmetic gives F-U002 arguments `0x1200` and `12`. If its result has lower word `0x34` and upper word `0x56`, F-U003 receives `0x3400`, `0x56`, and `12`.

Under 32-bit bit-vector semantics, shifting `0x12345678` left by eight produces `0x34567800`, not a widened result. The narrow selector produces `0`, `4`, `508`, and `0` from original values `0`, `1`, `127`, and `128`.

### Saved bases and captures

If address-`5` storage supplies base `0x80`, subsequent associated accesses use that saved base after the address-`1` write. No reread is implied.

If the shared-root capture is `0x1000`, later root-relative destinations use `0x1000`, including when intervening writes change the root slot.

If address `0xA9` supplies `0x123456AB`, the initial address-`0x67` byte receives `0xAB`, while the later field-base offset-`0x44` word write receives `0x123456AB`. If an intervening alias changes `0x67` to `0xCD`, its next explicit read supplies `0xCD`.

### Signed-value right shift

If the late signed 32-bit loaded value is `0x10000000` and the field base’s low byte is `24`, the right-shifted result is `16`; the destination byte receives `0x10`.

For a negative loaded value, this contract supplies no unconditional sign-extension result. For a count of `32` or greater, it supplies no unconditional shifted result or completion guarantee. Acceptance must retain those cases as **documented_only**, not silently choose arithmetic shifting or masked counts.

### Access width, repetition, and failure

An explicit unsigned byte load of `0xFE`, widened for a word store, supplies `254`, not negative two.

The four field-base offset-`0x20` writes remain four represented events. A final value of `0x20` alone does not demonstrate equivalence for write-sensitive storage.

If required low-address storage is inaccessible, successful completion is not promised. No friendly validation error or rollback is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose and semantic meanings of the inputs.
- Complete indirect target identities, result conventions, and side effects.
- Whether represented 64-bit results are semantic integers or paired result channels.
- The underlying read width of address-`5` storage.
- Target byte order, alignment, signed-overflow, address-conversion, and wraparound rules.
- **For the late signed 32-bit right shift:** negative-operand behavior and out-of-range count handling. Sign extension and count masking are not established.
- Storage validity, lifetime, ownership, concurrency, and environmental access semantics.
- Final-target behavior and failure reporting.

Exact recurring-group cardinality is not enumerated here. That documentation limit does not make repeated accesses optional or authorize coalescing them.

## Evidence

This contract rests on static examination of the supplied representation. Embedded comments, the prior draft, and reviewer feedback are not independent behavioral authority.

Established evidence includes input widths, early argument propagation, ordered invocation sites, field transformations, direct access widths and offsets, captured-value reuse, low-address effects, and final argument sources.

The representation establishes the address-`5` capture before the address-`1` write, shared-root capture before the address-`0x60` write, and address-`0xA9` capture before the address-`0x3A` write. It distinguishes the saved address-`0xA9` word from three later reads of address `0x67`.

For the late shifted byte, static evidence establishes a signed 32-bit operand and a count from the field base’s low byte. It does not independently establish negative-operand sign extension.

No dependency bodies, independent machine-instruction verification, or runtime observations were available. No runtime tests have been performed.
