<!-- Trusted provenance: F-cfee6567a2c3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cfee6567a2c3

## Purpose

F-cfee6567a2c3 performs an order-sensitive sequence of mixed-width memory reads and writes, prepares selected stack locations, invokes an indirect dependency, and conditionally continues through F-cdec09f5d432 to a terminal operation.

No application-level purpose is established. Repeated offsets do not establish objects, records, ownership, or valid address ranges. The contract concerns observable access order, retained observations, mutable stack storage, and continuation conditions.

All behavior below is conditional on execution reaching the relevant operation with valid arithmetic, accessible storage, and suitable execution state. Successful traversal of the entire path is not established. No runtime tests have been performed.

## Interface

### Arguments and incoming state

The declared interface has three 32-bit integer arguments and no declared return value.

| Argument | Established uses |
|---|---|
| **A** | Stored word payload; address base for reads and byte writes; source of shifted stack payloads |
| **B** | Base for the opening word read at offset `0x14` |
| **C** | Additive address component for the opening halfword store |

Additional incoming state supplies:

- An **origin**, whose `0x14` word supplies the initial **active base**.
- An **entry destination**, receiving A at offset `0x54`.
- A **narrow root**, supplying a narrowed halfword payload and receiving an installation at offset `0x44`.
- An indirect execution target.

The ABI supplying this additional state is **documented_only**.

Eight literal-backed selections supply successive **anchors**. An anchor is a 32-bit address-like value, not an established object. Anchors need not be distinct. Selection occurs when its stage is reached, rather than collectively at entry. Literal-backed storage is not established as immutable.

### Widths, layout, and arithmetic

Offsets are byte offsets. Explicit external accesses are 8, 16, or 32 bits wide. Designated stack assignments are 32-bit.

A **selector** is a signed 16-bit read promoted to 32 bits. A selector with bits `0xffff` therefore contributes negative one. Sampled plain bytes have unresolved signedness: promotion of `0xff` may produce negative one or 255.

Address calculations include 32-bit additions, signed multiplication, doubling, and fixed left shifts by one, four, or eight. Universal wrapping arithmetic is not established. Signed overflow, invalid signed shifts, integer-to-address interpretation, and implementation-dependent narrowing remain **documented_only**. The fixed shift counts themselves are nonnegative.

The designated stack layout is relative to the stack-base address passed to the indirect dependency. The highest designated word occupies offsets `0x1b8`–`0x1bb`; this does not establish the complete accessible stack extent.

## Behavior

### Observation and aliasing rules

The ordering described here is the order represented by the supplied static evidence, not a hardware memory-ordering or atomicity guarantee.

A retained observation preserves its sampled value; it does not freeze reachable storage. Other described reads are fresh observations. Address-derived writes may overlap subsequent read locations, literal backing, or stack storage.

In particular, stack offset `0x108` is mutable storage, not an immutable retained tail value. Its initialization and subsequent observations are distinguished below. Any description equating a later observation with its initial contents is conditional on that slot remaining unchanged.

No validation, bounds checking, recoverable error branch, or rollback is represented.

### Opening operations

The origin’s `0x14` word supplies the initial active base. Before reading through B, the function writes an origin-derived byte at active base plus `0x0d`. This payload derives from the origin value itself, not a byte loaded through the origin.

Next, B plus `0x14` supplies another base. A is stored at entry destination plus `0x54`; then a narrow-root-derived halfword is stored at C plus the loaded base. Only afterward is the loaded base’s `0x44` word sampled as the initial **carried value**. Either preceding store may affect that observation through overlap.

The active base is stored at its own `0x54` field. Anchor 1 is selected, its literal-backed installation value is stored at narrow root plus `0x44`, and the carried value is stored at retained anchor 1 plus `0x14`.

### Transitions between anchors

Each transition first reads a designated `0x44` source for the next carried value. It then stores the current active base at its own `0x54` field, selects the next anchor, performs its installation, and writes the carried value at retained new anchor plus `0x14`.

| Transition after anchor | Source base for the next `0x44` read | Installation base for the next anchor |
|---|---|---|
| 1 | Anchor 1 | Byte sampled in stage 1 |
| 2 | Anchor 2 | Byte sampled in stage 2 |
| 3 | Anchor 3 | Byte sampled in stage 3 |
| 4 | Destination loaded in stage 4 | Stage 4 witness |
| 5 | Destination loaded in stage 5 | Stage 5 witness |
| 6 | Anchor 6 | Stage 6 witness |
| 7 | Anchor 7 | Stage 7 witness |

