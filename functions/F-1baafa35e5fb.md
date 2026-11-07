<!-- Trusted provenance: F-1baafa35e5fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1baafa35e5fb

## Purpose

F-1baafa35e5fb conditionally updates a state structure using an identity value, a two-bit mode, and a 16-bit counter. Depending on the state, it either:

- Rejects a particular derived address.
- Returns a nonzero status from a dependency.
- Rejects a matching-identity state in mode 1.
- Increments a counter for a matching-identity state in mode 2.
- Invokes a dependency and, on success, records the sampled identity and sets the counter to 1.

**Inferred intent:** The identity comparison, special treatment of modes 1 and 2, and counter update resemble an ownership-sensitive acquisition operation with a recursive case. This interpretation does not establish locking, thread ownership, synchronization, or resource-acquisition guarantees. Those properties depend on unavailable contracts.

This specification distinguishes the function’s explicit accesses from any accesses performed by dependencies. Dependency calls may change shared state; preservation of memory or globals is not assumed.

## Interface

### Input

The function receives one 32-bit value used as a base address, denoted here as **B**. The visible interface does not supply a length, allocation descriptor, or independently validated object type.

The function uses the following byte offsets relative to B:

| Offset | Access width | Role established by visible use |
|---|---:|---|
| `0x148` | 32 bits | Value compared with a global comparison value; also the address passed to F-621dae46a146 |
| `0x14C` | 32 bits | Identity field, read for comparison and conditionally overwritten |
| `0x156` | 16 bits | Unsigned flags from which the mode is extracted |
| `0x15A` | 16 bits | Unsigned counter on read; low 16 bits of the new count on write |
| `0x15C` | Address passed onward | Argument to F-6f7f27926f9e; extent and contents are unknown |

The mode is the unsigned value obtained by masking the flags with `0x003F` and shifting right by four bits. Only original flag bits 4 and 5 affect the result, so the mode ranges from 0 through 3. Other flag bits do not affect this function’s branch selection.

### Output

The return value is a 32-bit integer status:

| Value | Explicit local meaning |
|---|---|
| `0` | One of the two explicit successful counter updates completed |
| `0x1A` / 26 | The address derived as B plus `0x148` compares equal to zero |
| `0x0F` / 15 | The identity matches and the sampled mode is 1 |
| `0x27` / 39 | The identity matches, the sampled mode is 2, and the counter is `0xFFFF` |
| Any nonzero dependency result | Returned unchanged from F-621dae46a146 or F-6f7f27926f9e |

Numeric values alone do not uniquely identify an error’s origin. A dependency may return the same value as a locally generated status, including a negative 32-bit result.

### Preconditions

For every path that accesses memory, the corresponding fields and global comparison value must be readable. Fields written on the selected path must be writable. Alignment, address validity, object lifetime, and dependency-specific requirements are caller obligations not validated by the visible body.

A conventionally valid structure extending through the explicit counter field requires storage through byte offset `0x15B`. The dependency receiving B plus `0x15C` may require additional storage; its extent cannot be established here.

The initial address check is **not** a check that B itself is nonzero. A zero B does not satisfy that guard and proceeds toward accesses at nonzero offsets from zero.

## Behavior

### 1. Check the derived address

The function first forms the address B plus `0x148`.

If that derived address compares equal to zero, it returns 26. No explicit structure read, dependency call, or structure write precedes this return.

Under 32-bit wrapping address arithmetic, B equal to `0xFFFFFEB8` would produce that zero address. This is a conditional machine-arithmetic interpretation, not a guarantee that an invalid base can safely be supplied under every execution environment.

### 2. Optionally invoke the preliminary dependency

The function compares the 32-bit value at B plus `0x148` with a global 32-bit comparison value.

If they differ, F-621dae46a146 is not called.

If they match, F-621dae46a146 is called with:

1. The address B plus `0x148`.
2. The integer value zero.

A nonzero result is returned immediately and unchanged. No subsequent mode read, identity query, explicit identity write, or explicit counter write occurs.

A zero result permits execution to continue. The function does not repeat the comparison or verify the state after this call. Any modifications made by the dependency can therefore affect subsequent field reads.

