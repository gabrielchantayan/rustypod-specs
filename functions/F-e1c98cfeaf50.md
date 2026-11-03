<!-- Trusted provenance: F-e1c98cfeaf50; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e1c98cfeaf50

## Purpose

F-e1c98cfeaf50 repeatedly invokes an operation on a supplied object until one of two stopping conditions is satisfied. A third input selects the stopping condition:

- When nonzero, the function queries the object and continues while the query result minus that input is positive under the applicable signed arithmetic.
- When zero, the function examines an associated object and continues until that object is absent or its queried value equals the second input.

A four-byte argument distinguishes the first operation invocation from subsequent invocations within the same call.

These are established control-flow properties. The business meaning of the queries, the repeated operation, and the first-versus-subsequent argument is not established. Terms such as “remove,” “advance,” “drain,” or “release” would imply behavior not supported by the supplied evidence.

## Interface

### Inputs

The recovered interface has three inputs:

| Input | Recovered representation | Behavioral role |
|---|---|---|
| Primary object | Pointer to integer-addressable storage | Supplies an indirect dispatch table and, in zero-selector mode, an associated-object pointer. |
| Comparison value | Integer | Forwarded to primary-object callbacks and used as the equality target in zero-selector mode. |
| Mode/threshold value | Integer | Selects the mode. Its nonzero value also participates in the threshold comparison. |

The source representation uses integer-sized fields for addresses and a four-byte operation argument. A 32-bit integer and pointer reconstruction is strongly indicated by this layout, but the actual ABI and executable instruction widths are not independently available. Numeric examples below use the recovered 32-bit signed integer model.

Neither integer input is validated. Negative values are not rejected.

### Output

The function has no explicit return value. Normal completion communicates no success indicator, operation count, or final query value.

The function may invoke callbacks zero or more times. Its externally visible effects consist of those calls, reads required for dispatch and stopping decisions, and any effects performed by the callbacks.

### Object layout requirements

The function accesses the primary object as follows:

- Its first integer-sized field supplies a dispatch-table address.
- The dispatch entry at byte offset `0x94` supplies the nonzero-mode query callback.
- The dispatch entry at byte offset `0x90` supplies the repeated-operation callback.
- Its second integer-sized field supplies the associated-object pointer used only in zero-selector mode.

For an associated object that is present:

- Its first integer-sized field supplies another dispatch-table address.
- The entry at byte offset `0x68` supplies the associated-object query callback.

These are accessed locations, not a complete object schema. No additional fields, allocation sizes, or class relationships are established.

### Preconditions

The caller must provide memory and callable entries valid for every access reached during execution. In particular:

- The primary-object pointer is not checked for null.
- Relevant dispatch tables and callback entries are not checked for null.
- A non-null associated-object pointer is assumed usable for the required dispatch lookup.
- Callback-driven state changes must leave subsequent accesses valid if execution continues.

The function does not acquire ownership of either object. It contains no explicit allocation, release, retain, or reference-count operation. Callback ownership effects remain unknown.

## Behavior

### Shared operation-call convention

Both modes use the same repeated-operation callback, identified below as F-000000000002.

Each invocation receives four explicit arguments, in this order:

1. The primary-object pointer.
2. A four-byte first-versus-subsequent value.
3. Integer zero.
4. The comparison value.

The first-versus-subsequent value starts at zero for every invocation of F-e1c98cfeaf50. After the first operation callback returns normally, it becomes one. It remains one for all later operation calls within that invocation.

Consequently:

- No operation call means that this value is never supplied to a callback.
- Exactly one operation call receives zero.
- Any additional operation calls receive one.
- Separate invocations each start again at zero.

The flag changes after the callback returns, not before it is called. Its meaning beyond this sequencing is unknown.

The operation callback’s return value, if any, is not used.

### Nonzero mode/threshold value

When the third input is nonzero, the function uses the primary-object query callback, F-000000000001.

