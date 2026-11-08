<!-- Trusted provenance: F-e84a74e08dfd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e84a74e08dfd

## Purpose

F-e84a74e08dfd conditionally updates a signed 32-bit state field, selects an ordered sequence of indirect operations, and performs two final notifications through the input object.

A discriminator mismatch produces a normal return of zero without explicit access to the input object. A matching discriminator leads to a state update and, if all selected dependencies return normally, a return of one.

The established behavior includes normalization toward zero or one, but not a guaranteed binary-state invariant across dependency calls. Application meanings such as “enable,” “toggle,” or “selection” are inferred intent, not established semantics.

## Interface

### Inputs and layouts

The function accepts:

- **Object:** an address used to access a state field and an indirect dispatch table.
- **Token:** an opaque 32-bit value forwarded unchanged to F-682adfd5a561.

The represented data model uses 32-bit integers and 32-bit addresses.

| Object byte offset | Width | Use |
|---|---:|---|
| `0x00` | 32 bits | Address of an indirect dispatch table |
| `0x94` | 32 bits | Signed state field |

The object’s dispatch table contains a callable entry at byte offset `0x58`, used for the final notifications.

F-682adfd5a561 supplies an address with this dependent layout:

| Location | Width | Use |
|---|---:|---|
| Returned address + `0x08` | 32 bits | Address of a subordinate record |
| Subordinate record + `0x04` | 32 bits | Discriminator |
| Subordinate record + `0x0C` | 32 bits | Address of a signed 32-bit delta |

The delta is obtained by dereferencing the address stored at subordinate offset `0x0C`. That field is not itself the delta.

Each manager receiver returned by F-17fe47085641 has a 32-bit dispatch-table address at offset zero. The manager operation is selected through that table’s entry at byte offset `0xF4`.

### Output

The nominal result is 32 bits:

- **Zero:** the discriminator did not equal `0x5680`.
- **One:** the discriminator matched and execution completed both final notifications normally.

There is no separate explicit error return. A return of one does not establish that dependencies accomplished any application-level objective.

### Preconditions

Normal completion requires all addresses actually accessed to be readable, all explicitly written state storage to be writable, and all selected indirect targets to be callable with the represented invocation forms.

The function does not explicitly validate:

- Null pointers, address ranges, alignment, or object lifetimes.
- Dispatch-table contents.
- Lookup success or receiver acquisition success.
- Arithmetic overflow.
- Dependency completion or application-level success.

Signed arithmetic results outside the signed 32-bit range require additional target-level evidence; this contract does not assume wrapping behavior.

The mismatch path does not explicitly dereference the input object. It still dereferences the lookup result and subordinate discriminator storage.

## Behavior

### Lookup and discriminator gate

The first dependency call is F-682adfd5a561 with the token and the 32-bit constant `0x00000F00`.

The function reads the subordinate-record address from returned-address offset `0x08`, then reads the 32-bit discriminator at subordinate offset `0x04`.

If that discriminator is not `0x00005680`, the function returns zero immediately. On this path, it performs no explicit:

- Delta read.
- Object state read or write.
- Normalization dependency call.
- Identifier-source read.
- Manager operation.
- Final notification.

F-682adfd5a561 may itself have side effects; the early return is not a guarantee of global inactivity.

### Additive state update

On a matching discriminator, the representation accesses returned-address offset `0x08` again to obtain the subordinate record for the delta access. It then obtains the delta address from subordinate offset `0x0C` and reads the signed 32-bit delta through it.

The function adds that delta to the signed 32-bit state at object offset `0x94` and writes the sum to that field.

The representation does not establish an evaluation order between the state read and the reads supplying the delta. It does establish that the sum write precedes normalization, identifier acquisition, and manager calls.

There is no explicit dependency call between the discriminator check and this sum write. Nevertheless, the contract does not assume synchronization against external modification or collapse the repeated subordinate-address access into one retained read.

### Normalization

For a representable signed sum, the next action is:

| Computed sum | Action after the initial sum write |
|---|---|
| Negative | Write zero to the state field |
| Zero | Call F-U01 with the 32-bit argument `1` |
| One | Call F-U01 with the 32-bit argument `1` |
| Greater than one | Write one to the state field |

F-U01 is an opaque, documentation-local dependency identifier.

Thus, exact boundary values and out-of-range values differ observably:

- Zero and one cause a dependency call, without an additional explicit normalization write.
- Negative values and values greater than one cause a second state write, without calling F-U01.
- The initial, possibly unnormalized sum is written in every matching case before these alternatives.

No result from F-U01 is used. Although it receives no explicit object argument, it is not known to preserve the object, shared storage, or dispatch tables.

Absent external mutation, normalization leaves the state at zero or one. That conclusion is conditional, not an unconditional postcondition across dependency activity.

### Six identifier-source reads

After normalization, including any F-U01 call, the function performs **six separate 32-bit reads of the same shared source**, in this order:

| Read order | Retained identifier |
|---:|---|
| 1 | Value read, unadjusted |
| 2 | Value read, minus three |
| 3 | Value read, minus five |
| 4 | Value read, minus one |
| 5 | Value read, minus two |
| 6 | Value read, minus four |

