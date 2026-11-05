<!-- Trusted provenance: F-c164c8dfc4aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c164c8dfc4aa

## Purpose

F-c164c8dfc4aa conditionally performs a creation sequence associated with a caller-supplied state object. A 32-bit field at state byte offset 4 gates the operation: an initially nonzero value causes an immediate return of 0, without dependency calls or direct state writes.

When that field is initially zero, the function records two caller-supplied values, prepares local storage, obtains two dependency-defined representations, calls a creation dependency, and stores its result at offset 4. A nonzero creation result selects an additional dependency call and a return value of 1. A zero creation result produces a return value of 0. Both ordinary post-creation paths invoke the same two final dependencies.

These control-flow and dataflow properties are established by the supplied static representation. Resource creation, descriptor construction, cleanup, and synchronization are interpretations of the visible call pattern, not established contracts of the dependencies.

## Interface

### Arguments

The function accepts five arguments:

| Argument | Established use |
|---|---|
| State address | Base address for four-byte state accesses. Passed directly to one dependency; interior addresses at byte offsets 12 and 20 are passed to others. |
| First recorded value | A 32-bit value written at state byte offset 0 on the creation path. |
| Representation input | A 32-bit value passed unchanged to F-44362605ab79. Its interpretation is unknown. |
| Second recorded value | A 32-bit value written at state byte offset 8 on the creation path. |
| Creation input | A 32-bit value passed as the fourth argument to F-195f5d43c91b. Its interpretation is unknown. |

The directly evidenced state layout is:

| Byte offset | Access or address use |
|---|---|
| 0–3 | Receives the first recorded value. |
| 4–7 | Read for the initial zero check; receives the creation result if creation is attempted. |
| 8–11 | Receives the second recorded value. |
| 12 | Interior address passed to F-338d16f9ce72 and subsequently F-1333331b2da1. |
| 20 | Interior address passed to F-162a41bac6a7 only after a nonzero creation result. |

The interior addresses do not establish the sizes or boundaries of embedded objects. The full state extent, the purpose of bytes 16–19, and dependency access ranges are unknown.

### Local storage

The supplied representation declares two separate 32-bit local objects:

- A local initialized to zero.
- A distinct local initialized from an external seed value.

Only the address of the seed-initialized local is explicitly passed to dependencies. The representation does not establish that these locals form one object, are adjacent, or occupy an eight-byte addressable region. It also does not establish that any dependency can access the zero-initialized local through the other local’s address.

“Seed-initialized local” identifies the object by its initial assignment; it does not imply that its contents remain unchanged after dependency calls.

### External data

Four external data locations participate in the creation path:

| Location | Established use |
|---|---|
| Seed source | Its value initializes the seed-initialized 32-bit local. |
| Metadata location | Its address is passed to F-1f62160dfd52. |
| Creation setting A | Its value is supplied as the second argument to F-195f5d43c91b. |
| Creation setting B | Its value is supplied as the fifth argument to F-195f5d43c91b. |

Passing the metadata address does not establish a direct read of its contents by this function. The external values’ meanings and mutability are unknown.

### Return value

The function returns a 32-bit value:

- **1:** creation returned nonzero, and all subsequent calls on that path returned normally.
- **0:** the initial gate was nonzero, or creation returned zero and the subsequent calls returned normally.

The return value is a zero/nonzero outcome indicator, not the creation result itself. Return 0 does not distinguish an initially populated state from an unsuccessful creation attempt.

### Preconditions and errors

The state address must permit the initial four-byte read at offset 4. If the gate is zero, the state must additionally support the direct writes and whatever accesses the dependencies require.

No null, alignment, bounds, or argument validation is visible. No separate error reporting or rollback mechanism is visible. Additional preconditions belong to the unknown dependency contracts.

This specification describes ordinary execution. Invalid memory, nonlocal exits, and dependencies that fail to return normally have no established recovery behavior.

## Behavior

### Initial gate

The first visible state access reads the 32-bit field at offset 4 and compares it with zero.

For any nonzero bit pattern, the function returns 0 without:

- Calling any listed dependency.
- Directly writing the state.
- Performing either local initialization.
- Reading the seed source or creation settings in the visible body.

A zero value admits the creation path. The function does not visibly recheck this field before calling the creation dependency.

### Entry operation and recorded values

On the creation path, F-338d16f9ce72 receives the state interior address at byte offset 12.

After that call returns, the function writes the first recorded value at offset 0, then the second recorded value at offset 8. Each write occupies four bytes. Neither write depends on a returned status from the entry dependency.