The observable sequence is:

1. Read the primary object’s dispatch-table reference and resolve the query entry at offset `0x94`.
2. Invoke that query with the primary-object pointer and comparison value as explicit arguments.
3. Evaluate whether the query result minus the mode/threshold value is positive.
4. If the predicate is false, return normally.
5. If the predicate is true, resolve the primary object’s operation entry at offset `0x90` and invoke it using the shared convention.
6. After that operation returns, set the first-versus-subsequent value to one and repeat the query.

The query therefore occurs before any operation call and again after every normally returning operation call.

The associated-object field is not consulted by this function in this mode. A null associated-object field does not itself cause an early return here.

The comparison value is passed unchanged to both the query and the operation. The mode/threshold value is not passed as an explicit callback argument.

#### Arithmetic boundary

The evidenced predicate is a signed subtraction followed by a positive-result test. It is not unconditionally specified as a mathematical greater-than comparison.

When the subtraction is representable in the applicable signed integer width, the function continues exactly when the query result is greater than the nonzero threshold. Equality stops execution.

When subtraction would overflow a 32-bit signed integer, the supplied representation does not establish a reliable outcome. Source-language overflow rules and machine-level wrapping behavior are not interchangeable. Overflow cases are therefore documented_only rather than assigned an assumed result.

### Zero mode/threshold value

When the third input is zero, the function instead uses the associated-object query callback, F-000000000003.

The observable sequence is:

1. Read the primary object’s second integer-sized field as an associated-object pointer.
2. If it is null, return immediately.
3. Resolve that associated object’s query entry at offset `0x68`.
4. Invoke the associated-object query.
5. Compare its integer result with the comparison value.
6. If equal, return normally.
7. If unequal, resolve the primary object’s operation entry at offset `0x90` and invoke it using the shared convention.
8. After that operation returns, set the first-versus-subsequent value to one and repeat from the associated-pointer read.

The associated-object query is shown with no explicit arguments. The evidence does not justify adding an explicit associated-object receiver, primary-object pointer, or comparison value to that call. Whether an implicit receiver or another ABI mechanism is involved is unknown.

The associated pointer is checked again after every operation. The function does not establish a stable associated-object identity across iterations. If an operation changes the associated pointer, the next iteration uses the subsequently read value. If it becomes null, the function returns without making another associated-object query.

Only equality matters in this mode. A query result below or above the comparison value produces the same continuation decision.

### Callback-sensitive dispatch

Callback entries are resolved at the places where they are used rather than visibly cached for the entire invocation.

In particular, the operation entry is obtained after the preceding query returns. A query that changes reachable dispatch state can therefore affect which operation callback is subsequently selected, provided the changed state remains valid.

Likewise, an operation can affect the object, associated pointer, dispatch tables, or state later queried. The function makes no visible attempt to preserve or restore such state.

### Termination and failure

There is no iteration limit or progress check.

Normal termination requires one of the following:

- Nonzero mode: the subtraction-and-positive predicate becomes false.
- Zero mode: the associated pointer is null.
- Zero mode: the associated-object query result equals the comparison value.

If callbacks return normally but never cause an applicable stopping condition, execution can continue indefinitely.

Invalid memory, invalid dispatch targets, callback exceptions, nonlocal exits, and callback failures have no visible local handling. The evidence does not establish a portable error result or recovery behavior for them.

## Side effects

The function itself performs no explicit writes through the supplied object pointers. Its directly maintained state is the first-versus-subsequent operation argument.

That does not make the function observationally read-only:

- Queries may mutate state.
- Operations may mutate state.
- Any callback may affect globals or other reachable objects.
- Callback allocation, retention, release, and ownership transfer are unknown.
- Callback reentrancy and synchronization behavior are unknown.

No callback is assumed to preserve global state, object layout, dispatch-table identity, or memory lifetime. Continued execution nevertheless requires the memory and callback targets subsequently accessed to remain valid.

