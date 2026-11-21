<!-- Trusted provenance: F-ebe1baaf24ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ebe1baaf24ed

## Purpose

F-ebe1baaf24ed is a state-mutating operation whose application-level purpose is **documented_only**. The supplied static representation establishes an ordered sequence of dependency invocations, byte-selector accesses, bit extractions, and memory writes. It does not establish an initializer, serializer, collection operation, or ownership-management API.

**[INFERENCE]** Repeated masks, shifts, and scaling suggest manipulation of encoded values. That observation does not establish tagged pointers, field meanings, or a common object layout.

This contract distinguishes:

- **Established static evidence:** accesses, widths, argument relationships, and sequencing explicitly represented in the supplied material.
- **[INFERENCE]:** arithmetic consequences conditional on the represented widths and stated execution assumptions.
- **documented_only:** behavior not established by the available evidence.

The representation reports invalid instruction data, omitted unreachable material, and overlapping global symbols. The behavior below describes the represented sequence conditionally; it does not prove that every region belongs to one executable path.

## Interface

### Explicit input

The represented interface accepts one opaque **32-bit argument**. Its signedness, interpretation, and valid domain are unknown.

The first represented dependency invocation receives this argument. No validation, normalization, or subsequent direct use of the argument is visible. Whether a dependency retains it or interprets it as an address is **documented_only**.

### Environmental inputs

Execution also depends on:

- Two incoming 32-bit execution-state values without visible initialization.
- Incoming control state used to construct the first indirect call target.
- Primary and secondary outputs of dependency calls.
- Memory at absolute low addresses and computed addresses.
- Separately represented reads of global storage.

These are environmental requirements, not established additional public parameters. Their calling convention is unknown.

### Outputs and completion

No explicit return value is represented. Observable results are dependency calls, memory reads, and memory writes.

Some dependency results are represented as 64-bit quantities whose low and high 32-bit halves are consumed separately. Whether these are genuine 64-bit return values or paired machine-level outputs is **documented_only**.

No normal return is visible. The endpoint marks failed instruction recovery, not a requirement that the real operation deliberately halt, trap, or run forever.

### Preconditions

For each reached operation to proceed, its call target must be executable and its addressed storage must support the represented access width. Multibyte accesses additionally require adequate alignment or platform support for unaligned access.

No address, length, null, alignment, or call-result validation is visible. Zero as the explicit argument does not establish a bypass.

The low-address accesses must not be silently replaced with null checks or ordinary object-relative accesses. Their validity in the original environment is unresolved.

## Behavior

### Widths and addressing

Selectors are unsigned **8-bit reads**. A selector used as a destination address is converted to its numeric value and increased by `0x11`, producing an address from `0x11` through `0x110`, inclusive.

Each such update comprises a source-byte read followed by a byte store to the selected address. The selector is not a full-width pointer. Repeated selector reads remain separate accesses, including when their source addresses are identical.

Byte stores retain the low eight bits of their payload. The visible halfword store retains the low 16 bits. Word accesses transfer 32 bits. Memory byte order is **documented_only**.

For represented unsigned 32-bit arithmetic, retained results discard bits above bit 31. Signed-looking scaling and shifts also occur; their overflow behavior cannot be established merely from the recovered expression types.

| Transformation before a byte store | Original bits retained |
|---|---|
| Unsigned 32-bit value shifted right 22 | 22–29 |
| Mask with `0x3fffffff`, then shift right 20 | 20–27 |
| Mask with `0x3fffff`, then shift right 12 | 12–19 |
| Mask with `0x7fffffff`, then shift right 21 | 21–28 |
| Unsigned 64-bit value shifted right 54 | 54–61 |

Offsets `0x04`, `0x08`, `0x0c`, `0x14`, `0x1c`, and `0x34` occur on differing bases. Their repetition does not prove a shared structure layout.

### Entry and initial selector updates

The first indirect dependency receives the explicit argument. Further indirect calls occur before the first memory update; one receives an incoming execution-state value shifted right 11.

The first selector is read at one incoming value plus one. Its selected destination receives a byte extracted from the other incoming value after masking away its top bit and shifting right 21.

