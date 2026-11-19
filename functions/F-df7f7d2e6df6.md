<!-- Trusted provenance: F-df7f7d2e6df6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df7f7d2e6df6

## Purpose

F-df7f7d2e6df6 performs a fixed, ordered sequence of indirect invocations, transforms dependency-produced values, modifies memory through computed and absolute addresses, and reaches an unresolved indirect transfer with four arguments.

**Established static evidence:** the supplied representation declares no explicit parameters and no application-level return value. Visible effects include indirect calls, byte and word reads, byte/halfword/word writes, overlapping accesses, repeated stores, and accesses to low absolute addresses.

**Inferred intent:** shifts and masks suggest packed-field manipulation; some computed stores resemble connections between records. Neither observation establishes an application purpose, object model, or record format.

This contract describes observable interactions and memory-state transitions. It does not classify the function as an allocator, parser, initializer, or collection operation.

## Interface

### Inputs and outputs

There are no explicit parameters. Implicit inputs include:

- Entry machine context used to derive the first indirect destination.
- Primary dependency results, including results represented as combined 64-bit quantities.
- Secondary machine-state words consumed after some invocations.
- Existing memory at computed addresses and absolute storage locations.
- A global word captured as the base for a later repeated-write region.

The visible interface provides no status result, error code, ownership token, or designated output buffer. A return appears after the terminal indirect transfer; reaching that return requires compatible return behavior from the terminal destination.

The authentic calling convention and the attribution of secondary machine-state words are **documented_only**. They are not established as independent return values or preserved registers.

### Widths and arithmetic

| Quantity | Width | Contract significance |
|---|---:|---|
| Byte access | 8 bits | Stores retain the low byte; unsigned loads produce 0–255. |
| Halfword store | 16 bits | Modifies two bytes. |
| Ordinary word | 32 bits | Used for most values, arguments, arithmetic, and addresses. |
| Combined dependency result | 64 bits | Its low and high 32-bit numerical portions are consumed separately. |

All offsets are byte offsets. Splitting a combined result does not establish an eight-byte memory object or memory byte order.

Arithmetic conclusions stated under **32-bit bit-vector semantics** assume truncation modulo 2³² for word calculations and modulo 256 for byte results. These are conditional conclusions, not claims that every signed expression in the representation has defined source-language behavior.

Signed overflow, signed right shift, excessive shift counts, and the actual interpretation of wrapped addresses remain **documented_only**.

### Preconditions for reaching return

Every reached indirect destination must support the actual calling convention and return wherever subsequent execution requires it. Every reached memory access must be valid for its width and access mode. Computed addresses must have meaning in the target environment.

No visible validation rejects null, low, unaligned, overlapping, or wrapped addresses. No explicit checks establish object bounds or storage lifetimes.

## Behavior

### Indirect invocation phase

The visible body contains no explicit conditional branch or loop. Repeated operations are statically present; no input-controlled repetition count is established.

The first indirect destination is derived from entry context plus 4094. Later destinations predominantly use embedded location-like values with the same displacement. These expressions establish invocation boundaries, not verified dependency identities or target equivalence.

Invocations consume zero to four visible arguments. Some obtain ordinary results, some obtain combined results, and others have no subsequently consumed primary result. An unused result does not make an invocation unobservable.

The argument transformations include:

- Ordinary results and combined results’ low words shifted left by eight.
- Unmodified or shifted high words.
- Results retained across intervening invocations.
- Secondary machine-state words.
- Explicit zeros, including repeated zero arguments.
- Duplicate transformed values in different argument positions.

Argument position, duplication, and invocation order are significant. Similar argument shapes do not establish interchangeable dependencies.

Visible masks select fields of 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, and 24 bits. Logical shifts of combined results by 34, 36, and 44 positions select portions of their high words. Masking to 24 bits before a right shift excludes the original upper eight bits.

Under 32-bit bit-vector semantics, successive left shifts discard bits beyond bit 31. For example, a word shifted left by 25 and subsequently by 11 becomes zero. This does not settle unresolved signed-expression semantics.

The complete invocation-by-invocation dependency contract is **documented_only**; the supplied dependency map identifies no external destination.

### Entry into direct memory modification

The direct-access boundary retains several dependency-derived quantities:

- A word shifted left by 25, called the **retained shifted word**.
- A separate ordinary result multiplied by 2²⁵, called the **large computed base**.
- Another ordinary result shifted left by eight, masked to 0x7FFF, then logically shifted right by six, called the **narrow extracted value**.
- An ordinary result already shifted left by eight, retained for subsequent stores.
- The final combined result preceding direct accesses.

