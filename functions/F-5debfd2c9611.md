<!-- Trusted provenance: F-5debfd2c9611; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5debfd2c9611

## Purpose

F-5debfd2c9611 combines dependency invocations with calculated memory reads, byte and word propagation, shared-state updates, and a terminal invocation of address zero. Its domain purpose is not established.

This is a **partial static behavioral contract**, not a complete ordered interaction trace. It specifies the selected operations and ordering relationships below. The complete intervening invocation sequence, including numerous calls without explicit arguments, is not enumerated; that coverage limitation is `documented_only`. Such calls remain potentially observable and cannot be omitted, merged, or reordered merely because their individual contracts are unknown.

**Established static evidence:** two explicit 32-bit inputs; ambient execution-state inputs; calculated invocation targets; 8-, 16-, and 32-bit memory accesses; paired 64-bit results; shared storage that may overlap smaller symbols; and a terminal zero-address invocation.

**Inferred intent:** repeated scaling and extraction may belong to an encoded execution or data-management protocol. No such interpretation is required for acceptance.

**Unknown:** domain meaning, dependency contracts, and whether the represented entry and exit describe a conventional callable interface.

No runtime tests have been performed.

## Interface

### Inputs and return

The explicit interface takes two opaque 32-bit inputs. Neither is established as signed, address-bearing, a count, or a handle.

The first invocation receives four argument positions: first input, second input, second input, first input.

No explicit return value is represented. A displayed normal return follows the invocation of address zero and is reachable only if that invocation returns. Earlier calls or memory accesses may prevent reaching it.

### Ambient inputs

Additional behavior depends on:

- Entry execution state used to derive the first invocation target.
- An ambient address and ambient byte used in an early selector update.
- Primary dependency results and paired results with low and high 32-bit components.
- Call-associated register values consumed beyond explicitly declared results.
- Shared storage, absolute low-address memory, and calculated memory.
- The address of a local stack region exposed to a dependency.

Provenance and preservation of call-associated values are `documented_only`. They are not independently initialized formal parameters, and unrelated dependencies are not assumed to preserve them.

### Widths and address relationships

| Item | Represented width or relationship |
|---|---|
| Explicit inputs and ordinary words | 32 bits |
| Explicit unsigned-byte samples | 8 bits, widened to values from 0 through 255 |
| Byte stores | Low eight bits of the supplied value |
| Halfword sample and terminal clear | 16 bits each |
| Paired result | 64 bits, consumed as separate 32-bit components |
| Final composition | An 8-bit low component and a 24-bit upper component |

Some byte-like and halfword-like arguments have opaque types; their argument-extension rules are unresolved.

A recurring **selector-directed update** reads an unsigned byte at a specified base plus 1, then writes a byte to the absolute address equal to that selector plus 17. The destination is not the selector-read base plus 17. Its range is 17 through 272 inclusive.

Offsets 9, 13, 14, 20, 21, 36, 52, 68, and 116 also occur. These observations do not establish one coherent object layout.

Explicit unsigned masks and right shifts support the bounds stated below. Signed-looking shifts, multiplication, and address arithmetic do not establish portable overflow behavior. Modulo-\(2^{32}\) interpretation of those expressions is conditional, not a universal precondition silently imposed on every address formula.

### Preconditions and errors

No explicit validation, error result, or recovery interface is represented.

Each reached invocation requires a usable target and compatible execution state. Each reached memory access requires suitable mapping, permissions, and any target-specific alignment. These are conditions for execution to proceed, not checks performed by the function.

Fault delivery, exception propagation, and state following interruption are `documented_only`.

## Behavior

### Ordering and interference

The represented body contains no explicit branches or loops. Reaching a later direct operation depends on all preceding operations allowing continuation.

Dependencies are interaction boundaries. They may mutate memory, restore earlier values, retain addresses, fail, or not return. Separate reads remain separate observations even when their address formulas match.

The following sections preserve selected local ordering relationships, but do not enumerate the entire dependency trace. Unenumerated invocations remain part of the behavior; their number, precise placement within summarized intervals, and full argument/result relationships are outside this document’s coverage.

No explicit rollback is represented. This does not guarantee persistence of any write: later direct stores and unknown dependency effects may replace or restore values.

### Entry and early selector updates

After the input-forwarding invocation, a no-explicit-argument invocation supplies a paired result.

Before forwarding that result, the function samples the selector at the ambient address plus 1 and stores the ambient byte through it. The next invocation receives the paired high component twice, zero, and the paired low component logically shifted right by 19.