Subsequent calls receive scaled call results and the captured selector. Their effects are unknown.

An explicitly zero-derived source calculation later reads a selector at absolute address `0x01` and clears its selected destination. After intervening calls, another selector read at `0x01` determines a destination receiving bits 22–29 of an earlier unsigned call result. The second read is not reuse of the first selector.

### First mapped dependency and publication

A later 32-bit result is passed once to **F-23325cd5e50c**.

Following calls and accesses include:

- Clearing a destination selected through another call result.
- Reading a selector at an earlier result multiplied by 16, plus one.
- Storing bits 20–27 of that earlier result.
- Passing the selector-minus-result difference in two argument positions.
- Additional selector clears and extracted-byte stores using shifted values and secondary call outputs.

Several address calculations remain zero-derived in the visible data flow and consequently read selectors at `0x01`. Each occurrence is a distinct represented access.

A later indirect result is written as a 32-bit word to absolute address **`0x7c`** before the next represented call receives that captured result in two argument positions. The arguments are not rereads of `0x7c`. No restoration of the previous word is visible.

### Captured fields and three distinct accesses at `0x28`

After publication, further calls and selector writes precede a 32-bit read at a secondary output plus `0x14`. This captured word survives across later calls and supplies shifted arguments and a later payload shifted left 11. Those uses do not reread its original source.

The global access at absolute address **`0x28`** has a different, important access pattern:

1. A 32-bit read captures the global value for later storage.
2. A following indirect invocation has two argument expressions, each separately reading the global at `0x28`.
3. After that invocation and another intervening call, the initially captured word is stored at a destination formed from a secondary output multiplied by four, plus `0x14`.

The capture precedes both argument reads. The evaluation order **between the two argument-read expressions is unspecified** by the supplied representation. Both supply their own values to the invocation; neither is established as reuse of the capture.

No equality among these three read results is guaranteed without additional memory semantics. The later store uses only the initial capture, regardless of values supplied by the two argument reads. Unknown intervening dependencies may modify either source or destination storage.

Further selector updates use scaled results. One unsigned value is masked to 30 bits, shifted right seven, and multiplied by `0x2000000`. **[INFERENCE]** In a retained 32-bit unsigned result, only the lowest seven bits of the value entering that multiplication can contribute.

### Paired outputs and middle dependency sequence

A paired result supplies a retained low 32-bit value and a high 32-bit address base. The operation reads a word at that base plus `0x08`, then a word at that base plus `0x0c`. The latter word supplies the source address for a later selector read at that value plus one; the selected destination receives the low byte of the former word.

A later word read at the retained low-half value plus `0x04` provides another byte payload. Its selected destination is derived from a selector read using that earlier low-half value shifted right nine, plus one.

The mapped call sequence then includes:

1. **F-bbb54b5f036d**, receiving one captured 32-bit value twice.
2. An unresolved direct dependency, receiving a paired result as low half, high half, low half, low half.
3. **F-f80b3ac065fc**, receiving that same four-position arrangement from a later paired result.

Calls and selector accesses separate these invocations. They are not adjacent or independent operations.

Later calls receive explicit zero-derived values in multiple argument positions. Another invocation receives zero in its first position. Recovery warnings limit confidence in the control-flow connection around this region.

### Low-address stores and later mapped calls

A subsequent paired result causes a byte store at its high half plus `0x1c`, carrying the low byte of its low half.

Later, an explicit zero assignment determines two overlapping destinations:

- A 32-bit call result is stored at absolute address **`0x00`**.
- After calls and selector updates, the low 16 bits of another result are stored at absolute address **`0x00`**.

The word store precedes the halfword store. They are not interchangeable with one final write. A call also contains two separately represented byte-global argument reads at `0x01`; repetition of the address does not establish equal argument values or an evaluation order between those expressions.

Further operations include a direct byte store using a masked, shifted portion of an earlier paired result as its destination; an adjustment of a later result by `0x0c`; a word read at the adjusted address plus `0x04`; and a difference involving heavily scaled call outputs.

The later mapped dependency order is:

1. **F-de15789688a8**, with one captured value twice.
2. An unresolved direct dependency, with another captured value twice.
3. **F-ea83f9a17404**, with another captured value twice.

No explicit error checks follow these mapped calls.

### Repeated stores and field-building accesses

One region performs two consecutive 32-bit writes to the same destination formed from a value shifted right nine, plus `0x34`. The first payload is a zero-extended selector byte; the second is that captured byte shifted left one.

A later pair writes the same shifted call result twice to another destination at offset `0x34`. Both stores are separately represented.

The subsequent region repeatedly uses distinct word-store address forms:

- A base multiplied by four, plus `0x14`.
- A base multiplied by 16, plus `0x14`.

Payloads include captured call outputs, scaled bases, secondary outputs, and extracted word portions. The two addressing forms must not be merged into one layout rule.

### Two separate reads of the later unnamed global

After a call-derived word has been stored at a computed `0x14` destination, the representation reads another global **twice in order**, before the following dependency-call sequence:

- The **first 32-bit read** is captured for a later argument masked with `0x3fffffff` and shifted right 10.
- The **second 32-bit read** is multiplied by four, and that scaled result is retained separately.

No dependency call is represented between these two reads. That does not establish that they agree: storage semantics, asynchronous mutation, and the actual memory map remain unknown.

The second read’s scaled result supplies a selector-source address at the scaled value plus one. After intervening calls, the first read supplies its masked right-shift argument. A later selector read again uses the second read’s retained scaled result plus one. Its payload is extracted from a call result, not from the first global capture.

The second read’s retained scaled value is subsequently shifted left 26 to form the base for **two separately represented 32-bit argument reads at offset `0x1c`**. Evaluation order between those argument expressions is unspecified, and their values are not guaranteed equal. The operation also further scales that shifted base by `0x400` for later selector addresses.

**[INFERENCE]** Under 32-bit retained-bit arithmetic, multiplying the second read by four and then shifting left 26 leaves only its original lowest four bits contributing to that base. Scaling that result by `0x400` yields zero modulo \(2^{32}\). Because the recovered expressions include signed-looking operations, these reductions are conditional arithmetic deductions, not unconditional guarantees of executable address behavior.

Neither global read may be replaced with the other. The first read drives the masked right-shift argument; the second drives selector addressing and the subsequent scaling leading to the `0x1c` accesses.

### Final represented region

**F-674e75a953e8** receives the same captured 32-bit value twice. Further computed `0x14` stores follow, including a zero payload derived from an explicit zero assignment.

The remaining region alternates calls, scaled stores, scaled arguments, and word reads. One call contains two separately represented reads of the same computed `0x14` location: one supplies a right-shifted argument and the other a left-shifted argument. Their evaluation order and equality are not established.

Elsewhere, a word is explicitly captured from a computed `0x14` location before intervening calls and later stored shifted right 12. That capture-and-use relationship is distinct from repeated argument reads.

A late three-argument invocation receives a six-bit masked fragment shifted left 14, a separately scaled value, and a value shifted left 30. Additional stores, selector-based byte updates, and indirect calls follow before recovery stops.

## Side effects

### Memory and ordering

Established direct effects include:

- Unsigned byte-selector reads and byte writes.
- One visible 16-bit store.
- Numerous 32-bit reads and stores.
- Publication of a word at `0x7c`.
- Three separately represented word reads at `0x28`.
- Repeated byte reads at `0x01`.
- Overlapping word and halfword stores at `0x00`.
- Two ordered reads of the later unnamed global.
- Repeated stores to identical computed `0x34` destinations.
- Repeated argument-read expressions at computed word locations.

Ordering follows the represented sequence except where evaluation order within a call’s argument expressions is unspecified. Dependencies may observe or alter memory between direct accesses.

There is no established atomicity, rollback, isolation, synchronization, or global-preservation guarantee. A final-memory snapshot alone does not capture the represented behavior.

### Ownership and lifetime

No ownership transfer, allocation, retention, borrowing, release, or destruction is established by the visible body. Unknown callees may perform such operations. Passing a value does not establish who owns any storage it might designate.

Captured values are reused across calls in the representation. This does not prove that pointed-to storage remains alive or unchanged.

