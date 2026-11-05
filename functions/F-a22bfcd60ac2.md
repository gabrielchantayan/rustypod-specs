<!-- Trusted provenance: F-a22bfcd60ac2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a22bfcd60ac2

## Purpose

F-a22bfcd60ac2 conditionally processes an input object through a classification, lookup, preparation, and status-check sequence. If those stages meet the observed conditions, it releases or finalizes an operation context and delegates production of the return value to F-9f44c2525d7c. Other observed normal-return paths report a diagnostic and return zero.

The established contract is control-flow and data-flow behavior, not a domain-level interpretation. Names such as “object,” “context,” “lookup,” and “finalizes” describe apparent roles only. The supplied evidence does not establish the underlying object types, the meaning of the accepted classifications, or the resource managed by the context.

## Interface

### Inputs

The function takes five arguments:

| Position | Observed representation | Established use |
|---|---|---|
| First | 32-bit value | Passed unchanged as the second argument to F-cd3cf16c4fbf. |
| Second | 32-bit value | Passed as the first argument to F-cd3cf16c4fbf and as the context argument to F-343d1ab3f962, F-eace0518d18e, and F-f3d8d5328cc4. |
| Third | 32-bit value | Passed unchanged as the first argument to F-9f44c2525d7c on the final delegation path. |
| Fourth | 32-bit address-bearing value | Identifies the primary object whose fields are read. It is also forwarded to F-cd3cf16c4fbf and F-9f44c2525d7c. |
| Fifth | 32-bit address-bearing value | Identifies an auxiliary object used to obtain two lookup inputs. It is later forwarded to F-9f44c2525d7c. |

The first three values are opaque. The function itself does not dereference them. Dependencies may impose additional requirements not established here.

### Required readable layouts

The function directly uses these layouts:

- In the primary object:
  - At byte offset 16: one 32-bit value supplied to F-2eb743f270d1.
  - At byte offset 20: one 32-bit address of a secondary object.
- In that secondary object:
  - At byte offset 8: one 32-bit value used by both F-3ff76ffda674 and F-cd3cf16c4fbf.
- In the auxiliary object:
  - At byte offset 4: one 32-bit address of a pair of consecutive 32-bit values.
- In that pair:
  - At byte offset 0: the first lookup input.
  - At byte offset 4: the second lookup input.

The minimum accessed extents are therefore 24 bytes for the primary object, 12 bytes for its secondary object, 8 bytes for the auxiliary object, and 8 bytes for the pair. These are access extents, not established allocation sizes or complete type definitions.

There are no visible null checks, bounds checks, alignment checks, or pointer-validity checks before the corresponding reads. Valid, readable storage is a precondition whenever a path reaches those reads. The auxiliary layout and secondary-object layout need not be accessed on an early classification rejection.

### Output

The return value is a 32-bit value.

- Observed rejection and nonpositive-status paths return exactly zero.
- The final delegation path returns the 32-bit result of F-9f44c2525d7c unchanged.

Zero is not an exclusive indicator that the final delegation was skipped: F-9f44c2525d7c may itself return zero. Its result range and interpretation are unknown.

## Behavior

### 1. Classify the primary object

The function reads the 32-bit value at primary-object offset 16 and calls F-2eb743f270d1 with it.

If the returned result equals 22, classification is accepted immediately. No second classification call is made.

Otherwise, the function reads primary-object offset 16 again and makes a second call to F-2eb743f270d1. Classification is accepted only if this second result equals 24.

Consequently, this is not equivalent to testing a single classification result for membership in the set containing 22 and 24:

- A first result of 24 still requires a second call.
- A second result of 22 does not qualify.
- A change to the field between calls can change the second call’s input.
- A dependency that returns different results on successive calls can affect acceptance even when the field is unchanged.

If classification is not accepted, execution proceeds directly to the common rejection behavior described below.

### 2. Obtain the lookup inputs

After classification is accepted, the function reads the address at primary-object offset 20 and then reads the 32-bit value at offset 8 of the referenced object. This value is retained for both the lookup and the subsequent preparation call.

