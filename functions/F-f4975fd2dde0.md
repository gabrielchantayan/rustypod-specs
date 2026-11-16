<!-- Trusted provenance: F-f4975fd2dde0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f4975fd2dde0

## Purpose

F-f4975fd2dde0 performs an ordered sequence of indirect dependency calls, transforms values exposed by those calls, modifies memory through calculated and fixed addresses, and ends with an unresolved four-argument indirect transfer.

The supplied static representation does not establish an application-level purpose. In particular, character-like constants do not establish text generation, and repeated address relationships do not establish a conventional object model.

This contract distinguishes:

- **Established static evidence:** operations, widths, and ordering represented in the supplied material.
- **[INFERENCE]:** conclusions requiring an explicitly stated machine-arithmetic assumption.
- **documented_only:** unresolved behavior that cannot support an unconditional executable acceptance requirement.

No runtime tests have been performed.

## Interface

### Inputs and completion

The represented interface has no explicit parameters and no declared result value. Implicit inputs nevertheless include:

- Entry control state used to determine the first indirect destination.
- Ordinary dependency results, combined results, and secondary register outputs.
- Fixed-address storage and memory reached through calculated addresses.
- The destination and behavior of the final indirect transfer.

There is no established status code, returned allocation, or returned object. Normal completion requires every reached dependency and memory operation to complete, followed by return from the final destination. The representation shows a return after that transfer, but whether the destination actually returns is **documented_only**.

### Widths and interpretation

| Represented quantity | Width |
|---|---:|
| Explicit byte access | 8 bits |
| Explicit halfword store | 16 bits |
| Ordinary integer, word access, or extracted result component | 32 bits |
| Selected combined dependency results | 64 bits |

An explicit byte store retains the low eight bits. Explicit unsigned-byte reads produce values from 0 through 255; conversion of such a value to a word is zero-extension.

A combined result is treated as lower and upper 32-bit components. This does not establish a memory layout or byte order. Selected extractions shift the 64-bit value by 34, 36, or 44 bits before narrowing.

Fixed-address symbol declarations are not supplied. Consequently, a conversion from a named global into a 32-bit value does not, by itself, prove that the underlying storage is a byte or word. Where this contract describes a fixed-address word assignment or read, it describes the represented 32-bit value use; the unavailable declaration and actual access realization remain unresolved.

### Preconditions

Normal execution requires executable indirect destinations, a compatible calling convention, and valid memory for every reached access. Calculated addresses may be small, zero-derived, unaligned, overlapping, or outside ordinary allocations.

The function visibly performs no null checks, bounds checks, allocation-size checks, destination validation, or error-status handling. Validity and lifetime of dependency-provided addresses are environmental preconditions, not guarantees supplied by this function.

## Behavior

### Ordered dependency prefix

The first indirect destination depends on entry control state plus displacement `0xFFE`. Subsequent destinations are largely derived from represented code positions using the same displacement.

The prefix includes calls with no represented arguments and calls with one through four represented arguments. It passes shifted results, masked fields, retained values, secondary register outputs, and explicit zeros. Some values survive numerous intervening calls.

Calls without explicit arguments are not established to be input-free or pure. Calls whose ordinary result is ignored remain part of the behavior. No dependency may be assumed to preserve globals, reachable memory, ownership, or secondary registers beyond what the represented data flow actually uses.

The visible caller sequence contains no success-dependent branch, retry loop, or early error return. Dependencies may nevertheless branch, fail, modify state, or not return.

### Arithmetic constraints

Transformations include left shifts, logical right shifts of unsigned values, masks, multiplication by powers of two, and later subtraction of `0x96`. Masking before a shift is not interchangeable with masking afterward. Repeated shifts of a retained 32-bit intermediate may discard information before subsequent operations.

[INFERENCE] Under a 32-bit wrapping machine model, shifts and multiplication retain the low 32 bits. Signed-looking expressions in the representation do not independently establish portable overflow behavior. Address wrapping, signed shifts, and mixed-width byte order remain **documented_only**.

### Address roles in the memory suffix

The following descriptive roles identify data relationships, not recovered fields or source identifiers:

