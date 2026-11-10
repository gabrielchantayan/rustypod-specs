<!-- Trusted provenance: F-6f64ea6fa5a8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6f64ea6fa5a8

## Purpose

**Established static behavior:** This operation initializes retained state and fixed storage, performs overlapping output transfers, and repeatedly invokes opaque dependencies. Dependency results supply both data and addresses. Each transaction either repeats directly or performs extended output. Extended output can lead back through shared output or reach software breakpoint four.

No normal return, returned status, or local recovery mechanism is represented. Completion of an earlier effect does not guarantee that any later effect occurs: memory access failures, unsupported arithmetic, dependency behavior, or the breakpoint may interrupt execution.

**Inferred intent:** The repeated construction of output records suggests a state-processing operation. No application purpose, hardware protocol, processor-state interpretation, or reset behavior is established.

This contract describes conditional observable behavior, not a guarantee that the complete operation executes meaningfully on any particular platform.

## Interface

### Inputs

The apparent interface has five 32-bit arguments.

| Input | Established contribution |
|---|---|
| Argument 1 | No explicit use. |
| Argument 2 | No explicit use. |
| Argument 3 | Signed operand in an initialization byte-address calculation. |
| Argument 4 | Unsigned value; its logical right shift by 18 is passed to F-D01. |
| Argument 5 | Signed operand in a quantity used as both initialization data and an address contributor. |

Additional incoming state includes a signed 32-bit entry contributor and an unsigned 32-bit link value. A signed 32-bit register quantity available after F-D01 contributes another initialization destination. Whether that quantity is a declared result, an incidental register result, or something else remains **documented_only**.

Memory contents and dependency results are also inputs. Important memory inputs are:

- A sample captured before F-D01.
- A scaling source read after F-D01.
- Transaction data and replacement-address slots.
- A 24-bit component read during extended output.

The sample and the later 24-bit component are separate observations; neither may be substituted for the other.

### Outputs and storage requirements

There is no explicit returned value. Observable outputs are dependency invocations, memory accesses, retained-state transitions, and a conditional breakpoint.

The apparent model uses 32-bit scalar state and addresses, with 64-bit results from selected dependencies. Explicit indirect transfers have widths of 8, 16, 32, or 64 bits. A stored address occupies 32 bits.

Two output layouts are established:

| Output region | Layout |
|---|---|
| Primary record | Three 32-bit fields at byte offsets 0, 4, and 8. |
| F-D08 result region | A 64-bit field at the base and a separately written byte at offset 8. |

Fixed-object declarations are absent. Assignments to those objects establish assigned values, but not their physical access widths. This limitation includes the sample source, configuration objects, and composite-output object.

Execution requires every reached access to have a meaningful address, suitable alignment, adequate extent, and appropriate permissions under the actual platform rules. None of these conditions is checked locally. Address zero is not rejected.

No allocation, release, ownership transfer, or retention permission is established. An address received from a dependency is not evidence that the addressed storage exists or is owned by this operation.

### Retained state terminology

The following labels describe relationships, not public interface fields:

- **Sample:** initialization input converted to unsigned 32 bits.
- **Read address:** source of the next transaction’s current-value load.
- **Current value, working value, accumulator, pending value:** retained unsigned 32-bit quantities.
- **Primary destination and secondary destination:** independently retained output addresses.
- **Captured byte:** the current value’s low byte saved at transaction entry.

## Behavior

### Arithmetic boundaries

Unsigned 32-bit arithmetic retains the low 32 bits. Unsigned right shifts are logical.

Signed arithmetic must not be replaced with unsigned wrapping. This distinction affects initialization address calculations, the pending-value multiplication, several destination calculations, and extended-output shifts.

The representation does not settle all consequences of signed overflow, invalid signed left shifts, negative signed right shifts, out-of-range signed conversions, or pointer-to-integer conversions. These remain **documented_only** where relevant.

Conversions of complete 64-bit dependency results to addresses also require an execution-model rule. Taking the low 32 bits is compatible with the apparent address width, but is not independently established as the actual conversion rule. Explicit extraction of an upper 32-bit result component is a separate, established operation.

