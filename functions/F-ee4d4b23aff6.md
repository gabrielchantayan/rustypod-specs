<!-- Trusted provenance: F-ee4d4b23aff6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ee4d4b23aff6

## Purpose

F-ee4d4b23aff6 invokes an ordered sequence of indirect dependencies, transforms their results and implicit execution state, modifies memory through computed addresses and fixed-address global objects, and reaches a final unresolved indirect invocation.

No domain-level purpose is established. Character-like constants do not establish text output, formatting, serialization, or resource management.

This contract distinguishes:

- **Established static evidence:** operations and relationships represented by the supplied body.
- **[INFERENCE]:** interpretations requiring assumptions beyond that representation.
- **documented_only:** unresolved behavior or details not fully specified here.

All behavioral claims are conditional on the supplied representation accurately describing execution. The apparent interface and control-flow boundaries have not been independently verified.

## Interface

### Explicit inputs and output

| Input | Represented width | Established use |
|---|---:|---|
| First argument | 32 bits | Forwarded unchanged to the initial dependency. |
| Second argument | 32 bits | No explicit use shown. |
| Third argument | 32 bits | No explicit use shown. |
| Fourth argument | Signed 32 bits | Shifted left seven positions for the initial dependency’s second displayed argument. |

No returned value, status code, or dedicated output-buffer parameter is represented. A return follows the final invocation, but whether that invocation returns is unresolved.

The apparent non-use of the second and third arguments does not establish their irrelevance under an unknown calling convention.

### Implicit inputs

Execution also depends on incoming control-related state, an incoming 32-bit execution-state quantity without visible initialization, secondary call-result state, dependency results, existing memory, global-object values, and the final indirect target.

Uninitialized incoming state must not be replaced with zero. A displayed argument list does not exhaust a dependency’s possible implicit inputs.

### Widths, layouts, and conversions

A byte is 8 bits, a halfword 16 bits, and a word 32 bits. Computed address-bearing values are represented with 32 bits. Some dependency results are represented with 64 bits and consumed as separate lower and upper words.

**Explicit pointer accesses and undeclared global accesses have different evidence strength:**

- Explicit byte, halfword, and word pointer accesses establish their represented access widths.
- References to fixed-address global objects do not establish memory-access widths when their declarations are absent.
- Conversion of a global value into a 32-bit temporary establishes the retained value’s width, not the width of the underlying read.
- Assigning an explicitly byte-sized value to an undeclared global does not establish a byte-sized destination or transaction.

These qualifications apply to global objects associated with `0x01`, `0x05`, `0x15`, `0x21`, `0x3a`, `0x54`, `0x60`, `0x67`, and `0xa9`, and to the separate base-storage object.

In particular, the global value associated with `0x05` is converted to unsigned 32 bits. Its original width and signedness are **documented_only**. The conversion does not independently establish a byte load, zero-extension, or a result restricted to 0–255.

Explicit byte reads represented as unsigned bytes are zero-extended when widened. Explicit byte stores retain the destination byte’s converted value. Byte order, alignment, mapped storage, and object boundaries remain unresolved.

Unsigned right shifts are logical. Signed overflow, negative signed left shifts, signed right shifts of negative values, and excessive shift counts remain unresolved.

The **conditional machine-arithmetic interpretation** used below retains the low 32 bits of word arithmetic. It is an assumption for deductions, not an established execution model. **Ordinary memory** additionally means byte-addressed storage with normal read-after-write behavior, no external mutation, and no access-triggered effects.

## Behavior

### Dependency phase

The represented sequence contains a long indirect-call phase, a direct-memory phase, and the final invocation. No input-dependent branch, validation guard, or loop condition is visible. Later effects require all preceding operations to complete normally.

The initial target is derived from incoming control state using displacement `0xffe`. Its displayed arguments are the first explicit argument and the fourth argument shifted left seven positions.

Intermediate targets are derived from represented code-location quantities using the same displacement. Similar invocation patterns do not prove target identity.

Arguments include shifted dependency results, separately transformed lower and upper result words, retained values spanning intervening calls, explicit zeros, secondary execution-state results, masked fields, and logical-right-shifted quantities. A lower word shifted left eight positions alongside an unchanged upper word is not an unchanged 64-bit argument.

Calls without displayed arguments remain observable invocations. Their effects and implicit inputs are unresolved. The complete intermediate-call inventory and argument trace are **documented_only** in this contract.

### Early computed-memory effects

The following descriptive labels denote retained values, not source identifiers or validated objects:

