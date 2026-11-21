<!-- Trusted provenance: F-d45b2821efbe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d45b2821efbe

## Purpose

F-d45b2821efbe performs a long sequence of opaque calls, fixed-width arithmetic, word transfers, and lookup-directed byte writes. Its application-level purpose is not established. The supplied evidence does not justify describing it as a parser, encoder, allocator, initializer, or cleanup operation.

The observable behavior depends on more than its two explicit inputs. Incoming machine state, primary and secondary call results, global reads, and memory reached through calculated addresses all contribute.

This contract describes the supplied static representation, conditional on execution reaching each operation. It does not establish that the represented path is executable in an ordinary process environment. In particular, the path includes low absolute addresses, an incoming value without a visible assignment, unreliable control-flow recovery, and an invalid-instruction endpoint.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| First input | 32 bits | No explicit consumption appears in the supplied body. |
| Second input | 32 bits | Supplied to the opening unresolved call and later written as a complete word to a result-derived address. |

Both inputs are opaque bit patterns. No signedness, pointer meaning, length unit, enumeration, or permitted range is established.

The first input’s lack of an explicit read does not establish independence from its machine-level argument location. The calling convention and implicit inputs are incompletely recovered.

### Implicit inputs

The visible path also depends on:

- Entry machine state determining the opening call target.
- An incoming 32-bit value used in shifts and lookup addresses without a visible assignment.
- Opaque primary results, secondary results, and paired results.
- The word at absolute address 0x28.
- A separate global word whose address is not supplied.
- Bytes at address 0x1 and at calculated lookup addresses.
- Words at calculated addresses, including offsets 0x4, 0x8, 0xC, 0x14, and 0x1C.
- Any dependency effects on memory or relevant machine state.

The two explicit inputs therefore do not determine the trace.

### Outputs, completion, and errors

The displayed interface declares no return value. Paired call results are intermediate values, not established outputs of F-d45b2821efbe.

Normal completion is not established. The supplied representation stops at an invalid-instruction boundary after a final opaque call. This endpoint does not establish an intentional trap, application exception, successful return, or error-reporting API.

No local validation, error status, rollback, or recovery path is visible. Whether an invalid address or call target faults, and how such a fault is handled, remains **documented_only**.

### Preconditions and ownership

Continuation through any displayed access requires the relevant memory to support that access width. Continuation through a call requires an executable target and compatible calling convention. These requirements include low absolute addresses and destinations constructed directly from unsigned bytes.

No allocation, ownership transfer, retention, release, or lifetime guarantee is established. Pointer-like results must not be treated as owned objects merely because subsequent operations dereference them. Dependencies are not assumed to preserve global memory or to own or release storage.

## Behavior

### Interpretation and arithmetic

The representation primarily uses 32-bit quantities and addresses, 8-bit bytes, and one 16-bit store. Some calls produce a represented 64-bit result divided into lower and upper 32-bit components.

Unsigned 32-bit right shifts introduce zeros. Unsigned left shifts, multiplication, and subtraction retain their results modulo \(2^{32}\). Byte stores retain eight bits; the halfword store retains sixteen.

Some address expressions pass through signed 32-bit intermediates. Conclusions requiring their overflow to truncate are conditional on ordinary 32-bit machine arithmetic; the representation does not establish portable signed-overflow semantics.

Memory byte order and alignment requirements are unknown. Overlapping accesses must retain their stated widths rather than being interpreted through an assumed byte order.

No visible loop or conditional branch appears in the displayed path. This does not establish complete control flow or normal return from dependencies.

### Opening word transfers

The second explicit input is passed to an unresolved target derived from entry machine state. Several opaque calls follow before a primary result controls a word store.

For that result \(r\), the store writes the complete second input at address \(4r+0x14\).

A later result \(s\) controls two different accesses:

- A word store at \(16s+0x14\), containing \(4s\).
- A subsequent word read at \(4s+0x14\).

These locations are not interchangeable. Their expressions do not establish a common record or table abstraction.

The loaded word is retained across additional calls. Its logical right shift by eleven is stored at another result-derived word destination. Its logical right shift by twenty-two is also used directly to form a word-read address by adding 0x14. No intervening table-pointer dereference is visible.

