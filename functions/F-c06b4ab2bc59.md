<!-- Trusted provenance: F-c06b4ab2bc59; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c06b4ab2bc59

## Purpose

F-c06b4ab2bc59 performs a long ordered sequence of unresolved indirect invocations, transforms dependency-produced integer values, mutates memory through calculated addresses and named fixed-address storage, and performs a final unresolved indirect invocation.

No application-level purpose is established. Numeric constants resembling characters do not establish text output. Repeated offsets do not establish a container, record format, or shared object type.

This contract distinguishes:

- **Established static evidence:** operations, explicit conversions, pointer-access widths, retained values, and ordering represented in the supplied source.
- **[INFERENCE]:** consequences conditional on stated arithmetic or storage assumptions.
- **documented_only:** interfaces, declarations, or execution properties that remain unresolved.

Every described effect is conditional on execution reaching it. No runtime tests have been performed.

## Interface

### Explicit arguments and result

The represented interface has four argument positions and no declared scalar result.

| Position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | Passed unchanged as the first explicit operand of the initial invocation. |
| Second | 32 bits | No explicit use in the body. |
| Third | 32 bits | No explicit use in the body. |
| Fourth | Signed 32 bits | Shifted left by five to form the initial invocation’s second explicit operand. |

The apparent non-use of the second and third arguments does not establish machine-level irrelevance. Hidden inputs and complete calling conventions are documented_only.

### Implicit inputs

Execution depends on incoming control context, unresolved callable targets, primary and secondary dependency results, existing memory, and a named global whose value is retained as a base during the memory-mutation phase.

The initial target is formed by adding `0xFFE` to incoming control context. Subsequent pre-memory targets are largely formed from embedded location values with the same displacement. Target validity and dependency identities are unresolved.

### Observable outputs

Outputs comprise dependency invocations and memory mutations. No success status, error status, or caller-visible scalar return value is established.

The final invocation receives four represented operands:

1. `0x72`.
2. A signed 32-bit value obtained by a late word read at offset `0x64` through a loaded address.
3. A retained 32-bit value read earlier at terminal-base offset `0x14`.
4. A 32-bit value obtained by a later word read at offset `0x74` through that same late-loaded address.

The second and fourth operands are captured at different times, with intervening stores. They are not a simultaneous snapshot.

A return is represented after the final invocation. Whether its target returns normally, never returns, or has another control-transfer interpretation is documented_only.

### Preconditions and ownership

Every reached target must be callable. Every reached memory access must satisfy the applicable address, width, alignment, permission, and lifetime requirements. No explicit null, bounds, capacity, alignment, or permission checks are represented.

No allocation handle, buffer extent, ownership transfer, release obligation, or lifetime guarantee is established. Dependencies may mutate globals or invalidate reachable memory; their effects cannot be excluded.

## Behavior

### Dependency phase

A substantial straight-line sequence of indirect invocations precedes the first explicit pointer-based store. Invocations with unused results or no displayed operands remain observable dependency events; neither form establishes purity.

Results are consumed as 32-bit values, split 64-bit results, and secondary-channel values whose origins and calling-convention meaning remain unresolved. Operands include zeros, masked fields, shifted values, repeated operands, and values retained across intervening invocations.

The representation distinguishes extraction of a 64-bit result’s upper word from transformations of its lower word. Unsigned full-result right shifts by 34, 36, and 44 select upper-word information shifted right by two, four, and twelve respectively.

Field masks include widths from one through 24 bits. Masking before shifting is not interchangeable with shifting before masking. Retention of an arithmetic value does not imply preservation of the memory or global state from which it originated.

### Width and arithmetic model

Explicit pointer dereferences establish accesses of 8, 16, or 32 bits. Explicit unsigned-byte reads propagated into unsigned 32-bit values are zero-extended.

