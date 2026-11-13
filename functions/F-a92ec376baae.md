<!-- Trusted provenance: F-a92ec376baae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a92ec376baae

## Purpose

F-a92ec376baae represents an ordered sequence of dependency calls and coprocessor transfers, followed by a computed-address 16-bit store. No application-level purpose is established.

The supplied static representation contains no explicit conditional branch or loop. Its visible continuation ends at an invalid-instruction boundary, not an ordinary return. A control-flow recovery warning limits confidence that this representation describes a complete executable function.

This contract distinguishes:

- **Established static evidence:** represented calls, operand relationships, data dependencies, transfer selectors, and the final store.
- **Conditional deductions:** consequences of interpreting the arithmetic as fixed-width 32-bit operations, with the stated right-shift behavior.
- **documented_only:** unresolved calling conventions, dependency effects, hardware semantics, memory validity, and behavior beyond the invalid boundary.

No runtime tests have been performed.

## Interface

### Entry state

The displayed interface declares no formal arguments and no returned value. Nevertheless, behavior depends on incoming execution state:

| Input label | Represented width | Consumption |
|---|---:|---|
| W | 32 bits, signed interpretation | Shifted left by 14 for F-A01. |
| L | 32 bits, signed interpretation | Determines F-A01’s indirect target through addition of `0x866`. |
| U | 32 bits, unsigned interpretation | Supplies F-A03’s first argument after logical right shift by 3 and left shift by 16. |
| V | 32 bits, unsigned interpretation | Supplies F-A03’s third argument after logical right shift by 10. |
| S1 | 32 bits, otherwise opaque | Supplies F-B13’s second argument. |
| S2 | 32 bits, otherwise opaque | Supplies F-C09’s second argument. |
| P1–P8 | Eight distinct 32-bit incoming coprocessor-associated operands | Used in the transfer combinations specified below. |

S1 and S2 denote otherwise unexplained secondary-register values at their respective consumption points. They are not established formal parameters, zero values, or results of a particular dependency.

P1–P8 identify distinct incoming operand sources, not established distinct numerical values. Their architectural roles are **documented_only**. Repeated use of a label preserves a source-visible dependency; it does not establish that hardware or global state remains unchanged across calls.

“No explicit arguments” below describes only the displayed call expression. It does not establish that a dependency consumes no additional registers, memory, or global state.

### Width and arithmetic conventions

The representation distinguishes opaque 32-bit results, represented 64-bit results, and a 16-bit destination.

For conditional arithmetic deductions:

- Left shifts retain the low 32 bits.
- Unsigned right shifts are logical.
- Signed right shifts are arithmetic.
- A 64-bit result’s low and high halves are bits 0–31 and 32–63 respectively.
- Half extraction concerns bit positions, not memory byte order.

Several shifts and additions are displayed on signed quantities. The fixed-width deductions do not establish portable signed-language semantics or confirm the actual instruction behavior. In particular, F-A01’s target is represented as L plus `0x866`; interpreting overflow as 32-bit wraparound requires machine-level confirmation.

### Output and completion

No ordinary return value or normal-return path is shown. Potential observable outcomes include dependency effects, coprocessor effects or faults, the final store of `0x00CD`, and arrival at the invalid-instruction boundary.

Continuation requires each preceding dependency and hardware operation to complete without diverting execution. Executability of indirect targets, instruction-set mode, privilege, alignment, memory mappings, and stack requirements are **documented_only**.

No allocation size, object layout, buffer extent, or ownership-bearing argument is identified.

## Behavior

### Ordering notation

F-A01 through F-E14 are opaque call-site identifiers. They do not imply distinct implementations. The two separately identified dependencies are F-0e02e4c13579 and F-3de949054bdd.

Within each ordered group below, calls occur left to right. A result-forwarding chain means that each recipient receives the immediately identified producer’s result. It does not imply any unshown purity, validation, or error handling.

Every later event is conditional on execution continuing along the represented path.

### Phase A: incoming values and the initial seed

The opening sequence is F-A01, F-A02, F-A03, then F-A04.

- F-A01’s target depends on L plus `0x866`; its sole explicit argument is W shifted left by 14.
- F-A02 has no explicit arguments.
- F-A03 receives four arguments: U logically shifted right by 3 and then left by 16; zero; V logically shifted right by 10; zero.
- F-A04 has no explicit arguments and produces a 32-bit result. Transfer T01 immediately consumes that result.

F-A05, F-A06, F-A07, and F-A08 then occur without explicit arguments. F-A09 follows and produces a represented 64-bit result. T02 consumes its low half before F-A10 is invoked.

