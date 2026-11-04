<!-- Trusted provenance: F-5e0fa7ff4b28; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5e0fa7ff4b28

## Purpose

F-5e0fa7ff4b28 recognizes four selector values and dispatches operations through one of two dynamically obtained objects. Three selectors use the first object and then invoke a completion dependency. The fourth uses the second object and returns without invoking that dependency.

The established behavior is dispatch and sequencing. The meanings of the selectors, objects, data values, and completion operation are unknown. In particular, a return value of one establishes that a recognized branch reached its normal return; it does not establish that any dispatched operation succeeded.

This contract distinguishes:

- **Established static evidence:** behavior directly represented by the supplied source.
- **Inferred intent:** interpretation consistent with that behavior but not independently established.
- **Unknown / documented_only:** information not supplied and not safe to assume.

No runtime tests have been performed.

## Interface

### Inputs

The function accepts three inputs:

| Input | Represented width | Established use |
|---|---:|---|
| First input | 32 bits | Not used by the visible function body. |
| Selector | 32-bit integer | Compared for exact equality with four constants. |
| Payload | 32 bits | Forwarded unchanged to selected dependencies. |

The recognized selector values are:

| Selector | Decimal | Dispatch route |
|---|---:|---|
| `0x9301` | 37633 | First object, two indirect calls, then completion |
| `0x9302` | 37634 | First object, one indirect call, then completion |
| `0x9303` | 37635 | First object, one indirect call, then completion |
| `0x9306` | 37638 | Second object, one indirect call, no completion |

All other selector values take the unrecognized-selector route.

The payload is an opaque 32-bit value at this interface. The visible body does not dereference it, validate it, transform it, or perform arithmetic on it. Some dependencies may interpret it as a pointer, handle, identifier, or scalar; none of those interpretations is established.

The first input has no visible effect on dispatch, dependency arguments, or the returned result. This does not establish any intended purpose for it.

### Output

The result is represented as a 32-bit value:

- **One:** a recognized route completed normally.
- **Zero:** an unrecognized selector reached the explicit unrecognized-selector return.

Dependency return values do not determine this result, except that the initial context and object-acquisition results are used to construct subsequent calls.

A dependency that fails to return normally can prevent this function from reaching either explicit result. No behavior for such failures is established.

### Object layout used by dispatch

Each dispatch object is treated as a pointer to storage whose leading field identifies a method table. The function obtains an indirect-call target from byte offset `0x54`, or 84 bytes, within that table.

The represented layout is consistent with a 32-bit environment: the source uses an integer-sized leading field to locate a table containing callable addresses. The exact machine ABI, pointer validity rules, table extent, and callable signatures are not independently established.

For a route that dispatches, the relevant object must support:

1. Reading its leading table reference.
2. Reading a callable target at table byte offset 84.
3. Calling that target with the route-specific argument list.

There is no visible check that these conditions hold.

## Behavior

### Shared acquisition sequence

Every selector, including an unrecognized selector, first triggers the same three dependency calls in this order:

1. Invoke F-9d0ecc5f6adf with no explicit arguments.
2. Invoke F-682adfd5a561 with the result of the first call and the constant `0x80`, decimal 128. Retain its result as the **primary object**.
3. Invoke F-6a21f4f07f67 with no explicit arguments. Retain its result as the **secondary object**.

Only after these calls does the selector-specific route proceed.

The second acquisition is not skipped when only the primary object will be dispatched. The primary acquisition is not skipped when only the secondary object will be dispatched. Both acquisitions also occur for an unrecognized selector.

The constant 128 is an argument, not established evidence of an allocation size, object type, capacity, or byte count.

### Opaque data roles

Six source-visible global data references supply dispatch or completion arguments. This contract names them by role rather than assigning unsupported meanings:

| Role | Use |
|---|---|
| **Prefix datum** | Second argument of the first dispatch for `0x9301` |
| **Primary-A datum** | Second argument of the final primary dispatch for `0x9301` |
| **Primary-B datum** | Second argument of the primary dispatch for `0x9302` |
| **Primary-C datum** | Second argument of the primary dispatch for `0x9303` |
| **Secondary datum** | Second argument of the secondary dispatch for `0x9306` |
| **Completion datum** | Second argument supplied to F-2484a607f41b |

