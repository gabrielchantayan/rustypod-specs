<!-- Trusted provenance: F-fe45ae35e2a4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fe45ae35e2a4

## Purpose

The established behavior is an ordered sequence of indirect calls, computed-address memory accesses, two assignments through one named global output symbol, and a final attempted call to address zero.

Values come from explicit arguments, incoming machine state, dependency results, two snapshots through a named global input symbol, and other memory reads. The routine propagates these values through word stores, high-bit extractions, selector-directed byte updates, and a final halfword clear.

No domain-level purpose is established. Object construction, collection management, hardware initialization, and supervisor operations are not contractual interpretations.

The named global quantities are not established as disjoint storage. In particular, the final 24-bit input quantity may overlap the output region. The final output expression does not explicitly read through the output symbol, but independence from the earlier output assignment is not established.

## Interface

### Explicit arguments and result

The represented interface accepts two 32-bit arguments and declares no return value.

- The first argument has no visible direct use.
- The second argument is passed to the first indirect dependency.
- Neither argument is explicitly validated.
- No explicit status or error result is produced.

Absence of direct use does not prove that the first argument’s incoming register contents are inaccessible to dependencies. The complete calling convention is unknown.

### Additional effective inputs

Execution also depends on:

- Incoming link-related state that determines the first call target after addition of `0xF38`.
- An incoming unsigned 32-bit value whose upper ten bits determine an early read address.
- An incoming 32-bit payload for the first word store.
- Secondary result-register state supplying indices, repeated arguments, and the terminal byte-clear pointer.
- Two temporally distinct 32-bit reads through the named global input symbol.
- Selector bytes, including repeated reads at address `1`.
- A 24-bit quantity read near termination.
- Contents of all other computed read addresses.

These are effective environmental inputs, not additional validated API parameters.

### Widths and arithmetic

The direct operations distinguish 8-bit bytes, 16-bit halfwords, and 32-bit words. Selected dependencies produce 64-bit results, split numerically into low and high 32-bit halves.

A byte store retains the low eight bits of its payload. A word-to-byte copy reads a complete 32-bit word and then narrows its value; it is not established as a read of the lowest-addressed byte of that word.

Unsigned 32-bit right shifts are logical. Unsigned arithmetic follows the represented finite width, including reduction modulo \(2^{32}\) where applicable. Some multiplications and left shifts instead operate on signed 32-bit representations. Their behavior for negative operands or signed overflow is not established as portable. Signed-to-address conversions and pointer arithmetic likewise require environmental interpretation.

The final composition places a 24-bit quantity in bits 8–31 and a retained 8-bit result in bits 0–7. The physical access mechanism used to obtain the 24-bit quantity is not independently established.

### Preconditions and completion

Reaching any later operation requires every preceding call to return normally and every preceding memory access to succeed. The routine does not establish these conditions.

Bounds, alignment, executable mappings, address width, byte order, and memory lifetime are unresolved. Successful completion of the preceding operations does not establish that address zero is callable. Normal return is possible in the representation only if the final call returns normally.

## Behavior

### Ordering model

There are no visible conditional branches or loops. The represented path executes once, in the order described below, subject to calls and accesses completing.

Intervening calls remain observable even when their results are unused. They are not established as pure or as preserving memory, globals, or allocation lifetimes.

The opaque dependency IDs identify selected call sites, not necessarily distinct implementations. Other intervening calls remain required parts of the represented sequence.

### Initial dispatch and linked word accesses

F-000000000001 is called at the target formed from incoming link-related state plus `0xF38`, receiving the second explicit argument. Subsequent calls include production of a 32-bit result and forwarding of that result to another dependency.

After further calls yield a signed index, the incoming payload is stored as a word at four times that index plus `0x14`. A word is then read at the incoming unsigned value logically shifted right by 22, plus `0x14`. A following dependency receives four times the index.

The read address is not word-scaled: its offset before `0x14` is in `0..1023`. Retain its value as the **initial link word**.

Further calls produce another signed index. That index shifted left by two is stored as a word at four times the initial link word shifted right by one, plus `0x14`. The routine next reads a word at the unshifted initial link word plus `0x14`. F-000000000002 receives the initial link word shifted right by one.

The shifted second index and the second memory word remain available for later operations. The two address calculations must not be conflated.

