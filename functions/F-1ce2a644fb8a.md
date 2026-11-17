<!-- Trusted provenance: F-1ce2a644fb8a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1ce2a644fb8a

## Purpose

F-1ce2a644fb8a invokes an ordered sequence of indirect dependencies, transforms selected input and dependency-result bits, and performs mixed-width memory mutations through computed addresses and fixed storage. Its final represented invocation passes four explicit arguments to F-D03.

No application-level purpose is established. Constants resembling character codes do not establish text output, formatting, or serialization. Repeated offsets and link-like values do not establish a particular object model.

This contract distinguishes:

- **Established static evidence:** operations, widths, and data relationships explicitly represented in the supplied body.
- **Inferred intent:** possible interpretations that impose no acceptance requirement.
- **documented_only:** behavior not resolved by the supplied evidence.

Ordering and completion requirements are conditional on execution reaching the relevant operation and preceding operations completing normally.

## Interface

### Explicit inputs

| Position | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No explicit reference in the body. |
| Second argument | 32 bits | No explicit reference in the body. |
| Third argument | Signed 32 bits | Supplies shifted and masked arguments to early dependencies. |

The third argument has no visible range, sign, or enumeration check. Its first explicit transformation is a left shift by three positions.

The absence of explicit references to the first two arguments does not establish independence from the complete invocation context. Interface completeness and additional machine state available to dependencies remain **documented_only**.

### Implicit inputs

Execution also depends on:

- An incoming link-related machine value used to derive the initial invocation target.
- Subsequent indirect targets derived from fixed code-related locations.
- Primary and secondary dependency-result channels.
- Existing contents of computed memory locations and fixed storage.
- An unresolved final continuation target.

The explicit arguments alone therefore do not determine the access trace or completion behavior.

One implicit input is the value of fixed storage associated with address `0x05`. The body converts that value to an **unsigned 32-bit quantity** and subsequently uses the converted quantity in address calculations. The storage declaration is absent. Its original access width and signedness are **documented_only**; neither an unsigned byte load nor a range of 0–255 is established there.

### Outputs

There is no explicit return value. Observable outputs comprise dependency invocations, memory accesses, and resulting state changes.

For the final invocation, distinguish these semantic roles:

- **Snapshot base:** the computed base whose `+0x14` word is captured before the final mutation cluster.
- **Late record base:** a word loaded later from the snapshot base’s `+0x04`.
- **Continuation word:** a word loaded from the late record base’s `+0x64`.

F-D03 receives four represented 32-bit arguments, in this order:

1. `0x72`.
2. The continuation word.
3. The retained snapshot from the snapshot base’s earlier `+0x14` read.
4. A later word read from the late record base’s `+0x74`.

Before F-D03, the continuation word also determines a word-store destination by multiplication by two. The third argument remains the retained snapshot rather than a fresh field read.

A return follows F-D03 in the representation. Whether F-D03 returns normally, permanently transfers control, or uses a specialized convention is **documented_only**.

### Preconditions and errors

Normal completion requires every reached invocation target and memory access to be valid under the actual execution model. Explicit pointer-based accesses include 8-, 16-, and 32-bit operations.

No capacity, buffer extent, ownership token, allocation handle, or nonaliasing condition is established. Alignment, byte order, permissions, address translation, and low-address validity are **documented_only**.

There is no visible validation, error result, recovery branch, or rollback. If an operation fails after partial progress, reversal of earlier effects is not established.

## Behavior

### Dependency-driven phase

A long straight-line sequence of indirect invocations precedes the first explicit data-memory store. No conditional branch or loop is visible in that sequence.

F-D01 derives its target from incoming machine state. Its primary result is shifted left by eight positions for the following invocation.

The intervening F-D02 boundaries have zero through four explicitly represented arguments. Arguments include constants, shifted primary results, secondary result-channel values, separated portions of represented 64-bit results, and values retained across intervening invocations.

A boundary with no explicit arguments is not established to consume no machine state. Likewise, an unused primary result does not establish that the dependency has no effects or no relevant secondary result.

