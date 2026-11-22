<!-- Trusted provenance: F-9d83e0d9ceed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9d83e0d9ceed

## Purpose

F-9d83e0d9ceed performs an ordered sequence of unresolved indirect invocations, transforms dependency results and ambient execution values, mutates memory through computed and fixed addresses, and prepares four arguments for a final unresolved indirect transfer.

No application-level purpose is established. The visible constants, offsets, and repeated writes do not establish text generation, initialization, allocation, serialization, or container maintenance.

This contract separates:

- **Established static evidence:** visible argument transformations, access widths, captures, mutations, and ordering in the supplied representation.
- **Inferred intent:** possible interpretations that impose no acceptance requirements.
- **documented_only:** behavior not resolved by the available evidence.

All behavioral claims are conditional on the supplied representation accurately describing execution. No runtime tests have been performed.

## Interface

### Inputs

The explicit interface accepts two opaque 32-bit words. Their application types and signedness are unknown. Both are forwarded unchanged to the first indirect invocation.

Additional inputs arise from execution context:

- An ambient unsigned 32-bit word, logically shifted right by 11 for the first invocation.
- Another ambient unsigned 32-bit word, logically shifted right by 12 for later transformations.
- An incoming control-link value used to derive the first indirect target.
- Primary and secondary dependency-result words.
- Represented 64-bit dependency results.
- Fixed globals and memory reached through computed addresses.

The two explicit arguments alone do not determine behavior. Any replacement interface must also account for these contextual inputs.

### Outputs and completion

No explicit return value is represented. Observable outputs include dependency invocations, memory reads and writes, and the final indirect transfer.

That transfer receives four visible argument words:

1. `0x72`.
2. A captured word from the late-selected base plus `0x64`.
3. An earlier captured word from the tail base plus `0x14`.
4. A subsequently captured word from the late-selected base plus `0x74`.

These are retained values, not fresh reads at dispatch time.

An apparent return follows the final transfer. Whether that transfer returns, and whether the represented function boundary matches the executable calling convention, remain **documented_only**.

### Widths and arithmetic

Computed-address accesses explicitly include:

- 8-bit loads and stores.
- 16-bit stores.
- 32-bit loads and stores.

Represented 64-bit results are separated into low and high 32-bit words. Unsigned-byte loads are zero-extended when used as words. Explicit byte destinations receive an 8-bit value; halfword stores of `0x003a` and `0x0065` affect two bytes, not one.

Bit-selection conclusions below use a low-32-bit interpretation of shifts and scaling. This does not establish executable treatment of signed overflow, address overflow, or oversized variable shifts. Alignment and endianness are unknown.

Fixed-global declarations are absent. A represented conversion or assignment does not, by itself, establish the destination’s physical storage width.

Two distinctions are particularly important:

- The global at `0x05` is converted to an unsigned 32-bit word. It is not established as an unsigned byte, so its converted value is not restricted to `0–255`.
- The assignment at `0x3a` converts the adjusted value to **plain char before arithmetic**. Integer promotion precedes multiplication by eight and addition of `0x30`. No explicit final 8-bit truncation is represented. Plain-char signedness, conversion of out-of-range values, and the destination conversion and storage width remain **documented_only**.

### Preconditions and ownership

Successful completion requires all executed indirect targets and accessed memory ranges to support the represented operations. This includes accesses to low fixed addresses and addresses derived from small integers.

No allocation, release, or ownership transfer is identifiable in the visible body. Unresolved dependencies may nevertheless retain, invalidate, allocate, or release memory. Their calls do not establish preservation of globals, memory contents, pointers, or lifetimes.

## Behavior

### Indirect invocation prefix

A long straight-line invocation prefix precedes the explicit memory-mutation region. No local validation, retry, conditional branch, or early-return decision is visible.

The first target is derived from the incoming control-link value plus `0xffe`. It receives the two explicit input words and the first ambient word logically shifted right by 11.

Subsequent targets are predominantly continuation-relative, using the same displacement. This does not establish that different invocation sites resolve to the same target.

Visible argument patterns include:

- No explicitly represented arguments.
- One or more zero-valued words.
- Earlier results shifted left by eight.
- Separate low and high words of represented 64-bit results.
- Duplicated, masked, or independently shifted results.
- Secondary result-register values whose complete producer signatures are unavailable.

An invocation with no represented arguments is not proven to have a genuinely zero-argument interface. A represented 64-bit result may describe two result registers rather than an application-level 64-bit quantity.

Invocation order is significant. Unused primary results, repeated arguments, and arguments that reduce to zero do not justify removing, merging, caching, or reordering calls.

### Bit-selection consequences