### First global snapshot and high-bit propagation

A later call group produces another signed index. The retained shifted second index is stored at four times this new index plus `0x14`.

Only after that store is the first 32-bit snapshot taken through the named global input symbol. A call receiving the new scaled index and additional calls intervene before the next direct store.

The retained second memory word is logically shifted right by 22, producing a ten-bit value in `0..1023`. The first global snapshot is logically shifted right by 23, producing a nine-bit value in `0..511`.

The ten-bit value is stored as a complete word at four times a signed secondary-result index plus `0x14`. F-000000000003 receives that secondary index twice, followed by the nine-bit global extraction and the ten-bit memory extraction.

A later ordinary result is forwarded. The nine-bit extraction is also shifted right by 23 again; this necessarily yields zero.

After more calls, the ten-bit memory extraction is stored at four times another signed returned index plus `0x14`, followed by a call receiving that scaled index. A subsequent call group supplies an unsigned secondary-result index. A zero word is stored at four times that unsigned index plus `0x14`, and F-000000000004 receives the unscaled index.

### Masked-address read and initial byte updates

Following result production, forwarding, and further calls, another returned index selects a word destination at four times the index plus `0x14`. The payload is the first global extraction shifted right by 23 again, so this store writes zero.

A word is then read at the retained unsigned secondary index masked with `0x1FFFFFFF`, logically shifted right by 19, plus `0x14`. The offset before `0x14` lies in `0..1023`; no additional word scaling or alignment is applied.

After intervening calls, that word is shifted right by 22. Retain the resulting ten-bit value as the **byte-phase extraction**.

The routine reads a selector byte at address `1` and writes the low byte of the byte-phase extraction at the selector value plus `0x11`. The selector address is exactly `1` because its represented address expression uses the guaranteed-zero double shift.

F-000000000005 receives one secondary-result value three times and zero as its fourth argument. Later calls include an explicit zero argument.

After another index is obtained, a selector byte is read at four times the complete byte-phase extraction plus `1`. The selected destination, selector value plus `0x11`, is cleared as a byte. A following dependency receives the new index shifted left by two.

This selector read uses the full ten-bit extraction, not its earlier byte truncation. Its address ranges from `1` to `4093`, in steps of four.

### Further clears and first output assignment

A later dependency receives two copies of a newly produced index shifted left by two. Another dependency returns an unsigned 32-bit word, which is forwarded.

After a further call, the byte at address `1` is freshly read, and the byte at that selector value plus `0x11` is cleared.

A signed secondary-result value is captured for later shifted arguments. F-000000000006 receives three copies of that captured value.

Following calls include one receiving a newly forwarded result together with the preceding unsigned returned word logically shifted right by 23. Further calls produce another index.

The byte at address `1` is then freshly read again. Its zero-extended value is retained as a base, and the byte at that base plus `0x11` is cleared. A dependency receives the latest index shifted left by two.

These selector reads cannot be replaced with earlier snapshots: intervening dependencies and preceding stores may alter the relevant memory.

After another duplicated-argument call and a result-producing call, the resulting unsigned word is assigned through the named global output symbol. This is the **first explicit assignment** through that symbol. The same returned value is then forwarded.

After one further call, a complete word is read at the retained byte base plus `0x74`, and its low byte is written at that base plus `1`.

### Repeated word-to-byte copies

F-000000000007 receives the captured signed secondary value shifted left by 11.

A subsequent dependency produces 64 bits. The low and high 32-bit halves are supplied separately to the next call, followed by the retained value from the first output assignment logically shifted right by 23. This argument uses the retained returned value, not a fresh read through the output symbol.

Further calls produce a signed index whose fourfold value becomes a base. The routine reads a word at base plus `0x74`, writes its low byte at base plus `1`, and passes the base to a dependency.

A later call group produces another signed index. Its left shift by two is retained in unsigned 32-bit form as the **working scaled index**.

Before using that working value, the routine repeats the word-to-byte copy at the preceding base. This is a fresh word read after the intervening calls, not reuse of the earlier word.

F-000000000008 receives, in order:

- The captured signed secondary value shifted left by 12.
- Two copies of the working scaled index.
- The most recent signed index shifted left by 18.

