<!-- Trusted provenance: F-c327d5d3ac2f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c327d5d3ac2f

## Purpose

F-c327d5d3ac2f performs an ordered series of narrow memory updates and external interactions. Its application-level purpose is not established.

The directly represented data-memory activity consists of seventeen one-byte writes, one two-byte read, and one four-byte write. External interactions separate most memory operations and continue after the last direct write. Consequently, equivalent final storage alone is insufficient to establish equivalent behavior: intermediate writes, observations, argument values, and dependency effects also matter.

The represented endpoint is exceptional and marked non-returning. An indirect invocation appears after that exceptional operation, using its represented result as the executable target. The available evidence does not establish that this continuation is reachable or that the function can complete normally.

This contract describes static relationships. Every later operation remains conditional on earlier operations permitting execution to continue.

## Interface

### Explicit inputs

There are three explicit signed 32-bit integer inputs, each ranging from −2,147,483,648 to 2,147,483,647.

| Input | Established behavioral role |
|---|---|
| First input | Supplies the basis for both arguments of the first external call, with a scale factor of 4,096. |
| Second input | Supplies the basis for the first byte destination, with a scale factor of two. |
| Third input | Supplies the seventh byte destination with a scale factor of 2,048, supplies a subsequent equal argument pair with a scale factor of 4,194,304, and appears unchanged as the second argument of four later calls. |

These numerical uses do not establish that any input is an application pointer, index, length, identifier, or handle.

### Additional required execution state

The supplied representation uses additional state without establishing how callers provide it:

- An eight-bit payload shared by all seventeen byte writes.
- A signed 32-bit basis used to select the first ordinary executable target.
- A signed 32-bit basis used to locate the two-byte read.
- A destination reference for the four-byte write.
- Six separate opaque 32-bit quantities passed to external dependencies.

Their delivery, initialization, semantic meaning, and validity are **documented_only**. They are not established public parameters.

In particular, the six opaque quantities have no explicit producers in the supplied representation. Their proximity to earlier calls does not prove that those calls produced them.

External results supply further values. Results used for signed numerical calculations are represented as signed 32-bit integers. Two other results are opaque 32-bit quantities: one is forwarded unchanged to another call, and one is duplicated into an argument pair.

### Outputs and environmental preconditions

No explicit returned data value or success status is established.

Each reached byte write requires one writable byte at its selected destination. The observation requires two readable bytes beginning at its selected address. The wider write requires four writable bytes beginning at its destination reference. Each external invocation requires an executable target and a compatible interface.

No local address validation, interface validation, or recoverable-error protocol establishes those conditions. A location valid earlier need not remain valid later: intervening dependencies may alter storage, mappings, resources, or lifetimes.

The integer widths do not establish pointer width, address conversion rules, byte order, alignment requirements, or accessible address ranges. Those properties remain **documented_only**.

## Behavior

### Memory-update structure

The byte writes have a fixed relative order. Their ordinal positions below identify observable milestones, not array elements or record fields.

The first byte write precedes every external call. Its destination is twice the second input. The first subsequent external call receives the first input scaled by 4,096 in both explicit argument positions. There is no established relationship requiring those arguments to identify the preceding destination.

The seventh byte write has a different input-derived relationship: its destination is the third input scaled by 2,048. The following external call receives that same input scaled by 4,194,304 in both positions.

Every other byte destination is derived from a separately obtained signed 32-bit dependency result, scaled by eight. The result occurrence associated with one destination is not interchangeable with another occurrence, even if their numerical values happen to agree.

Ten of those result-derived writes—positions 2, 3, 4, 6, 8, 9, 10, 14, 15, and 17—are followed by an external call whose two explicit arguments are both the associated result scaled by 16,384.

The remaining result-derived writes have distinct neighboring behavior:

- Positions 5, 12, and 16 are followed by a call with no represented explicit arguments.
- Position 11 immediately precedes the two-byte observation.
- Position 13 immediately precedes the four-byte update.

