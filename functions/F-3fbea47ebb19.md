<!-- Trusted provenance: F-3fbea47ebb19; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3fbea47ebb19

## Purpose

F-3fbea47ebb19 searches a sequence of 32-bit outer values and, for each outer value, a derived interval of 32-bit candidate values. It prepares two temporary representations, submits each candidate to a processing dependency, and invokes a second dependency only when candidate processing reports a nonzero result.

The function returns:

- **1** when candidate processing and the immediately following acceptance call both return nonzero.
- **0** when the current outer value is greater than a freshly read upper limit.

These are the only explicit return outcomes. Termination is not guaranteed: unsigned wraparound, changing limits, dependency behavior, or an inner interval ending at the maximum unsigned value can prevent either return.

The interpretation of this operation as a search is inferred intent. The iteration, dependency-call ordering, and return conditions are established static behavior.

## Interface

The function has three inputs and a 32-bit return value.

| Input | Representation | Observable use |
|---|---|---|
| Context address | Address-bearing value represented by a 32-bit signed integer in the supplied listing | Passed to F-b1f7d71d6d15. An address 28 bytes beyond it is passed to two setup dependencies. |
| Range/control address | Pointer to 32-bit unsigned words | Word at byte offset 0 supplies the initial outer value. Word at byte offset 8 supplies the repeatedly read upper limit. The pointer itself is passed to F-c457901e4032. |
| Auxiliary value | Opaque 32-bit value | Forwarded unchanged to F-b1f7d71d6d15 and F-c457901e4032. Its interpretation is unknown. |

The word at byte offset 4 of the range/control object is not directly read or written by this function. Dependencies receiving the object may nevertheless access it.

The function uses two distinct temporary storage regions, each 64 bytes long, and one addressable 32-bit unsigned candidate word. The temporary regions have automatic, invocation-local lifetimes. Their alignment and internal format are not established by the supplied evidence.

The arithmetic contract below uses the 32-bit integer model represented by the supplied listing:

- Unsigned words range from 0 through 4,294,967,295.
- Unsigned addition, subtraction, and increment wrap modulo 4,294,967,296.
- The setup dependency’s signed integer result is converted to unsigned for the candidate-interval arithmetic.
- Interval comparisons and the outer-limit comparison are unsigned.

Valid execution requires that all directly accessed memory and all memory accessed by dependencies satisfy their respective requirements. The evidence does not establish those dependencies’ complete memory requirements. In particular, treating the auxiliary value as an owning pointer, a borrowed pointer, or a non-pointer token would be an unsupported assumption.

## Behavior

### Setup and initial reads

Setup occurs before the first range-limit check, in this order:

1. F-460dba384a3e is called with the address 28 bytes beyond the context address. Its signed integer result is retained as the interval adjustment.
2. F-202f56eb285c is called with the first 64-byte temporary region as destination, the same context-relative address as source, and the integer value 1.
3. The 32-bit unsigned word at byte offset 0 of the range/control object is read once to obtain the initial outer value.

The first dependency can therefore affect memory observed by the second dependency or by the subsequent initial-value read. Likewise, the second dependency runs before the initial value is read. No assumption that either dependency preserves globals or aliased input memory is part of this contract.

Neither setup call is conditional on the range being nonempty. An initial outer value above the upper limit still incurs both setup calls.

The return value of F-202f56eb285c, if any, is not inspected here. The caller performs no explicit initialization of either temporary region before passing it to that dependency.

### Outer-value admission

At the beginning of every outer iteration, the function reads the unsigned word at byte offset 8 of the range/control object.

If that freshly read limit is less than the current outer value, the function returns 0 immediately. No candidate representation, candidate processing, or acceptance call is performed for that outer value.

Otherwise, it constructs the candidate interval for the current outer value.

The starting value is not reloaded from byte offset 0 on later iterations. A dependency changing that word does not directly replace the function’s current outer value. By contrast, changes to the word at byte offset 8 can affect admission of subsequent outer iterations.

There is no direct reread of the upper limit during an inner candidate iteration.