Four no-explicit-argument invocations follow that forwarding invocation; the fourth supplies a scalar result. The ambient selector is then sampled again, and its selected absolute byte is cleared. The scalar result, scaled by four, is forwarded afterward.

Later scalar results are also forwarded scaled by four, sometimes twice and sometimes with an additional zero.

An unsigned dependency result supplies a selector-read base equal to its logical right-by-18 extraction multiplied by 1024. **Before the first clear through this base, an invocation receives four copies of that unsigned result, followed by a no-explicit-argument invocation.** The selector-directed clear then occurs. A call-associated value is forwarded, four further no-explicit-argument invocations occur, and the selector is sampled again for a second clear. Consequently, the two clears may select different destinations.

Another region invokes a dependency with four argument positions supplied from the 32-bit contents of absolute address 48. These are memory values, not the literal integer 48. Later selector clears use a call-associated unsigned value logically shifted right by 9 as their base.

### Word slots and early propagation

Immediately before the first call-associated-index word store, a dependency is invoked with **four zero arguments**, followed by a no-explicit-argument invocation. The resulting call-associated index is stored as a 32-bit word at four times that index plus 20.

The next invocation receives that retained index and a logical right-by-19 extraction of a word read from a separately calculated base plus 20.

Later operations include:

- A result-selected four-times-scaled slot receiving the retained index.
- A destination at sixteen times a result plus 20 receiving four times that result.
- A word read at the four-times-scaled value plus 20, whose right-by-19 extraction is forwarded.
- Another four-times-scaled slot receiving the retained scaled value.

Four-times and sixteen-times destination scaling are distinct. Intervening dependencies may observe or change overlapping locations.

### Masked extraction and distinct paired results

An unsigned result supplies a four-times-scaled argument and a selector-directed byte store containing that scaled value’s low byte. After intervening calls, another selector-directed store receives the low byte of the same result with its top two bits cleared and then logically shifted right by 20.

Two retained extractions clear those same top two bits and shift right by 17 and by 7. A later invocation receives seven represented argument positions: four repetitions of a scalar dependency result, those two extractions, and an execution-sequence-derived value.

A paired result obtained next supplies an **earlier paired low value** and a high component. Selector clearing occurs before an invocation receiving the high component twice and a retained scalar.

The earlier paired low value supplies selector bases derived by shifts and, later, an unsigned-byte read at twice that low value. That byte is forwarded in four argument positions. These accesses belong to this earlier paired result.

**A different paired result subsequently replaces the retained low value.** Its high component is forwarded separately. The **newer paired low value**, not the earlier one used for the twice-value byte read, supplies:

- A selector-read base equal to the newer low value itself.
- A retained logical right-by-19 extraction.
- A word-read address formed from the newer low value plus that extraction shifted left by 31.
- A later selector-read base obtained by logically shifting the newer low value right by 9.

Under a 32-bit bit-vector interpretation, the left-by-31 contribution depends only on the extraction’s least-significant bit: bit 19 of the newer low value. It is not a general nonzero-to-boolean conversion. The resulting word is forwarded in four argument positions.

The two paired low values must not be treated as one dependency result or assumed equal.

### Narrow samples and local-address exposure

Subsequent operations include a byte read at a call-associated address plus 14, forwarded in four positions, and selector clearing using a retained quantity multiplied by 4096.

A dependency receives the same local stack-region address in four positions. No initialization, object size, retention rule, or ownership transfer is established for that region.

A later paired low component supplies part of an unsigned-byte-read address, combined with a call-associated displacement. A different call-associated unsigned value supplies a right-by-19 extraction. A halfword is read at the sum of that extraction and the sampled unsigned byte.

This is one represented 16-bit read. It must not be recast as two independently ordered byte reads.

A subsequent paired result supplies a high component selecting a word destination at four times that component plus 20. The stored value is the earlier right-by-19 extraction. A later result-selected word slot receives that same retained extraction.

### Field updates, scaled records, and linked words

A retained address receives two byte stores at offset 13. The first contains the low byte of twice that address; after dependency invocations, the second writes 4. Both accesses remain observable parts of the sequence.

Nearby word reads use distinct addresses:

- Four times an unsigned result plus 68.
- That result with its top two bits cleared and shifted right by 20, plus 68.

Another word store targets 256 times a result plus 20 and receives four times an earlier retained result. Associated invocations use values scaled by 64.