| Label | Definition |
|---|---|
| High base | Upper word of the final represented 64-bit dependency result. |
| Scaled low base | Lower word of that result multiplied by 256. |
| Shifted base | Retained earlier result shifted left 25 positions. |
| Product base | Another retained dependency result multiplied by \(2^{25}\). |
| Small base | Twice the logical-right-shift-by-six result of an earlier eight-position-left-shifted result masked with `0x7fff`. |
| Saved word | Another retained dependency result shifted left eight positions. |

The first memory effects preserve this order:

| Order | Explicit access |
|---:|---|
| 1 | Word write of shifted base at high base plus `0x14`. |
| 2 | Byte write of high base’s low byte at shifted base plus `0x09`. |
| 3 | Word write of product base at scaled low base plus `0x54`. |
| 4 | Byte write of shifted base’s low byte at shifted base plus `0x09`. |
| 5 | Word write of zero at product base plus `0x14`. |
| 6 | Unsigned byte read at product base plus `0x05`. |
| 7 | Word write of zero at small base plus `0x14`. |
| 8 | Word write of the retained byte, zero-extended, at product base plus `0x34`. |
| 9 | Word read at product base plus `0x14`. |
| 10 | Another zero word write at small base plus `0x14`. |
| 11 | Two consecutive byte writes of access 9’s low byte at small base plus `0x15`. |

In ordinary memory, access 9 reads zero. The widened-byte write is 32 bytes beyond the zeroed word and cannot overlap it. An overlapping intervening zero write cannot introduce nonzero bytes. Ordinary aliasing alone therefore does not explain a nonzero result.

Under conditional machine arithmetic, small base is a multiple of eight between 0 and 1016 inclusive. The left shift before masking excludes values that the mask alone would permit.

### Converted global value and early links

After the duplicate byte writes, the global associated with `0x05` is read and converted to unsigned 32 bits. Call the result **Converted global base**. The global associated with `0x01` is then assigned an explicitly 8-bit value derived from small base.

The global read precedes the global assignment. Their transaction widths are unresolved; neither may be modeled unconditionally as a one-byte access.

Converted global base supplies an explicit byte-write address at offset `0x15`, followed later by explicit word-read and word-write addresses at offset `0x54`. No additional table-base address is represented. Converted global base is not established to be byte-derived or confined to low addresses.

The explicit word read at converted global base plus `0x54`, before that location is overwritten, produces **Link word**. Subsequent derivations are:

- **Fourfold origin:** four times link word.
- **Adjusted origin:** fourfold origin minus 150.
- **Computed base:** four times adjusted origin.

The linked subsection includes explicit accesses at offsets `0x04`, `0x44`, `0x64`, and `0x74`; loaded words become later bases or stored link values. Self-valued word writes, duplicate writes, and potentially overlapping byte and word fields are represented. No alias-exclusion condition is visible.

An assignment to the global associated with `0x54` uses link word shifted left one position. It follows a computed word write at small base plus `0x14` and precedes two further word writes of link word there. The global assignment’s destination width and conversion remain unresolved.

Later, the global associated with `0x15` is assigned an explicitly 8-bit value derived from small base. This does not establish a byte transaction or exclude overlap with computed accesses.

Two explicit unsigned byte reads from the same loaded base plus `0x05` are separated by stores. The second read, zero-extended and used as a base with offset `0x74`, supplies a word called **Nested word**. Twice nested word is **Doubled nested base**. Replacing the second byte read with the first cached byte is not justified.

The linked subsection’s complete access trace is **documented_only** here; these offsets are not an exhaustive inventory.

### Constants, base retention, and repetition

Explicit stores place `0x3a`:

- As one byte at four times nested word.
- As one halfword at eight times nested word.
- As one word at four times nested word plus `0x20`.

At computed base plus `0x20`, successive word writes contain `0x20`, `0x2f`, an unsigned byte read from computed base plus `0x04` widened to a word, and `0x20` again. That byte read occurs after the `0x2f` write.

Next, separate base storage is read into a signed 32-bit retained value called **Loaded base**. Its underlying read width and original signedness are unresolved. This read precedes assignment of `0x20` to the global associated with `0x60`; that assignment’s destination width is also unresolved. Reordering is unjustified because overlap has not been excluded.

A finite series of repeated groups follows. Each group preserves this effect order:

1. Word write of four times nested word at a scaled origin plus `0x54`.
2. Byte write of computed base’s retained low byte at loaded base plus `0x19`.
3. Byte write of loaded base’s retained low byte at computed base plus `0x11`.
4. Word write of the scaled origin at loaded base plus `0x34`.