All seventeen byte writes use the same represented eight-bit payload. No direct read of a destination’s previous byte is represented. There is dependency activity between every pair of consecutive byte writes.

These relationships impose ordering, not atomicity. A fault, exception, or non-returning dependency can prevent any later milestone from being reached.

### External-interaction obligations

Calls remain observable obligations even when their results are unused. An equivalent implementation cannot omit, merge, or reorder them merely because the local representation does not consume their ordinary return values.

The interaction stream includes calls with no explicit arguments, calls receiving a dependency result scaled by four, equal-argument calls associated with the memory updates, and calls consuming opaque quantities. Absence of explicit arguments does not establish absence of implicit inputs or effects.

The six opaque 32-bit quantities are each consumed once as the sole explicit argument of a call. In consumption order, those calls lie in these intervals:

| Consumption | Interval bounded by byte writes |
|---|---|
| First | After position 1 and before position 2 |
| Second | After position 2 and before position 3 |
| Third | After position 3 and before position 4 |
| Fourth | After position 6 and before position 7 |
| Fifth | After position 14 and before position 15 |
| Sixth | After position 15 and before position 16 |

This placement establishes consumption order only. It does not establish production, ownership, or a relationship to adjacent call results.

Four calls preserve an explicit relationship to the third input: one occurs in each interval 7–8, 8–9, 9–10, and 10–11. Each receives four times a separately obtained signed result as its first argument and the unchanged third input as its second argument.

Two additional forwarding relationships are established:

- Between byte positions 4 and 5, an opaque 32-bit call result becomes the sole explicit argument of a subsequent call without a represented numerical transformation.
- After the four-byte update and before byte position 14, another opaque 32-bit result supplies both positions of a subsequent argument pair.

Forwarding and duplication describe value flow. They do not establish object copying, allocation, retention, consumption, release, or ownership transfer.

### Two-byte observation

Immediately after byte position 11, the function reads two bytes beginning at the observation basis plus 32. No dependency call intervenes between that byte write and the read.

The next external call receives the observed value as its sole represented explicit argument. Further dependency activity occurs before byte position 12.

This observation concerns storage at that execution point. It is not an entry-time snapshot. Earlier direct writes or dependency effects may have changed the observed storage.

The access width is sixteen bits. The representation does not establish the value’s numerical signedness or how the receiving interface adapts it. Neither sign extension nor zero extension is specified. Byte order is also **documented_only**.

If byte position 11 overlaps the observed region, the byte write locally precedes the read. That ordering does not by itself determine the argument’s numerical value. Byte order, access semantics, and concurrent interference remain relevant.

The displacement of 32 does not establish an enclosing object, a field type, an allocation boundary, or an object size.

### Four-byte update and later reuse

The signed result associated with byte position 13 determines both that byte destination and a separate four-byte value. The destination uses eightfold scaling; the wider value uses multiplication by four.

The wider value is calculated before either write. The byte write occurs first, followed by the four-byte write through the additional destination reference. No dependency call intervenes between these writes.

The next external call receives the calculated four-byte value in all three explicit argument positions.

The two destinations are not established as disjoint. If they overlap, the wider write occurs second and may replace the earlier byte’s effect within the overlapping region. Exact final bytes require byte-order and access-semantics information.

The calculated value is also retained for a call between byte positions 14 and 15. That call receives a newly obtained signed result scaled by four as its first argument and the retained value as its second argument. The fifth opaque-quantity consumer occurs earlier in that interval.

The retained argument is not obtained by rereading the four-byte destination. A change confined to the destination’s contents therefore does not itself substitute a new second argument. This is a represented value-flow relationship, not a guarantee that unknown dependencies preserve arbitrary execution state.

### Integer arithmetic

Numerical address selection and successful memory access are separate conditions. A calculation can be representable while selecting inaccessible storage or unusable executable code.

Several scales are represented by signed 32-bit shifts. A conservative interpretation as nonnegative mathematical multiplication is available within these domains:

| Scale factor | Operand domain |
|---:|---:|
| 4 | 0–536,870,911 |
| 8 | 0–268,435,455 |
| 2,048 | 0–1,048,575 |
| 4,096 | 0–524,287 |
| 16,384 | 0–131,071 |
| 4,194,304 | 0–511 |

When one value participates in several calculations, all applicable constraints matter. Thus:

- The first input’s initial argument pair has a conservative domain of 0–524,287.
- The third input’s coupled destination and equal-argument uses have a conservative domain of 0–511.
- A result used for both an eightfold byte destination and a 16,384-fold argument pair has a conservative domain of 0–131,071.
- The result shared by byte position 13 and the four-byte update has a conservative domain of 0–268,435,455 because its destination calculation is the narrower constraint.

Other arithmetic relationships have these signed 32-bit mathematical representability limits:

| Relationship | Operand domain |
|---|---:|
| Twice the second input | −1,073,741,824–1,073,741,823 |
| Four times the wider-update basis | −536,870,912–536,870,911 |
| Observation basis plus 32 | −2,147,483,648–2,147,483,615 |
| Initial executable-target basis plus 4,094 | −2,147,483,648–2,147,479,553 |

The multiplication-by-four domain does not relax the coupled destination constraint at byte position 13.

These domains are derived interpretation limits, not implemented guards or complete accepted-input ranges. Negative operands are outside the conservative signed-shift interpretation even where their mathematical products would fit.

No general wraparound, saturation, rejection, or overflow-recovery policy is established. Behavior outside these interpretation limits remains **documented_only**.

### Interruption and endpoint

No local retry, rollback, cleanup, or recoverable-error branch is represented. If execution stops, completed effects are not established to be reversed.

Dependency activity continues after byte position 17. The eventual exceptional operation is represented with selector zero and a fixed execution-location descriptor, and is marked non-returning.

The representation nevertheless includes an invocation through that operation’s result, with no explicit arguments and no intervening target-validation branch. This does not establish a reachable continuation or a valid executable result. If continuation were possible, that invocation would require a usable target.

The descriptor is not established as an application error code. Exception handling, partial effects of failing accesses, and the feasibility of reaching or passing this endpoint remain **documented_only**.

## Side effects

If all direct writes are reached, their access widths total 21 bytes: seventeen one-byte accesses and one four-byte access. This is not a count of distinct bytes, distinct destinations, or bytes retaining their assigned values.

The two-byte observation is the only directly represented data-memory read. This statement excludes instruction fetching, dependency-internal memory activity, and environmental accesses.

No structured storage layout or disjointness requirement is established. Destinations may repeat or overlap one another, the observation region, or storage affected by dependencies. Rewriting the same payload is not necessarily redundant because intervening activity may observe, replace, or invalidate it.

Local sequencing establishes no transaction boundary, memory barrier, synchronization discipline, inter-thread visibility guarantee, concurrency safety, or reentrancy guarantee. Access widths alone do not establish atomicity.

No direct allocation, deallocation, lifetime extension, or ownership transfer is represented. Unknown dependencies must not be assumed to preserve globals, storage, mappings, resources, or lifetimes. Passing an opaque quantity supplies no ownership or release contract.

## Dependencies

No opaque function ID is supplied for any outbound dependency. Their identities are therefore **documented_only**; the behavioral roles in this document do not invent function identities or assert that separate call occurrences share a callee.

The first ordinary executable target is selected using the additional target basis plus 4,094. Subsequent ordinary calls use fixed executable references with the same represented displacement. Where a reference is explicitly byte-addressed, the displacement is in bytes.

These static target relationships do not establish callable instruction boundaries, recursion, dispatch-table organization, or complete signatures.

Unavailable contracts include:

- Explicit and implicit inputs and calling conventions.
- Result meanings and argument-width adaptation.
- Memory, global-state, mapping, resource, and synchronization effects.
- Retention, ownership, consumption, and release obligations.
- Failure behavior and guarantees of returning control.

