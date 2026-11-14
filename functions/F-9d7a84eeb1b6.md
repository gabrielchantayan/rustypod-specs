<!-- Trusted provenance: F-9d7a84eeb1b6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9d7a84eeb1b6

## Purpose

The established behavior is an ordered mutation of memory reached through explicit arguments, additional entry state, shared storage, and 32-bit address-like fields, followed by an unresolved indirect transfer.

No application-level purpose is established. The evidence does not justify identifying this operation as construction, serialization, text processing, collection maintenance, or resource cleanup. Literal byte values and self-address stores do not establish those interpretations.

This contract separates:

- **Established static evidence:** represented access widths, ordering, constants, and captured-value dependencies.
- **Inferred intent:** possible interpretations, none of which are acceptance requirements.
- **documented_only:** unresolved behavior requiring additional machine, interface, or dependency information.

The visible mutations are not transactional. No rollback mechanism is represented.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | No visible use in the body. |
| Second argument | 32 bits | Doubled to form an early halfword destination. |
| Third argument | 32 bits | Used in an entry-state-dependent destination and a word read at offset `0x14`. |
| Fourth argument | 8 bits | Stored at offset `0x45` from a captured shared address base. |

The unused first argument is not established as optional at the external interface.

### Additional entry state

Execution also consumes:

- An address base for an early byte store.
- A 32-bit quantity used both as an offset from the third argument and independently as an address base.
- Two 16-bit quantities.
- An unresolved transfer target, also used in an early comparison.

These values are consumed without visible initialization. Their calling-convention locations are **documented_only**.

Shared storage supplies:

- An early address base.
- A value whose low 16 bits enter an early calculation.
- A 32-bit traversal payload.
- A terminal 32-bit arithmetic input.
- A later terminal address-like word.
- A terminal 32-bit payload.

These inputs are read at distinct points. They are not an entry-time snapshot.

### Output and preconditions

The represented interface has no explicit return value. Observable outputs are memory effects and the final transfer, including its three visible arguments.

There is no represented status code, error object, completion flag, or exception translation. Successful execution requires every actual destination, source, and transfer target to be usable under the applicable machine rules. No validation enforces that requirement.

Memory requirements include computed sums, doubled values, unaligned-looking offsets, and an access before a traversal-derived address. A valid object at each apparent base is therefore insufficient by itself.

### Width and arithmetic rules

The representation distinguishes 8-bit, 16-bit, and 32-bit accesses. Narrowing retains the corresponding low bits. Address-like fields are represented as 32-bit quantities; replacing them with host-width pointers would not preserve the stated interface.

The early arithmetic combines two signed 16-bit interpretations and a Boolean contribution, then stores the low 16 bits. Its mathematical range fits a signed 32-bit intermediate. This does not resolve overflow or wrapping for the separate 32-bit address calculations.

Endianness, alignment behavior, address-space mapping, signed address-arithmetic overflow, character signedness, and out-of-range shifts remain **documented_only**.

## Behavior

### Ordering and aliasing

Before the final transfer, the represented path is linear. The target comparison supplies an arithmetic contribution rather than selecting a different traversal.

No non-aliasing guarantee is established. Consequently:

- Every read observes memory after all preceding overlapping writes.
- Separate reads of the same field are separate observations.
- Captured values are not implicitly refreshed when their source storage changes.
- Byte and halfword stores can affect later word reads.
- An overwritten store remains an access in the represented sequence.
- A read-and-write-back operation is not dispensable merely because its destination appears unchanged.

These rules apply to shared storage as well as traversal fields. External observation, device-memory behavior, concurrency, and atomicity are not established.

The descriptive names below identify captured values, not object types, ownership, or private implementation identifiers.

### Entry mutations and early shared reads

Let the **entry offset** denote the additional 32-bit entry quantity, and the **initial destination** denote the third argument plus that quantity.

Three stores precede the early shared reads:

1. A halfword containing the low 16 bits of the entry offset is stored at the initial destination.
2. The second argument’s doubled value supplies the destination for the other entry-state halfword.
3. Byte `0x45` is stored at offset `0x45` from the entry-state byte-store base.

