<!-- Trusted provenance: F-6375e6fe2adb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6375e6fe2adb

## Purpose

F-6375e6fe2adb advances two integer cursors in steps of 24 until the first cursor equals a supplied endpoint. At each visited position, it invokes an external dependency if the current second cursor is nonzero. It returns the second cursor after all completed iterations.

The established behavior is an equality-terminated traversal with an optional call at each position. A plausible intent is to traverse corresponding fixed-stride locations, but the supplied evidence does not establish that either cursor denotes memory, that the stride is a record size, or that the dependency copies, constructs, destroys, or transfers anything.

Zero has a narrow role: a zero second cursor suppresses the dependency call for that iteration. It does not stop traversal or prevent advancement.

## Interface

### Inputs

The function accepts three integer values, described here by role rather than source identifiers:

| Input | Role |
|---|---|
| Traversal cursor | Compared with the endpoint before each iteration; passed as the second dependency argument when a call occurs; advanced by 24 after that iteration. |
| Endpoint | Exclusive stopping value for the traversal cursor. Equality, not ordering or range membership, controls termination. |
| Companion cursor | Tested against zero at each visited position; passed as the first dependency argument when nonzero; advanced by 24 each iteration and ultimately returned. |

The supplied representation declares these inputs as integer values. Their exact bit width, machine argument representation, and calling convention are not independently established. In particular, the evidence does not justify treating them unconditionally as unsigned 32-bit addresses.

The increment is the integer value 24. Whether it represents bytes, elements, offsets, handles, or another unit is unknown. There is no visible element-type scaling.

### Output

On normal return, the function returns the final companion cursor as an integer value.

For a normally completed traversal of \(n\) iterations, with every addition valid under the applicable arithmetic semantics:

- The traversal cursor has advanced by \(24n\).
- The returned companion cursor is its initial value plus \(24n\).
- The dependency has been called once for each iteration whose companion cursor was nonzero.

The iteration count includes iterations that skip the dependency because the companion cursor is zero.

### Preconditions and limits

The visible function performs no explicit validation. Useful execution therefore depends on several conditions:

- The endpoint must be reachable by the traversal cursor’s repeated additions if termination is required.
- Arithmetic must remain valid under the actual integer model.
- Each dependency call must meet that dependency’s unknown preconditions.
- Each dependency call must return normally if the enclosing function is expected to continue.

There is no visible requirement that the endpoint be greater than the traversal cursor. There is also no explicit check for alignment, accessible memory, overlap, or allocation size.

## Behavior

### Entry and the empty traversal

The first observable control decision is an equality comparison between the traversal cursor and the endpoint.

If they are equal on entry:

- No dependency call occurs.
- Neither cursor is advanced.
- The original companion cursor is returned, including when it is zero.

This is an exclusive-end traversal: the dependency is never called for a traversal position that already equals the endpoint at the loop comparison.

### Per-iteration behavior

Whenever the traversal cursor differs from the endpoint, the function performs the following ordered actions:

1. It tests the current companion cursor against zero.
2. If the companion cursor is nonzero, it invokes the dependency with the companion cursor first and the traversal cursor second.
3. After that call returns normally, or immediately if the call was skipped, it advances the companion cursor by 24.
4. It then advances the traversal cursor by 24.
5. It compares the updated traversal cursor with the endpoint before considering another iteration.

Both arguments supplied to the dependency are the values before either cursor is advanced for that iteration.

The companion-cursor update precedes the traversal-cursor update in the supplied representation. No dependency invocation occurs between those updates. If an earlier action fails to complete normally, subsequent actions are not established as having occurred.

### Meaning of a zero companion cursor

Zero suppresses only the current iteration’s call.

For example, when the companion cursor starts at zero and at least two iterations complete under ordinary representable arithmetic:

- The first iteration skips the dependency.
- The companion cursor becomes 24.
- The second iteration invokes the dependency with 24 as its first argument.

Consequently, an initially zero companion cursor does not disable dependency calls for the whole traversal. Likewise, any later iteration that encounters zero skips its own call but still performs both increments.

The source does not establish that zero means a null pointer, absent object, or invalid handle. Only the zero comparison and its effect on call suppression are established.

### Termination and arithmetic

The stopping rule is exact equality. The function does not stop when the traversal cursor exceeds the endpoint.

Under non-wrapping integer arithmetic, with all executed additions representable, normal termination after \(n\) iterations requires:

\[
\text{endpoint}=\text{initial traversal cursor}+24n,
\qquad n\geq 0.
\]

Thus, within that arithmetic model:

- Equal initial traversal cursor and endpoint produce zero iterations.
- A larger endpoint is reached only when the difference is an exact nonnegative multiple of 24.
- A smaller endpoint is not reached by repeated positive increments.
- An endpoint between two visited positions is not treated as reached or passed.

These statements do not establish portable behavior after signed overflow. The supplied integer representation does not itself authorize modular wrapping.

If separate evidence establishes fixed-width wrapping arithmetic, endpoint reachability must instead be analyzed modulo that width. A return reached only after wrapping must not be accepted solely on the evidence supplied here.

The companion cursor has its own arithmetic constraint. Even when the traversal cursor can reach the endpoint without overflow, companion-cursor advancement may overflow sooner. The return formula is therefore conditional on valid companion arithmetic as well as traversal arithmetic.

### Dependency results and abnormal completion

The function does not inspect a dependency return value. There is no visible success flag, retry, rollback, or error conversion.

If the dependency reports failure through a normally returned value, that value does not change the visible traversal behavior. If it reports failure through shared state, those effects remain subject to the dependency’s behavior; this function does not visibly inspect them.