Under the stated 32-bit interpretation:

| Transformation | Established consequence |
|---|---|
| Low 11 bits shifted left by 10 | Only source bits 0–10 contribute. |
| Low 24 bits shifted right by 8 | Unsigned extraction of source bits 8–23. |
| Low 24 bits shifted right by 12 | Unsigned extraction of source bits 12–23. |
| Low 24 bits shifted right by 19 | Five-bit result. |
| Low 18 bits shifted right by 11 | Seven-bit result. |
| Low 17 bits shifted right by 4 | Thirteen-bit result. |
| Left shift by eight, then mask to 17 bits | Only the original low nine bits contribute. |
| Low 24 bits shifted right by four, then left by 13 | Positioned field subject to 32-bit truncation. |

Several apparently wide expressions lose all information:

- A five-bit value shifted left by 31 retains only its least significant bit; shifting that result left by another 30 produces zero.
- A word shifted left by 14 and then by 23 produces zero.
- A word shifted left by 16 has a zero low halfword; masking that halfword and shifting right by five produces zero.
- Values explicitly reset to zero remain zero through later shifts.

These conclusions concern argument values, not dependency effects.

### Derived value roles

The following names describe roles, not recovered structure types:

| Role | Derivation |
|---|---|
| Shifted base | A dependency result scaled by `2²⁵`. |
| Packed base | A different dependency result shifted left by 25. |
| Shifted payload | An earlier dependency result shifted left by eight. |
| Small value | A result shifted left by eight, masked to 15 bits, then logically shifted right by six. |
| Small base | Twice the small value. |
| Converted-global base | Unsigned 32-bit conversion of the global at `0x05`. |
| Anchor | Word captured from the converted-global base plus `0x54`. |
| Adjusted value | Four times the anchor, reduced by `0x96`. |
| Derived base | Four times the adjusted value. |

The small value is a multiple of four between 0 and 508. The small base is a multiple of eight between 0 and 1016. These bounds establish possible low-address accesses, not heap-object provenance.

### Initial mutations and fixed accesses

The last represented 64-bit prefix result supplies separate low and high words. Initial mutations have this order:

- A word store at the high word plus `0x14`.
- A byte store at the packed base plus `0x09`.
- A word store at the low word scaled by 256 plus `0x54`.
- Another byte store at the packed base plus `0x09`, using a different source.

The shifted base’s `0x14` word is zeroed before a byte is loaded from its `0x05` location. That captured byte is later stored, zero-extended, at offset `0x34`.

The small base’s `0x14` word is zeroed twice, with intervening accesses. A reread of the shifted base’s `0x14` word supplies later data and an address. Aliasing prevents replacing that reread unconditionally with zero.

After two byte stores at the small base plus `0x15`, the global at `0x05` is converted and captured. The following fixed accesses occur in the indicated order relative to surrounding computed accesses:

| Fixed location | Represented effect |
|---|---|
| `0x01` | Receives an 8-bit conversion of the small base, after the `0x05` capture and before a byte store at the converted-global base plus `0x15`. |
| `0x54` | Receives twice the anchor, after a dependency-derived word is stored at the small base plus `0x14` and before two stores of the anchor to that computed field. |
| `0x15` | Receives an 8-bit conversion of the small base, after two computed byte stores using four times the anchor and before later word stores through anchor-derived addresses. |
| `0x60` | Receives the represented value `0x20`, after the repeated-region base is captured and before repeated mutations begin. |

Between the assignment at `0x01` and acquisition of the anchor, the shifted payload is stored through several computed addresses. A byte store at the shifted base plus `0x11` is subsequently overwritten with zero.

The anchor is captured from the converted-global base plus `0x54` before that location is overwritten with the earlier shifted-base `0x14` capture.

Later pointer-following operations use words at offsets `0x04`, `0x64`, and `0x74`. One selected address is obtained through a zero-extended byte load followed by a word load. Some expressions contain multiple reads of the same field; neither caching nor relative operand-evaluation order is established merely by a shared address.

### Layout and intermediate mutations

No common record layout is established.

| Offset | Visible computed-address access forms |
|---|---|
| `0x01`, `0x05`, `0x08`, `0x09`, `0x0d` | Byte reads or writes, depending on the base |
| `0x10` | Byte and word writes |
| `0x11`, `0x19` | Byte writes |
| `0x14`, `0x15` | Overlapping word and byte accesses |
| `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54` | Word accesses |
| `0x64` | Word reads |
| `0x74` | Word reads and a tail word write |

Fourfold and eightfold anchor-derived addresses undergo repeated mutations. Earlier byte or word stores can affect later pointer-following loads when ranges alias.