These are distinct referenced data locations. Their contents, types, equality or inequality, mutability, and semantic meanings are unknown. Distinct locations need not contain distinct values.

The source does not establish a full snapshot of these values at function entry. A value used after a dependency call must not be assumed to equal its value before that call.

### Selector `0x9301`

This route performs two primary-object dispatches:

1. Obtain the primary object's method-table target at byte offset 84.
2. Invoke that target with three explicit arguments, in this order:
   - The primary object.
   - The prefix datum.
   - The payload.
3. After that call returns, obtain the primary object's method-table target at byte offset 84 again.
4. Read the primary-A datum for the next dispatch.
5. Invoke the newly obtained target with two explicit arguments:
   - The primary object.
   - The primary-A datum.
6. Invoke F-2484a607f41b with:
   - The payload.
   - The completion datum.
7. Return one.

The second target lookup is significant. The first dispatch may have changed the primary object's table reference or the target stored at offset 84. The second dispatch is not required to use the target that handled the first dispatch.

The primary object value itself is retained from acquisition; there is no second call to acquire a replacement primary object.

The first dispatch receives the payload explicitly. The second dispatch does not. A specification that adds the payload as a third explicit argument to the second dispatch would change the visible call shape.

### Selector `0x9302`

This route:

1. Obtains the primary object's target at method-table byte offset 84.
2. Reads the primary-B datum.
3. Calls that target with the primary object and the primary-B datum.
4. Invokes F-2484a607f41b with the payload and the completion datum.
5. Returns one.

There is no prefix dispatch. The payload reaches the completion dependency, but is not explicitly supplied to the primary-object dispatch.

### Selector `0x9303`

This route has the same sequencing as `0x9302`, except that it uses the primary-C datum:

1. Obtain the primary object's target at method-table byte offset 84.
2. Read the primary-C datum.
3. Call that target with the primary object and that datum.
4. Invoke F-2484a607f41b with the payload and the completion datum.
5. Return one.

### Selector `0x9306`

This route:

1. Obtains the secondary object's target at method-table byte offset 84.
2. Calls that target with three explicit arguments:
   - The secondary object.
   - The secondary datum.
   - The payload.
3. Returns one.

F-2484a607f41b is not called on this route. There is no visible primary-object dispatch, although the primary object was acquired earlier.

### Every other selector

After completing the shared acquisition sequence, the function returns zero.

On this route, the visible function body:

- Does not read either object's method table.
- Does not perform an indirect dispatch.
- Does not read a route datum or the completion datum.
- Does not call F-2484a607f41b.

The acquisition dependencies may nevertheless have side effects.

### Ordering limits

The source establishes dependency-call order and the explicit target refresh between the two `0x9301` dispatches. It also places each completion call after the corresponding final primary dispatch.

It does not independently establish instruction-level ordering among argument evaluation reads within a single call expression. No stronger memory-ordering, atomicity, or concurrency guarantee is justified.

## Side effects

### Direct effects

The visible body performs reads needed to obtain dispatch targets and argument data. It contains no explicit write to global data, object fields, or payload-referenced storage.

Its observable external actions are dependency invocations and indirect method calls. Those calls may produce state changes not described by the supplied source.

### Ownership and lifetime

No direct allocation, deallocation, retain, release, or ownership transfer is established.

In particular:

- F-682adfd5a561 returning an object does not establish that the caller owns it.
- F-6a21f4f07f67 returning an object does not establish borrowing or ownership.
- Passing the payload to a dependency does not establish whether that dependency retains, consumes, mutates, or releases anything associated with it.
- The absence of a visible release operation does not prove a leak or prove that no release occurs inside a dependency.

The caller and dependencies must provide whatever lifetimes the actual dispatch protocol requires. Those lifetimes remain documented_only.

### Validation and errors

There are no visible null checks, object-validity checks, callable-target checks, payload checks, or dependency-status checks.

An invalid object or method-table target on a selected dispatch route can prevent normal completion; the precise failure mode is unknown. A null-looking acquisition result is not directly dereferenced on the unrecognized-selector route, but acquisition dependencies themselves may still fail or produce effects.

There is no visible rollback or compensating action if an earlier dependency changes state and a later operation fails.

## Dependencies

