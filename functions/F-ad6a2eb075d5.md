<!-- Trusted provenance: F-ad6a2eb075d5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad6a2eb075d5

## Purpose

F-ad6a2eb075d5 dispatches on a mode value and, in one mode, an event value. It either delegates processing, updates a 32-bit state word in a caller-supplied object, or stores a captured global value into that object.

The established behavior distinguishes event values `0x2f`, `0x3e`, and `0x5b` when the mode is exactly `1`. Other modes share a fallback path. Return values include `3`, `4`, and `13`, plus an unrestricted 32-bit result forwarded from a dependency.

The event values coincide with familiar character codes, but interpreting them as characters is inferred intent, not an established interface requirement. Likewise, the meanings of the state bits and numeric return values are unknown. They must not be labeled success, failure, consumption, or rejection without additional evidence.

## Interface

### Inputs and result

The function receives three arguments:

| Input | Established use |
|---|---|
| Object base | A 32-bit address-like value used to access object storage and form dependency arguments. |
| Mode | An integer compared with exactly `1`. All other values take the same local path. |
| Event | An integer compared with `0x2f`, `0x3e`, and `0x5b`, but only when mode equals `1`. |

The result is one 32-bit value. Directly produced results are `3`, `4`, and `13`. The default event branch returns the 32-bit result of F-3643422b0298 without local interpretation.

The supplied representation uses 32-bit integer and word operations. It does not establish a portable host-language signature, a larger-address-space interface, or an object’s complete type.

### Object storage

The following offsets are measured in bytes from the object base:

| Offset | Access or use |
|---|---|
| `+0x0c` | Read as a 32-bit value on the `0x3e` path; `4` is added before passing it to a dependency. |
| `+0x1c` | Its address is passed to dependencies. The embedded representation and extent are unknown. |
| `+0x20` | Read as a 32-bit argument on the `0x2f` path. |
| `+0x28` | Read and, on selected paths, written as a 32-bit state word. |
| `+0x38` | Written with a captured 32-bit global value on the fallback path. |

These observations do not establish separate semantic ownership for each region. In particular, an unknown dependency receiving the address at `+0x1c` may access storage overlapping other listed offsets.

### Global inputs

Three global locations are assigned descriptive labels for this contract:

- **G-01:** A 32-bit value read at function entry, before mode dispatch. Its captured value is used by the fallback store.
- **G-02:** A 32-bit value read on the qualifying `0x3e` path and passed to F-44362605ab79.
- **G-03:** A 32-bit value read on the repeated-`0x5b` path and passed to F-efdae002d289.

Their contents, types beyond the observed word-sized use, mutability, and ownership are unknown.

### Preconditions

For a normally completing invocation, every storage access reached by the selected path must be valid, and every called dependency must accept the arguments supplied to it.

There is no visible null check, object-size check, pointer validation, or argument-range validation. The object need not expose every listed field on every path: requirements are path-dependent, except for G-01, which is always read.

No alignment, byte order, concurrency protocol, or memory-fault behavior is established.

## Behavior

### Entry and non-dispatch mode

The function first reads G-01 and retains that 32-bit value.

If mode is not exactly `1`:

1. It writes the captured G-01 value to the object at `+0x38`.
2. It returns `13`.

This path does not inspect the event, read the state word, or call a dependency in the visible body.

The capture matters: the stored value is the entry-time read, not a fresh read performed at the store.

### Mode `1`, event `0x2f`

The function reads the 32-bit value at object offset `+0x20` and passes it to F-d8cbf9aa79a9.

If that dependency returns zero:

1. The captured entry-time G-01 value is written to `+0x38`.
2. The function returns `13`.

If the dependency returns any nonzero value:

1. The function reads the state word at `+0x28` after the dependency returns.
2. It sets bit `0x00000002`, retaining every other bit from that read.
3. It writes the resulting 32-bit word back to `+0x28`.
4. It returns `3`.

This path does not apply the state-word normalization used by the other special events. A dependency mutation of the state word can therefore affect the final stored value.

### Mode `1`, event `0x3e`

