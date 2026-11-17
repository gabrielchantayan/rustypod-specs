<!-- Trusted provenance: F-0b0e7f17cbae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0b0e7f17cbae

## Purpose

F-0b0e7f17cbae performs an ordered prefix of indirect dependency invocations, followed by mixed-width computed-memory operations, assignments involving undeclared globals, and a final unresolved four-argument invocation.

No application-level purpose is established. Pointer-like chains do not establish a container layout or ownership model. Numeric constants compatible with character encodings do not establish textual output.

This contract distinguishes:

- **Established static evidence:** explicit operations, represented value widths and conversions, ordering, and retained-value relationships.
- **[INFERENCE]:** consequences requiring additional arithmetic or environmental assumptions.
- **documented_only:** unresolved semantics or trace details not fully specified here.

Every effect is conditional on execution reaching it. Normal completion is not guaranteed.

## Interface

### Explicit inputs and result

The represented interface accepts four positional inputs and declares no result.

| Input | Represented width | Explicit use |
|---|---:|---|
| First | 32 bits | First argument of the initial dependency invocation |
| Second | 32 bits | No explicit body use |
| Third | 32 bits | No explicit body use |
| Fourth | 32 bits | Left-shifted by 25 for the initial invocation’s second argument |

The absence of explicit uses of the second and third inputs does not establish independence from incoming machine state under an unresolved calling convention.

No scalar result, status code, error object, or designated output buffer is established. Observable outputs consist of dependency interactions and memory effects.

### Implicit inputs

Execution also depends on incoming return-link state, an incoming general-purpose register value, implicit dependency outputs, computed-memory contents, several undeclared globals, and an unresolved final invocation destination.

The declarations of **all referenced globals** are absent. This applies to the globals associated with 0x01, 0x05, 0x15, 0x21, 0x3a, 0x54, 0x60, 0x67, and 0xa9, and to the separate global supplying the repeated region’s base.

Consequently:

- Their underlying access widths, signedness, and destination assignment conversions are **documented_only**.
- An explicitly narrowed assignment value does not establish a narrow destination access.
- Retaining a global’s value in a 32-bit quantity does not establish a 32-bit source load.
- Address-associated labels do not establish object sizes, access footprints, or disjointness.

Computed dereferences with explicit widths are specified separately.

### Preconditions, errors, and ownership

Each reached computed-memory access requires a valid mapping supporting its represented width and the environment’s alignment rules. Global accesses additionally require the missing declarations’ actual access footprints to be valid. Invocation destinations must be executable under the applicable ABI.

No explicit validation, bounds checking, recovery branch, rollback, or failure-status construction appears. A failure may leave preceding dependency effects and memory mutations intact.

The explicit body establishes no allocation, release, or ownership transfer. Unknown dependencies may retain references, alter shared state, allocate or release resources, or fail to return. They are not assumed to preserve globals, reachable memory, or resource lifetimes.

## Behavior

### Dependency prefix

Before the first explicit computed-memory store, the function performs a long straight-line sequence of indirect invocations.

The initial destination is derived from incoming return-link state plus 4094 address units. Subsequent destinations are derived from represented code-location values using the same displacement. These calculations do not resolve dependency identities or executable mappings.

The initial invocation receives the first input and the fourth input shifted left by 25. The next receives the preceding result’s low 32 bits shifted left by 8 and the implicit incoming register value shifted left by 28.

Later invocations consume transformed results, saved results spanning intervening calls, and implicit register outputs. Argument lists contain zero through four explicit arguments. An empty explicit list does not establish absence of implicit inputs.

Results include represented 32-bit quantities, 64-bit quantities whose halves are used separately, and additional register outputs. Transformations include masking, left shifts, unsigned right shifts, and extraction from high portions.

Invocation order is material. Unused ordinary results and zero-valued arguments do not make calls removable: dependencies may produce effects or fail to return. No loop or conditional branch is represented in this prefix. Its complete per-invocation trace and exact count remain **documented_only** in this contract; the prefix is not an interchangeable collection of calls.

### Widths, arithmetic, and layouts

Explicit computed-memory accesses distinguish 8-bit bytes, 16-bit halfwords, and 32-bit words. Explicit unsigned-byte samples widened to words are zero-extended.

**[INFERENCE]:** Numerical examples below use a 32-bit modular model for word arithmetic and left shifts. Explicit byte and halfword destinations retain the low 8 and 16 bits, respectively. This model does not establish portable semantics for signed-looking overflow, negative shifts, or excessive shift counts.

Unsigned 64-bit right shifts by 34, 36, and 44 expose the high word shifted right by 2, 4, and 12. Masking to 24 bits before shifting right by 20 leaves at most four significant bits.

Offsets identify access relationships, not recovered fields or object boundaries. Bases may overlap each other or global storage. Endianness and alignment behavior are unresolved.

