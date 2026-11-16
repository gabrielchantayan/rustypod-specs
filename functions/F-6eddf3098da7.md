<!-- Trusted provenance: F-6eddf3098da7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6eddf3098da7

## Purpose

F-6eddf3098da7 performs an ordered chain of indirect invocations and integer transformations, followed by mixed-width storage mutations, retained storage snapshots, and a terminal indirect invocation.

No domain-level purpose is established. Repeated offsets do not establish an object model, and character-like constants do not establish textual output.

This contract distinguishes **established static evidence**, **[INFERENCE] inferred intent**, and **documented_only** unknowns. Descriptions of later operations are conditional on earlier invocations returning and required accesses succeeding. No transaction, rollback, or success result is represented.

## Interface

### Effective inputs and outputs

No explicit formal parameters are displayed. Effective inputs include incoming machine state, return-link state, state exposed after indirect invocations, dependency results, and storage reached through fixed or computed addresses.

Ordinary retained values occupy 32-bit containers. Selected dependency results occupy 64-bit containers. The ABI, register mapping, and complete dependency interfaces are **documented_only**. An invocation with no displayed arguments is not established to consume no state.

No explicit returned data value is represented. Observable effects include dependency invocations, supplied arguments, direct storage updates, and any dependency effects.

The terminal invocation has four visible argument positions:

1. `0x72`.
2. A late word captured from Terminal base `+0x64`.
3. An earlier word captured from Doubled anchor `+0x14`.
4. A separate late word captured from Terminal base `+0x74`.

The fourth argument is not the earlier `+0x74` snapshot used for byte stores. A return is represented after the terminal invocation, conditional on that invocation returning normally.

### Widths and layouts

| Access or value | Established property |
|---|---|
| Explicit byte-pointer access | 8 bits |
| Explicit halfword-pointer store | 16 bits |
| Explicit word-pointer access | 32 bits |
| Ordinary retained value | 32-bit container |
| Selected dependency result | 64-bit container |
| Explicit one-byte conversion | Supplies an eight-bit representation |
| Undeclared named storage | Access width and signedness are **documented_only** |

Explicit unsigned-byte loads are zero-extended when converted to words. Explicit byte stores retain the low eight bits. Halfword and word stores have two-byte and four-byte footprints.

A supplied value’s width does not establish an undeclared destination’s access width:

- The named value at `0x05` is converted to a 32-bit unsigned value. Its source width and extension behavior are unresolved.
- Assignments to named storage at `0x01`, `0x15`, and `0x67` explicitly supply one-byte-converted representations, but do not establish byte-sized destination accesses.
- The named assignment at `0x3a` performs character conversion before multiplication and addition. Its final destination conversion and storage width are unresolved.
- Reads of named storage at `0x67`, including the assignment to named storage at `0x21`, do not independently establish byte-sized source accesses.

Similarly, retaining a named value in a word establishes the retained width, not the source footprint.

Separating the low and high words of a 64-bit result does not establish memory endianness. Offsets identify access locations, not proven fields. Bounds, alignment, ownership, and non-aliasing are unresolved.

### Preconditions and arithmetic

Completion requires executable indirect targets, normal return from required invocations, and accessible storage for all required reads and writes. No visible validation rejects zero addresses, low addresses, misalignment, or inaccessible storage.

Intermediate widths matter. Unsigned 32-bit operations use fixed-width arithmetic; they cannot be replaced indiscriminately with unrestricted integer arithmetic. Signed overflow, signed left shifts, signed right shifts of negative values, and oversized shift counts require additional target evidence and remain **documented_only**.

## Behavior

### Invocation and transformation phase

The opening phase is a long straight-line chain of indirect invocations. No explicit loop or conditional branch is represented.

The first target is formed from incoming return-link state plus `0xffe`. Subsequent targets predominantly use the same displacement from embedded execution-location values, including address-valued references. Concrete dependency identities are unresolved.

Invocation order is significant. Dependencies may change memory, named storage, or machine state subsequently consumed. A displayed result being unused does not make its invocation removable.

Visible argument relationships include:

- A result’s low word shifted left eight bits alongside its high word.
- Multiple shifted forms of one captured word.
- Identical values supplied in multiple positions.
- Explicit zeros and retained zeros that remain zero through later transformations.
- Masked low-bit ranges, unsigned right shifts, and extractions above bit 32 from 64-bit results.

Left-shift counts extend through 31 bits. Under 32-bit bit-vector interpretation, two successive retained left shifts by 28 bits produce zero; this observation does not resolve signed-operation semantics.

The complete invocation inventory and argument enumeration are **documented_only** in this contract.

### Opening mutations and captures

The following descriptive roles identify retained values:

| Role | Meaning |
|---|---|
| Opening base | A captured result scaled by `0x02000000` |
| Opening shifted word | Another captured result shifted left eight bits |
| Small scale | A retained unsigned masked-and-shifted word |
| Double scale | Twice Small scale |
| Global-derived base | Unsigned 32-bit conversion of named storage at `0x05` |
| Link snapshot | Word subsequently captured from Global-derived base `+0x54` |

These roles are not domain types or ownership claims.

The direct-memory phase begins with a word store at the final dependency result’s high word `+0x14`, supplying a retained shifted value. A byte destination at a retained base `+9` is written twice, with a word store through the final result’s scaled low word between those byte writes.

The next region has a required interleaving. Opening base `+0x14` is first cleared as a word, and an unsigned byte is then sampled from Opening base `+5`. **After that sample, the first word clear at Double scale `+0x14` precedes the word store of the zero-extended sampled byte at Opening base `+0x34`.** Only after that byte-derived word store is the word at Opening base `+0x14` captured. A second word clear at Double scale `+0x14` follows the capture.

Thus, the represented order is:

**Opening clear → byte sample → first Double scale clear → byte-derived word store → Opening word capture → second Double scale clear.**

Two byte stores at Double scale `+0x15` then supply the captured Opening word’s low byte. Neither aliasing uncertainty nor device-storage uncertainty permits reordering the preceding events. In particular, the byte-derived word store is between the two Double scale clears and before the Opening word capture.

Next, named storage at `0x05` is sampled and converted to Global-derived base. Named storage at `0x01` receives the one-byte conversion of Double scale. Explicit byte stores then supply Small scale’s low byte at Global-derived base `+0x15` and Opening shifted word’s low byte at Opening base `+0x11`.

Opening shifted word is stored as a word at the captured Opening word `+0x54`. Opening base `+0x11` is then overwritten with a zero byte.

Link snapshot is captured from Global-derived base `+0x54` before that location is overwritten with the captured Opening word. Double scale `+0x34` subsequently receives Small scale as a word.

Global-derived base remains retained; later uses do not reread `0x05`. It is not established to be between zero and 255.

### Linked and scaled mutations

Further mutations use word loads at `+0x74`. One assignment samples Double scale `+0x74` twice: one read supplies a destination base and another supplies stored data. Both samples belong to the represented access trace. Their relative evaluation order within that assignment is **documented_only**.

Repeated self-valued word stores at Link snapshot `+0x24` are interleaved with byte writes and other word stores. Double scale `+0x14` receives a shifted retained value; named storage at `0x54` then receives Link snapshot shifted left one bit; and Double scale `+0x14` subsequently receives Link snapshot twice.

Double scale `+0x15` receives the low byte of four times Link snapshot twice. Named storage at `0x15` receives the one-byte conversion of Double scale. Further word stores through four-scaled bases follow.

Later reads use offsets `+0x64`, `+0x74`, and `+4`; loaded values supply destinations at `+0x44` and `+0x54`. Byte writes at `+9`, `+0x0d`, and `+0x15` remain interleaved with these operations.

Some addresses are explicitly byte-derived: an unsigned byte sampled at a computed base `+5` becomes the base for a word load at `+0x74`. Such a sampled base is between zero and 255 before the offset is added. This guarantee does not apply to Global-derived base.

The later retained roles are:

- **Link anchor:** the later word obtained through an explicit unsigned-byte-derived `+0x74` access.
- **Doubled anchor:** twice Link anchor.
- **Quadrupled anchor:** four times Link anchor.
- **Adjusted base:** four times Link snapshot, less `0x96`.
- **Working base:** four times Adjusted base.

Subject to the represented arithmetic, constant `0x3a` is stored as a byte at Quadrupled anchor, as a halfword at eight times Link anchor, and as a word at Quadrupled anchor `+0x20`. These accesses are distinct from the later named assignment at address `0x3a`.

Working base `+0x20` receives four word writes in order: `0x20`, `0x2f`, a zero-extended unsigned byte sampled from Working base `+4`, and `0x20` again. The intervening byte read occurs after the `0x2f` write and before the byte-derived word write.

### Fixed-base capture and repeated updates

Named storage at `0x087fbbbc` is read into a retained 32-bit **Fixed-base snapshot**. Its source width and signedness are unresolved. Only afterward does named storage at `0x60` receive `0x20`; that destination’s footprint is also unresolved.

A repeated mutation region interleaves four access roles in each group:

- A word store at an eight-scaled base `+0x54`, supplying Quadrupled anchor.
- A byte store at Fixed-base snapshot `+0x19`, supplying Working base’s low byte.
- A byte store at Working base `+0x11`, supplying Fixed-base snapshot’s low byte.
- A word store at Fixed-base snapshot `+0x34`, supplying the eight-scaled base.

The first group uses eight times Link snapshot; subsequent groups use eight times Adjusted base.