### Candidate interval

Let:

- **T** be the current outer value.
- **R** be the setup result interpreted as an unsigned 32-bit value.

The candidate word is initially assigned **T − R**, with unsigned wraparound. The inner loop admits a candidate while its current unsigned value is at most **T + R**, also calculated with unsigned wraparound.

This is an ordinary unsigned comparison between wrapped endpoints. It is **not** a circular-interval traversal.

Consequences include:

- If the wrapped lower endpoint exceeds the wrapped upper endpoint, no candidate is processed for that outer value.
- If the upper endpoint is 4,294,967,295, the comparison cannot reject any 32-bit candidate value. Incrementing the candidate cannot terminate that inner loop.
- A negative setup result is not rejected, clamped, or interpreted as an error. Its unsigned conversion participates in both endpoint calculations.
- Even a positive adjustment can produce an empty interval near either end of the unsigned domain.

The candidate word is passed by address to F-202f56eb285c. The supplied evidence does not establish that dependency as read-only with respect to this word. Consequently, the value used by the subsequent candidate increment can differ from the value originally presented for transformation.

### Per-candidate processing

For each admitted candidate, the following sequence occurs:

1. F-202f56eb285c receives the second 64-byte temporary region as destination, the address of the candidate word as source, and the integer value 1.
2. F-b1f7d71d6d15 receives the context address, the first temporary region, the second temporary region, and the auxiliary value.
3. If F-b1f7d71d6d15 returns zero, the acceptance dependency is skipped.
4. If F-b1f7d71d6d15 returns nonzero, F-c457901e4032 receives the auxiliary value and the range/control address.
5. If that acceptance call also returns nonzero, this function returns 1 immediately.

A nonzero candidate-processing result alone does not cause success. If the acceptance call returns zero, iteration continues.

On continuation, the current contents of the candidate word are incremented modulo 4,294,967,296, and the inner comparison is evaluated again.

The first temporary representation is prepared only once per invocation. It is not refreshed between candidates or outer iterations. The second temporary region is reused for every candidate. Neither region is explicitly cleared between uses. Any contents retained or modified by dependencies may therefore matter to later calls.

### Advancing the outer value

When the inner comparison rejects the current candidate, the outer value is incremented modulo 4,294,967,296. Execution then returns to the outer-limit read.

There is no separate outer-overflow check. With a stable upper limit below 4,294,967,295, ordinary exhaustion can return 0 after the outer value advances beyond that limit, provided all admitted inner loops terminate. With a stable upper limit equal to 4,294,967,295, the outer comparison can never return 0, including after wraparound.

## Side effects

The function directly:

- Writes its invocation-local candidate word.
- Supplies writable temporary storage to dependencies.
- Reads the initial range word once, after setup.
- Reads the upper-limit word at each outer admission point.
- Invokes dependencies in the order described above.

It contains no direct store to the context object, range/control object, or auxiliary-designated state. This does **not** establish that those objects remain unchanged: dependencies receive addresses or values that may permit mutation, and the inputs may alias.

There is no explicit allocation, deallocation, ownership transfer, lock operation, or global-variable access in the visible body. Dependencies’ behavior in these areas is unknown.

Pointers to the temporary regions and candidate word refer to invocation-local storage. The function does not extend their lifetime beyond its return. Whether any dependency retains these pointers is documented_only; safe deferred use cannot be assumed.

A return of 0 does not imply absence of side effects. Setup always occurs first, and earlier unsuccessful candidates may already have caused dependency-visible changes. Likewise, an acceptance call returning zero is not evidence that it performed no work.

## Dependencies

| Opaque ID | Invocation contract visible here | Result handling |
|---|---|---|
| F-460dba384a3e | Receives the context-relative address at byte offset 28, once during setup. | Signed integer result becomes the interval adjustment; no error check is applied. |
| F-202f56eb285c | Called once with the first temporary region and the context-relative source; then once per admitted candidate with the second temporary region and the candidate-word address. Every call receives the third argument 1. | No result is inspected. Written extent, source mutation, and representation format are unknown. |
| F-b1f7d71d6d15 | Receives context, both temporary regions in preparation order, and the auxiliary value. | Zero skips acceptance; any nonzero value triggers acceptance. |
| F-c457901e4032 | Receives auxiliary value and range/control address, only after a nonzero candidate-processing result. | Zero continues iteration; any nonzero value causes return 1. |

