<!-- Trusted provenance: F-da419d18b7cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da419d18b7cb

## Purpose

F-da419d18b7cb conditionally replaces one byte, obtains data through external interactions, and produces an ordered history of six four-field records. Five records use one retained destination; the fifth record in the overall history uses companion storage instead.

The observable contract includes intermediate memory states and their relationship to dependency invocations. Producing only the final record is insufficient: intervening dependencies execute after earlier records have been written.

“Publication” below means four ordered, four-byte writes. It does not imply atomicity, synchronization, persistence, notification, or ownership transfer.

The supplied static representation establishes no successful normal return. Rejection leads to an invalid-instruction boundary if preceding interactions permit progress. Completion of the publication history leads to an exceptional interaction with unresolved continuation.

## Interface

### Explicit inputs

| Role | Width | Established meaning |
|---|---:|---|
| First opaque input | 32 bits | No direct use is established |
| Second opaque input | 32 bits | No direct use is established |
| Replacement-address precursor | Signed 32 bits | Numerical address is formed by adding 11 |
| Replacement value | 8 bits | Value of the admitted byte write |

The opaque inputs are not explicitly supplied to any depicted dependency. Their availability through implicit execution state is **documented_only**.

The replacement-address calculation is representable for precursors at or below 2,147,483,636. Addition of positive 11 cannot cause signed underflow. Behavior for larger precursors is **documented_only**; wraparound is not required.

A representable result does not establish a valid address. No special behavior is established for a zero or negative result, inaccessible memory, or a destination lacking write permission.

### Implicit inputs

Execution also depends on:

- An eight-bit condition used at five admission checkpoints.
- A signed 32-bit selector used for the first dependency selection.
- Primary and companion storage references.
- Dependency-associated 32-bit quantities.
- A 64-bit quantity supplied by one acquisition dependency.

Only zero versus nonzero matters for the condition. Its production and modification are unresolved. The contract must preserve the separate checkpoint decisions rather than impose a single entry-time decision.

Primary and companion denote storage roles. No dependency preservation contract establishes that references, referenced contents, or other implicit state remain unchanged across calls.

The 32-bit dependency-associated quantities are not established as ordinary return values, status codes, or identifiers. Their delivery convention and application meaning remain **documented_only**.

### Outputs and prerequisites

Observable outputs are memory accesses, dependency interactions, and exceptional control flow. No normal return value is established.

Acquisition fields and record fields are four bytes wide. The destination reference comes from a four-byte primary field. Its interpretation as an address and its representation when published into a four-byte field require an execution model compatible with those accesses; applicability to wider address spaces is unresolved.

Every access requires suitable storage extent, permissions, lifetime, and any required alignment. The representation establishes no validation, recovery, or rollback. All later effects are conditional on earlier accesses and dependencies allowing execution to continue.

## Behavior

### Admission contract

Admission requires nonzero observations at all five checkpoints. The following table defines the externally visible branch distinctions.

| Checkpoint | Interaction on nonzero observation | Interaction on zero observation | Zero outcome |
|---:|---|---|---|
| 1 | F-000000000002 | F-000000000001 | Rejection |
| 2 | F-000000000003 | None specific to rejection | Rejection |
| 3 | F-000000000005 | F-000000000004 | Rejection |
| 4 | F-000000000006 | F-000000000012 | Rejection |
| 5 | Admission to memory and acquisition work | None specific to rejection | Rejection |

Each nonzero-path interaction lies between its checkpoint and the next checkpoint. Thus dependencies may affect the circumstances of later admission decisions. A rejection-path interaction does not establish a retry.

These admission interactions have no established explicit arguments. That does not exclude implicit inputs or side effects.

The first two alternative targets depend on signed additions to the selector:

| Dependency selected at checkpoint 1 | Added amount | Greatest selector with representable signed sum |
|---|---:|---:|
| F-000000000001 | 4,056 | 2,147,479,591 |
| F-000000000002 | 4,006 | 2,147,479,641 |

Neither addition can underflow signed 32-bit range. Overflow behavior, conversion of the sum into an executable target, and target validity are **documented_only**.