Fixed-base snapshot is retained throughout. These operations do not reload its named source. Repetition establishes neither retry nor convergence. Exact multiplicity is **documented_only** here, not permission to replace the repeated access trace with final-state assignments.

### Later snapshots and named-storage sampling

After the repeated region, named storage at `0xa9` is captured into a retained 32-bit **Fixed-word snapshot**. A word from Quadrupled anchor `+0x74` is then captured as **Early-74 snapshot**.

The named assignment at `0x3a` converts Adjusted base to a character-sized value, multiplies its promoted value by eight, and adds `0x30`. Character signedness and final storage conversion are unresolved. The expression is not established to be truncated to eight bits on storage.

Two explicit word captures follow: **Saved-14** from Doubled anchor `+0x14`, and **Saved-4** from Working base `+4`.

Explicit byte stores supply Working base’s low byte at Doubled anchor `+0x0d`, `0x6c` at Working base `+8`, and Fixed-base snapshot’s low byte at Working base `+0x0d` and `+9`. Doubled anchor is stored as a word at its own `+0x54`.

Named storage at `0x67` receives the one-byte conversion of Fixed-word snapshot. Three subsequent reads of this named storage are distinct samples:

1. After a word is loaded from Doubled anchor `+0x44`, the current named value at `0x67` is assigned to named storage at `0x21`.
2. After intervening updates, another read supplies an explicit byte store at Working base `+9`.
3. After further updates and a word capture from Working base `+0x54`, another read supplies a second explicit byte store at Working base `+9`.

The first assignment has no independently established one-byte narrowing or footprint. The latter two destination accesses are explicitly one byte; their source footprints remain unresolved.

Between the first and second samples, the word captured from Doubled anchor `+0x44` supplies a byte at Working base `+0x10`, a word at Fixed-base snapshot `+0x54`, and a signed-right-shift result narrowed to a byte at Doubled anchor `+1`. The shift count is Working base’s low eight bits. Negative operands and large counts remain **documented_only**.

Fixed-base snapshot `+0x54` is then overwritten with Working base. A word is captured from Working base `+0x40`; Saved-4 is stored at Fixed-base snapshot `+0x74`; the captured `+0x40` word is stored at Working base `+0x10`; and Doubled anchor is stored at Fixed-base snapshot `+0x44`. The second named sample follows.

Before the third sample, Doubled anchor `+9` receives Working base’s low byte, Working base `+0x44` receives the full Fixed-word snapshot, and Working base `+0x54` is captured as **Loaded destination**.

Computed accesses may alias named storage. Later samples from `0x67` therefore need not equal its initially supplied representation. Stores sourced from Fixed-word snapshot still use that captured word.

### Terminal mutations and argument capture

Early-74 snapshot’s low byte is stored at Loaded destination `+0x0d`. Saved-14 is then stored at Doubled anchor `+0x54`.

A word is captured from Working base `+0x14`. Early-74 snapshot’s low byte is subsequently stored at Doubled anchor `+0x11`. Working base `+9` receives Loaded destination’s low byte, and Working base `+0x44` receives its full word.

A word loaded from Doubled anchor `+4` becomes **Terminal base**. A halfword containing `0x65` is stored at twice Terminal base.

The word at Terminal base `+0x64` becomes **Terminal-64 snapshot**. Loaded destination `+0x11` receives the earlier Working base `+0x14` capture’s low byte, and Loaded destination `+0x54` receives decimal `100` as a word.

**Late-74 snapshot** is then captured from Terminal base `+0x74`, followed by **Final destination** from Doubled anchor `+0x64`. Doubled anchor `+9` receives Terminal base’s low byte.

Two final word stores precede the terminal invocation: Terminal-64 snapshot at Final destination `+0x54`, then Final destination at twice Terminal-64 snapshot.

The terminal arguments are `0x72`, Terminal-64 snapshot, Saved-14, and Late-74 snapshot, in that order. Aliasing during the final stores does not resample already captured arguments. No cleanup or restoration is represented before return.

## Side effects

Direct effects comprise explicit 8-, 16-, and 32-bit pointer-based writes and named assignments with unresolved footprints.

Repeated writes, overwritten values, and intervening reads remain represented events. The opening interleaving specifically cannot be reduced to “clear both locations, then copy the byte”: its first Double scale clear precedes the byte-derived word store, while its second follows the Opening word capture.

A byte at `+0x15` can overlap a word beginning at `+0x14`. Endianness determines the affected bits. Named symbols do not establish independent storage; their missing declarations prevent a complete overlap map.

Atomicity, synchronization, concurrency, isolation, and reentrancy are unresolved. No explicit allocation, release, retain, or ownership transfer is identified. Dependencies may nevertheless manage memory.

No validation branch, error result, recovery handler, or rollback is shown. Failure may leave earlier effects in place and prevent later accesses or invocations. Exact fault, exception, and non-return behavior is **documented_only**.