Additional calls include result forwarding. Another returned index supplies a new fourfold base. A word is read at this base plus `0x74`; retain the full unsigned value as the **propagated word**. Its low byte is written at base plus `1`, and a following call receives the base.

### Full-word propagation and second global snapshot

After more calls, the propagated word is stored at 16 times a newly returned signed index plus `0x14`.

A dependency then receives the working scaled index logically shifted right by two and subsequently shifted left by 16. This operates on the already-scaled unsigned 32-bit value; it cannot unconditionally be replaced by a shift of the original signed index.

Following further result production and forwarding, the complete propagated word is stored at four times another returned index plus `0x14`. A dependency receives that scaled index.

Another returned signed index is shifted left by two. The result is stored at the working scaled index masked with `0xFFFFFFFC`, plus `0x14`, and is passed twice to a dependency.

A later call sequence clears a word at four times a newly returned index plus `0x14`, followed by a call receiving that scaled index.

Another dependency produces 64 bits. The signed interpretation of its low half is shifted left by two and stored at four times the signed interpretation of its high half plus `0x14`.

Only after this store is the second 32-bit snapshot taken through the named global input symbol. A dependency receives the shifted low half, the high half, and two further copies of the shifted low half.

After more calls, the same shifted low-half value is stored at four times another returned index plus `0x14`. A word is then read at the retained second global snapshot plus `0x14`.

That indirect read is not a third read through the named input symbol. Later changes to that symbol do not replace the retained address value.

### Final selector writes and termination attempt

A further dependency returns 64 bits; both halves are retained.

A selector byte is read at the propagated word logically shifted right by 23, plus `1`. The low byte of the most recent indirect memory word is written at the selector value plus `0x11`.

F-000000000009 receives three copies of the new high half, followed by the unsigned low half shifted left by two.

After forwarding and additional calls, another selector is read at the retained low half masked with `0x3FFFFFFF`, logically shifted right by 21, plus `1`. The low byte of the indirect memory word logically shifted right by 22 is written at that selector value plus `0x11`.

Each selector-read address lies in `1..512`. Each selected byte-write destination lies in `0x11..0x110`.

Further calls include a scaled-index argument, duplicated shifted arguments, a target calculation using displacement `0xB0`, and result forwarding. A final dependency supplies an 8-bit result.

The terminal direct effects have this order:

1. A byte is cleared through a pointer supplied in secondary result state.
2. The currently available 24-bit named quantity is read.
3. A 32-bit composition of that quantity in bits 8–31 and the retained returned byte in bits 0–7 is assigned through the output symbol.
4. A halfword is cleared at the retained low half masked with `0x3FFFFFFF`, logically shifted right by seven.
5. F-000000000010 is attempted at address zero with three zero arguments.

The halfword address lies in `0..0x7FFFFF` and is not explicitly aligned.

The composition is the **second explicit assignment through the same output symbol**. Its expression contains no explicit read through that symbol. Nevertheless, its 24-bit operand may overlap output storage, and intervening dependencies may propagate earlier output into either final operand. Independence from the first assignment is therefore not established.

## Side effects

Direct effects comprise the ordered memory accesses and two assignments through the output symbol. The input symbol is explicitly read twice. The low-address selector and final 24-bit quantity are read, not explicitly assigned through their own named symbols.

Symbol names do not establish disjoint storage. The supplied representation explicitly warns of overlap between wider global symbols and smaller symbols. Computed stores may also alias named quantities, other computed destinations, or storage relevant to dependencies.

Consequently, a quantity not explicitly assigned by name is not guaranteed unchanged. The terminal pointer-directed byte clear may affect the subsequently read 24-bit quantity if their storage overlaps.

Dependencies may add memory operations, allocation, release, exceptions, nonlocal transfers, or nontermination. None of these effects is established or excluded.

No ownership transfer, allocation responsibility, or release obligation is identified. Address use alone does not establish ownership. Earlier access success does not prove that an intervening dependency preserves validity.

There is no visible rollback, bounds check, null check, alignment check, or error handler. Failure may leave preceding writes observable.

## Dependencies

These opaque IDs designate selected call sites.

