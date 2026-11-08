<!-- Trusted provenance: F-d623afbb5d71; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d623afbb5d71

## Purpose

F-d623afbb5d71 implements an ordered candidate-attempt policy. It obtains one classification, maps that classification to an initial numeric candidate, and attempts that candidate. If the attempt returns zero, it tries the remaining candidates in ascending order from 1 through 8, excluding the initial candidate. The first nonzero attempt result stops processing.

The classification is obtained exactly once per invocation. Fallback does not reclassify, even if an attempt changes state that could affect a subsequent classification.

The meanings of the classifications, candidates, and attempted operation are not established. Terms such as format, device, protocol, or resource type would introduce unsupported semantics.

Possible initial candidates are 0, 1, 2, 3, 5, 6, and 8. Candidates 4 and 7 are attempted only during fallback. Candidate 0 is attempted only when selected initially.

## Interface

### Inputs

The function receives two opaque 32-bit arguments.

| Input | Established treatment |
|---|---|
| First argument | Forwarded unchanged as the first argument of every F-50f346be9118 call. |
| Second argument | Forwarded unchanged as the second argument of every F-50f346be9118 call. |

Neither argument is directly compared, dereferenced, validated, or used in arithmetic by this function.

The supplied representation does not establish whether either argument is a scalar, address, handle, or encoded value. It establishes no pointed-to layout, field offsets, alignment, encoding, nullability, or buffer size. Forwarding an argument unchanged does not establish that storage reachable through it remains unchanged.

### Output

The function has no explicit return value. It does not return the classification, initial candidate, final attempted candidate, or attempt result.

Normal completion therefore does not distinguish, through the explicit return interface, between:

- An attempt returning nonzero.
- Exhaustion after every eligible attempt returns zero.

Any externally available distinction would have to arise from dependency effects or another mechanism not established here.

### Dependency-facing values

- F-c453b9ccc22e receives no explicit arguments and supplies a 32-bit classification value.
- F-50f346be9118 receives the original two 32-bit arguments followed by a 32-bit candidate value.
- The attempt result is represented as a 32-bit integer and is tested only for equality with zero.

Every nonzero attempt result has the same stopping effect. There is no positive-only check, particular success constant, bit-mask interpretation, or propagation of the result to the caller.

These widths describe the supplied static representation. Machine-level calling convention, register use, stack layout, and byte order are not established.

### Preconditions

No local input validation is visible. The dependencies must accept the arguments and call sequences reached during execution for normal completion to be possible.

Any restrictions on input values, referenced storage, ownership, or permitted repeated calls remain dependency-specific and documented_only. The function does not establish those restrictions or enforce them.

## Behavior

### Classification and initial selection

F-c453b9ccc22e is called exactly once before the initial attempt. Its result selects the initial candidate according to this complete mapping. All numbers below are decimal.

| Classification values | Initial candidate |
|---|---:|
| 0, 1, 10, 11 | 6 |
| 2, 8, 9, 15, 16, 19, 20, 22, 23 | 2 |
| 3, 4, 6, 7, 12, 13, 21, 24, 25, 27, 28, 30, 32, 33, 34, 36, 37, 38 | 5 |
| 5, 29, 35 | 8 |
| 14 | 3 |
| 26, 31 | 1 |
| 17, 18, and every value not otherwise listed | 0 |

The mapping uses exact values. Adjacent classifications need not select adjacent or identical candidates. For example, classifications 13, 14, and 15 select candidates 5, 3, and 2 respectively.

Every unlisted classification selects candidate 0; it does not cause rejection or an early return. This applies to any 32-bit classification representation that does not match a listed value, without requiring a signed interpretation.

The selected initial candidate remains fixed throughout the invocation. Neither zero results nor dependency-mediated state changes cause another classification call.

### Initial attempt

After classification returns normally, F-50f346be9118 is called with the two original arguments and the selected initial candidate.

- A nonzero result causes immediate normal return.
- A zero result starts fallback.

Thus, at least one attempt is made whenever classification returns normally and execution proceeds normally to the attempt call.

### Fallback

Fallback starts by considering candidate 1, regardless of the initial candidate. It then considers candidates in ascending order through 8.

The initial candidate is skipped if it belongs to that range. Skipping it does not invoke F-50f346be9118, inspect a new attempt result, or reclassify.

For each eligible candidate:

- The attempt receives the original two argument values and that candidate.
- A nonzero result ends the invocation immediately.
- A zero result allows consideration of the next candidate.

Assuming every attempt returns normally with zero, the complete attempt orders are:

| Initial candidate | Complete attempt order |
|---:|---|
| 0 | 0, 1, 2, 3, 4, 5, 6, 7, 8 |
| 1 | 1, 2, 3, 4, 5, 6, 7, 8 |
| 2 | 2, 1, 3, 4, 5, 6, 7, 8 |
| 3 | 3, 1, 2, 4, 5, 6, 7, 8 |
| 5 | 5, 1, 2, 3, 4, 6, 7, 8 |
| 6 | 6, 1, 2, 3, 4, 5, 7, 8 |
| 8 | 8, 1, 2, 3, 4, 5, 6, 7 |

Consequences of this ordering:

- No candidate is attempted more than once during a normally progressing invocation.
- Initial candidate 0 permits at most nine attempt calls.
- Every other possible initial candidate permits at most eight attempt calls.
- Candidate 9 is never passed to the attempt dependency.
- Fallback does not resume numerically after the initial candidate; it starts at 1 and omits the already attempted candidate.
- The classification dependency is not called between any two attempts.

These are call-count and ordering limits, not duration guarantees. A dependency may fail to return.

### Completion and error interpretation

The two established normal stopping conditions are a nonzero attempt result and exhaustion of all eligible candidates. Both produce a return without an explicit result.

Zero and nonzero are control-flow categories. The source does not establish that nonzero means domain success, or that zero means rejection, an error, or absence of side effects.

There is no visible error conversion, separate exhaustion notification, rollback, or cleanup phase. If an attempt changes external state and returns zero, the next eligible attempt follows without restoration by this function.

### Widths and arithmetic

Classification is matched against fixed small constants. The input arguments are not modified by arithmetic.

Candidate selection produces only values from 0 through 8. Fallback progresses through candidates 1 through 8, with a terminating counter value of 9 possible internally. No candidate arithmetic approaches a 32-bit boundary or requires signed overflow or unsigned wraparound.

The terminating value 9 is not an attempt argument. Processing stops after candidate 8 has been considered, including when candidate 8 is skipped because it was the initial candidate.

## Side effects

### Direct effects and ordering

The established external interactions are ordered dependency calls:

1. One classification call.
2. The initial attempt.
3. Zero or more eligible fallback attempts.

The function contains no visible direct access to caller-provided storage or global memory, and no explicit allocation or release operation. Its candidate-selection and fallback bookkeeping are local.

The supplied body does not establish the reads and writes performed inside either dependency. Their possible effects occur within their respective calls; this function does not interpose an external-state snapshot, refresh, validation, or restoration between calls.

### Dependency-mediated state changes

Neither dependency may be assumed pure or state-preserving.

F-c453b9ccc22e may change state before the initial attempt. Each F-50f346be9118 call may read or change state before the following attempt. Returning zero does not imply that an attempt left memory, resources, or classification-related state unchanged.

The original argument values are reused throughout. If an argument identifies storage, later attempts may observe contents modified by earlier calls. The function does not reload replacement argument values from external storage.

Even if an attempt changes state relevant to classification, the original classification is not refreshed. Only that attempt’s zero-versus-nonzero result controls whether fallback continues.

### Ownership and lifetime

No ownership transfer is explicitly expressed by this function. This does not prove that the dependencies borrow the inputs, preserve referenced allocations, or refrain from retaining references.

There is no visible direct cloning, retention, allocation, release, or compensation for ownership changes. Required storage lifetimes and ownership compatibility across repeated attempts are documented_only unknowns.

In particular, the function’s reuse of the same argument values must not be interpreted as proof that all dependencies leave those values safe for later calls.

### Abnormal execution

No explicit handling is visible for a dependency that faults, terminates execution, does not return, or transfers control abnormally.

Later attempts are required only when preceding calls return normally and produce the results described by the ordering rules. The source establishes no recovery or cleanup guarantee for abnormal execution.

## Dependencies

| Opaque function ID | Role and established interaction |
|---|---|
| F-c453b9ccc22e | Called exactly once per invocation, without explicit arguments, before the initial attempt; never called again during fallback. Its 32-bit result determines the initial candidate. Its underlying state inputs and side effects are unknown. |
| F-50f346be9118 | Attempts the selected candidate first, then eligible fallback candidates until a nonzero result or exhaustion. Receives the original two 32-bit arguments and a 32-bit candidate. Its 32-bit result is used only for the zero-versus-nonzero continuation decision. |

