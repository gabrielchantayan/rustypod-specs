<!-- Trusted provenance: F-1affccecb252; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1affccecb252

## Purpose

F-1affccecb252 performs an ordered initialization sequence against a manager-state object selected through a global pointer. It obtains two initial values, invokes two interior-state initializers, requests entry storage, initializes entries at a 32-byte stride, prepares an auxiliary result, requests secondary storage, and makes two named registration calls.

This is not an atomic operation. Explicit state writes occur between dependency calls, and no rollback or success-validation branch is visible.

The names “EventManager” and “TimerEventManager” suggest event-management intent. They do not establish event delivery, scheduling, timer behavior, background execution, or successful registration.

## Interface

### Inputs

The function accepts two signed 32-bit integers:

| Input | Established uses |
|---|---|
| Entry count | Passed to the second F-50934d32ec63 call; shifted left by five for a storage request; used as the signed upper bound of the entry loop |
| Secondary count | Written to state offset +12; shifted left by two for another storage request |

There is no explicit input validation. In particular, the function does not visibly reject negative counts, zero counts, excessive sizes, or arithmetic overflow.

The function also reads externally supplied global state:

- A pointer selecting the manager-state object.
- A value supplied to each entry-initialization call.
- A pointer to a location holding a shared registration argument.
- Two registration-associated values, one for each registration.

The distinction between a global pointer and the contents of its target is observable. The manager-state pointer and shared-argument pointer are each captured at a particular point; other global values are obtained later as described below.

### Output

The function has no explicit return value or status result.

Observable outputs are its dependency calls, direct state writes, and any effects of those dependencies. Normal return alone does not establish that storage requests, initialization, or registrations succeeded.

### State representation

The supplied representation uses four-byte state slots. The explicit writes cover **nine distinct 32-bit fields**, at offsets +0 through +32 in four-byte increments.

| Byte offset | Explicitly written value |
|---|---|
| +0 | Result of the first F-50934d32ec63 call |
| +4 | Result of the second F-50934d32ec63 call |
| +8 | Result of the entry-storage request |
| +12 | Secondary count |
| +16 | Zero |
| +20 | Result of the secondary-storage request |
| +24 | Result of F-aa3c8fe3376b |
| +28 | Zero |
| +32 | Zero |

Exactly **three fields receive explicit zero writes**: +16, +28, and +32. Other returned or supplied values could happen to be zero, but that is not an explicit clearing operation.

The function additionally passes the addresses at offsets +36 and +44 to F-b00260d1087a. Those calls are not additional direct field writes by F-1affccecb252. Their write extents are unknown; the eight-byte spacing between the supplied addresses does not establish two eight-byte objects or nonoverlapping effects.

Stored results and the entry-address progression are represented with 32-bit quantities. A native ABI, broader pointer model, and complete object layout are not established by this representation.

## Behavior

### Initial values and manager selection

First, the function calls F-50934d32ec63 without an explicit argument in the supplied representation.

After that call returns, it reads the global manager-state pointer, retains the selected pointer, and writes the returned value to offset +0. Consequently, a change to the global manager-state pointer made by the first dependency call can affect the selected object.

Next, it calls F-50934d32ec63 with the entry count and writes that result to offset +4 of the retained state object.

Later changes to the global manager-state pointer do not redirect the function’s remaining explicit state accesses. Dependencies can nevertheless modify the retained object, invalidate it, or alter other state; none is assumed to preserve memory or globals.

The differing visible argument lists for F-50934d32ec63 remain unresolved. They establish two observed call forms, not an overload, default argument, or known hidden parameter.

### Interior-state initialization

The function calls F-b00260d1087a with the retained state address plus 36 bytes, then with that address plus 44 bytes.

Both calls precede the entry-storage request. Their results are not explicitly stored or checked.

The addresses identify starting locations only. The function does not establish which bytes the dependency reads or writes, whether existing contents matter, or whether it retains either address.

### Entry-storage request and clears

The function forms a size expression by shifting the entry count left by five positions. It passes that expression and the constant 11 to F-638beded7699.

After the dependency returns, the function:

1. Writes its result to state offset +8.
2. Writes zero to offset +32.
3. Writes zero to offset +28.

The returned entry-storage value is also retained separately for the loop’s address progression. The loop does not reload offset +8 to obtain each entry address.

For entry counts from 0 through 67,108,863, the shifted size is representable as a nonnegative signed 32-bit value and equals the count multiplied by 32. Outside that range, portable signed-shift behavior is not established by the supplied representation. No truncation or wraparound policy is specified here.

There is no explicit check for a null result, failure sentinel, inadequate capacity, or unsuitable alignment.

### Entry initialization

