<!-- Trusted provenance: F-e07f4fc06e0a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e07f4fc06e0a

## Purpose

F-e07f4fc06e0a performs an ordered sequence of indirect invocations, transforms ordinary and secondary results, mutates computed memory using mixed access widths, and reaches a final unresolved indirect invocation.

No domain-level purpose is established. Constants resembling characters and recurring offsets do not establish string construction, object initialization, container operations, allocation, or release.

This contract distinguishes:

- **Established static evidence:** operations, explicit dereference widths, value conversions, dependencies, and ordering visible in the supplied representation.
- **[INFERENCE]:** interpretations requiring additional assumptions.
- **documented_only:** unresolved behavior or details not exhaustively specified here.

A critical distinction applies throughout: **the width of a converted value or receiving temporary does not establish the storage-access width of a named fixed-location global whose declaration is absent.** Explicit computed dereferences provide stronger width evidence than those undeclared globals.

## Interface

### Explicit inputs

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit use in the body. |
| Second | 32 bits | No explicit use in the body. |
| Third | Signed 32 bits | Shifted left two positions and forwarded to two early invocations. |

The transformed third argument supplies argument two of one invocation and argument three of a subsequent invocation. The earlier invocation returns a represented 64-bit carrier; its low word shifted left eight and its unshifted high word supply the first two arguments of the later invocation.

There is no explicit range check or dereference of an explicit argument. Interpreting the third argument as a size, index, capacity, or pointer is unsupported.

Absence of explicit use does not prove that the first two arguments are irrelevant to the machine interface. Unresolved dependencies may consume incoming execution state beyond the visible argument expressions.

### Hidden inputs

Execution also depends on:

- Incoming control-link state selecting the initial target.
- Intermediate targets, calling conventions, ordinary results, and secondary result channels.
- Contents of computed memory.
- Named fixed-location inputs associated with `0x05`, `0x087fbbbc`, and `0xa9`.
- Three separate later references to the named global associated with `0x67`.
- The unresolved final target.

The declarations of these named globals are not supplied. Their storage widths, signedness, extension rules, and overlap relationships are **documented_only**. A represented 32-bit retained value does not prove a 32-bit source access.

### Outputs and completion

No explicit return value is established.

The final invocation has four visible argument positions:

1. `0x72`.
2. A late 32-bit read at offset `0x64` of a loaded base.
3. A previously saved 32-bit read at offset `0x14` of another derived base.
4. A later 32-bit read at offset `0x74` of the base used for argument two.

A represented return follows. Normal completion requires every preceding operation to complete and every dependency to return where execution continues. Whether the final transfer is an ordinary call is **documented_only**.

### Preconditions and errors

Completion requires valid reached targets, compatible execution conventions, and valid memory accesses with any required alignment and permissions. Computed addresses can enter low-address storage; no local guard prevents this.

No explicit validation, error status, retry, rollback, or cleanup is represented. Failure may leave earlier effects intact. Fault delivery, partial-write behavior, and dependency failure semantics are **documented_only**.

## Behavior

### Sequencing and observability

The represented path contains three phases:

1. Indirect invocations and value transformations.
2. Direct memory work.
3. Final indirect invocation.

No explicit conditional branch or loop appears. Dependencies may branch, mutate state, fail, or not return.

The ordering below describes the represented operation sequence, not a guarantee of independently observable hardware transactions. Volatility, synchronization, access-sensitive storage, and compiler treatment are unresolved. Nevertheless, equal final memory images do not justify discarding repeated references or stores when those accesses are observable.

Aliasing is not excluded. Saved values remain saved even if later stores overwrite their originating locations.

### Invocation dependencies and transformations

The initial target is selected using incoming control-link state plus `0xffe`. Subsequent targets are largely selected from represented internal values with the same displacement.

Invocations whose results are discarded remain part of the sequence. An invocation with no visible arguments is not established to have no hidden inputs. Calls passing zero are not interchangeable with calls having no visible argument.

Established transformation families include:

- Shifting represented 32-bit results left eight.
- Extracting low and high 32-bit components of 64-bit carriers.
- Retaining high components or their transformations across intervening calls.
- Forwarding secondary result-channel values.
- Masking fields before shifting.
- Repeating one transformed value in multiple argument positions.
- Passing explicit zero values.

The two early uses of the third explicit argument retain the paired-result relationship described under Interface. Later families combine ordinary results, saved components, masks, and secondary outputs.

The complete intermediate invocation trace and target equivalences are **documented_only** at this contract’s resolution. The summarized families do not establish dependency semantics or permission to reorder invocations.

### Widths and arithmetic