Every installation writes a word at installation base plus `0x44`. A sampled byte used as a base is promoted according to unresolved plain-byte signedness; it is not a loaded pointer.

The anchor address retained for forwarding comes from its selection. The installation separately references the corresponding literal-backed value. Equality of separately observed backing values requires that the backing remain unchanged.

### Byte-selected stages: anchors 1–3

Each stage starts with a byte write at active base plus `0x11`, derived from active base shifted left once. It obtains a signed selector at active base plus current anchor, then samples a byte at selector plus carried value, subject to the stage-specific ordering below.

**Anchor 1:** After the selector and byte observations, an entry-destination-derived byte is written at anchor plus `0x1d`. The sampled byte is widened into a halfword at selector plus anchor.

**Anchor 2:** After the byte sample, a fresh word at A plus `0x64` is read and stored at selector plus `0x54`. A carried-value-derived byte is written at sampled-byte base plus `0x0d`. Anchor 2 plus `0x64` is then sampled as the **persistent payload**. Finally, the sampled byte is widened into a halfword at selector plus anchor.

**Anchor 3:** A fresh A-plus-`0x64` word is retained after the opening `0x11` write but before the selector and byte observations. It supplies the next stage’s opening byte.

After sampling the selector and byte, stage 3 writes, in order:

1. An anchor-derived byte at A plus `0x01`.
2. A carried-value-derived byte at anchor plus `0x0d`.
3. The persistent payload as a word at anchor plus `0x44`.
4. An anchor-derived byte at sampled-byte base plus `0x0d`.
5. The widened sampled byte as a halfword at selector plus anchor.

The transition then rereads anchor 3 plus `0x44`. Intervening narrow writes can make that observation differ from the persistent payload previously stored there.

### Witness-selected destinations: anchors 4–5

A **witness** is a word read at active base plus `0x44`. In both stages, that observation precedes the selector read at active base plus current anchor.

**Anchor 4:** The opening active-base `0x11` byte comes from the A-derived word retained in stage 3. A new A-plus-`0x64` word is retained before the witness and selector reads.

The carried value’s `0x74` word then supplies a destination. The old active base’s `0x14` word supplies the replacement active base. Both reads precede a destination-derived byte write at witness plus `0x0d`, followed by a witness-derived halfword at selector plus loaded destination.

The transition reads the loaded destination’s `0x44` word and uses the replacement active base for the self-store.

**Anchor 5:** The opening `0x11` byte comes from the A-derived word retained in stage 4. After the witness and selector reads, an anchor-derived byte is written at carried value plus `0x15`, then the carried value is stored at anchor 5 plus `0x44`.

Only after those writes does witness plus `0x74` supply the halfword destination. A witness-derived halfword is stored at selector plus that destination. The transition reads that destination’s `0x44` word.

Neither halfword destination is required to equal its stage’s anchor.

### Selector-selected active bases: anchors 6–7

These stages replace the active base using a word read at selector plus `0x14`. The selector itself is obtained using the old active base.

**Anchor 6:** The opening `0x11` byte derives from old active base shifted left once. A fresh A-plus-`0x64` word is retained for stage 7. The witness, selector, and replacement-active-base reads follow in that order.

**Anchor 7:** The opening `0x11` byte uses that retained A-derived word. Another fresh A-plus-`0x64` read precedes the witness and selector observations.

Before reading the replacement active base, stage 7:

1. Stores the old active base at carried value plus `0x54`.
2. Updates old active base plus `0x11` using the newly sampled A-derived byte.
3. Stores anchor 7 at witness plus `0x44`.

The newly derived byte is retained for stage 8.

After active-base replacement, both stages perform these ordered effects:

1. Store the witness at its own `0x54` field.
2. Store the persistent payload at current anchor plus carried value.
3. Store the witness at current anchor plus `0x14`.
4. Read persistent payload plus `0x44` and write a byte derived from that word at current anchor plus `0x0d`.
5. Write a witness-derived byte at replacement active base plus `0x0d`.
6. Write a witness-derived halfword at selector plus current anchor.

The persistent payload is not refreshed. Its reachable `0x44` storage is sampled separately in these stages and may differ.

### Multiplication-dependent stage: anchor 8

After the final transition, active base plus `0x11` receives the byte retained in stage 7. Active base plus `0x44` supplies a retained multiplication factor; the signed selector is read next at active base plus anchor 8.