The function reads the 32-bit value at `+0x0c`, adds `4`, and passes the result to F-38a243955252.

It inspects bit `0x00000010` of that dependency’s 32-bit result.

If the bit is clear, the function returns `4`. There is no direct object write on this branch.

If the bit is set, processing continues in this order:

1. Call F-44362605ab79 with the address of object offset `+0x1c`.
2. After that call completes, read G-02.
3. Call F-44362605ab79 with the value read from G-02.
4. Call U-01 with the result from the second call as its first argument and the result from the first call as its second argument.
5. Inspect whether U-01 returned zero.

U-01 is an opaque identifier assigned here to the comparison-like dependency for which the supplied dependency map contains no function ID. Its actual semantics are unknown; only its arguments and zero/nonzero use are established.

If U-01 returns nonzero, the function returns `4` without a direct object write.

If U-01 returns zero:

1. Read the state word at `+0x28`, after all preceding dependency calls.
2. Set bit `0x00000001`.
3. Apply the normalization described below.
4. Write the resulting state word to `+0x28`.
5. Return `4`.

The return value alone does not distinguish the early exits from the branch that updates the state word.

### Mode `1`, event `0x5b`

The function reads the state word at `+0x28` and tests bit `0x00040000`.

If the bit is clear:

1. The function obtains the state word for the update.
2. It sets bit `0x00040000`, preserving all other bits.
3. It writes the result to `+0x28`.
4. It returns `3`.

The supplied representation contains a state-word read for the test and another read in forming this update. These are distinct represented reads, although their machine-level observability is not established.

No dependency is called on this branch, and no normalization is applied.

If bit `0x00040000` is already set:

1. Read G-03.
2. Call F-efdae002d289 with that value as its first argument and the address of object offset `+0x1c` as its second argument.
3. Inspect whether its result is zero.

If the dependency returns zero, return `3` without a direct state-word write.

If it returns nonzero:

1. Read the state word again, after the dependency returns.
2. Normalize that freshly read word.
3. Write the normalized value to `+0x28`.
4. Return `4`.

The final update uses the post-call state, not the state used to choose this branch. For example, if the dependency clears `0x00040000`, this function does not independently restore it.

### State-word normalization

For a 32-bit input word, normalization:

- Preserves bits `0` through `3`.
- Preserves bits `16` through `31`.
- Sets bit `9`, represented by `0x00000200`.
- Clears bits `4` through `8` and `10` through `15`.

Equivalently, the retained-bit mask is `0xffff000f`, followed by setting `0x00000200`.

On the qualifying `0x3e` path, bit `0` is set before normalization, so it remains set in the stored result. On the repeated-`0x5b` path, no extra low bit is set.

### Mode `1`, every other event

The function calls F-3643422b0298 with the object base and returns that dependency’s 32-bit result.

There is no direct object write in this branch. The dependency’s own effects remain unrestricted by the supplied evidence.

## Side effects

### Direct effects

A normal invocation performs at most one direct object write in the visible body:

- A 32-bit write to `+0x38` on either fallback route.
- A 32-bit write to `+0x28` on one of the state-update routes.
- No direct object write on the remaining routes.

G-01 is read even when its captured value is never stored. G-02 and G-03 are read only on their respective conditional paths.

“No direct write” does not mean “no mutation.” Dependencies may modify the object, globals, or other storage.

### Ordering and interference

The dependency calls described above are sequential. In particular:

- G-02 is read after the first F-44362605ab79 call.
- A successful `0x2f` update reads state after F-d8cbf9aa79a9 returns.
- A qualifying `0x3e` update reads state after U-01 returns.
- A repeated-`0x5b` update reads state after F-efdae002d289 returns.

No assumption that callees preserve globals or object fields is needed for these rules. Conversely, the evidence does not establish synchronization or safe behavior under concurrent mutation.

### Ownership and lifetime

The visible body does not directly allocate, free, retain, or transfer ownership of storage.

Passing an address or word to a dependency does not establish borrowing, copying, ownership transfer, or release behavior. Those matters remain unknown for every dependency. Normal completion requires any later accesses in this function to remain valid despite earlier dependency calls.

