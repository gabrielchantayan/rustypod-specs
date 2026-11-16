<!-- Trusted provenance: F-4a104faaa0f9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4a104faaa0f9

## Purpose

F-4a104faaa0f9 performs an ordered sequence of unresolved indirect calls, derives integer and address-like values from their results, mutates fixed-address and calculated-address storage, and reaches a final unresolved indirect transfer.

No application-level purpose is established. Character-like constants, recurring offsets, and self-referential stores do not establish formatting, allocation, initialization, serialization, or object management.

This contract distinguishes:

- **Established static evidence:** represented access widths, constants, saved-value relationships, and relative operation order.
- **[INFERENCE]:** possible interpretations that are not behavioral requirements.
- **documented_only:** unresolved behavior for which no implementation requirement is invented.

Descriptions are conditional on execution reaching the relevant operations. They are not runtime-validated guarantees.

## Interface

### Entry and implicit inputs

The represented interface has no explicit parameters and no scalar return value. It consumes implicit state comprising:

- Two incoming register-held values.
- Incoming link state used to calculate the first indirect target.
- Primary, secondary-register, and combined results from unresolved dependencies.
- Fixed-address memory and storage reached through calculated values.
- An unresolved final transfer target.

The first indirect target is incoming link state plus `0xFFE`. Later call targets use that displacement from represented continuation locations. Target validity, instruction-state encoding, and complete callable signatures are documented_only.

An empty displayed argument list does not establish that a dependency consumes no inputs.

### Outputs and completion

No output buffer, output length, status code, or scalar result is established. Observable effects comprise dependency effects, direct memory accesses, and the final transfer with four displayed arguments.

A return is represented after the final transfer. Whether the transfer returns normally, terminates execution, or continues elsewhere remains documented_only.

### Widths and storage

| Quantity or access | Represented width |
|---|---:|
| Byte | 8 bits |
| Halfword | 16 bits |
| Word and address-like integer | 32 bits |
| Combined intermediate result | 64 bits |

Combined results have low and high 32-bit components. This does not establish byte order or a conventional combined-result calling convention.

Unsigned byte loads widen to values from 0 through 255. Byte stores retain the source’s low eight bits.

Offsets identify access locations, not validated structure fields. Storage reached through different bases may overlap. Byte and word accesses at the same offset, including `+0x10`, can overlap directly. No allocation extent, object boundary, or disjointness guarantee is established.

### Preconditions and errors

Continued execution requires callable indirect targets, returning dependencies wherever continuation is needed, and supported accesses at every executed address.

No explicit checks establish these conditions. Null, low, misaligned, wrapped, and inaccessible addresses have no represented rejection path. Mapping, alignment, fault handling, and arithmetic behavior are environmental prerequisites rather than validation performed here.

## Behavior

### Indirect-call phase and arithmetic

The initial phase is a long straight-line sequence of indirect calls without a displayed controlling loop or conditional branch. Call order matters, including calls with unused results or no displayed arguments.

Early argument construction uses the first incoming value shifted left by 5 and 11, and the second shifted left by 20 and 26. Later arguments combine primary results shifted left by 8, secondary results, components of combined results, saved earlier values, masks, further shifts, and explicit zeros in several argument positions.

Unknown dependencies must not be assumed to preserve globals, maintain conventional register contents, or lack side effects. Displayed arguments establish relationships at call boundaries, not complete signatures.

Width boundaries are significant:

- Unsigned 32-bit transformations discard high bits at each operation.
- Masks select low 24, 22, 20, and narrower bit ranges.
- Extraction from combined results includes right shifts by 34, 36, and 44 bits. A 36-bit extraction uses the high 32-bit component shifted right by 4.
- Under 32-bit bit-vector semantics, two successive left shifts by 28 yield zero. The second shift operates on the already-truncated result.

Some represented operands are signed. Overflow, right shifts of negative values, and variable-count shifts cannot be assigned unconditional machine behavior from this material alone. Modular conclusions involving such operations remain conditional.

### Direct-memory entrance

The following descriptive names denote retained values, not identified objects:

- **Retained shifted base:** an earlier result shifted left by 25.
- **Initial scaled base:** a later scalar result multiplied by `0x02000000`.
- **Masked index:** a 32-bit left shift by 8, masked with `0x7FFF`, then logically shifted right by 6.
- **Doubled-index base:** twice the masked index.

Under 32-bit bit-vector semantics, the masked index is a multiple of 4 from 0 through 508; the doubled-index base is a multiple of 8 from 0 through 1016.