The shared address base is captured next. The shared halfword contribution is captured afterward by retaining the low 16 bits of another shared value.

Only then is the initial destination read as a 32-bit address-like word. This read reflects **all** preceding overlapping writes, including the doubled-argument halfword store and the entry-state-derived byte store. It is not necessarily a word containing the first halfword value.

The address obtained by this read receives byte `0x45` at offset `0x45`. The initial halfword store is then repeated. That intervening byte store may itself overlap the initial destination or shared storage.

A halfword is stored at offset `0x45` from the captured shared base. It contains the low 16 bits of the sum of:

- The remaining entry-state 16-bit quantity;
- The captured shared halfword contribution;
- One when the unresolved target compares below address `0x46`, otherwise zero.

The initial destination is read again as a word. Call this captured result the **reloaded entry address**. The second read occurs after the repeated halfword store and the shared-base halfword store; it need not equal the first read.

The reloaded entry address receives word `0x45` at offset `0x45`. Finally, the fourth argument is stored as one byte at offset `0x45` from the captured shared base. These are distinct-width accesses, not a combined update.

### First traversal and payload capture

A word from the third argument’s offset `0x14` becomes the **original anchor**. It is stored at offset `0x34` from the entry offset treated independently as an address.

The next two address reads obtain:

- The **shared anchor**, from offset `0x74` of the captured shared base;
- Another reached address, from offset `0x14` of the reloaded entry address.

The shared anchor’s `0x54` field receives the reloaded entry address. The other reached address receives its own address at `0x54`. The shared anchor’s `0x44` field receives the original anchor. The reloaded entry address receives the shared anchor’s low byte at offset `0x09`.

Traversal then reads the shared anchor’s `0x64` field and follows that result’s `0x14` field. Call the final 32-bit result the **shift-base value**. At offset `0x09` from this value, two byte writes occur consecutively: first the low byte of the address reached through the reloaded entry address, then the low byte of the intermediate address reached through the shared anchor’s `0x64` field.

The **shared traversal payload is read immediately after those two byte stores and before the subsequent word read from the original anchor’s offset `0x14`**. This boundary is observable: either byte store may overlap the payload’s shared storage, and subsequent traversal writes do not retroactively change the capture.

The word subsequently read from the original anchor’s `0x14` field is increased by `0x20`. The resulting value participates in byte writes and a word store. In particular, the original anchor’s `0x54` field receives that value and then the captured traversal payload.

### Derived byte and related mutations

A byte is read at twice the shift-base value, interpreted through the represented character type, promoted, and shifted right by the shift-base value’s low eight bits. The low byte of that result is used.

This has two unresolved conditions:

- Character promotion may sign-extend or zero-extend.
- The count can exceed the promoted operand’s ordinary width.

No shift-count masking rule is established. Out-of-range results are **documented_only**; neither zero nor a particular processor’s masking behavior may be assumed.

Before this byte derivation, the shared anchor’s `0x44` field is read twice as separate word observations. One capture is later stored at the shared anchor’s `0x74` field; the other is treated as an address whose `0x08` field receives the derived byte.

The shared anchor’s `0x11` field also receives the derived byte. Later, it receives the low byte of the captured traversal payload. The original anchor receives additional low-byte writes at offsets `0x11` and `0x19`, while related reached fields receive address-like values.

### Repeated traversal and replacement

The middle region follows fields at offsets including `0x04`, `0x14`, `0x64`, and `0x74`, interleaving those reads with byte and word stores. Destinations are determined by the values read at those points, not by an immutable entry-time graph.

Established mutation classes include:

- Self-address stores at offsets `0x44` and `0x54`.
- Cross-address stores at offsets `0x14`, `0x34`, `0x44`, `0x54`, and `0x74`.
- Captured-word copies into offsets `0x10`, `0x24`, `0x34`, `0x44`, and `0x54`.
- Byte stores containing low bytes of addresses or captured words.
- Replacement of fields previously used for traversal.
- Explicit read-and-write-back operations at offsets `0x54` and `0x64`.

The shared anchor’s `0x14` field is read repeatedly. A later such read establishes a **central anchor** used extensively through the terminal phase. It must not be equated unconditionally with an earlier value read from the same field.

