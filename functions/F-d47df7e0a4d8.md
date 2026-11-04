<!-- Trusted provenance: F-d47df7e0a4d8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d47df7e0a4d8

## Purpose

F-d47df7e0a4d8 conditionally submits a signed adjustment target for a mutable object. Eligibility depends on two ordered state-and-flag conditions. For an eligible object, the function requests measurement information, selects either an object-held target or zero, and suppresses the adjustment when the corresponding endpoint condition already holds.

An adjustment request is followed by an object-selected dependency call. A flag in that call’s returned storage determines whether another object-selected call and F-6dd3097a8880 follow.

The function provides no explicit completion status and does not verify that an adjustment achieved its target.

**Established limitation:** target selection consumes a signed value for which the supplied representation shows no initialization. The measurement dependency’s output guarantees are also unavailable. The admitted path therefore has no established, unconditional target-selection outcome. The conditional rules below describe the visible decision structure; they do not make an uninitialized read valid or define its result.

**Inferred intent:** the endpoint comparisons suggest reconciliation of an extent against a measurement. Units, application meaning, and any relationship to a visual surface are unknown.

## Interface

### Input and result

The sole explicit input identifies mutable object storage. There is no explicit return value, error code, or success indication.

Required accessibility is path-dependent. A rejected invocation needs only the eligibility fields actually examined. An admitted invocation additionally needs a usable association and measurement dependency. Later paths require associated endpoint storage, possibly the object-held target, and—if adjustment is not suppressed—callable adjustment and object-selected dependencies.

No validation of addresses, object type, field bounds, dispatch entries, or returned storage is visible. Invalid inputs have no established harmless-return behavior.

### Represented storage

The supplied representation uses 32-bit integer fields and four-byte address-bearing slots. These widths and the offsets below describe the available representation, not a complete portable calling convention or object model.

| Byte offset in the input object | Role |
|---|---|
| 0x00 | Address of a dispatch table |
| 0x5C | Secondary eligibility state |
| 0x60 | Secondary eligibility flags |
| 0x74 | Primary eligibility state |
| 0x78 | Primary eligibility flags |
| 0xB4 | Address of associated storage |
| 0x2E4 | Signed object-held target |

Each eligibility state qualifies only when zero. Its associated flags qualify when bit 0x20 is set. Other flag bits do not affect eligibility.

The associated storage supplies four signed 32-bit endpoints:

| Byte offset in associated storage | Role |
|---|---|
| 0x80 | Selection start |
| 0x84 | Adjustment start |
| 0x88 | Selection end |
| 0x8C | Adjustment end |

These role names distinguish the comparisons that use the fields. They imply neither ordered endpoints nor nonnegative values.

The object-selected dependency is obtained from a dispatch-table entry at byte offset 0x5C. That table entry is separate from the eligibility state at the same offset in the input object.

The first object-selected call returns an address. A 32-bit field at offset 0x48 in that returned storage is examined using mask 0x40.

No non-aliasing guarantee is established among these storage regions.

### Measurement storage and arithmetic

F-756b664b4a2c receives the association and the address of invocation-local storage represented as two 32-bit elements, totaling eight bytes. The enclosing function subsequently consumes the first element as a signed value; it does not directly consume the second.

Selection also consumes a separate signed 32-bit value with no visible initialization. The evidence does not establish that this value is part of the eight-byte measurement area, that the measurement dependency initializes it, or that the dependency may write beyond that area.

For describing the conditional decision:

- The **selection span** is selection end minus selection start.
- The **measurement threshold** is the separate comparison value minus the first measurement element.
- The **adjustment span** is adjustment end minus adjustment start.

These are signed 32-bit differences in the supplied representation. The ordinary arithmetic interpretation requires each evaluated difference to lie within −2,147,483,648 through 2,147,483,647. Wrapping, saturation, wider intermediate arithmetic, and overflow recovery are not established.

## Behavior

### Eligibility

Primary eligibility takes precedence. A zero primary state permits examination of the primary flags; bit 0x20 then admits the invocation. A nonzero primary state skips that flag examination.

Secondary eligibility is considered only when the primary condition does not admit the invocation. It uses the same zero-state and bit-0x20 requirements. A nonzero secondary state skips its flag examination.

Consequently, primary admission avoids all secondary eligibility reads. If neither condition qualifies, the function returns without accessing the association, target, or dispatch table and without calling a dependency.

Eligibility is not reconsidered after admission. Later dependency effects on eligibility fields do not trigger another eligibility decision in this invocation.

### Measurement and association identity

An admitted invocation calls F-756b664b4a2c once, passing the association obtained from the object and the local measurement-area address. Its explicit result is unused.

After normal return, the function reads the association from the original input object again. The later association supplies all endpoint reads for target selection and suppression.

