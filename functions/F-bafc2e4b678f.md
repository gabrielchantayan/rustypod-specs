<!-- Trusted provenance: F-bafc2e4b678f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bafc2e4b678f

## Purpose

F-bafc2e4b678f conditionally initializes a 32-bit field in a shared state object. A nonzero field bypasses initialization. A zero field causes a call to F-ccde8fa781e9, whose result determines whether the function stores an address-like value, leaves the field unchanged, or returns a distinct status.

The established behavior is a conditional state update, not allocation or resource acquisition. The interpretation of the field as a cached pointer, and of the dependency result as a mode selector, is inferred intent rather than an established semantic guarantee.

The function returns either the 32-bit value zero or the 32-bit value 19. Zero does not necessarily mean that initialization occurred: several paths return zero without this function writing the field.

## Interface

### Explicit inputs and output

The function has no explicit parameters.

Its return value is 32 bits wide:

- **0:** returned when the inspected field was initially nonzero, when F-ccde8fa781e9 returns 2 or 3, or when that dependency returns 4.
- **19:** returned when the inspected field was zero and F-ccde8fa781e9 returns any value other than 2, 3, or 4.

These are exact numerical outcomes. The supplied evidence does not establish application-level names for either status.

### Implicit state inputs

The function accesses two global locations and a field reached through one of them:

| State element | Established role |
|---|---|
| Primary global | Supplies a 32-bit address-like value identifying a state object. |
| Object field at byte offset 4 | A 32-bit field tested against zero and conditionally written. |
| Alternative global | Supplies a 32-bit value used for the dependency-result-3 path. It is also evaluated on the other non-2 dependency-result paths. |
| Address at byte offset 12 from the captured object base | Computed as a candidate value for the dependency-result-2 path. No contents at that address are read by this function. |

“Primary,” “alternative,” and “object field” are descriptive labels for this contract. They do not establish public symbol names or application-level types.

The object base is captured before calling F-ccde8fa781e9. A subsequent write uses that captured base, not a newly loaded base after the call.

### Widths and layout

The supplied representation establishes 32-bit integer values for the field, dependency result, candidate value, and function result. The address-bearing operations are represented through 32-bit integer quantities.

Offsets 4 and 12 are **byte offsets**, not counts of 32-bit elements:

- The inspected and conditionally updated field occupies four bytes beginning at object offset 4.
- The value computed for dependency result 2 is the captured base plus 12.
- The function does not read the four bytes, or any other bytes, at object offset 12.

Byte order, target alignment requirements, and the complete object layout are not supplied.

### Preconditions

Normal completion requires that the global reads and the field accesses performed on the selected path are valid.

In particular:

- The base used for the initial field inspection must support a readable 32-bit field at offset 4.
- If an assignment path is taken, the captured base must still support a writable 32-bit field at offset 4 after the dependency returns.
- The alternative global must be readable on every path where the dependency returns a value other than 2, including the paths returning without a field assignment.
- Address calculations must be meaningful under the target execution environment.

There is no explicit null check, bounds check, alignment check, lifetime check, or access-failure recovery in the supplied body.

## Behavior

### Initial inspection

The function first captures the address-like value supplied by the primary global.

It then inspects the 32-bit field at offset 4 through the primary global. The supplied representation expresses the initial capture and the base evaluation for this inspection separately. It does not establish whether the executable performs one physical global load or multiple physical loads.

If the inspected field is nonzero:

- F-ccde8fa781e9 is not called.
- The alternative global is not evaluated by this function.
- This function performs no field assignment.
- The function returns zero.

The test is only zero versus nonzero. No validity check is made on a nonzero value, and no distinction is made between different nonzero bit patterns.

### Zero-field path

If the inspected field is zero, the function calls F-ccde8fa781e9 once, with no explicit arguments.

Subsequent behavior depends on the dependency’s returned 32-bit value.

| Dependency result | Subsequent action by this function | Return |
|---|---|---|
| 2 | Compute captured base plus 12 and store the resulting 32-bit value into the field at captured base plus 4. | 0 |
| 3 | Read the alternative global and store that 32-bit value into the field at captured base plus 4. | 0 |
| 4 | Evaluate the alternative global, but perform no field assignment. | 0 |
| Any other value | Evaluate the alternative global, but perform no field assignment. | 19 |

The alternative-global evaluation is present for **all non-2 results**, not only result 3. This distinction can matter if reads of that location have externally visible behavior. Whether the target emits an actual load on paths where the value is unused is not established by the supplied representation.

### Result 2

The candidate value is the address-like value captured before the dependency call, increased by 12 bytes.

The function stores that candidate into the 32-bit field at offset 4 of the captured object. It does not initialize, inspect, or validate storage at offset 12.

The address computation is represented with 32-bit integer quantities. The evidence does not establish a portable rule for overflow, signed-overflow behavior, or address-space wrapping. A normal-use acceptance case must therefore use a base for which adding 12 produces a valid, representable result in the target environment.

### Result 3

The candidate is the alternative global’s value evaluated **after** the dependency returns.

That value is written without validation. In particular, a zero alternative value is accepted and stored. The function still returns zero, even though the field is then zero.

The candidate need not point inside the captured object. No relationship between the alternative value and the object’s address range is checked.

### Results 4 and other values

For result 4, this function returns zero without writing the field.

For any result outside 2, 3, and 4, this function returns 19 without writing the field.

Neither statement guarantees that the field remains equal to its pre-call value. F-ccde8fa781e9 may itself modify shared state, including the field. The body provides no preservation guarantee for that dependency.

### State changes during the dependency call

The field is not rechecked after the dependency returns.

Consequently:

- If the dependency changes the field and returns 2 or 3, this function subsequently overwrites the field at the captured object.
- If the dependency changes the primary global, the assignment still targets the object base captured before the call.
- If the dependency changes the alternative global and returns 3, the later read uses its post-call value.
- If the dependency returns 4 or another unsupported value, this function performs no compensating write or rollback.

The function therefore does not establish an atomic “initialize only if still zero” operation.

## Side effects

### Direct memory effects

The only direct memory write shown is a single 32-bit assignment to the field at captured base plus 4. It occurs only after the initially inspected field is zero and the dependency returns 2 or 3.

No direct writes to either global are shown.

The supplied body contains no explicit allocation, release, reference-count operation, resource acquisition, or resource cleanup. Storing an address-like value does not, by itself, establish ownership transfer or lifetime extension.

### Dependency effects

When called, F-ccde8fa781e9 may have additional effects not described by the supplied evidence. This contract does not assume it:

- Preserves either global.
- Preserves the captured object or its field.
- Keeps the captured object alive.
- Allocates or releases memory.
- Returns normally under all conditions.

Any such guarantees require separate evidence.

### Ordering and concurrency

The source-level ordering established here is:

1. Capture the primary base.
2. Inspect the field through the primary global.
3. On a zero field, call the dependency.
4. After that call, select or read the candidate according to the returned value.
5. On results 2 or 3, write the captured object’s field.
6. Return the selected status.

No lock, atomic operation, memory barrier, or concurrency protocol is shown. Concurrent initialization safety and physical memory-access ordering beyond what the target guarantees are documented_only.

## Dependencies

### F-ccde8fa781e9

- **Invocation condition:** the initially inspected field equals zero.
- **Invocation count:** once per invocation of F-bafc2e4b678f that reaches this path and proceeds normally.
- **Explicit arguments:** none.
- **Consumed result:** a 32-bit integer compared for exact equality with 2, 3, and 4.
- **Other result values:** all select return status 19, without a direct field assignment by this function.
- **State and ownership guarantees:** unknown.

No other callable dependency is shown in the supplied body.

## Acceptance examples

These are conditional contract examples, not executed tests. Unless stated otherwise, they assume valid memory accesses, normal dependency return, no concurrent interference, and no relevant dependency side effects.

### Existing nonzero field

Given that the inspected field contains 8:

- The dependency is not called.
- This function does not assign the field.
- The return value is zero.

The same control-flow result applies to any other nonzero 32-bit field value.

### Dependency result 2

Given a captured base of 4096, an initially zero field, and dependency result 2:

- The candidate is 4108.
- The 32-bit field beginning at address 4100 receives 4108.
- The alternative global is not evaluated on this path.
- The return value is zero.

No access to the contents at address 4108 is required by this body.

### Dependency result 3 with a nonzero alternative

Given an initially zero field, dependency result 3, and an alternative-global value of 8192 after the call:

- The field at captured base plus 4 receives 8192.
- The return value is zero.

### Dependency result 3 with a zero alternative

Given an initially zero field, dependency result 3, and a post-call alternative value of zero:

- This function performs a 32-bit assignment of zero to the field.
- The return value is zero.
- A later invocation that again observes zero reaches the dependency-call path.

This illustrates why return zero does not guarantee a nonzero initialized field.

### Dependency result 4

Given an initially zero field and dependency result 4:

- The alternative global is evaluated in the supplied representation.
- This function performs no field assignment.
- The return value is zero.

If the dependency leaves the field unchanged, the field remains zero. Without that condition, its final value is unspecified by this function’s direct behavior alone.

### Unsupported dependency result

Given an initially zero field and dependency result 5:

- The alternative global is evaluated in the supplied representation.
- This function performs no field assignment.
- The return value is 19.

The same rule applies to every dependency result other than 2, 3, and 4.

### Primary global changed by the dependency

Given that the function captures object A, observes a zero field, and then the dependency changes the primary global to object B before returning 2:

- The subsequent assignment targets object A’s field.
- The stored value is object A’s captured base plus 12.
- No assignment to object B follows merely from the global change.

This example requires object A to remain writable after the dependency returns.

### Field changed by the dependency

Given that the initially inspected field is zero, the dependency changes that field to a nonzero value, and the dependency returns 3:

- This function does not recheck the field.
- It writes the post-call alternative-global value to the captured object’s field.
- The dependency’s field value is overwritten if it occupies that same field.

## Uncertainties

The following remain documented_only:

- The application meaning of return values zero and 19.
- The application meaning of dependency results 2, 3, and 4.
- Whether the field and alternative value are genuine pointers, encoded addresses, handles, or another representation.
- Object size, byte order, alignment, and lifetime rules.
- The target’s exact address-overflow semantics.
- Whether global evaluations correspond to distinct physical reads.
- Volatile, device-memory, aliasing, or concurrent-access properties.
- Dependency side effects, ownership behavior, and exceptional or non-returning behavior.
- Whether external serialization makes the operation safe for concurrent callers.

The apparent intent is to select an embedded address for result 2 and an externally supplied value for result 3. That interpretation is not necessary to establish the branch, store, and return behavior above.

## Evidence

This contract is based solely on static inspection of the supplied function representation and the supplied opaque dependency mapping.

Established static evidence includes the zero-field guard, the single conditional dependency call, exact comparisons against 2, 3, and 4, byte offsets 4 and 12, the conditional 32-bit assignment, and return values zero and 19.

The representation also distinguishes the pre-call captured base from the post-call alternative-global evaluation. Those observations support the state-change examples without assuming that the dependency preserves globals or memory.

No runtime tests have been performed. The acceptance examples are derived expectations subject to their stated preconditions, not observations of executable behavior.