These names describe numerical roles, not object identities.

Under 32-bit bit-vector semantics, the narrow extracted value is four times its extraction input’s low seven bits. It ranges from 0 through 508 in multiples of four. Twice that value, the **doubled narrow base**, ranges from 0 through 1016 in multiples of eight.

The initial writes establish these ordered effects:

1. A word at the final combined result’s high word plus 0x14 receives the retained shifted word.
2. A byte at the retained shifted word plus 0x09 receives the low byte of that high word.
3. A word at 256 times the combined result’s low word plus 0x54 receives the large computed base.
4. The preceding byte destination receives the low byte of the retained shifted word.

The function then clears the large computed base’s offset-0x14 word. Before reading that word, it reads an unsigned byte at the large computed base plus 0x05, clears a word at the doubled narrow base plus 0x14, and stores the byte’s zero-extended value as a word at the large computed base plus 0x34.

**Conditional established consequence:** with ordinary coherent byte-addressed memory, 32-bit bit-vector address arithmetic, no external mutation, and no read side effects, the subsequent word read at the large computed base plus 0x14 yields zero.

The intervening accesses do not invalidate that conclusion:

- The byte load does not modify ordinary memory.
- The other word store writes zero, so any overlap can only replace bytes with zero.
- The offset-0x34 word is 0x20 bytes beyond the offset-0x14 word. Their four-byte ranges do not overlap, including under modulo-2³² addressing.

No dependency invocation intervenes in this interval. Ordinary overlap alone is therefore not a reason to permit a nonzero observation.

Device-memory semantics, read-triggered effects, asynchronous mutation, or mappings that alias otherwise distinct numerical addresses can invalidate this ordinary-memory conclusion. Those mechanisms remain **documented_only**.

After capturing the observation, the function clears the doubled narrow base’s offset-0x14 word again and performs two consecutive byte stores at that base plus 0x15 using the observation’s low byte. Under the ordinary-memory conditions above, both byte stores write zero.

### Low absolute storage and chained accesses

The byte at absolute 0x05 is read before the byte write at absolute 0x01. Its unsigned value is retained as a base. Later accesses use that captured base plus 0x15 and plus 0x54; the write at 0x01 does not retroactively alter it.

The next operations include two byte writes at the large computed base plus 0x11, first using a retained shifted result and then zero. Between them, a word is stored at the earlier offset-0x14 observation plus 0x54.

Under the ordinary-memory zero conclusion, this word destination is absolute 0x54. A subsequent word store at the captured byte-derived base plus 0x54 also uses that zero observation. These consequences apply at their respective stores; later writes need not leave either location unchanged.

The remaining chained-access region includes:

- Word observations at offsets 0x04, 0x44, 0x54, 0x64, and 0x74.
- Loaded words subsequently used as address bases.
- An unsigned byte whose value supplies the base for a word read at offset 0x74.
- Scaling by two, four, and eight without visible bounds checks.
- Repeated byte writes at offsets 0x09 and 0x15.
- Repeated self-value word stores at computed offset 0x24.
- A word write at absolute 0x54 and a byte write at absolute 0x15.

One later expression contains two reads from the same apparent offset-0x74 location: one supplies a destination base and the other the stored value. They remain separate observations. The representation does not establish their operand-evaluation order.

Offsets are not proof of a structure layout. A byte at offset 0x15 overlaps a word beginning at offset 0x14. Computed bases may coincide with each other or absolute storage. Outside the specifically proven zero interval, intervening nonzero stores can affect later reads.

### Literal stores and repeated-region setup

A loaded word supplies scaled destinations for three stores of numeric value 0x3A:

- One byte at four times the loaded word.
- One halfword at eight times the loaded word.
- One word at the four-times-scaled address plus 0x20.

An earlier retained base is reduced by decimal 150. Call the result the **adjusted value**. Four times it is the **scaled adjusted base**; eight times it supplies another repeated-region base.

At the scaled adjusted base plus 0x20, four ordered word stores write:

- 0x20.
- 0x2F.
- An unsigned byte freshly read at the scaled adjusted base plus 0x04.
- 0x20 again.

The intermediate stores and byte observation are observable boundaries, even when ordinary memory finishes with 0x20.

A global base word is captured before the absolute word store of 0x20 at 0x60. The captured value is retained throughout the repeated region, not reloaded for each group. If the absolute store aliases its source storage, it does not change the already captured value.

