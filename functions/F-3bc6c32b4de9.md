<!-- Trusted provenance: F-3bc6c32b4de9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3bc6c32b4de9

## Purpose

F-3bc6c32b4de9 performs a long sequence of indirect invocations, derives arguments and address components from their results and implicit entry state, mutates computed and fixed-address storage, and ends with an indirect transfer carrying four arguments.

No business-level purpose is established. Repeated offsets, self-valued writes, and character-like constants do not establish a container, text format, device interface, or serialization protocol.

This contract distinguishes:

- **Established static evidence:** represented operations, access widths, retained-value relationships, and ordering.
- **[INFERENCE]:** interpretations requiring assumptions beyond the supplied representation.
- **documented_only:** unresolved behavior for which no replacement guarantee is supplied.

Execution ordering is conditional on the representation accurately reflecting execution. It does not establish synchronization or hardware memory-ordering guarantees.

## Interface

### Explicit inputs

The represented interface has three 32-bit input slots and no explicit scalar return value.

| Input | Established use |
|---|---|
| First | No explicit body-level use. |
| Second | No explicit body-level use. |
| Third | Supplies bits 10–19, repositioned into bits 5–14, to an early indirect invocation. |

The third-input transformation is unsigned. Its result ranges from 0 to 32,736 in multiples of 32. Bits 0–9 and 20–31 do not affect this particular argument.

This does not prove whole-function independence from the first two inputs or the third input’s remaining bits. Dependencies’ access to incoming machine state and the actual calling convention remain unresolved.

### Implicit inputs

Execution also depends on:

- Incoming continuation-like state used to form the first invocation target.
- An incoming word used in early shifts by 6 and 12.
- Secondary result channels associated with indirect invocations.
- Fixed-address global storage.
- Memory reached through computed addresses and loaded pointer-like words.
- A terminal indirect target whose origin is not established.

These execution-context requirements cannot safely be replaced with zeros.

### Outputs, preconditions, and ownership

Observable outputs are dependency invocations, memory accesses and mutations, and the terminal transfer. No explicit success value, error value, allocation result, or ownership transfer is established.

Continuation requires executable targets, compatible calling conventions, returning dependencies where subsequent execution is needed, and valid reached memory accesses. Storage must support the represented widths and applicable alignment rules, including low-address accesses. No validation is visible.

No dependency is assumed to acquire, preserve, or release memory ownership. Object extents and lifetimes are **documented_only**.

The terminal transfer is represented as an invocation followed by return. Whether execution actually returns normally is **documented_only**.

## Behavior

### Dependency phase

A long straight-line invocation phase precedes the directly represented data-memory mutations. Calls are interleaved with arithmetic and retention of results. The initial target depends on incoming continuation-like state; most subsequent targets use retained continuation-like locations plus 4,094 bytes.

No explicit branch, retry loop, status-dependent exit, or recovery path appears. Dependencies may nevertheless branch, mutate state, fail, or prevent continuation.

Established argument relationships include:

- Word results shifted left by 8 and other fixed counts.
- Represented 64-bit results split into low and high 32-bit words.
- Shifted low words passed with separately supplied high words.
- Duplicate arguments derived from the same retained value.
- Explicit zeros, including values that remain zero through later shifts.
- Masks retaining fields of several widths.
- Truncating left shifts followed by unsigned logical right shifts.
- Wide-result extraction beginning at bit 34, 36, or 44.
- Results retained across intervening invocations.

Zero arguments do not make an invocation optional. Argument shapes alone establish no allocation, validation, I/O, or ownership semantics.

The complete per-site invocation trace is **documented_only** in this contract. Similar target formation does not establish a common target identity.

### Widths and arithmetic

Byte, halfword, and word accesses represent 8, 16, and 32 bits. Most values and address components are 32 bits; some dependency results are represented as 64 bits before extraction.

Unsigned 32-bit left shifts discard bits above bit 31. Unsigned right shifts insert zeros. Byte and halfword stores retain the low 8 and 16 bits. Explicit unsigned byte loads propagated into words are zero-extended.

Signed shifts, multiplication, subtraction, and address arithmetic also occur. A wrapping 32-bit interpretation is **[INFERENCE]** wherever signed expressions would otherwise overflow or require target-specific semantics. Address conclusions below that rely on this interpretation are conditional on it.

The late variable right shift has a signed 32-bit operand. Its count is the low 8 bits of the working base, not an explicitly restricted 0–31 count. Counts of 32 or greater are **documented_only**. For negative operands and in-range counts, the result is also **documented_only** without a target signed-right-shift convention. Unsigned shift rules do not resolve either issue.