Rejection reaches an invalid-instruction boundary when preceding interactions permit continuation. It performs none of this operation’s direct byte-replacement or record-publication writes. Dependencies already invoked may nevertheless have changed memory, resources, or implicit state.

### Replacement and acquired data

Successful admission authorizes exactly one direct one-byte replacement at the address formed from the precursor plus 11. The replacement precedes all direct primary and companion reads listed below. The previous byte is not directly preserved or returned.

Acquisition dependencies consume four 32-bit arguments. Arguments one, three, and four are stored field values, not pointers to those fields. Argument two is the quantity associated with the preceding interaction.

| Consumer | Storage role | Offsets supplying arguments 1, 3, and 4 | Association supplying argument 2 |
|---|---|---|---|
| F-000000000007 | Primary | 0, 4, 8 | F-000000000006 |
| F-000000000008 | Companion | 0, 4, 8 | F-000000000007 |
| F-000000000009 | Primary | 20, 24, 28 | F-000000000008 |
| F-00000000000a, acquisition role | Companion | 20, 24, 28 | F-000000000009 |
| F-00000000000b | Companion | 40, 44, 48 | Acquisition invocation of F-00000000000a |

The dependency relationships in the table also establish invocation order. Each consumer’s field reads occur after the preceding interaction and before the consumer. Relative evaluation order among the three reads for a particular consumer is not established.

There is no aggregate input snapshot. A dependency may alter fields or references relevant to later acquisitions. Likewise, an overlapping byte replacement may affect an acquisition value.

### Interpretation of the 64-bit quantity

F-00000000000b supplies the 64-bit quantity consumed by F-00000000000c. The latter receives four arguments:

| Position | Required relationship |
|---:|---|
| 1 | The complete 64-bit quantity converted to signed 32 bits |
| 2 | Its upper 32 numerical bits, selected under unsigned 64-bit interpretation, then converted to signed 32 bits |
| 3 | Four-byte primary field at offset 40 |
| 4 | Four-byte primary field at offset 44 |

The two primary reads occur after F-00000000000b and before F-00000000000c; their relative order is unresolved.

The first conversion must not be replaced unconditionally with a specified low-half extraction. A numerical value representable in the signed 32-bit destination retains that value; out-of-range conversion behavior is **documented_only**. The original quantity’s signedness is not independently established.

For the second argument, upper-half values from zero through 2,147,483,647 retain their value. Conversion of larger upper-half values is **documented_only**. Selecting the upper numerical half does not depend on memory byte order.

### Retained state and record format

After F-00000000000c, three four-byte primary fields are acquired in a defined order:

| Acquisition position | Primary byte offset | Retained role |
|---:|---:|---|
| 1 | 56 | First payload |
| 2 | 64 | Destination reference |
| 3 | 68 | Second payload |

All three reads precede publication one, without an intervening dependency. Their retained values serve the entire publication history. Later changes confined to these source fields neither refresh the payloads nor redirect the retained destination.

Every record has this layout:

| Relative byte offset | Width | Content |
|---:|---:|---|
| 0 | 4 bytes | Publication-specific leading quantity |
| 4 | 4 bytes | Retained first payload |
| 8 | 4 bytes | Retained destination reference |
| 12 | 4 bytes | Retained second payload |

Within each publication, writes occur in increasing offset order. Neither individual-field atomicity nor whole-record atomicity is established.

### Required publication history

| Publication | Association supplying leading quantity | Destination |
|---:|---|---|
| 1 | F-00000000000c | Retained destination |
| 2 | Publication invocation of F-00000000000a | Retained destination |
| 3 | F-00000000000d | Retained destination |
| 4 | F-00000000000e | Retained destination |
| 5 | F-00000000000f | Companion storage at byte offset 60 |
| 6 | F-000000000010 | Retained destination |

Each interaction associated with publications two through six occurs after the preceding record’s four writes and before the next record’s first write. These intervening interactions have no established explicit arguments.

F-00000000000a is one target with two roles. Its later invocation is not established to receive the earlier acquisition arguments.

