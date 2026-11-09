<!-- Trusted provenance: F-eb6105add51f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb6105add51f

## Purpose

F-eb6105add51f conditionally advances a stateful traversal, selecting records whose discriminator matches the traversal object and updating two 32-bit position fields.

The caller may specify an ordinal target, a value target, or both. The all-ones 32-bit value disables either target. The function returns a 32-bit result of zero or one.

The return value is not a simple “record found” indicator:

- A cached equality can return one without traversal.
- Traversal exhaustion can still return one, depending on the stored positions.
- A candidate rejected for exceeding the value target can leave the object unchanged by this function yet still lead to a return value of one.
- Specifying both targets does not mean that both must match for success.

The established behavior is the sequence of checks, calls, comparisons, and state accesses below. Interpreting the object as an iterator, its positions as an ordinal and an exclusive value boundary, and the conditional dependency call as a reset is inferred intent rather than an established dependency contract.

## Interface

### Inputs

The function receives four machine-level arguments:

| Argument | Interpretation |
|---|---|
| State address | Address of a writable object containing the fields described below. |
| Ordinal target | Unsigned 32-bit value. `0xffffffff` disables this target. |
| Value target | Unsigned 32-bit value. `0xffffffff` disables this target. |
| Extraction seed | A 32-bit value copied into a local slot before the first dependency call. The address of that slot is subsequently passed to F-3bcbe089f356. |

The extraction seed is passed by value to this function. Updates to its local copy are not returned directly to the caller.

The state address is represented by a 32-bit integer in the supplied material. A safe calling environment must provide valid addresses for the accesses actually reached. There is no explicit null check, size check, or memory-access error handling.

### State layout

The following offsets are byte offsets from the state address. Each listed field is accessed as four bytes.

| Offset | Observed use |
|---|---|
| `+0x04` | Opaque argument supplied to F-86f132defe26 on each traversal step. |
| `+0x0c` | Required discriminator, reread for each nonzero candidate. |
| `+0x10` | Unsigned bound checked against an enabled ordinal target. |
| `+0x14` | Stored traversal cursor. |
| `+0x18` | Stored value position. |
| `+0x1c` | Stored ordinal position. |

For every nonzero candidate returned by F-86f132defe26, the function reads a four-byte discriminator at candidate offset `+0x08`.

No interpretation is established for other object or candidate fields.

### Output

The return value is exactly zero or one in a 32-bit result.

There is no distinct return code for initialization failure, invalid target, traversal exhaustion, or rejection of an over-limit candidate. Dependency failures that are not represented by the explicitly checked return values have no established handling.

### Ownership and lifetime

This function contains no explicit allocation or deallocation. It does not explicitly release the state object, candidates, or the opaque traversal argument.

That observation does not establish ownership behavior inside dependencies. In particular:

- A dependency may mutate reachable state or globals.
- Candidate lifetime and ownership are unknown.
- Whether F-3bcbe089f356 retains the address of the local extraction slot is unknown. Retaining and later using that address would require a lifetime guarantee not provided here.

## Behavior

### 1. Initial gate and target validation

The extraction seed is first copied into the local extraction slot. F-2c367f61ccd2 is then called without explicit arguments.

If that dependency returns zero, the function returns zero immediately. No state-object field is explicitly read by this function before that return.

Otherwise:

1. If the ordinal target is disabled and the value target is also disabled, return zero.
2. If the ordinal target is enabled, read the unsigned bound at `+0x10`. Return zero when the bound is less than or equal to the ordinal target.

Thus an enabled ordinal target must be strictly below the stored bound. This validation precedes cached-position success checks: even a cached ordinal equality does not override an out-of-range ordinal target.

A value-only request does not read or validate against the bound at `+0x10`.

### 2. Cached-position success

After validation, the function checks:

1. If the ordinal target is enabled, whether the stored ordinal position equals it.
2. Only if that test does not succeed, and the value target is enabled, whether the stored value position equals it.

Either equality returns one immediately.

These are alternatives. If both targets are enabled, a cached match for either target is sufficient, regardless of the other target’s position.

No traversal or call to F-762f9ce15e4a occurs on this path.

### 3. Conditional state preparation

If neither cached equality succeeds, the function decides whether to call F-762f9ce15e4a with the state address.

The ordinal comparison in this decision is signed, unlike the unsigned ordinal-bound check. Each 32-bit ordinal bit pattern is interpreted as a signed two’s-complement quantity for this comparison.