No locks, atomic operations, or explicit synchronization are shown. Concurrent mutation is not established as supported.

## Dependencies

The following opaque IDs are document-local labels for indirect dependency roles. They do not establish verified identities for the functions ultimately reached, nor prove that the roles always resolve to distinct functions.

| Opaque ID | Resolution and explicit call interface | Used result |
|---|---|---|
| F-000000000001 | Primary dispatch entry at `0x94`; receives primary object and comparison value | Integer result used in the nonzero-mode subtraction predicate |
| F-000000000002 | Primary dispatch entry at `0x90`; receives primary object, four-byte first/subsequent value, zero, and comparison value | No result used |
| F-000000000003 | Associated-object dispatch entry at `0x68`; no explicit arguments shown | Integer result compared for equality with the comparison value |

The only verified function identity supplied for this contract is F-e1c98cfeaf50 itself.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. They assume valid dispatch state, normally returning callbacks, and no unmentioned state changes.

### Nonzero threshold already satisfied

Given a threshold of `5` and an initial primary query result of `5`:

- F-000000000001 is invoked once.
- The subtraction yields zero.
- F-000000000002 is not invoked.
- The function returns normally.

An initial query result of `4` also stops without an operation because the representable difference is negative.

### Multiple operations in nonzero mode

Given a threshold of `5` and successive primary query results of `8`, `6`, and `5`:

- Query and operation calls alternate until the final query.
- Two operation calls occur.
- Their first-versus-subsequent arguments are respectively zero and one.
- Both receive zero as their third explicit argument and the unchanged comparison value as their fourth.
- The final query causes return without another operation.

### Negative threshold

Given a threshold of `-2` and query results of `-1`, then `-2`:

- The first representable difference is positive, so one operation occurs.
- The second difference is zero, so execution stops.

No positivity requirement applies to the threshold itself.

### Null association in zero mode

Given a zero third input and an initially null associated pointer:

- No associated-object query occurs.
- No operation occurs.
- The primary nonzero-mode query is not used.
- The function returns normally.

### Equality on the first associated query

Given a zero third input, a valid associated object, and its query result equal to the comparison value:

- One associated-object query occurs.
- No operation occurs.
- The function returns normally.

### Operation changes the association

Given zero mode and an initial associated query result unequal to the comparison value, suppose the first operation sets the associated pointer to null.

The function performs one operation with first-versus-subsequent value zero. On the next iteration it observes the null pointer and returns without another associated-object query.

### No progress

If zero mode repeatedly observes a non-null association and an unequal query result, and every operation returns without changing those conditions, no normal termination is established.

### Overflow-sensitive subtraction

A 32-bit query result of `2,147,483,647` and threshold of `-1` require an unrepresentable signed subtraction result. Whether an operation occurs is documented_only; this contract does not infer wrapping behavior.

## Uncertainties

**Established static evidence:** branch selection, accessed field and dispatch offsets, explicit callback arguments, operation-flag sequencing, stopping predicates, and absence of explicit local recovery or object writes.

**Inferred intent:** the repeated operation probably changes state relevant to a later stopping query. No progress guarantee or specific operation semantics follows from that inference.

**Documented_only unknowns:**

- Exact ABI, calling conventions, and independently verified integer and pointer widths.
- Overflow behavior of the subtraction in the executable.
- Implicit arguments, if any, for the associated-object query.
- Callback identities, contracts, ownership effects, and global-state effects.
- Whether callbacks can safely replace or destroy objects or dispatch tables.
- Thread safety, reentrancy, and exceptional control flow.
- Business interpretation of the comparison value, threshold, and operation arguments.

## Evidence

This contract is based solely on static examination of the supplied recovered function representation. No implementation access, executable inspection, runtime tests, or smoke runs were performed.

The representation supports the described call ordering and control flow. It does not establish callback implementations, complete type layouts, or runtime behavior for arithmetic and ABI details left uncertain above.
