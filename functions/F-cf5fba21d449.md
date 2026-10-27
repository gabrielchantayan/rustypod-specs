<!-- Trusted provenance: F-cf5fba21d449; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cf5fba21d449

## Purpose

F-cf5fba21d449 compares two captured unsigned 16-bit object fields. When they differ, and an optional guard does not terminate processing, it calls a dependency with an address-derived argument and the low eight bits of one captured value. After that dependency returns, it copies the complete captured 16-bit value into the other field.

The guard can instead cause an immediate return of `0x58`. All other normal completion paths return zero.

This contract calls the field supplying the replacement value **requested** and the field conditionally overwritten **recorded**. It calls the dependencies **guard** and **notification**. These are inferred role descriptions, not established application semantics.

An important distinction is the timing of observations:

- Both 16-bit fields are captured before any dependency call.
- A separate global datum is read only when the guard is required.
- The address-bearing object field is read only when notification is required, after any guard call.
- The final write uses the captured requested value, not a refreshed object-field value.

## Interface

### Input and object layout

The function accepts one object-base argument. The supplied representation exposes this as a 32-bit integer-shaped address. The original declared pointer type, if any, is not established.

Only the following object fields are directly accessed:

| Offset from object base | Access width | Interpretation within this contract |
|---|---:|---|
| `0x0C` | 16 bits, unsigned | Requested value, captured second |
| `0x3C` | 32 bits | Address-bearing value used to derive the first notification argument |
| `0x4C` | 16 bits, unsigned | Recorded value, captured first and conditionally overwritten |

These offsets do not establish the complete object layout, its allocation size, or ownership of any referenced storage.

The function also conditionally reads a separate global datum to obtain the guard’s second argument. That argument is the **value of the global datum**. It is not established to be the address of the global datum. The datum’s width, concrete value, mutability, and interpretation are unknown.

### Output

Normal completion produces a 32-bit result:

| Result | Established condition |
|---|---|
| `0x00000058` | The guard was called and returned exactly `1` |
| `0x00000000` | Processing completed without that guard result |

Zero does not establish application-level success. In particular, the notification dependency’s return value is not inspected.

### Path-dependent preconditions

The caller and any returning dependency must leave the accesses required by the selected path valid:

- The initial 16-bit reads at `0x4C` and `0x0C` must be valid.
- If the captured recorded value is not `1`, the separate global datum must be readable when its value is obtained for the guard call.
- If notification is reached, the 32-bit field at `0x3C` must be readable after any guard call.
- The address-derived notification argument must satisfy whatever requirements F-cd0542290b1c imposes.
- If notification returns, the 16-bit field at `0x4C` must still be writable for the final update.

There is no established validation of the object address, global access, derived address, bounds, or lifetime. The function does not establish protection against a dependency releasing or invalidating storage needed later.

## Behavior

### Captured object state

The first direct object access reads the unsigned 16-bit recorded field at offset `0x4C`. The next reads the unsigned 16-bit requested field at offset `0x0C`. Each captured value is therefore between `0` and `65535`, inclusive.

Both reads precede the conditional global read and all dependency calls. They are ordered observations, not an established atomic snapshot. No locking or other synchronization is shown.

The captured values remain the basis for the guard-bypass decision, the equality comparison, the notification’s numeric argument, and the final replacement value. Changes to the object fields after capture do not refresh those values.

### Guard selection and global argument read

A captured recorded value of exactly `1` bypasses the guard. On this path, the function neither reads the separate global datum for the guard argument nor calls F-2706939fe3c5.

Every other captured recorded value requires the guard. After both initial object-field reads and before entering F-2706939fe3c5, the function reads the separate global datum. It passes the datum’s value as the second argument, with the original object-base argument as the first.

The global read is a direct memory observation by this function, not merely an unspecified action inside the dependency. Its access width and the argument’s exact representation are not established. Although the value might itself have address-like meaning, no such interpretation is established.

The guard result is compared with exactly `1`:

- A result equal to `1` terminates processing with return value `0x58`.
- Any other result allows processing to continue.