The preceding width rules do **not** assign widths to undeclared globals. Global assignment values can undergo further conversion according to their missing destination declarations.

### Initial computed-memory effects

Three retained prefix values establish initial bases:

- **Shifted base:** a dependency result shifted left by 25.
- **Scaled base:** another dependency result multiplied by \(2^{25}\).
- **Small index:** a saved result shifted left by 8, masked to 15 bits, then unsigned-shifted right by 6.

**[INFERENCE]:** Under modular arithmetic, the small index is a multiple of four from 0 through 508. Its doubled value is a multiple of eight from 0 through 1016. Zero is not explicitly rejected.

The final 64-bit prefix result supplies two initial destination components. Its high word plus 0x14 receives a 32-bit store of the shifted base. The high word’s low byte is then stored at shifted-base offset 9. A 32-bit store at 256 times the low word plus 0x54 receives the scaled base. Shifted-base offset 9 is subsequently overwritten with the shifted base’s low byte.

Scaled-base offset 0x14 is cleared by a word store before an unsigned byte is sampled at offset 5. Doubled-index offset 0x14 is cleared next; the sampled byte is then zero-extended into the word at scaled-base offset 0x34.

A word is read from scaled-base offset 0x14. Another doubled-index word clear and two consecutive byte stores at doubled-index offset 0x15 follow. These byte stores use the low byte of the retained read.

The operations are ordered effects, not final-state assignments. Offset 0x15 can overlap the word beginning at 0x14.

### Early global evaluation and chained state

After those initial effects, the global associated with 0x05 is evaluated and explicitly converted to unsigned 32 bits. The retained result is the **early global value**. Its source load width and signedness remain unresolved; it is not established as an unsigned-byte sample.

Only afterward is the global associated with 0x01 assigned a value explicitly narrowed to 8 bits from the doubled index. That narrowing establishes the assignment’s supplied value, not the destination object’s access width or full stored representation.

The early global value supplies bases for a computed byte store at offset 0x15 and a computed word read and replacement at offset 0x54. These accesses do not reevaluate the global at 0x05.

The old word from early-global-value offset 0x54 is retained as the **saved word** before replacement. Its original location may subsequently change without refreshing that retained value.

The middle phase includes:

- Word reads at offsets 0x64 and 0x74 whose values become subsequent bases.
- An explicit unsigned-byte sample at offset 5 used as the numeric base of a word read at offset 0x74.
- Self-referential word stores at offsets 0x14, 0x24, and 0x54.
- Repeated computed byte stores at offsets 9 and 0x15.
- Assignment to the global associated with 0x54 of the saved word shifted left by one.
- A later assignment to the global associated with 0x15 of the doubled index explicitly narrowed to 8 bits.

Neither global assignment establishes its destination access width. The shift feeding the 0x54 assignment also retains the unresolved signed-arithmetic qualification.

Repeated reads occur after preceding writes and may observe aliases. They cannot generally be replaced with entry-state snapshots. Complete middle-phase access enumeration remains **documented_only** here.

### Constant stores and repeated region

A later word read through an explicitly unsigned-byte-derived base supplies the **loaded word**. Twice it forms the **transfer base**; four times it forms the **payload base**.

The value 0x3a is stored through three distinct explicit accesses: a byte at four times the loaded word, a halfword at eight times it, and a word at payload-base offset 0x20.

The saved word also determines an **adjusted value**, formed by multiplying it by four and subtracting decimal 150. Four times the adjusted value forms the **working base**.

Working-base offset 0x20 receives consecutive word values 0x20 and 0x2f. An unsigned byte is then sampled at working-base offset 4 and widened into a word store at offset 0x20, which is subsequently overwritten with 0x20.

The separate base-supplying global is evaluated next, and its value is retained in a represented 32-bit signed quantity as the **global base**. This retention does not establish the source load’s width or signedness. It precedes assignment of 0x20 to the global associated with 0x60, whose destination width is likewise unresolved.

Repeated groups use the retained global base without reevaluating its source. Each group has four ordered, explicitly sized stores:

| Position | Destination | Supplied value |
|---|---|---|
| First | Word at scaled destination plus 0x54 | Payload base |
| Second | Byte at global-base offset 0x19 | Working base’s low byte |
| Third | Byte at working-base offset 0x11 | Global base’s low byte |
| Fourth | Word at global-base offset 0x34 | Scaled destination |

The first group uses eight times the saved word as its scaled destination. Subsequent groups use eight times the adjusted value. These expressions need not evaluate differently: saved word 50 also produces adjusted value 50 under the stated arithmetic model.

The representation repeats these effects explicitly rather than using a data-dependent loop. Exact multiplicity remains **documented_only** here. Repetition is not permission to collapse accesses.

### Late global evaluations and snapshots