A subsequently selected word supplies addresses for three stores of `0x3a`:

- An 8-bit store at four times that word.
- A 16-bit store at eight times that word.
- A 32-bit store at its fourfold address plus `0x20`.

These explicit-width stores are distinct from the later fixed-global assignment at `0x3a`.

The derived base’s `0x20` word receives, in order:

1. `0x20`.
2. `0x2f`.
3. A zero-extended byte loaded from the derived base plus `0x04`.
4. `0x20` again.

These are distinct writes, not one final assignment.

### Repeated mutation region

A word captured from fixed location `0x087fbbbc` becomes the **captured base**. Its acquisition precedes the fixed assignment at `0x60`.

Each ordinary repeated group performs four ordered stores:

- A word at eight times the adjusted value plus `0x54`, containing the selected word’s fourfold value.
- A byte at the captured base plus `0x19`, containing the derived base’s low byte.
- A byte at the derived base plus `0x11`, containing the captured base’s low byte.
- A word at the captured base plus `0x34`, containing eight times the adjusted value.

The first group differs: its first destination uses eight times the anchor plus `0x54`, and its last stored value is eight times the anchor.

No cursor advances and no dependency invocation appears within this region. Repetition is finite and statically represented, not input-controlled. Exact multiplicity is **documented_only** here; it must not be replaced by a guessed count or a variable-length termination rule.

### Tail captures and the assignment at `0x3a`

The previously selected word supplies a **tail base** by doubling and a **fourfold base** by quadrupling.

After repeated mutations, the tail captures a represented 32-bit value from fixed location `0xa9`, followed by a word from the fourfold base plus `0x74`.

The fixed-global assignment at `0x3a` occurs next. Its value formation has three distinct stages:

1. Convert the adjusted value to plain char.
2. Integer-promote that converted value, multiply by eight, then add `0x30`.
3. Convert the resulting arithmetic value for assignment to the undeclared destination.

There is no explicit conversion of the completed expression to eight bits. Consequently, the assignment cannot be specified as storing the low byte of eight times the adjusted value plus `0x30`.

Plain-char signedness and out-of-range conversion behavior are unresolved. The destination’s type, assignment conversion, and physical storage width are also unresolved. For an adjusted value of `0x42`, the character conversion yields 66 and the promoted arithmetic produces `0x240` before destination conversion. A final stored value of `0x40` would require an additional, unestablished narrowing rule.

Only after this assignment does the operation capture:

- The tail base’s `0x14` word, retained for final argument three.
- The derived base’s `0x04` word, retained for a later store.

Thus, if the assignment at `0x3a` overlaps either capture’s memory range, its actual destination width can affect subsequent behavior.

### Tail mutation ordering

Computed byte stores affect the tail base plus `0x0d` and the derived base plus `0x08`, `0x0d`, and `0x09`. The derived-base `0x08` byte receives `0x6c`.

A word store at the tail base plus `0x54` precedes assignment of an 8-bit conversion of the captured `0xa9` value to fixed location `0x67`.

The `0x67` interactions are separate observations:

1. Assignment from the captured `0xa9` value.
2. A word load from the tail base plus `0x44`.
3. A read of `0x67` for assignment to fixed location `0x21`.
4. A later read for a byte store at the derived base plus `0x09`.
5. Another read for another byte store at that same computed destination.

The intervening operations prevent substituting the original captured byte without an aliasing and memory-model guarantee.

Between the `0x21` assignment and the next `0x67` read, mutations involve the derived base plus `0x10` and the captured base plus `0x54`. A byte at the tail base plus `0x01` comes from a signed right shift whose count is the derived base’s low eight bits. Counts outside the ordinary 32-bit range are not resolved.

A word is loaded from the derived base plus `0x40`. Before that word is written to the derived base plus `0x10`, the earlier derived-base `0x04` capture is written to the captured base plus `0x74`. This is a word write from a retained value, not a fresh `0x04` read. A word store at the captured base plus `0x44` follows.

Between the two later `0x67` reads, the tail also writes a byte at the tail base plus `0x09` and the captured `0xa9` word at the derived base plus `0x44`.

### Final dispatch preparation

A word loaded from the derived base plus `0x54` selects further mutation addresses. The earlier tail-base `0x14` capture is restored into that base’s `0x54` field. The low byte of the fourfold-base `0x74` capture supplies additional byte stores.

A word read from the tail base plus `0x04` selects the late base. A halfword `0x0065` is stored at twice that late base before its `0x64` word is captured.

Subsequent operations include:

- A word store of decimal 100 at another selected base plus `0x54`.
- Capture of the late base’s `0x74` word.
- A word read from the tail base plus `0x64`.
- Stores using the retained late-base `0x64` capture, including a word write at twice that captured value.

