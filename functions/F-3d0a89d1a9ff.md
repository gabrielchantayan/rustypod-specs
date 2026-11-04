<!-- Trusted provenance: F-3d0a89d1a9ff; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3d0a89d1a9ff

## Purpose

F-3d0a89d1a9ff performs one of two delimiter-dependent processing loops over state accessed through a caller-supplied two-word structure. It delegates all stream inspection, state advancement, and item processing to opaque dependencies.

The selected branch determines the termination condition:

- A first inspection result of `0x44` selects a loop that finishes when an advancement result has low 32 bits equal to `0x3E`.
- Any other first inspection result selects a loop that finishes only after three dependency results match `0x2D`, `0x2D`, and `0x3E` in the prescribed order.

Both successful paths return the 32-bit value `0x22`. A nonzero result from the item-processing dependency causes an immediate return of `0x38`.

**Inferred intent:** the numeric comparisons are compatible with processing delimited textual content. This does not establish a character encoding, a grammar, or the meaning of the returned status values. The behavioral contract is the ordered dependency interaction and state handling below.

## Interface

### Inputs

The function takes two inputs:

1. **State pointer.** A pointer to storage whose first two entries are 32-bit words:
   - **Offset 0:** a word supplied as the first argument to F-558eaa680dc7.
   - **Offset 4:** a word supplied by value to inspection dependencies, and whose address is supplied to advancement dependencies.

2. **Auxiliary word.** An opaque 32-bit value. It is explicitly passed to F-558eaa680dc7 only on the branch selected by an initial inspection result of `0x44`. On that branch, later iterations replace it with the upper 32 bits of an advancement result.

The terms “first word” and “second word” below refer to these offsets, not to established application-level field types. The first word is not proven to be an owned object pointer; the second is not proven to be a stream cursor.

### Output

The directly returned value is 32 bits:

| Value | Established return condition |
|---|---|
| `0x22` | The selected branch reaches its normal termination condition and completes any required trailing calls. |
| `0x38` | An invocation of F-558eaa680dc7 returns a nonzero 32-bit result. |

No other direct return value appears in the supplied evidence. A dependency that does not return normally can prevent either result from being produced.

### Preconditions and validity

The state pointer must designate storage suitable for every access performed by this function and its dependencies. In particular:

- The first word must be readable whenever an item-processing call is reached.
- The second word must be readable whenever a by-value inspection call is reached.
- The address of the second word must be valid for the requirements of F-77013a05b043 and F-f36dbf4e0388.
- The state storage must remain valid across dependency calls that precede later accesses.

There is no visible null check, bounds check, structure-size check, alignment check, or state-validity check. The precise alignment and the amount of storage accessible through values held in either word are not established.

### Ownership

No direct allocation, deallocation, or ownership transfer is visible in this function. Passing a value or an address to a dependency does not establish whether that dependency retains, releases, or replaces any referenced resource.

The caller must not rely on the state remaining unchanged. In particular, dependencies receive the address of the second word and may have additional effects that are not specified here.

## Behavior

### Common entry sequence

The function first calls F-77013a05b043 with the address of the second word.

Its return value is discarded. No validation of that result occurs before execution continues.

After that call returns, the function reads the then-current second word and passes it to F-2979ae55a86a. The resulting 32-bit value selects the branch:

- Exactly `0x00000044`: use the single-result termination branch.
- Every other 32-bit value: use the three-result termination branch.

The second word used for branch selection is therefore read after the initial advancement call. It is not a saved copy of the incoming state.

### Single-result termination branch

This branch executes at least one item-processing attempt. It does not test for termination before the first attempt.

Each iteration proceeds in this order:

1. Read the current first word.
2. Call F-558eaa680dc7 with that word and the current auxiliary word as two explicit arguments.
3. If its 32-bit result is nonzero, return `0x38` immediately.
4. Otherwise, call F-77013a05b043 with the address of the second word.
5. Treat this call’s result as a 64-bit quantity for the two uses described below.
6. Replace the current auxiliary word with bits 63 through 32 of that result.
7. Compare bits 31 through 0 of that result with `0x0000003E`.