This is not a general nonzero test. A guard result of `2`, for example, does not cause the early return.

The early-return path performs no direct object write and does not call the notification dependency. The guard may nevertheless have changed the object, the global datum, or other state. Its internal effects are not established.

### Equality decision

When the guard is bypassed or returns a value other than `1`, the function compares the two captured 16-bit values.

If they are equal, it returns zero without reading the address-bearing field, calling notification, or performing the final write.

Equality does not bypass a required guard. For equal captured values other than `1`, the global read and guard call still occur before equality determines the remaining path. Such an invocation can therefore return `0x58`.

Dependency-induced changes do not alter the captured comparison. If the guard changes either object field, the function still compares their initial captures. Consequently:

- Initially equal captures remain an equality case even if the current fields now differ.
- Initially different captures remain a notification case even if the guard makes the current fields equal.

### Notification and replacement

If the captured values differ and the guard has not terminated processing, the function reads the current 32-bit address-bearing field at offset `0x3C`.

This read occurs after the guard returns. A guard-induced change to that field can therefore affect the notification argument. The field is not captured alongside the two initial 16-bit values.

The first argument to F-cd0542290b1c is the address-bearing value plus `0x18`, an addition of 24. The supplied representation does not settle overflow behavior or address validity at the arithmetic boundary; wrapping must not be assumed.

The second argument has the numeric value of the captured requested field masked to its low eight bits. Its possible values are `0` through `255`. This establishes the value restriction, not an eight-bit calling-convention argument slot.

After F-cd0542290b1c returns, the function writes the complete captured requested value into the 16-bit recorded field at offset `0x4C`, then returns zero.

The notification result does not control this write. A normally returning notification is followed by the write regardless of its return value. No result-based rollback, retry, or recovery is present.

The distinction between the notification argument and the stored value is observable: a captured requested value of `0xABCD` supplies notification value `0xCD`, but the recorded-field replacement is `0xABCD`.

## Side effects

### Direct memory access order

The established direct access order is:

| Stage | Access | Condition |
|---|---|---|
| Initial observation | Read 16 bits at object offset `0x4C` | Every invocation reaching the body |
| Initial observation | Read 16 bits at object offset `0x0C` | After the first read |
| Guard preparation | Read the separate global datum’s value | Captured recorded value is not `1` |
| Notification preparation | Read 32 bits at object offset `0x3C` | Guard has not terminated processing and captured values differ |
| Replacement | Write 16 bits at object offset `0x4C` | Notification has returned |

The conditional global read occurs before guard entry. The address-bearing field read occurs after any guard return. The replacement write occurs after notification returns.

No direct write to the requested field, address-bearing field, or separate global datum is shown. This does not restrict dependency side effects.

### Dependency-visible state

F-cd0542290b1c is entered before this function performs its replacement write. That does not guarantee that the recorded field still contains its initially captured value: the guard or another actor may already have changed it.

If notification modifies the recorded field and returns with the field still writable, the subsequent direct write overwrites that modification with the captured requested value.

If either dependency changes the requested field, the eventual replacement still uses the original capture. If a dependency changes the global datum, this function performs no later global read that would refresh the already supplied guard argument.

### Ownership and abnormal completion

The body contains no explicit allocation, release, retention, or ownership-transfer operation. Neither dependency is established to preserve storage, globals, object fields, or ownership.

If a dependency does not return normally, later actions are not established to occur. In particular, notification entry alone does not guarantee that the recorded field will be updated.

No atomic update, concurrency protocol, or rollback mechanism is established.

## Dependencies

| Opaque function ID | Invocation condition | Supplied arguments | Result handling |
|---|---|---|---|
| F-2706939fe3c5 | Captured recorded value is not `1` | Original object-base argument; value obtained by the conditional global read | Exactly `1` causes return `0x58`; other results permit continuation |
| F-cd0542290b1c | Captured values differ and guard has not terminated processing | Current 32-bit address-bearing field plus `0x18`; low eight bits of captured requested value | Return value is not inspected |