Each adjusted identifier is calculated from its own read. All six results are retained before state selection and before any manager acquisition or operation.

These are not six derivatives of one snapshot. They equal offsets from a common value only if the shared source supplies that same value for all six reads. The representation supplies no synchronization guarantee establishing that condition.

No dependency is explicitly called between these six reads. External modification remains outside the established synchronization contract.

Subsequent manager calls do not trigger further reads of this identifier source. A manager dependency changing the shared source therefore does not change the already retained identifiers.

The adjustments are represented using signed 32-bit arithmetic. Results below the signed minimum remain subject to the unresolved arithmetic model.

### State selection and manager sequence

After retaining the six identifiers, the function reads the object state and checks for zero. If that check fails, it reads the state again and checks for one.

These are separate state observations, not an atomic classification. If the first read is nonzero and the second read is zero, neither sequence is selected. If the first read is zero, the zero-state sequence is selected without requiring another state check.

Each manager operation consists of:

1. Calling F-17fe47085641 with no explicit arguments.
2. Obtaining the returned receiver’s dispatch-table address.
3. Selecting its callable entry at byte offset `0xF4`.
4. Invoking that entry with the receiver and two 32-bit arguments.

F-U02 denotes this indirect manager operation. A receiver is acquired separately for every operation; neither receiver identity nor target identity is guaranteed to remain constant.

When the zero-state sequence is selected:

| Order | First argument | Second argument |
|---:|---|---|
| 1 | Minus-four identifier | Zero |
| 2 | Minus-one identifier | Minus-two identifier |
| 3 | Minus-three identifier | Minus-five identifier |
| 4 | Unadjusted identifier | Zero |

When the one-state sequence is selected:

| Order | First argument | Second argument |
|---:|---|---|
| 1 | Minus-four identifier | Minus-five identifier |
| 2 | Minus-three identifier | Zero |
| 3 | Minus-one identifier | Zero |
| 4 | Unadjusted identifier | Minus-two identifier |

If neither sequence is selected, no manager receiver acquisitions or operations occur.

Once selected, a sequence contains no further explicit state checks. Dependency mutation of the state does not make this function switch sequences or omit later operations. The arguments remain the previously retained identifiers and specified zero values.

Manager operation results are not used. Each acquisition result is used as a receiver.

### Final notifications

After the selected manager sequence, or after skipping both sequences, the function performs two notifications through the input object’s dispatch entry at byte offset `0x58`. F-U03 denotes these indirect invocations.

Three shared sources are relevant:

- A **notification reference source**, read twice before the first notification is invoked.
- A **first payload source**, supplying the first notification.
- A **second payload source**, supplying the second notification.

The notification reference reads have distinct roles:

1. The function first reads the reference source and retains that value.
2. It separately reads the same source as an argument expression for the first notification.
3. The second notification uses the retained value from the first read.

The two reference reads need not agree. No stable-storage guarantee is established.

The first notification is represented as receiving:

- The input object as receiver.
- The reference source’s **second** read.
- The first payload source’s 32-bit value.
- A callable value read through the object’s `0x58` dispatch entry.
- An additional machine-register value whose origin is unresolved.

The last two arguments have uncertain semantic status. The representation does not establish a reliable complete signature or application meaning for them.

The retained reference read precedes the first call’s expression evaluation. Within the first call expression, no unsupported order is imposed among the second reference read, payload read, actual-target lookup, and callable-valued argument lookup. In particular, the contract does not collapse the callable-valued argument lookup and the actual call-target lookup into one access.

After the first notification returns, the second notification is represented as receiving:

- The input object as receiver.
- The retained **first** reference read.
- The second payload source’s 32-bit value.

The second payload and second notification target are obtained after the first notification returns. Their relative evaluation order is not established. The first notification may affect either source, the input object, or its dispatch table.

Changes to the notification reference source do not replace the retained value used by the second notification.

After the second notification returns normally, the function returns one.

## Side effects

The explicit state changes are confined to the object state field:

- One sum write on every matching-discriminator path.
- One additional zero-or-one write when the sum lies outside that range.

The matching path also performs the six identifier-source reads and, on normal completion, both final notifications. It conditionally calls F-U01 once and conditionally performs four manager acquisitions and four manager operations.

There is no explicit allocation, deallocation, reference-count adjustment, ownership transfer, or rollback.

Ownership and lifetime of the lookup result, subordinate record, delta storage, receivers, and input object are not established. Absence of a release operation does not establish borrowed, static, or permanently retained ownership. Dependencies may allocate, release, retain, or otherwise modify external state; none of those effects may be assumed absent.

No explicit recovery handles invalid addresses, invalid targets, arithmetic failure, or abnormal dependency completion. Earlier writes and dependency effects are not rolled back if a later operation fails.

Thread safety and reentrancy guarantees are not established. The ordering described here is the represented sequence of accesses and calls, not a memory-ordering guarantee for concurrent observers.

## Dependencies