The first direct word store places the retained shifted value at `+0x14` of the final combined result’s high component. A byte store then places that high component’s low byte at retained shifted base `+9`. A word store places the initial scaled base at `+0x54` of the final low component multiplied by 256. The retained shifted base’s `+9` byte is subsequently overwritten with the retained shifted value’s low byte.

Initial scaled `+0x14` is cleared before initial scaled `+5` is read as a byte. Doubled-index `+0x14` is then cleared before the saved byte is zero-extended into the word at initial scaled `+0x34`.

A later word read of initial scaled `+0x14` supplies the **post-clear word**. It cannot unconditionally be replaced with zero: intervening accesses can alias that storage. Another doubled-index `+0x14` clear precedes two stores of the post-clear word’s low byte at doubled-index `+0x15`.

### Fixed low addresses and linking updates

A byte read at `0x05` follows those two byte stores. Its zero-extended result becomes the **byte-derived base**, ranging from 0 through 255. This is not a full-width pointer load.

Immediately afterward, the byte at `0x01` receives the doubled-index base’s low byte. Byte-derived `+0x15` then receives the masked index’s low byte.

Initial scaled `+0x11` receives the low byte of a retained shifted result. A word store through post-clear word `+0x54` follows, then initial scaled `+0x11` is cleared. Byte-derived `+0x54` is read as a word to obtain the **retained link value**, before that field is overwritten with the post-clear word. Doubled-index `+0x34` receives the masked index.

A word read at initial scaled `+0x74` supplies another base. Its `+0x54` first receives the earlier scalar result shifted left by 26, then later receives a different retained shifted result. An intervening indirect word-store expression uses doubled-index `+0x74` as both a loaded destination base and a loaded value. Repeated stores place the retained link value at its own `+0x24`, interleaved with repeated byte stores of that value’s low byte at the other loaded base’s `+9`.

Doubled-index `+0x14` subsequently receives three ordered word stores: the scalar result shifted left by 26, the retained link value, and the retained link value again. Between the first and second stores, the fixed word at `0x54` receives the retained link value shifted left by 1.

Two subsequent bytes at doubled-index `+0x15` each receive the low byte of four times the retained link value. The fixed byte at `0x15` then receives the doubled-index base’s low byte. These repeated accesses are not interchangeable with a final-state assignment.

### Further scaled-address mutations

Four times the retained link value forms the **fourfold link base**; eight times the retained link value supplies another retained scaled value.

This region combines words at `+0x14`, `+0x34`, `+0x44`, and `+0x54` with bytes at `+9`, `+0xD`, and `+0x15`. Stores include self-referential values and overwrites of earlier fields.

A word loaded from doubled-index `+0x74`, multiplied by 2, supplies another base. A separate path reads the byte at fourfold link `+5`, zero-extends it, and reads a word at `+0x74` of that byte-derived address. That word supplies a base for `+0x44` mutations and a low-byte value used elsewhere. A word loaded at fourfold link `+0x64` supplies a base for a byte update at `+0xD` and a self-store at `+0x14`.

A further word read at fourfold link `+4` supplies a base whose `+5` byte is read twice. Intervening writes separate those reads, so they need not agree. Their widened values participate in a word update and a later `+0x74` lookup.

The later byte-derived `+0x74` word becomes the **terminal scale value**. Twice it forms the **terminal doubled base**; four times it forms the **terminal fourfold base**. These retained arithmetic values are distinct from later contents read at their addresses.

### Constants and repeated publication stores

The numeric value `0x3A` is stored as:

- A byte at terminal fourfold.
- A halfword at eight times the terminal scale value.
- A word at terminal fourfold `+0x20`.

Fourfold link reduced by `0x96` supplies the **adjusted value**. Four times the adjusted value forms the **working base**.

Working `+0x20` receives four ordered word stores: `0x20`, `0x2F`, a zero-extended byte read from working `+4`, and `0x20` again. The byte read occurs between the second and third stores.

The word at `0x087FBBBC` is then loaded as the **publication base**, before the fixed word at `0x60` receives `0x20`. The loaded publication base is retained across that store; there is no subsequent refresh from the fixed source.

The first publication group has this four-store order:

1. Terminal fourfold as a word at `+0x54` of eight times the retained link value.
2. Working base’s low byte at publication `+0x19`.
3. Publication base’s low byte at working `+0x11`.
4. Eight times the retained link value as a word at publication `+0x34`.