Further stores propagate an extraction retaining 26 low bits and shifting right by 13, then repeat sixteen-times-scaled destinations plus 20 receiving four-times-scaled values, with propagation through four-times-scaled slots.

One sixteen-times-scaled record-shaped region receives a zero word at offset 36. Its word at offset 116 is then loaded as an address-bearing value.

That loaded address later receives, at offset 36, an unsigned dependency result shifted right by 20. Its own offset-116 word is read and forwarded. Selector-directed updates use its offset-1 selector.

The right-by-20 extraction is at most 4095; shifting that retained value right by another 21 therefore produces zero. This bound concerns retained arithmetic, not memory that dependencies may change.

Separate unsigned-byte samples occur at call-associated addresses plus 21, with calls between them. A later sample is widened and stored as a 32-bit word.

Another call-associated unsigned value is masked to 29 low bits and shifted right by 18. That extraction and its scaling participate in subsequent invocations, word stores, and selector calculations. These patterns establish neither allocation nor ownership.

### Shared clearing and low-address sampling

The **first shared region**, a represented 32-bit quantity, is cleared explicitly. Calls, zero-word stores, scaled propagation, word reads, and selector-directed updates follow.

Later reads are fresh observations, not guaranteed zeros. Dependencies may modify the region, and shared-symbol overlap prevents assuming isolation.

Some retained arithmetic values are explicitly set to zero. Consequent selector reads target absolute address 1, and consequent invocation arguments are zero. These are arithmetic consequences, not null-input validation.

A later unsigned result is passed twice to F-UNRESOLVED-003. It subsequently supplies an unsigned-byte-read base at offset 1. That sampled byte becomes the base for another byte read at its own offset 1. Thus a value from 0 through 255 selects an absolute low address rather than necessarily a field within the original region.

### Late fields and word traversal

Late byte stores include a clear at a previously four-times-scaled result multiplied by 8192, plus 9, and a store at another retained address plus 9 containing that address’s low byte. The latter address also supplies a word read at offset 52.

Later traversal includes:

- A word read at a call-associated address plus 20.
- A word read at the preceding loaded value shifted right by 22, plus 20.
- A further word read at a retained loaded value plus 20.

Calls intervene; referenced memory is not assumed stable.

A sample of the first shared region is shifted right by 23. This retained extraction is at most 511, so another right shift by 23 yields zero without eliminating the original shared-memory read.

An unsigned dependency result is subsequently written to the **second shared region** and forwarded.

Several operations read a word at a retained base plus 116 and store its low byte at that base plus 1. The first relevant base is an unsigned byte sampled from absolute address 1. Later bases are dependency-produced values multiplied by four. A repeated operation on one base re-reads the word after intervening invocations.

### Late selector writes from a shared-derived word

A later fresh sample of the first shared region is retained before another forwarding invocation and intervening calls. That sample supplies the base for a word read at offset 20. The resulting **shared-derived word** supplies two distinct byte values.

After the load, an invocation receives a scaled slot address. Four no-explicit-argument invocations follow; the fourth supplies a newer paired result.

The first selector-directed store then:

- Samples its selector at an **earlier retained word logically shifted right by 23, plus 1**.
- Writes the low byte of the shared-derived word to absolute address selector plus 17.

An invocation next receives the newer paired high component in three positions and four times its low component in the fourth. A no-explicit-argument invocation supplies a result, that result is forwarded, and four more no-explicit-argument invocations occur, the fourth supplying a scalar.

The second selector-directed store then:

- Samples its selector at the **newer paired low component masked to 30 bits and logically shifted right by 21, plus 1**.
- Writes the low byte of the shared-derived word’s logical right-by-22 extraction.

These stores use different selector-read formulas and are separated by invocation boundaries. They are direct established operations, not hypothetical dependency effects.

The scalar is then forwarded scaled by four. Four no-explicit-argument invocations follow, the fourth supplying another scalar, which is forwarded scaled by four in two positions.

### Terminal effects

Following that two-position invocation, two successive no-explicit-argument invocations occur. The second supplies a value that is forwarded. A final dependency invocation supplies a byte result and a call-associated destination.

The terminal direct effects are ordered:

1. Clear one byte through that destination.
2. Write the second shared region with the returned byte in the low eight bits and a 24-bit source associated with absolute address 9 in the upper bits.
3. Clear a halfword at the newer paired low component masked to 30 bits and shifted right by 7.
4. Invoke address zero with three zero arguments.
5. Reach the displayed return only if that invocation returns.