A 16-bit store establishes the transfer width. It does not establish portable behavior for every intermediate conversion to signed 16 bits.

### Initialization and first output

Initialization has a significant ordering boundary at F-D01.

Before that call, a fixed control object receives `0x18`; the byte `0xCE` is then written at the signed sum of the entry contributor and argument 3. The sample is captured next. A separate fixed object is assigned zero before its address is supplied to F-D01.

F-D01 receives, in order:

1. Argument 4 logically shifted right by 18.
2. `0xC0`.
3. The address of the zero-assigned object.

Its call destination is obtained by clearing the incoming link value’s two lowest bits and adding `0x8A4` with unsigned 32-bit arithmetic.

After F-D01 returns, signed doubling of the entry contributor establishes the initial read address. The scaling source is then read and multiplied by 32. Its declaration is unavailable, so the multiplication’s arithmetic type is unresolved.

Two 16-bit transfers of `0xECA0` follow. The first destination is four times the signed post-call register quantity. The second is the scaling product plus six, with signed addition represented at that stage. Four fixed configuration objects then receive, in order, zero, `0x20000000`, `0x40`, and zero.

The initial accumulator is the sample masked by `0x0897ECA0`; the initial working value is `0x44BF64FE`. An initialization quantity is formed from the signed interpretation of the initial read address, argument 5, and eight. The composite-output object receives the sample before that quantity is written as a 32-bit value at its own signed doubled address.

Two further fixed assignments follow: `0x0897ECA0`, then `0xFE`.

The initial current value is `0x40000000`, and pending is zero. Primary and secondary destinations initially identify the same fixed output object.

### Shared output

Shared output occurs at initialization and after one extended-output outcome. Direct transaction repetition bypasses it.

Its observable transfer sequence is:

| Order | Destination | Transfer |
|---|---|---|
| 1 | Primary destination | Byte zero. |
| 2 | Primary destination | 16-bit representation of working value. |
| 3 | Primary destination | 32-bit pending value. |
| 4 | Secondary destination | Accumulator after adding current value modulo \(2^{32}\). |

The accumulator update occurs after the primary-destination transfers.

On initial entry, all four transfers address the same location. Their values are byte zero, halfword `0x64FE`, word zero, and the updated accumulator word. Implementing only the final contents would omit required observable transfers.

Shared output leads to a transaction with first-call argument zero. Initial entry selects F-D02I. Reentry from extended output selects F-D02Z.

### Transaction inputs and read timing

Each transaction captures the current value’s low byte before its first dependency invocation. That captured byte remains the transaction’s byte output even if the current value is subsequently replaced.

The first invocation selects F-D02I, F-D02S, F-D02Z, or F-D02B according to the incoming control path. Its apparent 64-bit result supplies an immediate output destination and a later address-slot location.

Before F-D03, the transaction has the following ordered effects:

| Order | Required effect |
|---|---|
| 1 | A 16-bit value derived from the first result’s low portion is written through an address converted from the complete result. |
| 2 | Working value multiplied by 64 modulo \(2^{32}\) is written as a word at the address represented by that same product. |
| 3 | A word is read through the existing read address and becomes current value. |
| 4 | A halfword is written at working value plus four. Its data comes from converting the old read address to signed 16 bits, adding four, and narrowing for storage. |
| 5 | A 32-bit replacement address is read from the slot 80 bytes beyond the signed interpretation of the first result’s upper 32 bits. |

The destination addition in effect 4 is unsigned; the slot-location addition in effect 5 is signed.

These accesses need not be disjoint. Earlier writes can modify the current-value source or replacement-address slot before it is read. The old read address nevertheless remains the source address and halfword-data contributor until the replacement-slot read occurs.

### Dependency chain and transaction selection

After the replacement-address read, the invocation order is F-D03, F-D04, F-D05, then F-D06.

F-D03 and F-D04 have no explicit arguments or consumed results. F-D05 receives the newly loaded current value. F-D06 receives that value logically shifted right by three.

F-D06’s upper 32-bit result component becomes the primary destination. Separately, conversion of its complete result to signed 32 bits supplies an operand for signed multiplication by 32. That product is assigned to unsigned pending state.