The supplied material does not establish the relative evaluation order of the two operands of the initial comparison. Both comparison values must be available before the branch is selected.

### 3. Sample the mode and obtain an identity

After the optional preliminary call succeeds or is skipped, the function reads the 16-bit flags at B plus `0x156` and computes the mode.

It then calls F-196812b4b122 with no explicit arguments and retains its 32-bit result as the sampled identity.

The mode is sampled **before** this call. The identity field is read **after** this call. Consequently:

- Changes to the flags during the identity call do not change the mode used by this invocation.
- Changes to the identity field during that call can change whether the later comparison matches.
- The identity result is not interpreted as a status code. Every returned bit pattern participates in comparison and may later be stored.

### 4. Handle matching-identity special cases

The function reads the 32-bit identity field at B plus `0x14C` and compares it with the sampled identity.

If they match and the sampled mode is 2:

1. Read the counter at B plus `0x15A` as an unsigned 16-bit value.
2. If the counter is `0xFFFF`, return 39 without an explicit counter write.
3. Otherwise, add one and write the resulting 16-bit value to B plus `0x15A`.
4. Return zero.

This path does not call F-6f7f27926f9e and does not explicitly rewrite the identity field.

If the identities match and the sampled mode is 1, return 15. This path does not explicitly read the counter, call F-6f7f27926f9e, or update either field.

Matching identities with mode 0 or 3 proceed to the general path.

### 5. Perform the general operation and record state

The general path is used whenever:

- The identities differ, regardless of mode; or
- The identities match and the sampled mode is 0 or 3.

The function calls F-6f7f27926f9e with the address B plus `0x15C`.

If this call returns nonzero, that result is returned unchanged. The function performs no subsequent explicit identity or counter write.

If the call returns zero, the function performs these explicit writes in order:

1. Write the previously sampled 32-bit identity to B plus `0x14C`.
2. Write the 16-bit value 1 to B plus `0x15A`.

It then returns zero.

The identity is not queried again after F-6f7f27926f9e. The function also does not re-read the mode, compare identities again, or validate the counter before these writes.

### Counter arithmetic

The matching-identity, mode-2 rejection condition excludes exactly `0xFFFF`; it does not reject the entire `0xFF00`–`0xFFFF` range.

The visible range test first restricts attention to values above `0xFEFF`, then rejects differences from `0xFF00` that exceed `0xFE`. Within an unsigned 16-bit input range, only `0xFFFF` meets both conditions.

Consequences include:

- `0x0000` increments to `0x0001`.
- `0x7FFF` increments to the stored bit pattern `0x8000`.
- `0xFEFF` increments to `0xFF00`.
- `0xFFFE` increments to `0xFFFF`.
- `0xFFFF` returns 39 rather than wrapping to zero.

The intermediate is represented as a signed 16-bit quantity in the supplied representation. The observable store is nevertheless a 16-bit bit pattern. This function does not subsequently read that intermediate as a signed numeric result.

The general path replaces the counter with 1 without reading its previous value, including when that previous value is `0xFFFF`.

## Side effects

The function’s explicit writes are limited to:

- A 16-bit counter write on successful matching-identity mode-2 execution.
- A 32-bit identity write followed by a 16-bit counter write after a successful general-path dependency call.

It performs no explicit allocation, deallocation, ownership transfer, flag update, or global write.

That limitation applies only to the visible body. Dependencies receive addresses into the caller-supplied state or may access external state. Their memory writes, resource handling, and ownership effects are unknown.

An error return does not establish that the entire operation had no side effects:

- F-621dae46a146 may have changed state before returning an error.
- F-196812b4b122 has already run before the locally generated statuses 15 and 39.
- F-6f7f27926f9e may have changed state before returning an error.

No rollback is visible.

No atomic operation, memory barrier, or synchronization guarantee is established by this body. In particular, the ordered identity and counter writes on the general success path are not established as one indivisible state transition. Concurrent mutation and observation require guarantees outside this specification.

## Dependencies

