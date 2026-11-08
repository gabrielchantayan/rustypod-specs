<!-- Trusted provenance: F-a41ee7bf45a2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a41ee7bf45a2

## Purpose

F-a41ee7bf45a2 validates a named operation, resolves a selector, conditionally checks an existing record, and populates a destination record obtained from a dependency.

A requested selector of 5 causes ordered operations for selectors 1, 2, and 3. A nonzero result from either of the first two operations stops subsequent work in the outer invocation. No rollback is visible in this function. Persistence of earlier effects, isolation, and transaction guarantees are not established: later operations and dependencies may change or undo earlier state.

“Name,” “registration,” “selector,” “payload,” and “callback” are descriptive labels. The established behavior is validation, dependency invocation, record inspection, and field writes; broader application semantics remain unknown.

## Interface

The function accepts eight arguments and returns a 32-bit integer status.

| Argument | Established representation and use |
|---|---|
| Context | A 32-bit value passed to dependencies and used as the base address for a 32-bit field at byte offset 144. |
| Name reference | A 32-bit value required to be nonzero. Passed to F-1cb585febd84 and F-d92bc662c47a. Its referenced format is unknown. |
| Index token | A pointer-shaped 32-bit value used numerically, not dereferenced through this argument. Validated with unsigned arithmetic, used in an existing-record comparison, and stored after truncation to 16 bits. |
| Selector | An unsigned 32-bit value. Values 4 and 5 receive special handling. Record-access calls receive its effective value masked to eight bits; existing-record comparisons use the full effective value. |
| Payload | An uninterpreted 32-bit value written into the destination. |
| Primary callback value | A 32-bit value checked for zero and written into the destination. |
| First paired callback value | A 32-bit value checked for zero and written into the destination. |
| Second paired callback value | A 32-bit value checked for zero and written into the destination. |

The callback values are not invoked directly. Their interpretation as executable addresses, handles, or other values is not established.

### Results

| Status | Explicit normal-return condition |
|---|---|
| 0 | The final record-access call returned a nonzero address and all six explicit destination writes completed. For selector 5, the preceding selector-1 and selector-2 invocations also returned zero. |
| 1 | Argument validation failed. A diagnostic call precedes the return. |
| 5 | The existing-record condition described below succeeded and the context field at offset 144 was nonzero. A diagnostic call precedes the return. |
| 7 | The final record-access call returned zero. No direct diagnostic call accompanies this return. |

For selector 5, a nonzero result from either recursive invocation is returned immediately.

These statuses describe normal returns visible in the supplied body. Memory faults, dependency exceptions, nonlocal transfers, and failure to return are not assigned status behavior here.

### Access and lifetime preconditions

Every location actually accessed must permit the relevant read or write. The function does not validate the context address, the remapping-byte location, or record addresses returned by dependencies.

A nonzero name reference passes only the local presence check. It must separately satisfy F-1cb585febd84’s requirements when that dependency is reached.

A successful destination must support the specified accesses through byte offset 27. This establishes neither its total allocation size nor its complete layout.

No record handle is returned to the caller. Ownership, allocation, release, and lifetime are dependency-defined. The mode-0 and mode-1 results need not designate the same object.

## Behavior

### Validation and short-circuit order

Validation precedes selector remapping, recursive expansion, and record access.

A zero name reference immediately fails.

The callback values must then satisfy one of these combinations:

| Primary | First paired | Second paired | Accepted |
|---|---|---|---|
| Zero | Zero | Zero | Yes |
| Nonzero | Zero | Zero | Yes |
| Zero | Nonzero | Nonzero | Yes |
| Nonzero | Either paired value nonzero | — | No |
| Zero | Exactly one paired value nonzero | — | No |

Only zero versus nonzero matters; no further restriction on nonzero bit patterns is visible.

Next, the numeric 32-bit index token is increased by one using unsigned arithmetic. The resulting value must be at most 128. Addition wraps modulo \(2^{32}\), so the accepted original representations are:

- 0 through 127, inclusive.
- 4,294,967,295, the all-ones 32-bit representation.

Token 128 and every other representation outside that set fail. This is not a memory-validity check, and the token is not dereferenced.

Only after these checks pass does the function call F-1cb585febd84 with the name reference. Its result is treated as a signed 32-bit measure. A result greater than 255 fails validation.

There is no visible lower-bound check. Negative signed results pass this comparison and are retained for record-access calls. Their semantic validity as a name measure is unknown.

Every explicit validation failure calls F-a061acf353ed with:

1. Context.
2. Status 1.
3. A fixed diagnostic identifying invalid parameters.

The function then returns 1. A failure before name measurement does not call F-1cb585febd84.

### Selector resolution and expansion

After validation, selector handling is:

| Requested selector | Established behavior |
|---|---|
| 4 | Read the remapping byte. A nonzero byte selects effective selector 2; a zero byte selects effective selector 3. |
| 5 | Invoke F-a41ee7bf45a2 with selector 1, then with selector 2 if the first returns zero. If both return zero, continue the outer invocation with effective selector 3. |
| Any other value | Retain the requested selector unchanged. |