**Named globals require a separate rule:** their declarations are absent. A conversion of a global’s value into a 32-bit value does not establish a 32-bit underlying read. Likewise, assignment of an explicitly narrowed 8-bit value to a global does not establish an 8-bit underlying write.

For such globals, the contract specifies the represented value conversion and its ordering, while leaving storage width, declared signedness, and assignment conversion documented_only.

**[INFERENCE] Fixed-width model:** where 32-bit arithmetic retains the low 32 bits and explicit 8-bit conversions retain the low eight bits:

- Successive left shifts whose total is at least 32 discard all original bits.
- A value shifted left by 25 has a zero low byte.
- Retained zero values remain zero through the represented shifts and masks.
- The narrow-index construction—left shift by eight, mask with `0x7FFF`, then unsigned right shift by six—produces multiples of four from 0 through 508.
- Twice that index ranges from 0 through 1016 in increments of eight.

These are not portable guarantees for signed overflow or signed left shifts. Address overflow, signed arithmetic, and target-specific shift behavior remain documented_only.

### Address relationships

The following labels describe arithmetic relationships, not object types:

| Label | Meaning |
|---|---|
| Saved shifted base | A retained dependency value shifted left by 25. |
| Initial field base | Another dependency value multiplied by `0x2000000`. |
| Doubled-index base | Twice the narrow index described above. |
| Fixed-source-derived base | The unsigned 32-bit conversion of the named global associated with `0x05`. |
| Retained link | A signed 32-bit word subsequently loaded at fixed-source-derived-base offset `0x54`. |
| Adjusted value | Four times the retained link, minus `0x96`. |
| Scaled adjusted base | Four times the adjusted value. |
| Global-derived base | The 32-bit signed value retained from a later named-global read. |
| Terminal base | Twice a later pointer-derived signed 32-bit value. |

The fixed-source-derived base is **not established to lie between 0 and 255**. Its source declaration is absent. An unsigned 32-bit conversion alone establishes neither an unsigned-byte read nor zero-extension from eight bits.

### Opening memory effects

The last pre-memory dependency supplies low and high words. The opening effects preserve this order:

- A 32-bit store of the saved shifted base at high-word-derived offset `0x14`.
- An 8-bit store of the high word’s low byte at saved-shifted-base offset nine.
- A 32-bit store of the initial field base at offset `0x54` from 256 times the low word.
- Another 8-bit store at saved-shifted-base offset nine, now carrying the saved shifted base’s low byte.

Under the fixed-width model, the second offset-nine byte value is zero. The two stores remain distinct effects.

The routine then writes a zero word at initial-field-base offset `0x14`, reads an unsigned byte at offset five, writes a zero word at doubled-index-base offset `0x14`, and propagates the byte as an unsigned 32-bit value to initial-field-base offset `0x34`.

A word read at initial-field-base offset `0x14` precedes another zero-word store through the doubled-index base. The read must not be replaced with a constant zero: calculated accesses can alias.

Two successive byte stores at doubled-index-base offset `0x15` carry the low byte of that loaded word.

### Fixed-source-derived accesses and retained-link mutations

The named storage associated with `0x05` is read and converted to an unsigned 32-bit value. Its underlying access width and signedness are documented_only.

After that read, the named storage associated with `0x01` receives an explicitly 8-bit-converted doubled-index-base value. The destination’s actual storage width is unresolved.

Subsequent calculated accesses include:

- A byte store at fixed-source-derived-base offset `0x15`, carrying the narrow index’s low byte.
- A byte store and later zero-byte store at initial-field-base offset `0x11`.
- A word read at fixed-source-derived-base offset `0x54`, retaining the link, followed by replacement of that field.
- Word reads at offset `0x74` that supply destinations for further word stores.
- Self-address word stores at offset `0x24` and repeated offset-nine byte stores.

One assignment reads the same calculated offset-`0x74` location both to derive a destination and to select its stored value. The relative evaluation order of those two reads is unresolved.

An important interleaving surrounds doubled-index-base offset `0x14`:

**Calculated word store of a dependency value shifted left by 26 → assignment to fixed storage `0x54` of the retained link shifted left by one → two calculated word stores of the retained link.**

The expression assigned to fixed storage is represented as a signed 32-bit shift. Its actual destination storage width is not established. Under fixed-width arithmetic, the expression carries twice the retained link; the resulting stored bytes additionally depend on the missing declaration.

Two later calculated byte stores at doubled-index-base offset `0x15` carry the low byte of four times the retained link. Only afterward does fixed storage `0x15` receive the explicitly 8-bit-converted doubled-index base.

Further accesses use fourfold and eightfold retained-link values, follow words at offsets `0x04`, `0x64`, and `0x74`, and mutate calculated byte and word fields. Separate unsigned-byte reads at offset five participate in address derivation; these explicit pointer reads do establish 8-bit access and zero-extension, unlike the named `0x05` source.

### Literal writes and repeated-update transition

A later loaded word supplies addresses for literal `0x3A` writes of three different widths:

- An 8-bit write at four times the loaded word.
- A 16-bit write at eight times it.
- A 32-bit write at offset `0x20` from four times it.

These are memory operations, not established text output. Endianness is unresolved.

At scaled-adjusted-base offset `0x20`, four word stores occur in order: `0x20`, `0x2F`, a zero-extended byte read from that base’s offset four, and `0x20` again.

The next transition is:

**Final calculated offset-`0x20` store → named-global read retained as the global-derived base → assignment of `0x20` to fixed storage `0x60` → repeated-update region.**

The global-derived base is retained before the `0x60` assignment. If the missing declarations and storage mapping permit overlap, that assignment does not cause the routine to reload the base.

Each repeated-update group preserves four calculated stores:

1. A 32-bit store at a scaled-base offset `0x54`.
2. An 8-bit store at global-derived-base offset `0x19`, carrying the scaled adjusted base’s low byte.
3. An 8-bit store at scaled-adjusted-base offset `0x11`, carrying the global-derived base’s low byte.
4. A 32-bit store at global-derived-base offset `0x34`, carrying the group’s scaled base.

The first group uses eight times the retained link as its scaled base. Subsequent groups use eight times the adjusted value. The offset-`0x54` data is four times the loaded value whose double forms the terminal base.

This is an unrolled sequence, not input-controlled iteration. Exact multiplicity is not enumerated in this contract and remains documented_only. Repeated accesses must not be collapsed into one final-state update.

### Final named-global interactions

After the repeated updates, the named source associated with `0xA9` is read into a retained 32-bit value. The source’s declared width and signedness are unresolved; this is not an established 32-bit underlying read.

That read precedes a calculated 32-bit read at offset `0x74` from twice the terminal base, and precedes the assignment to fixed storage `0x3A`.

The `0x3A` assignment is distinct from the earlier literal writes. Its arithmetic first converts the adjusted value to an 8-bit character value, then multiplies that converted value by eight and adds `0x30`. Integer promotion participates in the arithmetic. Character signedness and destination declaration are unresolved.

**[INFERENCE]:** under low-eight-bit character conversion, the expression’s low byte equals the low byte of eight times the adjusted value plus `0x30`. That does not establish an 8-bit destination write.

The routine retains words from terminal-base offset `0x14` and scaled-adjusted-base offset four. Calculated stores include the byte `0x6C` at scaled-adjusted-base offset eight and a self-address word at terminal-base offset `0x54`.

Only afterward does fixed storage `0x67` receive an explicitly 8-bit-converted value from the retained `0xA9`-derived result. The actual `0x67` write width remains unresolved.

A word read at terminal-base offset `0x44` follows. Fixed storage `0x21` then receives a fresh read of `0x67`; neither global’s access width or assignment conversion is established.

Two subsequent calculated byte stores at scaled-adjusted-base offset nine each have their own separate `0x67` read. Between these reads are:

- A byte store at terminal-base offset nine.
- A word store of the retained `0xA9`-derived 32-bit value at scaled-adjusted-base offset `0x44`.
- A word read at scaled-adjusted-base offset `0x54`.

The separate reads cannot be replaced by reuse of the original narrowed value. Calculated stores may affect the declared storage behind `0x67`, although exact overlap consequences require its declaration and memory layout.

### Final mutations and invocation ordering

The final phase also includes a byte store followed later by a word store at scaled-adjusted-base offset `0x10`, successive word replacements at global-derived-base offset `0x54`, and further offset-nine byte stores.

A byte written at terminal-base offset one derives from a signed 32-bit word read at terminal-base offset `0x44`, right-shifted by a count selected from the scaled adjusted base’s low eight bits. Counts may exceed 31. Sign propagation, count masking, and out-of-range behavior are documented_only.

The earlier retained terminal-base offset-`0x14` word is subsequently stored at terminal-base offset `0x54`. Intervening memory mutations do not replace this retained value.

Near completion, a word read at terminal-base offset four supplies the late-loaded address. A 16-bit literal `0x65` is written at twice that address.

The final operand-capture interleaving is established:

**Late offset-`0x64` word read → calculated offset-`0x11` byte store → calculated offset-`0x54` word store of decimal 100 → late offset-`0x74` word read.**

The byte store uses the low byte of a previously captured scaled-adjusted-base offset-`0x14` word. Both intervening stores use a separately loaded base.

Consequently, either store may affect the later offset-`0x74` read if addresses overlap. The offset-`0x64` result remains retained even if those stores overwrite its original source.

After the offset-`0x74` operand is captured, the routine reads another word at terminal-base offset `0x64`, writes a byte at terminal-base offset nine, and performs two pointer-derived word stores. Those later stores do not cause either final operand to be reloaded. The final invocation then receives the four operands specified in the interface.

## Side effects

### Fixed-address storage

Widths below describe **values or conversions**, not unresolved underlying global accesses.

| Associated address | Established value relationship and ordering | Underlying access width |
|---|---|---|
| `0x01` | Receives an explicitly 8-bit-converted doubled-index base after the `0x05` read. | documented_only |
| `0x05` | Read value is converted to unsigned 32 bits for base derivation. | documented_only |
| `0x15` | Receives an explicitly 8-bit-converted doubled-index base after two calculated byte stores. | documented_only |
| `0x21` | Receives a fresh read of `0x67`; source and destination conversions are unresolved. | documented_only |
| `0x3A` | Receives promoted arithmetic based on an 8-bit character conversion of the adjusted value. | documented_only |
| `0x54` | Receives the represented signed 32-bit left-shift expression for the retained link. | documented_only |
| `0x60` | Receives literal `0x20` after global-base capture. | documented_only |
| `0x67` | Receives an explicitly 8-bit-converted retained value; subsequently read three separate times. | documented_only |
| `0xA9` | Read value is captured in 32-bit storage before the final derived offset-`0x74` read. | documented_only |

Calculated accesses have their explicitly represented widths. They may overlap each other and may reach fixed or global storage. Exact byte-level consequences involving named globals require the missing declarations, endianness, and mapping.

No transaction, rollback, synchronization, atomicity, or memory-mapped-I/O guarantee is established.

### Failures and resources

Dependency effects—including allocation, release, retention, global mutation, I/O, reentrancy, and non-returning behavior—are documented_only.

No explicit diagnostic, recovery branch, error return, or cleanup path accompanies failure. Invalid targets or invalid accesses may prevent later effects. Earlier effects are not represented as undone. The actual fault or exception mechanism is unresolved.

## Dependencies

The supplied dependency map resolves no invoked target. These IDs are contract-local opaque references:

| Opaque ID | Scope |
|---|---|
| F-unknown-001 | Initial indirect invocation. |
| F-unknown-002 | Remaining ordered pre-memory invocations; a sequence of unresolved targets, not one common callee. |
| F-unknown-003 | Final unresolved invocation. |

F-unknown-001 receives the first argument unchanged and the fourth argument shifted left by five.

F-unknown-002 supplies primary words, split 64-bit values, and incompletely specified secondary-channel values. Complete calling conventions, operand contracts, and side effects remain documented_only.

F-unknown-003 receives the constant and three retained values described above. Its ownership protocol and return behavior are unknown.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Initial operand:** with fourth argument three, the initial invocation’s second explicit operand is 96, assuming the represented shift executes normally.

2. **Index boundary:** under fixed-width arithmetic, narrow index 508 is possible and doubles to 1016. Narrow index 511 is inconsistent with the construction.

3. **Established unsigned-byte propagation:** an explicit unsigned-byte read of `0xFF` at initial-field-base offset five propagates as 255 in the subsequent word store.

4. **Unresolved fixed source:** if the `0x05` source were a signed 8-bit value of −1, its unsigned 32-bit conversion would yield `0xFFFFFFFF`, not 255. An unsigned 8-bit declaration would instead yield 255 for stored bits `0xFF`. The declaration is therefore necessary to select either outcome.

5. **Narrow value versus storage width:** doubled-index base eight supplies the explicitly narrowed value eight to fixed storage `0x15`. An 8-bit write is required only if the missing destination declaration establishes that width.

6. **Fixed shift expression:** retained link seven supplies expression value 14 to fixed storage `0x54`, between the surrounding calculated word stores. Exact destination bytes remain declaration-dependent.

7. **Character arithmetic:** if the adjusted value converts to character value two, the expression assigned at `0x3A` evaluates to `0x40`. Calling this an 8-bit write requires an additional destination-width assumption.

8. **Fresh global reads:** if a calculated store changes the storage read as `0x67`, a later calculated offset-nine byte store uses its own fresh read, converted to eight bits, rather than necessarily reusing the earlier narrowed value.

9. **Final alias-sensitive read:** if the decimal-100 word store exactly aliases the late offset-`0x74` word source, ordinary coherent-memory semantics make that later read observe 100, absent other effects. The offset-`0x64` operand remains the previously captured value.

10. **Partial failure:** if a reached access faults after earlier stores complete, the evidence requires neither rollback nor execution of the final invocation.

## Uncertainties

The following remain documented_only:

- Application purpose and complete machine interface.
- Callee identities, hidden inputs, secondary-channel origins, and resource effects.
- Named-global declarations, access widths, signedness, and assignment conversions.
- In particular, the original width and signedness of the `0x05` source.
- Character signedness and exact stored representation of the `0x3A` expression.
- Signed overflow, address overflow, and target-specific shift semantics.
- Endianness, alignment restrictions, volatile behavior, and memory-mapped-I/O effects.
- Object extents, layouts, lifetimes, and ownership.
- Aliasing and byte-level overlap involving undeclared global storage.
- Relative evaluation order within the identified dual-read assignment.
- Exact repeated-update multiplicity, not enumerated here.
- Fidelity of the represented control flow and final-target return behavior.

Unusual targets and low-address references establish neither application intent nor representation accuracy.

## Evidence

This contract uses only the supplied static function representation and dependency map. Comments, prior documents, and reviewer feedback are not independent behavioral evidence.

Explicit pointer accesses establish their represented widths. Named-global expressions establish value conversions and sequencing without supplying the missing global declarations. In particular, unsigned conversion of the `0x05` source does not prove an unsigned-byte access.

The final source sequence places the late offset-`0x64` read before the calculated offset-`0x11` byte store and decimal-100 word store, and places the offset-`0x74` read after both stores.

Arithmetic simplifications and conditional memory outcomes are labeled as assumptions or inferences. No dependency purity, ownership protocol, successful execution, external implementation inspection, or runtime testing is assumed.
