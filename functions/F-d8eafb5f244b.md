<!-- Trusted provenance: F-d8eafb5f244b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d8eafb5f244b

## Purpose

F-d8eafb5f244b dispatches two scalar inputs to one of three outcomes:

- Return a literal zero without invoking a listed dependency.
- Invoke an ordered pair of dependencies and return the second dependency’s result.
- Invoke a single dependency with one of two fixed arguments and return its result.

The first input has precedence. Its values zero and one determine the outcome without consulting the second input for dispatch. Other first-input values permit second-input-based selection.

The established behavior is dispatch, dependency invocation order, and result forwarding. Application-level meanings such as initialization, validation, success, or failure are not established.

This contract distinguishes three evidence levels:

- **Established static behavior:** comparisons, branch precedence, explicit arguments, call/return ordering, and result handling visible in the supplied representation.
- **Inferred intent:** the inputs appear to select an operation and a dispatch category. Those interpretations are not required for conformance.
- **Unknown behavior:** dependency internals, external state effects, error semantics, environmental prerequisites, and abnormal completion.

All dependency-return statements below are conditional on normal return. Call ordering does not establish completion or ordering of deferred or asynchronous effects initiated by a dependency.

## Interface

### Inputs

The function receives two integer scalar inputs by value:

| Position | Contract name | Visible use |
|---|---|---|
| First | Selector | Compared for equality with zero, one, and a fixed set of other values. |
| Second | Mode | Used for dispatch only when the selector is neither zero nor one. |

The supplied representation is consistent with 32-bit integer inputs. This contract uses that working width, while retaining the original declarations, calling convention, and machine-level argument-extension rules as **documented_only**.

The function performs no visible input arithmetic, ordering comparisons, shifts, division, or pointer interpretation. Dispatch depends on equality with the specified constants. Interpreting another 32-bit pattern as signed or unsigned does not change whether it equals one of those constants.

There is no visible requirement that either input be nonnegative or belong to an enumeration. Unlisted values have defined local dispatch behavior. Any additional prerequisites imposed by a selected dependency remain unknown.

### Output

The function returns an opaque four-byte result, described here as a 32-bit value. Its source is either:

- A literal zero on a nondelegating path.
- The unchanged four-byte result of the final dependency on a delegating path.

The result is not visibly interpreted as a Boolean, status code, count, handle, pointer representation, or flag set. A zero result alone cannot distinguish a nondelegating path from a dependency call that returned zero.

### Memory, layout, and ownership

Neither input is visibly dereferenced. The exposed interface contains no caller-supplied buffer, output pointer, structure, array, length field, or variable-sized object. No aggregate layout or alignment requirement is established.

There is no explicit local allocation, release, memory copy, ownership transfer, or retained reference. This does not establish corresponding restrictions on dependencies. Their memory access and ownership behavior remain **documented_only**.

## Behavior

### Selector precedence

The selector determines whether mode-based dispatch is reached.

**Selector equals one**

The function invokes F-48dea5cf0ac4 with no explicit arguments. If that invocation returns normally, the function invokes F-21b925c3866f with the integer value one. If the second invocation returns normally, its four-byte result becomes the function’s result without visible modification.

The mode has no dispatch effect on this path, including when it is outside zero through three. F-cb62ad9f7f37 is not invoked by this path.

No result from F-48dea5cf0ac4 is used in a visible comparison or in constructing the returned value. Normal return from that dependency is sufficient for execution to proceed to F-21b925c3866f; the caller performs no visible success check.

**Selector equals zero**

The function returns a literal zero without invoking any listed dependency. The mode has no dispatch effect.

**Selector is neither zero nor one**

The function applies the mode-based rules below.

This precedence is mandatory. Mode filtering must neither reject the selector-one path nor redirect it to F-cb62ad9f7f37. Likewise, no mode may cause a dependency invocation when the selector is zero.

### Mode-based dispatch

The following table applies only when the selector is neither zero nor one:

| Mode | Additional condition | Required local action |
|---|---|---|
| Zero | None | Invoke F-cb62ad9f7f37 with one; return its result unchanged. |
| One | Selector is three, four, five, eight, or ten | Invoke F-cb62ad9f7f37 with one; return its result unchanged. |
| One | Selector is outside that exact set | Return literal zero without invoking a listed dependency. |
| Two | None | Invoke F-cb62ad9f7f37 with two; return its result unchanged. |
| Three | None | Invoke F-cb62ad9f7f37 with two; return its result unchanged. |
| Any other value | None | Return literal zero without invoking a listed dependency. |

The accepted selector set for mode one is exactly three, four, five, eight, and ten. It is not a continuous range: six, seven, and nine are rejected.

Modes two and three produce the same local dependency invocation and argument. This does not establish that they have the same meaning elsewhere.

Neither original input is explicitly forwarded as a dependency argument. F-cb62ad9f7f37 receives the selected constant one or two. F-21b925c3866f receives one.

### Result handling and errors

On a normally completing delegating path, the final dependency’s four-byte result is forwarded unchanged.

There is no visible:

- Boolean conversion or nonzero normalization.
- Filtering based on the result’s highest bit.
- Special handling of a dependency-returned zero.
- Error-code translation.
- Retry or fallback invocation.
- Cleanup invocation after the final dependency returns.

No local distinction between dependency success and failure is established. If a dependency reports an error through its ordinary four-byte return value, that value is forwarded on paths where its result is used. Whether any particular value represents an error is unknown.

### Integer boundaries

Only equality comparisons and small literal argument selections are visible. There is no input arithmetic and therefore no visible overflow, underflow, range clamping, or modular reduction.

Under the 32-bit working interpretation:

- A negative selector is eligible for mode-zero, mode-two, and mode-three dispatch unless it equals one of the distinguished selector values; negative values do not equal zero or one.
- A negative selector is rejected under mode one because it is outside the exact accepted set.
- A negative mode selects the unrecognized-mode outcome when mode-based dispatch is reached.
- Large, unlisted selectors behave like other unlisted selectors.
- Large modes outside zero through three select the unrecognized-mode outcome, unless selector precedence already determines the path.

These are local dispatch properties, not guarantees about dependency behavior.

## Side effects

### Explicit local effects

The supplied body shows dependency invocations and result production. It shows no explicit global read or write, object-field access, buffer modification, I/O operation, allocation, release, or synchronization operation.

A literal-zero path invokes none of the listed dependencies. That establishes the absence of those calls, not the absence of every possible machine-level effect.

The input comparisons establish dispatch precedence, but do not provide a memory-access trace for argument registers, stack storage, or compiler-generated bookkeeping. There are no visible application-memory reads or writes to order beyond effects that may occur inside dependencies.

### Ordered dependency invocations

For selector one, normal execution establishes this sequence:

1. Invocation of F-48dea5cf0ac4.
2. Normal return from F-48dea5cf0ac4.
3. Invocation of F-21b925c3866f with one.
4. Normal return from F-21b925c3866f.
5. Return of that second dependency’s result to the caller.

This is a call/return ordering guarantee only. It does not guarantee that all effects attributable to F-48dea5cf0ac4 have completed before F-21b925c3866f begins.

For example, the first dependency might initiate deferred work or asynchronous activity that continues after it returns. The supplied body contains no visible wait, join, completion check, or other mechanism establishing when such activity finishes. Whether deferred effects occur before, during, or after the second invocation remains unknown. Nor is any particular visibility of shared state to the second dependency established.

Other delegating paths invoke F-cb62ad9f7f37 once during normal execution of this function. No additional listed dependency is invoked afterward.

### State, ownership, and abnormal completion

A dependency must not be assumed to preserve globals, leave resources unchanged, acquire or release ownership, or finish all work before returning. Separate dependency contracts would be required to establish those properties.

There is no visible local recovery or compensation mechanism. If an invoked dependency blocks indefinitely, terminates execution, transfers control elsewhere, or otherwise does not return normally, the evidence does not establish a subsequent invocation or normal return from this function.

Likewise, return from this function does not establish that dependency-initiated asynchronous activity has finished.

## Dependencies