The following repeated groups preserve that order, but use eight times the adjusted value for both the scaled `+0x54` destination base and the publication `+0x34` value.

The representation contains many separate repetitions, not a displayed loop. They retain the publication base and narrowed byte values rather than reloading them. Collapsing the groups into one update would discard represented access order and multiplicity.

### Final region: fixed bytes and saved words

After the repeated stores, a word is read from `0xA9`, followed by a word from terminal fourfold `+0x74`. The byte at fixed address `0x3A` receives the low eight bits of eight times the adjusted value plus `0x30`. This destination is distinct from the earlier stores of the numeric constant `0x3A`.

Next, words are saved from terminal doubled `+0x14` and working `+4`. The first remains the final transfer’s third displayed argument.

Before the fixed `0x67` byte write, calculated-address stores place:

- Working base’s low byte at terminal doubled `+0xD`.
- `0x6C` at working `+8`.
- Publication base’s low byte at working `+0xD` and `+9`.
- Terminal doubled as a word at its own `+0x54`.

Fixed `0x67` then receives the low byte of the saved `0xA9` word. A word read at terminal doubled `+0x44` intervenes before the byte read of `0x67` used for the copy to fixed `0x21`. The copy therefore uses the value read at that later position, not an unconditional reuse of the initially stored byte.

### Final region: shift and delayed reads

The retained terminal doubled `+0x44` word supplies a low-byte store at working `+0x10` and a word store at publication `+0x54`.

That retained word is then right-shifted by a count equal to working base’s low eight bits. The result is narrowed to a byte and stored at terminal doubled `+1`. The operand is represented as signed; behavior for negative values and counts outside the ordinary 32-bit range remains documented_only.

Publication `+0x54` is overwritten with the working base. Only afterward is working `+0x40` read as a word. Its value is not an earlier snapshot.

The next stores place the saved working `+4` word at publication `+0x74`, the newly read working `+0x40` word at working `+0x10`, and terminal doubled at publication `+0x44`.

Two further distinct reads of fixed byte `0x67` each supply a byte store at working `+9`. Between those reads are a byte store of working base’s low byte at terminal doubled `+9`, a word store of the saved `0xA9` value at working `+0x44`, and a word read at working `+0x54`.

Those rereads need not equal each other, the initial `0x67` store, or the byte copied to `0x21`. Intervening calculated-address writes may overlap the fixed byte, including through wider accesses.

### Final updates and transfer

The word just read from working `+0x54` becomes the **last link base**. Its `+0xD` byte receives the low byte of the earlier terminal fourfold `+0x74` word.

The saved terminal doubled `+0x14` word is restored into terminal doubled `+0x54`. Working `+0x14` is then read and retained. Terminal doubled `+0x11` receives the low byte of the earlier terminal fourfold `+0x74` word. Working `+9` receives last link’s low byte, and working `+0x44` receives last link as a word.

The following relative order is established and must be preserved:

- A word read at terminal doubled `+4` obtains the **last address value**.
- A halfword store places `0x65` at twice the last address value.
- A word read at last address `+0x64` obtains the retained word used below and as the final transfer’s second argument.
- Only then does last link `+0x11` receive the retained working `+0x14` word’s low byte.
- That byte store is immediately followed by the word store of `100` at last link `+0x54`.

The last link `+0x11` byte store does not precede either address-related word read or the halfword store. Moving it earlier can change a preceding read through aliasing. Unresolved overlap semantics do not make the represented ordering unresolved.

After the word store of `100`, last address `+0x74` is read and retained. Terminal doubled `+0x64` is then read as a word. Following another byte store at terminal doubled `+9`, now carrying last address’s low byte, the retained last address `+0x64` word is stored at `+0x54` through the value loaded from terminal doubled `+0x64`. A final word store places that latter loaded value at twice the retained last address `+0x64` word.

F-U03 receives these four displayed arguments, in order:

1. `0x72`.
2. The retained word read from last address `+0x64`.
3. The word saved earlier from terminal doubled `+0x14`.
4. The retained word read from last address `+0x74`.

Later mutations do not replace these retained arguments with fresh reads.

## Side effects

The operation is not established as pure, atomic, idempotent, reentrant, or thread-safe.

Direct effects include fixed-address writes, calculated-address writes, overlapping access widths, repeated stores, and dependencies on reads separated by writes. Relative access order is part of the represented behavior. No synchronization or hardware memory-ordering guarantee is established.

