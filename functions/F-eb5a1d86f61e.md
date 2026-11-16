<!-- Trusted provenance: F-eb5a1d86f61e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb5a1d86f61e

## Purpose

F-eb5a1d86f61e invokes a long sequence of unresolved dependencies, transforms dependency-produced values, performs ordered memory mutations through calculated and fixed addresses, and finishes with an unresolved four-argument invocation.

**Established static evidence:** the supplied representation shows no explicit parameters, no explicit returned value, and no explicit conditional branches or loops. Its observable behavior includes dependency invocations, argument values, memory reads, memory writes, and their ordering.

**[INFERENCE] Intent:** the shifts, masks, and offset-based accesses may manipulate encoded state. They do not establish an application purpose, object model, serialization format, container operation, or output protocol.

This contract describes established relationships without assigning meanings to unknown dependencies or storage. It does not promise that the calculated addresses are valid in an ordinary process or that execution reaches the final invocation.

## Interface

### Inputs and outputs

- **Explicit inputs:** none shown.
- **Implicit inputs:** incoming execution context, indirect invocation targets, dependency results, secondary execution-state outputs, fixed-address storage, and memory reachable through derived addresses.
- **Explicit output:** none shown.
- **Observable outputs:** the dependency-invocation trace and direct memory-access trace.
- **Completion:** a return is represented after the final invocation, conditional on every preceding operation completing and that invocation returning.

The first indirect target depends on incoming return-link execution state plus `0xFFE`. Most subsequent targets are formed from embedded execution-location references plus the same displacement. Thus, an explicit-parameter-free interface does not imply independence from the caller or execution environment.

### Widths and arithmetic

| Quantity | Established interpretation |
|---|---|
| Byte access | 8 bits |
| Halfword store | 16 bits |
| Word access | 32 bits |
| Paired dependency result | 64-bit quantity consumed as low and high 32-bit portions |
| Address calculations | Represented using 32-bit integer-like quantities |
| Unsigned right shift | Logical, with zero fill |
| Byte conversion | Retains the low eight bits |
| Byte promoted for word storage | Zero-extended to 32 bits |

Extracting low and high portions of a paired result establishes numeric portion roles, not the byte order of a stored 64-bit object.

Some arithmetic is represented with signed-looking operands, including left shifts, multiplication, subtraction, and a final variable right shift. Target overflow, address wrapping, and exceptional shift semantics remain **documented_only**. Arithmetic examples below explicitly state when they assume truncating 32-bit bit-pattern semantics.

### Preconditions and ownership

Successful execution requires executable indirect targets, compatible calling conventions, usable dependency outputs, and supported memory accesses at every reached address and width. Fixed low addresses and unaligned accesses must be meaningful in the actual environment.

These are execution requirements, not checks performed by this function. No explicit null, bounds, alignment, allocation-success, or target-validity checks are shown.

No allocation, release, or ownership-transfer protocol is established. A loaded word used as an address is not proven to identify a live, owned object. Unknown dependencies may retain, allocate, release, or mutate memory; no preservation or lifetime guarantee is available.

## Behavior

### Dependency phase

An extensive sequence of indirect invocations precedes the first explicit direct-memory store. Invocation order matters even where a result is ignored or no explicit arguments are displayed. Neither case establishes absence of side effects or absence of implicit execution-state inputs.

Established argument relationships include:

- Left-shifting dependency-produced words by eight bits.
- Passing a paired result’s shifted low portion and unchanged high portion onward.
- Passing separately shifted versions of either portion.
- Supplying the same transformed value in multiple argument positions.
- Retaining values across intervening invocations for later consumption.
- Supplying explicit zeros, including zeros obtained by shifting a value explicitly set to zero.
- Consuming secondary execution-state values even where no ordinary result assignment is shown.

The transformations include masks and shifts across many field widths. Relevant boundaries include:

- A 24-bit mask followed by a six-bit left shift yields at most a 30-bit result.
- A 22-bit mask followed by an eight-bit left shift also yields at most a 30-bit result.
- Selecting bits 10–19 yields a ten-bit unsigned field before further scaling.
- Right-shifting a 64-bit result by 34, 36, or 44 bits is not equivalent to narrowing it to 32 bits first.
- A left shift followed by a right shift may discard information rather than recover the original word.

