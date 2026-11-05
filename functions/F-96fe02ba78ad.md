<!-- Trusted provenance: F-96fe02ba78ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-96fe02ba78ad

## Purpose

F-96fe02ba78ad conditionally requests a change to a stored 32-bit state, selects one of two processing paths, publishes a path-dependent marker, and dispatches indirectly through an acquired object.

Two inputs independently control the behavior:

- The requested state controls whether a state-change dependency is called and whether the function directly updates the state field.
- The path selector controls the preparation sequence, the marker value assigned through the publication destination, and the numeric argument supplied to the indirect target.

A nonzero state-change result suppresses the direct state-field write but does not stop path processing. An already-matching state or an unsupported requested state likewise does not prevent publication and dispatch.

The marker values are established as `1` and `4`. **The destination store width and any assignment conversion are not established.** A character-typed intermediate does not establish the type or width of the destination accessed through the publication expression.

Application meanings such as activation, shutdown, or cleanup are not established by the supplied body.

## Interface

### Inputs

| Input | Recovered representation | Established use |
|---|---|---|
| State-bearing address | Address carried in a 32-bit integer parameter | Locates a 32-bit state field at byte offset `0xD8`, decimal 216 |
| Path selector | 32-bit integer | Exactly `1` selects the short path; every other value selects the preparation path |
| Requested state | 32-bit integer | Compared with the initial state; only `1` and `2` can trigger the state-change dependency |

The function reads the state field before any explicit dependency call, regardless of either control input. A direct write to that field is conditional.

Normal execution requires valid storage for every reached access. The supplied body does not establish checked address arithmetic, a wraparound policy, alignment requirements, or byte order. The offset identifies the intended field location; it does not define behavior when adding the offset to the address-bearing input exceeds the usable address range.

Negative requested states are unsupported, just like other values outside `1` and `2`. Negative path selectors take the preparation path because they are not equal to `1`.

### Shared source and destination expressions

Two address-bearing expressions have distinct roles:

- The **source expression** selects storage whose contents become the argument to F-be4e5c73e101.
- The **publication expression** selects storage through which the path-dependent marker is assigned.

Neither expression’s declaration is supplied. It is unresolved whether either represents a mutable pointer-valued global, a fixed recovered address, or an equivalent representation.

Consequently:

- A separate runtime read of pointer-valued storage cannot be asserted unconditionally.
- The source argument is the contents obtained by dereferencing the selected source address, not simply the address itself.
- Publication assigns through the selected destination; it is not established as an assignment to the address-bearing expression itself.
- The source dereference width and the publication destination’s type and store width remain unresolved.

Neither selected address is established as an entry-time snapshot. Their evaluation belongs at the access points described below, after any preceding dependency calls.

### Output and completion

The recovered interface has no explicit return value. Observable behavior consists of memory accesses, calls, the conditional state-field write, marker publication, and indirect dispatch.

Normal completion requires all reached dependencies and the indirect target to return normally. There is no visible error return, retry, or local recovery mechanism. The indirect target’s result is unused.

## Behavior

### Initial state observation and optional update

The first explicit memory access in the body reads the four-byte state field at offset `0xD8` from the state-bearing address.

If that observed value equals the requested state, the function skips both the state-change dependency and the direct state-field write.

If they differ, the requested state determines the next action:

| Requested state | Action |
|---|---|
| `1` | Call F-b6a73313bbe9 with the 32-bit value `61`, hexadecimal `0x3D` |
| `2` | Call F-b6a73313bbe9 with the 32-bit value `11`, hexadecimal `0x0B` |
| Any other value | Skip the state-change dependency and direct state-field write |

For a supported request, the dependency’s 32-bit result is compared with zero:

- A zero result causes a four-byte write of the requested state to the state field.
- Any nonzero result suppresses that direct write.

Positive and negative nonzero results have the same effect. No sign-based success condition is visible.

The function does not reread the state field after the dependency returns. If the dependency modifies the field and returns zero, the subsequent direct write overwrites that modification with the requested state. If the dependency returns nonzero, this function does not restore the originally observed state.

The result controls only the direct state-field write. Path selection follows in either case.

### Short path

A path selector exactly equal to `1` selects the short path.

This path:

- Does not dereference the preparation source.
- Skips all four preparation dependencies.
- Selects marker value `1`.
- Selects final numeric argument `0`.

The initial state read and any applicable state-change processing still precede this path.

### Preparation path

Every path-selector value other than `1` selects the preparation path.

The observable preparation sequence is:

1. Evaluate the source expression and dereference its selected address.
2. Call F-be4e5c73e101 with the obtained contents.
3. Call F-6622e2509ce3 with no explicit arguments and retain its 32-bit result.
4. Call F-4ffd7bc0e22a with no explicit arguments and retain its 32-bit result.
5. Call F-b8522592e971 with the result of F-4ffd7bc0e22a first and the result of F-6622e2509ce3 second.

The source access occurs after the optional state-change dependency and after any resulting direct state-field write. It is not established as using a source address or source value captured before that stage.