| ID | Established use |
|---|---|
| F-000000000001 | Initial link-derived dispatch with the second explicit argument. |
| F-000000000002 | Receives the initial link word logically shifted right by one. |
| F-000000000003 | Receives a repeated secondary index and two high-bit extractions. |
| F-000000000004 | Receives the unsigned index associated with a preceding word clear. |
| F-000000000005 | Receives repeated secondary state and zero. |
| F-000000000006 | Receives three copies of captured signed secondary state. |
| F-000000000007 | Receives captured state shifted left by 11. |
| F-000000000008 | Receives shifted captured state, repeated working scaled index, and another shifted index. |
| F-000000000009 | Receives three high-half copies and a shifted unsigned low half. |
| F-000000000010 | Final zero-address call with three zero arguments. |

Other calls produce ordinary or 64-bit results, forward values, receive scaled indices, or have no explicit arguments in the representation. No-explicit-argument notation does not establish absence of machine-state inputs.

Dependency identities, full signatures, preservation rules, ownership effects, and termination properties are not established.

## Acceptance examples

These are conditional arithmetic and ordering requirements, not executed tests.

1. **Guaranteed zero.** A first global snapshot of `0xFFFFFFFF` yields `511` after a logical right shift by 23, then zero after another such shift. The affected later word stores write zero, and the first byte-phase selector is read at address `1`.

2. **Selector width.** A masked-address word value of `0xFFFFFFFF` produces a byte-phase extraction of `1023`. The first byte payload is `255`, but the later selector-read address is `4093`, using the complete extraction.

3. **Fresh selectors.** If address `1` contains `2` when a selector is read, its selected clear destination is `0x13`. If an intervening dependency changes that byte to `7`, a later fresh selection clears `0x18`.

4. **Full-word retention.** A propagated word of `0x123456AB` supplies `0xAB` to its immediate byte write while supplying the complete `0x123456AB` to subsequent word propagation.

5. **Retained snapshot.** If the second global snapshot is `0x100`, a later change of the named input to `0x200` does not change the subsequent indirect read address from `0x114`, assuming execution remains valid.

6. **Repeated output symbol.** Both explicit global output assignments target the same named symbol. This does not establish disjointness from the 24-bit input quantity or other storage.

7. **Conditional final composition.** If the 24-bit quantity actually read at the terminal composition is `0x123456` and the retained returned byte is `0xAB`, the assigned value is `0x123456AB`. This numerical result does not prove independence from earlier state. Either operand may have been influenced by the first output assignment through aliasing or dependency effects.

8. **Overlapping final input.** If the 24-bit quantity overlaps output storage, its terminal read must observe the value available at that point, after intervening effects and the terminal byte clear. The contract does not substitute an independently initialized value or the first assignment’s original bits.

9. **Final-call failure.** If address zero is not callable, normal return is not established. The terminal byte clear, output assignment, and halfword clear precede the attempted call, provided those operations succeed.

## Uncertainties

The following remain **documented_only**:

- Architecture, address width, byte order, alignment rules, and precise signed-arithmetic behavior.
- Accuracy and completeness of the represented boundary and calling convention.
- Exact relationships between secondary result state and dependency outputs.
- Dependency identities, signatures, memory effects, ownership effects, and termination.
- Accessibility and purpose of low-address memory.
- Exact global layouts, overlapping symbol extents, and computed-address aliasing.
- Whether earlier output influences the final operands through overlapping storage or dependencies.
- Physical access details for the 24-bit quantity.
- Executability of address zero.
- Domain-level intent behind the offsets, masks, and shifts.

No inferred domain intent is promoted to an acceptance requirement.

## Evidence

**Established static evidence:** the supplied representation shows the direct access widths, value dependencies, explicit-operation ordering, selected call arguments, two reads through the input symbol, two assignments through the output symbol, and the final zero-address call attempt. Its overlap warning prevents treating differently named global quantities as necessarily disjoint.

**Arithmetic deductions:** guaranteed-zero shifts, unsigned extraction ranges, selector-address ranges, truncation results, and conditional final composition follow from the represented widths and operations.

**Inferred intent:** no domain-level intent is established. Terms such as “snapshot,” “selector,” and “propagated word” describe observable data roles only.

**Unavailable evidence:** dependency bodies, runtime traces, an authoritative memory map, and architecture documentation were not supplied. No runtime tests have been performed.