The final transfer uses the four designated arguments without refreshing the earlier captures.

## Side effects

Calls and mutations are not transactional. No rollback, locking, null check, status result, or local error recovery is visible.

A fault, interruption, or nonreturning dependency may leave earlier effects intact. No defined error code or diagnostic is established.

Aliasing is material throughout:

- Byte stores can alter later word loads and derived addresses.
- Computed stores can overlap fixed globals.
- Captures retain acquired values; rereads observe memory later.
- Repeated identical stores can remain observable through device memory or instrumentation.
- The unknown destination width at `0x3a` prevents specifying its complete write footprint.

No universal final-memory image follows without concrete addresses, byte order, alias relationships, dependency effects, destination declarations, and successful completion.

## Dependencies

F-9d83e0d9ceed is the only resolved opaque function ID supplied. No outgoing target is mapped to another opaque function ID. Assigning identities to those targets would fabricate evidence.

| Boundary | Established | documented_only |
|---|---|---|
| First indirect invocation | Target derivation and three visible arguments | Identity, complete signature, effects, results |
| Continuation-relative invocations | Visible order and argument transformations | Target equivalence, preservation, ownership, errors |
| Secondary-result producers | Additional result words are consumed | Complete signatures and result convention |
| Final indirect transfer | Four visible argument words | Identity, return behavior, effects, errors |

No dependency is assumed pure, idempotent, nonthrowing, memory-preserving, or ownership-neutral.

## Acceptance examples

These are conditional acceptance examples, not executed tests.

1. **Input forwarding.** For explicit words `0x12345678` and `0x90abcdef`, and first-call ambient word `0x1800`, the first invocation receives those unchanged words and 3.

2. **Unsigned extraction.** Selecting the low 24 bits of `0xffabcdef` and logically shifting right by eight produces `0x0000abcd`.

3. **Small-address bound.** If the contributing result’s low seven bits are all set, the small value is 508 and the small base is 1016.

4. **Global conversion is not byte extraction.** If the global at `0x05` converts to unsigned 32-bit value `0x1234`, that value becomes the converted-global base, not `0x34`. Whether the missing declaration permits that value is unresolved.

5. **Arithmetic before destination conversion.** If the adjusted value is `0x42`, its plain-character conversion is 66, and multiplication by eight followed by addition of `0x30` produces `0x240`. Acceptance must not require the assignment at `0x3a` to store `0x40` unless the destination declaration independently establishes that narrowing.

6. **Character-signedness boundary.** For an adjusted value outside the nonnegative range shared by signed and unsigned plain-character interpretations, the arithmetic result requires the actual character-conversion rules. The contract does not silently choose unsigned low-byte arithmetic.

7. **Intervening global observation.** If the word read following the `0x67` assignment changes that global through device behavior, the subsequent `0x21` assignment uses the later observed value.

8. **Captured word write.** If the derived-base `0x04` capture is `0x11223344`, that retained word supplies the captured-base `0x74` store even if the original memory later changes.

9. **Failure without rollback.** If a derived access faults, earlier calls and writes may already have occurred. No restoration or defined failure result is established.

## Uncertainties

The following remain **documented_only**:

- Application purpose and validity of the apparent function boundary.
- Complete calling convention, ambient inputs, and secondary-result conventions.
- All outgoing dependency identities.
- Fixed-global declarations, particularly at `0x05` and `0x3a`.
- Plain-char signedness and out-of-range conversion behavior.
- Destination conversion and storage width for the assignment at `0x3a`.
- Memory map, byte order, alignment, address wrapping, and pointer provenance.
- Signed-overflow, negative signed-right-shift, and oversized-shift execution semantics.
- Aliasing, device behavior, synchronization, and lifetimes.
- Exact repeated-store multiplicity.
- Fault, exception, interruption, and final-transfer return behavior.

Packed-state manipulation is a possible **[INFERENCE]**, not an established domain contract.

## Evidence

The supplied static representation supports the explicit computed-access widths, ordered mutations, capture-versus-reread distinctions, fixed-global interactions, tail `0x74` word write, and unresolved indirect boundaries.

For the assignment at `0x3a`, the representation specifically places plain-character conversion before promoted multiplication and addition. It supplies neither an explicit final 8-bit conversion nor the destination declaration. Those omissions prevent establishing a final byte value or write width.

Comments, symbolic labels, prior prose, and reviewer feedback are not independent proof. Missing declarations and dependency contracts have not been reconstructed from names or recurring constants.

No instruction-level validation, dependency inspection, implementation access beyond the supplied representation, or runtime testing has been performed.