If the source expression denotes mutable pointer-valued storage, a preceding dependency can affect which address it selects. A preceding modification of the selected source storage can likewise affect the value forwarded to F-be4e5c73e101.

The two producer results are four-byte quantities. Their application types are unknown. Production order differs from argument order: the first-produced value becomes the second argument to F-b8522592e971 and is retained across the second producer call.

The body does not inspect results from F-be4e5c73e101 or F-b8522592e971. Their normal return allows processing to continue without a visible success test.

After preparation, this path selects marker value `4` and final numeric argument `1`.

### Marker selection and publication

The reachable path outcomes are:

| Path selector | Marker value supplied to the publication assignment | Final numeric argument |
|---|---:|---:|
| Exactly `1` | `1` | `0` |
| Any other value | `4` | `1` |

The marker is held in a character-typed intermediate. The body compares that intermediate with `1` and `4` to select the final numeric argument. Although an additional return is shown for an unexpected intermediate value, ordinary execution of the supplied assignments produces only `1` or `4`; no third input-reachable outcome is established.

After numeric-argument selection, the function evaluates the publication expression and assigns the selected marker through it.

Publication occurs:

- After the optional state-change dependency and any resulting state-field write.
- On the preparation path, after all four preparation dependencies return.
- Before object acquisition.

The publication destination is not established as cached before those calls. If its expression denotes mutable pointer-valued storage, a preceding dependency can redirect the assignment.

**The established operation is an assignment of marker value `1` or `4` through the destination expression.** The missing destination declaration prevents determining:

- The destination’s pointee type.
- The width of the resulting store.
- Any conversion required by the assignment.
- The exact resulting memory representation or affected byte range.

In particular, the character type of the intermediate does not prove a one-byte destination store. No guarantee is established that bytes adjacent to the selected destination remain untouched by publication.

The final numeric argument is selected before publication and is not obtained by rereading the destination. A later modification of the published storage therefore does not, through any visible operation in this body, change that argument.

### Object acquisition and indirect dispatch

After publication, F-378e2d0a4ff6 is called with no explicit arguments.

Its result is treated as an object address. Under the recovered 32-bit layout, dispatch then requires:

- Reading the first four-byte word of the object as a dispatch-table address.
- Reading a callable address at byte offset `0x20`, decimal 32, from that table.
- Invoking that target with the object address first and the selected numeric argument, `0` or `1`, second.

With four-byte table entries, offset `0x20` denotes zero-based entry index eight, or the ninth entry when counted from one.

The body shows no null check or validation of the object, table, or target. Valid storage, a usable table, and a callable target are preconditions for normal execution of this stage.

The recovered body represents the indirect transfer as a call. Its result is unused. If it returns normally, there is no subsequent explicit state-field write or marker publication before return.

## Side effects

### Direct memory effects

The function explicitly performs:

- One initial four-byte state-field read.
- At most one four-byte state-field write.
- One source-data dereference on the preparation path, with unresolved access width.
- One publication assignment when execution reaches that stage, with unresolved destination store width and assignment conversion.
- The object and dispatch-table reads required for indirect dispatch.

Evaluation of the source and publication expressions occurs at their respective access points. Whether evaluating either expression also reads mutable pointer storage depends on declarations not supplied.

The direct state-field write requires all three conditions:

1. The initial state differs from the requested state.
2. The requested state is `1` or `2`.
3. F-b6a73313bbe9 returns zero.

These are operation-level guarantees, not unconditional final-memory guarantees. Dependencies may modify the same storage later. The source, publication destination, state field, address-bearing storage, object, and table are not established as disjoint.

Because publication width is unknown, its possible overlap with another region cannot be bounded to a single byte from this excerpt.

### Observable ordering

For normal sequential execution, the relevant order is:

1. Initial state-field read.
2. Optional state-change dependency.
3. Optional direct state-field write.
4. On the preparation path, source access and the four preparation calls.
5. Publication assignment.
6. Object acquisition.
7. Dispatch-structure reads.
8. Indirect invocation.

The ordering does not establish atomicity, synchronization, or interthread visibility. In particular, the term “publication” describes the assignment’s role in this sequence, not a synchronization guarantee.

### Ownership and abnormal completion

The body contains no explicit allocation, release, or ownership-transfer operation. This does not constrain dependencies to avoid those effects.

Ownership, retention, and lifetime rules for the forwarded source value, the two producer results, and the acquired object are unresolved. The lack of an explicit release after dispatch does not establish that release is unnecessary or performed elsewhere.

If a reached dependency does not return normally, subsequent operations are not guaranteed to occur. Invalid memory access and exceptional control transfer have no specified recovery behavior in the supplied body.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-b6a73313bbe9 | Receives `61` or `11`; its 32-bit zero/nonzero result controls the direct state-field write |
| F-be4e5c73e101 | Receives contents obtained by dereferencing the source expression on the preparation path; the source value’s width and type are unresolved |
| F-6622e2509ce3 | Produces the first retained 32-bit preparation value |
| F-4ffd7bc0e22a | Produces the second retained 32-bit preparation value |
| F-b8522592e971 | Receives the second-produced value first and the first-produced value second |
| F-378e2d0a4ff6 | Runs after publication and supplies the address used for object dispatch |

