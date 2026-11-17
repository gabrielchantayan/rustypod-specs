<!-- Trusted provenance: F-440ad4d2aa2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-440ad4d2aa2d

## Purpose

F-440ad4d2aa2d executes an extensive ordered sequence of indirect dependency invocations, transforms their results, performs direct memory reads and writes, and finishes with a four-argument indirect invocation.

No application-level purpose is established. Numeric constants, scaled addresses, and self-referential writes do not establish text processing, initialization, serialization, or a particular data structure.

This contract distinguishes:

- **Established static evidence:** operations, widths, operands, and ordering visible in the supplied representation.
- **[INFERENCE]:** conclusions conditional on an explicitly stated arithmetic or memory model.
- **documented_only:** behavior whose complete contract is unresolved or not enumerated here.

All acceptance examples are conditional static expectations. No runtime tests have been performed.

## Interface

### Explicit inputs

| Argument position | Represented width | Established use |
|---|---:|---|
| First | 32 bits | No explicit consumption in the represented body. |
| Second | 32 bits | No explicit consumption in the represented body. |
| Third | Signed 32 bits | Supplies shifted operands to early indirect invocations. |

The third argument supplies its value shifted left by three bits. That transformed value is retained and later shifted by another ten or twenty-six bits. No range check or overflow check is represented.

The absence of explicit uses of the first two arguments does not establish independence from their incoming machine state: the initial indirect invocation and unresolved calling conventions prevent that stronger conclusion.

### Implicit inputs

Execution also depends on:

- An incoming return-link value.
- Indirect invocation targets derived from that value or embedded location values.
- Primary and secondary dependency results.
- Memory reached through derived addresses.
- Fixed-address storage and an additional fixed global word supplying a later address base.
- An unresolved terminal invocation target.

The explicit arguments alone do not determine execution.

### Output and completion

No explicit return value, status code, or output-buffer parameter is represented.

If every preceding operation completes and the terminal invocation returns, this routine returns without an explicit result. Nonreturning behavior, faults, exceptions, and other control transfers from dependencies remain documented_only.

### Preconditions and ownership

Completion requires valid reached invocation targets and an environment supporting every reached memory access, including low fixed addresses and potentially unaligned accesses. The routine does not visibly validate these conditions.

No allocation, release, ownership transfer, lifetime guarantee, or reference-count protocol is established. An integer subsequently used as an address is not thereby proven to identify an allocation.

## Behavior

### Execution structure

The represented body is straight-line: a dependency-heavy computation phase, a direct-memory phase containing extensive repeated stores, and a terminal invocation. No explicit conditional branch or input-controlled loop is represented.

This ordering describes the visible path, not guaranteed completion. A dependency that does not return prevents later operations on that path.

### Dependency-driven computation

The first indirect target is derived from the incoming return-link value by adding `0xFFE`. Many later targets use embedded location values with the same addition. Their identities and complete signatures are unresolved.

An early represented 64-bit result is split into independently used 32-bit halves. Its lower half supplies an eight-bit left shift; its upper half supplies a fourteen-bit left shift. An early invocation receives the latter together with the third explicit argument shifted left by three bits.

Later computation repeatedly uses:

- Eight- and fifteen-bit left shifts of 32-bit values.
- The same transformed operand in multiple argument positions.
- Upper and lower halves of represented 64-bit results.
- Logical right shifts of unsigned quantities.
- Masked bit-field extraction.
- Explicit zeros and transformations of values previously assigned zero.
- Secondary invocation results whose production conventions are unresolved.

Invocations with no displayed arguments are not established to be pure, effect-free, or independent of incoming machine state. Apparent secondary results must not be replaced by zero.

The complete invocation-occurrence manifest and every intermediate operand relationship are documented_only in this document. No permission to omit, merge, reorder, cache, or parallelize invocations follows from that omission.

### Arithmetic and widths

The representation distinguishes 8-, 16-, 32-, and 64-bit quantities. A byte write retains eight bits; a halfword write retains sixteen bits. Explicit unsigned byte reads widen to values from 0 through 255.

Full-width unsigned 64-bit right shifts by 34, 36, and 44 positions yield at most 30, 28, and 20 significant bits respectively. These must not be replaced by shifts applied only to a lower 32-bit half.

Representative masked extractions have these bounds:

| Extraction | Result width at most |
|---|---:|
| Low 20 bits, shifted right by 10 | 10 bits |
| Low 24 bits, shifted right by 20 | 4 bits |
| Low 24 bits, shifted right by 14 | 10 bits |
| Low 24 bits, shifted right by 8 | 16 bits |
| Low 18 bits, shifted right by 11 | 7 bits |
| Low 17 bits, shifted right by 4 | 13 bits |

Signed left shifts, signed multiplications, address arithmetic, and a later variable-count signed right shift require a target arithmetic model. The displayed signed types alone do not establish wrapping behavior.

**[INFERENCE]** Under a 32-bit wrapping bit-vector model, each intermediate 32-bit operation discards bits beyond that width. A value shifted left by 25 and then by seven becomes zero. Widening intermediates to preserve discarded bits would change this conditional behavior.

### Direct-memory roles

The following descriptive roles identify retained values, not source identifiers or recovered structure fields:

| Role | Meaning |
|---|---|
| **Multiplication seed** | Signed 32-bit dependency result used immediately before the direct-memory phase. |
| **Multiplication base** | Signed 32-bit product of the multiplication seed and `0x02000000`. |
| **Bounded index** | Another result shifted left by eight, masked with `0x7FFF`, then logically shifted right by six. |
| **Index base** | Twice the bounded index. |
| **Pending value** | A retained dependency result previously shifted left by eight. |
| **Link token** | A retained 32-bit shifted dependency value used in the first stores. |
| **First global base** | Widened value read from fixed storage at `0x05`. |
| **Retained root** | Word read from offset `0x54` of the first global base before that location is overwritten. |

The multiplication base is established as a **signed multiplication**, not as an unconditional left shift.

**[INFERENCE]** Only under the wrapping bit-vector model is that product equivalent to retaining the low 32 bits of a 25-bit left shift.

Under that same model, the bounded index is a multiple of four between 0 and 508 inclusive. The mask and right shift alone permit a broader upper bound of 511, but the preceding eight-bit shift forces the result’s low two bits to zero.

### Initial direct accesses and first fixed-global interaction

The last pre-memory invocation supplies a represented 64-bit result. A word at its upper-half-derived address plus `0x14` receives the link token. A byte at the link token plus `0x09` then receives the low byte of that upper half. A word at the lower half multiplied by `0x100`, plus `0x54`, receives the multiplication base. The earlier byte destination is then overwritten with the low byte of the link token.

The multiplication-base word at `0x14` is cleared before its byte at `0x05` is read. A clear at index-base offset `0x14` intervenes before the widened byte is written to multiplication-base offset `0x34`. The multiplication-base word at `0x14` is then read and retained as the **cleared-field snapshot**.

A second index-base clear precedes two byte writes at index-base offset `0x15`, both using the low byte of that snapshot. Those writes remain distinct events.

Next, fixed storage at `0x05` is read and widened to obtain the first global base, **before** the byte write at fixed address `0x01`. That write uses the low byte of the index base. The supplied material does not independently establish the declared storage width at `0x05`; its complete read-width contract is documented_only.

Following that fixed read/write pair:

- First-global-base offset `0x15` receives the low byte of the bounded index.
- Multiplication-base offset `0x11` receives the low byte of the pending value.
- Cleared-field-snapshot offset `0x54` receives the pending value as a word.
- Multiplication-base offset `0x11` is cleared.
- First-global-base offset `0x54` is read to obtain the retained root, then overwritten with the cleared-field snapshot.

These are ordered interactions, not independent initialization assignments. Derived addresses may overlap fixed storage or one another.

### Linking-like updates and remaining early globals

Further accesses copy words, follow values read at offset `0x74`, write self-referential values, and repeatedly overwrite byte and word locations.

Two important fixed-global writes occur within this activity:

- After an index-base word write at `0x14` using the multiplication seed shifted left by 26, fixed address `0x54` receives a **32-bit word** containing the retained root shifted left by one. Only afterward does index-base offset `0x14` receive the retained root, twice.
- After two byte writes at index-base offset `0x15` using the low byte of four times the retained root, fixed address `0x15` receives the low byte of the index base. Subsequent root-derived word writes follow this fixed-byte write.

The routine then uses four and eight times the retained root in derived addresses and stored values. It follows byte-derived bases as well as word-derived bases. Byte reads at offset `0x05` can therefore select further word reads at offset `0x74`.

These updates resemble links between records **[INFERENCE]**. They establish no list, tree, graph, allocation, or ownership invariant. The complete intervening access manifest is documented_only here; the fixed-global ordering above does not imply adjacency where other accesses are described as intervening.

### Multiwidth writes and repeated region