Before any product-dependent read, two identical write pairs occur. Each stores the selector as a word at selector plus `0x54`, then writes an active-base-derived byte at anchor 8 plus `0x11`. Both pairs are represented effects.

The **first product** is selector multiplied by the retained factor. Reads follow through:

- First product plus `0x44`.
- The resulting base plus `0x74`.
- That result plus `0x14`, yielding the **chain destination**.

The chain destination’s existing `0x14` word is saved before the field is replaced with active base shifted left once.

The saved old value remains the base for a fresh `0x14` read. Selector multiplied by that fresh word gives the **second product**. A selector self-store at selector plus `0x54` and a saved-old-value-derived byte at anchor 8 plus `0x11` precede the second product’s `0x74` read.

That read initializes stack offset `0x108`. Its sampled value is the **initial tail word**. Subsequent uses refer to the stack slot, not necessarily that initial value.

Multiplying the second product by the selector gives the **later product base**. The remaining order is:

1. Store the selector at selector plus `0x54`.
2. Observe stack `0x108`; narrow that observation into the byte written at anchor 8 plus `0x11`.
3. Read later product base plus `0x44`, retaining the **first product-field sample**.
4. Read A plus `0x74`, retaining its word.
5. Observe stack `0x108` again; store that observation as a word at anchor 8 plus `0x64`.
6. Write a byte derived from the first product-field sample at anchor 8 plus `0x09`.
7. Reread later product base plus `0x44`; write a byte derived from this second sample at later product base plus `0x11`.
8. Store the selector at selector plus `0x54`.
9. Observe stack `0x108` again; write a byte derived from that observation at twice the later product base plus `0x11`.
10. Read the `0x44` word at selector multiplied by later product base, retaining the **final product-field sample**.
11. Read A plus `0x14` to obtain the final destination base.
12. Observe stack `0x108` again; use that observation as the base for a byte write at offset `0x01`, whose payload derives from the earlier A-plus-`0x74` sample.
13. Write a final-product-field-derived byte at final destination base plus `0x11`.
14. Write a doubled-later-product-derived byte at final destination base plus `0x09`.

Each intervening address-derived write may overlap stack `0x108`, wholly or partially. Consequently, these slot observations may differ. The initial tail word determines all of them only if the slot remains unchanged.

A store targeting the slot uses the payload or address already obtained for that store; it can affect later observations without retroactively changing its own sampled input.

### Stack preparation and continuation

After the final external-memory writes, designated stack assignments occur in this order:

| Offset | Word assigned |
|---|---|
| `0x10` | A shifted left four |
| `0x34` | A shifted left eight |
| `0xe4`, `0x60`, `0xa8`, `0x1b8` | Zero, in listed order |
| `0x24` | Value from stack `0x10` |
| `0x4c` | Value from stack `0x34` |
| `0x128` | Fresh observation of stack `0x108` |
| `0x13c` | Value from stack `0x34` |
| `0x17c` | Another observation of stack `0x108` |
| `0x188` | Value from stack `0x34` |

The copies at `0x128` and `0x17c` are not unconditionally copies of the initial tail word. They follow all remaining address-derived writes, including the final three byte writes. The designated assignments between these two observations do not themselves overlap `0x108`; absent external interference, both copies therefore agree with the slot’s then-current contents.

The indirect dependency receives the stack-base address and two zero arguments. These assignments do not initialize a complete object, and address-derived writes may have modified other stack locations.

If that dependency returns, stack assignments occur in this order: zero at `0x08`, `0x08400e1c` at `0x19c`, then zero at `0x100`, `0x120`, and `0x140`.

Only afterward is a separate literal-backed value obtained for the first argument to F-cdec09f5d432. Its other three arguments are zero. The indirect dependency cannot be assumed to preserve that backing storage.

If F-cdec09f5d432 returns, a terminal architectural facility receives selector zero and fixed site operand `0x08401154`. Its resulting target is invoked. This path is annotated as nonreturning; the platform outcome is unresolved. No ordinary return path is represented.

## Side effects

The function mutates argument-derived, incoming-state-derived, literal-derived, loaded, arithmetic-derived, and stack addresses. Byte and halfword writes affect their represented widths, not an assumed enclosing field.

Stack `0x108` has an explicit initial assignment and later observations. Overlapping address-derived writes can change its contents and thereby change later payloads, an address base, and the stack copies exposed to the indirect dependency.