### Errors

No explicit error return, recovery branch, or cleanup path is visible. An invalid target or inaccessible address may prevent further progress; the platform’s response is **documented_only**.

Earlier writes are not visibly undone after a later failure.

## Dependencies

Mapped direct dependencies occur once each in this represented order:

| Opaque function ID | Represented argument relationship |
|---|---|
| F-23325cd5e50c | One 32-bit value |
| F-bbb54b5f036d | One captured 32-bit value twice |
| F-f80b3ac065fc | Low half, high half, low half, low half |
| F-de15789688a8 | One captured 32-bit value twice |
| F-ea83f9a17404 | One captured 32-bit value twice |
| F-674e75a953e8 | One captured 32-bit value twice |

Numerous indirect dependencies and two other direct dependencies lack supplied opaque IDs. Their identities, contracts, and possible equivalence are **documented_only**; no IDs are invented.

Unused return values do not make calls dispensable. Termination, secondary outputs, global changes, and memory-lifetime effects remain unknown.

## Acceptance examples

These examples are conditional specifications, not executed tests.

1. **Selector boundary.** A reached selector value of `0xff` selects byte destination `0x110`, not a negative address.

2. **Byte truncation.** An unsigned source of `0xffc00000`, shifted right 22, produces `0x3ff`; a subsequent byte store carries `0xff`.

3. **Fresh selector access.** If storage at `0x01` changes between selector reads, the associated writes may select different destinations. Reusing the first selector would lose a represented access.

4. **Publication.** A captured result of `0x12345678` is written to `0x7c` before the following call receives that captured value twice. A later change to `0x7c` does not retroactively change those captured arguments.

5. **Three reads at `0x28`.** Suppose the capture returns `0x11111111`, and the two argument expressions return `0x22222222` and `0x33333333` in their respective argument positions. Subject to continued execution, the later computed `0x14` store carries `0x11111111`. The example does not prescribe which argument expression was evaluated first.

6. **Separate later global captures.** Suppose the first unnamed-global read returns `0x400`, while the second returns `0x800`. The first supplies a later masked right-shift argument of one. The second supplies a scaled value of `0x2000`, used for selector-source addresses at `0x2001`, conditional on valid address arithmetic. Substituting either capture for the other is incorrect.

7. **Overlapping stores.** Reaching the zero-derived destinations requires a word write to `0x00` before the later halfword write there. Final bytes depend on byte order and intervening effects.

8. **Distinct repeated payloads.** If the captured selector byte in the first `0x34` pair is seven, the consecutive word payloads are seven and fourteen at the same address.

9. **Identical repeated payloads.** The later identical `0x34` pair still contains two stores. An observer of individual accesses may distinguish it from one store.

10. **Repeated word arguments.** If the two represented `0x1c` argument reads observe different words, the invocation receives those distinct observations. Equality is conditional on additional memory semantics.

11. **Unknown continuation.** Successful execution of all visible operations does not establish a normal return beyond the recovery boundary.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid explicit-input domain.
- Complete calling convention and incoming-state requirements.
- Unresolved dependency identities and contracts.
- Genuine return widths versus paired machine outputs.
- Memory map, byte order, alignment, and access-observation semantics.
- Equality of repeated global or computed-address reads.
- Signed-looking arithmetic overflow and executable address formation.
- Aliasing between globals, computed destinations, and callee-managed storage.
- Ownership, lifetime, allocation, and release effects of dependencies.
- Validity of absolute low-address operations.
- Control flow affected by recovery failures.
- Continuation and failure behavior beyond the visible endpoint.

## Evidence

This contract uses the supplied static representation and opaque dependency mapping only. Prior prose and reviewer feedback are not independent behavioral evidence.

The representation separately establishes the initial capture and two argument reads at `0x28`. It also separately establishes the two ordered reads of the later unnamed global and their different downstream uses.

Width, extraction, and retained-bit deductions are conditional on the stated arithmetic assumptions. Recovery warnings limit confidence in executable control flow and symbol interpretation; they do not themselves specify runtime behavior.

**No runtime tests have been performed.**