With a stable companion reference, publication five occupies companion bytes 60 through 75. Its reference field still contains the retained destination reference; it is not necessarily a reference to the companion record itself. The companion reference applicable to this publication is not guaranteed to match one used during acquisition.

Subsequent overwriting does not remove earlier publications from the required observable history.

### Terminal behavior

After publication six, F-000000000011 receives the numerical operands 198 and 141,615,330 in an exceptional interaction.

The representation identifies that interaction as nonreturning but also depicts a subsequent invocation whose target is selected through its result. F-000000000013 denotes this possible continuation. Neither mandatory execution nor impossibility of that continuation is established.

If F-000000000013 is reached and returns, an invalid-instruction boundary follows. No depicted path establishes a successful normal return, cleanup, or reversal of completed effects.

## Side effects

An admitted execution completing all publications has the following direct access counts:

| Access class | Count |
|---|---:|
| One-byte replacement write | 1 |
| Four-byte primary reads | 11 |
| Four-byte companion reads | 9 |
| Four-byte writes at the retained destination | 20 |
| Four-byte companion publication writes | 4 |

These count represented operation-level accesses, not distinct addresses, hardware transactions, or dependency-internal accesses.

For stable references:

- Primary reads use offsets 0, 4, 8, 20, 24, 28, 40, 44, 56, 64, and 68, requiring readable storage through byte 71.
- Companion acquisition reads use offsets 0, 4, 8, 20, 24, 28, 40, 44, and 48, requiring readable storage through byte 51.
- Companion publication requires writable storage through byte 75.
- Each retained-destination publication requires 16 writable bytes.

No non-aliasing requirement is established. Replacement storage, acquisition fields, retained-destination storage, and companion publication storage may overlap. Consequently, the replacement may alter later inputs, publications may overwrite already-consumed fields, and the two publication regions may overwrite one another. Partial-field overlap can make final byte interpretation dependent on byte order.

Completed effects have no established reversal mechanism. An interruption during a publication does not require restoration of earlier completed writes. The effect of an interrupted access itself is **documented_only**.

No direct allocation, deallocation, reference counting, synchronization, or ownership transfer is established. Publishing a reference does not establish an extension of the referenced storage’s lifetime.

Dependency implementations are unavailable. They must not be assumed to preserve globals, retain references, own storage, release storage, or return normally.

## Dependencies

Identifiers below are opaque and scoped to this contract.

| Identifier | Established role |
|---|---|
| F-000000000001, F-000000000002 | Alternative selector-dependent interactions at checkpoint 1 |
| F-000000000003 | Nonzero-path interaction at checkpoint 2 |
| F-000000000004, F-000000000005 | Zero-path and nonzero-path interactions at checkpoint 3 |
| F-000000000006 | Nonzero-path interaction at checkpoint 4 |
| F-000000000007–F-000000000009 | Acquisition consumers |
| F-00000000000a | Shared acquisition and publication target |
| F-00000000000b | Final acquisition consumer; supplies the 64-bit quantity |
| F-00000000000c | Consumes converted quantities and primary fields; associated with publication one |
| F-00000000000d–F-000000000010 | Interactions associated with publications three through six |
| F-000000000011 | Exceptional terminal interaction |
| F-000000000012 | Zero-path interaction at checkpoint 4 |
| F-000000000013 | Possible result-selected continuation after the exceptional interaction |

Distinct identifiers do not guarantee universally distinct executable targets. In particular, selector-dependent targets are not established to exclude other targets.

General calling conventions, implicit inputs, preservation rules, ownership effects, and termination behavior remain **documented_only**.

## Acceptance examples

These examples are conditional static consequences, not runtime test results.

### Admission and arithmetic

