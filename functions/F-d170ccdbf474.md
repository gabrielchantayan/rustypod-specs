<!-- Trusted provenance: F-d170ccdbf474; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d170ccdbf474

## Purpose

F-d170ccdbf474 drives a dependency-controlled state record until a mode-specific continuation condition no longer holds. It optionally writes a signed integer result derived from the last primary dependency result and the record’s first word.

The function has two paths:

- **Zero mode:** repeatedly invoke the primary dependency until the record’s status is positive, then clear that status and invoke the primary dependency again.
- **Nonzero mode:** conditionally invoke a second dependency when the status is below two, then clear the status and invoke the primary dependency.

Both paths may repeat. Neither path has a fixed iteration limit, timeout, or explicit error-reporting mechanism.

These are established source-level behaviors. The application meaning of the state record, its status values, and the derived output is unknown. Descriptions such as “status” and “primary dependency” are descriptive labels, not recovered domain semantics.

## Interface

### Inputs

The function accepts three arguments:

1. **Container reference.** An address identifying an object whose field at byte offset 4 holds a pointer to the state record.
2. **Mode.** A signed integer. Zero selects the zero-mode path; every nonzero value selects the other path. Positive and negative nonzero values are equivalent for this selection.
3. **Optional output reference.** Either a writable integer address or null. Null suppresses the final output calculation and write, but does not suppress state processing or dependency calls.

The function produces no direct return value.

### Relevant layout and widths

The supplied representation uses a four-byte integer/pointer layout:

| Location | Observed use |
|---|---|
| Container, byte offset 4 | Pointer to the state record |
| State record, byte offset 0 | Signed integer used only when calculating a nonzero-result output |
| State record, byte offset 12 | Signed integer status, read for control flow and explicitly written with zero |
| Output reference | Destination for one integer result |

The integer comparisons and output arithmetic are interpreted in this represented 32-bit signed-integer model. The source does not independently establish the original target ABI, pointer alignment requirements, or machine-level load/store instructions. This contract does not claim portability of these offsets to an arbitrary host ABI.

No other state-record fields are directly accessed by F-d170ccdbf474. Dependencies may access additional fields.

### Preconditions

For a defined execution:

- The container reference must permit reading the pointer at byte offset 4.
- That pointer must identify a state record whose relevant fields remain accessible throughout processing.
- The status field must permit both reads and writes.
- If the output reference is nonnull, it must permit the final integer write.
- If the final primary result is nonzero and the output reference is nonnull, the record’s first word must be readable and the output arithmetic must have defined semantics for the values involved.
- Every dependency invocation reached during execution must be valid for the supplied state-record pointer.

There are no explicit checks enforcing these conditions. Invalid addresses, dependency failure, nontermination, or arithmetic outside the supported signed range are not converted into a defined error result by this function.

## Behavior

### Record selection

The function first obtains the state-record pointer from the container’s field at byte offset 4. It retains that selected pointer for subsequent processing.

The container field is not reread by the visible body. If a dependency changes that field to refer to another record, subsequent visible accesses and dependency arguments still use the originally selected pointer.

Mode selection follows this record selection. The output reference is not checked until state processing has finished.

### Zero-mode processing

When the mode is zero, each processing cycle has a preparation stage followed by a primary invocation.

During preparation, the function examines the record’s signed status. While that status is less than one, it invokes F-2512af5fa847 with the selected record pointer and examines the status again.

Consequences of this preparation rule:

- Zero and every negative status require a preparation invocation.
- Any positive status ends preparation.
- Preparation may make no calls, one call, or arbitrarily many calls.
- Return values from preparation invocations do not influence control flow or the eventual output.
- A nonzero preparation return does not stop preparation if the status remains below one.

After preparation ends, the function writes zero to the status field. It then invokes F-2512af5fa847 again and retains this invocation’s signed integer result.