Pending is written as a 32-bit word at the address represented by pending before the continuation is selected.

The comparison is unsigned:

- Pending below 20 selects direct repetition through F-D02S, with argument zero.
- Pending at least 20 selects extended output.

When the signed multiplication has a defined mathematical result, pending is a multiple of 32. Zero is therefore the only such pending value below 20. A defined-arithmetic direct repeat consequently requires an attempted zero-word write at address zero before F-D02S.

This is a required access, not evidence that address zero is writable.

### Extended output

Extended output begins with a word containing working value at pending plus 88, followed by a halfword representation of the replacement read address at working value plus 14. Both destination additions use unsigned 32-bit arithmetic.

Current value then becomes working value shifted left by two, retaining 32 bits.

The composite-output object next receives a composition with the transaction’s captured byte in its low eight bits and a separately read 24-bit component in its upper bits. This component is read at this stage, not during initialization. Earlier writes or dependencies may have changed its storage.

F-D07 receives the replacement read address logically shifted right by two. F-D08 follows without explicit arguments.

F-D08 supplies three related outputs: its upper 32 bits become the secondary destination, conversion of its complete result supplies a base address, and the complete 64-bit result is retained for storage.

The associated transfer order is:

| Order | Destination | Transfer |
|---|---|---|
| 1 | Signed doubling of the converted base | Captured byte. |
| 2 | Base plus eight | Byte containing the upper result component’s low eight bits. |
| 3 | Unsigned doubling of the accumulator | Primary-destination address, stored as 32 bits. |
| 4 | Base | Complete 64-bit F-D08 result. |

The second destination is eight bytes beyond the base, not one byte beyond it. Signed doubling in the first destination is not guaranteed to wrap. Overlap can cause the final wide transfer to replace bytes written earlier.

For later output, the accumulator is derived from the primary destination interpreted as signed and shifted left by two. Working value is derived from the replacement read address interpreted as signed and shifted left by eight. Assignment to unsigned state does not remove the signed-shift restrictions.

The primary record then receives three words in order: current value, new working value, and replacement read address. Pending becomes current value multiplied by 256 modulo \(2^{32}\).

### Final selection and breakpoint

The final predicate is derived from the replacement read address using signed left shift by 31, right shift by 32 on an apparent 32-bit operand, conversion to unsigned, and logical right shift by 20.

The width-sized right shift prevents assigning this expression an unconditional portable meaning. It must not be specified as an established address-bit test.

The primary destination becomes zero before either represented continuation:

- **Zero predicate:** execution selects shared output. Its first specified memory effect is therefore a byte-zero write at address zero. If shared output completes, F-D02Z receives zero.
- **Nonzero predicate:** the predicate is written as a word at pending plus 32, using unsigned addition. A first-call argument is prepared by interpreting pending as signed and right-shifting it by 29. Software breakpoint four follows.

Only if execution resumes after the breakpoint does working value become the signed interpretation of the secondary destination right-shifted by 18. The next transaction then selects F-D02B with the prepared argument.

Neither predicate outcome is established as reachable under a particular concrete execution model.

## Side effects

Completed effects have no represented rollback mechanism. A later failure does not imply restoration of earlier writes.

Calculated destinations can overlap fixed storage, output records, later read locations, or dependency-visible memory. Captured values must remain distinct from later reads even when their storage might coincide.

Transfer width and order matter independently of final contents. Endianness, atomicity, synchronization, barriers, and device-level ordering are not established.

Dependencies may modify global or otherwise reachable memory. No preservation of the sample source, configuration objects, composite-output components, or output regions is assumed. No dependency is assumed to allocate, release, retain, or transfer ownership of storage.

There is no explicit error status, validation branch, or local recovery path for invalid addresses, arithmetic problems, dependency failure, or breakpoint handling.

## Dependencies

These opaque IDs distinguish interactions in this contract. They do not establish implementation identity or equivalence.