No allocation, release, reference-counting, ownership-transfer, or lifetime protocol is established. Address-like results do not prove allocation, and self-stores do not prove construction. Unknown dependencies may allocate, release, retain, or mutate memory.

There is no represented error result, rollback, cleanup path, or status check. Faulting accesses and nonreturning dependencies can prevent later operations while leaving earlier effects already performed. Fault delivery, recovery, and persistence remain documented_only.

## Dependencies

Only F-4a104faaa0f9 is resolved by the supplied dependency map. The following opaque IDs designate unresolved boundaries:

| Opaque ID | Boundary |
|---|---|
| F-U01 | Initial indirect call, targeted from incoming link state plus `0xFFE`. |
| F-U02 | Remaining ordered indirect-call family, with unresolved signatures and effects. |
| F-U03 | Final indirect transfer with four displayed arguments. |

F-U02 does not imply one shared callee. These boundaries establish neither target equality nor distinctness, target stability, global preservation, normal return, or memory ownership.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

### Retained publication base

If the publication-base load yields `0x11223344`, later publication accesses use that retained value even if the following word store at `0x60` changes the source storage through environmental aliasing. Reloading after the fixed store changes the specified dependency.

### Fixed-byte copy and rereads

If `0x67` initially receives `0xAA`, but the later read used for the `0x21` copy yields `0xBB`, the copied byte is `0xBB`. The intervening terminal doubled `+0x44` word read remains before that byte read.

If writes between the two later `0x67` rereads change the byte, the two working `+9` stores use their respective read values. Caching the original byte is insufficient.

### Delayed working-word read

If the shift’s byte store overlaps working `+0x40`, the subsequent word read occurs after that store and after the publication `+0x54` overwrite. Reading working `+0x40` before these writes is not equivalent.

### Last-link byte store remains delayed

If last link `+0x11` overlaps terminal doubled `+4`, the last-address word is read before the low-byte store to last link `+0x11`.

Likewise, if last link `+0x11` overlaps last address `+0x64`, that word is read before the byte store. Its retained value supplies F-U03’s second argument even if the later byte store changes the underlying storage.

In both cases, the intervening halfword store of `0x65` at twice last address remains after the last-address load and before the last address `+0x64` read. The last-link byte store remains immediately before the word store of `100`.

### Width boundaries

A byte load of `0xFE` widened for a word store produces `0x000000FE`, not `0xFFFFFFFE`.

Under 32-bit bit-vector semantics, shifting `0x12345678` left by 28 produces `0x80000000`; shifting that truncated result left by 28 produces zero.

### Saved argument and interrupted execution

If the saved terminal doubled `+0x14` word is `0x11223344`, F-U03’s third argument remains that value despite later changes to the original storage.

If F-U01 never returns, later direct accesses are not guaranteed and prior dependency effects need not be undone.

## Uncertainties

The following remain **documented_only**:

- Application purpose, valid entry conditions, and complete calling convention.
- Indirect target identities, signatures, and implicit register inputs.
- Combined-result delivery and secondary-register meaning.
- Dependency effects on globals, memory, ownership, and lifetimes.
- Byte order, address mapping, alignment, and overlap behavior.
- Signed overflow, address wraparound, signed right shifts, and variable-count shift semantics.
- Whether accessed storage is ordinary memory, device storage, or unsupported.
- Whether represented repetitions correspond one-for-one to actual runtime accesses.
- Final transfer destination, termination, and return behavior.

These unknowns do not erase established relative order within the supplied representation, including the final address load, halfword store, word read, delayed last-link byte store, and following word store of `100`.

**[INFERENCE]:** Recurring offsets and self-referential updates may represent linked state. That interpretation is not an acceptance requirement.

## Evidence

This contract uses the supplied static representation and dependency map. Prior prose, embedded comments, and reviewer claims are not independent behavioral authority.

Established evidence includes explicit widths, constants, saved-value reuse, fixed-address accesses, and represented ordering. In particular:

- The publication-base load precedes the fixed `0x60` word store.
- The fixed `0x67` copy and later rereads occupy distinct read positions.
- The variable shift’s byte store precedes the working `+0x40` word read.
- The final last-address load precedes the halfword store of `0x65`; the last address `+0x64` word read follows that halfword store; the last link `+0x11` byte store follows the word read and immediately precedes the word store of `100`.

No tools, implementation access, runtime execution, or tests were used. No snapshot hash is asserted; trusted provenance is supplied separately by the publishing system.
