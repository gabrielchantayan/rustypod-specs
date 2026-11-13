<!-- Trusted provenance: F-b5aeaa4e2997; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b5aeaa4e2997

## Purpose

F-b5aeaa4e2997 performs configuration-dependent memory updates and transfers control to opaque dependencies. Two routes are established:

- **Direct route:** populate a 16-byte destination, update a separate four-byte location, and invoke a continuation. An ambient arithmetic condition can instead select an exceptional sequence.
- **Contextual route:** perform preparation writes and dependent reads, then either populate a 28-byte destination and invoke a continuation, invoke an alternative dependency, or encounter an invalid-instruction boundary.

“Record,” “preparation,” and “continuation” describe observable roles only. They do not establish application types, publication, persistence, or successful consumption.

No ordinary return path is present in the supplied representation. This does not establish that every invocation terminates: inaccessible memory, opaque dependencies, and instruction handling may prevent subsequent effects.

## Interface

### Explicit inputs

| Input | Width | Established role |
|---|---:|---|
| First opaque input | 32 bits | No direct use is represented. |
| Second opaque input | 32 bits | No direct use is represented. |
| Numeric input | Signed 32 bits | Supplies an entry byte, contributes to an entry-write address, determines the direct destination, and contributes a continuation argument. |
| Configuration reference | 32-bit address representation | Identifies configuration fields and an entry-write destination. |

The represented execution model uses four-byte addresses. Wider-address behavior is not established.

### Configuration access layout

Offsets are byte offsets from the configuration reference.

| Offset | Width | Role |
|---:|---:|---|
| 0 | 4 bytes | Argument to F-unresolved-01 |
| 4 | 4 bytes | Contribution to the second entry-write address |
| 8 | 4 bytes | Route selector |
| 12 | 4 bytes | Second entry-write payload source and direct-update base |
| 16 | 4 bytes | Direct-record contribution or contextual destination |
| 20 | 4 bytes | First contextual preparation displacement |
| 29 | 1 byte | Destination for the numeric input’s low eight bits |

This is an access layout, not an allocation guarantee. It does not establish object extent, alignment, accessibility, or lifetime.

### Ambient inputs

The function also depends on:

- A signed 32-bit operand used to determine whether subtraction of 45 overflows.
- A signed 32-bit linkage contribution used to select F-unresolved-01.
- A four-byte shared payload.
- Shared decision storage whose complete type is unresolved.
- Exceptional-state storage whose assignment width is unresolved.
- Frame-associated values and addresses.

The **frame anchor** is the address placed in the contextual record’s final field. The supplied representation establishes these relationships:

| Position relative to anchor | Role |
|---:|---|
| −184 bytes | Signed 32-bit first preparation base |
| −12 bytes | Signed 32-bit second preparation base |
| −8 bytes | Four-byte contextual-record contribution |
| −4 bytes | Contribution supplying a contextual field’s upper 24 bits |
| +320 bytes | Address passed to F-unresolved-04 |

A frame-associated value is distinct from the address where it is associated. These relationships do not establish independent allocations or immutable storage.

### Results and preconditions

There is no established application-level return value or error code. Observable results consist of memory accesses, dependency invocations, and instruction events.

Each reached access requires suitable storage; each reached call requires a usable target and execution environment. No validation is represented for null references, low addresses, inadequate extents, overlap, alignment, expired storage, or invalid targets.

## Behavior

### Entry contract and retained state

The ambient subtraction-overflow condition is established before the entry writes. It depends on the ambient operand, not the numeric input.

The entry observations and effects have the following ordering:

1. The numeric input’s low eight bits are written at configuration offset 29.
2. Configuration fields at offsets 0, 8, 12, 16, and 20 are captured, in that order.
3. The offset-4 field is observed for the second entry-write address.
4. A second one-byte write occurs.
5. F-unresolved-01 is invoked.

The second byte is the low eight bits of the captured offset-12 field. Its destination combines the full numeric input with the offset-4 contribution. Truncation of the first byte does not truncate the numeric input used in this address calculation. Complete arithmetic typing for this calculation remains **documented_only**.

F-unresolved-01 receives the captured offset-0 field. Its target depends on the ambient linkage contribution increased by 2,384. If it returns, its eight-byte result is retained.