| ID | Established interaction |
|---|---|
| F-D01 | Link-derived initialization invocation with three arguments; a post-call register quantity is subsequently consumed. |
| F-D02I | First transaction invocation following initial shared output; argument zero. |
| F-D02S | First invocation following direct repetition; argument zero. |
| F-D02Z | First invocation following extended-path shared output; argument zero. |
| F-D02B | First invocation following breakpoint resumption; receives the prepared signed-shift result. |
| F-D03 | No explicit arguments or consumed result; precedes F-D04. |
| F-D04 | No explicit arguments or consumed result; precedes F-D05. |
| F-D05 | Receives newly loaded current value; no consumed result. |
| F-D06 | Receives current value logically shifted right by three; supplies primary destination and pending inputs. |
| F-D07 | Receives replacement read address logically shifted right by two; no consumed result. |
| F-D08 | Supplies secondary destination, base-address input, byte-output input, and complete 64-bit output. |

The four F-D02 selections represent distinct call destinations. Their shared result-consumption pattern does not prove interchangeable behavior.

Hidden arguments, full calling conventions, preservation, termination, exceptions, retention, and ownership remain **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes preceding operations complete and the relevant accesses, arithmetic, and conversions admit execution.

1. **Initialization argument reduction.** Argument 4 equal to `0xFFFFFFFF` supplies `0x3FFF` to F-D01. The object identified by its third argument has already been assigned zero.

2. **Captured sample.** A sample conversion yielding `0x0000FFFF` gives initial accumulator `0x0000ECA0`. Initial shared output ends with word `0x4000ECA0`. A subsequent sample-source change does not alter this retained contribution.

3. **Unresolved source width.** If sample conversion yields `0xFFFFFFFF`, the initial accumulator is `0x0897ECA0`. Requiring unconditional 16-bit zero extension would incorrectly exclude this conditional case.

4. **Overlapping initial output.** Byte zero, halfword `0x64FE`, word zero, and the updated accumulator word must occur in that order at the initial output location. Matching only its final contents is insufficient.

5. **Aliasing and read timing.** If an early transaction write changes the replacement-address slot, the later slot read observes the resulting storage. The replacement does not retroactively change the address used for the earlier current-value read.

6. **Direct repetition.** An F-D06 signed conversion yielding zero produces pending zero without multiplication overflow. The self-addressed zero-word write precedes F-D02S.

7. **Extended selection.** An F-D06 signed conversion yielding one produces pending 32. Word 32 is written at address 32 before extended output begins.

8. **No invented signed wraparound.** Signed `0x08000000` multiplied by 32 exceeds signed 32-bit range. Pending zero and direct repetition are not established consequences.

9. **Captured-byte lifetime.** The initial transaction captures byte zero from `0x40000000`. Loading a current value with another low byte does not change that transaction’s captured byte.

10. **Result-region extent.** F-D08 output requires a complete eight-byte result transfer at the converted base and a separate byte transfer at offset eight, subject to any additional overlapping writes.

11. **Breakpoint boundary.** Conditional on a meaningful nonzero predicate, its word output precedes breakpoint four. Working-value replacement and F-D02B occur only after resumption.

## Uncertainties

The following remain **documented_only**:

- Fixed-object declarations, widths, bindings, and overlap.
- Sample-source width and conversion details.
- Scaling-source arithmetic type.
- Hardware or application meaning of storage roles.
- Signed overflow, invalid shifts, negative signed right shifts, and out-of-range narrowing.
- Complete-result address conversions and pointer-to-integer behavior.
- Valid mappings, alignment, extent, permissions, and lifetime.
- Meaning and reachability of the final predicate.
- ABI interpretation of 64-bit results and post-F-D01 register state.
- Dependency hidden effects, preservation, ownership, retention, failure, and termination.
- Endianness, atomicity, external ordering, and breakpoint resumption.
- Behavior omitted from the supplied static representation.

No universally safe input domain or normal termination condition is established.

## Evidence

Established claims are limited to explicit argument uses, transfer widths, constants, arithmetic types, result-component selections, retained-value relationships, and observable ordering in the supplied static representation.

Storage labels and comments are not treated as proof of hardware roles, physical widths, ownership, or omitted behavior. Missing declarations and dependency implementations have not been reconstructed.

Inferred intent is identified separately. Conditional examples express consequences of the static evidence, not observed execution.

No runtime tests have been performed.
