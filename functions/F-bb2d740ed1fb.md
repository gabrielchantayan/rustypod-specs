<!-- Trusted provenance: F-bb2d740ed1fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bb2d740ed1fb

## Purpose

F-bb2d740ed1fb processes a position-change request for an object containing an associated-resource field, dispatch information, and position-related state.

The established behavior is:

- Reject an initially absent resource with status 3.
- Invoke preparation before validating the origin selector.
- Obtain a current-position value through an indirect call.
- Compute a target using unsigned 32-bit arithmetic.
- Require that target to be strictly below a bound read from a writable temporary.
- Return success without movement when the valid target equals the queried position.
- Otherwise select either reset-and-position processing or an alternate forward operation.

“Position,” “bound,” and “reset” are descriptive roles inferred from the operations. The supplied body does not establish resource type, position units, or the complete effects of any dependency.

The bound is not guaranteed to remain equal to the temporary’s value immediately after preparation. Preparation receives the temporary’s address; the subsequent indirect query could modify that still-live temporary through an address retained by preparation. Target computation and validation use the temporary when those respective reads occur.

## Interface

### Arguments and result

| Argument | Representation in the supplied model | Use |
|---|---|---|
| Object | Pointer to an object accessed in 32-bit words | Supplies dispatch information, resource state, and movement-related fields. |
| Displacement | Unsigned 32-bit integer | Supplies an absolute target or an unsigned addend. |
| Origin selector | Signed 32-bit integer | Passed twice to preparation, then interpreted locally as 0, 1, or 2. |
| Auxiliary value | Unsigned 32-bit integer | Initializes a writable temporary and is separately passed to preparation. |

The result is a signed integer status in the supplied 32-bit model:

| Condition | Result |
|---|---|
| Initial resource field is zero | 3 |
| Preparation returns nonzero | Preparation result, unchanged |
| Unsupported selector | 6 |
| Target fails the strict bound check | 6 |
| Valid target equals queried position | 0 |
| Reset-and-position path | Underlying positioning result, unchanged |
| Alternate forward path completes | 0, irrespective of that dependency’s return value |

A dependency can itself return 3 or 6. Those numeric results do not uniquely identify a locally generated rejection.

The displacement is not interpreted as signed. A bit pattern conventionally used for a negative displacement remains an unsigned value here. Addition may nevertheless wrap into a smaller value.

### Relevant layout

The supplied representation establishes these accesses under a 32-bit word model:

| Object byte offset | Width | Role |
|---|---:|---|
| 0 | 32-bit dispatch-address word | Locates the table used for the indirect query. |
| 28 | 32 bits | Unsigned comparison operand for forward-path selection. |
| 32 | 32 bits | Added to the unsigned forward distance. |
| 36 | 32 bits | Receives the target on the reset-and-position path. |
| 48 | 32 bits | Associated-resource value. |

The indirect query entry is at byte offset 64 within the dispatch table. It receives the object, and its result is consumed as an unsigned 32-bit value.

These accesses do not establish the complete object layout, field ownership, or an independently verified ABI.

### Preconditions

Defined execution requires valid storage and call targets for every access reached:

- The object must support the applicable reads and, on the reset-and-position path, the write at offset 36.
- After successful preparation, the dispatch information and selected indirect entry must be callable.
- Dependencies must receive arguments satisfying their own requirements.
- Any dependency access through the temporary’s address must respect its storage lifetime and applicable memory-access rules.

There is no explicit null-object check, validation of a nonzero resource, dispatch validation, or exception handler.

## Behavior

### Initial resource check

The first operation checks the resource field at offset 48. If it is zero, the function returns 3 without invoking a dependency or directly writing an object field.

This rejection precedes all selector handling and target computation.

On the nonzero path, the resource field supplies the preparation argument. The body does not establish an atomic snapshot across distinct accesses or dependency calls.

### Preparation and temporary lifetime

A writable 32-bit temporary is initialized from the auxiliary argument. F-5c1bc43ac1f8 then receives, in order:

1. The associated-resource value.
2. The temporary’s address.
3. The origin selector.
4. The original auxiliary argument.
5. The origin selector again.

A nonzero preparation result is returned immediately. No position query or subsequent local validation occurs on that path. Preparation effects are not rolled back.

A zero result allows execution to continue. It does not establish that the temporary remains unchanged afterward.

The temporary remains alive throughout the wrapper invocation, including during the later indirect query and movement calls. Retaining its address after preparation returns does not, by itself, exceed that lifetime. Using the address after the wrapper returns would require a lifetime guarantee not supplied here.

### Indirect current-position query

After successful preparation, the function obtains the dispatch entry at table offset 64 and invokes it with the object.

Its result becomes the unsigned queried position. There is no separate numeric error check: every 32-bit result participates in later arithmetic and comparisons.

The query occurs before selector rejection, selector-2 target computation, and bound validation. Consequently:

- Preparation can affect the dispatch state used to select the query.
- The query can affect object fields read later.
- The query could modify the temporary through an address retained by preparation.
- An unsupported selector still causes the query to run if preparation succeeds.

The body supplies no guarantee that the query is read-only.