For this region, the following roles apply:

- **Selected base:** a later word obtained through a byte-derived address at offset `0x74`.
- **Doubled selected base:** twice the selected base.
- **Adjusted base:** four times the retained root, minus `0x96`.
- **Working base:** four times the adjusted base.
- **Later global base:** a signed 32-bit word read from separate fixed global storage.

The selected base supplies three destinations receiving `0x3A`: a byte at four times that base, a halfword at eight times it, and a word at four times it plus `0x20`.

Working-base offset `0x20` receives four ordered word stores: `0x20`, `0x2F`, the zero-extended byte freshly read at working-base offset `0x04`, and `0x20` again. The byte read occurs after the first two stores, not before them.

The later global base is then read. **After that read and before the repeated region**, fixed address `0x60` receives the 32-bit word `0x20`.

The initial repeated group writes:

- Four times the selected base to a word at eight times the retained root plus `0x54`.
- The low byte of the working base to later-global-base offset `0x19`.
- The low byte of the later global base to working-base offset `0x11`.
- Eight times the retained root to later-global-base offset `0x34`.

Subsequent groups retain the same four-event order and byte operands, but use eight times the adjusted base for both the first destination base and the final stored word. The first word operand remains four times the selected base.

These are finite, explicitly repeated operations. Exact repetition count is documented_only in this contract, not an input-dependent or unbounded loop. Identical repeated writes are not equivalent to one write under an observation model recording access events.

### Final updates and invocation

The transition out of the repeated region has a material read/write interleaving:

1. Read the 32-bit word at fixed address `0xA9`; retain the **fixed-word snapshot**.
2. Read the word at four times the selected base plus `0x74`; retain the **selected-link snapshot**.
3. Write fixed byte `0x3A` using the narrowed result of eight times the adjusted base plus `0x30`.
4. Read doubled-selected-base offset `0x14`; retain the **retained-field snapshot**.
5. Read working-base offset `0x04`; retain the **working-field snapshot**.

The last two reads must not be moved ahead of the `0x3A` write: either derived location may overlap that fixed byte.

Subsequent byte updates use the low bytes of the working base and later global base. Working-base offset `0x08` receives `0x6C`; offsets `0x0D` and `0x09` receive the low byte of the later global base. Doubled-selected-base offset `0x54` receives its own base as a word.

Fixed byte `0x67` then receives the low byte of the fixed-word snapshot. This is the **initial write** to that location, not a retained-byte guarantee.

The next global interaction is explicitly interleaved:

- Doubled-selected-base offset `0x44` is read as a signed word.
- Fixed byte `0x67` is read, and that read supplies the byte written at fixed address `0x21`.
- The previously read signed word supplies a byte at working-base offset `0x10`, then a word at later-global-base offset `0x54`.
- Its signed right shift supplies a byte at doubled-selected-base offset `0x01`; the count is the working base’s low eight bits.
- Later-global-base offset `0x54` is overwritten with the working base.

The read at offset `0x44` may itself be environment-sensitive. Later derived stores may also overlap `0x67`; subsequent uses must therefore remain fresh reads.

A word read from working-base offset `0x40` supplies a later word write at working-base offset `0x10`. Before that write, later-global-base offset `0x74` receives the working-field snapshot. Later-global-base offset `0x44` then receives the doubled selected base.

Working-base offset `0x09` receives a fresh read of fixed byte `0x67`. After further derived writes, including the fixed-word snapshot at working-base offset `0x44`, working-base offset `0x54` is read to obtain the **final derived base**. Another fresh read of `0x67` again supplies a byte store at working-base offset `0x09`. Neither use may automatically reuse the initially stored byte.

The remaining updates include:

- Bytes from the selected-link snapshot written at final-derived-base offset `0x0D` and doubled-selected-base offset `0x11`.
- The retained-field snapshot written at doubled-selected-base offset `0x54`.
- A fresh word read from working-base offset `0x14`, whose low byte later supplies final-derived-base offset `0x11`.
- Overwrites of working-base offset `0x09` and its word at `0x44` using the final derived base.

A word read from doubled-selected-base offset `0x04` supplies the **terminal object base**. A halfword at twice that base receives `0x65` before its word at `0x64` is read for the second terminal argument. Intervening stores place the previously described low byte at final-derived-base offset `0x11` and decimal 100 at its offset `0x54`.

The fourth terminal argument is then read at terminal-object-base offset `0x74`. A later word read at doubled-selected-base offset `0x64` supplies the **terminal link base**.

