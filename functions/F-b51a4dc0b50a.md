<!-- Trusted provenance: F-b51a4dc0b50a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b51a4dc0b50a

## Purpose

F-b51a4dc0b50a performs an ordered chain of indirect invocations, transforms returned bit patterns, reads and modifies directly addressed and computed memory, and finishes with an unresolved indirect control transfer.

No application-level purpose is established. The evidence does not justify describing the function as a formatter, allocator, object initializer, collection operation, or protocol implementation. Constants resembling encoded characters do not establish text output.

This contract distinguishes:

- **Established static evidence:** behavior explicitly represented in the supplied function.
- **Inferred intent:** possible interpretations that impose no additional requirements.
- **documented_only:** behavior or environmental facts that remain unresolved.

All behavioral requirements are conditional on the represented control flow being accurate, dependencies returning where needed, and reached memory operations being supported by the execution environment.

## Interface

### Explicit inputs

The represented interface accepts two opaque 32-bit values and declares no result.

Both inputs are forwarded unchanged, in their original order, to the first indirect invocation. The enclosing function does not directly dereference either explicit input.

Their signedness, encoding, valid range, pointer interpretation, and ownership are not established. Zero is not explicitly rejected and does not select a no-op path.

### Implicit inputs

The explicit argument pair is insufficient to determine execution. Additional inputs include:

- Two incoming unsigned 32-bit register values. Logical right shifts by 15 supply the third and fourth arguments of the first invocation.
- An incoming control-link value from which the first target is derived.
- Return values and secondary register outputs from indirect dependencies.
- Memory at computed addresses and fixed locations, including `0x05`, `0xA9`, and `0x087FBBBC`.
- An unresolved final control-transfer target.

The first invocation’s third and fourth positions follow the register ordering shown in the supplied representation; they must not be interchanged.

### Output and completion

No scalar success result, error code, returned allocation, or explicit output buffer is established.

If execution reaches final dispatch, four 32-bit argument positions are supplied:

1. `0x72`.
2. A word captured at offset `0x64` from a late-loaded base.
3. A word captured earlier at offset `0x14` from a different derived base.
4. A subsequently captured word at offset `0x74` from the same base used for argument two.

These are captured values, not requirements to reread their source locations at dispatch.

A return is represented after the final transfer. Whether that target actually returns is **documented_only**.

### Preconditions and ownership

Execution requires accessible memory and valid executable targets for every reached operation. No enclosing validation establishes these conditions.

No allocation, release, reference-counting operation, ownership transfer, or lifetime guarantee is established. Address-like values are not necessarily heap pointers. Unknown dependencies must not be assumed to preserve globals, maintain address validity, or own or release memory.

## Behavior

### Execution model

The visible body is straight-line. It has no represented validation branch, retry, data-dependent loop, or explicit error-handling path.

Its broad order is:

1. Indirect invocations and bit-pattern processing.
2. Mixed-width memory mutation and pointer-like propagation.
3. A fixed repeated-write region.
4. Late captures, further mutation, and final dispatch.

Calls whose results are unused remain potentially effectful. Repeated or overwritten stores remain part of the represented behavior.

The ordering constraints below identify important read/write boundaries, especially where aliasing changes results. This document is not an exhaustive transaction-by-transaction trace of every indirect invocation and intermediate store. It does not authorize reordering unenumerated operations or collapsing repeated effects.

### Indirect invocation phase

The initial target is derived from the incoming control-link value by adding `0xFFE`. It receives the explicit argument pair followed by the two unsigned right-shifted implicit inputs.

Subsequent targets are predominantly derived from fixed continuation-like values using the same displacement. Some target expressions originate from instruction-associated addresses. Target identities and calling conventions are unresolved.

The call chain includes invocations with:

- No explicitly represented arguments.
- One or two explicit zero arguments.
- Shifted 32-bit results.
- Low and high halves of represented 64-bit results.
- Several transformed values supplied together.
- Secondary register outputs whose exact producers and contracts are unresolved.

“No explicitly represented arguments” does not mean that registers are cleared or that the callee has no implicit inputs. Similar argument transformations do not establish that targets are interchangeable.

### Widths and arithmetic

The principal data width is 32 bits. Represented 64-bit returns are divided into low and high 32-bit halves; this does not establish 64-bit pointers.

Unsigned masks and logical shifts establish the following extraction ranges:

| Extraction | Range |
|---|---:|
| Unsigned incoming word shifted right by 15 | `0`–`0x1FFFF` |
| Low 24 bits shifted right by 8 | `0`–`0xFFFF` |
| Low 24 bits shifted right by 12 | `0`–`0xFFF` |
| Low 18 bits shifted right by 11 | `0`–`0x7F` |
| Low 17 bits shifted right by 4 | `0`–`0x1FFF` |
| Low 16 bits shifted right by 5 | `0`–`0x7FF` |

Left shifts and multiplication by powers of two do not imply widened storage. Under a 32-bit bit-vector interpretation, excess high bits are discarded. For example, shifting a 32-bit quantity left by 14 and subsequently left by 23 leaves no surviving bits.

Several computations start with explicit zero values, so their later shifted forms remain zero. Calls receiving these zeros may still have arbitrary effects.

Signed-looking intermediate arithmetic does not establish portable signed-overflow behavior. Machine-level overflow, signed right shifts, and exceptional shift counts remain **documented_only** where not resolved by explicit unsigned operations.

### Address derivation and initial stores

Near the end of the call phase, the function retains several address-like quantities:

- An earlier returned low word shifted left by 25, used as a byte-store base.
- Another returned word shifted left by 8, retained as a payload.
- A returned word scaled by `0x02000000`, used as a memory base.
- A small base derived from another returned word.
- The low and high halves of the final represented 64-bit return.

The small-base extraction selects the original result’s **low seven bits**, places them in bits 2 through 8, and then doubles that quantity. Before doubling, the value is four times the original low-seven-bit value, with maximum `0x1FC`. After doubling, it is eight times that value, with maximum `0x3F8`, occupying bits 3 through 9. It is not an extraction of nine source bits.

Initial memory effects preserve these boundaries:

- A word store at the final return’s high half plus `0x14` precedes a byte store at the high-shift base plus `0x09`.
- A word store at the final return’s low half scaled by `0x100`, plus `0x54`, precedes a second byte store at that same high-shift-base offset.
- A zero word is stored at the scaled base plus `0x14` before reading a byte at scaled base plus `0x05`.
- A zero word is stored at small base plus `0x14`; the captured byte is then zero-extended into a word store at scaled base plus `0x34`.
- The scaled-base word at `0x14` is read back before another zero word is stored at small base plus `0x14`.
- Two byte stores to small base plus `0x15` follow, both using the low byte of the captured read-back word.

A byte at offset `0x15` overlaps a word spanning offsets `0x14` through `0x17`. Aliasing can therefore invalidate conclusions based only on the apparent final assignment to each field.

### Fixed read at `0x05` and subsequent propagation

After those two small-base byte stores, the function reads the directly addressed global at `0x05` and converts its value to an unsigned 32-bit quantity. That converted value becomes a base for subsequent accesses.

This read occurs **before** the fixed byte store at `0x01`. Its captured value is not replaced by a later read merely because subsequent stores might overlap `0x05`.

The converted base participates in:

- A byte store at offset `0x15`, receiving the low byte of the undoubled extraction.
- A word read at offset `0x54`.
- A subsequent word write at that same offset, receiving the earlier scaled-base read-back value.

Between the byte store and the word read, other stores update scaled-base and read-back-derived destinations. Those stores may alias the upcoming read location. The word read at converted base plus `0x54` therefore observes memory at its actual read point, not necessarily the state existing when `0x05` was captured.

The original storage width and signedness of the global at `0x05` are not declared in the supplied material. Its address, conversion to an unsigned 32-bit value, position in the sequence, and use as a base are established; its source access width remains **documented_only**.

### Link-like mutation

Subsequent operations use recurring offsets without establishing a structure type:

| Access width | Representative offsets |
|---|---|
| Byte | `0x01`, `0x05`, `0x08`, `0x09`, `0x0D`, `0x10`, `0x11`, `0x15`, `0x19` |
| Halfword | Computed destinations receiving `0x003A` and `0x0065` |
| Word | `0x04`, `0x10`, `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, `0x74` |

The phase includes self-valued stores, scaled address-like payloads, repeated stores, and reads that determine later destinations. Some byte reads become bases for word reads at offset `0x74`; low-memory access is therefore part of the representation, not grounds for silently substituting conventional pointers.

A byte store retains only the low eight bits. A zero-extended byte stored as a word has a numeric value from `0` through `255`.

Writes at offsets `0x14`, `0x44`, and `0x54` are overwritten, sometimes after intervening reads. These intermediate effects cannot generally be removed.

### Constant writes and captured global base

A later region writes `0x3A` as a byte, halfword, and word to differently derived destinations.

Another computed word destination receives, in order:

- `0x20`.
- `0x2F`.
- A zero-extended byte read from the corresponding base plus `0x04`.
- `0x20` again.

The byte read occurs after the first two stores and before the last. Aliasing may make those stores affect its payload.

Immediately after the final computed `0x20` store, the function captures the 32-bit global at **`0x087FBBBC`**. This load precedes the fixed word store of `0x20` at **`0x60`**.

That captured global value is a meaningful input to the repeated-write region. It is not an unspecified placeholder for whichever value the global might contain later.

### Fixed repeated-write region

For describing this region and its consumers:

- The **captured global base** is the word loaded from `0x087FBBBC`.
- The **working base** is four times an adjusted address-like value, where the adjustment subtracts `0x96` from a previously formed fourfold value.
- The **repetition payload** is a late-loaded address-like value multiplied by four.
- The **late-node base** is that same late-loaded value multiplied by two.

These are descriptive roles, not asserted object types.

Each repeated group orders four stores: a word at a scaled destination plus `0x54`, a byte at captured global base plus `0x19`, a byte at working base plus `0x11`, and a word at captured global base plus `0x34`.

The byte payloads are the low bytes of the working base and captured global base respectively. The first group differs from later groups in the scaled destination and final word payload. Later groups reuse fixed derived values rather than demonstrably traversing objects.

The global at `0x087FBBBC` is not visibly reloaded within these groups. Aliasing writes to that global do not change the already captured base.

Exact repetition multiplicity is **documented_only** in this contract. The representation establishes fixed repetition, not equivalence to one final assignment.

### Late captures and mutation

The boundary after the repeated region has this established order:

1. Capture the word at fixed address `0xA9`.
2. Read the word at repetition payload plus `0x74`.
3. Write the fixed byte at `0x3A`.
4. Capture the word at late-node base plus `0x14`.
5. Capture the word at working base plus `0x04`.

The fixed `0x3A` write is therefore between the second read and the last two captures. Its byte payload corresponds to eight times the adjusted value plus `0x30`, narrowed to eight bits. If it aliases either later capture, those captures can reflect the write. The captured late-node word becomes final argument three.

Later stores include `0x6C` at working base plus `0x08`, low-byte cross-propagation at offsets `0x0D` and `0x09`, and a self-valued word at late-node base plus `0x54`.

The byte at fixed address `0x67` is then written from the low byte of the **captured word at `0xA9`**. Subsequent propagation is not uniformly from that captured word:

- After reading late-node base plus `0x44`, the function reads `0x67` afresh for the fixed byte store at `0x21`.
- After intervening computed stores, it reads `0x67` afresh for a byte store at working base plus `0x09`.
- After another computed byte store, a word store, and a read at working base plus `0x54`, it reads `0x67` afresh again for another store at working base plus `0x09`.

Intervening computed stores may alias `0x67`. These three fresh reads must remain distinct from the earlier captured `0xA9` value and from each other.

Additional effects include a word write overlapping the byte destination at working base plus `0x10`, replacement of previously stored words at offsets `0x44` and `0x54`, and byte propagation from the captured repetition-payload read.

One byte payload uses a right shift of a signed-looking word by a count taken from the working base’s low eight bits. Out-of-range count behavior and sign filling are **documented_only**.

### Final dispatch preparation

A word loaded from late-node base plus `0x04` becomes the final argument-source base.

A halfword `0x0065` is stored at twice that loaded value **before** reading its word at offset `0x64` for final argument two. Intervening updates then precede the read at offset `0x74` for final argument four.

Later operations read late-node base plus `0x64`, perform additional byte and word stores, and finally write a word at twice the captured value used as final argument two. Final dispatch follows without visible validation or cleanup.

These later writes do not require refreshing the captured dispatch arguments, even if their source memory is overwritten.

## Side effects

The function directly accesses computed memory and fixed locations.

Established fixed-location inputs include the global at `0x05`, the word at `0xA9`, and the word at `0x087FBBBC`. Fixed writes include bytes at `0x01`, `0x15`, `0x21`, `0x3A`, and `0x67`, and words at `0x54` and `0x60`.

These addresses are not automatically null-pointer errors, relocated symbols, or valid device registers. Their environmental meaning is **documented_only**.

Byte order and alignment requirements are unresolved. Overlapping byte, halfword, and word operations cannot be converted into unique final numeric word values without additional environmental information.

No atomicity, synchronization, barrier behavior, concurrency safety, or rollback is established. A fault may leave earlier stores intact.

Unknown dependencies may additionally affect globals, devices, allocations, exceptions, or nonlocal control flow. Their effects are not bounded by the enclosing function’s direct accesses.

## Dependencies

F-b51a4dc0b50a is the only supplied opaque function ID. No dependency has a resolved opaque ID.

| Dependency category | Established relationship | Unresolved contract |
|---|---|---|
| Initial indirect target | Derived from incoming control-link value | Identity, effects, calling convention |
| Subsequent indirect targets | Ordered context-derived invocations | Identities, implicit inputs, memory effects |
| Secondary-result producers | Register outputs consumed later | Exact provenance and validity |
| Final indirect target | Receives four specified argument positions | Identity, effects, return behavior |

No dependency identity is invented. No dependency is assumed to preserve globals or memory lifetimes.

## Acceptance examples

These are conditional static examples, not executed tests.

### Input extraction

For incoming unsigned register values `0xFFFFFFFF` and `0x00008000`, the corresponding right-shifted arguments are `0x1FFFF` and `1`. The explicit inputs remain unchanged in positions one and two.

### Seven-bit small-base derivation

If the relevant returned word has low seven bits `0x7F`, the undoubled extraction is `0x1FC` and the small base is `0x3F8`. Changing only original bits above bit six does not change this extraction.

### Fixed-input capture

Suppose the `0x05` global supplies converted base `0x200`. The subsequent accesses use `0x215` and `0x254`. The read of `0x05` precedes the fixed write at `0x01`; later aliasing does not retroactively change the captured base.

### Global-base capture timing

Suppose the final computed `0x20` store aliases `0x087FBBBC`. The global-base load observes memory after that store. If the subsequent `0x60` write also aliases it, the captured base is nevertheless already fixed.

### Intervening fixed write

Suppose the late-node word at offset `0x14` overlaps `0x3A`. Its capture occurs after the fixed byte write. Moving that capture before the write can change final argument three and does not preserve the established ordering.

### Fresh reads from `0x67`

Suppose the captured `0xA9` word has low byte `0xAB`. The initial store to `0x67` writes `0xAB`. If an intervening computed store changes `0x67` to `0xCD`, a subsequent propagation reads `0xCD`, not the originally captured `0xAB`.

### Narrowing and repetition

A byte store supplied `0x123456AB` writes only `0xAB`. For write-observable storage, repeated groups remain repeated transactions; a single final assignment is not established as equivalent.

### Partial execution and dispatch

A dependency that does not return prevents requirements on later effects from taking effect. A late fault does not imply rollback.

If final dispatch is reached, its first argument is `0x72`; the other positions use their specified captured values, not substitute rereads.

## Uncertainties

The following remain **documented_only**:

- Application purpose and semantic input constraints.
- Accuracy of the represented executable control flow.
- Indirect target identities, calling conventions, and effects.
- Complete provenance of secondary register outputs.
- Source storage width and signedness of the global at `0x05`.
- Address validity, ownership, lifetime, and alias relationships.
- Byte order, alignment, fault behavior, and fixed-location meaning.
- Signed-overflow and exceptional-shift semantics.
- Exact repeated-store multiplicity.
- Atomicity, concurrency, and device visibility.
- Whether final dispatch returns.

Structure maintenance and character-oriented processing are inferred possibilities only.

## Evidence

This contract uses only the supplied static representation of F-b51a4dc0b50a. Prior prose and reviewer feedback are not independent evidence of behavior.

Static support includes the explicit argument forwarding, unsigned extractions, ordered indirect calls, mixed-width accesses, fixed-location reads, captured global base, repeated updates, distinct rereads of `0x67`, and captured final-dispatch arguments.

No dependency bodies, runtime traces, validated object layouts, or executable observations were supplied. No runtime tests have been performed. Acceptance examples state conditional consequences of the static evidence, not observed execution results.
