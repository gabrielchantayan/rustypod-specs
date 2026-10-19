<!-- Trusted provenance: F-0f1d6e8bee73; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0f1d6e8bee73

## Purpose

F-0f1d6e8bee73 conditionally updates a field in caller-supplied storage. The update is accepted only when the storage pointer is non-null, its leading integer equals a reference value held in global storage, and the requested value is either 1 or 2.

On acceptance, the function stores the requested value in the eighth integer slot of the supplied storage and returns zero. On rejection, it returns hexadecimal `0x1A`, decimal 26, without an explicit write to the supplied storage.

The leading-integer comparison appears intended to recognize an eligible object. That interpretation is inferred intent, not an established type, validity, ownership, or security guarantee.

## Interface

### Inputs

The function takes two arguments:

- **Storage pointer:** a pointer interpreted as addressing integer slots. It may be null. When non-null, the function attempts to read the first slot before deciding whether the requested value is acceptable.
- **Requested value:** an integer accepted only when its value is exactly 1 or exactly 2.

The recovered interface uses integer slots represented as 32-bit values and a 32-bit return value. Under that representation:

- The leading comparison field occupies bytes 0–3 relative to the supplied address.
- The destination field occupies bytes 28–31.
- The destination is the eighth integer slot, counting the leading field as the first.

The supplied material does not independently establish pointer width, byte order, or an ABI alignment requirement. The byte offsets above follow the recovered 32-bit integer layout; they are not evidence of a complete object definition.

### Output

The normal return values are:

| Value | Meaning |
|---|---|
| 0 | All acceptance conditions passed and the requested value was stored. |
| 26 | At least one acceptance condition failed. |

No other normal return value is present in the supplied body. Invalid memory access is not shown as being converted into either return value.

### Memory preconditions

A null storage pointer is handled without dereferencing it.

For a non-null pointer, the leading integer must be readable under the applicable execution environment. An unsupported requested value does not remove this requirement: the leading comparison precedes the requested-value check.

For an accepted request, the eighth integer slot must also be writable. The function does not establish allocation size, mapping validity, object lifetime, or write permission before accessing that slot.

The supplied body does not require reads from the intervening slots. A conventional contiguous allocation covering at least 32 bytes would cover the accessed locations under the recovered layout, but the body itself establishes only the particular read and write requirements.

## Behavior

### Acceptance rule

The request is accepted exactly when all three conditions hold:

1. The storage pointer is non-null.
2. The first integer in that storage equals the integer read from the global reference location.
3. The requested value is 1 or 2.

The comparison is equality against the value read from the global location, not against a literal constant embedded in the function body. The supplied material does not establish that this global value is immutable.

Neither range conversion nor normalization is shown. Values other than 1 and 2 are rejected rather than mapped to one of those values.

### Observable access ordering

The source establishes the following gating relationships:

- The null-pointer check precedes any access through the storage pointer.
- If the pointer is null, neither the leading-field comparison nor the global-reference read is reached.
- For a non-null pointer, the leading-field value and global reference value participate in an equality comparison.
- If that comparison fails, the requested-value condition is not reached.
- The destination write occurs only after all acceptance conditions have passed.
- A successful return follows the destination assignment.

The relative order of the leading-field read and the global-reference read is not established by the equality expression. The contract therefore does not require one of those reads to occur before the other.

These relationships describe the supplied source-level behavior. They do not specify machine instruction selection, speculative execution, or an exact count of physical memory transactions.

### Accepted request

When all conditions pass, the function assigns the requested value to the eighth integer slot and returns zero.

The destination’s previous contents are not tested. In particular:

- Acceptance does not depend on whether the destination already contains 1 or 2.
- Acceptance does not depend on the destination matching the requested value.
- Repeating an accepted request still reaches the assignment.
- No previous destination value is returned.

There is no arithmetic transformation of the requested value. No increment, accumulation, truncation from a wider argument, or comparison against the old destination value appears in the supplied body.

### Rejected request

If the pointer is null, the leading value does not match, or the requested value is unsupported, the function returns 26 without an explicit store.

Rejection does not imply that no memory was read. A non-null pointer causes the leading-field comparison to be evaluated even when the requested value is unsupported.

The function does not distinguish rejection causes in its return value. A caller cannot determine from 26 alone whether the pointer was null, the leading field mismatched, or the requested value was unsupported.

### Exceptional access conditions