### Lookup-directed byte writes

A recurring effect consists of an unsigned byte lookup followed by a byte store at the lookup result plus 0x11.

Under the literal address interpretation, the destination lies between 0x11 and 0x110 inclusive. No object base is visibly added.

Lookup addresses include values scaled by four, eight, sixteen, 0x40, 0x400, 0x1000, and larger powers of two, as well as values shifted right and incremented by one. Some expressions first mask a value. Several reduce to absolute address 0x1.

Stored bytes include:

- Zero.
- Low bytes of scaled primary results.
- Low bytes of loaded words.
- Selected high bits of 32-bit quantities.
- Bits selected from a represented 64-bit result.

Each lookup is a separate read. Repeated address expressions do not authorize reuse of an earlier byte, especially across opaque calls or overlapping writes. A zero store and a later nonzero store need not share a destination even when their lookup expressions match.

For each individual lookup-directed write, the lookup supplies the destination before the associated byte store occurs. This does not impose an evaluation order on unrelated argument expressions elsewhere.

### Bit-field transformations

The visible arithmetic includes masks retaining 29, 30, or 22 low bits. Truncation on a later byte store is significant:

| Transformation before byte storage | Original bits retained in the stored byte |
|---|---|
| Logical right shift by 22 | 22–29 |
| Retain 29 low bits, then right shift by 19 | 19–26 |
| Retain 30 low bits, then right shift by 20 | 20–27 |
| Retain 22 low bits, then right shift by 12 | 12–19 |

A logical right shift by nine followed by a 32-bit left shift by twenty-five retains only original bits 9–15, placed in result bits 25–31. Such compounded shifts are not reversible.

One byte value is obtained by shifting a represented unsigned 64-bit result right by 54 and then truncating. The stored byte selects original bits 54–61.

No domain names or packed-data format are established for these slices.

### Direct globals and the three reads at 0x28

An opaque primary result is stored as a word at absolute address 0x7C.

Later, the path performs three distinct word reads from absolute address 0x28:

1. A read captures a retained snapshot.
2. Two further reads appear as separate argument expressions of one opaque call, one per argument.
3. After that call, another opaque call occurs.
4. The retained snapshot is then stored at four times a secondary result plus 0x14.

The snapshot read precedes the two argument reads. The supplied representation does **not** establish which of the two argument expressions is evaluated first.

The two argument reads must not be replaced with the retained snapshot or collapsed into a single read. Nor does the evidence guarantee that the argument values equal each other or equal the snapshot. Whether repeated reads can observe different values depends on the memory environment and access semantics.

The final word store uses the originally retained value, not a fresh read of address 0x28. Intervening dependency changes to that global do not change the represented snapshot.

### Other offset-based accesses

One paired result supplies an upper-component base for a word read at offset 0x8 followed by a word read at offset 0xC. The first word later contributes a stored byte; the second contributes a lookup address. Calls intervene before those retained values are consumed.

Other result-derived bases supply word reads at offset 0x4. Address formation at offset 0xC must be distinguished from a dereference: some calls receive an address advanced by 0xC, while a later word read uses offset 0x4 from such an advanced address.

A separate paired result controls a byte store at its upper component plus 0x1C. The stored byte comes from its lower component. This is not one of the late word reads at offset 0x1C described below.

These relationships establish individual accesses, not complete object layouts, allocation sizes, alignment guarantees, or ownership.

### Late global snapshots and their consumers

A late region contains a word store at four times a secondary call result plus 0x14. The stored word is a retained primary result. Immediately afterward, the representation reads the separate, unlocated global word twice.

For this contract only:

- \(A\) denotes the value from the first global read.
- \(B\) denotes the value from the second global read.
- \(S\) denotes four times \(B\), subject to the 32-bit arithmetic qualification above.

These are explanatory labels, not dependency identities or source identifiers.

The first read is retained without scaling. The second read independently supplies the fourfold-scaled quantity. No opaque call is displayed between these two reads, but they remain distinct accesses; equality is not an established requirement.