### Target computation

Let:

- \(D\) denote the unsigned displacement.
- \(P\) denote the unsigned query result.
- \(B_{\mathrm{base}}\) denote the temporary’s value when read for selector-2 target computation.
- \(T\) denote the computed unsigned target.

| Selector | Target or result |
|---|---|
| 0 | \(T = D\) |
| 1 | \(T = (P + D) \bmod 2^{32}\) |
| 2 | \(T = (B_{\mathrm{base}} + D) \bmod 2^{32}\) |
| Any other signed value | Return 6 |

For selectors 0 and 1, target computation does not require reading the temporary. Selector 2 reads it after the query, not necessarily at the value left immediately after preparation.

No overflow diagnostic is produced. For selector 2, adding a small positive displacement to an unchanged bound normally places the target beyond that bound; wrapping addition can instead produce an admissible target.

### Strict bound validation

Let \(B_{\mathrm{check}}\) denote the temporary’s value when read for validation. The function returns 6 unless:

\[
T < B_{\mathrm{check}}
\]

The comparison is unsigned and 32-bit.

For selector 2, the target-base read and validation read are separate uses of the temporary. The body contains no intervening dependency call between those uses, but it does not establish a general immutability or synchronization guarantee. Their equality is conditional on no permitted modification occurring between them. Neither value is guaranteed to equal the temporary’s immediate post-preparation value.

Consequences:

- A target equal to \(B_{\mathrm{check}}\) is rejected.
- A zero \(B_{\mathrm{check}}\) rejects every target.
- Bound validation precedes the unchanged-position check.
- A target equal to the queried position is still rejected if it fails this bound check.
- Rejection does not undo preparation or query effects.

### Unchanged-position success

If the bound check succeeds and \(T = P\), the function returns 0.

Neither movement path runs, and the function does not directly write offset 36. Preparation and the query have already occurred, so this path is not necessarily side-effect-free.

### Movement-path selection

For a valid target different from the queried position:

- If \(T < P\), select reset-and-position processing without needing the forward-comparison fields.
- If \(T > P\), calculate the unsigned distance \(\Delta = T - P\).

In the forward case, define:

\[
S = \bigl(\operatorname{u32}(\text{offset-32 field}) + \Delta\bigr)
\bmod 2^{32}
\]

Reset-and-position processing is selected when:

\[
\operatorname{u32}(\text{offset-28 field}) < S
\]

Otherwise the alternate forward path is selected. Equality selects the alternate path.

The subtraction does not underflow because this calculation occurs only when \(T > P\). The subsequent addition can wrap, and wrapping affects path selection. Negative-looking stored bit patterns are interpreted unsigned for this comparison and addition.

Both field reads occur after the query. Their relative evaluation order is not established by the supplied expression.

### Reset-and-position processing

The observable sequence is:

1. Invoke F-304d991d26be with the object.
2. Write the target’s 32-bit representation to offset 36.
3. Read the resource field at offset 48.
4. Invoke F-3475b4980949 with that resource, followed by 0, target, 0, and 0.
5. Return its result unchanged.

F-304d991d26be’s result is ignored. Its effects are not.

The resource passed to F-3475b4980949 is read after F-304d991d26be returns. It need not equal the resource used for preparation. There is no second zero-resource check.

The target write precedes the underlying positioning call. A nonzero positioning result does not trigger a compensating write. Dependencies may independently modify the same field.

### Alternate forward processing

The function invokes F-b3acbed32aa7 with the object, ignores its result, and returns 0.

Neither the target nor the forward distance is an explicit argument. The wrapper does not directly write either value into object storage on this path.

Status 0 therefore establishes completion of this wrapper path, not independently verified arrival at the target.

## Side effects

### Direct storage changes

The body directly:

- Initializes the invocation-local temporary.
- Writes the target at object offset 36, only on the reset-and-position path.

Preparation has writable access to the temporary. Other dependencies may also gain access through a retained address; the body neither proves nor excludes such access.

No other direct object write is visible.

### Delegated state and ownership

Dependencies can affect later observations:

- Preparation can change reachable state before selector validation.
- The query can change the temporary or object fields before their subsequent reads.
- F-304d991d26be can change the resource subsequently passed to F-3475b4980949.
- F-b3acbed32aa7 can change state despite having no explicit target argument.
- F-3475b4980949 can change state even when it returns nonzero.

The body contains no explicit allocation, release, reference-count adjustment, or ownership transfer. This does not establish that dependencies preserve ownership, retain resources, or leave them alive.

The temporary’s address is valid storage for this invocation, subject to ordinary access requirements. Retention beyond preparation is distinct from use after the wrapper returns: only the latter necessarily outlasts this invocation-local storage.

### Atomicity

No synchronization, transaction, or rollback boundary is visible. There is no guarantee of a single consistent snapshot across calls and reads. This contract does not grant permission for otherwise invalid concurrent memory access.

## Dependencies

