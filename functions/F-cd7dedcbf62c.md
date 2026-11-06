<!-- Trusted provenance: F-cd7dedcbf62c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd7dedcbf62c

## Purpose

F-cd7dedcbf62c conditionally invokes a specialized handler, then falls back to a dynamically selected handler, and finally may invoke an additional context-specific handler. The first handler that returns a nonzero result terminates processing; otherwise, the function returns zero.

The routing decisions depend on an event-like record, fields in a context-like object, and callable entries in a table referenced by that context. “Event,” “context,” and “handler” describe inferred roles, not established application-level types.

The function does not itself modify either supplied object. Its callees may modify them or other state. Later decisions use fresh reads where specified below, so changes made by a callee can affect subsequent routing.

## Interface

### Inputs

Three arguments are visible:

1. **Context address.** Identifies storage containing a dispatch-table reference and at least two conditionally accessed fields.
2. **Record address.** Identifies storage containing a discriminator and additional fields whose interpretation depends on that discriminator.
3. **Additional integer.** Present in the interface, but its incoming value does not affect the established routing or return value. The only comparison that could appear to use this argument first replaces its working value with a context field. It is not forwarded to any visible callee.

Every visible handler invocation receives exactly the context address and record address, in that order.

### Representation and layout

The supplied representation uses 32-bit integer quantities and byte offsets consistent with a 32-bit address space. This contract describes that represented layout; an independent binary interface declaration was not supplied.

The context has the following accessed locations:

| Byte offset | Interpretation in this function |
|---|---|
| `0x00` | Address of a callable-entry table |
| `0x50` | Value used by the final conditional handler |
| `0xB4` | Value used by the initial discriminator-`0x0F` route |

The record has these conditionally accessed locations:

| Byte offset | Interpretation in this function |
|---|---|
| `0x04` | Discriminator |
| `0x08` | Comparison value for discriminator `0x0F`, or nested-object address for discriminator `0x16` |
| `0x0C` | Numeric value tested for discriminator `0x13` |

For the discriminator-`0x16` dispatch decision, the nested object is read at byte offset `0x04`.

The dispatch table contains these accessed callable entries:

| Byte offset | Selection |
|---|---|
| `0xE0` | Final conditional handler |
| `0x10C` | Fallback handler when the current discriminator is `0x16` and the nested value is `0x2A` |
| `0x110` | Fallback handler for every other case |

The supplied evidence does not establish complete object sizes, unrelated fields, type inheritance, or whether these tables are language-level virtual tables.

### Output

The function returns a 32-bit integer result:

- A nonzero handler result is returned unchanged.
- Zero is returned when no invoked handler produces a nonzero result.

No interpretation of individual nonzero values is established. In particular, nonzero is not proven to mean success, failure, or a Boolean value. A negative result is still nonzero and therefore terminates processing.

### Preconditions

Every location actually read on the executed path must be readable using the represented layout. Every selected callable entry must be invocable with the two visible arguments and a compatible result convention.

These are path-dependent requirements. For example, a record that is never examined as discriminator `0x16` does not require a valid nested-object address merely because storage exists at record offset `0x08`.

No null-address checks, bounds checks, or callable-entry validation are visible. Invalid storage or targets have no defined recovery behavior in the supplied evidence.

## Behavior

### Initial specialized routing

The function first reads the record discriminator at offset `0x04`.

If that value is `0x0F`, it then reads:

1. The record value at offset `0x08`.
2. The context value at offset `0xB4`.

If these values are equal, it invokes F-350b851df1b4 with the context and record addresses.

- A nonzero result is returned immediately.
- A zero result continues to fallback dispatch.
- This route does not also invoke F-a37a960bdd03.

If the initial discriminator is `0x13`, the function reads the record value at offset `0x0C` and tests whether subtracting `0x3C00` produces `0x0114`. When that test succeeds, it invokes F-a37a960bdd03 with the context and record addresses.

- A nonzero result is returned immediately.
- A zero result continues to fallback dispatch.