Subsequent calls intervene before either retained value is consumed:

- Calls occur, followed by a result-derived word store of the earlier primary result shifted right by twelve.
- More calls occur before a lookup at \(S+1\). Its associated byte store contains the low byte of four times a new primary result.
- An opaque call then receives \((A\mathbin{\&}0x3FFFFFFF)\) shifted right by ten.
- Further calls occur before a second lookup at \(S+1\). This lookup drives a byte store selecting bits 20–27 of the same new primary result.
- Additional calls and the use of that second lookup byte precede the offset-0x1C argument reads.

The two lookups at \(S+1\) are separate reads. Changes to the global after its two reads do not update \(A\), \(B\), or \(S\) in the represented dataflow. Changes to lookup memory can nevertheless affect the two lookup results independently.

### Global-derived repeated word reads at offset 0x1C

The fourfold-scaled second global value is subsequently shifted left by twenty-six. Let the resulting 32-bit address quantity be \(H\).

After the intervening calls and lookup-directed writes described above, one opaque call receives **two separate word reads at \(H+0x1C\)**, one as each argument expression.

Both reads depend on the second global read through its fourfold scaling and later left shift. They do not depend on the first global snapshot used for the masked-right-shifted call argument.

The relative evaluation order of the two word-read arguments is not established. Both must remain represented accesses; their values are not guaranteed equal. These are 32-bit reads, distinct from the earlier paired-result-directed byte store at offset 0x1C.

Under truncating 32-bit arithmetic:

\[
H=(B\mathbin{\&}0xF)\times 2^{28}.
\]

Only the low four bits of the second global read determine this base. The offset-0x1C reads therefore do not generally target absolute address 0x1C.

A further address quantity is \(H\) multiplied by 0x400. Under the same arithmetic assumption, it is zero because the combined scaling from \(B\) has factor \(2^{38}\).

After the repeated-read call, another call occurs before a lookup using that further quantity plus one and a zero byte store. More calls precede another lookup using the same expression and a byte store derived from a secondary result. Thus these lookup sources reduce to address 0x1 under the stated arithmetic model, but remain separate reads across calls.

### Zero-based writes and overlap

Elsewhere, an explicitly zero-valued intermediate controls a word destination through multiplication by eight. The word store starts at absolute address zero.

A later halfword destination is formed by masking and right shifting that same zero-valued intermediate. It also starts at zero and stores the low sixteen bits of an opaque result.

The halfword overlaps two bytes of the earlier word and includes address 0x1, which is repeatedly read as a byte lookup source. Calls occur between relevant operations, so the later lookup value depends on byte order and any intervening memory effects.

Another byte store uses a masked-and-shifted lower component of a paired result as its destination, plus a term already reduced to zero. That destination is not an ownership-bearing object reference established by this evidence.

### Repeated stores and final propagation

Two consecutive word stores target the same address at a shifted base plus 0x34. The first stores a zero-extended lookup byte; the second stores that value shifted left by one.

A separate pair of consecutive word stores writes the same fourfold-scaled result twice at another base plus 0x34. Both accesses remain part of the displayed trace.

The final region propagates primary and secondary results through word destinations at offset 0x14, repeatedly distinguishing fourfold from sixteenfold address scaling. It also includes:

- Word stores derived from a paired result’s components.
- Large power-of-two scalings that discard high bits.
- Two separate word-read argument expressions at the same calculated offset-0x14 address, one shifted right by twelve and the other left by two.
- A later retained word read whose right-shifted value is stored after intervening calls.
- Further lookup-directed byte writes before the unresolved endpoint.

No common container abstraction, write coalescing, or successful completion follows from these patterns.

## Side effects

The established effect surface includes opaque calls; byte and word reads and writes; one halfword write; direct accesses at 0x0, 0x1, 0x28, and 0x7C; two reads of an unlocated global word; and repeated accesses to identical calculated addresses.

The ordering distinctions above are part of the static contract. They are not a guarantee of synchronization, atomicity, or a particular hardware memory model.

Purity, reentrancy, thread safety, transactionality, locking, barriers, and rollback are not established. Dependencies may introduce additional effects. No global-preservation guarantee is supplied.