A wider mathematical intermediate is not automatically equivalent to the represented 32-bit calculation. Changing intermediate width can change arguments and later addresses.

No recognizable success or error-code test guards consumption of dependency results. Results are transformed or used directly. The complete call-by-call argument ledger is **documented_only** rather than enumerated here.

### Initial direct-memory effects

For this section, **terminal low** and **terminal high** denote the low and high 32-bit portions of the last paired dependency result before direct stores begin. These are descriptive roles, not source identifiers or object types.

The opening store order is established:

1. A word store at terminal high plus `0x14`.
2. A byte store at a retained shifted base plus `9`, containing the low byte of terminal high.
3. A word store at terminal low multiplied by 256, plus `0x54`.
4. A second byte store at that same retained base plus `9`, containing the low byte of another retained value.

The first byte store cannot be discarded merely because the second uses the same displayed destination. The intervening word store, aliases, or external observation may distinguish them.

Another dependency-produced word multiplied by `0x02000000` supplies a **scaled base**. A separate **small base** is obtained by left-shifting another result by eight bits, masking to fifteen bits, logically shifting right by six, and doubling.

Under truncating 32-bit bit-pattern semantics, the small base is a multiple of eight from zero through 1,016. This range does not establish address validity.

The next access group clears the scaled base’s word at `0x14`, reads its byte at `5`, clears the small base’s word at `0x14`, and stores the zero-extended byte at the scaled base plus `0x34`. A subsequent load rereads the scaled base’s `0x14` word.

That reread is not replaceable by the earlier zero without a non-aliasing and stability guarantee. An intervening clear through the small base may overlap it. Further clears and repeated byte writes occur before later address-bearing loads.

### Address-bearing words and overlapping accesses

Later memory operations use loaded words from offsets including `0x14`, `0x54`, `0x64`, and `0x74` as address bases or stored values. Retained quantities are also multiplied by two, four, or eight.

The observed relative-access vocabulary includes:

| Relative offset | Observed access widths and uses |
|---|---|
| `0` | Byte, halfword, or word stores at selected scaled addresses |
| `1` | Byte write |
| `4` | **Byte read and word reads** |
| `5` | Byte reads |
| `8`, `9`, `0x0D`, `0x11`, `0x15`, `0x19` | Byte writes |
| `0x10` | Byte write and word write |
| `0x14`, `0x20`, `0x24`, `0x34`, `0x40`, `0x44`, `0x54`, `0x64`, `0x74` | Word reads and/or writes |

The byte read at offset `4` supplies a zero-extended value during the later `0x20`-field overwrite sequence. Other offset-`4` accesses read complete words used subsequently as values or addresses. These are distinct width obligations.

The table does not describe one established structure. Byte writes at `0x15` overlap a word beginning at `0x14`; byte writes at `0x11` overlap a word beginning at `0x10`. Their numeric effect on a later word depends on unestablished byte order.

Some fields receive their own derived base value. This does not establish linked-list initialization or any other container invariant.

Repeated address-bearing reads remain distinct. One displayed operation uses separate reads of a `0x74` word for the destination base and stored value. Their multiplicity is established, but their relative evaluation order within that compound operation is not independently established. A single cached read would assume stability not supplied by the evidence.

### Constant stores and repeated mutation group

A later group stores numeric `0x3A` at three widths: a byte at a four-times-scaled address, a halfword at the corresponding eight-times-scaled address, and a word at the four-times-scaled address plus `0x20`.

The retained quantity used for a separate destination is reduced by `0x96` **after the byte store and before the halfword and word stores**. No underflow or range check is shown.

At four times this adjusted quantity, the `0x20` word field is overwritten with:

- `0x20`;
- `0x2F`;
- the zero-extended byte freshly read from that base plus `4`;
- `0x20` again.

The third value requires an **8-bit read**, not a word read. All four stores remain part of the visible trace. Numeric constants alone do not establish text output, encoding, stream append, or string termination.