The first scaled origin is eight times link word. Subsequent groups use eight times adjusted origin. Loaded base and both byte values are retained, not visibly reloaded.

The numeric repetition count is **documented_only** here. Explicit finite repetition is established, not an indefinite loop. Repeated accesses must not be collapsed merely because addresses and values repeat.

### Later fixed-global interactions

After repetition, the global associated with `0xa9` is read into a retained 32-bit value. Call it **Retained global word**. This name describes the retained width, not an independently established 32-bit memory read.

An explicit word read at four times nested word plus `0x74` follows. Its result remains distinct from retained global word.

The global associated with `0x3a` then receives a character-derived arithmetic expression: adjusted origin is converted to character width, multiplied by eight, and increased by `0x30`. Character signedness and destination conversion are unresolved. Under ordinary 8-bit character conversion and conditional machine arithmetic, the expression’s low byte equals the low byte of eight times adjusted origin plus `0x30`; the global assignment is not established to store exactly one byte or only that low byte.

Explicit word reads save doubled nested base plus `0x14` and computed base plus `0x04`. Subsequent explicit writes place:

- Computed base’s low byte at doubled nested base plus `0x0d`.
- Byte `0x6c` at computed base plus `0x08`.
- Loaded base’s low byte at computed base plus `0x0d` and plus `0x09`.
- Doubled nested base itself as a word at doubled nested base plus `0x54`.

The global associated with `0x67` is then assigned an explicitly 8-bit value extracted from retained global word. An explicit word read at doubled nested base plus `0x44` follows. Next, a fresh read of the global associated with `0x67` supplies an assignment to the global associated with `0x21`.

Neither global assignment has an established transaction width. The global-to-global assignment does not itself establish a byte-sized read, byte-sized destination, or explicit byte conversion. No computed store intervenes between the assignment to `0x67` and this first fresh read, but the computed word read remains part of the ordering.

### Dynamic shift

The word read at doubled nested base plus `0x44` supplies an explicit byte write at computed base plus `0x10`, a word write at loaded base plus `0x54`, and a signed-right-shift-derived byte write at doubled nested base plus `0x01`.

The shift count is computed base’s low eight bits. Under conditional machine arithmetic:

\[
\text{computed base}
\equiv 4(4\,\text{link word}-150)
\equiv 16\,\text{link word}-600
\pmod{2^{32}}.
\]

The candidate counts are exactly:

**8, 24, 40, 56, 72, 88, 104, 120, 136, 152, 168, 184, 200, 216, 232, 248.**

Every candidate arises arithmetically from some low four bits of link word. This does not prove successful traversal of preceding accesses.

Only 8 and 24 are below 32. No modulo-32 shift-count rule is established. Execution for larger counts, and signed right shift of a negative word, remains **documented_only**.

### Fresh global reads and final linking

After the shift-derived byte store, loaded base plus `0x54` is overwritten with computed base. An explicit word read at computed base plus `0x40` supplies a later word write at computed base plus `0x10`. Between that read and write, the saved computed-base-plus-`0x04` word is written at loaded base plus `0x74`. Doubled nested base is then written at loaded base plus `0x44`.

A second fresh read of the global associated with `0x67` supplies an explicit one-byte store at computed base plus `0x09`. A byte store at doubled nested base plus `0x09` and a word store of retained global word at computed base plus `0x44` follow.

An explicit word read at computed base plus `0x54` then produces **Final link base**. A third fresh read of the global associated with `0x67` supplies another explicit one-byte store at computed base plus `0x09`.

These are three distinct global observations. Their original read widths and signedness remain unresolved. The two explicit byte destinations establish the widths of those stores, not the global reads. Intervening computed writes may overlap the global object, so cached substitution is unjustified.

The retained low byte from four-times-nested-word plus `0x74` is published at final link base plus `0x0d`. The saved doubled-nested-base-plus-`0x14` word is restored at doubled nested base plus `0x54`.

An explicit word read at computed base plus `0x14` precedes further linking: the retained byte is published at doubled nested base plus `0x11`, and final link base’s low byte and full word are written at computed base plus `0x09` and plus `0x44`.

A word read at doubled nested base plus `0x04` supplies a new base. A halfword `0x65` is written at twice that base. Word reads at its offsets `0x64` and `0x74` follow, separated by a byte write through final link base plus `0x11` and a word write of 100 at final link base plus `0x54`.