F-A10 receives F-A09’s low half and its high half shifted left by 20. F-A10 itself produces a represented 64-bit result. F-A11 receives that result’s low half, its unchanged high half, and its high half shifted left by 12.

F-A12, F-A13, and F-A14 follow without explicit arguments. F-A15 then produces the 32-bit **initial seed**. F-0e02e4c13579 receives that seed; no explicit result of this direct dependency is consumed.

An initial-seed derivative, formed by shifting left by 11, is retained for later arguments.

### Phase B: initial-seed derivatives and forwarding

F-B01 receives zero. F-B02, F-B03, F-B04, and F-B05 then occur without explicit arguments.

F-B06 produces a represented 64-bit result. F-B07 receives:

1. Its low half shifted left by 4.
2. Its unchanged high half.
3. The retained initial-seed derivative shifted left by 17.

F-B07 produces a 32-bit result, which passes through F-B08 to F-B09. F-B08 also produces a 32-bit result. F-B10 follows without explicit arguments.

F-B11 receives two zeros and produces a 32-bit result consumed by F-B12. F-B13 then receives four arguments:

1. The initial seed’s left-shift-by-11 derivative shifted left by 31.
2. S1.
3. A zero-derived value shifted left by 26.
4. A zero-derived value shifted left by 2.

F-B13’s 32-bit result is consumed first by T03 and then by F-B14.

F-B15 produces a 32-bit result consumed by F-B16; F-B16 produces another 32-bit result consumed by F-B17.

Under the conditional fixed-width interpretation:

- F-B07’s third argument is the initial seed shifted left by 28.
- F-B13’s first, third, and fourth arguments are zero.
- F-B13’s second argument remains S1, whose value is unresolved.

### Phase C: the middle seed and paired transfers

F-C01, F-C02, and F-C03 occur without explicit arguments. F-C04 produces a 32-bit result consumed by F-C05. F-C06 receives zero.

F-C07 and F-C08 occur without explicit arguments. F-C09 receives zero, S2, and zero. F-C10 and F-C11 follow without explicit arguments.

F-C12 produces the 32-bit **middle seed**. F-C13 receives that seed, a zero-derived quantity shifted left by 22, and the seed shifted left by 5.

F-C14 produces a 32-bit result consumed by F-C15. F-3de949054bdd then occurs without explicit arguments or a consumed explicit result.

The following result-forwarding pairs occur in order:

- F-C16 produces a 32-bit result consumed by F-C17.
- F-C18 produces a 32-bit result consumed by F-C19.
- F-C20 produces a 32-bit result consumed by F-C21.

F-C20’s arguments are a zero-derived quantity shifted left by 19 and the signed right-shift-by-5 result of the middle seed’s left-shift-by-5 derivative. Under the stated arithmetic assumptions, the second argument is the sign extension of the middle seed’s low 27 bits, not generally the original seed.

F-C22 next produces a represented 64-bit result. T04 consumes its low half. F-C23 subsequently receives its high half shifted left by 1.

F-C24 produces a 32-bit result consumed by consecutive transfers T05 and T06. These transfers share the same data result and selector triple but differ in their penultimate incoming operand.

F-C25 then produces the 32-bit **late seed**.

### Phase D: late-seed derivatives and the terminal seed

F-D01 receives zero, zero, the late seed shifted left by 27, and zero.

F-D02, F-D03, F-D04, F-D05, and F-D06 occur without explicit arguments. F-D07 produces a 32-bit result consumed by T07. F-D08, F-D09, and F-D10 follow without explicit arguments.

T08 consumes the late seed’s left-shift-by-27 derivative shifted left again by 24. F-D11 then receives that same doubly shifted quantity and zero. Under fixed-width arithmetic, the doubly shifted quantity is zero.

F-D12 occurs without explicit arguments. F-D13 produces a 32-bit result consumed by T09. F-D14 follows without explicit arguments.

F-D15 receives three zero-derived arguments and produces the 32-bit **terminal seed**. T10 consumes that seed shifted left by 1.

F-D16 receives:

1. The terminal seed shifted left by 1.
2. The low 23 bits of the late seed’s left-shift-by-27 derivative, shifted left by 4.
3. The terminal seed shifted left by 4.
4. Zero.

Under fixed-width arithmetic, F-D16’s second argument is zero. Its first and third arguments need not be zero.

F-D17 and F-D18 follow without explicit arguments.

### Phase E: terminal processing and destination selection

T11 and T12 occur consecutively, each with a zero data operand but different selectors and incoming-operand combinations.

F-E01 receives zero and produces a 32-bit result. T13 then consumes the terminal seed shifted left by 4 and again by 8.

F-E02 receives:

1. The terminal seed’s left-shift-by-4 derivative shifted left by 8.
2. That derivative shifted left by 12.
3. F-E01’s result shifted left by 9.