The byte clear may overlap the 24-bit source. The composition uses that source after the clear, not an assumed earlier snapshot. Its physical read layout remains unresolved.

## Side effects

Direct mutations include selector-selected absolute bytes, calculated byte fields, 32-bit slots, a terminal halfword, and both shared regions.

Address formulas may alias. Narrow stores may overlap words or shared storage. Shared symbols are represented as potentially overlapping smaller symbols, but the full overlap map is unresolved.

Repeated accesses remain distinct even if they write equal bits. Ordinary-memory versus device-visible behavior, atomicity, synchronization, reentrancy, and interrupted-execution state are `documented_only`.

Ownership and lifetime are also `documented_only`. Address passing establishes neither borrowing nor transfer. Dependencies are not assumed to preserve globals, avoid retaining stack addresses, allocate memory, or release memory.

## Dependencies

| Opaque ID | Role | Unresolved properties |
|---|---|---|
| F-UNRESOLVED-001 | Entry-state-derived invocation | Identity, ABI, results, and effects |
| F-UNRESOLVED-002 | Remaining calculated-target invocations | Individual identities, contracts, and identity sharing |
| F-UNRESOLVED-003 | Explicitly referenced invocation receiving one unsigned result twice | Purpose and complete effect contract |
| F-UNRESOLVED-004 | Invocation of address zero | Executability, interruption, and return behavior |

These IDs group unresolved roles, not proven distinct implementations. Grouping does not authorize eliminating invocations. Calls without explicit arguments may consume ambient state; calls with unused results may still have effects.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes the relevant point is reached.

1. **Input order.** Inputs 7 and 11 produce initial positions 7, 11, 11, 7.

2. **Selector refresh.** Selector 3 directs a byte to address 20. If intervening effects change it to 8, a later clear targets address 25.

3. **Early invocation boundaries.** The unsigned result used for the right-by-18, times-1024 selector base is forwarded four times before the first corresponding clear. The four-zero invocation and following no-explicit-argument invocation precede the call-associated-index word store.

4. **Distinct scaling.** Index 3 selects address 32 under four-times-index plus 20, but address 68 under sixteen-times-index plus 20.

5. **Paired-result provenance.** If the earlier paired low value is 6, its twice-value byte read uses address 12. If the newer paired low value is 524288, its right-by-19 extraction is 1. Under 32-bit bit-vector addressing, the later word read uses address 2148007936, derived from the newer value—not address 6 or 12.

6. **Repeated field update.** Offset-13 base 129 first receives byte 2 from truncating 258, then receives 4 after intervening calls.

7. **Late selector stores.** Suppose the shared-derived word is `0xC04000AB`, the earlier retained word is `0x01000000`, and the newer paired low component is `0x00600000`. The selector-read addresses are respectively 3 and 4. If the sampled selectors are 5 and 9, the stores target addresses 22 and 26 and write bytes 171 and 1. The intervening invocations remain required interaction boundaries.

8. **Shared interference.** A dependency may modify the first shared region after its clear. Later reads need not return zero.

9. **Terminal overlap and interruption.** If the last byte clear overlaps the final composition’s upper source, the composition observes the post-clear source. If the zero-address invocation faults or does not return, normal return is not required; persistence of preceding values is not guaranteed.

## Uncertainties

The following remain `documented_only`:

- Complete invocation-trace coverage outside the explicitly described local sequences.
- Domain purpose, object identities, and protocol meaning.
- Exact ABI, argument extension, and ambient-register provenance.
- Dependency identities, preservation rules, and interruption effects.
- Endianness, alignment tolerance, mappings, and permissions.
- Physical access layout for the terminal 24-bit source.
- Allocation, retention, release, and lifetime behavior.
- Shared-region alias relationships.
- Signed overflow and shift interpretation where unsigned behavior is not established.
- Whether the terminal zero-address invocation is intentional or accurately represents executable behavior.

## Evidence

This document uses only the supplied static representation. No dependency inspection, instruction-level validation, implementation access, or runtime testing was performed.

Established evidence supports the described argument positions, widths, masks, separate paired-result provenance, direct accesses, local ordering, shared writes, and terminal invocation.

The overall dependency trace is intentionally not claimed complete. Unknown effects do not erase established calls or stores. Absence of explicit rollback does not establish write persistence or constrain restoration by dependencies.

Trusted provenance is reserved for the publishing system; no snapshot hash is asserted here.