| Opaque ID | Established use | Unresolved properties |
|---|---|---|
| F-682adfd5a561 | Receives the token and `0xF00`; supplies the lookup structure | Failure convention, ownership, lifetime, side effects |
| F-17fe47085641 | Acquires a receiver separately for each manager operation | Identity stability, ownership, allocation and release effects |
| F-U01 | Receives `1` exactly when the computed sum is zero or one | Application meaning, effects, full calling convention |
| F-U02 | Receiver operation selected at dispatch offset `0xF4`, with two 32-bit arguments | Meaning, return semantics, ownership effects |
| F-U03 | Input-object operation selected at dispatch offset `0x58` | Complete signature, extra first-call arguments, notification semantics |

F-U01, F-U02, and F-U03 are documentation-local opaque identifiers, not recovered identities. No dependency is assumed to preserve globals, object state, memory ownership, or dispatch tables. Unresolved properties are **documented_only**.

## Acceptance examples

These cases are conditional consequences of static evidence. No runtime tests have been performed.

### Discriminator mismatch

With valid lookup storage and discriminator `0x5681`:

- F-682adfd5a561 receives the token and `0xF00`.
- The function returns zero.
- It performs no explicit object access, delta read, identifier-source read, manager operation, or notification.

Lookup-side effects remain possible.

### Normalization boundaries

Assuming a matching discriminator, representable arithmetic, normal dependency completion, and no relevant mutation:

| Initial state | Delta | Ordered state writes | F-U01 | Selected sequence |
|---:|---:|---|---|---|
| 4 | −4 | 0 | Called with `1` | Zero-state |
| −2 | 3 | 1 | Called with `1` | One-state |
| 0 | −1 | −1, then 0 | Not called | Zero-state |
| 1 | 1 | 2, then 1 | Not called | One-state |

Both notifications follow, and the result is one.

### Stable identifier source

If all six identifier reads return `100`, the retained identifiers are respectively `100`, `97`, `95`, `99`, `98`, and `96`.

The zero-state sequence uses these ordered pairs:

1. `(96, 0)`
2. `(99, 98)`
3. `(97, 95)`
4. `(100, 0)`

The one-state sequence uses:

1. `(96, 95)`
2. `(97, 0)`
3. `(99, 0)`
4. `(100, 98)`

Each pair uses a separately acquired receiver. These examples require agreement across all six reads, not merely an initial source value of `100`.

### Changing identifier source

If the six ordered reads return `100`, `200`, `300`, `400`, `500`, and `600`, with representable arithmetic, the retained identifiers are respectively `100`, `197`, `295`, `399`, `498`, and `596`.

The zero-state sequence then uses:

1. `(596, 0)`
2. `(399, 498)`
3. `(197, 295)`
4. `(100, 0)`

Later changes to the shared source during manager calls do not alter these arguments or cause more reads of that source.

### Distinct notification reference reads

If the retained reference read yields `10` and the separate first-notification argument read yields `20`, the first notification receives `20` in that position and the second receives `10`.

If both reads yield `10`, both notifications receive `10`; that equality is conditional on the two reads agreeing.

If the first notification changes the reference source to `30` and the second payload source to `40`, the second notification still receives the retained `10` and obtains payload `40`, assuming no further mutation.

### Dependency mutation and arithmetic limits

If F-U01 changes the object state to `7`, and both subsequent state checks observe `7`, no manager sequence runs. The two notifications still follow, with a result of one if they return normally.

An initial state of `2,147,483,647` plus delta `1` has no established portable outcome here. Likewise, an identifier read of `−2,147,483,648` followed by a negative adjustment has no established portable result. Wrapping expectations require additional arithmetic evidence.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the discriminator, state, identifiers, and notification values.
- Signed-overflow behavior for state addition and identifier adjustments.
- The complete F-U03 calling convention, including the first invocation’s extra represented arguments.
- Origin and meaning of the additional register-derived value.
- Dependency ownership, retention, release, and global-preservation behavior.
- Synchronization, reentrancy, and concurrent-access guarantees.
- Evaluation order within expressions where the representation does not establish it.
- Abnormal completion behavior, including unwinding, termination, or other control transfer.

Binary-state normalization is suggested by the explicit writes. Relationship changes between identifiers are suggested by the argument pairs. Both are inferred intent only; neither supplies missing application semantics.

## Evidence

This contract derives solely from the supplied static representation and dependency mapping.

Established static evidence includes:

- Lookup arguments, dependent 32-bit layouts, repeated subordinate-address access, and discriminator comparison.
- The state sum write, normalization writes, and exact zero-or-one condition for F-U01.
- Six separate ordered reads of one identifier source, each retained independently with its specified adjustment.
- State checks after those reads and the two ordered manager sequences.
- Fresh receiver acquisition before every manager operation, without later identifier-source rereads.
- A retained notification reference read followed by a separate read for the first notification’s argument.
- Use of the retained first reference value in the second notification.
- Final notification ordering and the two normal return values.

No implementation access, runtime execution, instrumentation, or tests were used. Behavior beyond these static observations remains unverified.