A non-null pointer is not necessarily a valid pointer. The body contains no explicit handling for unreadable leading storage or an unwritable destination.

If a required access cannot complete, the resulting behavior depends on the execution environment and is not specified here. In particular, such an access must not be described as a normal rejection returning 26.

## Side effects

On the accepted path, the only explicit store in the supplied body targets the eighth integer slot.

On a rejected path, the supplied body contains no explicit store.

The function contains no allocation, deallocation, ownership-transfer operation, or dependency call. It does not explicitly retain the supplied pointer. The caller remains responsible for ensuring that the storage is valid for the accesses reached.

The global reference location is read but is not explicitly assigned by the function. However, the supplied material does not establish non-aliasing. If the destination address aliases that global location, the destination store would also modify the reference storage. The function contains no check preventing such an arrangement.

Likewise, preservation of unrelated bytes is a statement about this function’s explicit accesses, not a guarantee against concurrent modification, memory-mapped side effects, or other execution-environment behavior.

No lock, atomic operation, barrier, or retry mechanism is shown. Thread safety and consistency under concurrent changes are not established.

## Dependencies

No callable dependencies appear in the supplied body, and the supplied dependency map is empty.

The function has one visible external data dependency: an integer-valued global reference location used for the leading-field equality check.

The following properties of that location are unspecified:

- Its initialization procedure.
- Its actual value at any invocation.
- Whether another execution context may modify it.
- Whether its storage has special read semantics.
- Whether caller-supplied storage can alias it.

No callee behavior, memory ownership convention, or global-preservation guarantee is needed to explain the visible body.

## Acceptance examples

These examples are conditional static cases, not runtime observations. Unless stated otherwise, assume the necessary accesses are valid and no concurrent mutation occurs.

| Conditions | Required normal result |
|---|---|
| Null pointer; requested value 1 | Return 26. Do not dereference the pointer or reach the global-reference comparison. |
| Null pointer; requested value 2 | Return 26 with the same access behavior as any other null-pointer request. |
| Non-null pointer; leading value differs from the global reference; requested value 1 | Return 26 without writing the destination. |
| Non-null pointer; leading value matches; requested value 1 | Store 1 in the eighth integer slot, then return zero. |
| Non-null pointer; leading value matches; requested value 2 | Store 2 in the eighth integer slot, then return zero. |
| Non-null pointer; leading value matches; requested value 0 | Return 26 without writing the destination. |
| Non-null pointer; leading value matches; requested value −1 or 3 | Return 26 without writing the destination. |
| Non-null pointer; leading value matches; destination already equals the requested accepted value | Reach the destination assignment and return zero. No old-value check suppresses the assignment. |

Additional boundary cases:

- **Unsupported request with unreadable leading storage:** normal rejection is not guaranteed. The leading comparison is reached before the unsupported request is rejected.
- **Matching leading field with unwritable destination:** an accepted requested value reaches the failing write; the body supplies no alternate error result.
- **Unwritable destination with a mismatching leading field:** no destination write is reached, assuming the leading comparison itself completes.
- **Destination aliases the global reference:** a successful assignment also changes the aliased reference storage. No separate reference write is required for that consequence.

## Uncertainties

### Established static evidence

The visible body establishes the null guard, the equality gate, acceptance of exactly 1 and 2, one destination assignment on acceptance, and normal returns of zero or 26. It shows no calls and no explicit ownership operations.

### Inferred intent

The leading field likely serves as an object-recognition marker, and the destination likely represents a selectable state or mode. Neither interpretation supplies additional behavior. No state-transition policy beyond accepting the two requested values is visible.

### documented_only

The following remain unspecified rather than assumed:

- The semantic meanings of requested values 1 and 2.
- The semantic name of return value 26.
- The complete object layout and allocation size.
- The global reference value and its lifetime.
- Pointer width, byte order, and ABI alignment requirements.
- Volatility, atomicity, concurrency guarantees, and hardware access behavior.
- Behavior after an invalid memory access.
- Any caller-side validation or ownership discipline.

## Evidence

This contract is based solely on the supplied function body and empty dependency map.

The evidence supports a short-circuited non-null check and leading-value comparison, followed by a two-value acceptance condition, one indexed integer assignment, and two normal return outcomes. The destination offset is derived from the eighth-slot access under the recovered 32-bit integer layout.

No implementation inspection beyond the supplied material, runtime execution, or tests have been performed. The acceptance examples state consequences of the static evidence, not observed execution results.
