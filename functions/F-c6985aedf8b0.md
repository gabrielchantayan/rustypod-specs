<!-- Trusted provenance: F-c6985aedf8b0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c6985aedf8b0

## Purpose

F-c6985aedf8b0 performs a two-phase traversal over data reached through a pointer stored in a caller-supplied object. It first invokes a forward-traversal dependency until the current cursor addresses a zero-valued character. It then invokes a second dependency repeatedly, comparing each returned integer with a requested value.

On a match, the function returns the number of reverse-phase attempts still available after consuming the matching attempt. If no match occurs within the allowance established by the forward phase, it returns −1.

**Inferred intent:** If the dependencies implement mutually compatible forward and backward traversal over logical elements, the function finds the last occurrence of a requested value and returns its zero-based logical position.

**Established limitation:** That interpretation depends on dependency behavior absent from the supplied evidence. The function itself establishes a traversal count and a sequence of comparisons, not a particular character encoding, element size, or backward-decoding rule.

## Interface

### Inputs

The function takes two integer-valued arguments:

1. **Object address.** Used to obtain a stored character pointer from the location four address units beyond the supplied address. The visible interface does not pass the character buffer directly.
2. **Requested value.** Compared for integer equality with the result of each invocation of F-13bcc08bc7a3.

There is no explicit buffer length, object size, encoding selector, error-output parameter, or allocator parameter.

### Required object layout

The only object field accessed directly is:

| Relative location | Interpretation | Access |
|---|---|---|
| Address + 4 | Stored pointer to character data | Read once |

The evidence does not establish the object’s total size, any preceding field’s meaning, the pointer field’s alignment, or the pointer representation’s width. In particular, an offset of four does not by itself establish a four-byte pointer.

The field must be readable using the target’s pointer representation. A valid non-null field value must initially designate readable character storage. Further validity requirements depend on how the dependencies change the cursor.

The supplied object address is not checked for null or validity before the field read. A null character pointer stored in a valid object is handled explicitly; an invalid object address is not.

### Integer and storage widths

The visible interface and traversal counter use the ordinary integer type shown in the supplied representation. Neither a fixed-width declaration nor independent target ABI information is available.

Accordingly:

- The exact integer width and signed arithmetic model are **documented_only**.
- The requested value is compared as an integer; no explicit narrowing to a character-sized value appears.
- The counter begins at zero, increases by one per completed forward call, and decreases by one per reverse attempt.
- The character test examines whether the current character is zero. It does not establish the signedness of character values or their encoding.
- The field displacement is four address units. On a conventional byte-addressed target, this is four bytes; the excerpt does not independently establish that target property.

A conforming description must not silently assume that the counter, addresses, and stored pointers all have the same width.

### Output

The return value is:

- **−1** when the stored character pointer is initially null.
- **−1** when the forward phase completes with no counted calls.
- **−1** when every permitted reverse attempt returns a value unequal to the requested value.
- Otherwise, the counter value remaining immediately after decrementing it for the successful reverse attempt.

With defined arithmetic and a forward count of \(N\), a successful return lies between zero and \(N-1\), inclusive. The first reverse attempt can return \(N-1\); the final permitted attempt can return zero.

No separate status distinguishes null data, empty data, or an unsuccessful search.

## Behavior

### 1. Obtain the initial cursor

The function reads the stored character pointer at object address + 4 and copies it into a local cursor slot.

If this copied pointer is null, the function returns −1 without directly reading character data or invoking either dependency.

The object field is not visibly reread. Later changes to that field therefore do not directly replace the local cursor. This does not imply isolation from changes to shared character storage or dependency-managed state.

### 2. Establish the traversal allowance

For an initially non-null cursor, the function initializes its counter to zero and examines the character at the current cursor.

While that character is nonzero, it performs these observable actions in order:

1. Invokes F-d3922c7fe0ec, supplying the address of the local cursor slot.
2. Increments the counter by one after that invocation returns.
3. Reads the character at the cursor value then present in the slot to determine whether traversal continues.

The dependency’s return value does not influence visible control flow. Its changes to the cursor slot can influence the next character read.

The counter therefore counts **completed dependency invocations**, not directly bytes, characters, code points, or object elements. Equating that count with any such unit requires a dependency contract.

The function does not independently advance the cursor or require it to move monotonically. It contains no explicit iteration limit and no check for a null cursor after a dependency invocation.

Consequences include:

- An initially zero-valued character produces a count of zero and no forward calls.
- A dependency that leaves the cursor at a nonzero character can cause repeated calls without progress.
- A dependency that supplies an invalid cursor can cause the next direct character read to fail.
- Termination is determined by the character observed after each call, including any changes to storage made by that call.

### 3. Search using the second dependency

Once the forward phase observes a zero-valued character, the function retains the current cursor and the accumulated count. It does not visibly reset the cursor to its original value.

Before each reverse-phase invocation, it checks whether the counter is at least one.

- If not, it returns −1.
- Otherwise, it decrements the counter and invokes F-13bcc08bc7a3 with the address of the same local cursor slot.

The integer returned by that dependency is then compared with the requested value:

- Equality causes immediate return of the decremented counter.
- Inequality causes another allowance check and, if permitted, another attempt.

No direct character read appears in this phase. The dependency controls any cursor movement or underlying data access associated with each attempt.

For a forward count of \(N\), normal execution permits at most \(N\) calls to F-13bcc08bc7a3. A match on attempt \(k\), counting attempts from one, returns \(N-k\). If all \(N\) results differ from the requested value, the next allowance check returns −1.

### 4. Error and exceptional behavior

No explicit error result from either dependency is recognized.

In particular:

- F-d3922c7fe0ec has no visibly checked result.
- Any integer returned by F-13bcc08bc7a3 participates in equality comparison, including a value that its own contract might designate as an error.
- The function does not visibly validate cursor progress, encoding validity, buffer boundaries, or dependency success.
- There is no visible recovery path for invalid memory access, nonreturning dependency calls, or arithmetic outside the supported counter range.

The ordinary return contract therefore requires readable accesses, returning dependencies, and representable counter arithmetic.

## Side effects

### Directly established effects

The function directly:

- Reads the pointer field at object address + 4 once.
- Reads the character addressed by the local cursor before each forward-phase continuation decision.
- Updates its local counter and cursor storage.
- Passes the address of that local cursor storage to both dependencies.

It contains no visible direct write to the caller’s object field or character storage.

### Dependency-mediated effects

Both dependencies receive a writable reference to the local cursor slot. Their implementations are unavailable, so this contract does not assert that they:

- Modify only that slot.
- Preserve globals or other shared state.
- Leave character storage unchanged.
- Preserve the object containing the original pointer.
- Avoid allocation, release, I/O, callbacks, or other externally observable effects.

The visible call order is F-d3922c7fe0ec repeated during the forward phase, followed by zero or more F-13bcc08bc7a3 calls. A match stops subsequent calls immediately.

### Ownership and lifetime

No allocation, release, or ownership-transfer operation is directly visible in this function.

The evidence does not establish whether either dependency owns or releases memory reachable through the cursor. Required storage lifetimes must therefore be resolved from their contracts.

The cursor slot itself is local to this invocation. A dependency that retains its address would need a separate lifetime guarantee not established here. No guarantee of safe reentrancy or concurrent mutation follows from the visible function alone.

## Dependencies

| Opaque ID | Visible use | Required knowledge not supplied |
|---|---|---|
| F-d3922c7fe0ec | Called once for each nonzero-character forward iteration; receives the address of the cursor slot; its result is not used | Cursor movement, accessed extent, progress, encoding rules, errors, ownership, shared-state effects |
| F-13bcc08bc7a3 | Called after decrementing a positive allowance; receives the same cursor-slot address; its integer result is compared with the requested value | Traversal direction, returned-value meaning, boundary handling, errors, ownership, shared-state effects |

For the inferred “last logical occurrence” interpretation, additional conditions are necessary:

- Each forward call advances exactly one logical element.
- The forward phase ends at the boundary following the final element.
- Each second-phase call retrieves exactly one preceding element in reverse order.
- The returned integer represents the value intended for comparison.
- Relevant contents remain consistent across both phases.

These are conditional requirements, not established dependency behavior.

## Acceptance examples

These examples specify conditional outcomes from observable dependency behavior. They do not claim runtime execution.

### Null stored pointer

Given a readable object whose pointer field at offset four is null:

- Return −1.
- Invoke neither dependency.
- Perform no direct character read through that null value.

### Initially empty data

Given a non-null initial cursor addressing a zero-valued character:

- Make no forward calls.
- Make no second-phase calls.
- Return −1.

Thus, a requested value of zero does not make an initially empty sequence match.

### Match on the first second-phase call

Given three completed forward calls followed by an observed zero character, and a first F-13bcc08bc7a3 result equal to the requested value:

- Make one second-phase call.
- Return 2.
- Make no further dependency calls.

### Match on the final permitted call

Given the same forward count of three, with the first two comparison results unequal and the third equal:

- Make three second-phase calls.
- Return 0.

### No match within the allowance

Given a forward count of three and three unequal second-phase results:

- Return −1 after those three calls.
- Do not make a fourth second-phase call, regardless of the cursor’s current position.

### Conditional last-occurrence interpretation

Assume the dependencies traverse a stable logical sequence containing values 7, 9, 7 from beginning to end and then back toward the beginning, returning each preceding value.

Under those assumptions:

- Requesting 7 returns 2.
- Requesting 9 returns 1.
- Requesting an absent value returns −1.

These results establish logical positions only under the stated traversal assumptions.

### Dependency error value equals the request

If F-13bcc08bc7a3 returns an integer designated by its own contract as an error, and that integer equals the requested value, this function’s visible comparison treats it as a match. No local error discrimination overrides equality.

### Nonprogressing forward traversal

If every forward call returns while leaving the cursor at readable nonzero data, the visible termination condition is never satisfied. No bounded completion result is established; counter-range exhaustion is an additional unresolved hazard.

## Uncertainties

The following remain **documented_only**:

- Exact integer, address, pointer, and character widths.
- Target arithmetic behavior if the forward count exceeds the representable range.
- Full object layout and alignment requirements.
- Buffer extent and whether zero termination is guaranteed.
- Whether either dependency moves forward or backward by bytes or logical elements.
- Encoding rules and malformed-input handling.
- Dependency error conventions and nonreturning behavior.
- Dependency effects on globals, storage contents, allocation, ownership, and lifetimes.
- Concurrency, aliasing, and reentrancy guarantees.
- Whether the inferred last-occurrence interpretation is the intended public API.

The function’s name and unavailable callers provide no usable evidence for resolving these points. In particular, neither an offset of four nor a character-pointer interpretation is sufficient to prove a complete 32-bit ABI or a specific text encoding.

## Evidence

The supplied static representation establishes:

- One initial pointer-field read at object address + 4.
- An explicit null-pointer path returning −1.
- A forward phase controlled by direct zero-character tests.
- One counter increment after each completed F-d3922c7fe0ec call.
- Preservation of the resulting cursor into the second phase.
- A positive-allowance check and decrement before each F-13bcc08bc7a3 call.
- Integer equality as the sole visible match criterion.
- Immediate return of the remaining count on equality.
- Return of −1 when the allowance is exhausted.

The traversal semantics, ownership rules, and target-width details are not established by that evidence. No runtime tests have been performed.