Literal stores retain their individual widths:

| Literal | Established destinations and widths |
|---|---|
| `0x73` | Words at offsets `0x10` and `0x20`; bytes at `0x14` and `0x0c`. |
| `0x6e` | Words at `0x10` and `0x20`, including replacement of an earlier `0x73` word. |
| `0x79`, later `0x68` | Bytes at the same reached address’s `0x1c` field, with other stores between them. |
| `0x64` | Bytes at `0x10`; words at `0x30`, `0x40`, `0x54`, and `0x60` on reached addresses. |
| `0x74` | Bytes at `0x09`, `0x10`, and `0x11`; words at `0x10`, `0x30`, and `0x54`. |
| `0x66` | Both word and byte stores at `0x10`; words at `0x40`, `0x54`, and `0x60`. |

This table is not a replacement execution order. Reads and other writes separate many listed occurrences.

One word destination lies `0x20` before a captured traversal address. The stored value is that same captured address minus `0x74`. Both subtractions require the target’s address-arithmetic interpretation.

### Later central-anchor mutations

The central anchor undergoes repeated updates at offsets `0x14`, `0x24`, `0x34`, `0x44`, `0x54`, `0x64`, and `0x74`, together with low-byte writes elsewhere.

Ordering-sensitive effects include:

- A reached word is captured and later written back after intervening mutations.
- Two identical byte stores occur consecutively at the central anchor’s `0x0d` field.
- A captured word from a reached address’s `0x64` field later replaces the central anchor’s `0x14` field.
- A reached address’s offset `0x00` receives the low byte of its separately captured `0x50` word.
- A reached address receives its own address at `0x54`.
- A late reached address receives byte `0x64` at `0x10` and word `0x64` at `0x40`.
- The central anchor’s `0x54` field receives word `0x64` after earlier replacements.

A late traversal also captures an address from a reached `0x74` field for the terminal character read. A separate late reached address supplies the doubled-address halfword destination. Another previously captured traversal address supplies the base of the final halfword destination.

These are value captures. Subsequent aliasing may alter their source fields without changing the captured values.

### Terminal shared reads, writes, and transfer

The first additional terminal shared word is read **before** reloading the central anchor’s `0x74` field. Call these captures the **first terminal shared word** and the **terminal link**, respectively.

The terminal link’s low half is stored at the sum of the terminal link and central anchor. The late doubled-address destination then receives the first terminal shared word’s low half.

A character is read from the sum of the saved late traversal address and central anchor, then converted to an unsigned 32-bit value. Call that result the **character-derived value**. A source byte above `0x7f` may produce different values depending on character signedness.

The terminal link’s low half is next stored at the sum of the first terminal shared word and terminal link. The earlier halfword store at the sum of terminal link and central anchor is repeated. Word `0x45` is then stored at offset `0x45` from the character-derived value.

The remaining capture order is exact:

- The later shared address-like word is read **after** that character-derived word store.
- The terminal shared payload is read next.
- The transfer’s third argument is then captured from the central anchor’s `0x44` field.

Thus the word store of `0x45` can affect either subsequent shared read through aliasing. Conversely, the following terminal stores cannot retroactively change these captures.

After those reads, byte `0x44` is stored at offset `0x44` from the character-derived value. The captured terminal payload is stored as a word at twice the later shared address-like value. Finally, the later shared address-like value’s low half is stored at its sum with the saved traversal address used for this final destination.

The unresolved target is transferred to with three visible arguments:

1. The character-derived value bitwise-ORed with `0x44`;
2. The captured later shared address-like value;
3. The captured central-anchor `0x44` word.

The final argument is not re-read after the terminal byte, word, or halfword stores.

## Side effects

The established effects are ordered memory reads, byte/halfword/word writes, and the unresolved transfer.

No allocation or release operation is visible before that transfer. This does not constrain the target’s allocation, release, global mutation, or ownership behavior.

No owning-pointer interpretation, lifetime extension, or ownership transfer is established for any stored address. Callers must not infer ownership from self-address or cross-address stores.