No allocation, release, reference counting, or ownership transfer is established. An address-like payload does not grant ownership.

Passing the stack-base address does not establish the dependency’s permitted extent, mutation rights, retention rights, or lifetime obligations. Neither dependency may be assumed to preserve globals or reachable memory.

Faults and nonreturning dependencies can prevent later effects without reversing earlier ones. Fault delivery, partial-access behavior, and consequences of invalid arithmetic are **documented_only**.

## Dependencies

| Identity | Established interaction | Unknown properties |
|---|---|---|
| Entry-supplied indirect target; opaque function ID unavailable | Stack-base address and two zeros | ABI, effects, extent, retention, ownership, return behavior |
| F-cdec09f5d432 | Freshly obtained literal-backed argument and three zeros, after the indirect call returns | Argument meaning, effects, ownership, return behavior |
| Terminal facility and resulting target; opaque function IDs unavailable | Selector zero, fixed site operand, then target invocation | Exception semantics, target behavior, termination outcome |

No dependency implementation is supplied. No preservation or memory-management contract is inferred.

## Acceptance examples

These are conditional static expectations, not executed tests.

- **Opening overlap:** If the opening halfword store overlaps the following loaded-base `0x44` word, the word is sampled after that store.
- **Signed selector:** Selector bits `0xffff` contribute negative one, without establishing address validity or overflow safety.
- **Byte promotion:** A sampled byte with bits `0xff` may contribute negative one or 255 as an address component; its widened halfword may be `0xffff` or `0x00ff`.
- **Retained payload:** Changing anchor 2’s `0x64` storage after sampling does not refresh the persistent payload. Changing storage at persistent payload plus `0x44` can affect subsequent fresh reads.
- **Delayed destination:** If stage 5’s preceding writes overlap witness plus `0x74`, destination selection observes their effects.
- **Saved old field:** Updating chain destination plus `0x14` does not replace the saved old value used as the next read’s base.
- **Repeated product observation:** Intervening writes can make the two later-product `0x44` samples differ. The second supplies the later-product `0x11` byte.
- **Early tail-slot overwrite:** If the selector self-store immediately after initialization overwrites stack `0x108`, the subsequent anchor-8 `0x11` byte uses the changed slot contents, not necessarily the initial tail word.
- **Tail-slot changes between uses:** If the anchor-8 `0x64` store overlaps `0x108`, later uses of the slot may change even though the payload for that store was sampled beforehand.
- **Final address selection:** If an intervening write changes `0x108` before its final address-forming observation, the final offset-`0x01` byte targets the changed base.
- **Late stack copies:** If a final byte write changes `0x108`, the copies at `0x128` and `0x17c` reflect its later contents. If the slot remains unchanged throughout, all tail-slot uses equal the initial tail word.
- **Partial preparation:** The post-call zero at stack `0x100` is not a designated pre-call initialization.
- **Continuation boundary:** If the indirect target does not return, no designated post-call assignments or later invocations occur.
- **Mutable backing:** If the indirect dependency changes the later argument’s literal backing, F-cdec09f5d432 receives the subsequently obtained value.

## Uncertainties

The following remain **documented_only**:

- Application purpose, complete entry ABI, concrete literal values, and actual alias relationships.
- Plain-byte signedness, narrowing details, overflow behavior, and integer-address semantics.
- Endianness, alignment, access validity, fault behavior, and concurrent observation.
- Concrete effects of partial overlaps with stack `0x108`; these depend on addresses, widths, and byte order.
- Complete stack contents and accessible extent beyond designated assignments.
- Dependency effects, ownership, retention, preservation, and return behavior.
- Terminal architectural outcome and behavior of omitted regions.

Repeated forwarding suggests propagation of state, but that is inferred intent only. It establishes no object-layout, validity, or ownership invariant.

## Evidence

Established evidence is limited to the supplied static representation: three 32-bit arguments, additional incoming state, mixed-width accesses, signed selectors, eight literal-backed anchor selections, fresh and retained observations, multiplication-dependent addressing, explicit stack assignments and later stack-slot references, dependency invocations, and a nonreturning terminal annotation.

The references to stack `0x108` occur after intervening address-derived writes. Their values therefore cannot be described unconditionally as an immutable initial tail word while stack overlap remains possible.

Annotations identify omitted unreachable regions; their behavior is not reconstructed. No dependency implementation was inspected. No runtime execution or tests were performed, and complete-path executability has not been established.
