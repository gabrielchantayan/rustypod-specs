<!-- Trusted provenance: F-4c955ebbbc0d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4c955ebbbc0d

## Purpose

F-4c955ebbbc0d conditionally updates a one-byte state field and returns the field value captured after that decision.

The update recognizes a four-state cycle. Under the conventional ASCII interpretation, the cycle is:

**`*` → `+` → `,` → `-` → `*`**

A requested value is accepted only when it is the immediate successor of the current state. All other requests leave the field unchanged by this function.

A separate control argument enables two dependency calls surrounding the state access. Their placement is consistent with an enter/leave protocol, possibly synchronization, but that purpose is inferred rather than established. This contract does not assume that either dependency preserves memory, preserves globals, acquires a lock, or releases a resource.

## Interface

### Inputs

The function has three integer-typed inputs:

| Input | Meaning established by the supplied source |
|---|---|
| Object address | Address-like integer used to locate the state byte and the dependency argument. |
| Requested value | Integer compared against one of four numeric successor values. |
| Bracketing control | Zero disables both dependency calls; any nonzero value enables both calls on a normally completing execution. |

The source represents the object address as an integer rather than a typed object pointer. Callers must nevertheless supply an address suitable for the accesses described below.

The supplied representation does not independently establish the target ABI, integer width, pointer width, or signed-address arithmetic rules. The numeric constants and offsets fit within 32 bits, but that alone does not prove a 32-bit interface. Those widths remain **documented_only**.

### Relevant object layout

Only two offsets from the object address are visible:

| Offset | Use |
|---|---|
| `0xFC` / 252 | Address passed to each enabled dependency. No direct access to the storage at this offset is shown. |
| `0x104` / 260 | Address of the one-byte state field, directly read and conditionally written. |

The state field begins eight address units after the address passed to the dependencies. This does not establish an eight-byte dependency object, its type, or its boundaries.

The state access uses a character-sized unit. The intended byte-oriented interpretation is clear from the supplied representation, but the underlying target’s character width is not independently documented.

### Output

On normal completion, the function returns the second value read from the state field, converted from the source’s plain character type to its integer return type.

For the four ordinary ASCII cycle values, the result is the corresponding positive integer: 42, 43, 44, or 45.

For other stored values, the exact integer result can depend on plain-character signedness and target character width. In particular, the supplied source does not resolve whether an eight-bit value such as `0xFF` returns as −1 or 255.

There is no separate success flag, rejection flag, or dependency-status result.

## Behavior

### Preconditions

For the directly expressed behavior to complete normally:

- The object address plus `0x104` must identify readable storage.
- If a transition is accepted, that same location must be writable.
- Address calculations must be valid under the actual execution model.
- When bracketing is enabled, each dependency must accept the object address plus `0xFC` and return normally.
- Any synchronization, lifetime, or exclusion requirements imposed by the surrounding program must already be satisfied.

No null check, bounds check, state-validation error, or address-overflow check is present.

### Observable ordering

The established sequence is:

1. If bracketing control is nonzero, invoke **F-338d16f9ce72** with the object address plus `0xFC`.
2. Read the state byte at the object address plus `0x104`.
3. Compare that state against the recognized predecessor states and compare the requested integer against the associated successor value.
4. If and only if a recognized pair matches, write the requested value, converted to the character type, to the state field.
5. Read the state field again, regardless of whether a write occurred.
6. If bracketing control is nonzero, invoke **F-1333331b2da1** with the object address plus `0xFC`.
7. Return the value captured by the second state read, converted to the integer return type.

This sequence describes the source-level accesses and calls. It does not establish volatile accesses, atomicity, hardware memory ordering, or the exact number of machine-level loads.

The second read is significant: the return value is taken from the field after the update decision, not directly from the requested argument or necessarily from the first read.

### Accepted transitions

The source compares the current state with character literals and the requested value with numeric constants:

| Current-state character | Required requested integer | Directly written value |
|---|---:|---:|
| `*` | `0x2B` / 43 | 43 converted to character |
| `+` | `0x2C` / 44 | 44 converted to character |
| `,` | `0x2D` / 45 | 45 converted to character |
| `-` | `0x2A` / 42 | 42 converted to character |

On an ASCII-compatible target, these are exactly the four cycle transitions described in Purpose.

The comparisons use the full requested integer. The function does not first truncate that integer to a byte. Consequently, a request whose low byte equals an accepted successor is still rejected unless the entire integer equals the required constant.

The accepted numeric values fit in the ordinary signed eight-bit character range. Their conversion for storage therefore introduces no signedness ambiguity on an eight-bit target.

### Rejected requests and unrecognized states

A request is rejected without a direct state write when:

- It requests the current state again.
- It skips a state in the cycle.
- It requests a predecessor instead of the successor.
- It is any other integer not equal to the required successor constant.
- The current state is not one of the four recognized characters.

Rejection does not produce an error result. The function still performs its second state read, makes the trailing dependency call if enabled, and returns the captured value.

