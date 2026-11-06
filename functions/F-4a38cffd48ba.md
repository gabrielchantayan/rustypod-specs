<!-- Trusted provenance: F-4a38cffd48ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4a38cffd48ba

## Purpose

F-4a38cffd48ba conditionally updates six 32-bit fields in a supplied state region. An event record provides a discriminator and a value. Depending on that discriminator and the existing state, the function records an opening value, increments a counter, accumulates a difference, or clears an opening.

This contract calls the two groups of fields **primary** and **secondary**. These are descriptive labels. An opening field is considered active exactly when it is nonzero.

The established behavior is conditional field manipulation. Interpreting event values as timestamps, differences as elapsed durations, or counters as numbers of intervals is inferred intent. No units, clock properties, or application-level meaning are established.

The function has no explicit return value and invokes no dependency functions. Its observable results are memory reads and conditional state writes.

## Interface

### Arguments and result

The function receives:

1. **State address:** an address-like argument used to locate the six state fields below.
2. **Event address:** a pointer to a record containing a discriminator and an event value.

The supplied representation uses 32-bit integer fields. The state address is represented through an integer argument; a complete platform ABI and address model are not supplied.

There is no returned status, result object, or explicit error report.

### State layout

Offsets are byte offsets from the state address.

| Offset | Width | Contract label | Use |
|---|---:|---|---|
| `0x40` | 32 bits | Primary accumulator | Receives a difference when an active primary opening is closed |
| `0x44` | 32 bits | Primary counter | Incremented when a primary opening is accepted |
| `0x48` | 32 bits | Primary opening | Zero denotes inactive |
| `0x4C` | 32 bits | Secondary accumulator | Receives a difference when an active secondary opening is closed |
| `0x50` | 32 bits | Secondary counter | Incremented when a secondary opening is accepted |
| `0x54` | 32 bits | Secondary opening | Zero denotes inactive |

The highest accessed state byte is at offset `0x57`. A suitably aligned, readable and writable region covering the first `0x58` bytes is sufficient for the state accesses, provided address calculations are valid. The function does not access every byte in that region.

No initialization is performed. Existing accumulator and counter values participate directly in subsequent updates.

### Event layout

Offsets are byte offsets from the event address.

| Offset | Width | Use |
|---|---:|---|
| `0x00` | 32 bits | Discriminator |
| `0x04` | 32 bits | Not directly accessed |
| `0x08` | 32 bits | Event value |

The recognized discriminators are:

| Discriminator | Ordinary interpretation |
|---|---|
| `0` | Request a primary opening |
| `1` | Request a primary closing |
| `14` | Request a secondary opening, then consider primary closing |
| `15` | Request a secondary closing, then consider primary closing |

Other stable discriminator values produce no state writes.

### Preconditions

Every location actually accessed must be valid for the corresponding read or write, with suitable alignment and valid address arithmetic. No null, bounds, alignment, or address checks are present.

The ordinary event-level behavior below assumes:

- The event contents remain stable throughout the call.
- The event record does not overlap the state fields being manipulated.
- No concurrent activity interferes with the relevant state accesses.
- Arithmetic remains within the representable signed 32-bit range at each operation.

These assumptions are not enforced by the function. Non-overlap is a sufficient way to prevent state writes from altering event storage; it is not necessary for every individual call to leave event contents unchanged.

Storage ownership and lifetime are not established by the interface.

## Behavior

### Ordinary event-level behavior

These rules apply under the ordinary assumptions above. Except where stated, fields retain their existing values.

#### Discriminator `0`: primary opening

A primary opening is accepted only when both opening fields are zero.

The primary opening is inspected first. If it is nonzero, the function returns without changing state and without inspecting the secondary opening on this path.

If the primary opening is zero, the secondary opening is inspected. A nonzero secondary opening prevents acceptance.

When both are zero:

1. The event value is stored in the primary opening.
2. The primary counter is incremented by one.
3. The function returns.

Neither accumulator nor either secondary field is written.

An event value of zero is accepted. It increments the primary counter but leaves the primary opening equal to the inactive sentinel. Consequently, repeated zero-valued opening requests can each be accepted while both openings remain zero.

#### Discriminator `1`: primary closing

If the primary opening is zero, no state is written.

If it is nonzero, the primary accumulator receives its old value plus the difference between the event value and the primary opening. The primary opening is then cleared to zero.

The secondary opening does not gate this operation. Secondary fields and both counters remain unchanged.

#### Discriminator `14`: secondary opening and primary closing

The secondary opening is inspected first.

If it is zero:

1. The event value is stored in the secondary opening.
2. The secondary counter is incremented by one.
3. Primary closing is considered afterward.

If the secondary opening is already nonzero, that opening and its counter remain unchanged, and execution proceeds directly to the primary closing check.

Primary closing occurs whenever the primary opening is nonzero: its difference is added to the primary accumulator, and the primary opening is cleared afterward.

A preexisting active secondary opening therefore does not prevent an active primary opening from being closed. No mutual-exclusion invariant is enforced between the two groups.