### Descriptive address components

These names describe retained values, not established object fields.

| Term | Meaning |
|---|---|
| Shifted base | A dependency-result low word shifted left by 25. |
| Scaled base | Another dependency result multiplied by 33,554,432. |
| Shifted payload | A separate dependency result shifted left by 8. |
| Bounded index | A result shifted left by 8, masked to 15 bits, then shifted right logically by 6. |
| Indexed base | Twice the bounded index. |
| Early global base | The value read at address 0x05 and converted to an unsigned 32-bit address component. |
| Retained link | The word subsequently loaded from early global base plus 0x54. |
| Quadrupled link | Four times the retained link. |
| Adjusted component | Quadrupled link minus decimal 150. |
| Working base | Four times the adjusted component, retained as an unsigned word. |
| Auxiliary base | Twice a later pointer-like word loaded through a byte-derived address. |
| Repeated payload | Four times that same pointer-like word. |
| Retained global base | A later 32-bit value loaded from address 0x087fbbbc. |

Under 32-bit bit-vector arithmetic, the bounded index ranges from 0 to 508 in multiples of 4; the indexed base ranges from 0 to 1,016 in multiples of 8. No additional arena base is visibly added.

### Initial mutations and zero propagation

The initial mutation group connects the final wide dependency result to earlier retained components. Its ordered effects are:

- A word at the final high word plus 0x14 receives the shifted base.
- A byte at shifted base plus 9 receives the final high word’s low byte.
- A word at 256 times the final low word plus 0x54 receives the scaled base.
- The byte at shifted base plus 9 is overwritten with the shifted base’s low byte.

The word at scaled base plus 0x14 is then cleared. A byte from scaled base plus 5 is retained, the word at indexed base plus 0x14 is cleared, and the retained byte is zero-extended into the word at scaled base plus 0x34. The function next reads the word at scaled base plus 0x14.

**Under ordinary byte-addressed memory semantics, 32-bit address arithmetic, successful accesses, and no external mutation, this read yields zero.** The only intervening stores are:

- A zero-word store at indexed base plus 0x14. Even if it overlaps the cleared word, it cannot introduce nonzero bits.
- The byte-derived word store at scaled base plus 0x34. Its destination is 32 bytes beyond the cleared word’s start; the two four-byte ranges cannot overlap, including under 32-bit address wraparound.

Thus the represented intervening stores do not make this read potentially nonzero. A different result requires unresolved device or memory semantics, external mutation, or failure of the stated interpretation.

After the read, the indexed word clear is repeated. Two consecutive byte writes at indexed base plus 0x15 receive the read word’s low byte—therefore zero under the same assumptions.

### First fixed-address accesses

Immediately after those duplicate byte writes, the function reads address 0x05 and converts the value into the early global base. The original global declaration and physical read width are not supplied; conversion to 32 bits does not establish a 32-bit memory read.

A represented byte write at address 0x01 then receives the indexed base’s low byte. Subsequent computed mutations:

- Write the bounded index’s low byte at early global base plus 0x15.
- Write the shifted payload’s low byte at scaled base plus 0x11.
- Write the shifted payload as a word at the earlier zero-read value plus 0x54.
- Clear the byte at scaled base plus 0x11.

Under the ordinary-memory assumptions above, the shifted-payload word destination is therefore **address 0x54**. This is an earlier effect distinct from the later explicit word assignment at that address. Other intervening or subsequent writes may alias it; no lasting final value is implied.

Only afterward is the retained link loaded from early global base plus 0x54. That location is overwritten with the earlier zero-read value, followed by a word write of the bounded index at indexed base plus 0x34.

Under the stated assumptions, this overwrite stores zero. It does **not** prove that the retained link was zero: its load precedes the overwrite, and earlier writes may affect its source.

The early global base remains retained independently of later writes that overlap address 0x05.

### Linked updates and additional globals

The next region loads pointer-like words at offset 0x74, writes through them at offset 0x54, repeatedly writes the retained link into its own offset 0x24, and writes its low byte at another loaded base plus 9.

Within this region, fixed-address ordering is significant:

- The indexed word at 0x14 receives an earlier dependency result shifted left by 26.
- A represented 32-bit write at address 0x54 receives twice the retained link, expressed through a signed left shift.
- The indexed word at 0x14 receives the retained link twice.
- Two indexed byte writes at 0x15 receive the quadrupled link’s low byte.
- A represented byte write at address 0x15 receives the indexed base’s low byte.