| Opaque dependency | Visible contract |
|---|---|
| F-9d0ecc5f6adf | Called first, without explicit arguments. Its 32-bit result is forwarded to F-682adfd5a561. |
| F-682adfd5a561 | Called second with the preceding result and 128. Its result supplies the primary object. |
| F-6a21f4f07f67 | Called third, without explicit arguments. Its result supplies the secondary object. |
| F-2484a607f41b | Called after the final primary dispatch for `0x9301`, `0x9302`, and `0x9303`, with payload first and completion datum second. Not called for other selectors. |
| Primary-object indirect target | Resolved through table byte offset 84. Receives three explicit arguments for the prefix dispatch and two for the subsequent or standalone primary dispatch. |
| Secondary-object indirect target | Resolved through table byte offset 84. Receives the secondary object, secondary datum, and payload. |

The indirect targets have no supplied opaque function IDs. Their identity is determined at runtime rather than established by the supplied dependency map.

No dependency is assumed to preserve global data, preserve object state, retain object validity, or follow a particular ownership policy. Results from the indirect calls and completion dependency are not visibly inspected.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Each normal-result example assumes all invoked dependencies and relevant memory accesses complete normally.

### Unrecognized selector

Given selector `0x9304`:

- The three acquisition dependencies execute in their specified order.
- No indirect target is read or called by this function.
- No completion call occurs.
- The result is zero.

Returning zero before the acquisition calls would not conform.

### Primary route with completion

Given selector `0x9302` and an arbitrary 32-bit payload:

- The primary dispatch receives exactly the primary object and primary-B datum as explicit arguments.
- F-2484a607f41b subsequently receives that unchanged payload and the completion datum.
- The result is one, irrespective of any ordinary return value produced by either call.

Substituting primary-C datum, forwarding the payload into the primary dispatch, or omitting completion would not conform.

### Target mutation during the prefix dispatch

Given selector `0x9301`, suppose the first primary dispatch changes the primary object's table reference or its offset-84 target and returns normally.

The second dispatch uses the target obtained by the subsequent lookup. Reusing a target cached before the first dispatch would not conform.

This example does not require the first dispatch to mutate anything; it defines the consequence if it does.

### Global mutation before completion

Given selector `0x9303`, suppose the primary dispatch changes the completion datum and returns normally.

The later completion call obtains its argument from that datum after the dispatch. Treating the completion datum as an entry-time snapshot would not conform.

### Secondary-only dispatch

Given selector `0x9306`:

- Both objects are acquired.
- Only the secondary object is dispatched.
- That dispatch receives the payload explicitly.
- No completion call occurs.
- The result is one.

Adding completion to make this route resemble the three primary routes would change behavior.

### Ignored first input

For otherwise identical invocations, changing only the first input does not change the visible function's selections or explicit argument construction. This criterion concerns the function body; it does not assert deterministic behavior from stateful dependencies.

## Uncertainties

The following remain **documented_only**:

- Exact calling convention and legal signatures of the indirect targets.
- Whether the two-argument and three-argument calls resolve to different implementations.
- Concrete values, widths beyond the represented argument flow, and meanings of the six data references.
- Semantic purpose of selector values and the constant 128.
- Whether any object acquisition allocates, caches, retains, or releases resources.
- Payload interpretation and lifetime requirements.
- Exception, fault, nonlocal-exit, and dependency-failure behavior.
- Thread safety, reentrancy, synchronization, and concurrent mutation behavior.
- Whether any dependency invalidates objects or changes data used by later calls.

**Inferred intent:** the function resembles a selector-driven handler with a shared completion step for three related operations. That interpretation is not a basis for naming domain actions, adding validation, or treating one as a success status from downstream operations.

## Evidence

The supplied static source establishes:

- Three unconditional dependency calls before selector-specific behavior.
- Exact equality checks against four selector constants.
- Dispatch through method-table byte offset 84.
- Two primary dispatches for `0x9301`, including a fresh target lookup between them.
- One primary dispatch for each of `0x9302` and `0x9303`.
- One secondary dispatch and an immediate normal return for `0x9306`.
- Completion calls only on the three primary routes.
- Explicit normal results of one or zero.
- No visible use of the first input, no direct memory writes, and no explicit ownership operation.

The supplied source does not establish dependency internals or runtime outcomes. No execution, runtime observation, or tests support claims beyond this static evidence.
