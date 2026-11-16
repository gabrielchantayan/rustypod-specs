<!-- Trusted provenance: F-3c3bb88d43ef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3c3bb88d43ef

## Purpose

F-3c3bb88d43ef performs a long sequence of unresolved indirect invocations, transforms values supplied through their result channels, and then performs ordered, mixed-width memory updates and address chasing before a final indirect transfer.

No application-level purpose is established. Character-like constants and recurring offsets do not establish formatting, serialization, allocation, initialization, or maintenance of a particular object type.

This contract distinguishes:

- **Established static evidence:** explicitly represented accesses, widths, calculations, retained values, and sequencing.
- **Inferred intent:** possible interpretations, none of which are acceptance requirements.
- **documented_only:** unresolved behavior or detail not specified as an executable acceptance requirement.

Every later effect is conditional on preceding invocations and accesses completing. The supplied material does not establish that the entire path can execute successfully in an ordinary process address space.

## Interface

### Explicit inputs

| Input | Represented width | Established consumption |
|---|---:|---|
| First argument | 32 bits | No explicit consumption is shown. |
| Second argument | 32 bits | No explicit consumption is shown. |
| Third argument | Signed 32 bits | Shifted left by six positions for an early indirect invocation. |

Incomplete indirect-call interfaces prevent concluding that the first two arguments are irrelevant to the actual calling convention.

No explicit argument is established as a buffer, count, object pointer, string, or ownership-bearing handle.

### Implicit inputs

The operation also depends on:

- The incoming return-link value.
- Primary and secondary result channels from indirect invocations.
- Existing dynamically addressed memory.
- Fixed-address storage.
- A separately stored repetition anchor.
- The final indirect target.

The value associated with fixed storage at `0x05` is converted to an unsigned 32-bit word. Its declaration is absent. Its physical load width, signedness before conversion, and restricted range remain **documented_only**. Conversion to an unsigned word does not establish a byte load or a range of 0–255.

That access is distinct from explicitly unsigned byte reads at dynamic bases plus `0x05`.

### Outputs and completion

No explicit return value is represented. Outputs consist of dependency invocations, memory accesses and mutations, and the final indirect transfer.

The final target receives four represented argument words:

1. `0x72`.
2. A word loaded near completion from a terminally obtained base plus `0x64`.
3. A retained word read earlier from the chased base plus `0x14`.
4. A word loaded near completion from the terminally obtained base plus `0x74`.

An ordinary return follows the represented transfer. Whether the target actually returns, terminates execution, or transfers control elsewhere is **documented_only**.

### Preconditions

Successful completion requires every reached target and access to be valid in the actual environment: executable targets, usable result channels, readable and writable storage, and supported alignment.

Fixed low-address accesses must also be permitted. No represented validation enforces these requirements. Allocation sizes, object boundaries, and a sufficient entry-state predicate guaranteeing completion are not established.

## Behavior

### Indirect invocation phase

The first target is derived from the incoming return-link value plus `0xFFE`. Subsequent targets are generally derived from embedded code-related anchors plus that displacement. These relationships do not establish public dependency identities.

Three invocations precede the invocation consuming the shifted third argument. That invocation also consumes a secondary result word and that word shifted left by six.

The remaining chain propagates primary results, secondary results, and components of represented 64-bit results. Transformations include masks, left shifts, unsigned right shifts, explicit zeros, and repeated argument values. Retained results sometimes remain relevant across many intervening calls.

An invocation with no displayed arguments is not established as having a zero-argument interface. An ignored result does not make its invocation effect-free.

No explicit branch or loop is represented in this phase. Dependencies may nevertheless branch, fail, or never return. They must not be assumed to preserve globals, memory, ownership, or incompletely represented result channels.

### Width and arithmetic rules

A word is 32 bits, a halfword 16 bits, and a byte 8 bits where an access is explicitly typed accordingly.

For represented 64-bit results, the low word comprises bits 0–31 and the high word bits 32–63. Unsigned full-container shifts by 34, 36, and 44 correspond to logical shifts of the high word by 2, 4, and 12. They are not 32-bit shifts by those larger counts.

Masks select fields of widths including 11, 12, 13, 14, 17, 18, 19, 20, 21, 22, and 24 bits. Masking before shifting is consequential. In particular, selecting the low 24 bits and then shifting right by eight produces a 16-bit field.

Unsigned 32-bit calculations retain the low 32 bits. Discarded bits cannot be recovered by a subsequent right shift. Successive stored shifts can therefore eliminate a nonzero original value.

Several calculations instead use signed interpretations, including address scaling, subtraction, and shifts. Signed overflow, negative signed shifts, address wraparound, and target-specific shift behavior are not established as portable arithmetic. For example, multiplication by `0x02000000` retaining only seven input bits is conditional on 32-bit wrapping bit-pattern semantics.