- **Shifted base:** a retained dependency-result component shifted left by 25 bits.
- **Scaled base:** another dependency result multiplied by \(2^{25}\).
- **Compact index:** a result shifted left by eight bits, masked with `0x7FFF`, then logically shifted right by six.
- **Compact base:** twice the compact index.
- **Saved link:** a word subsequently read at offset `0x54` from the address obtained through fixed storage at `0x05`.
- **Adjusted scale:** four times the saved link, minus `0x96`.
- **Publication base:** four times the adjusted scale.
- **Global base:** the retained value read from fixed storage at `0x087FBBBC`.
- **Final doubled base:** twice a later word loaded through an unsigned-byte-selected address at offset `0x74`.
- **Published target:** four times that same later loaded word.

[INFERENCE] With 32-bit arithmetic, the compact index ranges from 0 through 508 in multiples of four; the compact base ranges from 0 through 1016 in multiples of eight. These are calculated addresses, not proven offsets within a larger allocation.

After the last prefix dependency call, direct memory operations continue without another represented dependency call until the final transfer.

### Initial writes and dependent loads

The suffix first stores the shifted base at offset `0x14` from the upper component of the last combined result. It writes the low byte of that upper component at shifted-base offset `0x09`, stores the scaled base at offset `0x54` from 256 times the lower component, and overwrites shifted-base offset `0x09` with the low byte of the shifted base.

It then clears scaled-base offset `0x14`, reads an unsigned byte at scaled-base offset `0x05`, clears compact-base offset `0x14`, and stores the sampled byte as a word at scaled-base offset `0x34`. The subsequent word read at scaled-base offset `0x14` observes memory at that point, including any overlap with intervening accesses. Another clear and duplicate byte stores follow at the compact base.

The relative-access inventory is:

| Relative offset | Represented uses |
|---|---|
| `0x01` | Byte write |
| `0x04` | Byte and word reads |
| `0x05` | Unsigned-byte reads |
| `0x08`, `0x09`, `0x0D` | Byte writes |
| `0x10` | Byte and word writes |
| `0x11`, `0x15`, `0x19` | Byte writes |
| `0x14` | Word reads and writes |
| `0x20`, `0x24`, `0x34` | Word writes |
| `0x40` | Word read |
| `0x44`, `0x54` | Word reads and writes |
| `0x64` | Word reads |
| `0x74` | **Word reads and writes** |

Different rows and different bases do not imply disjoint storage. In particular, the final region writes a saved word to **global-base offset `0x74`**.

The intervening calculated-memory region repeatedly clears, samples, and replaces fields. Significant relationships include:

- A word at offset `0x54` is saved before that location is replaced; the saved link later supplies addresses and scaled values.
- Words reached through offset `0x74` become bases for writes at `0x54`.
- One dependent assignment contains two reads of the same calculated `0x74` location. A single stable snapshot is not established; expression-internal evaluation order is unresolved.
- Several locations receive their own base values at offsets `0x14`, `0x24`, or `0x54`.
- Explicit unsigned-byte samples at offset `0x05` feed later word stores and address selection.
- Repeated writes and overlapping byte/word accesses remain ordered effects, not merely a final memory image.

### Fixed-address accesses before publication

The following ordering is established within the suffix:

| Fixed address | Access and represented value | Relevant ordering |
|---|---|---|
| `0x05` | Read, converted to a 32-bit unsigned value and subsequently used as an address base | After the initial compact-base duplicate byte stores; before the write at `0x01` |
| `0x01` | Byte store of the low byte of the compact base | Before a calculated byte store at offset `0x15` from the base obtained through `0x05` |
| `0x54` | Word assignment containing twice the saved link | After repeated self-link and byte writes through calculated bases; between compact-base `0x14` stores |
| `0x15` | Byte store of the low byte of the compact base | After two compact-base `0x15` stores containing the low byte of four times the saved link |
| `0x087FBBBC` | Read whose retained 32-bit value becomes the global base | After the four publication-base `0x20` stores described below; before the fixed assignment at `0x60` |
| `0x60` | Word assignment containing `0x20` | Before the repeated publication writes |

The read at fixed `0x05` must not be specified as an established unsigned-byte read: its declaration is unavailable. This differs from the explicitly unsigned-byte reads at calculated offset `0x05`.

Between the fixed `0x01` and `0x54` stores, the function reads and replaces offset `0x54` from the base obtained through `0x05`, follows calculated `0x74` links, and performs repeated calculated writes. These operations may overlap fixed storage.