The two recursive calls receive the same other seven argument values as the outer invocation. Each repeats validation and, if its earlier checks pass, independently calls F-1cb585febd84.

A nonzero selector-1 result prevents both selector-2 execution and the outer selector-3 operation. A nonzero selector-2 result prevents the outer selector-3 operation.

The outer invocation retains its own previously obtained name measure for selector 3. It does not measure the name again after recursion. Consequently, record-access calls in different operations may receive different measures if dependency results change.

Preserving argument values does not establish preservation of referenced memory. Recursive operations and their dependencies may change context, name storage, records, or globals.

Selectors other than 4 and 5 are not range-checked.

### Initial record access

For each operation that reaches record processing, F-d92bc662c47a is called with these arguments, in order:

1. Context.
2. Name reference.
3. Retained signed name measure.
4. Original index token.
5. Effective selector masked to its low eight bits.
6. Mode 0.

A zero result skips all existing-record reads and proceeds to the mode-1 access.

A nonzero result is treated as an existing-record address. The comparison behavior depends on two separate reads of its selector byte.

### Existing-record comparison and conditional operand

The original returned record address initially supplies the 32-bit comparison operand.

The function then performs the following observable sequence:

1. Read the unsigned byte at offset 2 from the original record address.
2. Compare that byte with the full effective selector.
3. Only if that first comparison matches, read the signed 16-bit field at offset 0 from the original record and replace the comparison operand with that field’s sign-extended 32-bit representation.
4. Read the unsigned byte at offset 2 from the original record address again.
5. If the second selector comparison matches, compare the current operand with the original index token.
6. Only if both comparisons in the preceding step succeed, read the 32-bit context field at offset 144.

The second selector-byte read remains based on the original record address even when the comparison operand has been replaced.

The resulting cases are:

| First selector comparison | Second selector comparison | Condition for examining the context field |
|---|---|---|
| Matches | Matches | Sign-extended record index equals the index token. |
| Matches | Does not match | Never; the record index was nevertheless read. |
| Does not match | Matches | Original returned record address equals the index token; no record-index read occurred. |
| Does not match | Does not match | Never. |

Thus, “matching record index” is an accurate simplification only when the first selector comparison matches. If the two selector reads differ from nonmatching to matching, the equality uses the original record address, not the record’s index field.

For stable selector-byte contents, both reads agree. With a matching selector byte, the signed 16-bit index is sign-extended before comparison. A stored all-ones 16-bit index therefore compares equal to the accepted all-ones 32-bit index token.

Each selector-byte comparison uses the full effective selector, not merely its low eight bits. An effective selector greater than 255 cannot equal any unsigned byte, although its low eight bits are still passed to F-d92bc662c47a.

### Context restriction and preparation call

When the existing-record condition above succeeds, the function reads the 32-bit context field at byte offset 144.

- If nonzero, call F-a061acf353ed with context, status 5, and a fixed diagnostic concerning inability to delete or modify a user function; then return 5.
- If zero, call F-ad6543e399ad with context and continue to the final record access.

If the existing-record condition fails, neither the context-field read nor F-ad6543e399ad occurs.

The purpose and effects of F-ad6543e399ad are not established. It must not be assumed to release memory, preserve records, preserve globals, or leave context unchanged.

### Final record access and writes

Unless an earlier path returned, call F-d92bc662c47a with the same first five argument values as the initial access and mode 1.

These are retained argument values, not a guarantee that referenced data or global state is unchanged.

If the result is zero, return 7.

Otherwise, use the returned address as the destination and perform these writes in order:

| Order | Byte offset | Width | Value |
|---|---:|---:|---|
| 1 | 4 | 8 bits | Zero |
| 2 | 20 | 32 bits | First paired callback value |
| 3 | 24 | 32 bits | Second paired callback value |
| 4 | 16 | 32 bits | Primary callback value |
| 5 | 8 | 32 bits | Payload |
| 6 | 0 | 16 bits | Low 16 bits of the index token |

Return 0 after the last write.

Accepted tokens 0 through 127 are stored unchanged in the 16-bit field. The accepted all-ones token is stored as an all-ones 16-bit value.

The function does not explicitly write the selector byte at offset 2. Any initialization or modification of that byte by a dependency is outside the established local behavior.

## Side effects

The explicit local effects are the ordered record writes, conditional memory reads, and dependency calls described above. Recursive expansion may execute the same effects for multiple effective selectors.

Important ordering properties:

- Validation completes before any record-access call in that invocation.
- Selector-4 remapping reads its byte only after validation.
- A mode-0 access precedes the mode-1 access.
- The context restriction is inspected only after the second selector comparison and operand equality succeed.
- Status 5 is returned before mode 1 and before the destination writes in that operation.
- F-ad6543e399ad, when reached, runs before mode 1.
- A zero mode-1 result prevents the six explicit destination writes.
- Selector-5 operations execute in order 1, 2, 3, stopping on an earlier nonzero return.

There is no explicit rollback, allocation, deallocation, ownership transfer, or direct callback invocation in the body. This absence does not establish corresponding behavior for dependencies.