If the low 32 bits equal `0x3E`, the function returns `0x22`. Otherwise, it repeats the iteration with the newly extracted auxiliary word.

The extraction uses an unsigned 64-bit right shift by 32 followed by a 32-bit conversion. It is a numeric high-half extraction, not a memory-byte operation. No byte order needs to be assumed to describe it.

The upper half is taken from the same advancement result whose lower half controls repetition. Even when that result terminates the loop, the extraction is present before the termination decision, although no subsequent item-processing call uses the extracted value.

The original auxiliary input is used for the first processing attempt. The initial advancement call’s ignored return value does not replace it.

### Three-result termination branch

Before entering its loop, this branch calls F-2979ae55a86a a second time with the then-current second word. This second result is discarded.

That call is not redundant in the behavioral contract: its invocation occurs even though its result is unused, and its effects are unknown.

The loop then executes at least one item-processing attempt. Each iteration begins by reading the current first word and calling F-558eaa680dc7 with that word as its sole explicit argument in the supplied evidence.

If the processing result is nonzero, the function immediately returns `0x38`. It makes no advancement or termination-probe call for that iteration.

After a zero processing result, the function performs a short-circuit termination check:

1. Call F-77013a05b043 with the address of the second word and compare its 32-bit result with `0x0000002D`.
2. Only if that result equals `0x2D`, read the current second word, call F-3b19cbd01e9c with that value, and compare its 32-bit result with `0x0000002D`.
3. Only if that result also equals `0x2D`, call F-f36dbf4e0388 with the address of the second word and compare its 32-bit result with `0x0000003E`.

All three comparisons must match in that order to end the loop. Otherwise, another iteration begins with an item-processing call.

In particular:

- A first comparison mismatch suppresses both subsequent probes.
- A second comparison mismatch suppresses the third probe.
- A third comparison mismatch causes repetition after that third dependency has already run.
- No explicit rollback or state restoration occurs after a partially matching termination check.

Once all three comparisons match, the function performs two trailing calls:

1. Read the current second word and call F-2979ae55a86a with it.
2. Call F-77013a05b043 with the address of the second word.

Both return values are ignored. The function then returns `0x22`.

Consequently, matching the three-result condition alone is not sufficient for the caller to observe a successful return: both trailing dependencies must also return normally.

### Widths and arithmetic

The state entries and auxiliary value are 32 bits. The processing results, branch-selection result, and termination comparisons are evaluated as 32-bit values.

The single-result branch additionally uses a 64-bit result from F-77013a05b043. Its low and high halves have distinct roles:

- Low half: equality test against `0x3E`.
- High half: next auxiliary word.

In the other branch, only a 32-bit result from that dependency is used. The evidence does not establish a single canonical return-type declaration for all its call sites.

There is no visible counting arithmetic, accumulation, length calculation, or overflow-dependent loop control. Equality against the listed constants is exact; a 64-bit result whose low half is `0x3E` satisfies the single-result termination comparison regardless of its upper half.

## Side effects

The observable dependency-call order is part of the contract. Calls with ignored results must not be omitted merely because their returned values are unused.

This function contains no explicit assignment to either state word. Nevertheless:

- F-77013a05b043 receives the address of the second word on entry, during loop advancement, and on the successful trailing path of the three-result branch.
- F-f36dbf4e0388 receives that address during eligible termination checks.
- Any dependency may have effects beyond its explicit argument locations; those effects are not established here.

The first word is read again for each item-processing call. The function does not visibly cache it across iterations. The second word is likewise read at each by-value inspection call after preceding dependencies have returned.

On a processing failure, the function does not visibly undo prior changes or perform local cleanup. Any effects from the initial calls, previous iterations, and the failing processing call may remain.