The decision proceeds as follows:

- With an enabled ordinal target, reread the stored ordinal position.
- If that position is signed-greater-than the ordinal target, call F-762f9ce15e4a without first consulting the stored value position.
- If the ordinal target is disabled, or the stored ordinal is signed-less-than the target, inspect the value condition:
  - A disabled value target does not cause this call.
  - An enabled value target causes the call only when the stored value position is unsigned-greater-than the value target.

The supplied branch structure also places ordinal equality in the value-condition path. In an execution where the state is unchanged between the earlier equality check and this reread, that equality case has already returned.

F-762f9ce15e4a has no inspected return value. The function subsequently reads the stored cursor at `+0x14`, so any cursor change made by this dependency affects traversal. Its other effects are unknown; the function does not verify that it reset or normalized any field.

### 4. Candidate traversal

The stored cursor is loaded once to initialize a local traversal cursor.

For each traversal step:

1. Read the opaque argument at state offset `+0x04`.
2. Call F-86f132defe26 with that argument and the current local cursor.
3. Replace the local cursor with the returned value.
4. If the returned value is zero, terminate traversal.
5. Otherwise, compare the four-byte discriminator at candidate offset `+0x08` with the current discriminator at state offset `+0x0c`.

A discriminator mismatch causes another traversal step. This function performs no explicit state-field write for that skipped candidate. The skipped candidate nevertheless becomes the local cursor passed to the next traversal call.

For a matching discriminator, call F-3bcbe089f356 with the candidate and the address of the local extraction slot. Its return value, if any, is not inspected.

The extraction slot is initialized only once, before the initial gate. It is not reseeded before each matching candidate. If the dependency leaves it unchanged, the subsequent behavior uses its previous contents.

### 5. Value rejection and state updates

After extraction, compare the value target with the extraction slot as unsigned 32-bit values.

If the extracted value is greater than the value target, terminate traversal before performing this function’s candidate-related state writes. Any effects already produced by dependencies remain outside that guarantee.

Consequences include:

- An enabled value target rejects a candidate above that target.
- A disabled value target is `0xffffffff`, so no 32-bit extracted value exceeds it.
- An extracted value of `0xffffffff` is rejected whenever the value target is enabled.

For a candidate that is not rejected, the explicit state accesses and writes occur in this order:

1. Write the extracted value to the stored value position.
2. Write the candidate address to the stored cursor.
3. Read the stored ordinal position and form its 32-bit increment.
4. Write the incremented ordinal position.
5. If the extracted value is not `0xffffffff`, overwrite the stored value position with the extracted value plus one.

The first value-position write therefore precedes both the cursor and ordinal writes. For an ordinary extracted value, the value position is subsequently written a second time.

The final stored value position is:

- `extracted value + 1` when the extracted value is below `0xffffffff`;
- `0xffffffff` when the extracted value equals that sentinel.

The ordinary addition cannot overflow because the sentinel is excluded. The ordinal increment has no explicit overflow guard; under the represented 32-bit arithmetic, its bit pattern wraps from `0xffffffff` to zero. No resulting range or monotonicity invariant is checked.

### 6. Continue or stop

After an accepted candidate updates the state:

- If the ordinal target is enabled and the newly incremented ordinal does not equal it, continue traversal immediately.
- Otherwise:
  - Stop if the value target is disabled.
  - With an enabled value target, reread the stored value position and stop when it is unsigned-greater-than-or-equal-to the value target.
  - Continue if it remains below the value target.

This ordering matters when both targets are enabled. Reaching the value threshold does not stop this part of the loop while the new ordinal differs from its target. However, the earlier extracted-value rejection still terminates traversal whenever a matching candidate’s extracted value exceeds the value target.

There is no explicit iteration limit or cycle detection. Termination depends on dependency results, discriminator matches, extracted values, and the target conditions.

### 7. Final result

Following traversal termination, return one if either condition holds:

- The ordinal target is enabled and the current stored ordinal equals it.
- The value target is enabled and the current stored value position is unsigned-less-than-or-equal-to `value target + 1`.

Otherwise return zero.

The addition in the second condition cannot overflow: it is evaluated only for enabled value targets, whose maximum is `0xfffffffe`.

The value condition is intentionally stated as an upper-bound comparison. It is not a requirement that the stored value reach the target. A stored value well below the requested target can satisfy it.