Immediately after this overwrite sequence, the function **reads a separate 32-bit base from fixed storage at `0x087FBBBC` before writing the word `0x20` to fixed address `0x60`**.

The value obtained by that earlier read is the **captured base**. The following mutation region uses that captured value; no reload from the fixed storage is shown between its capture and those uses. Unknown storage relationships do not justify moving the read after the `0x60` write or replacing the captured value with a later storage value.

The repeated mutation region has a four-store ordering:

1. A word write at an eight-times-scaled destination plus `0x54`.
2. A byte write at the captured base plus `0x19`.
3. A byte write at the adjusted quantity’s four-times-scaled base plus `0x11`.
4. A word write at the captured base plus `0x34`.

The byte values retain the low eight bits of the four-times-adjusted base and captured base, respectively.

The first group uses an earlier retained quantity’s eight-times-scaled value for both its destination relationship and its final stored field value. Subsequent repeated groups use the adjusted quantity’s eight-times-scaled value. The captured base remains the one obtained before the `0x60` write throughout this region.

The source explicitly repeats these accesses. They cannot be collapsed to final memory contents without additional guarantees about aliases, ordinary-memory behavior, and observers. The exact repetition count is **documented_only** in this contract; it is not a license to substitute an arbitrary count.

### Final mutations and dispatch

After the repeated region, the function reads a word from fixed address `0xA9` and words through derived addresses, then performs further interleaved byte and word mutations.

Established final relationships include:

- A byte derived from eight times the adjusted quantity plus `0x30` is written to fixed address `0x3A`.
- Numeric `0x6C` is written as a byte at a derived base plus `8`.
- The low byte of the fixed `0xA9` word is written to fixed byte address `0x67`.
- A later transfer reads fixed byte `0x67` and writes fixed byte `0x21`.
- The byte at `0x67` is read again for later derived-address stores; those reads are not replaced by an assumed unchanged retained byte.
- A word read from a derived base plus `0x40` is subsequently stored at that base plus `0x10`, after an earlier byte write at `0x10`.
- A field at a captured-base-derived destination plus `0x54` receives multiple different word values.
- A halfword `0x65` is stored at twice a loaded value.
- A derived destination’s `0x54` word receives decimal `100`.
- A later word store targets twice another loaded value.

One byte store takes its value from a retained word right-shifted by a count obtained from the low eight bits of the four-times-adjusted base. Counts above 31 are possible from that expression. Both the target’s oversized-count behavior and the exact signed right-shift interpretation remain **documented_only**.

The final invocation receives four visible arguments:

1. `0x72`;
2. a recently loaded word obtained through a loaded base at offset `0x64`;
3. a word captured earlier in the final phase from another derived base plus `0x14`;
4. a word read later from the loaded base plus `0x74`.

These argument values retain their respective load origins. The target, dispatch interpretation, effects, and return behavior are unresolved.

## Side effects

### Ordering and aliasing

Direct accesses retain their established ordering. Different address expressions are not proven to identify disjoint storage.

Consequences include:

- Word writes can affect later byte reads.
- Byte writes can change later address-bearing words.
- Previously cleared storage need not remain zero.
- Identical repeated stores can remain observable.
- Values captured before a write remain captured values even if their source storage subsequently changes.
- Separate displayed reads cannot be merged solely because their address expressions match.

In particular, the captured-base read precedes the `0x60` word write. Subsequent stores through that base do not imply a new read of its fixed-storage source.

Fixed byte locations include `1`, `0x15`, `0x21`, `0x3A`, and `0x67`. Fixed word locations include `0x54`, `0x60`, and `0xA9`; the separate captured-base source is also read as a word. Fixed storage at address `5` is read and converted to a 32-bit quantity, but its original source-object width is not independently established.

The word access at `0xA9` is not naturally four-byte aligned. Ordinary execution therefore requires suitable alignment support or a different established memory interpretation.

### Errors, lifetime, and concurrency

No explicit error result, recovery path, rollback, or cleanup is shown. A faulting memory access or nonreturning invocation may prevent all later effects. Earlier effects are not visibly reversed.

