<!-- Trusted provenance: F-337a2fdf4aae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-337a2fdf4aae

## Purpose

F-337a2fdf4aae dispatches a size-based resource operation using a context, an element width, an old element count, a requested element count, an existing resource value, and a destination for a 32-bit status word.

The visible behavior is consistent with an allocation/reallocation/release wrapper, but that interpretation is **inferred intent**, not an established contract of the dependencies. Established behavior is the selection of dependency calls, their arguments, the returned value, and the final status write.

The function distinguishes:

- Invalid width or negative counts.
- A request for zero elements.
- A nonzero request rejected by a dependency-derived limit.
- A nonzero request with zero old elements.
- A nonzero request with nonzero old elements.

Failures do not uniformly produce a zero return value. Several branches return the supplied existing resource value, including when that value is nonzero. Consumers must interpret the status word separately from the return value.

## Interface

### Inputs

The function has six positional inputs:

| Position | Contract role | Representation and use |
|---|---|---|
| 1 | Context | An address-like value passed to dependencies and, on one branch, used to locate an indirect callback. |
| 2 | Element width | A signed integer. It must be strictly positive to pass initial validation. |
| 3 | Old element count | A signed integer. It must be nonnegative to pass initial validation. |
| 4 | Requested element count | A signed integer. It must be nonnegative to pass initial validation. |
| 5 | Existing resource value | An integer-sized, pointer-like value. Zero is tested explicitly on the zero-old-count branch; otherwise it is forwarded or returned without validation. |
| 6 | Status destination | An address at which the function writes one 32-bit status word before normal return. |

The supplied representation uses integer-typed operands and a 32-bit status storage type. Its constants and pointer-like integer usage are consistent with a 32-bit execution model. Exact ABI details, pointer representation, and multiplication behavior beyond the representable signed range remain subject to the uncertainties below.

### Outputs

On normal return, the function produces both:

1. An integer-sized result, often representing a resource value.
2. A 32-bit status written through the sixth input.

Directly selected status values are:

| Status | Established selection |
|---|---|
| 0 | Zero-request branch, or a nonzero callback result on the old-count-nonzero branch. |
| 6 | Initial validation failure. |
| 10 | The limit dependency returns a signed value smaller than the requested count. |
| 64 | The indirect callback returns zero. |
| Dependency-supplied value | The zero-old-count branch completes its acquisition dependency call. |

The status destination is not checked for nullness, alignment, writability, or overlap. A caller requiring normal completion must provide storage valid for the final 32-bit write.

Context validity is branch-dependent. Invalid-input branches do not dereference the context. The indirect-callback branch requires a readable callable entry at byte offset 12 from the context, along with whatever additional validity that callback requires.

## Behavior

### Initial validation

Before making any dependency call, the function tests the signs and zero values needed to enforce all three conditions:

- Element width is greater than zero.
- Old element count is at least zero.
- Requested element count is at least zero.

If any condition fails:

- No visible dependency is called.
- The existing resource value is retained as the return value.
- Status 6 is written to the status destination.
- The function returns.

This validation applies even when the requested count is zero. A negative old count or a nonpositive width therefore prevents entry into the zero-request branch.

The function also initializes a private 32-bit status cell to zero. That cell is later exposed by address only to the acquisition dependency.

### Valid request for zero elements

When initial validation succeeds and the requested count is zero:

- F-bf7242c98d06 is called with the context and existing resource value.
- Its return value, if any, is not used.
- If the call returns normally, the function selects a zero return value and status 0.
- The final status write occurs after the dependency call.

This call is made even if the existing resource value is zero. The old element count does not otherwise alter this branch.

Interpreting this dependency as a release operation is plausible, but the visible caller does not establish that memory is freed, that a zero resource is accepted safely, or that the dependency reports failure.

### Valid request for a nonzero number of elements

For any initially valid request with a nonzero requested count, F-870c4fcf7295 is called before either the acquisition dependency or the indirect callback.

Its two arguments are:

- The integer constant 2,147,483,647.
- The positive element width.

The returned integer is compared with the requested count using a signed less-than comparison.

If the returned value is smaller than the requested count:

- Status 10 is selected.
- The existing resource value remains the return value.
- Neither F-3ce007b692cc nor the indirect callback is called.
- The final status write follows the limit check.

Equality passes this check. No established evidence identifies the dependency’s operation as division or proves that this comparison prevents arithmetic overflow.

### Accepted nonzero request with zero old elements

If the limit comparison permits the request and the old element count is zero, the function first examines the existing resource value.

If that value is nonzero, it calls F-540af0155702 with:

- A fixed diagnostic-text reference.
- The integer diagnostic marker 139.

There is no direct return, status assignment, or alternate branch immediately following this diagnostic call. If it returns normally, execution continues into the acquisition call. If it terminates execution or transfers control elsewhere, the later acquisition call and final status write are not reached.

The acquisition dependency, F-3ce007b692cc, receives:

- The context.
- The product of element width and requested element count.
- The address of the private, initially zero, 32-bit status cell.

Its return becomes the function’s return value. The status word left in the private cell becomes the final output status.

The caller performs no additional consistency check between these two outputs. In particular:

- A zero result does not independently force status 64 on this branch.
- A nonzero result does not independently force status 0.
- A nonzero status does not cause the returned result to be replaced with the existing resource value.

The existing resource value is not passed to this acquisition dependency. If it was nonzero and the diagnostic returned, the visible caller does not release it before proceeding.

### Accepted nonzero request with nonzero old elements

If the limit comparison permits the request and the old element count is nonzero, the function loads and calls an indirect function entry located at byte offset 12 from the context.

The callback receives four arguments, in this order:

1. Context.
2. Element width multiplied by old element count.
3. Element width multiplied by requested element count.
4. Existing resource value.

Because validation has already succeeded, both counts and the width are positive on this branch. The existing resource value may nevertheless be zero; this branch contains no diagnostic or rejection for that case.

The callback result controls the wrapper’s result:

- **Nonzero callback result:** return that value and select status 0.
- **Zero callback result:** return the original existing resource value and select status 64.

The private status cell is not passed to the callback and remains zero in the visible caller.

Returning the original resource value after a zero callback result does **not** establish that the resource remains valid. The callback’s ownership and failure semantics are unknown.

### Final status write

Every path that reaches the common normal-return tail writes the selected 32-bit status to the supplied destination and then returns the selected result.

This write occurs after any selected dependency call. There is no early status initialization at the caller-provided destination. Consequently, a dependency that does not return normally can prevent the status destination from being updated.

## Side effects

The directly established caller-side memory operations are:

- Initializing a private 32-bit status cell.
- Reading a callable entry at context byte offset 12 on the indirect branch.
- Allowing F-3ce007b692cc to update the private status cell through its address.
- Writing the final 32-bit status to the supplied destination.

The context-entry read occurs only after F-870c4fcf7295 has returned and the limit check has passed. The visible caller does not cache that entry before the limit dependency runs.

No general ordering between evaluation of the callback’s arguments and its entry-address load is established by the supplied representation. Both precede the actual indirect call.

The final status destination may overlap other caller-controlled storage unless an external contract prohibits that. Such overlap can make the final status write observable as a mutation of context or resource storage. The function contains no overlap detection.

No global state, callback-table contents, or resource state is explicitly saved and restored around dependency calls. Their effects must not be assumed absent merely because the caller subsequently selects status 0.

### Ownership

The visible caller forwards resource-like values and selects return values; it does not itself establish allocation or ownership rules.

In particular:

- The zero-request branch calls a release-like dependency, but release is not proven.
- The acquisition branch returns whatever F-3ce007b692cc supplies.
- The callback branch forwards both size products and the existing resource value.
- The callback-zero branch preserves the original value numerically, not necessarily its ownership or validity.
- Initial rejection invokes no resource dependency, but the final status write can still affect aliased storage.

## Dependencies