The global base is sampled **before** the fixed `0x60` assignment. Subsequent publication uses that saved base, not a fresh global read, even if the fixed assignment or later calculated stores overlap its original storage.

### Constants and repeated publication

A later group stores `0x3A` as a byte at the published target, as a halfword at twice that target, and as a word at published-target offset `0x20`. These accesses do not establish a string or contiguous character sequence.

Publication-base offset `0x20` then receives four word stores in order:

1. `0x20`.
2. `0x2F`.
3. The zero-extended unsigned byte sampled at publication-base offset `0x04`.
4. `0x20`.

The byte is sampled after the first two stores. Acceptance must account for aliasing at the actual sampling point.

After the global-base read and fixed `0x60` assignment, repeated publication groups write:

- A word containing the published target through a scaled destination at offset `0x54`.
- A byte containing the low byte of the publication base at global-base offset `0x19`.
- A byte containing the low byte of the global base at publication-base offset `0x11`.
- A scaled link value as a word at global-base offset `0x34`.

The first group uses eight times the saved link for its scaled destination and link value. Later groups use eight times the adjusted scale. This distinction must survive any behavioral substitution.

The repeated stores are explicit static effects. Exact hardware transaction multiplicity is not established; this contract does not provide a counted trace oracle for this extended region.

### Final-region snapshots and fixed byte storage

After publication, the function reads a represented word from fixed `0xA9`, then a word at published-target offset `0x74`. It next writes fixed byte storage at `0x3A` with the low-byte result of **eight times the adjusted scale plus `0x30`**.

It subsequently saves words from final-doubled-base offset `0x14` and publication-base offset `0x04`. Before initializing fixed `0x67`, it performs calculated byte stores at offsets `0x0D`, `0x08`, and `0x09`, including `0x6C` at publication-base offset `0x08`, and writes the final doubled base to its own offset `0x54`.

The consequential fixed-byte sequence is:

- **Initialization:** fixed `0x67` receives the low byte of the word saved from fixed `0xA9`.
- **First fresh read:** after reading a word at final-doubled-base offset `0x44`, the function reads fixed byte `0x67` and stores that byte at fixed `0x21`.
- **Second fresh read:** later, fixed byte `0x67` is read for a byte store at publication-base offset `0x09`.
- **Third fresh read:** still later, fixed byte `0x67` is read again for another byte store at publication-base offset `0x09`.

These are separate reads, not reuse of the initialization byte.

Between the first and second reads, calculated stores affect publication-base offset `0x10`, global-base offset `0x54`, final-doubled-base offset `0x01`, global-base offset `0x74`, and global-base offset `0x44`. A word is also read at publication-base offset `0x40` and stored at publication-base offset `0x10`. The global-base `0x74` store contains the previously saved word from publication-base offset `0x04`.

Between the second and third reads, the function writes a byte at final-doubled-base offset `0x09`, stores the saved `0xA9` word at publication-base offset `0x44`, and reads a word at publication-base offset `0x54`.

Any of these calculated stores may alias fixed `0x67`. Therefore neither the fixed `0x21` value nor either publication-base `0x09` value is unconditionally equal to the low byte originally saved from `0xA9`.

### Remaining mutation and final transfer

The word read at final-doubled-base offset `0x44` also supplies a variable right shift. Its count is the low eight bits of the publication base, and the low byte of the shifted result is stored at final-doubled-base offset `0x01`. Counts may exceed 31; effective behavior is **documented_only**.

After the third `0x67` read, further writes use the saved published-target `0x74` word and restore the earlier final-doubled-base `0x14` snapshot into offset `0x54`. A fresh publication-base `0x14` word supplies a later byte store.

Near completion, a word loaded at final-doubled-base offset `0x04` becomes another base. The function stores halfword `0x65` at twice that base, reads its `0x64` word, performs further byte and word mutations including a word containing decimal 100 (`0x64`), and reads its `0x74` word. A subsequent read at final-doubled-base offset `0x64` supplies another destination. The last stores include a word at that destination’s offset `0x54` and a word at twice the previously loaded `0x64` value.

The final indirect transfer receives, in order:

1. `0x72`.
2. The saved word read through the later base’s offset `0x64`.
3. The earlier snapshot from final-doubled-base offset `0x14`.
4. The word read through the later base’s offset `0x74`.