A computed offset-0x54 word store follows the absolute 0x60 write before the repeated groups.

### Repeated writes and final prefix

Each complete group contains four stores in this order:

| Position | Destination | Value |
|---|---|---|
| First | Captured global base + 0x19 | Low byte of the scaled adjusted base |
| Second | Scaled adjusted base + 0x11 | Low byte of the captured global base |
| Third | Captured global base + 0x34 | Retained scaled word |
| Fourth | Eight times the adjusted value + 0x54 | Four times the loaded word used for the literal stores |

The first complete group’s third store uses eight times an earlier retained word. Subsequent complete groups use eight times the adjusted value. This difference must not be normalized away.

After the last complete group, a final three-store prefix performs only the first, second, and third stores. The absolute word read at 0xA9 follows without an associated fourth store.

The exact complete-group count is not enumerated here and remains **documented_only**. The final three-store shape is established; no variable repetition count is inferred.

### Post-repetition observations

After that prefix, the function captures:

- A word at absolute 0xA9.
- A word at offset 0x74 from the retained four-times-scaled loaded base.

It then writes a byte at absolute 0x3A. Its value is the low byte of eight times the adjusted value plus 0x30.

Only afterward does it capture:

- A word at offset 0x14 from a separately retained doubled loaded base.
- A word at offset 0x04 from the scaled adjusted base.

The absolute byte store can affect either later word read if their byte ranges include 0x3A. It cannot alter the observations already captured from 0xA9 and the earlier offset-0x74 read.

### Final updates and terminal transfer

The final region writes the retained scaled-base byte at the doubled loaded base plus 0x0D, writes 0x6C at the scaled adjusted base plus 0x08, and writes the captured global base’s low byte at offsets 0x0D and 0x09 of the scaled adjusted base.

A word at the doubled loaded base plus 0x54 receives that base itself. Absolute 0x67 then receives the low byte of the captured 0xA9 word.

A word read at the doubled loaded base plus 0x44 precedes a fresh byte read of absolute 0x67 for the write to absolute 0x21. Two additional, separate reads of 0x67 later supply byte stores at the scaled adjusted base plus 0x09. Computed stores intervene and may alias 0x67; the three observed bytes need not equal the originally written byte.

Other final effects include:

- A byte at the doubled loaded base plus 0x01 derived from a signed right shift of the offset-0x44 observation. The count is the scaled adjusted base’s low byte.
- A word read at the scaled adjusted base plus 0x40, later stored at offset 0x10, overlapping an earlier byte store there.
- Storage of the captured offset-0x04 word at the global-derived base plus 0x74.
- Replacement of the doubled loaded base’s offset-0x54 word with its earlier offset-0x14 observation.
- Further computed reads at offsets 0x54 and 0x14, separated by writes.
- Low-byte stores using the already captured post-repetition offset-0x74 value.
- A halfword store of 0x65 at twice a newly loaded word.
- A word store of decimal 100 at a computed offset 0x54.
- A word store at twice another loaded word.

The terminal indirect transfer receives, in order:

1. Literal 0x72.
2. A word loaded late from offset 0x64.
3. The word retained from the doubled loaded base’s earlier offset-0x14 observation.
4. A word read near the end from offset 0x74.

A visible return follows. Neither destination identity nor application-level success is established.

## Side effects

The observable surface includes every reached indirect invocation, memory observation, intermediate overwritten store, repeated store, absolute-storage modification, and terminal transfer.

Required ordering distinctions include:

- Absolute 0x05 read before absolute 0x01 write.
- Large-base offset-0x14 clear before its observation, with only the specified byte read, zero store, and disjoint offset-0x34 store intervening.
- Global-base capture before absolute 0x60 write.
- Final three-store prefix before absolute 0xA9 read.
- Absolute 0xA9 and computed offset-0x74 observations before absolute 0x3A write.
- Absolute 0x3A write before the following offset-0x14 and offset-0x04 observations.
- Absolute 0x67 write followed by three distinct later reads.

There is no visible rollback, transaction, retry, locking, exception translation, or explicit error-reporting path. A fault or nonreturning dependency can leave earlier effects intact.

No allocation, release, ownership transfer, or lifetime extension is established. Address-like values are not automatically owned pointers. Unknown dependencies cannot be assumed to preserve globals, preserve address validity, or refrain from allocating or releasing memory.

Byte order, access atomicity, alignment requirements, concurrency, reentrancy, and device-memory behavior remain **documented_only**.