Under a 32-bit truncating interpretation, selected arguments derived from the third explicit input correspond to left shifts by three, six, thirteen, and twenty-seven positions. Another argument selects the low eight bits after the three-position shift and then shifts that selection left by five.

These relationships constrain invocation arguments, not final memory contents. Dependencies may modify memory or global state subsequently read by this function.

### Widths and arithmetic

Widths apply to particular operations, not to every access sharing an address or offset.

| Established operation | Behavioral distinction |
|---|---|
| Explicit unsigned byte load | Produces a numerical value from 0 through 255. |
| Byte store | Writes eight bits, retaining the low byte of the supplied value. |
| Halfword store | Writes 16 bits. |
| Word load or store | Reads or writes 32 bits. |
| Separation of a 64-bit result | Distinguishes low and high 32-bit portions. |
| Unsigned right shift | Introduces zero bits. |
| Conversion of fixed-storage value at `0x05` | Produces an unsigned 32-bit quantity; does not establish the preceding access width or signedness. |

Masks select fields of several widths, including 11 through 24 bits. Masking, shifting, and truncation retain their relative order; unrestricted mathematical simplification need not preserve the represented result.

Numerous left shifts, multiplications, and address additions involve quantities represented as signed 32-bit values. Whether these expressions describe defined language-level arithmetic or summarize machine operations is unresolved. Universal modulo-\(2^{32}\) semantics must not be assumed for all such expressions.

A late byte assignment uses a signed right shift with a count obtained from the low eight bits of a computed quantity. Negative-operand behavior and counts outside 0–31 are **documented_only**. A five-bit shift-count mask is not established.

### Initial memory mutations and captures

The explicit memory phase begins with an ordered word store at a dependency-produced base’s `+0x14`, a byte store at another computed base’s `+0x09`, a word store at a scaled dependency result’s `+0x54`, and a second byte store to the same `+0x09`.

The two byte stores have different value expressions, but their stored bytes need not differ. Both accesses remain represented, including their separation by the intervening word store.

A computed `+0x14` word is cleared before an explicit unsigned byte read from that base’s `+0x05`. A separate computed `+0x14` is then cleared before the captured byte is widened for a word store at the former base’s `+0x34`.

The former base’s `+0x14` is subsequently reloaded. That reload cannot be replaced by the earlier zero because intervening stores may alias it. Following another clear, two consecutive byte stores place the low byte of the reloaded word at the same computed `+0x15`.

Only after those duplicate stores does the function read the fixed-storage value associated with `0x05` and convert it to unsigned 32 bits. The converted quantity supplies a base for a byte store at `+0x15` and later word accesses at `+0x54`.

Those subsequent accesses have explicit widths. They do not retrospectively establish the width or signedness of the fixed-storage read that supplied their base.

### Computed-record mutations

The next region combines word-valued link loads, explicit unsigned byte address components, scaled retained quantities, clears, and self-associated stores.

Recurring byte-access offsets include `+0x01`, `+0x05`, `+0x09`, `+0x0D`, `+0x11`, `+0x15`, and `+0x19`. Recurring word-access offsets include `+0x04`, `+0x14`, `+0x24`, `+0x34`, `+0x44`, `+0x54`, `+0x64`, and `+0x74`. Other offsets, including `+0x08`, `+0x10`, `+0x20`, and `+0x40`, require operation-specific width interpretation.

This is an access map, not a structure layout. It establishes no independent objects or nonoverlapping fields.

Explicit fixed-address effects include byte stores at `0x01` and `0x15`, and a word store at `0x54`. The fixed-storage read associated with `0x05` remains separately width-unresolved.

The function repeatedly writes self-associated words, low bytes of retained address-like quantities, and computed `+0x14` fields. Equal eventual values do not eliminate the represented intermediate writes.

Word reads through `+0x74` supply additional destination bases. One assignment separately reads the same `+0x74` field for its destination and stored value. Their relative evaluation order is not established. Under externally changing or read-sensitive storage, the two reads need not return equal values.

Later explicit unsigned byte reads contribute zero-extended numerical values to word stores and address calculations. These are distinct from the unresolved-width fixed-storage read at `0x05`.

### Constants and repeated-store region