The entry loop uses a signed 32-bit index beginning at one and continues while that index is less than or equal to the entry count.

For each executed iteration:

- The global entry-initialization argument is read.
- F-37efbe955e3e receives that value and the current retained entry address.
- The retained address is advanced by 32.
- The signed loop index is incremented.

For an ordinary positive count, subject to valid arithmetic and normal dependency returns, there are as many calls as the count. Their address arguments begin at the storage-request result and advance in 32-byte steps.

Two state distinctions matter:

- Changing state offset +8 during an entry call does not redirect the separately retained address progression.
- Changing the global entry-initialization argument during one call can affect the next call, because that global value is read again for each iteration.

For zero or negative entry counts, the initial signed comparison fails and no entry-initialization call occurs. The preceding size expression, storage request, and explicit clears are still present on the path to that comparison. For negative counts, this observation does not resolve the earlier signed-shift uncertainty.

The address increment occurs after every entry call, including the last. Arithmetic validity therefore includes the increment after the final used address, not merely the addresses passed to dependencies.

For an entry count of 2,147,483,647, the index increment after the iteration at that bound exceeds the signed 32-bit range. Finite termination is not established for that case. The earlier size expression is already outside the established nonnegative signed-shift range.

### Auxiliary preparation

After the entry loop, the function calls F-8157a180d901 with 60.

Its result is passed to F-aa3c8fe3376b together with 1024 and 1024. The result of F-aa3c8fe3376b is then written to state offset +24.

There is no intervening validation. The value written to +24 is the second dependency’s result, not necessarily the value produced by F-8157a180d901.

The arguments do not prove that a usable 60-byte object is allocated, that the two 1024 values are dimensions, or that either dependency transfers ownership.

### Secondary storage

The function next writes the secondary count to offset +12, then zero to offset +16.

It shifts the secondary count left by two positions and calls F-638beded7699 with that expression and 11.

After the call returns, it reads and retains the global pointer to the shared registration-argument location. It then writes the storage-request result to offset +20.

The capture order is observable: the shared-argument pointer is selected after the storage dependency returns and before the explicit +20 write. A dependency-induced change to that global pointer before the capture can therefore affect both later registration reads.

For secondary counts from 0 through 536,870,911, the size expression is representable as a nonnegative signed 32-bit value and equals four times the count. Negative or larger counts have unresolved signed-shift behavior.

There is no explicit loop initializing the secondary storage. Whether F-638beded7699 initializes it is unknown.

### Named registrations

The function makes two calls to F-195f5d43c91b in the following order:

| Argument position | First call | Second call |
|---|---|---|
| 1 | Name “EventManager” | Name “TimerEventManager” |
| 2 | First registration-associated global value | Second registration-associated global value |
| 3 | 0 | 0 |
| 4 | 16 | 16 |
| 5 | Contents read through the captured shared-argument pointer | Contents read again through the same captured pointer |
| 6 | 0 | 0 |
| 7 | 0 | 0 |

The shared-argument pointer is retained across the two calls, but its target’s contents are read separately for each call. If the first registration changes those contents, the second registration can receive the changed value.

If the first registration changes only the global pointer, the second read still uses the previously captured pointer. That pointer must remain usable; preservation is not guaranteed.

The second registration-associated global value is obtained for the second call, after the first call returns. Relative evaluation order among independent argument reads within either individual call is not established.

Neither registration result is explicitly checked. After the second call returns normally, the function returns without an explicit status.

## Side effects

### Direct effects

The function explicitly writes **nine distinct 32-bit state fields**, in this order:

| Write order | Offset | Value category |
|---|---|---|
| 1 | +0 | First initial result |
| 2 | +4 | Second initial result |
| 3 | +8 | Entry-storage result |
| 4 | +32 | Explicit zero |
| 5 | +28 | Explicit zero |
| 6 | +24 | Auxiliary result |
| 7 | +12 | Secondary count |
| 8 | +16 | Explicit zero |
| 9 | +20 | Secondary-storage result |

These are explicit write events, not guaranteed final field contents. Later dependencies may modify previously written fields.

Additional effects can occur through the two interior-state addresses, entry addresses, registration arguments, or global state accessed by dependencies. Their extent is unknown.

### Ownership and lifetime

No explicit release operation is visible.

Existing contents of several state fields are overwritten without an explicit disposal step. This does not establish a leak: ownership, validity, and release requirements for those contents are unknown.

The function does not establish whether dependencies retain passed addresses, consume objects, return aliases, transfer ownership, or release resources internally. Neither allocation-like terminology nor call ordering supplies those missing contracts.

### Preconditions and failure exposure

Defined execution requires the relevant memory accesses and dependency preconditions to be satisfied, including:

- Writable storage for the nine explicit state fields.
- Valid interior-state arguments for both F-b00260d1087a calls.
- Suitable entry arguments for every executed F-37efbe955e3e call.
- A readable captured shared-argument location at both registration reads.
- Valid size and address arithmetic along the executed path.

These conditions are not explicitly enforced.

There is no visible retry, rollback, exception handler, synchronization, or repeated-initialization guard. A dependency that does not return normally can leave earlier effects in place. A dependency that returns a failure-like value normally is not explicitly recognized as failing.

## Dependencies

| Opaque function ID | Established interaction | Unresolved contract |
|---|---|---|
| F-50934d32ec63 | Two early calls; results written to +0 and +4 | Meaning of results and differing argument lists |
| F-b00260d1087a | Called with state addresses +36 and +44 | Read/write extent, retention, initialization and ownership |
| F-638beded7699 | Two calls with shifted counts and 11 | Storage semantics, zero-size handling, failure values and initialization |
| F-37efbe955e3e | Receives a freshly read global value and successive entry addresses | Entry layout, accessed extent, retained references and failure behavior |
| F-8157a180d901 | Receives 60; result passed onward | Resource type, validity and ownership |
| F-aa3c8fe3376b | Receives the preceding result, 1024 and 1024; result stored at +24 | Argument units, transformation, aliasing and ownership |
| F-195f5d43c91b | Two seven-argument named calls | Registration effects, persistence, argument meanings and failure convention |

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless explicitly discussing an uncertainty, they assume valid arithmetic and memory accesses and normal returns from all dependencies.

### Positive counts

For entry count 3 and secondary count 5:

- The entry-storage request receives 96 and 11.
- Three entry-initialization calls use the returned starting value, that value plus 32, and that value plus 64.
- The retained address is advanced once more after the third call.
- Auxiliary preparation follows the three entry calls.
- The secondary count is written as 5 at +12.
- The secondary-storage request receives 20 and 11.
- “EventManager” precedes “TimerEventManager.”

Earlier-written fields need not retain their written values if later dependencies modify them.

### Explicit write accounting

On a complete normal execution, the direct-write trace contains nine field writes at offsets +0, +4, +8, +32, +28, +24, +12, +16, and +20.

Only the writes to +32, +28, and +16 are explicit zero writes. Passing +36 and +44 to a dependency does not increase the direct-write count.

### Zero counts

With entry count zero, the first storage-size argument is zero and no entry calls occur. The +32 and +28 clears, auxiliary preparation, secondary setup, and registrations still follow.

With secondary count zero, zero is written to +12, +16 is explicitly cleared, and the secondary-storage request receives zero and 11. The registrations are not skipped.

### Manager selection changes

If the first F-50934d32ec63 call changes the manager-state global pointer, the subsequent selection uses the changed pointer.

If the second call changes that global again, the +4 write and remaining explicit state accesses still use the previously retained pointer, provided it remains valid.

### Entry-data changes

For at least two entries, a first entry call that changes the global entry-initialization argument can affect the second call’s argument.

A change to state offset +8 alone does not change the retained entry-address progression.

### Registration-data changes

If the first registration changes the contents of the captured shared-argument location, the second registration reads those changed contents.

Changing only the global pointer does not redirect that second read.

### Failure-like storage results

A normally returned failure-like storage value is still stored without an explicit validity check. For a positive entry count, the first storage result is also used as the initial entry argument.

This establishes continuation without validation, not safe handling of invalid storage.

## Uncertainties

The following remain **documented_only**:

- Exact dependency signatures and native calling conventions.
- Full state-object extent, alignment, validity, aliasing, and lifetime.
- Signed-shift behavior outside the stated representable nonnegative ranges.
- Signed entry-address overflow and loop-index overflow behavior.
- Dependency effects on globals, manager state, and passed memory.
- Ownership, retention, release, and failure conventions.
- Meanings of constants 11, 16, 60, and 1024.
- Storage initialization and registration persistence.
- Thread safety, reentrancy, and repeated-initialization semantics.
- Semantic roles of fields beyond their observed data flow.

“Storage request,” “auxiliary preparation,” and “registration” describe inferred roles. They do not strengthen the unknown dependency contracts.

## Evidence

Established static evidence consists of the supplied function representation and opaque dependency mapping. It supports the call sequence, explicit arguments, four-byte field layout, nine distinct direct field writes, three explicit zero writes, signed loop comparison, 32-byte address progression, pointer-capture timing, and absence of explicit result validation or release operations.

Event-management intent and allocation-like dependency roles are inferences. Dependency internals and runtime outcomes remain unresolved.

No runtime tests have been performed. Acceptance examples specify conditional behavior rather than observed execution.