The entry call occurs after the gate read. Even if that dependency implements synchronization, the visible function does not provide a check performed after entry. A dependency-induced change to the gate field therefore does not suppress the remaining creation sequence.

### Local preparation and representation extraction

After the recorded-value writes, the zero-initialized local receives zero. The distinct seed-initialized local then receives the value read from the external seed source.

F-1f62160dfd52 subsequently receives, in positional order:

1. The address of the seed-initialized local.
2. The metadata location’s address.
3. The original state address.

The zero-initialized local is not an explicit argument. Its initialization is established, but no subsequent direct read of it is visible. Its relationship to any dependency-accessed storage remains unknown.

After F-1f62160dfd52 returns, F-44362605ab79 is called twice:

1. With the address of the seed-initialized local.
2. With the caller’s representation input.

The first returned representation is retained across the second call. The second returned representation and the retained first representation are used in different argument positions during creation.

These calls do not establish the representation input’s type or any equality between the two inputs. The address passed in the first call identifies the same local object previously supplied to F-1f62160dfd52, but that object’s contents may already have changed.

No dependency is assumed to preserve external data. In particular, preceding calls may affect the creation settings before those settings are obtained for the creation call.

### Creation and result publication

F-195f5d43c91b receives seven arguments:

| Position | Supplied value |
|---|---|
| 1 | Representation returned for the caller’s representation input. |
| 2 | Creation setting A. |
| 3 | Representation returned for the seed-initialized local’s address. |
| 4 | Caller’s creation input. |
| 5 | Creation setting B. |
| 6 | Numeric value 1. |
| 7 | Numeric value 60. |

No meaning or unit is established for the final two values. They are not proven to be flags, counts, priorities, sizes, or durations.

The setting values are supplied after both representation calls have completed. The supplied representation does not establish the relative evaluation order of the separate setting reads within the creation-call expression.

The creation result is represented as a 32-bit integer. It is written to the four-byte state field at offset 4. The subsequent branch tests the retained result, not a new read of that field.

Only equality with zero matters. A nonzero result with its high bit set follows the nonzero path just as any other nonzero pattern does. The function performs no visible range check, signed comparison, or normalization before storing the result.

### Nonzero creation result

After storing a nonzero result, the function calls:

1. F-162a41bac6a7 with the state address plus 20 bytes.
2. F-289ec4311c64 with the seed-initialized local’s address.
3. F-1333331b2da1 with the previously formed state address plus 12 bytes.

If all three calls return normally, the function returns 1.

The result write precedes these calls. The function itself performs no later state-field write, but the dependencies are not known to preserve the stored result or any other state. Return 1 therefore does not guarantee that offset 4 remains nonzero at return.

### Zero creation result

After storing a zero result, the function skips F-162a41bac6a7 and calls:

1. F-289ec4311c64 with the seed-initialized local’s address.
2. F-1333331b2da1 with the previously formed state address plus 12 bytes.

If both calls return normally, the function returns 0.

The earlier writes at offsets 0 and 8 are not directly undone. The final calls may nevertheless modify state through their own behavior.

## Side effects

On the creation path, the function’s explicit state writes occur at offsets 0, 8, and 4, in that order, with the intervening calls described above. The gate-failure path performs none of these writes.

The function also initializes two separate local objects, first with zero and then with the external seed value. Only the seed-initialized object’s address is explicitly exposed. No combined layout or additional addressable extent is established.

Dependencies may have further effects through:

- The original state address.
- State interior addresses at offsets 12 and 20.
- The seed-initialized local’s address.
- The metadata address.
- The representation input, if its unknown interpretation permits access to other state.
- Other storage or external resources accessible to the dependencies.

No visible call has independently established allocation, ownership-transfer, or release semantics. F-289ec4311c64 is invoked on both ordinary post-creation paths, which is consistent with cleanup, but does not prove resource release.

The address-exposed local has invocation-local storage. Whether any dependency retains that address, and whether retention is permitted, is undocumented. The function does not visibly arrange a longer lifetime.

The calls using offset 12 resemble entry and exit operations. Locking, mutual exclusion, reentrancy safety, and thread safety cannot be concluded from that resemblance. Likewise, no exception-safe or nonlocal-exit-safe cleanup guarantee is established.

## Dependencies

