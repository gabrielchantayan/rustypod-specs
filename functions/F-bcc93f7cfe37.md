<!-- Trusted provenance: F-bcc93f7cfe37; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bcc93f7cfe37

## Purpose

F-bcc93f7cfe37 selectively adjusts two adjacent signed integers in caller-provided storage. Each element qualifies independently when its value is strictly greater than a shared threshold. Every qualifying element receives the same signed adjustment once.

The threshold governs selection, not the permitted result. A negative adjustment may move a qualifying element below the threshold. The operation does not clamp values, maintain ordering between the elements, or repeat adjustments until a boundary is reached.

The storage is modified in place. No result value or status is returned. The application-specific meaning of the pair is not established.

## Interface

### Inputs and output

| Input | Meaning |
|---|---|
| Storage address | Address of the first element in a pair of consecutive signed integers. |
| Threshold | Signed value used as the strict eligibility boundary for both elements. |
| Adjustment | Signed value added once to each qualifying element. |

The threshold and adjustment are value inputs. Writes to the pair do not alter these invocation-specific scalar values.

The output consists solely of the resulting contents of the addressed pair. There is no returned replacement address, count of changes, success indicator, or error code.

### Integer width and storage layout

The elements, threshold, and adjustment have the same declared signed-integer type. The supplied evidence does not establish its numerical width, storage size, byte order, alignment, or signed representation. In particular, a 32-bit width is not an established requirement.

The pair occupies two consecutive element positions. The second element’s displacement from the first is one element storage size; no fixed byte displacement is established. There is no explicit length field, header, terminator, or access to a third element.

In this contract, **minimum** and **maximum** mean the least and greatest representable values of this signed-integer type. These names do not assume a particular representation.

### Preconditions for defined completion

The caller must provide:

- A live storage region in which both consecutive element positions are valid to access.
- Valid readable integer values at both positions.
- Write permission for every element that qualifies, including when the adjustment is zero.
- Representable results for every selected addition.
- Compliance with the applicable object-lifetime, alignment, and access rules.

Both elements must be readable even when neither qualifies. The operation has no empty-input convention and accepts no length argument. A null address or storage containing only one accessible element does not satisfy the contract.

Numerical outcomes below assume that the pair remains stable except for this operation’s own writes. No concurrent-mutation or device-memory protocol is established.

## Behavior

### Selection and numerical result

Eligibility uses signed ordering. An element equal to the threshold is excluded; an element below the threshold is also excluded. Excluded elements retain their values.

Each qualifying element receives exactly one addition of the supplied adjustment. The adjustment’s sign and magnitude do not affect selection. Consequently:

- A positive adjustment does not trigger further adjustment.
- A negative adjustment may produce a result equal to or below the threshold.
- A negative adjustment cannot make an initially excluded element eligible during this invocation.
- A zero adjustment preserves numerical contents even where an assignment is selected.
- The first element’s numerical value does not determine the second element’s eligibility.

There is no requirement that the final pair remain sorted or retain its original relative ordering. There is also no post-adjustment threshold check, saturation, or range correction.

### Semantic access order

Processing of the first element precedes the second element’s eligibility observation. When the first element qualifies, its assignment occurs before that second observation.

The second element is considered regardless of the first element’s eligibility. Neither an excluded first element nor a zero adjustment establishes an early-completion case.

Each eligibility decision observes the corresponding stored value. A selected addition also obtains its arithmetic operand from that same element. With stable storage, the eligibility value and arithmetic operand agree. The evidence does not establish a supported outcome if another actor changes storage between those observations.

These are requirements on the represented evaluation and assignment order, not an exact hardware transaction trace. They do not establish separate machine loads for every represented observation, mandatory physical stores of unchanged values, or a particular instruction sequence.

The operation does not provide an atomic snapshot of both inputs or an indivisible replacement of both outputs. Its local ordering is not, by itself, a synchronization guarantee for another execution context.

### Arithmetic boundaries

Selected additions use the common signed-integer type. A mathematically exact result equal to minimum or maximum is permitted. A selected result outside that type’s representable range has no defined result under the represented signed-arithmetic semantics.

No wrapping, saturation, checked rejection, or error return is provided for overflow. A final-pair result must not be required for an invocation containing a selected overflowing addition.

Representability matters only for additions that are selected. An excluded element does not acquire an overflow condition merely because adding the adjustment to it would have exceeded the range.

Selection uses direct signed comparison; it does not require an arithmetic difference between an element and the threshold. No subtraction-overflow condition follows from the comparison.

Two width-independent boundary properties are established:

- A threshold of maximum excludes every representable element value.
- An element whose eligibility value is minimum cannot qualify against any representable threshold.

An eligible value above minimum may nevertheless become minimum through a representable negative adjustment.

### Completion and invalid execution

Normal completion includes consideration of both elements and all selected assignments. It produces no explicit completion signal.

No address validation, access-permission check, or arithmetic-overflow handling is present. Invalid storage, prohibited writes, and selected signed overflow therefore have no specified recoverable outcome.

There is no rollback mechanism. This does not establish a partial-output guarantee after invalid execution. For example, an inaccessible second element does not guarantee that a preceding first-element update remains observable.

## Side effects

The explicit writable state consists of the qualifying elements in the supplied pair. Excluded elements have no selected assignment. No explicit modification of unrelated storage or global state is shown.