- A zero first observation selects F-000000000001. If it permits continuation, rejection follows without this operation’s direct replacement or publication writes.
- Nonzero first and second observations followed by a zero third observation require F-000000000004 before rejection.
- A zero fourth observation requires F-000000000012. Zero observations at checkpoints two and five require no additional rejection-specific interaction.
- Selector 2,147,479,600 is within the established addition range for F-000000000002 but outside that for F-000000000001. Acceptance does not require a wrapped target for the latter case.
- Replacement precursor 200 gives numerical address 211, subject to access validity.
- Replacement precursor 2,147,483,636 permits the addition. The next greater precursor does not have an established signed-range result.
- Precursor −11 gives numerical address zero, without establishing validity or a special rejection rule.

### Freshness and retention

- A replacement overlapping primary bytes 56 through 59 may affect the first payload eventually retained. Byte order and intervening effects determine the resulting field value.
- If F-000000000008 changes the applicable primary field at offset 24, that changed field supplies argument three to F-000000000009.
- A destination-field change during F-00000000000c may affect subsequent retention. A change confined to the source field after retention does not redirect publications.
- A dependency-induced change to the applicable companion reference can make later companion accesses use a different region; preservation must not be assumed.

### Conversion boundaries

- For a 64-bit quantity numerically equal to 23, F-00000000000c receives 23 and zero as its first two arguments.
- For a quantity numerically equal to 12,884,901,895, the upper numerical half is 3, so argument two is 3. Argument one remains subject to the unresolved out-of-range conversion; requiring 7 would impose an unsupported narrowing rule.

### Publication visibility and overlap

Assume valid accesses, stable references, and no additional relevant memory changes:

- With disjoint publication regions, the retained destination contains publication six immediately before the exceptional interaction. Companion bytes 60 through 75 contain publication five.
- The dependency between publications one and two executes after all four writes of publication one. Skipping intermediate records violates the required history.
- If the retained destination begins at companion offset 60, publication six completely overwrites publication five.
- If the retained destination begins at companion offset 64, the final field-aligned contents are:

| Companion byte offset | Final content |
|---:|---|
| 60 | Publication-five leading quantity |
| 64 | Publication-six leading quantity |
| 68 | Retained first payload |
| 72 | Retained destination reference |
| 76 | Retained second payload |

The last example requires writable storage through byte 79. Its field-aligned result does not depend on byte order.

If a publication’s first write completes and its second access prevents further progress, restoring the first write is not required. The interrupted second access’s own effects remain unresolved.

## Uncertainties

The following are **documented_only**, not implementation obligations to invent missing behavior:

- Condition origin, modification rules, and feasible checkpoint histories.
- Implicit dependency access to the otherwise directly unused inputs.
- Delivery convention, signedness, and meaning of dependency-associated quantities.
- Dependency preservation, ownership, resource-management, and termination behavior.
- Signed-overflow handling and out-of-range signed conversion behavior.
- Original interpretation of the 64-bit quantity.
- Address encoding, pointer conversion, byte order, alignment, and address-space constraints.
- Relative read order within each acquisition argument group.
- Atomicity, concurrent-access legality, and interrupted-access effects.
- Runtime manifestation of invalid-instruction boundaries.
- Reachability and validity of the result-selected continuation.
- Application meaning of the records.
- Fidelity of the represented function boundary and exceptional control flow.

These uncertainties establish no retry, fallback, validation, synchronization, cleanup, rollback, or successful-return guarantee.

## Evidence

**Established static evidence:** The supplied representation supports the explicit input widths, five admission decisions, branch-specific interactions, replacement-before-acquisition ordering, acquisition field relationships, 64-bit conversion operations, ordered retention, six-record publication history, and exceptional endpoint.

**Derived consequences:** Signed 32-bit bounds determine the addition limits. Four-byte field widths determine access extents and counts. Target relationships establish the shared acquisition/publication dependency. Ordering and layout establish the conditional freshness, retention, and overlap examples.

**Inferred intent:** Repeated records appear to associate changing dependency-related quantities with two retained payloads and a destination reference. This interpretation does not establish application semantics, ownership, or synchronization.

**Evidence limits:** Prior prose, feedback, and embedded comments are not independent proof. Dependency implementations and an external execution contract are absent. The conflicting terminal-flow indications remain unresolved rather than being converted into a definite continuation rule. No runtime tests have been performed.
