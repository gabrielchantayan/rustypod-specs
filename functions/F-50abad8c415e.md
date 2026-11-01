<!-- Trusted provenance: F-50abad8c415e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-50abad8c415e

## Purpose

F-50abad8c415e conditionally advances a signed 32-bit value in caller-provided storage. Acceptance depends on a selector match, a selected signed upper bound, and a second observation of the value being advanced.

An accepted operation writes the proposed value into the record. It may then publish the sum of that proposal and a record-held base through independently located storage. Disabling publication does not disable the accepted record update.

**Established static evidence:** the accessed record layout, acceptance comparisons, conditional writes, and ordering described below.

**Inferred intent:** the operation may advance a bounded position and publish a corresponding base-relative position. This interpretation adds no acceptance conditions.

**Unknown:** application meaning, complete storage declarations, ownership, supported aliasing and concurrency arrangements, and behavior after invalid accesses or unrepresentable arithmetic. In particular, the publication destination’s byte displacement cannot be established without the missing shared-storage declaration.

## Interface

### Inputs and result

The operation takes:

- A record address.
- An unsigned 32-bit selector.

It has no explicit return value or status output. Normal return does not distinguish rejection, acceptance without publication, or acceptance with publication.

### Record access layout

The following offsets are byte offsets from the record address.

| Offset | Access width | Interpretation |
|---|---:|---|
| 0 | 1 byte | Unsigned stored selector |
| 4 | 4 bytes | Signed 32-bit current value |
| 8 | 4 bytes | Signed 32-bit publication base |
| 16 + 4 × selector | 4 bytes | Selected signed 32-bit upper bound |

This is an access layout, not a complete record declaration. It establishes neither the total allocation size nor the meaning of unlisted storage.

The stored selector byte is compared with the entire unsigned 32-bit argument. The argument is not truncated to a byte. Consequently, only arguments from 0 through 255 can match.

The selected bound is accessed only after a selector match. Reachable bound starting offsets therefore range from 16 through 1,036, inclusive, in four-byte steps. Selector 255 selects bytes 1,036–1,039. The multiplication and offset arithmetic for these reachable indices fit within 32 bits.

### Publication storage and destination formation

Publication involves three storage roles:

| Role | Meaning |
|---|---|
| Shared locator | Identifies storage containing a publication reference |
| Slot | Contains the reference tested for enablement and separately evaluated for destination formation |
| Publication destination | Receives a four-byte signed value at the address formed from the assignment-side reference |

A zero **slot value** disables publication. A zero shared locator is not itself a checked disablement condition: the operation must access the selected slot to obtain the value tested against zero.

The destination is formed by adding the numeric displacement **36** to the assignment-side reference, then interpreting the resulting address as the location for a signed four-byte store.

The byte extent of that displacement is **documented_only**. The shared declaration is absent, so the reference’s arithmetic type is unknown:

- If the reference uses integer arithmetic, addition follows that integer type’s width and arithmetic rules before address interpretation.
- If it uses pointer arithmetic, the displacement is scaled by the pointed-to element size.
- The eventual interpretation as a four-byte signed destination does not establish the units of the preceding addition.

The contract therefore does **not** identify the publication destination as a fixed byte offset from the reference. A concrete byte address requires the missing type, representation, and address-formation rules.

The widths and signedness of both the shared locator and slot contents are also **documented_only**. Their address-forming roles do not establish native pointer width or an integer-address convention.

### Preconditions

Each reached access requires valid storage with sufficient extent, lifetime, permissions, and alignment. No visible validation establishes those properties.

Signed 32-bit values range from −2,147,483,648 through 2,147,483,647. For behavior restricted to representable signed arithmetic:

- The initial current value plus one must fit in that range.
- If publication is enabled and reached, the publication base plus the retained proposal must fit in that range.

These are conditional preconditions, not checks performed by the operation. No wrapping, saturation, or overflow-reporting policy is established.

Publication additionally requires valid destination formation under the unknown reference type and valid storage at the resulting destination. A nonzero enablement reference alone establishes neither property.

## Behavior

### Proposal and acceptance

The operation first observes the signed current value at record offset 4 and forms a **proposal** one greater than that observation. This proposal is retained for the comparisons and potential writes.

For valid reached accesses and representable arithmetic, acceptance requires all three relationships:

| Condition | Required relationship |
|---|---|
| Selector match | Stored unsigned byte equals the full unsigned 32-bit selector |
| Upper-bound eligibility | Proposal is less than or equal to the selected signed upper bound |
| Later-value eligibility | Proposal is greater than or equal to a later signed observation of the current-value field |

Both numerical boundaries are inclusive.

The later current-value observation is distinct from the initial observation. Under stable storage and representable increment arithmetic, the later-value condition necessarily passes. The contract does not assume stability or replace the later observation with the initial one.