## Dependencies

F-df7f7d2e6df6 is the only authoritative opaque function ID supplied. No opaque IDs are available for the indirect destinations; none are invented here.

| Boundary | Established interaction | Unresolved behavior |
|---|---|---|
| Entry-derived invocation | Destination derived from entry context plus 4094 | Identity, ABI, effects |
| Subsequent indirect invocations | Ordered calls with zero to four visible arguments | Target equivalence, memory effects, errors, ownership |
| Secondary-state boundary | Machine-state words feed arguments and transformations | Attribution, validity, preservation |
| Terminal transfer | Four arguments beginning with 0x72 | Destination, return behavior, completion meaning |

Calls without consumed results remain part of the interaction contract. Explicit zero arguments do not authorize call elimination.

## Acceptance examples

These are conditional static examples, not executed tests.

### Width-sensitive extraction

For an extraction input of 0xFFFFFFFF, selecting its low 24 bits and logically shifting right by 20 produces 15.

For a combined result with low word 1 and high word 2, the low-word-left-shifted-by-eight and unmodified-high-word argument positions receive 256 and 2.

### Narrow address generation

Under 32-bit bit-vector semantics, an extraction input of 1 produces narrow value 4 and doubled narrow base 8. Accesses from this low base must not be redirected to a presumed heap object.

### Zero observation despite ordinary overlap

Assume ordinary coherent memory, modulo-2³² address arithmetic, successful accesses, and no external or read-triggered mutation. Even if the doubled narrow base’s zero word store overlaps the cleared large-base offset-0x14 word, the subsequent observation is zero.

A nonzero byte copied into the large-base offset-0x34 word does not change this result: that destination is disjoint. Both following offset-0x15 byte stores therefore write zero. The subsequent word store through the zero observation targets absolute 0x54.

### Exceptional memory model

If the byte load at the large computed base plus 0x05 triggers a device action changing the cleared word, the ordinary-memory zero guarantee does not apply. Whether such a mapping exists is **documented_only**, not established behavior.

### Captured global base

If the absolute 0x60 store changes the global base’s source storage, repeated groups still use the value captured before that store.

### Incomplete final group

Acceptance at the repetition boundary requires the offset-0x19 byte, offset-0x11 byte, and offset-0x34 word stores followed by the 0xA9 read. Adding an associated offset-0x54 store introduces an absent state transition.

### Absolute 0x3A update

For adjusted value 3, the byte written at 0x3A is 0x48. A following overlapping word read observes memory after that write, subject to target byte order. Earlier captured observations remain unchanged.

### Fresh reads of 0x67

If an intervening computed store changes 0x67, a later read-supplied byte store uses the changed value. Reusing the originally written byte is not equivalent.

### Failure and completion

The ordered 0x20, 0x2F, byte-derived word, 0x20 stores remain observable where the environment observes individual stores. A later fault establishes no compensation.

Reaching the terminal transfer requires its four specified arguments. It does not establish that the destination returns or that an application operation succeeds.

## Uncertainties

The following remain **documented_only**:

- Application purpose and authentic calling convention.
- External dependency identities, contracts, and secondary-state attribution.
- Full invocation-by-invocation dependency specification.
- Signed overflow, signed shifts, excessive shift counts, and actual address wrapping.
- Byte order, alignment, mapping, and access atomicity.
- Object layouts, storage bounds, ownership, and lifetimes.
- Exact complete-group count, not enumerated here.
- Device-memory effects, read side effects, asynchronous mutation, and concurrency.
- Final destination and completion behavior.

The large-base offset-0x14 observation is zero under the stated ordinary-memory conditions. Ordinary overlapping accesses do not leave that specific result unresolved. This guarantee concerns the captured observation, not permanent zero contents at the location.

## Evidence

This contract derives from static inspection of the supplied representation and dependency map. Prior prose, feedback, and embedded comments are not independent authority.

Established evidence covers visible widths, literals, transformations, retained observations, memory-access ordering, repeated-region boundaries, and terminal argument positions. The zero-observation conclusion follows from the intervening accesses’ widths, values, and disjoint numerical ranges under the explicitly stated memory model.

Arithmetic and memory-model deductions are conditional. Inferred intent is separate from established behavior; unavailable dependency and platform contracts remain unresolved.

No runtime tests, execution traces, external dependency verification, or implementation access beyond the supplied material have been performed. No snapshot hash or independently verified provenance is asserted.