This distinction is observable: measurement can receive one associated address while the decision uses another. The reload exposes a replacement made during measurement, provided the original object and replacement storage remain usable. It does not establish that replacement is intended, safe, or part of the dependency’s contract.

No dependency call intervenes between the post-measurement association read and the endpoint-based decision. This establishes neither synchronization nor a coherent snapshot in the presence of concurrent mutation.

### Target selection and suppression

The following obligations are conditional on defined comparison inputs and supported arithmetic.

When the selection span is strictly less than the measurement threshold, the function reads the signed object-held target. Equality between that target and the adjustment span suppresses the request. Otherwise, the object-held target is submitted to the adjustment dependency.

When the selection span is equal to or greater than the threshold, the selected target is zero. The object-held target is not read. Equal adjustment endpoints suppress the request; unequal adjustment endpoints require a request for zero.

Selection equality therefore belongs to the zero-target case. Negative object-held targets are neither rejected nor clamped.

The suppression conditions have different arithmetic requirements. The object-held-target path evaluates an adjustment difference. The zero-target path compares the adjustment endpoints directly, so their difference need not be representable.

Suppression ends the invocation without adjustment or object-selected calls. Subject to normal completion, measurement is the only dependency interaction on that path.

### Adjustment and continuation

An unsuppressed target is passed to F-d25031d00f00 together with the original input-object address. Its explicit result is unused.

The request alone does not establish successful adjustment. There is no remeasurement, endpoint verification, retry, or success-dependent continuation.

After normal adjustment return, the function selects a dependency using the object’s then-current dispatch table and entry offset 0x5C. That dependency receives the original input-object address.

The returned storage controls continuation:

- If its flag field lacks bit 0x40, the invocation ends.
- If bit 0x40 is present, the object-selected call occurs again, followed by F-6dd3097a8880 after normal return.

The second selection rereads the object’s dispatch table. It may therefore reach a different implementation from the first call. The second call’s result is unused: there is no second flag examination, and no explicit result is forwarded to F-6dd3097a8880.

F-6dd3097a8880 receives no explicit arguments. Shared state or calling-convention effects remain possible and unspecified.

## Side effects

### Observable ordering

The eligibility decision precedes every dependency interaction. Within eligibility, state checks control whether their flag fields are examined, and primary qualification prevents secondary examination.

Measurement separates two association reads: its argument uses the earlier association, while endpoint decisions use the later one. Target selection precedes any access to the object-held target. When that target is selected, its read precedes the adjustment-span suppression comparison.

Adjustment precedes the first dispatch lookup. The first returned flag examination precedes any second dispatch lookup, and the final dependency follows normal return from the second object-selected call.

These are control-flow ordering constraints. The evidence does not establish a relative order for operand reads within an individual subtraction or equality comparison, nor does it establish hardware memory-ordering guarantees.

The association reload exposes measurement-side changes. The dispatch lookups expose changes made by adjustment and by the first object-selected call. No dependency is assumed to preserve object fields or globals.

### Writes and state changes

No direct persistent write by F-d47df7e0a4d8 to the input object, associated storage, returned storage, or globals is visible. Selecting a target does not itself update an endpoint or object field.

The overall invocation need not be read-only. Dependencies may write through supplied addresses, modify reachable storage, or affect globals. Their resulting state is unknown.

No rollback, recovery, or cleanup sequence is visible for effects preceding a later failure.

### Ownership and lifetime

No explicit allocation, release, retention operation, or ownership transfer is established.

The measurement area has invocation-local lifetime. Passing its address does not establish a retention right, a guarantee against retention, or permission to write beyond its represented extent.

Normal continuation requires the original object to remain usable for each subsequent read and call argument. The post-measurement association must support all reached endpoint accesses. The first object-selected result must support its flag-field read. Selected dispatch entries must remain callable when invoked.

The function does not directly use the second object-selected result. That absence establishes no ownership rule and does not imply that the result or associated resources are released elsewhere.

### Errors and abnormal completion

There is no explicit error channel, fallback target, or recovery branch.

Later actions depend on earlier dependencies returning normally. Faults, exceptions, nonlocal exits, invalid storage, and invalid call targets have no specified recovery behavior.

Uninitialized consumption and unsupported signed arithmetic are **documented_only**. They must not be converted into a deterministic target, a no-op, or an assumed machine-level wraparound result.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| F-756b664b4a2c | Called once after admission with the pre-call association and local measurement-area address; explicit result unused | Output initialization, output schema, write extent, mutation, retention, ownership, implicit inputs, and failure |
| F-d25031d00f00 | Called for an unsuppressed request with the original object and selected signed target; explicit result unused | Meaning of adjustment, success, resulting state, allocation, ownership, and failure |
| Object-selected dependency; opaque identity unavailable | Selected through dispatch-table offset 0x5C after adjustment, then selected afresh if continuation is required; receives the original object | Concrete identity, identity stability, result validity, mutation, ownership, and failure |
| F-6dd3097a8880 | Called without explicit arguments after normal return from the conditional second object-selected call | Purpose, implicit inputs, shared-state effects, ownership, and failure |