A zero-valued secondary opening increments the secondary counter but leaves the secondary opening inactive. The same call can still close an active primary opening.

#### Discriminator `15`: secondary closing and primary closing

The secondary opening is inspected first.

If it is nonzero:

1. The secondary accumulator receives its old value plus the event value minus the secondary opening.
2. The secondary opening is cleared to zero.
3. Primary closing is considered afterward.

If the secondary opening is already zero, execution proceeds directly to the primary closing check without writing secondary fields.

Primary closing uses the rule described for discriminator `1`.

One call can therefore update both accumulators and clear both openings. Neither counter is incremented by closing.

#### Other discriminators

A stable discriminator other than `0`, `1`, `14`, or `15` causes no state writes and produces no error indication.

### Observable access sequence

The function does not capture the event record as a single snapshot.

Its discriminator selection proceeds in stages:

1. Read the discriminator and compare it with `14`.
2. Only if that comparison fails, read the discriminator again and compare it with `15`.
3. Perform any selected secondary operation.
4. Unless a direct path below applies, read the discriminator again for primary-operation selection.

There are two direct paths to the primary closing check:

- The first discriminator read selects `14`, and the secondary opening is nonzero.
- The second discriminator read selects `15`, and the secondary opening is zero.

Neither direct path performs the later discriminator read.

For paths that do perform that later read:

- `0` selects the conditional primary-opening operation.
- `1`, `14`, and `15` select the primary closing check.
- Any other value returns without a primary operation.

The established store order is:

- An accepted secondary opening is stored before its counter is incremented.
- A secondary accumulator update is stored before the secondary opening is cleared.
- Those secondary stores precede the later discriminator read, when that read occurs.
- An accepted primary opening is stored before its counter is incremented.
- A primary accumulator update is stored before the primary opening is cleared.

Each needed event value is read for its particular operation. A call closing both groups has a separate event-value read for each accumulator update; there is no established single value snapshot shared by both operations.

An active opening is tested before a closing update. The arithmetic expression also reads that opening and the existing accumulator. The supplied representation does not establish a complete relative order among the operand reads inside an arithmetic expression.

### Overlapping storage

The staged accesses matter when event and state storage overlap.

A secondary store can alter the discriminator seen by the later primary-selection stage. It can also alter event-value bytes read by a subsequent operation. Thus, the ordinary discriminator table is not an alias-independent guarantee.

Overlap does not necessarily alter event contents. An overlapping location may not be written on the selected path, or a write may preserve its existing bytes. Non-overlap remains an explicit assumption of the ordinary behavioral contract because it excludes this source of feedback without requiring case-specific overlap analysis.

Repeated reads do not establish support for asynchronous event modification, volatile storage, or concurrent state changes.

### Arithmetic and boundaries

For either group, the mathematical accumulator update is:

**new accumulator = old accumulator + (event value − opening).**

The represented subtraction and addition use signed 32-bit integer operands. Counter increments likewise use the represented 32-bit integer width. No explicit widening, saturation, overflow check, or overflow report appears.

For an established numerical result, the subtraction must be representable, and the subsequent addition must also be representable. A representable final mathematical total alone is insufficient if the intermediate subtraction overflows.

Boundary-crossing arithmetic is **documented_only**: the supplied representation does not establish a portable wrapping rule or a wider intermediate result.

The function imposes no monotonicity requirement. An event value smaller than an active opening produces a negative difference when representable, reducing the accumulator.

Zero alone denotes inactivity. Negative opening values are active. No sign validation or chronological validation occurs.

## Side effects

The only explicit write destinations are the six state words listed in the interface. Which words are written depends on the selected path and current opening values.

The function performs no explicit allocation, release, ownership transfer, or other ownership-management operation. Storage ownership and lifetime remain unknown; receiving an address from a caller does not establish who owns the referenced storage.

No callback, logging, external I/O, or explicit global access appears. No dependency function is invoked.

The event argument is used for reads, not direct writes. State writes can nevertheless modify event bytes when the two storage regions overlap. Non-overlap is sufficient to prevent that particular effect, but overlap alone does not imply that any event byte changes.

Updates are sequential, not an explicitly atomic transaction. There is no rollback or recovery path. If an access fails after earlier writes, the function contains no mechanism to restore the previous state. The execution environment’s response to invalid memory access is not established.

No synchronization operation is shown. The contract provides no atomicity or consistency guarantee for concurrent observers or writers.

## Dependencies

F-4a38cffd48ba invokes no other functions. There are no callee contracts to describe using additional opaque function IDs.

The behavior depends externally on:

- Valid storage for the accesses actually performed.
- Valid address calculations and suitable alignment.
- The execution environment’s integer and memory-access semantics.
- The state and event contents encountered at each read.

No assumption about an unknown callee preserving globals, allocating storage, or releasing memory is needed.

## Acceptance examples

