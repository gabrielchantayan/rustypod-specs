<!-- Trusted provenance: F-aac5ed5b7a73; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aac5ed5b7a73

## Purpose

F-aac5ed5b7a73 is a three-input dispatcher with a 32-bit recognition result. It consults an external comparison dependency to distinguish three event categories. For the first category, it conditionally invokes F-43d4f3dbddf4 using the first and third inputs. The other two categories are recognized without that invocation.

The established behavior is defined by the comparison dependency’s zero or nonzero results, not by an independently established string-matching rule. Interpreting the categories as selection, confirmation, and timed-workout selection is inferred from the supplied token labels.

A return value of one indicates that execution followed a recognized branch. It does not establish that an action succeeded or that an action occurred. A return value of zero occurs only after all three comparisons have returned nonzero.

## Interface

### Inputs

| Input | Established representation | Use |
|---|---|---|
| First input | One 32-bit value with no established internal type | Forwarded unchanged as the first argument to F-43d4f3dbddf4 when dispatch is required. |
| Second input | One 32-bit value with no established internal type | Passed as the first argument of every comparison that is reached. |
| Third input | A value recovered as a 32-bit integer | Compared with a shared reference value and, when unequal, forwarded as the second argument to F-43d4f3dbddf4. |

The first and second inputs might represent addresses, handles, or other machine-sized values. Their semantic types are not established. In particular, the body does not establish that the second input is a valid, terminated character string merely because it is supplied to a comparison-like dependency.

The third input participates only in an equality decision and argument forwarding. No addition, subtraction, multiplication, indexing, or range conversion is visible. Consequently, there is no visible arithmetic overflow case. Under the recovered 32-bit interpretation, equality depends on the complete value; treating its bit pattern as signed or unsigned does not change the equality outcome.

### Output

Normal completion returns a 32-bit value:

- **One:** the first comparison returned zero, or a subsequently reached comparison returned zero.
- **Zero:** all three comparisons returned nonzero.

No other normal return value is visible. A comparison result’s magnitude is irrelevant: any nonzero result takes the same branch at that comparison.

### Other inputs to behavior

The function also depends on:

- Three comparison tokens supplied to the comparison dependency.
- A shared reference value read only on the first recognized branch.
- The behavior and side effects of both external dependencies.

The token labels suggest these event texts, in comparison order:

1. “HandlePowerSongSelected”
2. “HandlePowerSongChosen”
3. “HandleSelectTimedWorkout”

These texts are inferred from the supplied labels. The underlying token bytes, encoding, termination, storage layout, and exact spelling are not independently exposed.

### Preconditions

No explicit input validation is visible. Safe completion therefore requires that every dependency reached by the selected path can accept its arguments and return normally. The validity requirements for address-like inputs, token storage, and the shared reference storage are documented_only.

There is no established null-input policy, ownership-transfer policy, or caller-visible diagnostic mechanism.

## Behavior

### Ordered recognition

The comparison dependency is first called with the second input and the first token. This call occurs on every ordinary invocation before the function decides whether to read the shared reference value or perform another comparison.

If the first comparison returns zero:

- The shared reference value is read.
- The third input is compared with that value.
- If they differ, F-43d4f3dbddf4 is called with the first input and the third input, in that order.
- If they are equal, that call is omitted.
- The function returns one after any required call completes normally.

Neither the second nor the third token is compared on this branch.

If the first comparison returns nonzero, the comparison dependency is called with the second input and the second token.

If the second comparison returns zero, the function returns one. It does not compare the third token, read the shared reference value through the visible body, or invoke F-43d4f3dbddf4.

If the second comparison returns nonzero, the comparison dependency is called with the second input and the third token.

If the third comparison returns zero, the function returns one. If it returns nonzero, the function returns zero. Neither outcome directly invokes F-43d4f3dbddf4 or directly reads the shared reference value.

### Recognition precedence

The first zero result ends recognition. Later comparisons are not performed.

This ordering is observable even if the comparison dependency is stateful or recognizes multiple tokens. For example, a zero result on the first comparison suppresses the second and third calls regardless of what those calls might otherwise have returned.

Recognition is not necessarily equivalent to membership in a fixed set of strings. That equivalence would require an independently established comparison contract and token contents.

### Timing of the shared reference read

The shared reference value is read after the first comparison has returned zero. It is not visibly captured before that call.

Accordingly, the decision must use the value available at that later read. If the comparison dependency changes the shared reference storage, the changed value can affect whether F-43d4f3dbddf4 is invoked.

The body does not reread the reference after the action call. It also does not visibly assign the third input to the reference storage. Any reference update performed inside a dependency is outside the established behavior of this function.

### Action result and completion

No result from F-43d4f3dbddf4 is inspected. Its normal return is followed by the dispatcher returning one.