Before invocation, a byte at doubled-selected-base offset `0x09` receives the low byte of the terminal object base. Terminal-link-base offset `0x54` receives the second argument as a word, and a word at twice the second argument receives the terminal link base.

The terminal invocation receives four represented 32-bit arguments:

| Position | Value |
|---|---|
| First | `0x72` |
| Second | Retained word read at terminal-object-base offset `0x64` |
| Third | Retained-field snapshot |
| Fourth | Retained word read at terminal-object-base offset `0x74` |

## Side effects

Direct effects include byte, halfword, and word accesses; overlapping-width updates; fixed-global interactions; address-following reads; and repeated stores.

Observed offsets include byte offsets `0x01`, `0x05`, `0x08`, `0x09`, `0x0D`, `0x10`, `0x11`, `0x15`, and `0x19`, and word offsets through `0x74`. These form an access map, not an established structure layout. A byte at `0x15` overlaps the word beginning at `0x14`; a byte at `0x11` overlaps the word beginning at `0x10`.

Distinct descriptive bases are not assumed disjoint. Fixed storage is not assumed isolated from derived accesses.

No rollback, synchronization, atomic transaction, bounds enforcement, or cleanup path is represented. Earlier writes are not visibly restored if execution stops.

Dependency side effects are additional and unknown. No dependency is assumed to preserve globals, allocate or release memory, or maintain ownership.

## Dependencies

The supplied map establishes no additional dependency identities.

| Opaque role ID | Role | Status |
|---|---|---|
| F-000000000001 | Ordered pre-memory indirect invocation occurrences | Targets, signatures, secondary results, effects, and errors documented_only. |
| F-000000000002 | Terminal indirect invocation | Four argument positions established; target identity, effects, errors, and return behavior documented_only. |

These are document-local role identifiers. F-000000000001 does not assert a shared target among occurrences.

## Acceptance examples

### Multiplication semantics

If the multiplication seed is 1, its represented product with `0x02000000` is `0x02000000`. If it is 64, the mathematical product exceeds the signed 32-bit positive range. Only under the wrapping model is the resulting bit pattern `0x80000000`; unconditional signed-overflow behavior is not specified.

### Bounded-index boundary

Under 32-bit bit-vector arithmetic, a relevant result whose low seven bits are `0x7F` produces index 508, not 511.

### Fixed-write/read interleaving

If working-base offset `0x04` overlaps fixed byte `0x3A`, its retained word read occurs after the `0x3A` write. The earlier selected-link read occurs before that write. Exact word values additionally require the overlap position, endianness, and memory model.

### Fresh fixed-byte reads

If an intervening derived store changes fixed byte `0x67` after its initial write, a later read supplying working-base offset `0x09` uses the then-current byte. It does not necessarily use the low byte of the fixed-word snapshot.

### Read after clear

A cleared word reads as zero only if no intervening aliasing or environment-specific effect changes its bytes. An overlapping byte write invalidates an unconditional zero expectation.

### Ordered overwrites

A store-event observer sees all four working-base offset `0x20` writes in order, including the widened-byte value. Final-state equality does not justify eliminating them.

### Completion

If F-000000000001 fails to return at any occurrence, later direct operations are not reached along the represented path. If all preceding operations complete and F-000000000002 returns, the routine returns without an explicit result.

## Uncertainties

The following remain documented_only:

- Application intent and object model.
- Dependency identities, full signatures, and secondary-result conventions.
- Complete invocation and intervening-access manifests not enumerated here.
- Exact repeated-region count.
- Declared read width of fixed storage at `0x05`.
- Endianness, alignment, address mapping, and volatile or device-memory behavior.
- Signed overflow, address overflow, and variable-count shift semantics, especially counts outside 0–31.
- Aliasing, allocation bounds, ownership, and lifetimes.
- Concurrency, interruption, reentrancy, faults, and error reporting.
- Terminal-target identity and return behavior.

These gaps do not authorize invented validation, fallback values, ownership rules, or error returns.

## Evidence

This contract uses only the supplied static representation as behavioral evidence. Prior prose, comments, and reviewer feedback are not independent proof.

Static evidence establishes the signed multiplication forming the multiplication base; the fixed-global access operands and ordering described above; fresh reads of fixed byte `0x67`; the post-repetition `0xA9` read, derived `0x74` read, `0x3A` write, and subsequent retained reads; and the terminal four-argument invocation.

Bit-vector deductions are explicitly conditional, not runtime observations. No runtime tests have been performed.