Unless explicitly stated otherwise, these examples assume valid, suitably aligned, non-overlapping storage, stable event contents, no concurrent interference, and representable signed 32-bit arithmetic. Unmentioned fields retain their initial values. These are derived expectations, not observed test results.

1. **Accept a primary opening.**  
   Both openings start at zero, and the primary counter is `4`. Discriminator `0` with value `100` sets the primary opening to `100` and the counter to `5`. Both accumulators remain unchanged.

2. **Reject a primary opening while secondary is active.**  
   The primary opening is zero, and the secondary opening is `80`. Discriminator `0` with value `100` changes no state.

3. **Preserve an existing primary opening.**  
   The primary opening is `60`. Discriminator `0` with value `100` leaves the opening and primary counter unchanged. The secondary opening does not need to be read on this path.

4. **Close only primary.**  
   The primary opening is `100`, its accumulator is `7`, and the secondary opening is `90`. Discriminator `1` with value `130` sets the primary accumulator to `37` and clears the primary opening. Secondary fields remain unchanged.

5. **Open secondary and close primary.**  
   The primary opening is `100`, its accumulator is `7`, the secondary opening is zero, and its counter is `2`. Discriminator `14` with value `130` sets the secondary opening to `130`, increments its counter to `3`, sets the primary accumulator to `37`, and clears the primary opening. Secondary writes occur first.

6. **A duplicate secondary opening still closes primary.**  
   The secondary opening is `90`, its counter is `3`, the primary opening is `100`, and its accumulator is `7`. Discriminator `14` with value `130` preserves the secondary opening and counter, but sets the primary accumulator to `37` and clears its opening.

7. **Close both groups.**  
   The secondary opening is `100` with accumulator `5`; the primary opening is `110` with accumulator `7`. Discriminator `15` with value `140` produces secondary accumulator `45`, primary accumulator `37`, and two zero openings. Neither counter changes.

8. **An unmatched secondary close still closes primary.**  
   The secondary opening is zero, the primary opening is `110`, and its accumulator is `7`. Discriminator `15` with value `140` sets the primary accumulator to `37` and clears its opening. Secondary fields remain unchanged.

9. **Repeated zero-valued openings remain acceptable.**  
   Both openings start at zero, and the primary counter is `4`. Two consecutive discriminator-`0` events with value zero leave both openings zero and increase the primary counter to `6`.

10. **A zero secondary opening can accompany primary closing.**  
    The primary opening is `10`, its accumulator is `20`, the secondary opening is zero, and its counter is `2`. Discriminator `14` with value zero increments the secondary counter to `3`, leaves the secondary opening zero, sets the primary accumulator to `10`, and clears the primary opening.

11. **A decreasing event value is not rejected.**  
    The primary opening is `100`, and its accumulator is `50`. Discriminator `1` with value `90` sets the accumulator to `40` and clears the opening.

12. **Overlap can leave event contents unchanged.**  
    Relax only the non-overlap assumption. If the event overlaps state but its discriminator remains `2`, the selected path performs no state writes. With no external modification, event bytes remain unchanged despite the overlap.

13. **Overflow has no assigned numerical acceptance result.**  
    Both openings are zero, and the primary counter is `2,147,483,647`. A discriminator-`0` event reaches the counter increment after the opening store, but this contract does not assign a portable counter result or guarantee completed-call behavior for that overflowing operation.

## Uncertainties

- **Application meaning — inferred intent.** Opening values, differences, and counters suggest interval accounting. Units, clock source, event origin, and intended application are not established.
- **Overflow — documented_only.** Signed boundary crossings, including counter overflow and intermediate subtraction overflow, have no portable result assigned here.
- **Address model — documented_only.** The state argument is represented as an integer used in address calculations. A complete ABI, pointer-width specification, and alignment policy are unavailable.
- **Ownership and lifetime — documented_only.** Neither storage ownership nor required lifetime beyond valid accesses during execution is established. The function performs no explicit ownership-management operation.
- **Aliasing discipline — documented_only.** No non-alias guarantee is supplied. The ordinary contract assumes non-overlap; overlapping cases depend on the actual access path and affected bytes.
- **Concurrency — documented_only.** Repeated reads do not establish synchronization, supported asynchronous modification, or atomic access guarantees.
- **Initialization and counter scope — documented_only.** Initial values and whether counters represent lifetime, session, or another scope are unknown.
- **Invalid-access behavior — documented_only.** No validation or recovery exists in the supplied function; the environment’s response is unspecified.

## Evidence

Established static evidence consists of the field offsets, represented 32-bit integer accesses, discriminator comparisons, zero-sentinel checks, arithmetic expressions, conditional counter increments, store sequencing, repeated discriminator reads, direct paths to primary closing, and absence of function calls.

The source establishes no explicit allocation, release, or ownership-management operation. It does not establish who owns either storage region. Its conditional writes also show why overlap can alter event contents without making such alteration inevitable.

Interval terminology and application-level interpretations are inferred intent, not additional evidence.

No runtime tests have been performed. The acceptance examples are conditional consequences of static analysis for later review or verification.