## Dependencies

| Opaque function ID | Documentary role |
|---|---|
| F-unknown-01 | Initial indirect target derived from incoming return-link state |
| F-unknown-02 | Subsequent relative-target invocation family |
| F-unknown-03 | Terminal indirect target |

These identifiers do not resolve concrete functions. F-unknown-02 does not imply that its invocations share one target.

Complete inputs, outputs, global effects, memory effects, ownership behavior, errors, and termination are **documented_only**. No dependency is assumed pure, globally preserving, or responsible for releasing memory.

## Acceptance examples

These are conditional static examples, not executed tests.

### Opening access order

Suppose the unsigned byte sampled from Opening base `+5` is `0xfe`. The required trace places the first word clear at Double scale `+0x14` before the word write supplying `0x000000fe` at Opening base `+0x34`. The Opening base `+0x14` word capture follows that write, and the second Double scale clear follows the capture.

A trace placing the byte-derived word write before the first Double scale clear does not satisfy this contract, even if ordinary non-aliasing memory would end with identical contents.

If the first Double scale clear faults, the earlier Opening clear and byte sample have already been reached, but the byte-derived word write and Opening word capture have not.

### Captured values and aliasing

If Opening base `+0x14` is ordinary storage and no intervening access aliases or otherwise changes it, its later capture yields zero. Without those conditions, the earlier clear alone does not establish the captured value.

The two later byte stores at Double scale `+0x15` use that captured word’s low byte, not a fresh read.

### Explicit width versus named conversion

An explicit byte store supplied `0x12345678` writes `0x78`. An explicit unsigned-byte load yielding `0xfe`, converted and stored as a word, supplies `0x000000fe`.

A one-byte conversion supplied to named storage at `0x67` establishes an eight-bit representation, not a one-byte destination footprint.

### Global-derived base and character arithmetic

If named storage at `0x05` is an unsigned byte containing `0xfe`, Global-derived base becomes `0x000000fe`. If it is instead a signed byte representing `-2`, conversion to unsigned 32 bits produces `0xfffffffe`. The supplied evidence does not select a declaration.

If Adjusted base’s character conversion yields numeric `127`, the subsequent arithmetic produces `1064`, or `0x428`. An unsigned eight-bit destination retains `0x28`; a sufficiently wide destination may retain `1064`.

### Retained and resampled values

If Fixed-word snapshot is `0x11223344`, its one-byte conversion supplies representation `0x44` to named storage at `0x67`. If an intervening effect makes a later named read yield `0x9a`, the following explicit byte store supplies `0x9a`, while the separate word store sourced from Fixed-word snapshot still supplies `0x11223344`.

### Terminal captures and partial failure

If Early-74 snapshot is `0x11111111` and Late-74 snapshot is `0x22222222`, the former supplies its designated byte stores; the fourth terminal argument is `0x22222222`.

If Terminal-64 snapshot is `0x1000` and Final destination is `0x2000`, valid arithmetic and writable storage produce a word write of `0x1000` at `0x2054`, followed by a word write of `0x2000` at `0x2000`. If the second write faults, earlier effects need not be undone and the terminal invocation need not occur.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, callers, ABI, and effective input mapping.
- Concrete dependency identities and complete contracts.
- Function-boundary reliability and terminal control-flow interpretation.
- Named-storage declarations, widths, signedness, and overlap footprints.
- Global-derived base’s range beyond its retained unsigned 32-bit value.
- Character signedness and destination conversion at `0x3a`.
- Endianness, alignment, address mapping, and device-storage behavior.
- Signed overflow, signed shifts, and oversized shift counts.
- Relative evaluation order within the doubly sampled assignment.
- Ownership, bounds, aliasing, concurrency, and reentrancy.
- Invocation inventory and repeated-store multiplicity not enumerated here.

These unknowns do not weaken explicitly established cross-statement access ordering.

**[INFERENCE]** Masks may describe packed representations, and recurring offsets may describe structured storage. Neither interpretation is an acceptance requirement.

## Evidence

The supplied static representation supports the indirect invocation chain, retained-width transformations, partitioned 64-bit results, mixed-width pointer accesses, named-storage sampling, and terminal argument dependencies described here.

The opening region explicitly places the first Double scale clear after the unsigned-byte sample, the byte-derived word store after that clear, the Opening word capture after that store, and the second Double scale clear after the capture. This is an established ordering constraint, not inferred intent.

Explicit pointer types support access-width claims. Missing named-storage declarations do not support equivalent width claims for named reads or assignments. Separate reads at `0x67` establish distinct sampling events, while the retained capture from `0xa9` remains a separate value.

No dependency bodies, independent instruction validation, or runtime observations were available. **No runtime tests have been performed.**