There is no visible end-of-input, cancellation, timeout, or iteration-limit handling. If processing continues to report zero and the selected termination condition never occurs, the loop can continue indefinitely. A dependency may implement additional termination behavior, but that is outside the established contract.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-77013a05b043 | Receives the address of the second word. Called unconditionally on entry, after each successful item-processing attempt, and once after successful termination of the three-result branch. One branch uses a 64-bit result; other uses discard the result or compare its 32-bit value. |
| F-2979ae55a86a | Receives the current second word by value. Its first result selects the branch. On the three-result branch, two additional calls occur: one before the loop and one after the termination condition matches. Those additional results are ignored. |
| F-558eaa680dc7 | Processes using the current first word. A nonzero 32-bit result causes immediate return of `0x38`. The single-result branch explicitly supplies an auxiliary second argument; the three-result branch shows only one explicit argument. |
| F-3b19cbd01e9c | Receives the current second word by value, but only after the first termination comparison in the three-result branch matches `0x2D`. Its result must also equal `0x2D` to permit the final probe. |
| F-f36dbf4e0388 | Receives the address of the second word, but only after both preceding termination comparisons match `0x2D`. Its result must equal `0x3E` to terminate that loop. |

No dependency is established to be pure, nonmutating, nonblocking, allocation-free, or ownership-neutral.

## Acceptance examples

These are conditional contract examples, not executed tests. Each assumes valid storage and normal return from every dependency reached.

### First iteration succeeds in the single-result branch

Suppose the initial advancement returns normally and the subsequent branch-selection result is `0x44`. The first item-processing call receives the current first word and the original auxiliary input.

If processing returns zero and the next advancement returns the 64-bit value `0x123456780000003E`, the function returns `0x22`. The low half matches the termination value; the high half does not prevent termination.

No three-result probe dependencies are invoked.

### Auxiliary value changes between iterations

Suppose the single-result branch is selected with auxiliary input `0xAABBCCDD`.

If the first processing call succeeds and the following advancement returns `0x1122334400000041`, the function repeats. The next processing call receives auxiliary value `0x11223344`, not the original input.

If the first word changed during an intervening dependency call, this next processing call uses its current value.

### Processing failure stops before advancement

On either branch, suppose an item-processing call returns `0xFFFFFFFF`.

The function returns `0x38` without making that iteration’s following advancement call. It does not return the dependency’s failure value unchanged.

On the three-result branch, neither conditional termination probe is reached for that iteration.

### First termination comparison mismatches

Suppose the three-result branch is selected, its extra pre-loop inspection returns normally, and item processing returns zero.

If the next advancement result is `0x2C`, the function repeats without calling F-3b19cbd01e9c or F-f36dbf4e0388.

### Partial termination match

Suppose advancement returns `0x2D`, followed by an F-3b19cbd01e9c result of `0x2D`, followed by an F-f36dbf4e0388 result of `0x40`.

The function repeats rather than returning success. The final probe has already occurred, and no explicit restoration of the second word is performed.

### Complete three-result termination

Suppose a successful processing attempt is followed by results `0x2D`, `0x2D`, and `0x3E` from the three ordered termination calls.

The function then invokes F-2979ae55a86a with the current second word and invokes F-77013a05b043 with its address. If both calls return normally, the function returns `0x22`, irrespective of their returned values.

## Uncertainties

The following are **documented_only** limitations:

- The application-level meaning of both state words and the auxiliary input is unknown.
- The encoding and grammatical meaning of the compared constants are unknown.
- The semantic names of return codes `0x22` and `0x38` are unknown.
- The exact dependency prototypes and calling convention are not established. In particular, F-558eaa680dc7 has differing explicit argument counts across branches, and F-77013a05b043 has differing observed result widths. This contract does not invent a default second argument or resolve that ABI uncertainty.
- The effects, ownership rules, global-state access, and failure mechanisms of all dependencies are unknown.
- Whether a probe observes, consumes, rewinds, or otherwise changes an underlying stream is unknown. Ordered calls are established; delimiter consumption semantics are not.
- Behavior for invalid pointers, exhausted input, malformed content, or dependency-internal failures is not defined by the supplied evidence.
- No external liveness guarantee establishes that either loop eventually terminates.

## Evidence

Established static evidence consists of the supplied function body: its 32-bit state-word accesses, opaque dependency calls, branch comparisons, short-circuit conditions, 64-bit high-half extraction, and direct return constants.

The purpose description distinguishes these observations from inferred text-processing intent. No external dependency implementation, runtime trace, trusted type declaration, or prior behavioral document was used to fill the documented unknowns.

No runtime tests have been performed.