Rejection causes no explicit write by this operation. Acceptance causes the record update regardless of whether publication is subsequently enabled.

### Reachability and ordering

The supplied static representation establishes these evaluation and control-flow constraints:

1. The initial current-value observation and increment precede selector rejection.
2. A selector mismatch excludes the bound access, later current-value observation, publication accesses, and explicit writes.
3. A proposal above the selected bound excludes the later current-value observation, publication accesses, and explicit writes.
4. A proposal below the later current-value observation excludes publication accesses and explicit writes.
5. All acceptance conditions precede the record update.
6. The record update precedes the publication enablement test.
7. An enabled publication write follows the record update.

These are constraints of the static representation, not guarantees of physical memory transactions, atomicity, or inter-thread ordering.

Even a selector that cannot match does not avoid the initial read and increment. Conversely, a rejected request does not require valid publication storage because publication is not reached.

### Accepted record update

Acceptance writes the retained proposal as a signed four-byte value to record offset 4.

The update is not conditional on publication enablement. No visible rollback compensates for a subsequent invalid publication access or unrepresentable publication sum.

The proposal used for publication is the retained value. It is not obtained by rereading the current-value field after the update.

### Publication

After the record update, the operation evaluates the shared locator and accesses the selected slot for the enablement test.

- A zero slot value suppresses publication.
- A nonzero slot value permits the publication assignment.

Destination formation separately evaluates the shared locator and slot. The enablement reference is not established as a saved destination. Therefore, the assignment can use a different reference if either the locator or the selected slot contents differ between evaluations.

Stability of only one level is insufficient to establish destination stability. A stable locator can identify a slot whose contents change; stable contents in one slot do not matter if a different locator selects another slot.

There is no visible second zero check on the assignment-side reference. If enablement observes a nonzero reference but destination formation obtains zero, the operation does not convert that case into checked, disabled publication. The resulting displacement and address validity remain dependent on the missing type and environment rules.

The publication value is the signed base at record offset 8 plus the retained proposal. The base is observed after the record update. Its order relative to assignment-side destination formation is not established.

The resulting signed four-byte value is written to the destination formed using the type-dependent numeric displacement of 36. The operation does not compare the destination’s old value before writing. Valid enabled publication therefore includes a write even if the destination already contains the same numerical value.

## Side effects

### Explicit writes

For valid accesses and representable arithmetic:

| Outcome | Explicit writes |
|---|---|
| Rejection by any acceptance condition | None |
| Acceptance with publication disabled | Proposal to record offset 4 |
| Acceptance with valid enabled publication | Proposal to record offset 4, followed by base-plus-proposal to the publication destination |

These describe writes, not unconditional final-state guarantees. Overlapping storage can allow the publication write to replace or partially modify the record update.

### Aliasing and storage stability

No general non-overlap requirement is visible among the record, shared locator, slots, and publication destination.

Within the record layout, the current-value and base fields occupy separate four-byte ranges. Independently addressed shared or publication storage can nevertheless overlap record storage if the surrounding environment permits it.

If the current-value field overlaps a locator or consulted slot, the record update can affect the subsequent publication lookup. Enablement uses the shared-storage observations made after that update, not necessarily entry-time contents.

A valid publication destination could also overlap:

- The current-value field, replacing or partially modifying the proposal.
- The base field, modifying it after its operand has been obtained.
- The selected bound, modifying it after the acceptance comparison.
- The shared locator or a slot, affecting later operations.

These are conditional possibilities, not established supported configurations. In particular, whether a reference can form any specified overlapping destination depends on the unresolved displacement and address rules.

Partial-overlap results additionally depend on representation, byte order, access widths, alignment, and permitted aliasing. No byte-level final state is promised without those prerequisites.

### Ownership and errors

There is no visible allocation, release, ownership transfer, or lifetime extension. The operation directly accesses supplied and shared storage.

No explicit error result, retry, recovery path, or transactional protection is present. An invalid access or unrepresentable arithmetic operation can occur before rejection, during acceptance evaluation, or after the primary update.

The absence of a visible rollback does not establish what remains observable after a fault or an operation outside the defined arithmetic contract. In particular, a later publication failure is not specified as either successful record-only completion or restoration of the previous record value.

## Dependencies

No function calls are visible. There are no dependency function IDs to enumerate.

The operation depends directly on:

- Reached record fields.
- The shared locator and consulted slots.
- The publication destination when publication is reached.
- The execution environment’s arithmetic, address formation, alignment, and memory-access rules.

The missing shared-storage declaration is necessary to resolve the units and width of publication-reference arithmetic. The final four-byte destination interpretation does not supply that declaration.

The absence of calls does not establish exclusive access, stable globals, atomicity, or a supported concurrent-access protocol.