Explicit computed dereferences distinguish 8-bit bytes, 16-bit halfwords, and 32-bit words. Result carriers include represented 64-bit values.

For explicitly unsigned byte dereferences, widening to 32 bits zero-extends the byte. Explicit byte and halfword destinations retain eight and sixteen bits respectively. These rules do **not** establish widths or extension behavior for undeclared named globals.

Established unsigned field constraints include:

| Operation | Value constraint |
|---|---|
| Mask to 24 bits | Removes original bits 24–31. |
| Mask to 22 bits | Retains a value from zero through `0x3fffff`. |
| Mask to 20 bits, then right shift ten | Produces at most ten bits. |
| Mask to 19 bits | Retains a value from zero through `0x7ffff`. |
| Mask to 17 bits, then right shift four | Produces at most thirteen bits. |
| Mask to 13 bits, then left shift thirteen | Places the field beginning at bit 13. |
| Unsigned 64-bit right shift by 34, 36, or 44 | Uses the high word shifted right by 2, 4, or 12. |

Addresses depend on scaling by two, four, and eight, subtraction of `0x96`, and larger shifts.

**[INFERENCE]:** A fixed-width machine interpretation may truncate overflowing 32-bit results and wrap addresses. The representation alone does not establish that interpretation for signed shifts, signed multiplication, pointer conversion, or address arithmetic. Those semantics remain **documented_only**.

A late byte destination receives a signed-represented word shifted right by a count derived from the working base’s low eight bits. That count can exceed 31. Count masking and sign propagation are not established.

### Fixed-location globals: value evidence versus access evidence

The following table describes named-global references without assigning unsupported storage widths.

| Associated address | Established value-level behavior | Unresolved storage behavior |
|---|---|---|
| `0x05` | Evaluated and converted to an unsigned 32-bit retained base. | Source width, signedness, and extension before or during conversion. |
| `0x01` | Assigned a value explicitly narrowed to eight bits. | Destination width and conversion imposed by its declaration. |
| `0x15` | Assigned another explicitly eight-bit-narrowed value. | Destination width and declaration-dependent conversion. |
| `0x54` | Assigned a retained root shifted left one position. | Destination width and assignment conversion. |
| `0x087fbbbc` | Evaluated into a represented signed 32-bit retained base. | Source width and conversion to that representation. |
| `0x60` | Assigned `0x20`. | Destination width and stored byte layout. |
| `0xa9` | Evaluated into a retained 32-bit value. | Source width, signedness, and extension or truncation. |
| `0x3a` | Assigned arithmetic involving an eight-bit character conversion of the adjusted quantity, multiplication by eight, and addition of `0x30`. | Character signedness, final assignment conversion, and destination width. |
| `0x67` | Assigned the explicitly eight-bit-narrowed saved `0xa9` value; subsequently referenced three times. | Destination width, assignment conversion, and the width and extension of each later reference. |
| `0x21` | Assigned the value of the first later `0x67` reference. | Both globals’ declaration-dependent conversions and access widths. |

These are associated locations in the representation, not independently established object extents. No row proves disjointness or excludes overlap with computed accesses.

### Initial computed-memory effects

The memory phase begins with dependency-derived mutations: a word at offset `0x14`, two successive values at a byte destination at offset `0x09`, and a word at offset `0x54` of a scaled result.

Ordering-sensitive effects include:

- Clearing a computed word at offset `0x14` before an explicit unsigned byte read at offset `0x05` of that base.
- Widening that byte to a word written at offset `0x34`.
- Clearing another computed word destination twice.
- Reading the earlier cleared word for subsequent calculations.
- Issuing two consecutive identical byte writes at offset `0x15`.

After those consecutive writes, the named global associated with `0x05` is sampled and converted to an unsigned 32-bit value. **This evaluation precedes the assignment to the named global associated with `0x01`.**

Call the converted value the **sampled base**. It is not established to be byte-derived or bounded by `0xff`. It supplies a byte destination at offset `0x15`, a word read at offset `0x54`, and a later word overwrite at that same offset.

The offset-`0x54` read is retained despite the overwrite. Call that saved word the **retained root**. Re-reading its former source is not equivalent to using the saved value.

Further pointer chasing uses word offsets `0x74`, `0x64`, and `0x04`, and explicit unsigned byte reads at offset `0x05`. One expression separately reads the same offset-`0x74` location for destination selection and stored data. Equality of those observations is not guaranteed; their intra-expression evaluation order is unresolved.