Under the represented 32-bit subtraction model, the matching input value is `0x3D14`. The subtraction has no visible overflow check or distinct error path. The source-level presentation alone does not establish a general signed-overflow policy for other arithmetic or other functions.

For every other initial condition, the function proceeds directly to fallback dispatch. This includes:

- Initial discriminator `0x0F` with unequal comparison values.
- Initial discriminator `0x13` with a nonmatching numeric value.
- Any other initial discriminator.

The two specialized handlers are mutually exclusive within one invocation. At most one is called.

### Fallback dispatch

Fallback dispatch begins with a fresh read of the record discriminator at offset `0x04`. It does not reuse the initial discriminator.

If the current discriminator is `0x16`, the function reads the address at record offset `0x08`, then reads the value at offset `0x04` within that referenced object.

- If the nested value is `0x2A`, it selects dispatch-table offset `0x10C`.
- Otherwise, it selects dispatch-table offset `0x110`.

For every current discriminator other than `0x16`, it selects offset `0x110` without reading the nested-object address or nested value for this decision.

After the selection, the function reads the dispatch-table address from context offset `0x00`, loads the selected callable entry, and invokes it with the context and record addresses.

A nonzero result is returned immediately. A zero result continues to the final conditional stage.

The fallback selection therefore observes mutations made by an earlier specialized handler, if that handler returned zero. Neither the record discriminator nor the context’s table address is assumed to retain its original value.

### Final conditional handler

After a fallback handler returns zero, the function again reads the record discriminator at offset `0x04`.

If it is not `0x0F`, the function returns zero.

If it is `0x0F`, the function reads:

1. The record value at offset `0x08`.
2. The context value at offset `0x50`.

If these values differ, the function returns zero.

If they are equal, the function reads the context’s current dispatch-table address, loads the callable entry at table offset `0xE0`, and invokes that entry with the context and record addresses.

- A nonzero result is returned unchanged.
- A zero result causes the function to return zero.

This final comparison uses context offset `0x50`, not the offset `0xB4` used by the initial specialized route. Matching one field does not establish that the other matches.

The final stage observes changes made by the fallback handler. In particular, the discriminator, comparison fields, and dispatch-table address are read after that handler returns.

### Call count and termination

Assuming every invoked handler returns normally:

- At least one handler is invoked: fallback dispatch is reached unless a specialized handler already returned nonzero.
- At most three handlers are invoked: one specialized handler, one fallback handler, and one final conditional handler.
- There is no visible loop or retry.
- No later handler is invoked after a nonzero result.

The function does not combine handler results or retain a zero result as a distinct status.

## Side effects

### Direct effects

The function directly performs reads and calls. No direct write to the context, record, nested object, dispatch table, or global storage is visible.

No direct allocation, deallocation, ownership transfer, reference-count adjustment, or cleanup operation is visible.

### Callee-mediated effects

All handler side effects remain possible unless separately constrained by their contracts. The supplied evidence does not establish that a handler:

- Preserves globals.
- Preserves the context, record, nested object, or dispatch table.
- Leaves callable entries unchanged.
- Retains or releases memory.
- Returns normally.

The caller passes the same two addresses to each invoked handler, rather than copies of the objects. Fresh reads after a zero result make handler-mediated mutations behaviorally significant.

Any handler that returns and permits further processing must leave the subsequently accessed storage valid for those reads and calls. There is no visible repair or validation if it does not.

The read sequence described here is the sequence shown by the supplied static representation. It does not establish thread synchronization, atomicity, volatile-access guarantees, or a concurrency contract.

## Dependencies

| Dependency | Established role |
|---|---|
| F-350b851df1b4 | Initial specialized handler for discriminator `0x0F` when record offset `0x08` equals context offset `0xB4` |
| F-a37a960bdd03 | Initial specialized handler for discriminator `0x13` when the arithmetic test succeeds |
| Callable at current table offset `0x10C` | Fallback handler for current discriminator `0x16` with nested value `0x2A` |
| Callable at current table offset `0x110` | Fallback handler for all other fallback selections |
| Callable at current table offset `0xE0` | Final handler for current discriminator `0x0F` with record/context comparison equality |