No bounds checking, null checking, synchronization, atomicity, recovery, or rollback is represented. A later fault can follow earlier completed mutations. Fault delivery, partial completion of individual stores, and resumability are **documented_only**.

## Dependencies

The only supplied opaque function identity is **F-9d7a84eeb1b6**. No distinct opaque function ID is supplied for its indirect dependency; assigning one would invent provenance.

| Dependency | Required behavior or information | Unresolved portion |
|---|---|---|
| Additional entry state | Supplies every consumed entry value. | External ABI and locations. |
| Shared storage | Supplies values at the specified read boundaries. | Ownership, overlap extent, and concurrency. |
| Reachable memory | Supports actual computed addresses and access widths. | Layout, lifetime, permissions, and alignment rules. |
| Indirect dependency | Receives the three captured terminal arguments under the actual calling convention. | Identity, side effects, preservation, failure, and return behavior. |
| Machine model | Determines conversions, shifts, arithmetic, and memory access semantics. | Target-specific rules not supplied here. |

No assumption that the indirect dependency preserves globals or owns, retains, or releases memory is permitted.

## Acceptance examples

These are conditional static criteria, not reported executions.

1. **Intervening entry overwrite.**  
   If the doubled-second-argument destination equals the initial destination, its halfword store follows and may replace the first halfword. The first word read must reflect that replacement, plus any overlap from the intervening byte store.

2. **Qualified narrow-write dependency.**  
   If neither intervening entry store overlaps the initial destination, the first word read reflects the initial halfword modification under the selected byte order. Its remaining bytes come from the then-current storage, not an assumed original-word snapshot.

3. **Early partial progress.**  
   If the first store succeeds and the doubled-address store faults, the first store precedes the fault. No rollback is represented.

4. **Traversal payload boundary.**  
   If the second consecutive `0x09` byte store overlaps the shared traversal payload, the subsequent payload capture observes that effect. A later overlapping traversal store does not refresh the capture.

5. **Terminal first-read boundary.**  
   If the first terminal shared word overlaps the central anchor’s `0x74` field, both reads still occur in the specified order. They are not interchangeable or one combined snapshot.

6. **Terminal post-write capture.**  
   If the character-derived word destination overlaps the later shared address-like word or terminal payload, those shared captures occur after the word store and observe its effect according to overlap and byte order.

7. **Captured transfer argument.**  
   If a final terminal store overlaps the central anchor’s `0x44` field, the transfer still receives the word captured before those stores.

8. **Intermediate literals.**  
   For memory observing each byte write, the reached `0x1c` field receives `0x79` before `0x68`, with intervening stores preserved. Reporting only the latter loses represented behavior.

9. **Character-dependent terminal address.**  
   A terminal byte `0xff` can convert to unsigned `0xffffffff` after signed interpretation or `0x000000ff` after unsigned interpretation. The evidence does not select one. Subsequent addresses and the first transfer argument remain conditional.

10. **Transfer completion is unresolved.**  
    Completing every preceding store does not establish successful return. The indirect dependency may mutate memory, fail, or not return.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and complete external ABI.
- Target byte order, character signedness, alignment, address arithmetic, and excessive-shift behavior.
- The machine interpretation of the target comparison.
- Object boundaries, valid ranges, permissible aliasing, concurrent changes, and memory-observation rules.
- Semantic meanings of fields and literals.
- Shared-symbol overlap beyond the indicated possibility.
- The indirect dependency’s identity, ownership effects, global effects, and return behavior.
- Machine-level fault delivery and individual-store partial-write behavior.

The supplied representation reports overlapping shared symbols and an unresolved terminal transfer. These limitations prevent certification of a portable executable contract. They do not justify ignoring visible accesses or replacing the operation with a guessed algorithm.

## Evidence

This contract is based solely on static inspection of the supplied representation for **F-9d7a84eeb1b6**.

Established evidence includes the represented arguments, consumed additional entry state, access widths, literal values, traversal offsets, distinct shared-read boundaries, repeated stores, captured-value dependencies, and three visible terminal transfer arguments.

No runtime tests, execution traces, ABI verification, independent memory-layout inspection, or analysis of the indirect dependency have been performed. Acceptance examples express conditional consequences of the static evidence, not observed outcomes.
