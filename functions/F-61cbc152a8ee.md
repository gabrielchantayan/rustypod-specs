<!-- Trusted provenance: F-61cbc152a8ee; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-61cbc152a8ee

## Purpose

F-61cbc152a8ee selects a position and an associated quantity from a bounded examination of classified positions. A zero classification contributes to a candidate; a nonzero classification ordinarily separates candidates.

Selection depends on a policy: first sufficient prefix, exact completed candidate with an excess preference, largest sufficient completed candidate, or largest completed candidate without regard to the request. A largest-completed fallback also applies when the policy does not supply a result with a nonzero starting position.

The requested quantity is not a universal success requirement. Normal completion can return a smaller quantity or zero. Dependency failure returns the largest unsigned 32-bit value, which can also be a successful quantity.

Position zero has two roles: an examinable position and an absence marker. This affects both candidate accounting and final selection.

**Inferred intent:** quantity-based selection among classified regions. No allocation, reservation, availability, or ownership-transfer meaning is established.

## Interface

| Element | Representation | Meaning |
|---|---|---|
| Context | Signed 32-bit value | Identifies the boundary-bearing context and is passed unchanged to F-c9f1dc723c52 |
| Initial position | Unsigned 32-bit value | First position considered for examination |
| Position destination | Address of writable four-byte unsigned storage | Receives the selected position on normal completion |
| Requested quantity | Unsigned 32-bit value | Policy-dependent threshold or exact target |
| Policy | Signed 32-bit value | Values 1, 2, and 3 have distinct behavior; every other value uses largest-completed selection |
| Return | Unsigned 32-bit value | Selected quantity, or 4,294,967,295 on dependency failure |

The boundary is an unsigned four-byte field at byte displacement **124** from the address represented by the context. It is read once before any classification call, including when the initial position already exceeds it.

The context representation does not establish a general pointer width or a complete object layout. Valid address interpretation, address arithmetic, alignment, byte order, and memory accessibility are external preconditions. In particular, signed address arithmetic must not be assigned unsigned wraparound semantics merely because positions use unsigned arithmetic.

The incoming value in the position destination is not used. Normal completion requires that the function’s four-byte publication be valid. Failure bypasses that publication, but does not establish that an invalid destination is permissible under the complete calling contract.

There is no separate error indicator. A return of 4,294,967,295 is ambiguous without additional caller knowledge.

## Behavior

### Examination extent

The captured boundary remains the examination limit for the invocation. Changes to its backing memory during dependency calls do not change that limit.

The initial-position comparison and subsequent position comparisons are unsigned and inclusive. If the initial position exceeds the boundary, there are no classification calls; normal completion publishes position zero and returns zero.

Otherwise, examination proceeds in ascending unsigned 32-bit position order. Each visited position is supplied to F-c9f1dc723c52. A zero dependency status terminates the invocation immediately with failure. Any nonzero signed status, including a negative status, permits classification processing.

For a boundary below 4,294,967,295, examination can finish by passing the boundary. Policy selection can stop it earlier.

For a boundary equal to 4,294,967,295, advancement wraps to zero and never produces a position greater than the boundary. Consequently:

- Policies 1 and 2 can terminate normally through their respective stopping conditions.
- Policy 3 and all other policy values have no normal termination path.
- Dependency failure can still terminate any policy.
- A dependency call that does not return prevents further progress.

The maximal boundary is also a candidate-completion point. Continuing after wraparound does not, by itself, merge a pending candidate across that boundary.

### Candidate meaning and the position-zero exception

With defined classifications that remain consistent during each visit, a candidate consists of retained contributing positions. A separator contributes no quantity.

A positive pending candidate completes upon a separator or upon examination of the boundary. Thus a contributing boundary position may complete a candidate. Policy 1 can stop before completion.

No positive candidate is created solely by separators. Requesting zero does not create an empty candidate.

For a candidate with a nonzero start, each further contributing position increases its quantity by one. Position zero behaves differently because zero also represents an absent start:

- A contribution at zero produces a pending quantity of one with start zero.
- A subsequent contribution while the start is still zero establishes the current position as the start and sets the quantity to one.
- Later contributions accumulate normally once the start is nonzero.

A singleton contribution at zero can therefore complete with quantity one. Consecutive contributions at positions zero through three, completed at three, instead produce start one and quantity three.

Once a candidate completes, it is no longer pending. At a maximal boundary, examination may subsequently continue from zero with a fresh candidate.

### Largest-completed fallback

The fallback initially represents position zero and quantity zero.

Every completed positive candidate is considered for fallback, regardless of policy or requested quantity. It replaces the fallback only if its quantity is strictly greater. Ties retain the earlier completed candidate, including across wraparound.

A zero-start candidate is eligible for fallback. Its positive quantity can replace the initial zero-quantity result.

Fallback need not meet the requested quantity. A policy-1 prefix that causes an immediate stop is not completed or considered for fallback on that visit.

### Policy 1: first sufficient prefix

Policy 1 stops at the first contribution for which the retained pending quantity is at least the requested quantity. It does not wait for a separator or boundary completion.

