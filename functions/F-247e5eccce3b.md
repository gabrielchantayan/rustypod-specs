<!-- Trusted provenance: F-247e5eccce3b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-247e5eccce3b

## Purpose

F-247e5eccce3b conditionally updates two pairs of 32-bit fields in a caller-supplied object. Each pair is compared with two requested values. If either field in a pair differs, both fields in that pair are written and a dependency is notified with a pair-specific selector.

The second pair has an additional action: after its notification returns, a separate dependency receives an address within the object and the second pair’s requested values.

The two pairs are processed in a fixed order. The first notification, when required, completes before the second pair is inspected. Consequently, dependency-induced changes to the second pair can affect whether its update occurs.

The field meanings and the purpose of the embedded region are not established. Describing the pairs as coordinates, dimensions, ranges, or cached values would be inferred intent, not an established contract.

## Interface

### Inputs

The function accepts five scalar arguments:

| Argument | Role |
|---|---|
| Object address | Base address used for all direct field accesses and passed to the notification dependency. |
| First-pair first value | Requested value for the field at byte offset 200. |
| Second-pair first value | Requested value for the field at byte offset 404. |
| First-pair second value | Requested value for the field at byte offset 204. |
| Second-pair second value | Requested value for the field at byte offset 408. |

The argument order interleaves the two pairs: the first pair uses the second and fourth arguments; the second pair uses the third and fifth arguments.

The supplied representation describes the requested values and accessed fields as 32-bit integers. Comparisons are equality comparisons; there is no ordering comparison, range restriction, scaling, or arithmetic on the requested values. Their signed interpretation therefore does not affect the match-or-mismatch decision.

The object argument is represented as an integer used as an address. A portable pointer ABI, address-space width, and calling convention are not independently established by the supplied material.

### Relevant object layout

Offsets are byte offsets from the object address:

| Offset | Width or extent | Direct use |
|---|---:|---|
| 200, hexadecimal 0xC8 | 4 bytes | First-pair first field. |
| 204, hexadecimal 0xCC | 4 bytes | First-pair second field. |
| 404, hexadecimal 0x194 | 4 bytes | Second-pair first field. |
| 408, hexadecimal 0x198 | 4 bytes | Second-pair second field. |
| 412, hexadecimal 0x19C | Unknown extent | Address passed to F-9a3a1fbd933f. |

The function itself does not read or write the region beginning at offset 412. Its size, layout, and access requirements depend on F-9a3a1fbd933f.

### Output

On normal completion, the function returns a 32-bit zero.

It does not expose a changed/unchanged flag, a count of updates, or a dependency result. Any dependency return values are unused.

### Preconditions

For defined execution of the direct accesses, the object address must support the relevant 32-bit reads and any conditionally required writes at the listed offsets, with whatever alignment and address-validity requirements apply to the execution environment.

When a dependency is called, its own preconditions must also hold. These are not supplied. In particular, the validity of the address at offset 412 cannot be established solely from the four directly accessed fields.

No null-address check, object-size check, alignment check, or address-overflow check is visible. There is no specified recoverable error for violating these preconditions.

## Behavior

### First pair: comparison and update

The function first reads the 32-bit field at offset 200 and compares it with the first-pair first requested value.

The field at offset 204 is read for this comparison only if the field at offset 200 matches. If the first field already differs, the function does not need to read the second field before deciding to update the pair.

The pair is considered unchanged only when both comparisons match.

If either comparison establishes a mismatch, the function performs these actions in order:

1. Writes the first-pair first requested value to offset 200.
2. Writes the first-pair second requested value to offset 204.
3. Calls F-5071a094742d with the object address and selector 0.

Both writes occur whenever this pair requires an update. A field whose old value already matches is still written if the other field differs. This is not a field-by-field selective update.

If both fields match, neither field is written and the selector-0 call is omitted.

### Second pair: comparison and update

After the first-pair processing has finished, including the selector-0 call if made, the function reads the 32-bit field at offset 404. It compares that field with the second-pair first requested value.

The field at offset 408 is read for this comparison only if the field at offset 404 matches. A mismatch in the first field is sufficient to trigger the update without first reading the second field.

If either field differs, the function performs these actions in order:

1. Writes the second-pair first requested value to offset 404.
2. Writes the second-pair second requested value to offset 408.
3. Calls F-5071a094742d with the object address and selector 1.
4. After that call returns, calls F-9a3a1fbd933f with the address at object offset 412, followed by the second-pair first and second requested values.

If both fields match, neither field is written, the selector-1 call is omitted, and F-9a3a1fbd933f is not called.

The arguments passed to F-9a3a1fbd933f are the requested input values. They are not obtained by rereading offsets 404 and 408 after the selector-1 notification. If that notification changes those fields, the later dependency still receives the original requested values.

### Sequencing and dependency interference

The comparison of the second pair uses the object state present after the first notification returns, not a snapshot taken at entry.

This permits two important outcomes:

- A selector-0 notification can make the second pair match its requested values, thereby suppressing the second update and both associated calls.
- A selector-0 notification can make the second pair differ, thereby causing an update even if that pair matched at function entry.

There is no corresponding recheck of the first pair after either dependency has run. The function also does not verify that any written field retains its requested value after a dependency returns.

If both pairs require updates at their respective comparison times, the dependency call order is:

1. F-5071a094742d with selector 0.
2. F-5071a094742d with selector 1.
3. F-9a3a1fbd933f for the region at offset 412.

There is no rollback path in the supplied body. If execution stops during a dependency call, any earlier writes or dependency effects may already have occurred.

## Side effects

The function’s explicit memory writes are limited to four 32-bit field locations:

- Offsets 200 and 204, when the first pair differs.
- Offsets 404 and 408, when the second pair differs at the time it is inspected.

Within each pair, the lower-offset field is written first. Both writes precede the associated notification.

A no-change path performs no explicit writes and makes no dependency calls. It still performs the comparisons, including the conditional second-field reads.

No allocation, deallocation, ownership transfer, locking, logging, or global-memory access is explicit in this function. This does not establish that its dependencies lack such effects.

The object is not explicitly released by this function. Whether either dependency retains, releases, replaces, or otherwise affects the lifetime of the object or embedded region is unknown. Safe execution requires that later accesses and calls remain valid despite any earlier dependency effects.

The ordered accesses described here are the explicit operations in the supplied representation. They do not establish atomicity, synchronization, volatile-access semantics, or a guarantee about observations by concurrent threads. In particular, each pair is updated through two separate writes, not an established atomic pair replacement.

## Dependencies

### F-5071a094742d

**Established use:** Called with the object address and a selector.

- Selector 0 follows a first-pair update.
- Selector 1 follows a second-pair update.
- The pair’s two writes precede its call.
- The call is omitted when the corresponding pair matches.
- Its return value is unused.

**Unknowns:** Its purpose, return type, failure behavior, memory footprint, reentrancy, lifetime effects, and effects on globals or other object fields are not supplied.

Calling it a notification describes its position in this function’s behavior, not a proven restriction on its effects.

### F-9a3a1fbd933f

**Established use:** Called only after a second-pair update and after the selector-1 call returns.

Its arguments are, in order:

1. The address at object offset 412.
2. The second-pair first requested value.
3. The second-pair second requested value.

Its return value is unused.

**Unknowns:** The embedded region’s type and extent, whether this dependency initializes or modifies it, whether it allocates or releases resources, and whether it accesses other state are not established.

The additional call suggests that a second-pair change requires some further processing. The nature of that processing remains inferred intent.

## Acceptance examples

These are conditional specification examples, not executed tests. Unless a dependency effect is explicitly stated, assume valid memory, normal dependency returns, and no dependency mutation of the four fields.

### Both pairs already match

Suppose offsets 200 and 204 contain 10 and 20, and offsets 404 and 408 contain 30 and 40. The requested pairs are likewise (10, 20) and (30, 40).

Expected behavior:

- Reads offsets 200, 204, 404, and 408 in that order.
- Performs no explicit writes.
- Makes no dependency calls.
- Returns zero.

### Only the first pair’s second field differs

Suppose the first pair contains (10, 99), and the requested first pair is (10, 20). The second pair already matches.

Expected behavior:

- Reads offset 200, then offset 204.
- Writes 10 to offset 200 even though it already matches.
- Writes 20 to offset 204.
- Calls F-5071a094742d with selector 0.
- Inspects the second pair after that call returns.
- Omits the second pair’s update and associated calls if it still matches.
- Returns zero.

### The second pair’s first field differs

Suppose the first pair matches. The second pair contains (99, 40), and its requested values are (30, 40).

Expected behavior:

- Reads offset 404 and detects the mismatch.
- Does not read offset 408 as part of the comparison.
- Writes 30 to offset 404, then 40 to offset 408.
- Calls F-5071a094742d with selector 1.
- Calls F-9a3a1fbd933f with the address at offset 412 and values 30 and 40.
- Returns zero.

The matching second field does not prevent it from being written.

### The first notification removes a second-pair mismatch

Suppose both pairs differ at entry. After the first-pair writes, the selector-0 dependency changes offsets 404 and 408 to exactly the second pair’s requested values.

Expected behavior:

- Performs the first-pair update and selector-0 call.
- Reads the now-matching second pair.
- Makes no selector-1 call and no F-9a3a1fbd933f call.

An expectation based solely on the second pair’s entry state would be incorrect.

### The second notification changes the freshly written pair

Suppose the second pair requires updating to (30, 40). After those writes, the selector-1 dependency changes its fields to (70, 80).

Expected behavior:

- F-9a3a1fbd933f still receives 30 and 40.
- This function does not reread or restore the second pair.
- The eventual field contents depend on dependency effects, not merely on this function’s writes.

### Boundary bit patterns

Requested 32-bit values with the high bit set, or all bits set, follow the same equality-and-replacement behavior as other values. There is no visible signed-order comparison or value arithmetic that would create a special negative-value or overflow branch.

## Uncertainties

The following are **documented_only** rather than filled in by assumption:

- Semantic names, units, and valid application-level ranges for all four fields.
- Full object size, object type, and embedded-region layout.
- Target byte order, pointer ABI, and address-arithmetic overflow behavior.
- Dependency ownership, retention, release, global-state, and failure behavior.
- Whether concurrent access is supported.
- Whether callbacks, reentrancy, or nonlocal exits are possible.
- Any external invariant connecting the two pairs or the embedded region.

The supplied body establishes an unconditional zero return only for normal completion. It does not establish that invalid addresses or dependency failures are converted into zero, nor that every invocation reaches a return.

## Evidence

The static evidence supplied for F-247e5eccce3b establishes:

- Five inputs and a zero-valued 32-bit normal return.
- Two ordered pair comparisons with conditional reads of each pair’s second field.
- Two ordered 32-bit writes whenever a pair differs.
- Pair-specific calls to F-5071a094742d.
- A further call to F-9a3a1fbd933f only on the second-pair update path.
- No explicit validation, rollback, allocation, release, or dependency-result handling.

The two-pair update model is established by those accesses and branches. Application-level interpretations of the fields or dependencies are not established.

No runtime tests have been performed. The acceptance examples describe consequences of the static control flow under their stated conditions; they are not observed execution results.
