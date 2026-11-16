<!-- Trusted provenance: F-be71af3feebd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be71af3feebd

## Purpose

F-be71af3feebd performs an ordered sequence of unresolved indirect calls, transforms values obtained from those calls and ambient execution state, performs mixed-width memory updates, and reaches a final unresolved indirect transfer.

The supplied static representation does not establish a business-level purpose. Character-sized constants do not establish text formatting, and recurring offsets do not establish a particular object or container model.

This contract distinguishes:

- **Established static evidence:** represented accesses, widths, value dependencies and ordering.
- **Inferred intent:** plausible interpretations that are not behavioral requirements.
- **documented_only:** unresolved semantics that cannot safely become executable acceptance requirements.

Every described effect is conditional on execution reaching it. Valid addresses, returning dependencies and successful completion are not established.

## Interface

### Entry and implicit inputs

The displayed interface has no explicit parameters and no declared result. Its implicit inputs include:

- Incoming control-address state used to derive the first indirect-call target.
- An ambient 32-bit register value used in early shifted arguments.
- Primary and secondary result-register observations following unresolved calls.
- Memory at fixed low addresses and addresses derived from call results.
- A separately stored 32-bit word later captured as a publication base.
- The environment’s calling convention, address mapping, endianness and alignment behavior.

The absence of explicit parameters does not make this operation independent of caller or global state.

### Outputs

No explicit status or result value is established. Observable outputs include indirect calls, their represented arguments, byte/halfword/word stores, dependency effects and the final indirect transfer.

That final transfer has four displayed argument words: 0x72, a recently loaded word, an earlier captured word and a word loaded from offset +0x74.

No output-buffer, allocation-result, length-result or ownership-transfer interface is established.

### Widths and access layout

| Quantity or access | Represented width |
|---|---:|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access and ordinary integer value | 32 bits |
| Combined dependency result | 64 bits |
| Low or high portion of a combined result | 32 bits |

All offsets are byte offsets. Byte stores retain the low eight source bits; halfword stores retain the low sixteen. Byte reads used as word values are zero-extended where represented.

Recurring word offsets include +0x14, +0x24, +0x34, +0x44, +0x54, +0x64 and +0x74. Other significant word accesses occur at +4, +0x10 and +0x40. Byte accesses include +1, +5, +8, +9, +0x0d, +0x10, +0x11, +0x15 and +0x19.

These are access relationships, not structure declarations. A byte at +0x15 overlaps a word at +0x14; a byte at +0x11 overlaps a word at +0x10. No allocation size, padding or non-aliasing guarantee follows.

### Preconditions

No entry validation is represented. Normal progress requires every reached indirect target to be executable and every reached memory access to be valid for its width and mode.

This includes fixed low addresses, potentially unaligned words and addresses derived from bytes or heavily shifted words. Ordinary process-memory assumptions do not establish those conditions.

## Behavior

### Indirect-call prefix

The represented body begins with indirect calls, before its explicit data-memory stores. The first target uses incoming control state plus displacement 0x0ffe. Later targets use successive embedded control anchors plus the same displacement.

The prefix is straight-line in the supplied representation. There is no represented result-validation branch, conditional bypass of the update phase or error-return selection.

Its significant properties are:

- Calls display between zero and four explicit argument words.
- Many arguments are earlier results shifted left by eight.
- Other shifts include small counts and counts through 31.
- A transformed word sometimes occupies several argument positions.
- Some dependency results are represented as 64-bit quantities, with separate low/high-word uses and additional right shifts of the combined value.
- Explicit zeros recur, including shifts of values already established as zero.
- Masks retain low portions of words before subsequent scaling or extraction.

Calls with unused primary results remain part of the behavior. Repeated-looking calls cannot be merged or omitted on the assumption that dependencies are pure.

The displayed argument dependencies are static evidence, not verified prototypes. Complete per-call ABI details, target identities and dependency semantics remain **documented_only**.

### Arithmetic boundaries

Unsigned masks and logical right shifts have their represented bit-width meaning. A 24-bit mask removes the upper eight bits of a word; it does not validate an address or establish backing storage.

Other expressions use signed-looking 32-bit shifts, multiplication and address arithmetic. Portable overflow behavior is not established.

Under an explicitly assumed truncating 32-bit bit-vector interpretation:

- A left shift by eight followed by logical right shift by eight retains the original low 24 bits.
- Each intermediate left shift discards bits independently.
- Two successive left shifts by 28 yield zero, not a 56-bit scaled result.
- Multiplication by 0x02000000 retains only seven original bits.
- Narrow stores truncate after the represented calculation.

Address wrapping, signed overflow and signed right-shift behavior are not independently verified.

### Bases used by the direct-memory phase

The following descriptive terms identify value dependencies, not recovered object types:

- **Shifted base:** an earlier call result multiplied by 0x02000000.
- **Compact index:** a call result shifted left by eight, masked to 0x7fff, then logically shifted right by six.
- **Compact base:** twice the compact index.
- **Link base:** a retained result shifted left by 25.
- **Final result pair:** the low and high words of the final combined result before direct stores.

Under 32-bit bit-vector arithmetic, the compact index is a multiple of four from 0 through 508. The compact base is a multiple of eight from 0 through 1016. Neither range establishes memory validity.

### Opening updates and linked accesses

The opening update region writes the link base as a word at final-result-high +0x14, writes the low byte of final-result-high at link-base +9, and writes the shifted base at 256 times final-result-low +0x54. It then writes the low byte of the link base at link-base +9.

Those two byte stores are separate events, even when their values happen to match.

The shifted-base +0x14 word is cleared before a later read. Between that clear and read, the function reads shifted-base +5 as a byte, clears compact-base +0x14 and writes the zero-extended byte at shifted-base +0x34. A further compact-base +0x14 clear precedes two byte stores at compact-base +0x15 using the captured word read from shifted-base +0x14.

Aliasing can affect that captured word despite the earlier clear. It must not invariably be treated as zero.

The next region reads a byte at fixed address 0x05 and promotes it to a base. It also stores at fixed addresses 0x01, 0x15 and 0x54. Derived reads at +0x54 and +0x74 supply additional bases and values.

Subsequent effects include:

- Repeated self-referential word stores at +0x24.
- Overwritten and repeated word stores at +0x14 and +0x54.
- Byte stores at +9, +0x0d, +0x11 and +0x15.
- Scaling by two, four and eight.
- Word updates at +0x34 and +0x44.
- Promotion of a byte read at +5 into a base for a word read at +0x74.
- A secondary base loaded from +0x64.

Repeated reads of the same apparent byte source occur with intervening stores. They are not guaranteed to return the same value.

A nested linked update contains two word reads from the same +0x74 location: one supplies a destination base, the other the stored value. A single-read replacement is not established. Operand-level ordering within that expression remains **documented_only**.

### Constant stores and publication-base capture

A later word obtained through a byte-derived address determines three numeric stores of 0x3a:

- A byte at four times that word.
- A halfword at eight times that word.
- A word at four times that word +0x20.

These do not establish a string or character-stream operation.

For this and the remaining regions, the following terms identify retained arithmetic values:

- **Scale origin:** the earlier word used to form the recurring four-times and eight-times bases.
- **Adjusted origin:** four times the scale origin minus 0x96.
- **Working base:** four times the adjusted origin.
- **Constant base:** four times the word used for the three 0x3a stores.
- **Doubled base:** twice that same word.

The working-base +0x20 word receives, in order, 0x20, 0x2f, a zero-extended byte read from working-base +4, and 0x20 again.

**After those four stores, the function reads the separately stored publication-base word. Only after capturing that value does it store the word 0x20 at fixed address 0x60.** Subsequent publication accesses use the captured base, not an implicit reload after the 0x60 store.

This capture boundary matters even if the 0x60 store aliases the storage supplying the publication base or has environment-specific memory effects.

### Repeated publication

The first publication group has four ordered stores:

- The constant base as a word at eight times the scale origin +0x54.
- The low byte of the working base at captured publication-base +0x19.
- The low byte of the captured publication base at working-base +0x11.
- Eight times the scale origin as a word at captured publication-base +0x34.

Later repeated groups retain the two byte-store destinations and values, but use eight times the adjusted origin for both the +0x54 destination base and the value written at publication-base +0x34.

Thus the first group differs from later groups in both its scaled word destination and its +0x34 word value. The publication base and byte sources are retained values; the represented groups do not reload them from memory.