## Acceptance examples

These are conditional static examples, not runtime observations. Unless stated otherwise, assume valid reached accesses, representable arithmetic, stable relevant storage, and no overlap between the record and publication storage.

### Full-width selector rejection

An initial current value of 10 produces proposal 11.

With stored selector 3 and argument 4, rejection occurs without a bound read, later current-value observation, publication access, or explicit write.

Argument 256 cannot match any stored selector byte, including zero. The initial read and increment still occur before rejection; the argument is not reduced to its low byte.

### Inclusive upper bound with publication disabled

Matching selectors, initial current value 10, selected bound 11, and later current-value observation 10 satisfy all acceptance conditions.

If the consulted slot is valid and contains zero, the record receives 11 and publication is skipped. Under the stated separation and stability assumptions, the final current value is 11.

### Upper-bound rejection

Matching selectors, initial current value 10, and selected bound 10 make proposal 11 ineligible.

There is no later current-value observation, publication access, or explicit write.

### Later-observation boundary

With matching selectors, initial current value 10, and selected bound 20, the retained proposal is 11.

| Later current-value observation | Outcome |
|---:|---|
| 12 | Reject |
| 11 | Accept |
| 9 | Accept |

These cases establish the comparison boundary. They do not establish that concurrent modification is supported or explain how differing observations can validly arise.

### Signed values and enabled publication

Matching selectors, initial current value −3, selected bound −2, and later observation −3 permit proposal −2.

With base 100 and valid enabled publication, the record receives −2 and the destination receives 98.

This example assumes that the actual, presently unknown reference arithmetic forms a valid destination. It does not locate that destination at a fixed byte distance from the reference.

### Publication replacing the record update

Initial current value 7, selected bound 20, and later observation 7 permit proposal 8. A base of 100 gives publication value 108.

If the actual destination-formation rules and permitted aliasing make the publication destination exactly coincide with the current-value field, that field is written first with 8 and then with 108. Its final value is 108 under those conditions.

No particular reference value is asserted to produce that coincidence.

### Enablement does not pin the destination

Suppose enablement observes a nonzero reference, but the assignment-side evaluation obtains a reference identifying different storage. Publication follows the assignment-side reference and its type-dependent displacement.

The difference can arise from a changed locator or changed slot contents. If the assignment-side reference is zero, there is no second visible zero check; neither successful publication nor checked disablement is promised.

### Primary update affecting publication selection

If permitted aliasing places a consulted slot over the current-value field, enablement observes that storage after the accepted update.

If the current-value field overlaps the locator instead, the update can affect which slot is consulted. The resulting reference or destination cannot be calculated from the accepted integer alone without the missing widths, representations, and displacement rules.

### Arithmetic boundaries

Initial current value 2,147,483,646 produces the representable proposal 2,147,483,647. Acceptance still requires a matching selector, a sufficient signed bound, and a passing later-value comparison.

Initial current value 2,147,483,647 has no representable signed 32-bit increment. Selector mismatch does not avoid that arithmetic.

Publication can overflow even when the proposal is representable. Proposal 1 and base 2,147,483,647 exceed the signed range when added. No wrapped value, saturation, or error status is promised.

## Uncertainties

The following remain **documented_only**:

- Application meaning and the complete record declaration.
- Allocation extents, lifetimes, permissions, and ownership.
- Widths, signedness, and representations of the shared locator and slot contents.
- Whether publication-reference addition is integer arithmetic or scaled pointer arithmetic.
- The byte extent corresponding to the numeric displacement of 36.
- Address conversion, address overflow, and destination validity.
- Alignment requirements, byte order, and permitted aliasing.
- Stability of repeated observations.
- Atomicity, synchronization, and concurrent-access rules.
- Evaluation order between the publication-base observation and assignment-side destination formation.
- Treatment of unrepresentable signed arithmetic.
- Fault delivery and observable state after invalid operations.
- Whether surrounding contracts permit the conditional aliasing examples.

The bounded-position interpretation remains **inferred intent**, not an acceptance requirement.

## Evidence

The supplied static fragment supports the two inputs, lack of an explicit result, record access layout, full-width selector comparison, signed numerical relationships, inclusive acceptance boundaries, retained proposal, and conditional write outcomes.

It supports the initial increment before selector rejection, a distinct later current-value observation, the record update before publication enablement, and separate shared-storage evaluations for enablement and destination formation.

For publication, it establishes addition of the numeric displacement 36 before interpretation as a signed four-byte destination. It does not supply the shared declaration needed to determine that addition’s units. Neither a fixed byte displacement nor a particular pointer element size is established.

The fragment does not establish shared-address representations, storage stability, ownership, non-aliasing, synchronization, transactional guarantees, or runtime failure semantics.

No runtime tests have been performed.