F-E03 produces a represented 64-bit result. F-E04 receives its low half, its high half, and its low half shifted left by 1.

For the remaining masked arguments, define a **masked intermediate** as the terminal seed’s left-shift-by-4 derivative, masked with `3`, then shifted left by 16.

F-E05 receives that intermediate shifted left by 8. Its 32-bit result passes through F-E06 to F-E07; F-E06 also produces a 32-bit result. F-E08 follows without explicit arguments.

F-E09 produces a represented 64-bit result. F-E10 receives its low half, its high half, and its high half shifted left by 11. F-E10 produces another represented 64-bit result, retained for destination selection.

F-E11 receives F-E10’s low half. F-E12 then produces a 32-bit result. F-E13 receives F-E12’s result and the masked intermediate shifted left by 25. This latter argument is derived from the intermediate itself, not from the already further-shifted argument supplied to F-E05.

F-E14 follows without explicit arguments.

Under fixed-width arithmetic, the masked intermediate is zero because shifting the terminal seed left by 4 clears both bits selected by mask `3`. Consequently, F-E05’s argument and F-E13’s second argument are zero.

### Final store

The destination depends on the retained F-E10 result, not a newly obtained result from F-E11 through F-E14.

The represented address calculation first logically shifts the complete unsigned 64-bit result right by 41. That leaves at most 23 bits. The resulting 32-bit signed interpretation is therefore nonnegative. A further signed right shift by 11 yields the **address component**.

Equivalently, the address component is F-E10’s unsigned 64-bit result logically shifted right by 52. The destination is:

**address component + floor(address component / 4)**.

The component ranges from 0 through 4095. The destination ranges from 0 through 5118, with gaps.

After F-E14, the represented path writes the 16-bit value `0x00CD` to that destination. The next represented event is the invalid-instruction boundary. No ordinary return is visible.

## Side effects

### Coprocessor transfers

There are exactly **13 explicit coprocessor transfers**.

The selector triple records the three displayed selector fields in order. The final two columns preserve the two incoming coprocessor-associated operands in order. P1–P8 are opaque labels; neither their names nor the selector values establish an architectural interpretation.

| Transfer | Selector triple | Data operand | Penultimate operand | Final operand |
|---|---|---|---|---|
| T01 | `(0, 5, 7)` | F-A04’s 32-bit result | P8 | P8 |
| T02 | `(0, 4, 7)` | F-A09’s low half | P5 | P8 |
| T03 | `(0, 6, 7)` | F-B13’s 32-bit result | P2 | P8 |
| T04 | `(0, 6, 7)` | F-C22’s low half | P2 | P8 |
| T05 | `(0, 3, 7)` | F-C24’s 32-bit result | P4 | P8 |
| T06 | `(0, 3, 7)` | The same F-C24 result | P8 | P8 |
| T07 | `(0, 1, 7)` | F-D07’s 32-bit result | P1 | P8 |
| T08 | `(0, 5, 7)` | Late seed shifted left by 27, then by 24 | P3 | P8 |
| T09 | `(0, 6, 7)` | F-D13’s 32-bit result | P6 | P8 |
| T10 | `(0, 5, 7)` | Terminal seed shifted left by 1 | P7 | P8 |
| T11 | `(0, 7, 7)` | Zero | P6 | P8 |
| T12 | `(0, 0, 7)` | Zero | P8 | P8 |
| T13 | `(0, 2, 7)` | Terminal seed shifted left by 4, then by 8 | P6 | P8 |

All transfers share P8 as their final incoming operand. T01, T06, and T12 also use P8 in the penultimate position.

T03 and T04 share the same selector triple and incoming-operand pair but consume different dependency results. T05 and T06 share their data operand and selector triple, while their penultimate operands differ. T01, T08, and T10 share a selector triple but use different penultimate operands and data expressions. These distinctions are established static evidence, even though their hardware meanings remain **documented_only**.

The table describes represented operand consumption, not additional ordinary memory reads or a guarantee of preserved hardware state.

### Memory, ordering, and ownership

The only ordinary data-memory access explicitly shown is the final 16-bit store. No ordinary data-memory reads are explicitly shown. This does not constrain dependency effects: callees may read or modify memory, globals, hardware, or ownership state.

The store follows F-E14. Retained values are forwarded across intervening calls as represented; this is not a general callee-preservation guarantee.

The destination may be zero or odd. No valid mapping, alignment, writable extent, or ownership of that destination is established. Endianness is unknown.

No allocation, release, ownership transfer, cleanup, retry, rollback, or explicit failure-result check appears. A nonreturning dependency or fault can prevent every later event. The invalid boundary does not establish a particular signal, exception number, or recovery mechanism.