A captured selector of zero selects the direct route; every nonzero value selects the contextual route. Changes to configuration storage after capture do not replace captured values. They may nevertheless affect storage addressed by those values.

Under ordinary, nonwrapping configuration addressing, the first byte write does not overlap the listed four-byte fields. No comparable separation is established for the second byte write.

### Direct-route outcomes

The retained ambient overflow condition selects between the exceptional sequence and ordinary direct construction.

Signed 32-bit subtraction of 45 overflows for ambient operands from −2,147,483,648 through −2,147,483,604, inclusive. On this condition, the represented exceptional sequence consists of:

- A zero assignment to exceptional-state storage.
- Software-interrupt operation 226.
- Undefined-instruction operation 148 with auxiliary operand 144,195,884.
- Invocation of the resulting target, identified here as F-unresolved-06.

The exceptional-state assignment width is unresolved. This route has no represented return to ordinary construction. Handler behavior, target production, and resumption are not established.

Otherwise, the direct destination is the address conversion of the signed numeric input shifted right by 24 bits. For nonnegative inputs, the resulting address representation lies between 0 and 127. These low addresses are not rejected or replaced. Negative signed-right-shift behavior and integer-to-address conversion semantics remain **documented_only**.

The direct record has this layout:

| Destination offset | Width | Content |
|---:|---:|---|
| 0 | 8 bytes | Retained F-unresolved-01 result |
| 8 | 4 bytes | Captured configuration offset-16 value |
| 12 | 4 bytes | Direct destination address representation |

Field writes occur in increasing offset order. They are followed by a separate four-byte write of the captured offset-16 value at the captured offset-12 base displaced by 56 bytes.

F-unresolved-02 receives:

- The retained result’s low 32 bits, converted to signed 32 bits.
- The retained result’s high 32 bits, converted to signed 32 bits.
- The numeric input decreased by 44.

The result arguments are derived from the retained result, not read back from the destination. An overlapping separate update can therefore damage the stored result without replacing those arguments.

If F-unresolved-02 returns, the represented execution encounters an invalid-instruction boundary rather than returning normally.

### Contextual preparation

Contextual preparation occurs before the outcome decision, including when the retained decision difference is zero.

The first preparation write stores the first preparation base’s low 16 bits at that base displaced by the captured configuration offset-20 value. The displacement changes the destination, not the payload.

The four-byte shared payload is captured after this write and before breakpoint operation 217. All decision observations occur after the breakpoint. A breakpoint handler may therefore affect later observations without replacing the retained payload.

Decision storage participates in three logical observations:

| Observation | Established consequence |
|---|---|
| Comparison against 74 | Retains whether the observed value is greater than 74 |
| Subtraction of 75 | Retains a 32-bit difference and whether it is zero |
| Address adjustment by 46 bytes | Identifies the source of an unsigned two-byte read |

These observations are not established to use one stable snapshot. The comparison signedness, complete storage type, and physical read count remain **documented_only**.

The two-byte read produces an unsigned displacement from 0 through 65,535. That displacement selects a four-byte dependent read relative to the captured contextual destination. This address calculation uses unsigned 32-bit addition, wrapping modulo 4,294,967,296.

Both dependent reads precede the second preparation write. That write stores the frame anchor address’s low 16 bits at the second preparation base displaced by 46 bytes.

The dependent value is retained before this second write. Overlap with its original location does not replace the value used for outcome selection.

### Contextual outcome contract

| Retained difference | Retained dependent value | Outcome |
|---|---|---|
| Zero | Any | Invalid-instruction boundary after preparation |
| Nonzero | Zero | F-unresolved-03, record construction, then F-unresolved-04 |
| Nonzero | Nonzero | F-unresolved-05, then an invalid-instruction boundary if it returns |

For the construction outcome, F-unresolved-03 has no visible explicit arguments. If it returns, its eight-byte result is retained. The contextual destination then receives:

| Destination offset | Width | Content |
|---:|---:|---|
| 0 | 8 bytes | Retained F-unresolved-03 result |
| 8 | 4 bytes | Shared payload captured before the breakpoint |
| 12 | 4 bytes | Frame-associated contribution at −8 bytes |
| 16 | 4 bytes | Contextual destination address representation |
| 20 | 4 bytes | Upper 24 bits of the contribution at −4 bytes, combined with the retained difference’s low eight bits |
| 24 | 4 bytes | Frame anchor address representation |

Writes occur in increasing field-offset order. The offset-20 field is a full four-byte replacement; its upper bytes are not preserved from the destination.

The shared payload has an established capture point before the breakpoint. Equivalent acquisition timing is not established for the two frame-associated contributions. Their exposure to intervening calls and overlapping writes remains **documented_only**.

F-unresolved-04 receives the retained result’s low and high 32-bit portions, each converted to signed 32 bits, followed by the address 320 bytes beyond the frame anchor. That address is not established to identify the contextual record. A return encounters an invalid-instruction boundary.

For the alternative outcome, F-unresolved-05 has no visible explicit arguments. Its four-byte result has no reachable consumer. The retained greater-than condition distinguishes subsequent represented paths, but all reachable paths encounter an invalid-instruction boundary if the dependency returns.

F-4747de84c84c is unreachable within the represented control flow. Entering this alternative outcome requires a nonzero retained difference. Reaching that invocation would additionally require the same retained difference’s zero condition. Mutation of shared decision storage cannot reconcile contradictory conditions already retained. Argument-preparation reads exclusive to that invocation are likewise unreachable.

### Arithmetic boundaries

These are representability conditions, not validations performed by the function.

| Signed 32-bit calculation | Condition avoiding overflow |
|---|---|
| Numeric input decreased by 44 | Input at least −2,147,483,604 |
| Linkage contribution increased by 2,384 | Contribution at most 2,147,481,263 |
| Signed-converted configuration address increased by 29 | Converted value at most 2,147,483,618 |
| Direct-update base increased by 56 | Base at most 2,147,483,591 |
| Second preparation base increased by 46 | Base at most 2,147,483,601 |
| First preparation base plus its displacement | Mathematical sum within the signed 32-bit range |

If decision subtraction is signed 32-bit arithmetic, subtraction of 75 overflows below −2,147,483,573. If its address adjustment is signed 32-bit arithmetic, addition of 46 overflows above 2,147,483,601.

Consequences of signed overflow and out-of-range signed conversions remain **documented_only**. The established unsigned wrapping of the dependent-read address does not establish wrapping for other calculations.

## Side effects

Excluding opaque dependencies and instruction handlers, the reached portions have these write footprints:

| Portion | Writes |
|---|---|
| Entry | Two one-byte writes |
| Direct construction and separate update | One eight-byte write and three four-byte writes |
| Contextual preparation | Two two-byte writes |
| Contextual construction | One eight-byte write and five four-byte writes |
| Direct exceptional sequence | Zero assignment of unresolved width |

These widths do not establish atomicity, fault granularity, or all-or-nothing completion. No rollback, cleanup, or recovery mechanism is represented.

Configuration, destinations, preparation storage, shared state, and frame-associated storage are not established to be disjoint. Earlier writes may affect later reads. Later writes may overwrite completed fields. A continuation invocation does not prove that an intact record remains.

No visible operation allocates memory, releases it, extends its lifetime, or transfers ownership. This observation does not constrain opaque dependencies or instruction handlers. Their effects on globals, allocations, ownership, and storage lifetime are unknown.

Retention protects a value from subsequent changes to its backing storage. It does not preserve memory referenced by that value.

## Dependencies

The following IDs identify invocation roles, not necessarily distinct implementations.

| Opaque ID | Established interface |
|---|---|
| F-unresolved-01 | Ambient-selected target; one four-byte argument; eight-byte result retained on return |
| F-unresolved-02 | Direct continuation; two signed-converted result portions and the numeric input decreased by 44 |
| F-unresolved-03 | Contextual result provider; no visible explicit arguments; eight-byte result retained on return |
| F-unresolved-04 | Contextual continuation; two signed-converted result portions and a frame-relative address |
| F-unresolved-05 | Alternative dependency; no visible explicit arguments; four-byte result without a reachable consumer |
| F-unresolved-06 | Exceptional-operation-produced target; no visible explicit arguments |
| F-4747de84c84c | Invocation excluded by contradictory retained conditions |