After the repeated region, the global associated with 0xa9 is evaluated and retained as a represented 32-bit value, the **saved global value**. Its underlying load width and signedness are unresolved. A word is then sampled at payload-base offset 0x74.

Next, the global associated with 0x3a is assigned a value formed by converting the adjusted value to a character-sized value, multiplying that value by eight, and adding decimal 48.

The character conversion and subsequent arithmetic do not establish an 8-bit destination store. Character signedness, promotion details, and the missing global declaration affect interpretation. In particular, the representation does not establish that the complete assignment value is truncated to the low byte of eight times the adjusted value plus 48.

After that assignment, two computed word snapshots are taken, first from transfer-base offset 0x14 and then from working-base offset 4. The first is retained for the final invocation’s third argument. The second is later stored at global-base offset 0x74.

Thus, the 0xa9 evaluation precedes the payload-base sample; both precede the 0x3a assignment; both later snapshots follow that assignment. If the global assignment’s actual footprint overlaps either later sample, its effect depends on the missing declaration and memory semantics.

Further computed mutations include a byte store of 0x6c at working-base offset 8, byte stores at offsets 9 and 0x0d, and a self-referential word store at transfer-base offset 0x54.

The global associated with 0x67 is then assigned the saved global value explicitly narrowed to 8 bits. Its destination access width and any subsequent assignment conversion are unresolved.

There are **three distinct later evaluations** of that global:

1. After a word read at transfer-base offset 0x44, its value supplies the assignment to the global associated with 0x21. Neither global’s declaration is supplied, and no explicit byte-width read or byte-width destination write is established for this copy.
2. After intervening computed stores and another word sample, its current value supplies an explicitly 8-bit computed store at working-base offset 9.
3. After further computed effects, including a word read at working-base offset 0x54, its current value supplies another explicitly 8-bit computed store at working-base offset 9.

The last two destinations impose 8-bit storage on the consumed values; that does not impose an 8-bit load on the global evaluations. None of the three evaluations is specified as reuse of the originally assigned value.

Intervening writes include a byte store at working-base offset 0x10, word updates at global-base offset 0x54, and a byte store at transfer-base offset 1. The latter uses a signed-looking right shift of a loaded word, with a count equal to the working base’s low byte. Negative-operand and excessive-count behavior remains unresolved.

### Final invocation and retained arguments

The saved transfer-base offset-0x14 word is eventually stored at transfer-base offset 0x54 and remains retained for the final invocation.

A word loaded from transfer-base offset 4 supplies the **argument base**. Before sampling the final second argument, an explicit halfword store of 0x65 occurs at twice the argument base’s numeric value.

The final second argument is then sampled as a word at argument-base offset 0x64. Later computed effects include a byte update at an already loaded destination’s offset 0x11 and a word store of decimal 100 at its offset 0x54. The final fourth argument is subsequently sampled as a word at argument-base offset 0x74.

A later word read at transfer-base offset 0x64 supplies another destination base. After a transfer-base byte update, the retained second argument is stored at that destination’s offset 0x54. The last explicit memory mutation stores that destination base as a word at twice the second argument’s numeric value.

The unresolved final invocation receives:

1. The 32-bit numeric value 0x72.
2. The retained word sampled at argument-base offset 0x64.
3. The earlier retained word sampled at transfer-base offset 0x14.
4. The retained word sampled at argument-base offset 0x74.

Later writes do not refresh retained arguments. No invocation result is visibly consumed. A return is represented afterward, conditional on the destination returning.

## Side effects

### Undeclared-global interactions

All underlying widths in this table are **documented_only**.

| Global association | Established interaction |
|---|---|
| 0x01 | Assignment supplied with an explicitly 8-bit-converted doubled-index value |
| 0x05 | Evaluation followed by explicit unsigned 32-bit conversion |
| 0x15 | Assignment supplied with an explicitly 8-bit-converted doubled-index value |
| 0x21 | Assignment supplied by the first later evaluation of the global at 0x67 |
| 0x3a | Assignment supplied by character conversion, multiplication by eight, and addition of 48 |
| 0x54 | Assignment supplied by the saved word shifted left by one |
| 0x60 | Assignment supplied with numeric value 0x20 |
| 0x67 | Assignment supplied with an explicitly 8-bit-converted saved global value; three later evaluations |
| 0xa9 | Evaluation retained in a represented 32-bit quantity |
| Separate base-supplying global | One evaluation retained in a represented 32-bit signed quantity |

This inventory specifies value flow and ordering, not underlying memory transaction sizes. Destination conversions may affect global representations; source declarations may affect global value interpretation.

### Computed-memory effects

Explicit computed accesses include byte reads and writes, halfword writes, and word reads and writes. Computed addresses may coincide with any global association above. Such coincidence does not change the explicit computed access width.