The region contains many explicit repetitions. Their effects are ordered and cannot be replaced by a single final-state assignment where access traces are observable. The exact multiplicity is not enumerated by this contract and remains **documented_only** here.

### Final update group: captured words and fresh global-byte reads

At the start of the final region, the function captures a word from fixed address 0xa9 and a word from constant-base +0x74. It then writes a byte at fixed address 0x3a: the low eight bits of eight times the adjusted origin plus 0x30.

It subsequently captures a word from doubled-base +0x14 and a word from working-base +4. These captured words have later uses even if their original storage changes.

Before the first fixed-address 0x67 reload, the function writes:

- The low byte of the working base at doubled-base +0x0d.
- Byte 0x6c at working-base +8.
- The low byte of the captured publication base at working-base +0x0d and +9.
- The doubled base as a word at doubled-base +0x54.
- The low byte of the captured 0xa9 word at fixed address 0x67.

**The byte assigned to 0x67 is not an invariably preserved source for all later byte stores.** Three later uses explicitly read 0x67 anew.

First, after assigning 0x67, the function reads a word at doubled-base +0x44. Only then does it read the byte at 0x67 for the byte store at fixed address 0x21. The intervening derived word read is part of the observable order; its memory effects, if any, are not assumed absent.

Further updates use the captured +0x44 word:

- Its low byte is stored at working-base +0x10.
- It is stored as a word at captured publication-base +0x54.
- A shifted byte derived from it is stored at doubled-base +1, using the low eight bits of the working base as the right-shift count.
- Publication-base +0x54 is subsequently overwritten with the working base.

A word read from working-base +0x40 supplies a word store at working-base +0x10, overlapping the earlier byte store. Between that read and store, the captured working-base +4 word is written at publication-base +0x74. A word store at publication-base +0x44 then records the doubled base.

The function next **reloads 0x67 for a byte store at working-base +9**. After that store, it writes the low byte of the working base at doubled-base +9, writes the full captured 0xa9 word at working-base +0x44, and reads a word at working-base +0x54.

It then **reloads 0x67 separately again for a second byte store at working-base +9**. These two reloads are separated by writes and a derived word read. Aliasing or externally changing memory can make their byte values differ from each other and from the earlier value stored at 0x21.

The full word captured from 0xa9 remains a distinct retained value. Changes to 0x67 do not redefine that captured word.

### Remaining updates and final transfer

The word read from working-base +0x54 becomes another destination base. The low byte of the earlier constant-base +0x74 word is stored at its +0x0d and later at doubled-base +0x11.

The earlier doubled-base +0x14 word is written at doubled-base +0x54. A fresh word from working-base +0x14 supplies a later byte store. Working-base +9 is subsequently overwritten again, now with the low byte of the destination base obtained from working-base +0x54; that base is also stored as a word at working-base +0x44.

A word read at doubled-base +4 determines a halfword destination at twice its value, where 0x65 is stored. Reads at that newly obtained base +0x64 and +0x74 supply a later stored word and the final transfer’s fourth argument.

Further effects include a byte store at the previous destination base +0x11, a word store of decimal 100 at its +0x54, a word read at doubled-base +0x64, and a byte store at doubled-base +9. The +0x64-derived destination receives the newly obtained +0x64 word at offset +0x54, and that destination value is stored at twice the newly obtained +0x64 word.

The final indirect transfer displays these argument words, in order:

1. 0x72.
2. The word read from the newly obtained base +0x64.
3. The earlier captured doubled-base +0x14 word.
4. The word read from the newly obtained base +0x74.

Neither the final target’s identity nor successful return is established.

## Side effects

The direct-memory phase contains 8-, 16- and 32-bit writes. It is not an atomic transaction or validated batch.

Repeated reads, repeated stores, overwritten stores and overlapping accesses are part of the represented trace. A final memory snapshot alone cannot characterize behavior for volatile, device-backed, concurrently observed or faulting memory.

No disjointness is established between fixed addresses, calculated addresses, source storage or destination storage. Captured values and explicit reloads must nevertheless remain distinct: aliasing does not authorize inventing a reload, and retained values do not authorize removing an explicit read.