Except for the ambient-selected and exceptional-operation-produced targets, reachable targets are fixed in the supplied representation rather than selected from configuration storage.

“No visible explicit arguments” does not exclude ambient inputs. Calling conventions, preservation guarantees, termination, memory-management responsibilities, and other effects are not established.

## Acceptance examples

These are conditional static consequences, not executed tests. They assume suitable storage, applicable arithmetic and conversion semantics, and progress through preceding dependencies and instruction events.

1. **Byte truncation does not truncate an address contribution.** Numeric input 770 supplies first-write byte value 2. The second write’s address still incorporates 770 rather than 2.

2. **Captured routing survives configuration mutation.** A nonzero captured selector selects contextual processing even if F-unresolved-01 changes its backing field to zero.

3. **Exceptional selection is ambient and route-specific.** Ambient operand −2,147,483,604 satisfies the overflow condition; −2,147,483,603 does not. A nonzero selector avoids the direct exceptional sequence in either case.

4. **Low direct addresses are not filtered.** Numeric input 50,331,649 produces direct destination representation 3. Construction depends on accessibility there; no replacement destination is provided.

5. **Stored-result corruption does not replace continuation arguments.** If the separate direct update overlaps the stored eight-byte result, F-unresolved-02 still receives portions of the retained result.

6. **Preparation displacement affects location only.** First preparation base 65,543 with displacement 9 selects destination 65,552 and stores two-byte value 7.

7. **Zero difference does not skip preparation.** A subtraction observation of 75 produces zero. If the address observation also sees 75, the two-byte read uses address representation 121. The dependent four-byte read and second preparation write still precede the invalid-instruction boundary.

8. **Decision observations need not agree.** A comparison observation of 100 and subtraction observation of 75 produce a true greater-than condition and a zero difference. The zero-difference outcome applies.

9. **The dependent address wraps.** Contextual destination representation 4,294,967,294 with displacement 5 selects address representation 3.

10. **Field composition replaces all four bytes.** Retained difference 258 and upper-bit contribution 0xA1B2C3D4 produce offset-20 field value 0xA1B2C302, independently of the destination’s prior contents.

11. **Capture guarantees are limited.** Mutation of shared-payload storage during F-unresolved-03 does not replace the retained payload. Equivalent protection is not established for frame-associated contributions.

12. **Shared-state mutation cannot enable the excluded call.** A nonzero retained difference remains incompatible with its retained zero condition even if F-unresolved-05 modifies decision storage.

## Uncertainties

The following remain **documented_only**:

- Exceptional-state assignment width and exceptional target-production mechanics.
- Instruction handling, including delivery, termination, and resumption.
- Decision-storage type, comparison signedness, arithmetic typing, physical read count, and stability.
- Complete arithmetic typing of the second entry-write address.
- Negative signed-right-shift and integer-to-address conversion semantics.
- Signed-overflow consequences and out-of-range signed conversions.
- Acquisition timing of frame-associated record contributions.
- Storage extents, alignment, lifetime, byte order, atomicity, and partial-write behavior.
- Dependency effects, preservation guarantees, termination, and ownership responsibilities.
- Application meaning, publication, persistence, and consumption protocols.

These unknowns do not authorize assumptions of recoverable faults, accessible low addresses, disjoint storage, preserved globals, or automatic cleanup.

## Evidence

**Established static evidence:** The supplied function representation supports the stated access widths, configuration capture order, destination layouts, dependency arguments, preparation ordering, retained-condition relationships, and instruction boundaries. No ordinary return path is represented.

**Derived static conclusions:** Width-sensitive arithmetic establishes the ambient overflow interval, listed representability limits, unsigned displacement range, and wrapping dependent-read address. Contradictory conditions on one retained difference exclude F-4747de84c84c without requiring stable shared storage.

**Inferred intent:** Contiguous destination writes may prepare records for subsequent processing. Application-level record types and publication or ownership protocols are not established.

**Evidence limits:** Comments, the prior draft, and reviewer feedback are not independent behavioral proof. Dependency implementations and execution-environment semantics are unavailable. No runtime tests have been performed.