| Opaque ID | Invocation condition | Arguments | Result handling |
|---|---|---|---|
| F-621dae46a146 | Initial 32-bit field equals the global comparison value | Address B plus `0x148`; integer zero | Nonzero returned unchanged; zero continues |
| F-196812b4b122 | After the preliminary stage permits continuation | No explicit arguments | Retained as a 32-bit identity |
| F-6f7f27926f9e | Identity mismatch, or matching identity with mode 0 or 3 | Address B plus `0x15C` | Nonzero returned unchanged; zero permits state writes |

The global comparison value is an additional data dependency. Its value, mutability, initialization, and semantic role are not supplied.

No dependency is assumed to preserve globals, preserve the input structure, allocate memory, release memory, or transfer ownership. Successful continuation requires that any dependency effects leave the addresses needed by later accesses valid.

## Acceptance examples

These are conditional contract examples, not executed tests. Unless stated otherwise, assume valid memory, a skipped or successful preliminary stage, and no dependency mutations beyond the stated results.

1. **Preliminary failure stops processing.**  
   The initial comparison matches and F-621dae46a146 returns −7. The function returns −7. Neither later dependency is called, and this function performs no explicit identity or counter write.

2. **Matching identity, mode 1.**  
   The sampled identity equals the identity field and the sampled mode is 1. The result is 15. The counter is not explicitly read or written, and F-6f7f27926f9e is not called.

3. **Ordinary matching-identity increment.**  
   The identities match, mode is 2, and the counter is 9. The function stores 10 and returns zero without calling F-6f7f27926f9e.

4. **Crossing the signed 16-bit boundary.**  
   Under the same matching-identity mode-2 conditions, counter `0x7FFF` becomes the 16-bit pattern `0x8000`; the function returns zero.

5. **Reaching the maximum count.**  
   Counter `0xFFFE` becomes `0xFFFF` and returns zero. A subsequent invocation with the same matching identity, sampled mode 2, and unchanged counter returns 39 without writing the counter.

6. **Identity mismatch bypasses the mode-2 increment.**  
   The sampled mode is 2, but the identities differ. Even if the counter is `0xFFFF`, F-6f7f27926f9e is called. If it succeeds, the sampled identity is stored and the counter becomes 1.

7. **Matching identity in mode 3 uses the general path.**  
   F-6f7f27926f9e is called. On zero, the identity field is written and then the counter is set to 1; the previous count is not incremented.

8. **General-path failure prevents explicit final writes.**  
   The identities differ and F-6f7f27926f9e returns 8. The function returns 8 without its own final identity or counter write. Dependency side effects remain possible.

9. **Mode sampling precedes the identity call.**  
   The flags initially select mode 1. During F-196812b4b122, the flags change to mode 2. If the subsequently read identity field matches the returned identity, the function returns 15 using the earlier mode sample.

10. **The recorded identity is not refreshed.**  
    F-196812b4b122 supplies an identity, and the general path is selected. If F-6f7f27926f9e succeeds, the function stores that earlier identity even if external identity-related state changed during the dependency call.

## Uncertainties

The following are **documented_only**:

- The actual object type, allocation extent, alignment requirements, and meaning of unspecified bytes.
- Whether the identity denotes a thread, task, process, owner token, or another entity.
- Whether either address-taking dependency implements initialization, waiting, locking, or another operation.
- The meaning and lifecycle of the global comparison value.
- Dependency side effects, preservation guarantees, memory ownership rules, and validity requirements.
- Concurrent-access guarantees and behavior under races.
- The execution environment’s treatment of invalid addresses, alignment faults, address wrapping, and signed narrowing.
- Symbolic names or broader protocol meanings for statuses 15, 26, and 39.
- Behavior outside the supplied reachable representation.

These unknowns do not justify adding validation, retries, cleanup, rollback, or resource-release behavior.

## Evidence

Established static evidence consists of the supplied function representation: its field access widths and offsets, call arguments and ordering, branch conditions, explicit stores, arithmetic, and return paths.

Ownership-sensitive acquisition is an inferred purpose, not an established dependency contract. The exact `0xFFFF` rejection and mode range follow from the visible integer operations.

The supplied annotations indicate that unreachable regions were omitted. They do not establish additional behavior or recover the omitted regions’ semantics.

No runtime tests have been performed. The acceptance examples are derived conditions for review and future verification, not observed execution results.