These final checks read object fields after the dependencies and any explicit updates; they do not rely solely on cached pre-traversal positions.

## Side effects

The function explicitly writes only the stored cursor, stored value position, and stored ordinal position, and only after accepting a matching candidate.

Earlier returns perform no explicit object writes by this function. That does not imply an unchanged process or object: F-2c367f61ccd2 has already been called on all paths, and later paths may also invoke other dependencies.

Observable ordering includes:

- The initial gate precedes target validation.
- The ordinal bound check precedes cached-position success.
- Candidate classification precedes extraction.
- Extraction precedes value rejection.
- Value rejection precedes this function’s candidate-related state writes.
- The optional state-preparation dependency precedes loading the initial traversal cursor.

No atomicity, synchronization, rollback, or thread-safety guarantee is established. Concurrent mutation can invalidate stable-state interpretations of the comparisons.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-2c367f61ccd2 | Called once before target checks; zero causes immediate failure. | Meaning of nonzero results, global effects, initialization semantics. |
| F-762f9ce15e4a | Conditionally called with the state address before traversal; return ignored. | Whether it resets, rewinds, releases resources, or changes any particular field. |
| F-86f132defe26 | Receives the current opaque traversal argument and local cursor; zero terminates traversal; nonzero is treated as a candidate address. | Ordering, allocation, ownership, cycle freedom, and mutation behavior. |
| F-3bcbe089f356 | Receives a matching candidate and a pointer to the seeded 32-bit extraction slot. | Whether it always writes that slot, its value interpretation, retention behavior, and other side effects. |

## Acceptance examples

These examples are conditional on valid accessible memory and the stated dependency behavior. They are not runtime observations.

1. **Initial gate fails.**  
   F-2c367f61ccd2 returns zero. Expected result: zero, without explicit state-object reads or writes by this function.

2. **Both targets disabled.**  
   The initial gate succeeds and both targets are `0xffffffff`. Expected result: zero, without traversal.

3. **Ordinal bound takes precedence.**  
   The ordinal target is 7, the bound is 7, and the stored ordinal is also 7. Expected result: zero, not cached success.

4. **Either cached equality suffices.**  
   The ordinal target is valid and equals the stored ordinal. The value target is enabled but differs from the stored value. Expected result: one without traversal.

5. **Signed preparation decision differs from unsigned ordering.**  
   The bound is `0xffffffff`, the ordinal target is `0x80000000`, the stored ordinal is zero, and the value target is disabled. With stable state, the ordinal target passes validation, but signed zero exceeds signed `0x80000000`. Expected behavior: call F-762f9ce15e4a before loading the cursor.

6. **Accepted extraction produces an exclusive-looking position.**  
   A matching candidate extracts 10, the value target is 10, the ordinal target is disabled, and the stored ordinal was 3. Expected explicit final fields: candidate cursor, ordinal 4, value position 11. Traversal stops and the final value condition returns one.

7. **Over-limit candidate does not imply failure.**  
   The value target is 10, the stored value is 4, and the first matching candidate extracts 12. With dependencies leaving state unchanged, no candidate-related state writes occur. Expected result: one, because 4 is at most 11.

8. **Sentinel extraction remains sentinel.**  
   The value target is disabled, an enabled valid ordinal target is reached by the next increment, and the matching candidate extracts `0xffffffff`. Expected stored value: `0xffffffff`, not zero. Expected result: one through ordinal equality.

## Uncertainties

The following remain **documented_only**:

- The application meaning and full layout of the state object.
- Dependency implementations, ownership rules, and mutation effects.
- Whether extracted values are ordered or related to candidate order.
- Whether all reachable ordinal increments avoid wraparound.
- Whether external mutation can occur between field accesses.
- Whether dependencies always return valid candidate addresses and initialize extraction output.
- Whether traversal is guaranteed to terminate.

“Ordinal,” “value position,” “cursor,” and “preparation” are descriptive labels. Their domain meanings are inferred, not independently established.

## Evidence

The supplied static representation establishes the listed field offsets, four-byte accesses, dependency-call order, sentinel tests, comparison directions, explicit writes, and return predicates.

Its overflow-aware ordinal comparison supports a signed comparison for the preparation decision. Bound checks, extracted-value rejection, loop value thresholds, and the final value predicate use unsigned comparisons.

The contract does not incorporate dependency implementations or assume that callees preserve globals, object fields, or memory ownership. No runtime tests have been performed.