A later region writes `0x3A` with three different widths:

- Eight bits at four times a link-derived quantity.
- Sixteen bits at eight times that quantity.
- Thirty-two bits at four times that quantity plus `0x20`.

At another computed base’s `+0x20`, four word stores occur in the value order `0x20`, `0x2F`, a widened unsigned byte, and `0x20`. The byte is read from that base’s `+0x04` after the second store and before the third.

After these four stores, a word is captured from fixed storage to provide the base for the repeated-store region. **That capture precedes the word store of `0x20` at fixed address `0x60`.** The captured base is retained; no intervening reload after the `0x60` store is represented.

The repeated region preserves ordered groups containing:

- A word store to a computed `+0x54`.
- A byte store at the captured base’s `+0x19`.
- A byte store at another retained base’s `+0x11`.
- A word store at the captured base’s `+0x34`.

For the first group, the `+0x54` destination and the `+0x34` value derive from eight times an earlier saved word. Subsequent groups use eight times an adjusted quantity obtained by multiplying that saved word by four and subtracting decimal 150.

The expressions differ, but their results are not guaranteed unequal. The byte-store expressions remain unchanged across the groups. Later groups repeat their represented word-store expressions without explicit intervening reads.

Exact repetition count is not enumerated here and remains **documented_only** for exact trace acceptance. This does not authorize choosing another count or collapsing the region to its final contents.

### Final snapshots and continuation

After the repeated region, a word is read from fixed address `0xA9`, followed by a computed `+0x74` word. A byte store at fixed address `0x3A` uses the low-byte form of eight times the adjusted retained quantity plus `0x30`.

Two further word snapshots precede the mutation cluster:

- The snapshot base’s `+0x14`.
- Another retained base’s `+0x04`.

The first snapshot originates at `+0x14`, not `+0x54`.

The mutation cluster writes low bytes at `+0x09` and `+0x0D`, the byte constant `0x6C` at `+0x08`, and a self-associated word at the snapshot base’s `+0x54`.

The low byte of the fixed `0xA9` word is stored at fixed `0x67`. A computed `+0x44` word is then read before fixed `0x21` is assigned from a read of `0x67`. Later byte assignments also read `0x67`. Intervening accesses may alias it, so these reads cannot universally be replaced by the original `0xA9` snapshot.

A computed `+0x10` byte store precedes a word store at that same offset. Between them, a `+0x40` word is captured and the saved `+0x04` snapshot is written into another base’s `+0x74`. The captured `+0x40` word then supplies the `+0x10` word assignment.

Further mutations affect byte fields and `+0x44` and `+0x54` words. A computed `+0x54` word supplies another destination base.

The snapshot base’s `+0x54` subsequently receives the saved value from its earlier `+0x14` read. **This is a cross-field snapshot assignment, not an established restoration of previous `+0x54` contents.**

A further computed `+0x14` word is captured afterward. The remaining effects include a halfword store of `0x65` at twice the late record base, the continuation-word read through `+0x64`, a byte assignment derived from the recently captured `+0x14`, and a word assignment of decimal 100 to a computed `+0x54`.

The final `+0x74` argument is captured later. Another link is then loaded from the snapshot base’s `+0x64`. That link’s `+0x54` receives the continuation word, and a word store at twice the continuation word receives the link. F-D03 follows with the arguments specified under Interface.

## Side effects

Mixed-width accesses may overlap. Different expressions do not prove distinct addresses, and different value expressions do not prove distinct stored values.

Separate represented operations establish ordering. Within a multi-read expression, a stricter order is not established unless required by address dependence.

Repeated accesses are part of the represented trace. The evidence does not independently establish volatile semantics, hardware visibility, atomicity, synchronization, or device-memory behavior.

The explicit memory phase contains no established allocation or release operation. Dependencies may nevertheless allocate, release, retain references, mutate globals, or transfer ownership. No dependency is assumed pure, memory-preserving, or ownership-neutral.

Accessing storage does not establish ownership of it. Thread safety and reentrancy remain **documented_only**.

## Dependencies