The cycle repeats only when that retained result is zero and the post-invocation status is strictly greater than zero and strictly less than three. For a stable signed integer status, this means exactly one or two.

Thus, with a zero retained result:

- Status one or two causes another complete zero-mode cycle.
- Status zero, a negative value, or a value of at least three ends processing.

A nonzero retained result ends processing without requiring the continuation-status comparisons.

A positive status left by the retained-result invocation does not itself require another preparation invocation on the next cycle. If the repeated cycle begins with status one or two, preparation is skipped; the function still clears the status before invoking the primary dependency again.

### Nonzero-mode processing

When the mode is nonzero, each cycle begins by examining the signed status.

If the status is less than two, the function invokes F-e9daa5a15ff6 once with the selected record pointer. This includes negative status values, zero, and one.

This preparation step is conditional, not a waiting loop. The function does not keep invoking F-e9daa5a15ff6 until a threshold is reached. Its return value, if any, is not used by the visible function.

Whether or not that invocation occurs, the function next writes zero to the status field and invokes F-2512af5fa847. It retains the primary dependency’s signed integer result.

The cycle repeats only when that result is zero and the post-invocation status is strictly greater than one and strictly less than three. For a stable signed integer status, this means exactly two.

Thus, with a zero retained result:

- Status two causes another complete nonzero-mode cycle.
- Every other signed status ends processing.

A nonzero retained result ends processing without requiring the continuation-status comparisons.

### Condition-evaluation order

For both modes, continuation is assessed in this logical order:

1. Whether the retained primary result equals zero.
2. If so, whether the status is less than three.
3. If so, whether the status exceeds the mode-specific lower bound: zero in zero mode, one in nonzero mode.

The later comparisons are short-circuited when an earlier condition fails.

The “exactly one or two” and “exactly two” descriptions assume the status does not change independently between observations. The supplied source does not establish synchronization or guarantee coherent behavior under concurrent mutation. It also does not establish how repeated source-level field accesses map to physical memory reads.

### Optional output

Once state processing terminates, the function checks the output reference.

If it is null, the function returns immediately. In particular, it does not read the record’s first word for output calculation.

If the output reference is nonnull:

- A zero final primary result produces an output value of zero.
- A nonzero final primary result produces the record’s current first word minus that result minus one.

The relevant primary result is from the final invocation made immediately after an explicit status reset. It is never a result discarded during zero-mode preparation.

For a nonzero primary result, the first word is read after dependency processing has completed. The function does not preserve an earlier snapshot of that word.

The output expression contains two signed subtractions: first subtract the final primary result from the first word, then subtract one. Defined mathematical results require both intermediate and final values to be representable under ordinary signed 32-bit semantics. The supplied source does not justify assuming saturation or modulo wraparound for overflow cases.

## Side effects

The directly visible effects are:

- Reading the state-record pointer from the container.
- Reading the selected record’s status for preparation and continuation decisions.
- Writing zero to that status immediately before every retained-result primary invocation.
- Calling dependencies with the selected record pointer.
- Conditionally reading the selected record’s first word.
- Writing exactly one integer through the output reference when that reference is nonnull and execution reaches completion.

There is no visible allocation, deallocation, ownership transfer, or explicit global access in F-d170ccdbf474 itself.

That absence does not constrain the dependencies. They may modify the record, other reachable memory, or globals; their complete side effects are unknown.

The function performs no explicit release of the container, record, or output storage. Safe execution requires those objects to remain valid across all reached calls. Dependency ownership and lifetime behavior cannot be inferred from this body.

Aliasing is not prohibited by the visible implementation. If the output destination aliases the status or first word, the final output write changes that field after all processing decisions are complete. When output calculation reads the first word, that read precedes the output store. No additional processing follows the final store.

## Dependencies

### F-2512af5fa847

Receives the selected state-record pointer.

It is used in two roles:

- Zero-mode preparation, where its return value is ignored and the status controls repetition.
- The primary invocation immediately following a status reset, where its integer result and subsequent status determine termination.