In particular, a later failure does not prove either persistence or reversal of earlier effects. Subsequent recursive operations and dependencies may overwrite, undo, or otherwise change earlier state. The function alone establishes neither transactional behavior nor guaranteed nontransactional persistence.

No synchronization or atomicity is visible. The two selector-byte reads are distinct; coherent snapshots and concurrent-access safety are not established.

## Dependencies

| Opaque function ID | Established use | Unresolved behavior |
|---|---|---|
| F-a061acf353ed | Receives context, status 1 or 5, and the corresponding fixed diagnostic. Its result is ignored. | Diagnostic destination, memory effects, global preservation, and normal-return guarantees. |
| F-1cb585febd84 | Receives the name reference and supplies the signed 32-bit measure. | Name representation, measurement semantics, negative-result meaning, and side effects. |
| F-d92bc662c47a | Receives the six arguments listed above in modes 0 and 1; supplies zero or a record address. | Lookup and creation policy, mutation in either mode, initialization, ownership, allocation, release, and lifetime. |
| F-ad6543e399ad | Receives context after the existing-record condition succeeds and the restriction field is zero. Its result is ignored. | Preparation or cleanup semantics, memory release, and preservation of context, records, or globals. |
| F-a41ee7bf45a2 | Recursively invoked for selectors 1 and 2 during selector-5 expansion. | No separate contract beyond the behavior specified here. |

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. They assume required memory accesses are valid and dependencies return normally.

1. **Missing name.** A zero name reference causes a status-1 diagnostic and return 1. Neither name measurement nor record access occurs.

2. **Invalid callback mixture.** A nonzero primary callback with either paired callback nonzero fails before name measurement. A zero primary with exactly one paired callback nonzero also fails.

3. **No callbacks.** Three zero callback values pass the combination check. This does not alone establish deletion semantics or successful completion.

4. **Index boundaries.** Tokens 0, 127, and all-ones pass the numeric check. Token 128 fails before name measurement. No token dereference is required.

5. **Name-measure boundaries.** A measure of 255 passes the upper-bound check; 256 fails with status 1. A negative signed result passes this check and is forwarded unchanged if processing continues.

6. **Selector remapping.** Requested selector 4 becomes 2 for a nonzero remapping byte and 3 for a zero byte. Each subsequent record-access call receives that effective selector’s eight-bit projection.

7. **Stable restricted record.** If both selector reads match, the sign-extended record index equals the token, and the context restriction is nonzero, the function returns 5 after the diagnostic. It does not call F-ad6543e399ad or perform mode 1.

8. **Stable unrestricted record.** Under the same matching conditions with a zero restriction field, F-ad6543e399ad runs before mode 1. A nonzero writable mode-1 result leads to the six ordered writes and return 0.

9. **First selector read fails, second matches.** No record-index read occurs. If the original returned address equals the index token, the context restriction is examined. If that address differs from the token, it is not examined, regardless of the stored record index.

10. **First selector read matches, second fails.** The signed record index is read, but the context restriction is not examined and F-ad6543e399ad is not called. Processing continues to mode 1.

11. **Wide selector.** Effective selector 256 is passed to record access as zero. No unsigned selector byte can match the full value 256, so the existing-record condition cannot succeed. Mode 1 and successful writes remain possible.

12. **Unavailable destination.** A zero mode-1 result returns 7 without the destination writes. Any earlier dependency effects have no persistence or reversal guarantee from this function.

13. **Stopped expansion.** If selector 1 returns zero and selector 2 returns 7, the outer result is 7 and selector 3 is not attempted. The resulting state of selector-1 effects is not established.

14. **Successful expansion.** If all operations return zero, selectors 1, 2, and 3 are processed in order, and the outer result is zero. This does not prove that all earlier destination contents remain present afterward.

## Uncertainties

The following remain **documented_only**:

- Application meanings of selectors, callbacks, payload, context, and “registration.”
- Name encoding, storage requirements, measurement semantics, and negative measures.
- Full record layout, byte order, alignment requirements, and allocation size.
- Record ownership, lifetime, and relationships between the two record-access results.
- Meaning and mutability of the remapping byte and context restriction field.
- Dependency effects, including allocation, release, rollback, and global mutation.
- Transaction, persistence, isolation, concurrency, and reentrancy guarantees.
- Original interface declarations beyond the widths and operations evidenced here.

**Inferred intent:** the function appears to install or update a named registration, permit alternative callback arrangements, and conditionally prepare existing state before replacement. These interpretations do not establish deletion, memory release, callback execution, or durable state changes.

## Evidence

Established behavior comes from static inspection of the supplied function body: validation predicates, arithmetic widths, dependency arguments, selector branches, conditional reads, destination offsets and widths, write order, and normal return paths.

The divergent-read behavior follows from the comparison operand retaining the original returned address unless the first selector comparison succeeds. The source separately performs the second selector-byte read before deciding whether to compare that operand with the index token.

The body contains no explicit rollback. Because dependency implementations are not supplied, that fact establishes neither persistence of earlier changes nor transaction semantics.

No runtime tests have been performed. Acceptance examples describe conditional consequences of the static evidence, not observed executions.