There is no normalization of an unrecognized state and no default transition into the cycle.

### Effects of dependency execution

When bracketing is enabled, the first state read occurs after F-338d16f9ce72 returns. If that dependency changes the state field, the transition decision uses the changed value.

The final state read occurs before F-1333331b2da1 is invoked. If that dependency subsequently changes the field, the function still returns the previously captured value. The returned integer therefore need not equal the field’s value at function exit.

No dependency result is inspected. The supplied source provides no recovery path if either dependency fails to return normally.

## Side effects

### Direct effects

The function directly performs:

- Two source-level reads of the state field on a normally completing execution.
- One state-field write for an accepted transition.
- No state-field write for a rejected request.
- Either zero dependency calls or the ordered pair of calls described above.

There is no directly expressed allocation, deallocation, ownership transfer, object construction, or object destruction.

The address passed to a dependency is an address within or relative to the supplied object. Passing it does not, by itself, establish borrowing rules, ownership transfer, retention permission, or release obligations.

### Indirect effects and concurrency

The dependencies’ effects are unknown. They may affect the state field, other object storage, globals, or external state; this function’s body does not constrain those possibilities.

Disabling bracketing removes both dependency calls. It does not otherwise change the transition rule or eliminate either state read.

The read/compare/write sequence is not established as an atomic transition. Dependency names, placement, and shared arguments are insufficient to prove synchronization. Concurrent access requirements and behavior remain **documented_only**.

## Dependencies

| Opaque function ID | Invocation condition | Argument | Position |
|---|---|---|---|
| **F-338d16f9ce72** | Bracketing control is nonzero | Object address plus `0xFC` | Before the first state read |
| **F-1333331b2da1** | Bracketing control is nonzero and execution reaches the trailing call | Object address plus `0xFC` | After the second state read, before return |

Each dependency is invoked once on a normally completing enabled execution.

Their return values, if any, do not participate in the transition decision or returned state value. Their actual signatures, memory effects, error mechanisms, lifetime requirements, and synchronization guarantees are not established.

There is no visible conditional cleanup path ensuring that the trailing dependency runs if execution stops during the first dependency or during a state access.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Character examples assume an ASCII-compatible target and valid object storage. Unless explicitly stated otherwise, no outside actor or dependency changes the state.

| Initial state | Request | Bracketing control | Required direct behavior and result |
|---|---:|---:|---|
| `*` | 43 | 0 | Write `+`; return 43; invoke neither dependency. |
| `+` | 44 | 0 | Write `,`; return 44. |
| `,` | 45 | 0 | Write `-`; return 45. |
| `-` | 42 | 0 | Write `*`; return 42. |
| `*` | 42 | 0 | Perform no write; return 42. |
| `*` | 44 | 0 | Reject the skipped transition; perform no write; return 42. |
| `+` | 42 | 0 | Reject the backward transition; perform no write; return 43. |
| `*` | 299 (`0x12B`) | 0 | Perform no write despite the low byte being `0x2B`; return 42. |
| `A` | 43 | 0 | Perform no write because the current state is unrecognized; return 65. |
| `*` | 43 | −1 | Invoke the leading dependency, accept the transition, capture 43, invoke the trailing dependency, and return 43. |

Additional dependency-sensitive criteria:

- If the state begins as `*`, the request is 44, and the leading dependency changes the state to `+`, the function accepts the `+` to `,` transition. It does not decide using the state that existed before the dependency call.
- If the second state read captures `+` and the trailing dependency changes the stored state to `-`, the function returns 43 while the field may contain 45 at exit.
- If bracketing is enabled and the request is rejected, both dependency calls still occur on normal completion.
- If an unrecognized eight-bit state is `0xFF` and bracketing is disabled, no write occurs. The integer result depends on the unresolved character signedness; neither −1 nor 255 is universally required by the available evidence.

## Uncertainties

### Inferred intent

The four recognized states appear intended to form an ASCII punctuation cycle. The dependency placement appears intended to bracket access to shared state.

Neither interpretation establishes broader semantics such as progress, authorization, locking, or thread safety.

### Documented-only unknowns

The following remain unresolved:

- Target integer, pointer, and character widths.
- Plain-character signedness.
- Execution character encoding.
- Object type, total size, alignment, allocation origin, and lifetime.
- Meaning and extent of the storage at offset `0xFC`.
- Dependency side effects, ownership rules, failure behavior, and synchronization guarantees.
- Behavior for invalid addresses or invalid address arithmetic.
- Whether surrounding code guarantees exclusive access.
- Whether external changes can occur between the two state reads.

These unknowns must not be replaced with assumed locking, memory ownership, error handling, or byte-truncation behavior.

## Evidence

The supplied static body establishes the two relative addresses, the optional leading and trailing dependency calls, the four guarded transitions, the conditional character write, the unconditional second state read, and the return conversion.

It does not establish dependency implementations, a target ABI, runtime memory validity, or observed concurrent behavior.

No runtime tests have been performed. The acceptance examples are static contract cases, not claims of measured execution.