## Dependencies

| Opaque identifier | Established interaction |
|---|---|
| F-0e02e4c13579 | Receives the initial seed once; no explicit result consumed. |
| F-3de949054bdd | Called once during Phase C; no explicit arguments or consumed result. |
| F-A01–F-A15 | Phase A’s indirect call sites. |
| F-B01–F-B17 | Phase B’s indirect call sites. |
| F-C01–F-C25 | Phase C’s indirect call sites. |
| F-D01–F-D18 | Phase D’s indirect call sites. |
| F-E01–F-E14 | Phase E’s indirect call sites. |

These identifiers establish call-site order and displayed data flow, not complete signatures or distinct target identities. Other than F-A01’s incoming-state-dependent target calculation, the displayed indirect targets are constructed from fixed code-relative values.

Dependency implementations, target aliasing, additional inputs, error contracts, memory effects, ownership behavior, and preservation guarantees are **documented_only**.

## Acceptance examples

These examples are conditional static-review criteria, not executed tests.

1. **Incoming transformation.** If U is `8` and V is `1024`, F-A03 receives `65536`, zero, `1`, and zero under the stated arithmetic interpretation.

2. **Initial-seed truncation.** An initial seed of `1` gives F-B07 a third argument of `0x10000000`. A seed of `16` gives zero. Neither case determines S1.

3. **Unresolved secondary values.** Even when F-B13’s other three arguments are zero, substituting zero for S1 is not justified. The same restriction applies to S2 at F-C09.

4. **Signed extraction.** A middle seed of `0x04000000` produces `0x80000000` as F-C13’s third argument. With arithmetic right shift, F-C20’s second argument is `0xFC000000`.

5. **Paired-transfer identity.** If F-C24 returns `0x12345678`, T05 and T06 both consume that data value with selector triple `(0, 3, 7)`. Their operand pairs remain `(P4, P8)` and `(P8, P8)`. Numerically equal incoming values would not erase the distinct source relationships.

6. **Common final operand.** Every transfer T01–T13 consumes P8 in its final operand position. T09, T11, and T13 additionally share P6 in the penultimate position while retaining different selector triples.

7. **Transfer inventory.** A complete represented continuation contains 13 explicit transfers. Dependency-internal transfers, if any, are outside this inventory and cannot be counted from the supplied material.

8. **Late-seed truncation.** For every 32-bit late seed, T08’s data operand and both F-D11 arguments are zero under fixed-width arithmetic. F-D16’s second argument is also zero, without forcing its first or third argument to zero.

9. **Terminal masking.** For every 32-bit terminal seed, F-E05 receives zero and F-E13’s second argument is zero under the stated interpretation.

10. **Destination boundaries.** Address components `0`, `1`, `4`, and `4095` produce destinations `0`, `1`, `5`, and `5118`, respectively. Address `4` is not generated: neighboring components `3` and `4` produce addresses `3` and `5`.

11. **Store and interruption.** If execution reaches the store and the platform permits it, the stored value has width 16 bits and value `0x00CD`. If F-E14 does not return, the store is not reached. If the destination rejects the access, successful completion is not promised.

## Uncertainties

- **Control flow — documented_only:** recovery warnings and the invalid boundary prevent confirmation of complete executable control flow.
- **Application purpose — unresolved:** hardware interaction is suggested, but no subsystem or higher-level intent is established.
- **ABI — documented_only:** formal signature reliability, secondary-register origins, stack requirements, and 64-bit return conventions lack independent confirmation.
- **Arithmetic — conditional:** fixed-width truncation and arithmetic signed right shifts require instruction-level confirmation.
- **Coprocessor meaning — documented_only:** selector meanings, operand roles, privileges, synchronization, exceptions, and state effects are unknown. Selector values and operand-sharing relationships are nevertheless explicit.
- **Dependencies — documented_only:** no purity, memory ownership, global preservation, or returning behavior is assumed.
- **Memory — documented_only:** mapping, alignment, endianness, device behavior, and writability are unavailable.
- **Terminal outcome — documented_only:** no specific runtime trap or omitted continuation is established.

## Evidence

The supplied static representation establishes the call order, displayed argument and result widths, retained-value dependencies, 13 transfer sites, their selector triples and incoming-operand relationships, and the final 16-bit store of `0x00CD`.

The supplied dependency map provides the two direct dependency identities. Embedded warnings identify limitations of the representation; prior prose and reviewer feedback are not independent behavioral authority.

Zero reductions, signed extraction, destination equivalence, and address bounds are conditional arithmetic deductions. They are not runtime observations.

No dependency execution, runtime tests, hardware checks, or memory-write observations have been performed.