The object-selected role does not establish one fixed dependency identity. No concrete opaque function ID for it is available.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Except where an uncertainty is illustrated, assume valid reached storage, normally returning dependencies, defined consumed values, and representable required arithmetic. In particular, hypothetical comparison values do not resolve the missing initialization in the supplied representation.

### Admission

- **Primary state zero; primary flags 0xA0:** admission occurs through the primary condition. Secondary eligibility fields are not accessed.
- **Primary state nonzero; secondary state zero; secondary flags 0x20:** secondary eligibility admits the invocation. Primary flags are not accessed.
- **Both states nonzero:** rejection occurs without either flag read or any dependency call.
- **Both states zero; both flag fields 0x40:** rejection occurs. The continuation bit does not satisfy eligibility.

### Selection and suppression

- **Selection span 40; threshold 60; object-held target 12:** adjustment span 12 suppresses the request. Adjustment span 7 requires a request for 12.
- **Selection span −10; threshold −5; object-held target −3; adjustment span 2:** the requested target is −3. No nonnegative clamp applies.
- **Selection span and threshold both 60:** the target is zero, without reading the object-held target. Equal adjustment endpoints suppress the request; unequal endpoints require the zero-target request.
- **Zero-target case; adjustment start −2,147,483,648 and adjustment end 2,147,483,647:** endpoint inequality requires a zero-target request. No adjustment subtraction is needed.
- **Object-held-target case with those same adjustment endpoints:** the required difference is outside the signed 32-bit range. A supported request outcome is not established.
- **Selection start −2,147,483,648 and selection end 2,147,483,647:** the selection difference is outside the supported range. These values do not establish a target.

### Mutation and continuation

- **Measurement replaces the association:** endpoint decisions use the replacement, conditional on valid continued access.
- **Adjustment replaces the dispatch table:** the first object-selected call uses the post-adjustment table.
- **The first object-selected call changes the dispatch table and returns flags 0xC0:** bit 0x40 requires another call using a fresh lookup. F-6dd3097a8880 follows its normal return, without examination of its result.
- **The first returned flags are 0x80:** neither the second object-selected call nor F-6dd3097a8880 occurs.

None of these request examples establishes that F-d25031d00f00 achieves the requested state.

### Unresolved execution

- **All object fields are known, but the separate comparison value lacks defined initialization:** those fields do not establish an admitted-path target.
- **Measurement leaves the consumed output element without a defined value:** selection remains unsupported.
- **Measurement exits abnormally:** association reloading and later interactions are not guaranteed.

## Uncertainties

The following remain **documented_only**:

- **Comparison initialization:** the separate signed comparison value has no visible initialization. It must not be modeled as an arbitrary but valid integer merely to complete the contract.
- **Measurement output:** initialization of the consumed element and the complete output agreement are unavailable.
- **Storage relationship:** no larger measurement structure, adjacency guarantee, or authorization to write beyond eight bytes is established.
- **Arithmetic:** signed-overflow semantics and recovery are unknown.
- **ABI:** represented widths and offsets do not settle alignment, address interpretation, or all calling-convention requirements.
- **Dependency effects:** preservation of globals, mutation, allocation, release, retention, ownership, and abnormal completion are unknown.
- **Dynamic identity:** concrete object-selected implementations and identity stability are unresolved.
- **Concurrency:** no synchronization or snapshot guarantee is visible.
- **Invalid inputs:** no safe failure behavior is established.
- **Application meaning:** extent reconciliation is an interpretation, not a guaranteed application-level operation.

## Evidence

**Established static evidence:** the supplied representation supports ordered eligibility with conditional flag reads; the listed field widths and offsets; measurement followed by association reloading; strict signed target selection; conditional access to the object-held target; distinct suppression comparisons; adjustment arguments; post-adjustment dispatch selection; continuation controlled by bit 0x40; a fresh second dispatch selection; and the final conditional dependency call.

It also shows consumption of a signed comparison value without visible initialization. No explicit persistent write, input validation, adjustment-success verification, retry, or recovery mechanism is present.

**Inferred intent:** endpoint differences and a stored signed target suggest extent reconciliation. Stronger semantic claims are unsupported.

**Unavailable evidence:** dependency implementations, a complete measurement-output agreement, ownership contracts, complete ABI requirements, and a defined overflow policy.

No runtime tests have been performed.