No dependency implementation is supplied. Their validation, state access, resource behavior, ownership effects, and abnormal-control-flow behavior cannot be inferred from their use here.

The classification dependency’s single-call placement is significant: state changes made by attempt calls do not cause this function to obtain a new classification or change the initial-candidate exclusion.

## Acceptance examples

These examples are conditional consequences of the static control flow, not executed tests. Unless stated otherwise, all mentioned calls return normally. Every attempt must receive the function’s original first and second argument values.

1. **Initial nonzero result stops immediately.**  
   If classification returns 0, candidate 6 is attempted first. If that attempt returns nonzero, the complete dependency sequence is one classification call followed by one candidate-6 attempt. There is no fallback or second classification call.

2. **Fallback starts at 1.**  
   If classification returns 3, candidate 5 is selected. If attempts for 5 and 1 return zero and the attempt for 2 returns nonzero, the attempt order is exactly 5, 1, 2.

3. **The initial candidate is excluded from fallback.**  
   If classification returns 14 and every attempt returns zero, the attempt order is exactly 3, 1, 2, 4, 5, 6, 7, 8. Candidate 3 is not retried.

4. **Initial candidate 8 is not repeated at the end.**  
   If classification returns 35 and every attempt returns zero, the attempt order is exactly 8, 1, 2, 3, 4, 5, 6, 7. Considering and skipping candidate 8 during fallback does not produce another attempt.

5. **Explicit zero-selection values allow nine attempts.**  
   If classification returns 17 and every attempt returns zero, the order is exactly 0, 1, 2, 3, 4, 5, 6, 7, 8.

6. **Unlisted classifications are attempted through candidate 0.**  
   If classification returns 39, candidate 0 is attempted first. If that attempt returns zero and candidate 1 returns nonzero, the attempt order is exactly 0, 1.

7. **Any nonzero result stops processing.**  
   If classification returns 26, the initial candidate is 1. If its attempt returns the 32-bit all-ones value, processing stops. A positive-only interpretation is not imposed.

8. **Exhaustion has no distinct explicit output.**  
   If classification returns 2 and every attempt returns zero, the order is exactly 2, 1, 3, 4, 5, 6, 7, 8, followed by normal return without an explicit result.

9. **State changes do not trigger reclassification.**  
   If the sole classification call selects candidate 6, and its attempt changes classification-related state before returning zero, fallback still proceeds through eligible candidates 1, 2, 3, 4, 5, 7, 8. No further classification call occurs. A later nonzero attempt result still stops that sequence.

## Uncertainties

**Inferred intent:** The structure resembles a preferred-candidate attempt followed by a search over alternatives. The mapping may express preference or compatibility. Neither interpretation is necessary for the established ordering contract.

**documented_only unknowns:**

- Domain meanings of classifications, candidates, and attempt results.
- Semantic types, layouts, alignment, encoding, and nullability of the two inputs.
- Whether either input denotes storage, a resource, or an owned allocation.
- Dependency-specific validation, preconditions, and error reporting.
- Reads and writes performed inside the dependencies.
- Allocation, release, retention, ownership transfer, and required lifetimes.
- Whether zero-returning attempts preserve any relevant external state.
- Whether repeated attempts are valid after dependency-mediated state changes.
- Concurrency, synchronization, and reentrancy behavior.
- Machine-level ABI details and abnormal-execution behavior.

These unknowns must not be resolved by assuming harmless failed attempts, preserved globals, stable referenced contents, or dependency-managed cleanup.

## Evidence

The supplied static representation establishes:

- Two opaque 32-bit inputs and no explicit return value.
- Exactly one classification call per invocation, before the initial attempt, with no reclassification during fallback.
- The complete classification-to-candidate mapping shown above.
- An initial attempt after normal classification return.
- Immediate return on any nonzero attempt result.
- Ascending fallback over candidates 1 through 8, excluding the initial candidate.
- Forwarding of the original argument values to every attempt.
- No visible direct external-memory access, memory-management operation, rollback, or restoration.
- Bounded candidate arithmetic without a reachable 32-bit overflow.

No runtime tests have been performed. The acceptance examples describe conditional behavior derived from static evidence; they do not establish dependency internals, domain success, or external-state safety.