Opaque IDs for the dynamically selected targets were not supplied. Their identities must not be inferred from their table slots.

F-d5dd567991e2 is associated in the supplied metadata with the name attached to the fallback control-flow destination. The provided body shows a branch to that destination within this function, not an explicit invocation of F-d5dd567991e2. A separate dependency call to that ID is therefore not established.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Each assumes valid storage, compatible callable targets, normal returns, and only the mutations stated.

1. **Initial specialized result terminates processing.**  
   The initial discriminator is `0x0F`, and record offset `0x08` equals context offset `0xB4`. If F-350b851df1b4 returns `7`, the overall result is `7`. Neither fallback dispatch nor the final conditional handler occurs.

2. **The two context comparisons are independent.**  
   The initial discriminator is `0x0F`. The record comparison value differs from context offset `0xB4` but equals context offset `0x50`. With no intervening mutation, fallback uses table offset `0x110`. If that handler returns zero, the final handler at offset `0xE0` is invoked.

3. **The numeric specialized route preserves the exact result.**  
   The initial discriminator is `0x13`, and the numeric field is `0x3D14`. F-a37a960bdd03 is invoked. If it returns a negative nonzero result, that result is returned unchanged and fallback dispatch is skipped.

4. **A nonmatching numeric field bypasses specialization.**  
   The initial discriminator is `0x13`, and the numeric field is `0x3D15`. F-a37a960bdd03 is not invoked. With no relevant mutation, fallback selects table offset `0x110`.

5. **Nested dispatch chooses the alternate fallback.**  
   At fallback entry, the discriminator is `0x16`, the nested address is valid, and its offset-`0x04` value is `0x2A`. The handler at table offset `0x10C` is selected. A different nested value selects offset `0x110`.

6. **A zero specialized result allows changed routing.**  
   F-350b851df1b4 is initially selected, changes the record to discriminator `0x16` with a valid nested value of `0x2A`, and returns zero. Fallback uses the changed record and selects offset `0x10C`.

7. **A fallback mutation can enable the final handler.**  
   The fallback handler changes the discriminator to `0x0F`, makes the record comparison value equal context offset `0x50`, replaces the context’s table reference, and returns zero. The final call uses offset `0xE0` from the replacement table.

8. **The third argument is behaviorally irrelevant.**  
   Holding memory state and handler behavior fixed, changing only the incoming third integer does not change the established decisions or result.

9. **All-zero handling returns zero.**  
   If every invoked handler returns zero, the overall result is zero, whether the final handler is invoked or bypassed.

## Uncertainties

The following remain **documented_only**:

- Application meanings of discriminators `0x0F`, `0x13`, and `0x16`, nested value `0x2A`, and numeric value `0x3D14`.
- Application-level meanings of zero and individual nonzero results.
- Exact ABI, pointer representation, alignment rules, and endianness beyond the represented layout.
- Full object extents, ownership rules, lifetimes, and aliasing constraints.
- Concrete identities and complete contracts of the dynamically selected handlers.
- Handler mutation, allocation, release, exception, and nonlocal-control-transfer behavior.
- Concurrency and synchronization requirements.
- Whether the fallback destination’s metadata association represents anything beyond an internal control-flow label.

No safety guarantee follows merely from the absence of direct writes: callees may mutate state, and later reads depend on that state remaining valid.

## Evidence

**Established static evidence:** The supplied function body supports the conditional field accesses, handler order, result short-circuiting, fresh reads after zero results, table-slot selection, and absence of visible direct stores. The supplied dependency map identifies the two directly invoked specialized handlers.

**Inferred intent:** The function appears to be a layered event dispatcher with specialized handling, general fallback, and a final context-specific opportunity to handle the record. That interpretation is not required for the concrete behavior above.

**Verification limit:** No runtime tests have been performed. The acceptance examples express consequences of the supplied static representation, not observed executions.