| Opaque function ID | Visible contract at this call site |
|---|---|
| F-338d16f9ce72 | Called once after a zero gate, with state address plus 12 bytes. No returned value controls the caller’s behavior. |
| F-1f62160dfd52 | Receives the seed-initialized local’s address, metadata address, and original state address after both local initializations. No returned value is used. |
| F-44362605ab79 | Called first with the seed-initialized local’s address, then with the representation input. Both results feed creation. |
| F-195f5d43c91b | Receives the seven arguments specified above. Its 32-bit result is stored and classified as zero or nonzero. |
| F-162a41bac6a7 | Receives state address plus 20 bytes only after a nonzero creation result has been stored. No returned value is used. |
| F-289ec4311c64 | Receives the seed-initialized local’s address on either ordinary post-creation path. No returned value is used. |
| F-1333331b2da1 | Receives state address plus 12 bytes after F-289ec4311c64 returns. No returned value is used. |

These are call-site obligations and observations, not independent dependency specifications. Ignoring a return value does not imply that a dependency cannot fail, mutate memory, retain addresses, or affect external resources.

## Acceptance examples

The following are conditional acceptance cases derived from static evidence. They are not executed tests.

### Initially nonzero gate

Given valid state storage with 9 at offset 4:

- The return value is 0.
- No listed dependency is called.
- No direct state write or local initialization on the creation path occurs.

Every other nonzero 32-bit gate value selects the same path.

### Zero creation result

Given an initially zero gate and normally returning dependencies, if F-195f5d43c91b returns zero:

- The entry call precedes the recorded-value writes.
- Both local initializations precede F-1f62160dfd52.
- Both representation calls precede creation.
- Zero is written at state offset 4.
- F-162a41bac6a7 is skipped.
- F-289ec4311c64 precedes F-1333331b2da1.
- The return value is 0.

If dependencies do not subsequently alter offsets 0 and 8, those fields retain the supplied recorded values. Their final contents are otherwise not guaranteed.

### High-bit-set creation result

Given an initially zero gate and a creation result with hexadecimal bit pattern 80000000:

- The complete 32-bit pattern is written at offset 4.
- The nonzero path is selected regardless of signed interpretation.
- F-162a41bac6a7 precedes the two final calls.
- Normal completion returns 1.

### Gate changed during entry

If the initial gate is zero but F-338d16f9ce72 changes it to a nonzero value and returns normally, creation still proceeds. The later creation-result write replaces that field; no intervening gate recheck is visible.

### Stored result changed afterward

If creation returns nonzero and a subsequent dependency changes offset 4 to zero, normal completion still returns 1. The return decision follows the retained creation result, not the field’s final contents.

### Local-address observations

A dependency observing the address supplied to F-1f62160dfd52, the first F-44362605ab79 call, and F-289ec4311c64 receives the same seed-initialized local’s address.

Acceptance must not require a zero word at address plus four bytes, an eight-byte descriptor, or access to the separately zero-initialized local. None of those layout properties is established.

## Uncertainties

The following remain **documented_only**:

- Concrete state type, full storage extent, alignment, and lifecycle.
- Semantic meanings of the recorded values, representation input, creation input, and external settings.
- Exact ABI, pointer representation, and endianness.
- Dependency argument and result types beyond their visible use.
- Adjacency, ordering in memory, or combined extent of the two local objects.
- Whether dependencies access the zero-initialized local.
- Whether dependency accesses extend beyond the seed-initialized local’s four bytes, and what would authorize such accesses.
- Whether the address-exposed local participates in a descriptor, callback binding, or another structure.
- Dependency mutation, aliasing, address-retention, ownership, and release rules.
- Whether the creation result represents a resource or ownership obligation.
- Whether the offset-12 calls implement synchronization.
- The meaning of the offset-20 operation.
- Recovery behavior when dependencies do not return normally.

Four-byte indexing establishes the listed state offsets. No variable-length arithmetic, size multiplication, checked overflow behavior, or pointer-wrap policy is established. Valid storage and valid interior addresses are preconditions, not outcomes guaranteed by the function.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. No implementation access or runtime tests have been performed.

Established evidence comprises the initial zero gate; four-byte state accesses; direct write order; two separate 32-bit local initializations, zero first and seed second; explicit exposure of only the seed-initialized local’s address; dependency call order; creation argument positions; stored-result classification; and the two post-creation paths.

A contiguous two-word local object, an eight-byte descriptor, and dependency access to the zero-initialized local are not established. Resource creation, descriptor processing, cleanup, and synchronization remain inferred intent rather than substituted dependency guarantees.