| Opaque function ID | Visible role and contract boundary |
|---|---|
| F-bf7242c98d06 | Called for valid zero-count requests with context and existing resource value. Return information is ignored. Release semantics are documented_only. |
| F-870c4fcf7295 | Called for valid nonzero requests with 2,147,483,647 and element width. Its signed result gates further work. Mathematical meaning and side effects are documented_only. |
| F-3ce007b692cc | Called after a passing limit check when the old count is zero. Receives context, requested-size product, and a writable private status-cell address. Supplies both returned result and status. |
| F-540af0155702 | Called when old count is zero but the existing resource value is nonzero. Receives fixed diagnostic text and marker 139. Whether it returns is documented_only. |
| Unidentified indirect callback | Loaded from context byte offset 12. Receives context, old-size product, requested-size product, and existing resource value. Zero and nonzero results are handled differently. Its identity and ownership contract are documented_only. |

There is no visible recursive call to F-337a2fdf4aae.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. They assume valid storage and that all mentioned calls return normally.

1. **Zero width is rejected.**  
   With width 0, nonnegative counts, and existing resource value R, the result is R and status is 6. No dependency is called.

2. **A negative old count overrides a zero request.**  
   With width 4, old count −1, and requested count 0, the result remains the existing resource value and status is 6. The zero-request dependency is not called.

3. **A valid zero request dispatches even for a zero resource.**  
   With width 4, old count 3, requested count 0, and existing resource value 0, F-bf7242c98d06 receives the context and 0. After it returns, the result and status are both zero. F-870c4fcf7295 is not called.

4. **A limit rejection retains the original value.**  
   With width 4 and requested count 8, suppose F-870c4fcf7295 returns 7. The result is the supplied existing resource value and status is 10, regardless of whether the old count is zero or positive.

5. **Equality at the limit proceeds to acquisition.**  
   With width 4, old count 0, requested count 8, and existing resource value 0, suppose the limit dependency returns 8. F-3ce007b692cc receives a requested size of 32. If it returns R and writes status 5, the wrapper returns R and writes status 5.

6. **Acquisition results are not normalized.**  
   Under the preceding acquisition conditions, if F-3ce007b692cc returns zero and leaves its status cell unchanged, the wrapper returns zero and writes status 0, not 64.

7. **A returning diagnostic does not stop acquisition.**  
   With width 4, old count 0, requested count 8, existing resource value R ≠ 0, and a passing limit result, F-540af0155702 is called first. If it returns, F-3ce007b692cc is still called with requested size 32.

8. **A successful callback replaces the result.**  
   With width 4, old count 3, requested count 8, and a passing limit result, the callback receives old size 12 and requested size 32. If it returns Q ≠ 0, the wrapper returns Q and writes status 0.

9. **A zero callback result selects status 64.**  
   Under the same callback conditions, if the callback returns zero, the wrapper returns the supplied existing resource value and writes status 64. If that original value was zero, the result is also zero.

## Uncertainties

- **documented_only — Arithmetic model:** The integer operations appear 32-bit, but signed-overflow behavior and exact machine multiplication semantics are not independently established.
- **documented_only — Limit calculation:** The constant and comparison suggest a requested-size bound check. The dependency’s implementation is unavailable, so division and overflow protection cannot be asserted.
- **Established unchecked boundary:** No corresponding limit check is made against the old count. Even if the requested-size check is protective, the old-size product has no independently visible bound.
- **documented_only — Resource semantics:** Allocation, release, movement, preservation on failure, and ownership transfer depend on unavailable callees.
- **documented_only — Diagnostic behavior:** The diagnostic may return, terminate, or transfer control. Only the returning continuation is visible.
- **documented_only — Dependency state changes:** Dependencies may affect shared state. No preservation guarantee is established.
- **documented_only — Invalid addresses:** Behavior for invalid context entries, resource values, or status destinations is not defined by visible validation.
- **documented_only — Concurrency and reentrancy:** No synchronization or shared-state contract is established.

## Evidence

This contract is based solely on the supplied static representation of F-337a2fdf4aae and its provided opaque dependency mapping.

Established evidence covers branch conditions, literal status values, argument construction, the callback-entry offset, return-value selection, and the final status write. Allocation-related terminology is explicitly interpretive where dependency behavior is unavailable.

No runtime tests, dependency implementations, ABI inspection, or independent execution observations were available. The acceptance examples express conditional consequences of the visible control flow, not measured results.