If the dependency does not return normally, the supplied evidence does not establish a normal return from F-6375e6fe2adb. In particular, neither the remaining cursor updates nor later calls should be assumed to occur.

## Side effects

### Direct effects

The visible body directly:

- Compares integer cursor values.
- Updates its local cursor state.
- Conditionally invokes one dependency.
- Returns an integer value.

It contains no explicit memory dereference, allocation, release, global access, or write through either cursor. Treating the integer values as addresses would not, by itself, establish any direct memory access by this function.

The endpoint is not modified. The caller’s argument storage is not visibly updated through an output pointer; the final companion value is communicated by the return value.

### Indirect effects and ownership

All effects caused by the dependency remain possible within its unknown contract. The supplied evidence does not establish:

- Which memory, globals, or external resources it reads or writes.
- Whether it acquires, transfers, retains, or releases ownership.
- Whether it preserves unrelated global state.
- Whether it invokes callbacks or reenters this function.
- Whether argument values must refer to distinct or nonoverlapping regions.
- Whether prior calls affect the validity of subsequent calls.

No caller may infer ownership preservation, copying semantics, or resource release merely from this wrapper.

The function provides no visible cleanup for partially completed traversal. If a later dependency invocation fails abnormally, effects from earlier invocations are not visibly undone.

## Dependencies

### F-external-1

**F-external-1** is a document-local opaque identifier for the single external dependency visible in the supplied representation. A published opaque identifier for that dependency was not provided.

Its established use is:

| Property | Established behavior |
|---|---|
| Invocation condition | Current companion cursor is nonzero, and traversal cursor differs from endpoint. |
| First argument | Current companion cursor, before advancement. |
| Second argument | Current traversal cursor, before advancement. |
| Frequency | At most once per iteration. |
| Return handling | No returned value is used by the enclosing function. |
| Sequencing | A normal return precedes both cursor increments. |

Its implementation, argument interpretation, ownership rules, error behavior, and state preservation are **documented_only** unknowns.

## Acceptance examples

The following are conditional acceptance examples, not reported executions. Unless stated otherwise, they assume all arithmetic is representable and every dependency call returns normally. Call pairs are listed in dependency argument order: companion cursor first, traversal cursor second.

### Empty traversal

Inputs:

- Traversal cursor: 96
- Endpoint: 96
- Companion cursor: 700

Expected behavior:

- No dependency call.
- Return value: 700.

The same equal-boundary case with companion cursor zero returns zero without a call.

### One ordinary iteration

Inputs:

- Traversal cursor: 96
- Endpoint: 120
- Companion cursor: 700

Expected behavior:

- One call with argument pair \((700,96)\).
- Return value: 724.
- No call at traversal position 120.

This distinguishes pre-increment arguments from post-increment arguments.

### Multiple iterations

Inputs:

- Traversal cursor: 96
- Endpoint: 168
- Companion cursor: 700

Expected calls, in order:

1. \((700,96)\)
2. \((724,120)\)
3. \((748,144)\)

Expected return value: 772.

### Initially zero companion cursor

Inputs:

- Traversal cursor: 96
- Endpoint: 144
- Companion cursor: 0

Expected behavior:

- The iteration at traversal position 96 skips the dependency.
- The iteration at traversal position 120 calls with \((24,120)\).
- Return value: 48.

An implementation that suppresses all calls because the initial companion cursor is zero would not match the supplied behavior.

### Companion cursor reaches zero

Inputs:

- Traversal cursor: 96
- Endpoint: 168
- Companion cursor: −24

If these values and additions are valid in the actual integer model, expected behavior is:

1. Call with \((-24,96)\).
2. Skip the call at traversal position 120 because the companion cursor is then zero.
3. Call with \((24,144)\).
4. Return 48.

This example establishes that nonzero negative values are not excluded by the visible call condition. It does not establish their validity for the dependency.

### Unreachable endpoint without wrapping

Inputs:

- Traversal cursor: 96
- Endpoint: 121
- Companion cursor: 700

Under non-wrapping arithmetic, the visited traversal values do not reach 121. The function must not be specified to stop at 120 or 144 merely because either is near the endpoint.

No finite return value is established for this case. Actual execution may eventually encounter an arithmetic limit or another failure.

### Dependency does not return normally

For any iteration that invokes F-external-1, if that invocation does not return normally:

- No normal return value from the enclosing function is established.
- The two following increments are not established as completed.
- Earlier dependency effects are not guaranteed to be reversed.

## Uncertainties

The following remain **documented_only**:

- Exact integer width, signedness at the machine interface, and overflow behavior.
- Whether cursor values represent addresses, offsets, object identifiers, or other quantities.
- Any memory layout beyond the observed numeric stride of 24.
- Required alignment, accessible extent, and overlap restrictions.
- The dependency’s published opaque identifier and complete contract.
- Ownership, global-state preservation, failure signaling, and reentrancy.
- Behavior once arithmetic is outside the valid range of the applicable integer model.

Traversal over paired fixed-size records is inferred intent only. It is not a basis for adding copying semantics, pointer validity guarantees, or cleanup obligations.

## Evidence

Established static evidence consists solely of the supplied function representation:

- Three integer inputs and an integer return.
- An equality-controlled loop.
- A zero test of the companion cursor.
- One conditional external call with the companion cursor preceding the traversal cursor.
- Companion advancement by 24 followed by traversal advancement by 24.
- Return of the resulting companion cursor.

The formulas and acceptance examples are deductions from those operations under their stated conditions. No runtime tests, dependency inspection, memory observations, or machine-width verification have been performed.