Explicit byte and halfword stores retain eight and sixteen bits respectively. Explicit unsigned byte reads widened to words yield 0–255. This rule does not resolve the separate fixed-storage read at `0x05`.

Narrowing to plain character width is represented in several calculations. The signedness and out-of-range conversion semantics of that character interpretation require target confirmation.

### Address roles

The memory phase uses the following descriptive roles, not object types or source identifiers:

| Role | Established relationship |
|---|---|
| Shifted base | A retained signed result multiplied by `0x02000000`. |
| Paired base | Twice an extracted unsigned field obtained using a 15-bit mask and a right shift by six. |
| Tag base | A retained result shifted left by 25. |
| Fixed-derived base | The unsigned-word conversion of fixed storage at `0x05`. |
| Retained word | A word read from fixed-derived base plus `0x54` before that field is replaced. |
| Adjusted retained value | Four times the retained word, followed by subtraction of `0x96`. |
| Derived base | Four times the adjusted retained value, retained as an unsigned word. |
| Repetition anchor | A word loaded from separate global storage before the repeated-store region. |
| Chased base | Twice a word obtained through the later byte-mediated address chase. |
| Retained payload base | Four times the same chased word whose doubling produces the chased base. |

Signed-arithmetic qualifications apply to these scaling relationships. Derived addresses may alias one another and fixed-address storage.

### Opening memory effects

A word store through the high half of the final result pair precedes a byte store at tag base plus `0x09`. A word store through an address derived from the low half follows. The tag byte is then overwritten with the low byte of a retained shifted value.

The shifted-base word at `0x14` is cleared before an unsigned byte is read at shifted base plus `0x05`. A paired-base clearing store intervenes before that byte is widened and copied into the shifted-base word at `0x34`.

The cleared shifted-base word is subsequently read. Another paired-base clearing store and two duplicate byte stores at paired base plus `0x15` follow.

Only afterward is fixed storage at `0x05` consumed to obtain the fixed-derived base. It cannot be replaced by the earlier dynamic byte read: address description, type evidence, and timing differ.

Further effects include a byte-valued assignment at fixed `0x01`, a byte store at fixed-derived base plus `0x15`, byte updates at shifted base plus `0x11`, and word accesses at fixed-derived base plus `0x54`.

The word read from that last field becomes the retained word. Its value reflects all earlier overlapping writes at the actual read point, not necessarily entry-state memory.

### Address chasing and direct fixed-storage writes

The next region follows words from dynamic offsets including `0x04`, `0x44`, `0x54`, `0x64`, and `0x74`. Loaded values become addresses or components of scaled addresses. Retained values also supply repeated stores and byte payloads.

Two direct fixed-address assignments are part of this region:

- **Fixed `0x54`, 32 bits:** receives the retained word shifted left by one. This assignment occurs after a word store at paired base plus `0x14` carrying a shifted earlier result, and before the next word store at paired base plus `0x14` carrying the retained word. Another store of that retained word to the same dynamic destination follows. The fixed assignment is therefore between successive dynamic stores, not an initial or final-state-only effect.
- **Fixed `0x15`, 8 bits:** receives the paired base narrowed to eight bits. It follows the later pair of duplicate byte stores at paired base plus `0x15`; those duplicate stores carry the low byte of four times the retained word. The fixed assignment precedes the following scaled dynamic word updates.

The fixed `0x54` calculation retains the signed-left-shift qualifications. It is not an unconditional portable doubling guarantee for every signed operand.

Neither fixed write is interchangeable with a write at a dynamic base plus the same numerical offset. They may nevertheless alias dynamically addressed storage. Such aliasing affects later reads and the intermediate states between stores.

Later chasing includes unsigned byte reads used as bases for word reads at `0x74`. The same byte location is read again after intervening writes; a single entry-state read cannot replace these events.

### Layout and overlap

The represented access inventory includes:

| Width | Representative dynamic offsets |
|---|---|
| 8 bits | `0x01`, `0x05`, `0x08`, `0x09`, `0x0D`, `0x10`, `0x11`, `0x15`, `0x19` |
| 16 bits | Scaled addresses without an additional field offset |
| 32 bits | `0x04`, `0x10`, `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, `0x74` |

This inventory is not a structure declaration. It establishes neither disjoint fields nor object ownership.

A word at `0x14` overlaps a byte at `0x15`; a word at `0x10` overlaps a byte at `0x11`. Endianness determines their numerical interaction. Additional overlap depends on the calculated addresses, including overlap with fixed `0x15` and fixed `0x54`.

Duplicate stores remain distinct represented effects. Reads after writes must use memory at their actual sequence positions. Where one expression contains multiple reads without independently established evaluation order, that finer ordering remains **documented_only**.

### Constants and repeated stores

A later region stores `0x3A` at byte, halfword, and word widths to separately scaled destinations.

Another word destination then receives, in order, `0x20`, `0x2F`, a freshly read unsigned byte widened to a word, and `0x20`. These are memory effects, not established textual output.

The repetition anchor is loaded next. Fixed storage at `0x60` receives the represented word value `0x20`.

The repeated region consists of ordered four-store groups:

- A word store at a scaled address plus `0x54`.
- A byte store at repetition anchor plus `0x19`, carrying the low byte of the derived base.
- A byte store at derived base plus `0x11`, carrying the low byte of the repetition anchor.
- A word store at repetition anchor plus `0x34`.

The first group uses eight times the retained word for its scaled destination and its anchor-relative word payload. Subsequent groups use eight times the adjusted retained value instead. The word payload stored at the scaled destination is the retained payload base.

The repetition anchor and byte payloads are retained across the groups; they are not freshly loaded for each group. Equal-value writes remain separate represented effects. Exact group count is **documented_only in this contract**, with no exact-count acceptance claim.

### Terminal fixed `0x3A` assignment

The terminal region first reads a represented word from fixed `0xA9`, then a dynamic word at retained payload base plus `0x74`.

**After that dynamic read, fixed `0x3A` receives a byte-valued calculation derived from the adjusted retained value:** the adjusted retained value is first narrowed to plain character width, that narrowed value is multiplied by eight, and `0x30` is added before assignment.

The narrowing precedes multiplication. It is not a calculation from the derived base, nor is it merely a constant assignment. Character signedness, out-of-range narrowing, and destination conversion retain the arithmetic qualifications above.

This assignment precedes both the retained word read at chased base plus `0x14` and the word read at derived base plus `0x04`. Consequently:

- A fault in the retained-payload-base `0x74` read prevents reaching the fixed assignment.
- A fault in either later retained read does not undo the fixed assignment if it completed.
- If fixed `0x3A` overlaps either later dynamic read, that read observes the resulting memory state.

### Terminal fixed `0x67` reads

After the retained terminal reads and several dynamic updates, fixed `0x67` receives the low byte of the retained `0xA9` word.

Three subsequent consumptions of `0x67` are distinct read events:

1. **For fixed `0x21`:** a dynamic word read at chased base plus `0x44` intervenes between the assignment at `0x67` and the read supplying `0x21`.
2. **For the first later derived-base `0x09` byte store:** intervening effects include a byte store at derived base plus `0x10`, anchor-relative word stores, a variable-shift-derived byte store at chased base plus `0x01`, and a word read at derived base plus `0x40`. Stores then update anchor plus `0x74`, derived base plus `0x10`, and anchor plus `0x44` before `0x67` is read again.
3. **For the next derived-base `0x09` byte store:** a byte store at chased base plus `0x09`, a word store of the retained `0xA9` value at derived base plus `0x44`, and a word read at derived base plus `0x54` intervene before another fresh read of `0x67`.

Intervening writes can overlap `0x67`. The three consumed values therefore need not equal the initial assigned byte or one another.

The full retained `0xA9` word remains separately available for its later word store. Retaining it does not eliminate the fresh fixed-storage reads.

### Remaining terminal effects

The chased-base word read at `0x44` supplies a signed right shift. Its count is the low byte of the derived base. Negative operands and counts outside 0–31 require target confirmation; logical shifting or masked counts must not be assumed.

The word read at derived base plus `0x54` subsequently supplies both an address and replacement payloads, including the derived-base byte at `0x09` and word at `0x44`.

Near completion, a word loaded at chased base plus `0x04` becomes the terminally obtained base. A halfword value `0x65` is stored at twice that base. Further effects include:

- A word store of decimal 100 at a dynamically obtained destination plus `0x54`.
- A loaded-word store at another destination plus `0x54`.
- A word store at twice another loaded address.

The final four-argument indirect transfer follows these effects.

## Side effects

The function mutates dynamic and fixed-address storage. Direct fixed writes include byte-valued assignments at `0x01`, `0x15`, `0x21`, `0x3A`, and `0x67`, and represented word assignments at `0x54` and `0x60`.

Fixed and dynamic destinations are not guaranteed disjoint. Intermediate writes, duplicate writes, and fresh reads are part of the state contract even when eventual memory contents coincide.

No atomicity, rollback, locking, cleanup, restoration, or transactional publication is established. A fault or nonreturning dependency can leave a completed prefix of effects visible.

Represented ordering does not establish hardware barriers or a language-level concurrency model.

No explicit allocation, release, reference counting, ownership transfer, or lifetime extension is established. Unknown dependencies may perform any of these operations.

No explicit status result, invalid-address check, error-code assignment, or exception handler is represented. Recovery from invalid targets, inaccessible storage, unsupported alignment, or unexpected dependency behavior is unresolved.

## Dependencies

F-3c3bb88d43ef is the only supplied opaque function identity. No evidence-backed opaque IDs are supplied for the indirect targets; none are invented here.

| Dependency class | Established relationship | Unresolved contract |
|---|---|---|
| Initial target | Incoming return-link value plus `0xFFE` | Identity, full interface, effects, termination |
| Later targets | Embedded anchors plus `0xFFE` | Identities, full interfaces, effects |
| Result channels | Values consumed by transformations and accesses | Calling-convention meaning and validity |
| Final target | Receives four represented argument words | Identity, ownership effects, return behavior |

No dependency is established as pure, memory-preserving, ownership-neutral, allocation-free, or guaranteed to return.

## Acceptance examples

These are conditional static criteria, not runtime test results.

1. **Third argument:** if it is 1 and preceding invocations return normally, the early consuming invocation receives 64 in its first represented argument position.

2. **Masked extraction:** selecting the low 24 bits of `0xAB123456` and shifting right by eight yields `0x1234`.

3. **Full-container extraction:** unsigned `0x123456789ABCDEF0` shifted right by 36 yields `0x01234567`.

4. **Dynamic byte widening:** a byte value `0xFE` read at shifted base plus `0x05` produces a word copy containing 254, not a sign-extended negative value.

5. **Fixed `0x05`:** without declaration evidence, rejecting its converted value solely because it exceeds 255 is unsupported.

6. **Fixed `0x54` ordering:** if paired base is `0x40` and the retained word is `0x1234`, the fixed word assignment places `0x2468` at `0x54` between the adjacent dynamic stores that also address `0x54`. Later stores of `0x1234` do not erase that intermediate effect.

7. **Fixed `0x15` narrowing:** if paired base is `0x108`, the direct fixed byte assignment carries `0x08`. It occurs after both later duplicate byte stores at paired base plus `0x15`.

8. **Fixed `0x3A` calculation:** if the adjusted retained value is `0x12`, narrowing is unambiguous and the assigned byte is `0xC0`. The retained-payload-base `0x74` read precedes this assignment; the chased-base `0x14` and derived-base `0x04` reads follow it.

9. **Terminal fault timing:** if the chased-base `0x44` read faults after fixed `0x67` is assigned, the subsequent assignment at fixed `0x21` is not reached.

10. **Fresh fixed reads:** if aliasing writes make the three consumed `0x67` values `0x31`, `0x42`, and `0x53`, fixed `0x21` and the two later derived-base `0x09` stores consume those respective values. Caching one value is incompatible.

11. **Intermediate stores:** where accesses are observable, the word stores of `0x20`, `0x2F`, the freshly byte-derived word, and `0x20` remain distinct and ordered.

12. **Completion:** if all preceding effects complete, the terminal target receives `0x72` first and the retained chased-base `0x14` word third. Ordinary return remains conditional on that target returning.

## Uncertainties

- **documented_only — purpose:** application semantics.
- **documented_only — interface:** complete incoming state and calling convention.
- **documented_only — dependencies:** target identities, effects, ownership, and termination.
- **documented_only — fixed `0x05`:** declaration, load width, signedness, and restricted range.
- **documented_only — arithmetic:** signed overflow, character conversion, signed right shifts, oversized counts, and address wraparound.
- **documented_only — layout:** allocation boundaries, endianness, and alignment.
- **documented_only — expression ordering:** finer ordering of multiple reads within individual expressions.
- **documented_only — repetition:** exact group count is not specified here.
- **documented_only — environment:** validity of fixed low-address and unusual derived-address accesses.
- **documented_only — concurrency:** synchronization and hardware visibility beyond represented sequencing.

## Evidence

The evidence is the supplied static body, evaluated independently of comments, prior prose, and reviewer assertions. No implementation access, independent instruction inspection, or runtime tests have been performed.

Explicit assignments establish the fixed 32-bit write at `0x54` between paired-base word stores and the fixed 8-bit write at `0x15` after the later duplicate paired-base byte stores.

The terminal calculation establishes character-width narrowing of the adjusted retained value before multiplication by eight and addition of `0x30`. Its placement establishes the preceding retained-payload-base `0x74` read and the following chased-base `0x14` and derived-base `0x04` reads.

Separate fixed-storage consumptions establish fresh-read timing for `0x67`. Conversely, the unsigned conversion associated with fixed `0x05` does not independently establish its physical load width or byte range.