These are saved values, not unrestricted rereads after the final mutations.

## Side effects

Direct effects comprise ordered byte, halfword, and word accesses through calculated addresses and the fixed-address interactions specified above. Dependencies may add unknown effects.

No identifiable explicit allocation or release establishes ownership. Passing or storing a value does not prove transfer, borrowing, retention, release, reference counting, or lifetime extension.

The function is not established to be pure, idempotent, reentrant, atomic, or thread-safe. There is no visible transaction boundary or rollback. Failure after completed writes leaves no represented restoration requirement.

Repeated stores must not be discarded solely because ordinary memory would retain the same final value. Conversely, volatile semantics, device transactions, synchronization, and cross-thread ordering are not established.

## Dependencies

The supplied identity map establishes only **F-f4975fd2dde0**. No additional opaque function IDs are available for the indirect destinations; no self-recursive relationship is established.

The following dependency contracts remain **documented_only**:

| Interaction | Established use | Unresolved contract |
|---|---|---|
| Entry-state-derived invocation | First call, using displacement `0xFFE` | Identity, valid entry state, ABI, effects |
| Position-derived invocations | Ordered calls with transformed or absent explicit arguments | Identities, argument interpretation, failures, memory effects |
| Combined-result interactions | Selected results consumed as 64-bit values | Actual ABI and component validity |
| Secondary-register interactions | Exposed values feed later transformations | Definition, validity, preservation |
| Final indirect transfer | Four arguments after direct mutations | Identity, ownership, interpretation, return behavior |

## Acceptance examples

These are conditional static examples, not executed tests.

- **32-bit truncation:** if a dependency result is `0x12345678`, an eight-bit left shift under wrapping 32-bit arithmetic produces `0x34567800`, not a widened product.
- **Compact-index boundary:** inputs `0x7F` and `0x80` to the compact-index transformation produce indices 508 and 0 respectively; their doubled bases are 1016 and 0.
- **Unsigned-byte propagation:** an explicit calculated-offset `0x05` sample of `0xE7` becomes word `0x000000E7`. This example does not determine the width of fixed storage at `0x05`.
- **Ordered overwrites:** absent competing or overlapping modification, publication-base offset `0x20` ends with word `0x20`; its preceding stores and intervening byte sample remain required.
- **Fixed byte initialization:** if the saved `0xA9` word is `0x123456E7`, initialization stores `0xE7` at fixed `0x67`.
- **Fresh fixed-byte reads:** if a calculated store changes `0x67` to `0x42` after its use for fixed `0x21` but before its next read, that next publication-base `0x09` store uses `0x42`. Another intervening change must likewise affect the third read.
- **Saved global base:** if the `0x60` assignment overlaps storage supplying the global base, publication still uses the value sampled before that assignment.
- **Global-base `0x74` write:** the final-region word store there uses the earlier publication-base `0x04` snapshot, even if its original location has since changed.
- **Partial failure:** an invalid late access provides no requirement to undo earlier writes or return a particular error.
- **Final transfer:** reaching the destination requires first argument `0x72` and the specified saved or late-loaded words, not replacement snapshots taken after all stores.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and safe public invocation conditions.
- Dependency identities, ABI, effects, ownership, and termination.
- Validity of combined and secondary-register outputs.
- Overflow, address wrapping, signed shifts, and oversized shift counts.
- Byte order and consequences of overlapping mixed-width accesses.
- Missing fixed-storage declarations and their exact access realization.
- Memory mappings, object boundaries, alignment, lifetimes, and aliasing.
- Exact externally observable repetition count, atomicity, and concurrency behavior.
- Whether the final indirect destination returns.

[INFERENCE] The unusual entry-state dependency and extensive position-derived calls may indicate an unconventional boundary or incomplete control-flow representation. That possibility does not justify replacing the observed effects with a cleaner invented algorithm.

## Evidence

This contract uses only the supplied static representation. Prior prose, comments, warnings, and reviewer observations are not independent behavioral proof.

The represented accesses establish both reads and writes at relative `0x74`, the fixed-address ordering specified above, and three distinct final-region reads of fixed byte `0x67` after initialization. The supplied material does not provide dependency bodies, global declarations, a verified memory map, runtime traces, or execution results.

No implementation access, tools, or runtime verification were used.