The visible caller does not enforce any particular status transition. It does not establish that this dependency preserves other record fields, globals, allocation ownership, or record validity.

### F-e9daa5a15ff6

Receives the selected state-record pointer.

It is called only in nonzero mode, once at the start of a cycle whose observed status is below two. No returned value affects the caller’s decisions.

The caller clears the status after this dependency returns, even if the dependency changed it. Other changes may remain visible to the subsequent primary invocation, but their meaning is unknown.

## Acceptance examples

These examples are conditional specifications, not observed executions. Unless otherwise stated, dependency calls return normally, all referenced storage remains valid, statuses are stable between calls, and arithmetic is representable.

### Zero mode with immediate termination

Given zero mode and initial status four:

- No preparation invocation is needed.
- The status is written to zero.
- F-2512af5fa847 is invoked.
- If it returns seven, processing ends regardless of the status it leaves.
- If the current first word is twenty and output is requested, the output is twelve.

### Zero-mode preparation ignores returned results

Given zero mode and initial status zero:

- A preparation invocation returns nine but leaves status zero.
- Preparation continues.
- A second preparation invocation leaves status one.
- The function clears the status and makes a separate retained-result invocation.

The earlier return of nine neither terminates processing nor contributes to the output.

### Zero-mode continuation boundary

If a retained-result invocation returns zero:

| Status left by that invocation | Required outcome |
|---|---|
| −1 | End processing |
| 0 | End processing |
| 1 | Repeat |
| 2 | Repeat |
| 3 | End processing |

If processing ends and output is requested, the output is zero.

### Nonzero mode uses a single conditional preparation call

Given any nonzero mode and initial status negative five:

- F-e9daa5a15ff6 is called once.
- Even if it leaves status negative five, no additional preparation call occurs within that cycle.
- The caller clears the status and invokes F-2512af5fa847.

### Nonzero-mode continuation boundary

With a zero primary result, status two causes another cycle; status one and status three do not.

If the repeated cycle begins with unchanged status two, F-e9daa5a15ff6 is skipped in that cycle. The status is nevertheless cleared before the primary invocation.

### Null output

Given a null output reference, all applicable preparation, status resets, primary invocations, and continuation decisions still occur. After termination, there is no output store and no first-word read for output calculation.

### Negative primary result

If the final primary result is negative two and the current first word is ten, processing terminates because the result is nonzero. A requested output is eleven.

### Nontermination

If zero-mode preparation always leaves the status below one, the function never reaches its explicit status reset or final output handling.

If retained-result invocations indefinitely return zero and leave a continuing status, processing likewise never reaches the output handling.

## Uncertainties

- **documented_only — Domain meaning:** The semantic meanings of the first word, status values, mode, and derived output are not established.
- **documented_only — Dependency contracts:** Complete effects, failure behavior, ownership rules, and termination guarantees for both dependencies are unavailable.
- **documented_only — ABI and overflow:** The supplied representation supports the four-byte layout described above, but does not independently establish the original ABI or defined machine behavior for overflowing signed arithmetic.
- **documented_only — Concurrency:** No synchronization, atomicity, or supported concurrent-access model is established.
- **documented_only — Exceptional execution:** Invalid-memory behavior, nonlocal exits, and dependency exceptions or faults are not handled explicitly.
- **Inferred intent:** The preparation and repetition rules may represent progress through a larger processing operation. No specific protocol, resource, or application interpretation is justified.

## Evidence

Established behavior comes solely from the supplied static function representation: its field accesses, signed comparisons, dependency calls, status resets, short-circuit continuation conditions, and optional final store.

Dependency bodies and surrounding caller contracts were not supplied. Their behavior has not been assumed beyond what is necessary to describe the visible call interface.

No runtime tests have been performed. The acceptance examples specify conditional outcomes rather than report measured results.