The guard’s second argument must not be specified as an established address of the global datum. The supplied expression evaluates that datum to obtain an argument value; the value’s meaning remains unknown.

The notification’s first argument is derived from the address-bearing field rather than passed unchanged. Whether it identifies a subobject, resource, or another kind of entity is unknown.

The dependencies’ complete interfaces, memory requirements, ownership behavior, global effects, and abnormal-return behavior remain **documented_only**.

## Acceptance examples

These examples are conditional contract illustrations, not executed tests. Unless an example states otherwise, assume valid accesses, unproblematic address arithmetic, normally returning dependencies, and no dependency-induced changes.

### Recorded `1`, requested `1`

Both 16-bit fields are read. The global datum is not read for the guard, and the guard is not called. The captured values are equal, so there is no notification, no address-bearing field read, and no write. The result is zero.

### Recorded `1`, requested `0x1234`

The global read and guard call are skipped. The current address-bearing field is read, and notification receives its value plus `0x18` together with numeric value `0x34`. After notification returns, the recorded field becomes `0x1234`. The result is zero.

### Recorded `2`, requested `2`, guard result `1`

After reading both fields, the function reads the global datum and supplies that value to the guard. The guard is called despite equality of the captures. The function returns `0x58` without reading the address-bearing field, notifying, or directly writing an object field.

### Recorded `2`, requested `2`, guard result `0`

The global read and guard call occur. Equality of the captures then produces a zero return with no notification or direct write.

If the guard changed the current requested field to `3`, this conclusion would remain unchanged: the equality decision uses the captured value `2`.

### Recorded `2`, requested `3`, guard result `2`

The global datum is read before guard entry. A guard result of `2` permits continuation. Notification receives numeric value `3`, the recorded field is subsequently written with `3`, and the result is zero.

### Different full values with the same low byte

With recorded `0x0001` and requested `0x0101`, the guard and global read are skipped. The full 16-bit comparison detects a difference even though the low bytes match. Notification receives numeric value `1`; the final recorded-field value is `0x0101`.

### Guard changes object fields and the global datum

Suppose the captures are recorded `2` and requested `0xABCD`. The function reads the global datum and passes that observed value to the guard.

The guard then changes the global datum, changes the requested field to `7`, replaces the address-bearing field with another valid value, and returns `0`.

The function does not reread the global datum or requested field. Notification uses the replacement address-bearing value plus `0x18` and numeric value `0xCD`. The subsequent recorded-field write stores `0xABCD`, not `7`.

### Notification modifies the destination

If notification changes the recorded field and returns with that field still writable, this function overwrites the notification’s value with the captured requested value. The notification’s return value does not suppress that write.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the two 16-bit fields, the bypass value `1`, and status `0x58`.
- The global datum’s width, concrete value, mutability, representation, and interpretation.
- Whether the global datum’s value has any address-like meaning.
- Complete dependency contracts, including global preservation, ownership, lifetime effects, and failure modes.
- Object layout beyond the three accessed fields.
- Original declared interface types and calling-convention details not established by the representation.
- Byte order, alignment requirements, and behavior of invalid memory accesses.
- Overflow behavior for address formation, including the addition of `0x18`.
- Behavior under concurrent mutation or abnormal control transfer.
- Whether zero corresponds to application-level success.

These unknowns do not weaken the established distinctions between captured object values, the conditional global read, the later address-bearing field read, and the final 16-bit write.

## Evidence

This contract is based on static inspection of the supplied function body and opaque dependency mapping. No runtime tests have been performed.

The static evidence establishes the field offsets and access widths, the order of the initial reads, the guard bypass for captured recorded value `1`, and the conditional global read supplying the guard’s second argument. That global read occurs after both initial field reads and before guard entry; its result is an argument value, not an established address of the datum.

The evidence also establishes the exact guard-result comparison, full 16-bit equality decision, conditional address-bearing field read, addition of `0x18`, low-eight-bit notification value, post-notification 16-bit replacement, and normal return values.

The application role names are inferred intent. Dependency internals, broader invariants, ownership guarantees, arithmetic boundary behavior, and runtime outcomes are not established by the supplied body.