Afterward, a word read at doubled nested base plus `0x64` supplies another link destination. A byte write at doubled nested base plus `0x09` precedes the two final word links.

The final dependency receives, in order, `0x72`, the earlier `0x64` result, the saved doubled-nested-base-plus-`0x14` word, and the new-base-plus-`0x74` result.

## Side effects

Computed-memory mutation and global assignments are established. Global declarations are insufficient to specify every affected byte or overlap.

No ownership transfer, allocation, release, or retention policy is established. Dependencies must not be assumed to preserve globals or memory ownership.

Successful traversal requires compatible executable targets and valid reached accesses with appropriate permissions and alignment. These are environmental preconditions, not performed checks.

No null, range, capacity, recovery, rollback, locking, or atomicity mechanism is visible. Earlier writes may remain after a fault or nonlocal exit. External mutation and special-memory behavior can invalidate ordinary-memory deductions.

## Dependencies

| Opaque ID | Role | Unresolved properties |
|---|---|---|
| F-unknown-001 | Initial incoming-state-derived dependency | Identity, calling convention, results, effects. |
| F-unknown-002 | Intermediate indirect dependencies | Individual identities, complete argument/result contracts, effects. |
| F-unknown-003 | Final unresolved dependency | Identity, effects, termination, return behavior. |

F-unknown-002 is a grouping label, not an assertion that all invocations share one target. None is assumed pure, ownership-preserving, or global-state-preserving.

## Acceptance examples

These are conditional static scenarios, not executed tests.

1. **Initial forwarding:** with fourth argument 3 and valid arithmetic, the initial second displayed argument is 384; the first remains unchanged.

2. **Separate result halves:** lower word `0x12` shifted eight positions becomes `0x1200`; a separately forwarded upper word `0x3456789a` remains unchanged.

3. **Explicit unsigned byte:** an explicit unsigned byte read of `0xfe`, widened for a word store, yields `0x000000fe`.

4. **Undeclared global conversion:** if the `0x05` object were unsigned 8-bit with value `0xfe`, its unsigned-32 conversion would yield `0x000000fe`. If it were signed 8-bit with value −2, that conversion would yield `0xfffffffe`. A wider declaration could yield a value above 255. No declaration is selected by this contract.

5. **Global destination width:** assigning an 8-bit value derived from small base to the `0x01` object does not permit an acceptance requirement that exactly one byte changes. Its declaration is required to determine that footprint.

6. **Early zero:** in ordinary memory, the early product-base-plus-`0x14` word read yields zero despite possible overlap by the intervening zero write.

7. **Restricted arithmetic:** small base can be 0 or 1016, but not 2 or 1022. Link word 54 produces computed base `0x108` and count 8; link word 56 produces `0x128` and count 40. The latter does not establish shift execution semantics.

8. **Fresh observations:** if intervening writes change the value subsequently read from the `0x67` object, later explicit byte stores use the newly observed value after destination conversion, not a cached first observation.

9. **Retained width:** conversion of the `0xa9` global value retains 32 bits; it does not require a four-byte source transaction.

10. **Partial failure:** a later fault does not require restoration of earlier writes.

## Uncertainties

| Topic | Classification |
|---|---|
| Domain purpose | documented_only; text-oriented intent is [INFERENCE]. |
| Complete calling convention and dependency identities | documented_only. |
| Global declarations, transaction widths, signedness, and destination conversions | documented_only. |
| Signed arithmetic, character conversion, and excessive shifts | documented_only. |
| Byte order, alignment, mappings, and special-memory effects | documented_only. |
| Complete intermediate-call and linked-memory traces; numeric repetition count | documented_only in this contract. |
| Ownership, dependency errors, termination, and final return | documented_only. |

These limits prevent a complete standalone reimplementation contract. They do not authorize invented validation, fallback initialization, narrowed global values, cached fresh reads, or recovery.

## Evidence

The supplied body establishes explicit pointer-access widths, retained temporary widths, conversion points, ordering, repeated effects, and the final displayed arguments.

Global declarations are absent. Consequently, the unsigned-32 conversion of the `0x05` value does not establish a byte load; retention of the `0xa9` value in 32 bits does not establish a word read; and byte-valued global assignments do not establish byte-sized objects.

The early-zero deduction follows from intervening store values and nonoverlapping offsets. The small-base restriction follows from shifting before masking. The dynamic-shift candidate set follows from reducing \(16\,\text{link word}-600\) modulo 256.

Prior prose, comments, and feedback are not independent behavioral authority. No tools, dependency inspection, runtime tests, execution, or memory-layout validation have been performed.