### Errors and abnormal completion

There is no explicit local error-reporting object, recovery mechanism, or cleanup path. Zero and nonzero dependency results are branch conditions, not established error classifications.

The stated return behavior is conditional on reached accesses and calls completing normally. Faults, nonlocal exits, exceptions, and dependency termination behavior are not specified.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-d8cbf9aa79a9 | Receives the word from `+0x20`; zero selects fallback storage and return `13`, nonzero selects a state-bit update and return `3`. |
| F-38a243955252 | Receives the value from `+0x0c` plus `4`; result bit `0x10` gates further `0x3e` processing. |
| F-44362605ab79 | Called first with the address at `+0x1c`, then with the value read from G-02. Both results are supplied to U-01. |
| U-01 | Receives those results in reverse call order; zero permits the `0x3e` state update. Identity and internal semantics are documented_only. |
| F-efdae002d289 | Receives G-03 and the address at `+0x1c`; its zero/nonzero result selects repeated-`0x5b` behavior. |
| F-3643422b0298 | Receives the object base for otherwise-unhandled events in mode `1`; its return is forwarded. |

The contract does not infer string operations, parsing, resource management, or comparison semantics from these interactions.

## Acceptance examples

These are conditional static examples, not results from runtime tests. Unless stated otherwise, assume valid storage, normal dependency completion, and no dependency mutations beyond those explicitly described.

1. **Non-dispatch mode.** With mode `0` and entry-time G-01 equal to `0x12345678`, the function writes `0x12345678` to `+0x38` and returns `13`, regardless of event.

2. **Zero result for `0x2f`.** In mode `1`, if F-d8cbf9aa79a9 returns zero, the captured G-01 value is stored at `+0x38` and the result is `13`. The visible body does not write `+0x28`.

3. **Nonzero result for `0x2f`.** If the post-call state word is `0x12345678`, the stored word is `0x1234567a`, and the return is `3`. Bits otherwise subject to normalization remain unchanged.

4. **Failed `0x3e` gate.** If F-38a243955252 returns a value with bit `0x10` clear, the function returns `4` without calling F-44362605ab79 or U-01.

5. **Qualifying `0x3e` update.** If the gate bit is set, U-01 returns zero, and the state read after all calls is `0xabcdfff0`, the stored word is `0xabcd0201`. The result is `4`.

6. **Nonzero U-01 result.** With the gate bit set but U-01 returning nonzero, the result is still `4`, but there is no direct state-word write.

7. **First `0x5b` transition.** If the tested and update-read state is `0x00001230`, the stored word is `0x00041230`, and the result is `3`. No normalization occurs.

8. **Repeated `0x5b`, zero dependency result.** If `0x00040000` is initially set and F-efdae002d289 returns zero, the function returns `3` without a direct write. Any dependency mutation remains outside that statement.

9. **Repeated `0x5b`, post-call state change.** If the initial test finds `0x00040000` set, but the dependency returns nonzero after changing state to `0x1234fff5`, the stored result is `0x12340205`, and the return is `4`. The original state is not reused.

10. **Delegated event.** With mode `1` and event `0`, F-3643422b0298 receives the object base. If it returns `0x87654321`, that exact 32-bit value is returned.

## Uncertainties

The following are documented_only:

- Semantic names for events, fields, state bits, and return values.
- The identity and implementation of U-01.
- Global contents and initialization requirements.
- The representation and extent of storage beginning at `+0x1c`.
- Dependency ownership, lifetime, mutation, and failure contracts.
- Object alignment, byte order, and concurrency guarantees.
- Arithmetic overflow behavior for address formation and the value-at-`+0x0c` addition.

The last item is significant: adding `4` is established, but the supplied representation does not justify an unconditional mathematical-integer or wraparound guarantee at the boundary. Examples avoid overflowing operands.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID map. Established facts are the represented branch conditions, field offsets, word widths, masks, call arguments and order, direct stores, and returned values.

Character interpretation and broader dispatcher intent are inferences only. No runtime tests have been performed. The acceptance examples express consequences of the static behavior under their stated conditions; they are not execution evidence.