The names “preparation,” “processing,” and “acceptance” describe their positions in this function’s control flow, not independently established dependency semantics.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Candidate sequences assume dependencies return normally and do not change the candidate word or relevant loop inputs unless stated otherwise.

1. **Initially excluded range.**  
   After setup, the initial outer word is 12 and the first upper-limit read is 11. The function returns 0. Both setup dependencies have already run; no candidate-processing or acceptance call occurs.

2. **Zero adjustment and immediate success.**  
   The adjustment is 0, the initial outer value is 7, and the upper limit is at least 7. Candidate 7 is prepared and processed. If candidate processing and acceptance both return nonzero, the function returns 1 without incrementing either loop value.

3. **Failed processing skips acceptance.**  
   The adjustment is 0 and the admitted outer value is 7. If processing candidate 7 returns zero, acceptance is not called. The candidate advances to 8, which exceeds the inner upper endpoint of 7, so the outer value then advances.

4. **Nonzero processing is insufficient.**  
   For an admitted candidate, processing returns a negative integer and acceptance returns zero. The processing result still counts as nonzero, so acceptance is called, but the function does not return 1.

5. **Ordinary inclusive interval.**  
   With outer value 10 and adjustment 2, the candidate sequence is 8, 9, 10, 11, 12. If none succeeds, candidate 13 fails the inner comparison and the outer value advances to 11.

6. **Wrapped endpoints produce an empty interval.**  
   With outer value 0 and adjustment 1, the lower endpoint is 4,294,967,295 and the upper endpoint is 1. No candidate is processed for that outer value.

7. **Maximum upper endpoint prevents inner exhaustion.**  
   With outer value 4,294,967,294 and adjustment 1, the inner upper endpoint is 4,294,967,295. Assuming dependencies keep returning normally without success, the candidate wraps from the maximum value to zero and remains admitted. Outer advancement is never reached.

8. **A negative adjustment can create a broad interval.**  
   With outer value 5 and setup result −1, unsigned conversion makes the lower endpoint 6 and upper endpoint 4. That interval is empty. With outer value 0 and the same result, the endpoints instead become 1 and 4,294,967,295, producing a non-exhausting inner loop unless success or some other dependency behavior interrupts it.

9. **A changed upper limit takes effect at the next outer admission.**  
   During outer value 10, a dependency changes the range/control upper limit to 9 and returns without success. This change does not itself interrupt the current inner loop. If that loop terminates, the outer value advances to 11; the next limit read then causes return 0.

## Uncertainties

The following remain documented_only:

- The meanings and complete layouts of the context and range/control objects.
- The meaning of the auxiliary value.
- The internal formats and required initialized extents of both 64-byte temporary regions.
- Whether dependencies mutate source arguments, temporary regions, shared state, or aliased inputs.
- Whether dependencies retain pointers, allocate or release memory, block, fail to return, or signal exceptional conditions.
- The validity requirements and overflow behavior for forming the context-relative address outside an ordinary valid address range.
- Thread-safety, reentrancy, and synchronization requirements.

No explicit null checks, range-object size checks, adjustment validation, or dependency-error translation are visible. This absence is not a guarantee that invalid inputs are supported.

The algorithm’s apparent intent is neighborhood search around successive outer values. The unsigned endpoint behavior is established; whether wraparound cases are intentional is unknown.

## Evidence

This contract is derived solely from static inspection of the supplied function body and the supplied opaque dependency mapping.

Established evidence includes call order, directly accessed offsets, temporary-storage sizes, integer comparisons and wraparound, short-circuit acceptance, and explicit return values. Search-oriented terminology is inferred intent. Dependency internals and ownership semantics are not established.

No runtime tests have been performed.