No synchronization, atomicity, transaction boundary, or restoration mechanism is established. Endianness, volatile behavior, device semantics, concurrency, and global access footprints remain unresolved. Final memory state alone is insufficient to characterize all represented effects.

## Dependencies

These document-local opaque IDs do not resolve actual dependency identities.

| ID | Role | Unresolved contract |
|---|---|---|
| F-U001 | Initial return-link-derived invocation | Identity, ABI, outputs, effects, ownership, return behavior |
| F-U002 | Ordered subsequent prefix invocations | Per-destination identities, complete argument trace, implicit outputs, effects, lifetimes |
| F-U003 | Final four-argument invocation | Destination selection, argument interpretation, effects, return behavior |

F-U002 does not imply one shared function or interchangeable invocations. No dependency is assumed to preserve globals or own, retain, or release memory.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial truncation.** Under modular 32-bit shifts, fourth inputs 1 and 129 both produce 0x02000000 for the initial second argument. This does not imply identical complete executions.

2. **Small-index boundary.** A saved source word of 0x7f produces small index 508 and doubled index 1016; 0x80 produces zero. No represented check rejects zero.

3. **Explicit unsigned-byte widening.** A byte sample of 0xfe at scaled-base offset 5 supplies word value 0x000000fe at offset 0x34, not a sign-extended value.

4. **Global conversion is not a byte load.** If the 0x05 evaluation converts to unsigned value 0x00000122, its computed offset-0x15 and offset-0x54 addresses are 0x137 and 0x176 under modular arithmetic. The base must not be truncated to 0x22.

5. **Narrow assignment value, unknown destination width.** If the doubled index is 0x120, the explicit 8-bit conversion supplying the 0x01 assignment yields 0x20. The contract does not determine how many bytes the destination assignment accesses or which neighboring storage it affects.

6. **Retained global base.** If the separate global’s evaluation produces retained base 0x200, repeated groups continue using that base even if their computed stores alter its source storage. This does not establish a 32-bit source load.

7. **Late global assignment overlap.** If the actual footprint of the 0x3a assignment overlaps transfer-base offset 0x14, the snapshot follows that assignment. A precise resulting word requires the global declaration, arithmetic interpretation, endianness, and memory semantics.

8. **Three fresh evaluations.** If an intervening computed write changes the value subsequently obtained from the global at 0x67, the next evaluation uses that changed value. It need not match the value supplied to the earlier 0x21 assignment.

9. **Computed narrowing remains established.** If either later 0x67 evaluation supplies numeric value 0x1234, its consuming computed 8-bit store writes 0x34 under ordinary integer narrowing. This condition supplies the evaluated value; it does not assert that the missing global declaration permits it.

10. **Unknown global-to-global conversion.** Knowing only the 8-bit value originally supplied to the 0x67 assignment is insufficient to specify the complete later 0x21 assignment or its access footprint.

11. **Saved final argument.** Overwriting transfer-base offset 0x14 after its snapshot does not refresh the final third argument.

12. **Partial failure.** If the first explicit computed store fails, later effects and the final invocation are not guaranteed. Earlier dependency effects have no represented rollback.

## Uncertainties

The following remain **documented_only**:

- Application purpose, actual ABI, implicit inputs, and register-result conventions.
- Dependency identities, complete prefix trace, repeated-region multiplicity, and complete middle-phase enumeration.
- Declarations, widths, signedness, qualifiers, and assignment conversions of every referenced global.
- Character conversion and promotion semantics affecting the 0x3a assignment.
- Address mappings, object boundaries, aliasing restrictions, and resource lifetimes.
- Endianness, alignment, device-memory behavior, concurrency, and atomicity.
- Signed overflow and signed or excessive-count shift behavior.
- Final invocation destination, guaranteed return behavior, and environmental failure handling.

Unenumerated trace details are coverage limits, not evidence that operations are interchangeable. Missing declarations prevent exact global access and overlap semantics; they do not invalidate explicitly sized computed accesses.

**[INFERENCE]:** Pointer chaining may suggest linked state, and some constants may suggest textual activity. Neither establishes intended application behavior.

## Evidence

Evidence consists solely of the supplied static representation. Explicit computed dereferences support the stated widths. Explicit conversions support supplied assignment values and retained-value widths. Ordered expressions support the snapshot and evaluation relationships.

No global declarations were supplied. Accordingly, neither assignment-side narrowing nor retention in a 32-bit quantity establishes any global’s underlying access width or signedness. This limitation applies equally to fixed-address-associated globals and the separate base-supplying global. It also applies to all three evaluations associated with 0x67.

Comments, warnings, prior prose, and reviewer feedback are not independent runtime evidence. No dependency bodies, machine instructions, ABI documentation, memory map, or execution observations were supplied.

No runtime tests have been performed. Acceptance examples are conditional consequences of represented operations and stated assumptions, not observed execution results.