The indirect target has no resolved opaque identity. Its selection is established by the acquired object’s table and byte offset `0x20`.

No dependency is assumed to preserve globals, selected addresses, source contents, the state field, publication storage, or other shared memory. No ownership or release contract is inferred for any dependency.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. They assume valid reached accesses and normal return from invoked dependencies. Claims about later memory contents additionally require that subsequent operations neither overwrite nor alias the relevant storage.

### Matching state and short path

Given initial state `1`, requested state `1`, and path selector `1`:

- No state-change dependency or direct state-field write occurs.
- The preparation source is not dereferenced.
- All preparation dependencies are skipped.
- Marker value `1` is assigned through the publication destination, subject to its unresolved destination type and conversion.
- Object acquisition follows publication.
- The indirect target receives the acquired object address and numeric argument `0`.

This criterion does not prescribe publication width or adjacent-byte preservation.

### Successful supported request

Given initial state `1`, requested state `2`, and a zero result from F-b6a73313bbe9:

- The dependency receives `11`.
- After it returns, the function writes four-byte value `2` to the state field without rereading it.
- The path selector independently determines subsequent processing.

If the dependency modified the state field before returning zero, the direct write still occurs.

### Nonzero result does not stop dispatch

Given initial state `2`, requested state `1`, path selector `1`, and a nonzero dependency result:

- F-b6a73313bbe9 receives `61`.
- No direct state-field write follows.
- Marker value `1` is assigned through the publication destination.
- Object acquisition and dispatch with numeric argument `0` follow.

The state is not guaranteed to remain `2`, because dependencies or aliasing may affect it.

### Preparation order and argument placement

Given path selector `0`, suppose F-6622e2509ce3 returns `0x11223344` and F-4ffd7bc0e22a returns `0x55667788`.

After the source access and F-be4e5c73e101 call, the producers run in that order. F-b8522592e971 receives `0x55667788` first and `0x11223344` second.

Only after that dependency returns is marker value `4` assigned through the publication destination. Object acquisition follows, and final dispatch receives numeric argument `1`.

No exact publication byte sequence is required without the destination declaration.

### Conditional source redirection

Assume the source expression denotes mutable pointer-valued storage. On a supported, initially different requested state and a preparation-path selector, suppose F-b6a73313bbe9 changes that pointer from address A to address B.

If the source expression yields B at the later source access, the contents selected at B are forwarded to F-be4e5c73e101. An entry-time snapshot at A is not established.

This example does not assert that the missing declaration actually defines mutable pointer storage.

### Conditional publication redirection

Assume the publication expression denotes mutable pointer-valued storage. Suppose a preparation dependency changes it from address C to address D.

If the expression yields D when publication is reached, the marker assignment occurs through D with supplied value `4`. The assignment’s width, conversion, and affected byte range remain unresolved.

On the short path, a corresponding change by the optional state-change dependency can redirect the assignment of marker value `1`.

### Unsupported values

For requested state `3` or any negative requested state, the initial state read still occurs, but there is no state-change dependency or direct state-field write.

A path selector of `-1`, `0`, or `2` selects preparation, marker value `4`, and final numeric argument `1`.

## Uncertainties

The following remain **documented_only**:

- Application meanings of states, dependency constants, marker values, and dispatch arguments.
- Whether either shared address expression denotes mutable pointer storage or a fixed address.
- Source dereference width, value type, and argument interpretation.
- Publication destination type, store width, assignment conversion, exact representation, and affected byte range.
- Aliasing among all accessed regions and dependency-accessible memory.
- Dependency side effects, ownership, retention, release, error protocols, and reentrancy.
- Concrete identity and exact calling convention of the indirect target.
- Object and dispatch-table validity and lifetime guarantees.
- Byte order, alignment, address-arithmetic edge behavior, and concurrency properties.
- Behavior after invalid accesses or abnormal control transfer.

The layout is consistent with a 32-bit object and dispatch table. A particular object model or class hierarchy is inferred intent, not an established contract.

## Evidence

The supplied static body establishes the state comparison, supported requests, dependency constants, zero-controlled four-byte state write, preparation-call order, four-byte retained producer results, reversed argument placement, marker values, publication order, and dispatch offset.

It establishes a character-typed marker intermediate, but does not supply the publication destination’s declaration. Therefore, it does not establish a one-byte publication store or preservation of neighboring bytes.

The source dereference occurs after optional state-change processing. Publication occurs after path processing and before object acquisition. Missing declarations prevent resolving additional pointer-storage reads, source access width, or publication conversion and width.

Dependency semantics and application intent are not established. No runtime tests have been performed; the acceptance examples are conditional static criteria, not observed executions.