## Dependencies

The mapped dependencies appear in this relative order, with unmapped calls and memory effects between them:

| Dependency | Visible argument relationship |
|---|---|
| F-23325cd5e50c | One 32-bit value obtained from an earlier opaque call. |
| F-bbb54b5f036d | The same retained 32-bit value in two positions. |
| F-f80b3ac065fc | Lower and upper components of a paired result, then two further repetitions of the lower component. |
| F-de15789688a8 | The same retained 32-bit value in two positions. |
| F-ea83f9a17404 | The same retained 32-bit value in two positions. |
| F-674e75a953e8 | The same address-like 32-bit value in two positions. |

These are argument relationships, not semantic contracts. Equal retained arguments differ from separate memory-read expressions that happen to use the same address.

Many calls have no supplied opaque ID. Their identities, full interfaces, secondary-result conventions, memory effects, ownership behavior, and completion behavior remain **documented_only**. They must not be equated with mapped dependencies merely because their arguments resemble one another.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Complete input transfer.** If the opening destination-controlling result is 3, the word store targets 0x20 and writes all 32 bits of the second input.

2. **Different scaling.** For result 2, a store at sixteen times the result plus 0x14 targets 0x34; a read at four times the result plus 0x14 targets 0x1C.

3. **Lookup destination and truncation.** A lookup byte of 0xFE selects destination 0x10F. A store value of 0x1234 writes byte 0x34.

4. **High-bit truncation.** If a logical right shift by twenty-two produces 0x3FF, byte storage writes 0xFF.

5. **Three reads at 0x28.** If the snapshot read observes 0x11223344, the later store writes that word even if either subsequent argument read observes another value. Acceptance requires two argument-read events, without imposing left-to-right evaluation.

6. **Independent global samples.** If the first unlocated-global read observes 0x400 and the second observes 3, the masked-right-shifted argument is 1, while both initial global-derived lookup addresses are 0xD. Using the first sample for the scaling is incorrect.

7. **Global-derived offset-0x1C reads.** With second global sample 3 and truncating 32-bit arithmetic, both word-read argument expressions address 0x3000001C. Two reads are required; their evaluation order is unresolved.

8. **Scaling collapse.** Under that same arithmetic model, multiplying the shifted global-derived base by 0x400 produces zero. The subsequent lookup source is 0x1, not 0x30000001.

9. **Repeated stores.** For lookup value 0x80, the first offset-0x34 pair writes 0x00000080 followed by 0x00000100. The final value alone does not demonstrate both accesses.

10. **Overlap and mutation.** The word and halfword stores both start at zero. A later read at 0x1 must observe memory at that point, subject to byte order and intervening effects, rather than reuse an earlier lookup.

11. **Completion.** Reaching the final opaque call does not establish a normal return or successful operation.

## Uncertainties

The following remain **documented_only**:

- Application purpose and any intended data format.
- Complete calling convention and implicit machine-state inputs.
- Address and access semantics of the unlocated global.
- Whether repeated reads can observe changing or device-backed storage.
- Byte order, alignment rules, fault handling, and memory-map validity.
- Signed-intermediate overflow behavior.
- Unmapped dependency identities and all dependency preservation guarantees.
- Allocation, ownership, and lifetime behavior.
- Completeness of the displayed control flow.
- Whether the invalid endpoint reflects executable behavior or incomplete recovery.

Packed-value manipulation is an **inferred intent**, supported by repeated masks, shifts, and byte truncation. It does not establish an encoding format or permit replacement of the recorded accesses with a higher-level transformation.

## Evidence

The evidence is the supplied static body and dependency-ID mapping. It supports the displayed widths, arithmetic relationships, explicit memory-access multiplicity, retained-value dependencies, and relative sequencing described here.

The representation reports bad instruction data, removal of an unreachable region, and overlapping global symbols. Those limitations constrain whole-function conclusions and prevent treating the apparent straight-line path as a validated execution trace.

No dependency bodies, validated instruction-level reconstruction, runtime traces, or external behavioral documentation were supplied. No runtime tests have been performed.