The exceptional endpoint also requires an unavailable environmental contract. It must not be treated as a successful lookup operation or a guaranteed provider of executable targets.

## Acceptance examples

These are conditional acceptance obligations, not observed executions. Each assumes usable addresses, compatible interfaces, applicable arithmetic interpretation, and continuation through preceding operations.

### Input relationships

- With second input 256 and payload 90, the first direct effect writes 90 at numerical address 512. No external call precedes it.
- With first input 3, the first external call receives 12,288 in both positions. Changing only the second input does not change that argument pair.
- With third input 3, byte position 7 selects numerical address 6,144, and the following equal-argument call receives 12,582,912 twice. The four later unchanged second-argument uses each supply 3.
- Second input zero selects numerical address zero without a represented local rejection. This does not imply that the write succeeds.

### Arithmetic boundaries

- A result of 5 in a coupled destination-and-pair role selects numerical address 40 and produces two arguments of 81,920.
- Result 131,071 lies within that role’s conservative domain. Result 131,072 would require mathematical paired arguments of 2,147,483,648; no wrapped replacement is specified.
- Third input 511 produces paired arguments of 2,143,289,344. Third input 512 exceeds the combined conservative domain even though its destination calculation alone remains representable.
- A negative result does not enter a conservative signed-shift domain merely because its mathematical product would fit.

### Observation and retention

- If byte position 11 overlaps the two-byte observation, the write precedes the read without an intervening dependency call. A numerical expected observation still requires byte-order and access-semantics information.
- If the result associated with byte position 13 is 7, its byte destination is numerical address 56, its four-byte value is 28, and the following three-argument call receives 28 in every position.
- If those write destinations overlap, the four-byte write occurs second. The payload byte need not survive.
- If subsequent activity changes only the four-byte destination’s contents, the later retained-value call still uses 28 as its second argument, rather than rereading replacement contents.

### Partial progress

- If the first dependency never returns, the initial byte write has already occurred; subsequent local effects need not occur.
- If execution stops immediately after the four-byte update, byte positions 14–17 have not occurred. No rollback is established.
- Matching final storage does not justify eliminating repeated accesses or intervening calls.
- Reaching byte position 17 establishes neither normal return nor successful continuation through the exceptional endpoint.

## Uncertainties

The following remain **documented_only**:

- Application purpose and application-level success criteria.
- Semantic input types and delivery of additional execution state.
- Producers and meanings of the six opaque quantities.
- Outbound dependency IDs and complete dependency contracts.
- Pointer width, address conversion, alignment, mappings, and byte order.
- Arithmetic behavior beyond the conservative interpretation limits.
- Signedness and interface adaptation of the two-byte observation.
- Dependency effects on shared state, resources, mappings, and lifetimes.
- Ownership, retention, release, and cleanup obligations.
- Fault delivery, partial-access effects, atomicity, synchronization, and reentrancy.
- Exceptional handling and feasibility of complete execution.

These gaps cannot be resolved by assuming benign callees, preserved globals, valid addresses, automatic release, or successful completion.

## Evidence

**Established static evidence:** The supplied representation supports three signed 32-bit explicit inputs; seventeen byte writes using one represented eight-bit payload; one two-byte read; one four-byte write; the numerical dependencies and local ordering described above; opaque-result forwarding and duplication; and later reuse of the calculated four-byte value.

**Derived constraints:** Arithmetic domains follow from signed 32-bit representability and a conservative nonnegative interpretation of signed shifts. The 21-byte write total sums access widths rather than unique storage. Neither deduction establishes runtime validation.

**Inferred intent:** No application-level intent is adopted. Terms such as “observation,” “destination,” and “retained value” identify behavioral roles without implying application objects or ownership.

**Evidence limits:** Prior prose, comments, and reviewer feedback are not independent behavioral authority. Dependency implementations and environmental contracts are unavailable. Static relationships do not prove executable targets, accessible storage, fault behavior, or successful completion.

No runtime execution or tests have been performed.