The function also obtains the address stored at auxiliary-object offset 4 and reads the first two consecutive 32-bit values through it.

F-3ff76ffda674 is then called with three arguments, in this logical argument order:

1. The retained value from the secondary object.
2. The first value of the auxiliary pair.
3. The second value of the auxiliary pair.

The evidence establishes these argument sources, but not a total evaluation order for the individual reads that form this call’s arguments. In particular, the address at auxiliary-object offset 4 appears in both pair-access expressions; no single stable address snapshot should be assumed across those expressions.

If F-3ff76ffda674 returns zero, execution proceeds to common rejection. F-cd3cf16c4fbf is not called.

### 3. Prepare the operation

For a nonzero lookup result, the function calls F-cd3cf16c4fbf with five arguments:

1. The original second input.
2. The original first input.
3. The lookup result.
4. The retained value from the secondary object.
5. The original fourth input.

The lookup result is tested only for zero versus nonzero. The displayed signed representation does not introduce a positive-only requirement; a negative nonzero result follows this path.

If the preparation result is zero, execution proceeds to common rejection.

A nonzero preparation result is used only as a condition. It is not substituted for the primary object, returned to the caller, or passed to the final dependency. In particular, final delegation receives the original fourth input.

### 4. Issue the fixed context operation and check status

After a nonzero preparation result, the function calls F-343d1ab3f962 with:

1. The original second input.
2. The integer value 4.

No result from this call is inspected in the supplied evidence.

Next, it calls F-eace0518d18e with the original second input. This result is compared as a signed 32-bit integer against 1.

- A result of zero or any negative signed value follows the nonpositive-status path.
- A result from 1 through the largest positive signed 32-bit value follows the final delegation path.

For example, a returned bit pattern with only the highest bit set is negative under this comparison and therefore does not qualify as a positive status.

### 5. Handle a nonpositive status

For a signed status less than 1, the function performs these calls in order:

1. F-4fc89ec41a6d with five displayed arguments: 33, 107, 11, 0, and 0.
2. F-f3d8d5328cc4 with the original second input.

It then returns zero.

This path does not call F-9f44c2525d7c and does not fall through to the common rejection diagnostic.

### 6. Delegate after a positive status

For a positive signed status, the function first calls F-f3d8d5328cc4 with the original second input.

Only after that call returns does it invoke F-9f44c2525d7c with:

1. The original third input.
2. The original fourth input.
3. The original fifth input.
4. The retained lookup result.

Its return value becomes the function’s return value without a further check or transformation.

The ordering is material: F-f3d8d5328cc4 runs before final delegation, not afterward. Any effects it has on storage later inspected by F-9f44c2525d7c are therefore potentially observable.

### Common rejection

Classification rejection, a zero lookup result, and a zero preparation result all reach the same rejection action:

- Call F-4fc89ec41a6d with the two displayed arguments 33 and 107.
- Return zero.

The evidence does not show three additional arguments at this call site. It is not established that the two-argument and five-argument diagnostic calls are interchangeable forms, or that unspecified argument positions contain zero.

## Side effects

No direct memory writes, allocations, or releases are visible in the function itself. Its direct memory accesses are reads through the fourth and fifth inputs and through addresses obtained from them.

All other side effects are dependency-mediated. The evidence does not establish whether any dependency:

- Writes to the supplied objects.
- Changes shared or global state.
- Allocates, retains, transfers, or releases memory.
- Changes diagnostic state or emits externally visible output.
- Returns normally for every input.

In particular, F-f3d8d5328cc4 is not proven to be a deallocator. Its position suggests a context-completion role, but ownership and resource-release semantics remain unknown.

Subject to every called dependency returning normally, the visible call-count rules are:

- F-2eb743f270d1: once or twice.
- F-3ff76ffda674: at most once.
- F-cd3cf16c4fbf: at most once.
- F-343d1ab3f962 and F-eace0518d18e: once each after nonzero preparation.
- F-f3d8d5328cc4: exactly once after the status check, on either status outcome.
- F-9f44c2525d7c: once only for positive status.
- F-4fc89ec41a6d: once on each observed rejection path, and not directly called on the final delegation path.