This stop has priority over completion at the same position. Even a boundary contribution that satisfies the request stops before receiving fallback consideration.

A request of zero still requires a contribution. Separators alone do not trigger selection.

If the qualifying prefix starts at a nonzero position, that prefix supplies the final result. If it starts at zero, examination still stops, but the final result comes from fallback. A first contribution at zero can consequently stop examination and return quantity zero.

If finite examination ends without a qualifying prefix, fallback supplies the result.

### Policy 2: exact completion, otherwise excess preference

Policy 2 stops when a completed candidate has quantity exactly equal to the request. That candidate receives fallback consideration before the stop.

In the absence of an exact completion, completed quantities strictly greater than the request are eligible for an excess preference:

- The first eligible candidate establishes the preference.
- A preference with a nonzero start is replaced only by a strictly smaller eligible quantity.
- Equal quantities do not replace an existing nonzero-start preference.
- A preference with start zero is treated as absent for replacement purposes. A later eligible candidate may replace it even when larger.

Accordingly, this policy is not an unconditional smallest-excess selection.

An exact zero-start candidate still stops examination. Its zero start prevents direct publication as the policy result, so fallback supplies the final pair. An earlier, larger completed candidate can therefore be returned instead of the exact candidate.

Completed quantities are positive. Request zero cannot produce an exact completion; every completed candidate is instead eligible as an excess candidate.

### Policy 3: largest sufficient completion

Policy 3 considers completed candidates whose quantities are at least the request.

A preference with a nonzero start is replaced only by a strictly larger eligible quantity. Equal quantities retain that preference.

As with policy 2, a preference starting at zero is treated as absent when deciding whether a later eligible candidate may replace it. A later smaller or equal candidate can therefore displace a zero-start policy preference. This exception does not change fallback’s strict-growth rule.

Policy 3 never stops merely because an eligible candidate was found. With request zero, all completed candidates are eligible.

### Other policy values

Every signed policy value other than 1, 2, and 3 selects through fallback alone. The request has no effect on selection or termination.

### Final result and failure

On normal completion, a policy result with a nonzero start takes precedence. Otherwise, fallback supplies both position and quantity. The returned quantity and published position belong to the same selected pair.

The function directly writes the selected position once, as a four-byte unsigned value, then returns the selected quantity.

A zero status from F-c9f1dc723c52 instead causes immediate return of **4,294,967,295**. The classification from that call is not processed. Previously completed candidates do not alter this outcome, and the function performs no direct destination write on this path.

Position and quantity arithmetic have unsigned 32-bit width, without saturation. Position wraparound is observable. The representation alone must not be used to invent an execution that overflows the candidate quantity.

## Side effects

### Memory access and ordering

The established logical ordering is:

- Boundary capture precedes all classification calls.
- Each dependency status is checked before processing that call’s classification.
- Policy-1 selection can precede candidate completion and fallback consideration.
- An exact policy-2 candidate receives fallback consideration before termination.
- Direct destination publication occurs only after normal selection.
- Dependency failure bypasses that direct publication.

These are behavioral ordering properties, not guarantees of synchronization, atomicity, or exact physical memory-access counts.

The destination is not initialized on entry. If an external memory contract permits it to overlap the boundary field, normal publication can overwrite that field. The already captured examination limit remains unchanged.

### Classification storage

F-c9f1dc723c52 receives the address of reusable invocation-local signed 32-bit classification storage. No initialization before its first use is established. The function does not supply a fresh classification value before each call.

A nonzero status therefore does not, by itself, establish that the dependency assigned a classification. If a prior call established a defined value and a later call leaves it intact, the retained value may govern the later position. Whether that behavior satisfies the dependency’s contract is unknown.

Contribution and completion depend on distinct logical classification observations. When a positive candidate remains pending and policy 1 has not stopped, completion observes the classification even at the boundary.

If the execution model permits different, defined observations during one visit:

- A contributing observation followed by a separating observation can count the position and complete the resulting candidate during that visit.
- A separating observation followed by a contributing observation can leave an existing candidate pending below the boundary without counting the current position.
- At the boundary, a positive pending candidate completes regardless of the completion-related classification value, unless policy 1 already stopped.

These conditional consequences do not establish permission for concurrent mutation, retained-address access, data races, or indeterminate reads. The mechanisms that could permit such observations remain **documented_only**. Undefined observations have no deterministic result specified here.

### Ownership and external state

No direct allocation, release, reservation, or ownership transfer is established.

The classification storage belongs to the invocation. Permission for the dependency to retain its address, access it after returning, or access it after the invocation ends is not established.

The dependency’s wider effects are unknown. Neither success nor failure guarantees preservation of globals, caller storage, or resources. No rollback or cleanup guarantee is established.

In particular, absence of a direct destination write on failure does not guarantee that the destination remains unchanged: dependency effects are not constrained by the available evidence.

## Dependencies

### F-c9f1dc723c52