| Opaque function ID | Established call-site behavior |
|---|---|
| F-5c1bc43ac1f8 | Receives resource, writable temporary address, selector, original auxiliary value, and repeated selector. Nonzero result terminates the wrapper unchanged. |
| F-304d991d26be | Receives the object before the target write and underlying positioning call. Result ignored. |
| F-3475b4980949 | Receives the freshly read resource, 0, target, 0, 0. Result becomes the wrapper result. |
| F-b3acbed32aa7 | Receives the object on the alternate forward path. Result ignored; wrapper returns 0. |

There is also an unidentified indirect dependency selected through dispatch-table offset 64. No opaque implementation ID was supplied for it. It receives the object and supplies the value consumed as the unsigned queried position.

These are call-site facts, not complete dependency contracts. No dependency is assumed to preserve globals, preserve the temporary, or retain or release memory.

## Acceptance examples

These are conditional expectations, not executed tests. Unless stated otherwise:

- Preparation returns 0.
- All reached storage and call targets are valid.
- A stated bound is the temporary’s value at the validation read.
- For selector 2, the stated bound also applies at the target-base read.
- No unmentioned dependency changes the values needed by the example.

1. **Missing resource.** An initial zero resource produces 3. No dependency runs, regardless of selector.

2. **Preparation failure takes precedence.** A nonzero initial resource and preparation result 17 produce 17. The query is skipped, even for an unsupported selector.

3. **Unsupported selector still queries.** Selector −1 with successful preparation invokes the query and then returns 6. Neither movement path runs.

4. **Query changes the later bound.** Preparation leaves the temporary at 100 and retains its address. During the query, that address is used to set the temporary to 40. With selector 0 and displacement 50, validation reads 40 and returns 6. The earlier value 100 does not govern validation.

5. **Query changes the selector-2 base.** Preparation leaves the temporary at 100; the query changes it to 60. If it remains 60 through both subsequent reads, selector 2 with displacement 4,294,967,286 produces target 50. With queried position 50, the result is 0 without movement.

6. **Exact endpoint remains invalid.** A validation bound of 100 and selector-0 displacement 100 produce 6, even if the queried position is also 100.

7. **Zero bound admits nothing.** A validation bound of 0 rejects selector-0 displacement 0 rather than returning unchanged-position success.

8. **Valid unchanged target.** Bound 100, queried position 25, and selector-0 displacement 25 produce 0. Neither movement dependency runs; offset 36 is not directly written.

9. **Backward processing records before positioning.** Bound 100, queried position 60, and absolute target 20 invoke F-304d991d26be, then write 20 at offset 36, then pass the subsequently read resource to F-3475b4980949.

10. **Failure does not undo the record.** In the preceding example, if F-3475b4980949 returns 9 and does not overwrite offset 36, the wrapper returns 9 with that field still containing 20.

11. **Forward equality selects the alternate path.** Bound 100, queried position 10, target 15, offset-32 value 3, and offset-28 value 8 produce comparison sum 8. F-b3acbed32aa7 runs, and the wrapper returns 0 regardless of its result.

12. **A smaller comparison field selects repositioning.** With the preceding values except offset 28 contains 7, reset-and-position processing runs because 7 is less than 8.

13. **Current-relative addition wraps.** Selector 1, queried position 4,294,967,290, displacement 10, and validation bound 100 produce target 4. The target passes validation and selects backward processing.

14. **Bound-relative addition wraps.** Selector 2, temporary value 100 at both relevant reads, displacement 4,294,967,246, and queried position 50 produce target 50 and unchanged-position success.

15. **Forward-comparison addition wraps.** Bound 100, queried position 10, target 20, offset-32 bit pattern 4,294,967,290, and offset-28 value 4 produce comparison sum 4. Equality selects the alternate path.

## Uncertainties

### Inferred intent

The selector cases resemble absolute, current-relative, and bound-relative positioning. The forward comparison resembles a choice between an operation within buffered state and underlying repositioning.

These interpretations are not required to describe the established behavior. The bound is not proven to be resource length, and position units are unknown.

### documented_only

Unresolved behavior includes:

- The independently verified ABI and complete object layout.
- The indirect query implementation and any numeric error convention.
- Whether preparation retains the temporary’s address, and whether the query uses it.
- The temporary’s actual values at later reads without dependency contracts.
- Field meanings and invariants at offsets 28, 32, and 36.
- How the alternate forward dependency determines its operation without an explicit target.
- Whether dependencies preserve, replace, retain, or release resources or global state.
- Failure-state usability, reentrancy, and concurrency rules.
- Whether callers intentionally rely on wrapping arithmetic.
- Domain-wide meanings of statuses 3 and 6.

No unresolved dependency behavior is converted into a preservation or ownership guarantee.

## Evidence

The supplied function body is the sole static evidence. It establishes the visible branches, argument order, result propagation, field accesses, temporary-address exposure, unsigned arithmetic, and sequencing of calls and writes.

In particular, preparation receives the temporary’s address, the indirect query runs next, and only afterward is the temporary read for selector-2 arithmetic and bound validation. No supplied dependency contract guarantees preservation across that query.

Dependency implementations, callers, and external interface documentation were not supplied. Descriptive intent remains separate from established call-site behavior.

No runtime tests have been performed. All acceptance examples are conditional specifications derived from static evidence, not observed executions.