There is no visible rollback on common rejection. In particular, a zero preparation result is followed by the common diagnostic, not by F-f3d8d5328cc4.

## Dependencies

| Opaque ID | Established role in this function |
|---|---|
| F-2eb743f270d1 | Classifies a value reread from primary-object offset 16; its call position determines whether 22 or 24 is accepted. |
| F-3ff76ffda674 | Receives the secondary-object value and two auxiliary values; zero rejects, nonzero is retained. |
| F-cd3cf16c4fbf | Receives five inputs in the specified order; zero rejects, nonzero enables context processing. |
| F-343d1ab3f962 | Receives the context and fixed value 4 before the status query. |
| F-eace0518d18e | Supplies the signed status controlling completion versus final delegation. |
| F-4fc89ec41a6d | Receives either the two-argument common diagnostic call or the five-argument nonpositive-status call. |
| F-f3d8d5328cc4 | Receives the context after either status outcome, before return or final delegation. |
| F-9f44c2525d7c | Receives the third, fourth, and fifth original inputs plus the lookup result; supplies the final return value. |

These roles do not establish broader dependency contracts.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They assume valid reached memory accesses and normal returns from dependencies.

1. **Immediate classification acceptance.** If the first classification result is 22, there is exactly one classification call. Lookup is attempted using the specified fields.

2. **First result 24 is insufficient.** If the first classification result is 24 and the second is 23, the function calls the two-argument diagnostic and returns zero. No lookup occurs.

3. **Second result 22 is insufficient.** If the first classification result is 0 and the second is 22, the function rejects. Acceptance at the second call requires 24.

4. **Classification may observe mutation.** If the first classification call changes primary-object offset 16 and returns a value other than 22, the second call receives the reread field value, not necessarily the first call’s input.

5. **Zero lookup short-circuits preparation.** With accepted classification and a lookup result of zero, the function issues the common diagnostic and returns zero without calling preparation or either context-status dependency.

6. **Negative lookup remains eligible.** With accepted classification and a lookup result of −1, preparation is called with that result. Subsequent behavior depends on preparation and status, not on the lookup result being positive.

7. **Preparation rejection has no visible completion call.** With a nonzero lookup and zero preparation result, the common diagnostic is called. F-f3d8d5328cc4 is not called by this path.

8. **Zero status rejects after completion handling.** With successful preceding conditions and status zero, the five-argument diagnostic precedes F-f3d8d5328cc4, then the function returns zero.

9. **Negative status behaves like zero status.** A signed status of −1 follows the same path as zero, not the final delegation path.

10. **Positive status delegates after completion handling.** With status 1, F-f3d8d5328cc4 precedes F-9f44c2525d7c. If the latter returns a particular 32-bit value, that exact value is returned.

11. **A nonzero preparation result is not a replacement object.** Even if preparation returns a different address-like value, final delegation receives the original fourth input.

## Uncertainties

The following are **documented_only**:

- The domain meanings of classification values 22 and 24, context operation 4, and diagnostic values 33, 107, and 11.
- Complete object types, alignment requirements, byte order, and valid address ranges.
- Exact dependency prototypes and the reason the diagnostic call sites display different argument counts.
- Whether any dependency preserves globals, input storage, object identity, or resource ownership.
- Whether invalid memory accesses produce a trap, another failure mode, or platform-specific behavior.
- Whether diagnostic calls can fail or prevent subsequent execution.
- The interpretation of the final return value beyond its unmodified propagation.
- A precise ordering or consistency guarantee for separate memory reads within the lookup argument expressions.

No unsupported preservation, synchronization, ownership, or recovery guarantee is part of this contract.

## Evidence

The supplied static function representation establishes the branch conditions, field offsets, 32-bit data accesses, dependency argument relationships, visible call ordering, and return propagation described above.

The staged-processing interpretation and context-completion terminology are inferred intent, not established domain semantics. Dependency internals and runtime behavior were not supplied.

No runtime tests have been performed. The acceptance examples express consequences of the static evidence under their stated conditions; they are not observed execution results.