| Interaction | Established contract at this boundary |
|---|---|
| Context input | Original signed 32-bit context, unchanged |
| Position input | Current unsigned 32-bit position |
| Classification destination | Address of reusable invocation-local signed 32-bit storage |
| Status output | Signed 32-bit; zero causes immediate failure, every nonzero value permits processing |
| Classification interpretation | Zero contributes; nonzero separates, subject to the observation qualifications above |

Input restrictions, assignment obligations, failure causes, termination guarantees, external memory effects, resource ownership, cleanup duties, and address-retention permissions remain **documented_only**.

Calling this dependency does not establish that it preserves globals or owns, releases, or retains any memory.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. Unless stated otherwise, accesses are valid, calls return nonzero status, and classifications are defined and consistent within each visit.

Each result is **(published position, returned quantity)**.

### Finite examination

| Conditions | Required outcome |
|---|---|
| Initial position 24; boundary 23 | Capture the boundary, make no classification calls, publish **(0, 0)** |
| Finite interval containing only separators | Publish **(0, 0)** for every policy and request |
| Positions 10–14 contribute; 15 separates and is the boundary; policy 1; request 3 | Stop at 12; publish **(10, 3)** |
| Same interval; policy 2; request 3 | Complete examination; publish **(10, 5)** |
| First contribution at 10; policy 1; request zero | Stop at 10; publish **(10, 1)** |
| Completed quantities 7, 5, and 6, all with nonzero starts; policy 2; request 3 | At finite exhaustion, select the quantity-five candidate |
| Quantity seven precedes quantity three, both with nonzero starts; policy 2; request 3 | Stop when quantity three completes; select that candidate |
| Largest completed quantity two; request four; finite exhaustion | Policies 1–3 return the quantity-two fallback |
| Equal largest sufficient candidates start at 10 and 30; policy 3 | Retain the candidate starting at 10 |
| Finite examination; policy −7 | Select the earliest largest completed candidate, independently of request |

### Position zero

| Conditions | Required outcome |
|---|---|
| Initial position and boundary zero; zero contributes; policy 1; request zero or one | Stop before completion; publish **(0, 0)** |
| Same singleton interval; policy 2; request one | Complete and stop; publish **(0, 1)** through fallback |
| Initial position zero; boundary three; all contribute; fallback-only policy | Publish **(1, 3)** |
| Initial position zero; boundary five; only zero and 2–4 contribute; policy 2; request zero | Publish **(2, 3)**; the larger later excess replaces the zero-start preference |
| Initial position zero; boundary four; only zero and three contribute; policy 3; request one | Publish **(3, 1)**, although fallback retains the singleton at zero |

### Wraparound and limits

Start at **4,294,967,294**, with maximal boundary, policy 2, and request one. Let both highest positions contribute. Their quantity-two candidate completes at the boundary. After wraparound, let zero contribute and one separate. The singleton at zero is exact and stops examination, but fallback supplies **(4,294,967,294, 2)**.

Starting at one with maximal boundary, maximal request, policy 1, and consistently contributing classifications produces **(1, 4,294,967,295)**. This normal quantity equals the failure representation.

With maximal boundary, fallback-only selection, and indefinitely returning successful separator classifications, examination does not terminate.

### Failure and observation conditions

- A zero dependency status after earlier completed candidates returns **4,294,967,295**, with no direct destination publication by this function.
- A negative status permits processing of defined classification observations.
- With initial position 10, boundary 11, policy 1, and request two, a first call establishing zero classification followed by a call leaving that defined value intact produces **(10, 2)**.
- If the captured boundary is 11, a dependency change of the backing field to 10 does not exclude position 11.
- With initial position 10, boundary 12, and fallback-only selection, permitted defined observations that contribute and then separate at position 10 complete a singleton. If later visits separate, the result is **(10, 1)**.

## Uncertainties

The following remain **documented_only**:

- Application meanings and units of positions, quantities, classifications, and policies.
- The reason zero also denotes an absent start.
- How callers distinguish failure from a maximal successful quantity.
- Context layout beyond the boundary field.
- Supported address representation, valid address arithmetic, alignment, and byte order.
- Permitted overlap, aliasing, synchronization, and publication atomicity.
- Dependency assignment requirements, failure conditions, termination, and external effects.
- Ownership, release, cleanup, and classification-address retention permissions.
- Permitted mechanisms for changing classification observations within a visit.
- Exact physical memory-access counts.
- Consequences of invalid accesses, indeterminate observations, and lifetime violations.

An unresolved behavior is neither permission to exercise it nor a preservation guarantee.

## Evidence

**Established static evidence:** the supplied representation supports the interface widths, boundary displacement and one-time capture, inclusive unsigned examination, dependency interaction, zero-status failure rule, reusable classification storage, candidate accounting, strict fallback comparison, policy-specific selection, position-zero exceptions, publication precedence, and position wraparound described here.

**Inferred intent:** selection among contributing regions by quantity. Allocation, availability, reservation, and ownership semantics are not established.

**Unavailable evidence:** an independent dependency contract, dependency implementation, caller contracts, supported address model, permitted mechanisms for changing classification observations, and runtime observations.

No runtime tests have been performed.