That is a control-flow guarantee, not a success guarantee for the action. The supplied evidence does not establish whether the action can fail, report an error elsewhere, alter shared state, or terminate execution abnormally. If a dependency does not return normally, the visible normal-return rules do not establish what the caller observes.

## Side effects

### Directly established effects

The function’s direct externally relevant operations are:

- One to three ordered calls to the comparison dependency.
- A conditional read of a shared reference value.
- At most one call to F-43d4f3dbddf4.
- A normal return of zero or one.

No direct write to caller-provided storage, the shared reference storage, or another shared object is visible. No allocation, release, copying of a caller-owned buffer, or ownership transfer is explicit in the body.

### Dependency-mediated effects

The absence of direct writes does not make the function pure. Either dependency may read or modify memory, perform I/O, allocate or release resources, or otherwise affect the process. Such behavior is not established here and must not be assumed absent.

Likewise, a return of zero does not imply that nothing happened: three dependency calls precede that return. A return of one without an action call also follows at least one comparison call.

No synchronization or atomic access guarantee is established. The reference read’s ordering relative to the first comparison is visible, but cross-thread visibility and concurrent modification semantics are documented_only.

### Ownership

The function visibly forwards values rather than creating or releasing owned objects. That observation does not establish that either dependency borrows its arguments, preserves pointed-to storage, or refrains from releasing resources. Dependency ownership contracts remain unknown.

## Dependencies

### F-43d4f3dbddf4

This supplied opaque dependency ID identifies the conditional action.

Its established invocation contract within this function is:

- Called only after the first comparison returns zero.
- Called only when the third input differs from the subsequently read shared reference value.
- Receives the first input followed by the third input.
- Called no more than once per invocation.
- Has no result consumed by this function.

Its parameter semantics, memory effects, ownership rules, error reporting, and relationship to the shared reference value are documented_only.

### F-U1

F-U1 is a document-local opaque ID for the comparison dependency; no published opaque ID was supplied for it.

Its established invocation contract within this function is:

- Receives the second input followed by one comparison token.
- Is called with the first token unconditionally on the normal entry path.
- Is called with the second token only after a nonzero first result.
- Is called with the third token only after two nonzero results.
- Supplies a recovered integer result tested only for zero versus nonzero.

Whether F-U1 performs string comparison, token identity comparison, case folding, normalization, or another operation is documented_only. Its purity and preservation of shared state are not established.

## Acceptance examples

These are conditional static expectations, not executed tests. Each example assumes that all reached dependencies return normally.

| Conditions | Required observable behavior | Return |
|---|---|---|
| First comparison returns zero; third input equals the reference value read afterward | One comparison; reference read; no action call; no later comparisons | One |
| First comparison returns zero; third input differs from the reference value read afterward | One comparison; reference read; one call to F-43d4f3dbddf4 with the first and third inputs | One |
| First comparison returns nonzero; second returns zero | Two comparisons in order; no direct reference read or action call | One |
| First two comparisons return nonzero; third returns zero | Three comparisons in order; no direct reference read or action call | One |
| All comparisons return nonzero | Three comparisons in order; no direct reference read or action call | Zero |

Additional boundary expectations:

- **Reference mutation during comparison:** If F-U1 changes the reference value and then returns zero on the first comparison, dispatch depends on the value read after that call, not an entry-time value.
- **Nonzero comparison values:** Negative and positive nonzero comparison results have the same branching effect.
- **Identical third-input bit patterns:** Equality with the read reference suppresses the action, including when the bit pattern represents a negative recovered integer.
- **Action changes the reference:** No second action call or post-action reference check follows within this invocation.
- **Second-category recognition:** A zero second result prevents a third comparison even if the third token would also be recognized.
- **Invalid dependency arguments:** No guaranteed return or recovery behavior can be specified without the missing dependency contracts.

## Uncertainties

The following remain documented_only:

- Exact semantic types and validity constraints for the first two inputs.
- Exact token contents, encoding, and layout.
- F-U1’s matching semantics and side effects.
- The shared reference value’s initialization, mutability, semantic meaning, and synchronization.
- F-43d4f3dbddf4’s action semantics, errors, memory ownership, and state changes.
- Behavior on faults, exceptions, nonlocal exits, or nonreturning dependency calls.
- ABI details beyond the recovered 32-bit value representations.

The apparent intent is to recognize three related event categories and conditionally act when a selected value differs from a current reference. Calling that reference a “current selection,” or claiming the action updates it, would exceed the evidence.

## Evidence

Established static evidence consists of the supplied function body and the supplied opaque mapping for F-43d4f3dbddf4. They establish the comparison order, zero/nonzero decisions, conditional reference read, action arguments, and normal return values.

The recovered declarations support 32-bit inputs and output and an integer comparison result. They do not establish higher-level object layouts or pointer validity.

Event meanings and likely token text are inferred from labels, not independently verified token storage. No dependency implementation was supplied. No runtime tests have been performed.