Unknown dependencies may modify globals or reachable memory. No ownership, preservation, or release behavior is assumed.

Concurrency, synchronization, atomicity, reentrancy, and interrupt safety remain **documented_only**. No explicit synchronization operation is visible.

## Dependencies

F-eb5a1d86f61e is the only supplied opaque function ID. No opaque IDs are supplied for the indirect callees. Their identities must remain unresolved rather than being assigned invented helper names or conflated with this function.

| Visible dependency pattern | Established consumption |
|---|---|
| Result ignored | Invocation still occurs for unknown effects |
| One word | Shifted, masked, retained, stored, or used in addressing |
| Paired result | Low and high 32-bit portions consumed separately |
| Secondary execution-state output | Used in arguments without an ordinary result assignment |
| Final invocation | Receives `0x72` and three retained or loaded words |

No dependency is established as an allocator, destructor, formatter, I/O routine, or container operation. No recursive call to F-eb5a1d86f61e is established.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Truncating left shift

Given a consumed word of `0x12345678` and truncating 32-bit left-shift semantics, its eight-bit-left-shifted argument is `0x34567800`, not the original word or a widened 40-bit result.

### Paired-result portion roles

Given a paired result with low portion `1` and high portion `2`, an invocation consuming shifted-low, unchanged-high, and high shifted left by six receives `0x100`, `2`, and `0x80` in those positions.

### Unsigned field extraction

Given an all-ones word, selecting bits 10–19 yields `1,023`; shifting that field left by six yields `65,472`. Sign extension is not an equivalent extraction.

### Offset `4` byte-read width

Given that the byte at the adjusted four-times-scaled base plus `4` is `0xE7`, the associated `0x20` field receives word `0x000000E7` between the stores of `0x2F` and `0x20`.

Reading a complete word at offset `4` for this operation is not accepted. Separate word reads at offset `4` elsewhere must nevertheless retain their 32-bit width.

### Captured base survives source-storage changes

Given that the fixed-storage read captures `0x1000`, and the subsequent word write to `0x60` changes what a later read of that storage would return, the repeated region still uses captured base `0x1000`.

Subject to successful execution, its captured-base field destinations remain `0x1019` and `0x1034`. Reloading the changed storage or capturing only after the `0x60` write violates the established relationship.

### Intermediate overwrites

Given observation of the adjusted base’s `0x20` field, its store sequence includes `0x20`, `0x2F`, the freshly read and zero-extended byte, and `0x20`, in that order. Matching only the final value is insufficient for trace equivalence.

### Aliasing and rereads

Given an intervening store that overlaps a previously cleared word, a later load consumes the value present at that later read, not an assumed zero.

### Invalid addresses and terminal invocation

Given an unsupported derived address, no graceful failure or rollback is promised. Given successful arrival at the final invocation, its first argument is `0x72`; successful dispatch or normal return is not established.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid execution environment.
- Callee identities, prototypes, and secondary-output conventions.
- Overflow, address wrapping, signed shifts, and oversized variable-shift behavior.
- Byte order, unaligned access support, and memory-mapped device behavior.
- Fixed-storage interpretation and alias relationships.
- Object boundaries, lifetimes, ownership, and field meanings.
- Complete invocation ledger and exact repeated-region count, not enumerated here.
- Intra-expression read ordering where not independently specified.
- Final dispatch target, termination, and exception behavior.

The captured-base read-before-write relationship and the byte read at offset `4` are established, not uncertainties.

## Evidence

This contract is based solely on static inspection of the supplied representation. Prior prose, comments, labels, and reviewer statements are not independent behavioral authority.

The representation directly establishes the invocation and transformation patterns, access widths, overlapping offsets, fixed-address operations, repeated mutation groups, and terminal argument origins. It specifically establishes a word-sized base capture before the `0x60` store and an 8-bit offset-`4` read alongside separate 32-bit offset-`4` reads.

No dependency implementations, target instruction specification, memory map, or calling-convention reference were supplied.

**No runtime tests have been performed.** Acceptance examples state conditional arithmetic and ordering obligations, not observed execution results.