No explicit allocation, release, retain or ownership transfer is identifiable in the direct-memory phase. Unresolved dependencies may perform any of those operations. Storing a pointer-like word establishes no ownership or lifetime rule.

No null check, bounds check, alignment check, rollback, explicit status handling or error-reporting path is represented. A fault or non-returning dependency can prevent later effects; earlier effects are not shown as undone. Thread safety, reentrancy and synchronization are not established.

## Dependencies

Only F-be71af3feebd has a supplied opaque identity. These additional opaque IDs are specification-local dependency designations, not verified target mappings:

| Opaque ID | Role | Boundary |
|---|---|---|
| F-U001 | Initial incoming-control-derived target | ABI, validity and effects unresolved |
| F-U002 | Subsequent anchor-relative target family | Distinct sites need not share a target |
| F-U003 | Final unresolved transfer target | Four displayed arguments; continuation unresolved |

F-U002 denotes a family, not a single proven function.

No dependency is assumed to preserve globals, unrelated memory or resource ownership, or to return normally. Embedded comments do not independently establish target identity or transfer semantics.

## Acceptance examples

These are conditional static acceptance examples. No runtime tests have been performed.

### Width and promotion

If a reached byte store has source word 0x123456ab, it stores 0xab, not the full word. If shifted-base +5 yields byte 0xfe, its corresponding word publication uses 254, not −2.

For valid, disjoint constant destinations, the three 0x3a stores have numeric values 0x3a, 0x003a and 0x0000003a at byte, halfword and word widths respectively. Their byte encodings depend on endianness.

### Compact-base boundary

Under 32-bit bit-vector arithmetic, a source word with all low seven bits set produces compact index 508 and compact base 1016. Higher source bits cannot enlarge that range. This does not establish that address 1016 is accessible.

### Publication-base capture before 0x60

Suppose the publication-base read yields address 0x2000. If the subsequent word store at 0x60 changes that source storage to represent 0x3000, the represented publication destinations still use captured base 0x2000, including +0x19 and +0x34.

Reloading the publication base after the 0x60 store would change the specified dependency.

### Independent 0x67 samples

Suppose the captured 0xa9 word has low byte 0xab. The function initially assigns 0xab to 0x67. If the explicit later reads of 0x67 yield 0x11, 0x22 and 0x33, respectively, the corresponding byte stores use:

- 0x11 at fixed address 0x21.
- 0x22 at the first later working-base +9 store.
- 0x33 at the second later working-base +9 store.

The first sample occurs after the doubled-base +0x44 word read. The last two samples are separated by the doubled-base +9 store, working-base +0x44 word store and working-base +0x54 word read.

This example permits external change or memory-side-effect behavior; it does not require ordinary RAM to change spontaneously. A later overwrite of working-base +9 does not remove these earlier events.

### Sequential overwrite and partial failure

If working-base +0x20 is ordinary RAM without interfering aliases, its final word after the four described stores is 0x20. Omitting the intermediate 0x2f store is not access-trace-equivalent.

If an access faults after earlier stores complete, no rollback or returned error code is required by the static evidence.

## Uncertainties

The following remain **documented_only**:

- Business purpose and intended data structures.
- Actual ABI, complete entry-state requirements and dependency identities.
- Whether combined results are true 64-bit returns or paired register observations.
- Signed overflow, signed right shifts and out-of-range variable-shift semantics.
- Endianness, address wrapping, alignment support and low-address validity.
- Exact repeated-publication multiplicity as an executable requirement here.
- Operand-level ordering within nested expressions.
- Dependency effects on globals, memory, lifetime and ownership.
- Final-target behavior and reachability of normal completion.

A linked-state or publication interpretation is inferred intent only. It does not override the represented capture boundaries, fresh reads or mixed-width effects.

## Evidence

This contract uses only the supplied static representation. Established observations include the implicit-state interface, indirect-call prefix, masks and shifts, result-word extraction, mixed-width accesses, fixed-address effects and final transfer.

Two ordering relationships are directly represented: publication-base capture precedes the 0x60 word store; and the initial 0x67 assignment is followed by three separate byte reloads, with the intervening accesses described above.

No dependency bodies, independently verified machine instructions or runtime observations were available. No runtime tests have been performed. Unsupported semantics remain **documented_only**, rather than becoming invented prototypes, ownership guarantees or successful-execution requirements.