Other established geometry includes byte offsets `0x01`, `0x08`, `0x09`, `0x0d`, `0x10`, `0x11`, `0x15`, and `0x19`, and word offsets `0x10`, `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, and `0x54`. This does not establish structure types or allocation extents.

### Derived bases and constant writes

The following descriptive terms identify retained values, not objects:

- **Adjusted quantity:** four times the retained root minus `0x96`.
- **Working base:** four times the adjusted quantity.
- **Nested value:** the word obtained through the later explicit-byte-based offset-`0x74` lookup.
- **Repetition base:** the represented signed 32-bit value sampled from the named global associated with `0x087fbbbc`.

Three explicit computed stores supply `0x3a`:

| Width | Destination |
|---|---|
| Byte | Four times the nested value |
| Halfword | Eight times the nested value |
| Word | Four times the nested value plus `0x20` |

These address expressions are not established to evaluate to distinct or nonoverlapping destinations.

At working-base offset `0x20`, four word writes supply, in order:

1. `0x20`.
2. `0x2f`.
3. A zero-extended explicit unsigned byte read from working-base offset `0x04`.
4. `0x20`.

The byte read occurs after the second write and supplies the third.

**After the fourth write, the fixed-location input associated with `0x087fbbbc` is evaluated into the repetition base. That evaluation precedes assignment of `0x20` to the named global associated with `0x60`.** Neither operation’s storage width follows from the receiving value or literal.

The repetition base is retained. Later references in the repeated region do not re-sample its originating global.

### Repeated-write region

An initial group performs these explicit computed accesses in order:

- A word write at eight times the retained root plus `0x54`, containing four times the nested value.
- A byte write at repetition-base offset `0x19`, containing the working base’s low eight bits.
- A byte write at working-base offset `0x11`, containing the repetition base’s low eight bits.
- A word write at repetition-base offset `0x34`, containing eight times the retained root.

The following finite repeated region preserves the four-write order but substitutes eight times the adjusted quantity for the first destination’s scaled base and the last write’s contents. The first write’s contents remain four times the nested value.

There are no visible intervening calls or reads within this region. Exact group multiplicity is **documented_only** here; it is neither one group nor an unbounded loop.

All destinations may alias. Retained bases do not change merely because their originating storage is overwritten, but overlapping destinations still receive the ordered writes.

### Late fixed-location dependencies

After the repeated region, the named global associated with `0xa9` is evaluated into a retained 32-bit value. This is not established to be a 32-bit load.

Next, an explicit word read occurs at offset `0x74` of four times the nested value. The assignment to the named global associated with `0x3a` follows that read.

Two subsequent explicit word reads save:

- The word at offset `0x14` of twice the nested value.
- The word at working-base offset `0x04`.

The first supplies final argument three. The second is later forwarded to repetition-base offset `0x74`.

After derived byte mutations and a word write at offset `0x54` of twice the nested value, the saved `0xa9` value is explicitly narrowed to eight bits and assigned to the named global associated with `0x67`. This narrowing does not establish a byte store to that global. The full retained 32-bit value remains available separately.

There are then **three separate represented references to `0x67`**:

1. After a word read at offset `0x44` of twice the nested value, its value supplies the assignment to the named global associated with `0x21`.
2. After further derived mutations, its value supplies an explicit byte destination at working-base offset `0x09`.
3. After intervening mutations and a word read at working-base offset `0x54`, its value again supplies that explicit byte destination.

These are not established to be three byte loads. Their source widths and extension behavior are **documented_only**. The second and third destinations are explicitly one byte; their destination width does not determine the source access width.

Substituting the original saved `0xa9` value, or one shared `0x67` snapshot, loses the three-reference dependency. Aliasing, access-sensitive storage, or external mutation can distinguish them.

The retained 32-bit `0xa9` value also supplies an explicit word write at working-base offset `0x44`. That destination is overwritten later.

### Final memory work and invocation

The late region includes an explicit byte write of `0x6c` at working-base offset `0x08`, low-byte forwarding of saved values, and restoration of the saved offset-`0x14` word to an offset-`0x54` destination. Different offsets are involved; this does not establish whole-object restoration.

Additional mutations include byte and word writes at working-base offset `0x10`, writes at repetition-base offsets `0x54` and `0x74`, the variable-shift byte write, and forwarding of saved words.

A word read at offset `0x04` of twice the nested value supplies the **late base**. A halfword `0x65` is written at twice that base. The word at late-base offset `0x64` is then saved as final argument two.

Intervening effects include a derived byte write at offset `0x11` and a word write of `100` at offset `0x54`. The word at late-base offset `0x74` is subsequently read for final argument four.

A word read at offset `0x64` of twice the nested value supplies another destination base. Before final invocation:

- A byte at offset `0x09` of twice the nested value receives the late base’s low eight bits.
- Final argument two is written as a word at the newly loaded destination base’s offset `0x54`.
- That destination base is written as a word at twice final argument two’s value.

The final invocation follows these effects.

## Side effects

The function mutates computed memory and named fixed-location globals. Dependencies may introduce additional effects before the direct-memory phase and during the final invocation.

No atomicity, synchronization, thread-safety, reentrancy, or disjointness guarantee is established. Endianness is **documented_only**. Explicitly overlapping mixed-width accesses therefore cannot be expanded into a unique byte image.

No operation establishes allocation, release, ownership transfer, or resource retention. Address-like values are not guaranteed to designate live or owned storage. Storing such a value does not establish transfer of ownership.

Unknown dependencies must not be assumed to preserve globals or memory, allocate storage, or release storage.

## Dependencies

Only F-e07f4fc06e0a has a resolved supplied identity.

- **F-U001:** document-local opaque identifier for the initial entry-state-selected invocation. No explicit arguments are shown; hidden inputs remain unresolved.
- **F-U002:** document-local opaque identifier for the final invocation, with the four visible argument positions described above.

These identifiers do not establish target distinctness.

Intermediate indirect dependencies remain unresolved. Their complete identities, signatures, preservation rules, ownership effects, errors, and target equivalences are **documented_only**.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Explicit argument forwarding.** With third argument `3`, the two early shifted positions receive `12`, provided the represented signed shift is valid under the applicable semantics.

2. **Explicit byte conversion.** A value `0x123456ab` supplied to an explicit byte destination contributes `0xab`. An explicit unsigned byte dereference yielding `0xe1`, widened to a word, contributes `0x000000e1`. This example does not apply to undeclared globals.

3. **Paired extraction.** A 64-bit carrier with high word `0x12345678`, shifted unsigned right by 36, produces `0x01234567`.

4. **Sampled-base retention.** If conversion of the `0x05` global produces unsigned 32-bit value `0x2000`, that is the retained sampled base. Later changes to its source do not change it. The representation does not establish that the source was one byte.

5. **Repetition-base sampling.** If the `0x087fbbbc` evaluation produces retained value `0x2000`, destinations include `0x2019` and `0x2034`, assuming ordinary address arithmetic. Re-sampling between groups violates the represented dependency.

6. **Narrow value, unknown global width.** Narrowing saved value `0x123456ab` before assignment to `0x67` produces an eight-bit intermediate `0xab`. Without a declaration, acceptance cannot require a one-byte destination or a particular wider stored representation.

7. **Three independent references.** If the three `0x67` references evaluate to `0x41`, `0x42`, and `0x43`, they respectively supply the `0x21` assignment and the two explicit working-base byte writes. This does not assert byte-width source loads or that `0x21` stores one byte.

8. **Aliasing and repeated effects.** The three computed `0x3a` stores retain their explicit widths and address expressions. Acceptance cannot require separate objects or collapse repeated stores solely because final values match.

9. **Clear followed by read.** An earlier cleared word reads as zero only when ordinary-memory behavior and absence of relevant intervening changes justify it.

10. **Final transfer.** If F-U002 is reached, argument one is `0x72`, and argument three is the saved word rather than a fresh read. A preceding fault carries no promised rollback or status return.

## Uncertainties

The following remain **documented_only**:

- Domain intent and consumer-level contract.
- Architecture, endianness, alignment, and address-space rules.
- Signed arithmetic, overflow, address wrapping, and variable-shift semantics.
- Declarations, storage widths, signedness, extension, truncation, and assignment conversions for every named fixed-location global.
- Whether represented references correspond to separate observable machine accesses.
- Calling conventions, secondary outputs, intermediate invocation trace, and target equivalences.
- Exact repeated-group multiplicity at this document’s resolution.
- Object extents, aliasing, validity, external mutation, and access-sensitive behavior.
- Ownership, lifetimes, dependency preservation, and final-transfer semantics.
- Accuracy of the represented function boundary.

The fixed-global sampling positions and three separate `0x67` references are established ordering dependencies. Their unresolved widths do not erase those dependencies.

## Evidence

Evidence is limited to the supplied static representation: explicit argument representations, invocation expressions, result extraction, arithmetic, explicit dereference types, conversions, fixed-location references, constants, and visible ordering.

Named-global declarations, dependency bodies, caller contracts, architecture documentation, and executable observations were not supplied. Embedded comments, the prior draft, and reviewer feedback are not independent execution evidence.

No runtime tests have been performed. This contract records supported static constraints without treating receiving widths, narrowing conversions, or suggestive names as proof of undeclared storage layouts.