The absolute 0x15 write and computed base-plus-0x15 writes are separate effects, whether or not they alias.

Further linked accesses use quadrupled and eight-times-retained-link components, loads at offsets 4, 0x64, and 0x74, and byte-derived address bases. Mutations include self-valued word writes and repeated byte writes at offsets 9, 0x0d, 0x14, 0x15, 0x34, 0x44, and 0x54.

Separate byte loads at offset 5 feed zero-extended word stores and later address formation. Intervening writes can affect these loads; they are not one invariant field value.

A pointer-like word reached through a byte-derived address is doubled to form the auxiliary base. The function separately writes 0x3a as a byte at four times that word, a halfword at eight times it, and a word at four times it plus 0x20.

At working base plus 0x20, successive word writes contain 0x20, 0x2f, a zero-extended byte freshly loaded from working base plus 4, and 0x20. These constants establish no string interpretation.

### Retained global base and repeated writes

After the final 0x20 write, the function retains the 32-bit global base read from address 0x087fbbbc. This read precedes the represented 32-bit write of decimal 32 at address 0x60 and the repeated-write region.

Later writes overlapping the global-base storage do not implicitly reload the retained value.

The initial repeated group has four ordered effects:

1. Repeated payload is written as a word at eight times the retained link plus 0x54.
2. Working base’s low byte is written at retained global base plus 0x19.
3. Retained global base’s low byte is written at working base plus 0x11.
4. Eight times the retained link is written as a word at retained global base plus 0x34.

Subsequent groups substitute eight times the adjusted component for eight times the retained link in the first destination and final stored word. The two byte destinations and values remain unchanged.

The representation contains consecutive repetitions without an advancing cursor or intervening direct loads within these groups. Exact cardinality is **documented_only** here. This omission does not authorize collapsing observable writes.

### Terminal reads and reload dependencies

After the repeated region, a represented 32-bit global read at address 0xa9 is retained. A word is then loaded from repeated payload plus 0x74. Next, address 0x3a receives a byte equal to the low 8 bits of eight times the adjusted component plus decimal 48.

Only after that byte write are words retained from auxiliary base plus 0x14 and working base plus 4. Moving the 0x3a write after these loads could change their values through aliasing.

Computed byte mutations then populate auxiliary base plus 0x0d and working base plus 8, 0x0d, and 9. Working base plus 8 receives 0x6c. The auxiliary word at 0x54 receives the auxiliary base itself.

Address 0x67 receives the low byte of the retained 0xa9 word. A word is loaded from auxiliary base plus 0x44; then a fresh byte read from 0x67 supplies the byte written to address 0x21. This transfer is not an unconditional second assignment from the retained word.

The loaded auxiliary word supplies a byte at working base plus 0x10 and a word at retained global base plus 0x54. Its signed variable right shift supplies a byte at auxiliary base plus 1. The global-derived word at 0x54 is subsequently overwritten with the working base.

A word loaded from working base plus 0x40 is later stored at working base plus 0x10. Between that load and store, retained global base plus 0x74 receives the earlier retained word from working base plus 4. Retained global base plus 0x44 then receives the auxiliary base.

Two independent later reads of byte address 0x67 each supply a write at working base plus 9. Between them, the function writes working base’s low byte at auxiliary base plus 9, writes the retained 0xa9 word at working base plus 0x44, and loads a destination base from working base plus 0x54. Both 0x67 reads remain fresh because computed writes may alias their source.

The earlier word from repeated payload plus 0x74 supplies low-byte writes at destination base plus 0x0d and auxiliary base plus 0x11. The retained word from auxiliary base plus 0x14 is restored to auxiliary base plus 0x54.

A fresh word from working base plus 0x14 supplies a later byte at destination base plus 0x11. Working base plus 9 receives destination base’s low byte; working base plus 0x44 receives that base as a word.

Finally, a word loaded from auxiliary base plus 4 supplies a halfword destination at twice its value, receiving 0x65. Loads at its offsets 0x64 and 0x74, and from auxiliary base plus 0x64, drive the remaining stores. These include decimal 100 at destination base plus 0x54, an offset-0x54 word write through the later loaded base, and a final word write at twice the value loaded at offset 0x64.

The terminal dependency receives:

1. Decimal 114.
2. The word loaded at offset 0x64 of the value obtained from auxiliary base plus 4.
3. The earlier retained word from auxiliary base plus 0x14.
4. The word subsequently loaded at offset 0x74 of the same base used for argument two.

No explicit cleanup or result check follows.

## Side effects

Represented fixed-address writes occur at 0x01, 0x15, 0x21, 0x3a, 0x54, 0x60, and 0x67. Under the ordinary-memory zero propagation established above, an additional earlier computed word write resolves to 0x54.

Direct reads include 0x05, 0x67, 0xa9, and 0x087fbbbc. These descriptions establish neither independent global objects nor complete declarations.

Computed-address aliasing and endianness can connect apparently separate accesses. A word at an address plus 0x14 overlaps the byte at that address plus 0x15.

No purity, idempotence, reentrancy, thread safety, transactionality, allocation policy, or release policy is established. Dependency-internal effects are unrestricted by the supplied evidence.

There is no visible rollback or error translation. Faulting accesses or nonreturning dependencies can leave earlier effects completed.

## Dependencies

These are document-local opaque references, not resolved identities.

| ID | Role | Established boundary |
|---|---|---|
| F-opaque-001 | Incoming-context-derived initial invocation | Target, implicit arguments, effects, and result convention are documented_only. |
| F-opaque-002 | Continuation-relative invocation family | Visible transformations and ordering; individual contracts are documented_only. |
| F-opaque-003 | Terminal indirect transfer | Four represented arguments; target and return behavior are documented_only. |

F-opaque-002 does not assert one common callee. No dependency is assumed to preserve globals, secondary result channels, or memory ownership, or to validate addresses.

## Acceptance examples

These are conditional acceptance examples, not executed tests.

- **Input extraction:** Third-input values 0, 1, and 1,023 supply zero to the extracted argument; 1,024 supplies 32; 0x000fffff supplies 0x7fe0. This concerns that argument only.

- **Unsigned byte propagation:** If the initial byte load at scaled base plus 5 returns 0xfe, its word propagation stores 254, not −2.

- **Cleared-word invariant:** With ordinary memory, successful 32-bit-addressed accesses, and no external mutation, the read from scaled base plus 0x14 returns zero even if the indexed zero-word store overlaps it. The word store at scaled base plus 0x34 cannot change that result.

- **Zero-derived effects:** Under those assumptions, both initial indexed byte writes at 0x15 store zero, the shifted-payload word destination resolves to address 0x54, and the post-load overwrite at early global base plus 0x54 stores zero. None implies a zero retained link.

- **Exceptional memory semantics:** A device-backed location or external mutation can invalidate the ordinary-memory zero result. Such a deviation is documented_only, not attributable to nonzero overlap from the represented intervening stores.

- **Retained base:** If later writes change storage at 0x087fbbbc, repeated groups still use the previously retained base.

- **Fresh global-byte reads:** If a computed write changes byte 0x67 before a later read, that read uses the applicable memory value rather than automatically reusing the retained 0xa9 word’s low byte.

- **Shift boundary:** Negative signed operands and oversized counts have no unconditional numeric acceptance result without target conventions. Masking the count to five bits is not authorized.

- **Partial failure:** A late fault requires neither rollback nor an invented error return.

## Uncertainties

The following remain **documented_only**:

- Business purpose, valid callers, ABI, implicit state, and secondary-result conventions.
- Individual dependency identities and complete per-site invocation trace.
- Exact repeated-group cardinality in this contract.
- Control-flow fidelity and terminal return behavior.
- Original global declarations, particularly the physical read width at 0x05.
- Endianness, alignment, address mapping, object extents, and alias relationships.
- Signed-overflow behavior, negative signed right shifts, and oversized shift counts.
- Device-memory behavior, external mutation, concurrency, ownership, and lifetimes.
- Dependency-internal effects.

The initial cleared-word value is not an unresolved ordinary-memory aliasing issue: the represented intervening stores preserve zero under the stated assumptions.

## Evidence

This contract derives from static inspection of the supplied representation. Prior prose, comments, and reviewer claims are not independent behavioral evidence.

Widths, argument transformations, retained-value dependencies, fixed-address expressions, separate reloads, and the terminal four-argument transfer support the established claims. The cleared-word conclusion additionally uses ordinary byte-addressed memory and 32-bit address arithmetic: the intervening zero store cannot introduce nonzero bits, and the other word destination is 32 bytes away.

No dependency bodies, authoritative caller contracts, global declarations, target memory map, ABI specification, or runtime observations were supplied. No runtime tests have been performed.