| Opaque function ID | Invocation condition | Explicit arguments | Visible result handling |
|---|---|---|---|
| F-48dea5cf0ac4 | Selector equals one | None | No result is used by this caller. |
| F-21b925c3866f | Selector equals one, after normal return from F-48dea5cf0ac4 | Integer one | Four-byte result returned unchanged. |
| F-cb62ad9f7f37 | Eligible mode-based path with selector unequal to zero and one | Integer one or two | Four-byte result returned unchanged. |

“No explicit arguments” does not establish absence of implicit inputs. Dependencies may access global state, thread-local state, external resources, or other context.

Constant explicit arguments do not establish constant results. Purity, determinism, idempotence, synchronization, thread-safety, and reentrancy are not established. The dependency interfaces beyond the visible calls, including their complete argument declarations and internal ownership rules, remain **documented_only**.

## Acceptance examples

These examples are static consequences, not executed tests. Examples involving dependency results assume normal completion.

### Selector precedence

- **Selector one, mode ninety-nine:** Invoke F-48dea5cf0ac4, wait for that call to return normally, then invoke F-21b925c3866f with one. Return the second dependency’s result. Do not invoke F-cb62ad9f7f37.
- **Selector one, mode zero:** Use the same two-call path. Mode zero must not select F-cb62ad9f7f37.
- **Selector one, mode three:** Again use the two-call path, rather than the mode-three single-call path.
- **Selector zero, mode two:** Return literal zero without invoking any listed dependency.
- **Selector zero, negative mode:** Return literal zero without invoking any listed dependency.

### Mode-zero dispatch

- **Selector two, mode zero:** Invoke F-cb62ad9f7f37 once with one and return its result unchanged.
- **Selector negative one, mode zero:** Perform the same invocation. No visible positivity requirement excludes this selector.

### Exact mode-one membership

- **Selector three, four, five, eight, or ten; mode one:** Invoke F-cb62ad9f7f37 once with one.
- **Selector six, seven, or nine; mode one:** Return literal zero without invoking a listed dependency.
- **Selector two, mode one:** Return literal zero without a dependency invocation.
- **Selector negative one, mode one:** Return literal zero without a dependency invocation.

### Modes two, three, and unrecognized modes

- **Selector nine, mode two:** Invoke F-cb62ad9f7f37 once with two, despite selector nine being rejected under mode one.
- **Selector negative one, mode three:** Invoke F-cb62ad9f7f37 once with two.
- **Selector three, mode four:** Return literal zero without invoking a listed dependency.
- **Selector three, mode negative one:** Return literal zero without invoking a listed dependency.

### Result forwarding and effect boundaries

- **An eligible final dependency returns zero:** Return zero, while preserving the required dependency invocation trace.
- **An eligible final dependency returns a four-byte value with its highest bit set:** Return the same four-byte value without Boolean conversion or sign-based filtering.
- **F-48dea5cf0ac4 returns normally after initiating deferred work:** The next visible call must be F-21b925c3866f with one. This contract imposes no completion requirement on that deferred work before the second call.
- **F-48dea5cf0ac4 does not return normally:** The normal two-call completion guarantee does not apply. No recovery behavior is established.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the selector, mode, fixed dependency arguments, and result.
- Original declarations and ABI details beyond the supplied scalar representation.
- Dependency memory accesses, global changes, I/O, resource use, and ownership behavior.
- Initialization requirements and environmental prerequisites.
- Dependency-specific error conventions and abnormal control flow.
- Whether dependencies initiate deferred or asynchronous activity.
- Completion, synchronization, and visibility of dependency effects.
- Thread-safety, reentrancy, determinism, and concurrent interaction.

The selector-one sequence may suggest preparation followed by another operation. That is inferred intent only. The established requirement is invocation after normal return, not preparation semantics, a successful first operation, or completion of all first-dependency effects.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent evidence of behavior.

The representation establishes selector-first precedence, the exact mode-one selector set, the constants passed to dependencies, the ordered selector-one calls, literal-zero return paths, and unchanged forwarding of the final dependency result.

It does not expose dependency implementations, asynchronous behavior, a shared-state model, original interface declarations, or runtime traces. In particular, normal return from the first dependency establishes when the second call may begin, not when all effects attributable to the first dependency have completed.

No runtime tests have been performed. All acceptance examples are conditional consequences of the static evidence.