A numerically unchanged pair does not prove that no assignments were selected. An eligible element with zero adjustment follows the assignment behavior while retaining its value. That semantic distinction does not require a distinguishable hardware store.

No explicit allocation, deallocation, callback, logging, external I/O, or address retention appears. No ownership transfer is expressed. The caller remains responsible for providing storage with the required lifetime and permissions; the operation does not create or return replacement storage.

No locking, publication mechanism, or pair-wide atomicity is established. Callers needing coordinated visibility of both results require an external protocol.

These statements describe the explicit behavior in the supplied evidence. They do not characterize environmental fault handling or instrumentation.

## Dependencies

No explicit dependency calls are present. There are no dependency function IDs to enumerate.

The operation relies on signed comparison, signed addition, and valid access to caller-provided storage. Its described behavior requires no assumptions about an unknown callee preserving globals, retaining an address, or allocating or releasing memory.

The target calling convention, parameter transport, generated instruction sequence, and environmental response to invalid access are not established.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. Unless stated otherwise, each example assumes valid stable storage, permission for selected writes, and representability of all inputs and selected results.

### Selection and adjustment

| Initial pair | Threshold | Adjustment | Required final pair | Acceptance distinction |
|---|---:|---:|---|---|
| (4, 9) | 5 | 3 | (4, 12) | Only the second element qualifies. |
| (9, 4) | 5 | 3 | (12, 4) | Only the first element qualifies. |
| (6, 9) | 5 | 3 | (9, 12) | Both elements qualify independently. |
| (5, 4) | 5 | 3 | (5, 4) | Equality is excluded, as are smaller values. |
| (−4, −8) | −6 | 2 | (−2, −8) | Selection follows signed ordering. |
| (6, 9) | 5 | −8 | (−2, 1) | Selected results may fall below the threshold. |
| (6, 5) | 5 | −3 | (3, 5) | Relative ordering may reverse. |
| (6, 9) | 5 | −3 | (3, 6) | Each qualifying element receives one adjustment. |
| (6, 5) | 5 | 0 | (6, 5) | The first element selects assignment without numerical change. |

### Representable endpoints

| Initial pair | Threshold | Adjustment | Required interpretation |
|---|---|---|---|
| (maximum minus one, 0) | 0 | 1 | The final pair is (maximum, 0). Reaching maximum is permitted. |
| (maximum minus one, 0) | 0 | 2 | The first selected addition exceeds maximum. No defined final pair is required. |
| (minimum plus one, 0) | minimum | −1 | Both qualify. The final pair is (minimum, −1). |
| (minimum plus one, 0) | minimum | −2 | The first selected addition falls below minimum. No defined final pair is required. |
| (minimum, maximum) | maximum | maximum | Neither qualifies. The pair remains unchanged; hypothetical additions are irrelevant. |

These endpoint examples do not establish concrete integer bounds or a binary representation.

### Access, ordering, and failure conditions

- **First excluded, second eligible:** Normal completion still adjusts the second element.
- **Neither eligible:** Both elements remain readable requirements; neither selects a write.
- **Both eligible:** The first assignment precedes the second eligibility observation. An indivisible pair update is not required.
- **Eligible element readable but not writable:** The preconditions are unsatisfied, including with zero adjustment. No checked rejection or unchanged-output guarantee is established.
- **Only one accessible element:** The input is insufficient regardless of the first element’s value.
- **Second element inaccessible:** Successful completion and persistence of an earlier first-element update are not guaranteed.
- **Null address:** No successful no-op behavior is specified.
- **Zero adjustment:** Unchanged contents are compatible with selected assignments.

These conditions must not be interpreted as requirements for specific machine-access counts, fault types, or externally observable partial writes.

## Uncertainties

**documented_only — Integer representation and layout.** The concrete bit width, storage size, byte order, alignment, padding, signed representation, and numerical endpoints are unresolved. A shared declared type establishes consistency among operands, not a complete binary format.

**documented_only — Physical execution.** Machine instructions, load and store counts, access granularity, same-value store elimination, and cross-context visibility are unresolved. Semantic ordering alone does not establish a hardware memory protocol.

**documented_only — Invalid execution.** No particular fault, diagnostic, recovery behavior, continuation, or partial output is established for invalid access, prohibited writes, or selected signed overflow.

**documented_only — Externally changing storage.** Concurrent mutation and device-backed memory have no established contract here. The represented observations are not an atomic snapshot.

**documented_only — Binary interface.** Calling convention and parameter transport are unspecified.

**documented_only — Application meaning.** The pair’s domain meaning and any broader ownership protocol are unknown. Coordinate semantics, domain-specific bounds, and an ordering invariant are not established.

**Inferred intent.** Selective application of a shared offset is a reasonable description of purpose. It adds no requirements beyond the evidenced selection, arithmetic, ordering, and storage effects.

## Evidence

The supplied static evidence establishes an address input, two value inputs of the same signed-integer type as the stored elements, and no returned value. It supports independent strict-threshold selection for two consecutive elements and one selected addition-and-assignment for each qualifying element.

It also establishes first-element processing before second-element eligibility evaluation, with continued processing when the first element is excluded. No explicit calls, validation, overflow handling, allocation, release, diagnostics, retained address, or unrelated state changes are shown.

The acceptance examples and endpoint properties are deductions under the stated preconditions. Representation details, physical transactions, invalid-execution outcomes, and application meaning remain unresolved.

No runtime tests have been performed.