| Opaque ID | Boundary |
|---|---|
| F-D01 | Initial indirect invocation, targeted using incoming machine state. |
| F-D02 | Collective designation for intervening indirect invocation sites; not a claim that they share a target. |
| F-D03 | Final unresolved indirect continuation with four explicit arguments. |

Executable identities, complete prototypes, effects, failures, and ownership contracts remain **documented_only**.

The evidence establishes ordered invocation boundaries and selected result-to-argument relationships. This contract does not enumerate a fully replayable dependency trace. Replacing that sequence with a presumed mathematical equivalent is unsupported.

## Acceptance examples

These examples are conditional static deductions, not observed executions.

### Selected input transformations

Assuming 32-bit truncating shifts, a third argument of one supplies selected derived arguments of 8, 64, 8192, and `0x08000000`. Selecting the low byte after the three-position shift and then shifting by five yields 256.

Under that assumption, `0x20000000` becomes zero after the three-position shift. The dependency sequence is not thereby eliminated.

### Fixed-storage conversion does not imply byte range

If additional valid type information permits the fixed-storage read at `0x05` to produce `0x1234`, its unsigned 32-bit conversion remains `0x00001234`. The contract does not permit replacing it with `0x34` merely because the storage is associated with a low address.

If valid type information instead establishes a signed source value of −1, conversion to unsigned 32 bits yields `0xFFFFFFFF`, not 255. Whether that source value is possible remains **documented_only** without the missing declaration.

### Explicit unsigned byte promotion

An explicitly represented unsigned byte read of `0xFF`, when widened for a word store, supplies 255 rather than −1. This requirement applies to the explicit byte operation; it does not resolve the fixed-storage read at `0x05`.

### Equal bytes do not remove stores

If both initial `+0x09` value expressions have low byte zero, both stores write zero. Acceptance still preserves both stores and their intervening word store.

### Repeated expressions may coincide

If the saved word is decimal 50 and the arithmetic is defined, four times 50 minus 150 is also 50. First and later repeated groups can therefore both supply 400 to the captured base’s `+0x34`. Equality does not justify removing groups.

### Fixed-base capture retains its timing

If aliasing or address translation permits the fixed-base source to interact with the word store at `0x60`, the repeated region uses the value captured before that store. Moving the capture afterward changes the established ordering.

### Cross-field snapshot is not restoration

If the snapshot base’s `+0x14` reads `0x11223344` while its previous `+0x54` contains `0x55667788`, the later snapshot assignment writes `0x11223344` to `+0x54`. F-D03’s third argument is also the retained `0x11223344`, regardless of later changes to `+0x14`.

### Partial progress

Failure after earlier effects does not require rollback. If F-D03 returns normally and the trailing return is valid under the actual convention, completion has no explicit result. If F-D03 does not return, normal completion is not required.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid invocation context, and interface completeness.
- Dependency identities, result-channel meanings, and effects.
- Original declaration, access width, and signedness of fixed storage at `0x05`.
- Signed overflow, negative-operand shifts, and excessive variable-shift counts.
- Address validity, alignment, byte order, permissions, and translation.
- Object boundaries and alias relationships.
- Exact repeated-store count and a fully enumerated dependency trace.
- Dependency allocation, release, retention, and ownership transfer.
- Synchronization, atomicity, reentrancy, and hardware visibility.
- F-D03’s calling convention and return behavior.

Formatting intent suggested by constants is inference only.

## Evidence

This contract uses only the supplied static body. Embedded comments, prior prose, and reviewer feedback are not independent behavioral authority.

The fixed-storage operation associated with `0x05` visibly converts its value to unsigned 32 bits. No supplied declaration or explicit unsigned-byte dereference establishes its original width, signedness, or a 0–255 range. Explicit unsigned byte loads elsewhere provide stronger, operation-specific width evidence and are treated separately.

Other static evidence includes the fixed-base capture before the `0x60` store, the `+0x14` origin of the snapshot later assigned to `+0x54`, mixed-width accesses, retained snapshots, repeated writes, dependency-result data flow, and the final four-argument invocation.

No tools, external implementation access, or runtime tests were used. Acceptance examples express conditional deductions, not measured behavior.
