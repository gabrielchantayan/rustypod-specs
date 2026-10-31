<!-- Trusted provenance: F-aab112b81c3b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aab112b81c3b

## Purpose

F-aab112b81c3b selects one of several 32-bit results through two ordered, short-circuiting comparison gates. If neither gate matches, it selects between a value read from an input-relative field and a fallback global value.

The established behavior is selection and forwarding. The supplied representation does not establish that the results are pointers, handles, status codes, or owned resources. It also does not establish what constitutes a match beyond a comparison dependency returning zero.

The fixed text “CntrlHistoryFn” suggests an association with control-history behavior. That is inferred intent, not an established interpretation of the interface or returned values.

## Interface

### Input

The function accepts one argument represented as an integer. It uses that argument as an address basis for:

- Passing the location at byte offset **+0x28**, or **+40**, to F-44362605ab79.
- Reading a 32-bit field at byte offset **+0x2c**, or **+44**, on the final selection path.

No direct read of the storage at offset +40 is visible in this function. That location is passed to a dependency, whose own access requirements are unknown.

The representation supports a 32-bit interpretation of the argument, return values, and explicitly loaded field. It does not independently establish the original source-language types, pointer model, or arithmetic rules. In particular, the integer-shaped interface must not be taken as evidence that arbitrary integers are valid inputs.

### Output

The function returns one 32-bit value, selected from:

1. A global result associated with the first comparison gate.
2. A different global result associated with the second comparison gate.
3. The first of two reads from the input-relative field at offset +44.
4. A fallback global result.

These are semantic labels for distinct storage locations or observations. They do not assert that the stored bit patterns are distinct.

No transformation, allocation, or explicit ownership transfer of the selected result is visible.

### Contract labels

The following labels describe data roles without exposing private source identifiers:

| Label | Meaning |
|---|---|
| Primary result | Global value returned when the first comparison returns zero. |
| Secondary comparison operand | Global value supplied as the second operand of the second comparison. |
| Secondary result | Global value returned when the second comparison returns zero. |
| Fallback result | Global value returned when the final field test observes zero. |
| First field observation | First 32-bit read at input offset +44. |
| Test field observation | Second 32-bit read at input offset +44, used for the zero test. |

The distinction between the two field observations is intentional and affects the result if the observed value changes between reads.

### Preconditions

For a path to complete normally:

- Every invoked dependency must accept the values supplied to it and return normally.
- Every global value actually read must be accessible.
- If the second gate is reached, forming the input-relative address at +40 must have valid meaning in the execution environment.
- If the final selection is reached, the location at +44 must support both shown 32-bit reads.

The function performs no visible null check, bounds check, alignment check, or input validation.

The required validity is path-dependent. An early first-gate match avoids all visible input-relative address use. That does not prove that the dependencies themselves are independent of the input’s surrounding state.

## Behavior

### First comparison gate

F-aab112b81c3b first calls F-3ad65987df9a with no explicit arguments.

Its 32-bit result becomes the first operand of an unresolved comparison dependency, identified here as **F-unresolved-01**. The second operand refers to the fixed text “CntrlHistoryFn”.

The caller tests whether the comparison result is exactly zero:

- **Zero:** read and return the primary result.
- **Any nonzero value:** continue to the second gate.

Positive and negative nonzero comparison results are treated identically. The caller performs no visible ordering comparison.

On the zero branch, F-44362605ab79 is not called, the second comparison is not performed, and neither visible read of the field at offset +44 occurs.

The primary result is read after the first comparison has completed. The supplied representation does not show it being captured before either dependency call. If a dependency changes that global, the later read can observe the changed value.

### Second comparison gate

This gate is reached only after the first comparison returns nonzero.

The function forms an input-relative address at offset +40 and passes it as the sole explicit argument to F-44362605ab79.

After F-44362605ab79 returns, its 32-bit result becomes the first operand of another call to F-unresolved-01. The second operand is the secondary comparison operand read from global storage.

The caller again distinguishes only zero from nonzero:

- **Zero:** read and return the secondary result.
- **Any nonzero value:** continue to the final selection.

The secondary comparison operand is obtained after F-44362605ab79 returns. Consequently, a change to that global made during F-44362605ab79 is not excluded from affecting the comparison.

Likewise, the secondary result is read after the second comparison returns. No guarantee is established that it retains a value held before that comparison.

On a zero second-comparison result, the function performs neither visible read of the field at offset +44.

### Final field-based selection

This stage is reached only when both comparisons return nonzero.

The supplied representation shows two separate 32-bit reads from the same input-relative location at offset +44:

1. The first field observation supplies the candidate return value.
2. The test field observation determines whether that candidate is replaced.

The resulting selection is:

| Test field observation | Returned value |
|---|---|
| Zero | Fallback result, read from global storage after the zero decision |
| Nonzero | First field observation |

The second observation is not itself the returned field value. In particular, a nonzero test observation does not guarantee a nonzero return.

If both observations see the same stable value, the behavior reduces to returning a nonzero field value or substituting the fallback when the field is zero. That simplification is valid only under an explicit stability assumption.

The fallback result is returned as read, including if it is itself zero. There is no further visible validation or substitution.

### Arithmetic and layout boundaries

The two relevant offsets are byte offsets. The function supplies the address at +40 to a dependency and reads the field beginning at +44 itself. The supplied material does not establish:

- The size or layout of the object beginning at +40.
- Whether that object includes, aliases, or can modify the field at +44.
- The required allocation size for the overall input object.
- The behavior of address formation near the address-space boundary.
- Whether a particular implementation applies signed-overflow rules, modular arithmetic, or some other address interpretation.

No overflow check is visible. A portable contract must not promise a specific wraparound result from the integer-shaped representation alone.

## Side effects

### Directly established effects

The function visibly:

- Calls dependencies in the order described above.
- Conditionally reads global storage.
- Conditionally reads the input-relative field twice.
- Returns a selected 32-bit value.

No explicit memory write, allocation, release, reference-count operation, or ownership-management operation appears in the supplied representation.

This is not a purity guarantee. Dependency calls may have effects that are not shown here.

### Dependency-mediated effects

No preservation guarantee is established for any dependency. In particular, dependencies may affect globals, input-related storage, external state, or resource ownership if their actual contracts permit it.

Later reads must therefore be understood as observations at their stated positions, not as implicit snapshots taken on entry:

- The primary result is read after the first comparison.
- The secondary comparison operand is read after F-44362605ab79.
- The secondary result is read after the second comparison.
- Both field observations occur after the second comparison.
- The fallback result is read only after the test field observation is zero.

### Ownership and lifetime

The function performs no visible direct release or retention of its input, dependency results, or selected result. Nevertheless:

- Input ownership is not established.
- Dependency ownership behavior is unknown.
- A pointer interpretation of any returned value would require a separate lifetime contract.
- Returning a global value does not by itself establish ownership of any object it might designate.

No visible cleanup or recovery path handles dependency failure. An exception, fault, nonlocal exit, or nonreturning dependency is outside the normal-return behavior established here.

## Dependencies

| Opaque function ID | Established use | Unknowns |
|---|---|---|
| F-3ad65987df9a | Called first, with no explicit arguments; its 32-bit result feeds the first comparison. | Meaning of the result, validity requirements, side effects, ownership, and failure behavior. |
| F-44362605ab79 | Called only after a nonzero first comparison; receives the input-relative address at +40; its result feeds the second comparison. | Whether it reads or writes the supplied storage, interprets a nested object, changes globals, allocates, releases, or can fail. |
| F-unresolved-01 | Called once or twice with two operands; zero selects an early result and every nonzero value continues. | Actual comparison semantics, operand types, memory accesses, side effects, ownership, and failure behavior. |

F-unresolved-01 is an editorial opaque identifier for the comparison dependency whose trusted mapping was not supplied. It is not a claimed published identity.

The use of fixed text in the first call suggests a text comparison, but the supplied material does not establish string encoding, case sensitivity, termination rules, normalization, null handling, or even that ordinary string equality is the complete operation.

## Acceptance examples

These are conditional review examples, not executed tests. Values are illustrative 32-bit values; storage is assumed valid, and dependencies are assumed to return normally unless stated otherwise.

### First-gate precedence

Given:

- The first comparison returns zero.
- The primary result is 17 when read.

The function returns 17. F-44362605ab79 and the second comparison are not invoked. The input-relative field is not directly read by this function.

The returned primary result may also be zero; zero does not trigger another selection stage.

### Second-gate precedence

Given:

- The first comparison returns nonzero.
- F-44362605ab79 receives the address at input offset +40.
- The second comparison returns zero.
- The secondary result is 29 when read.

The function returns 29 without reading the field at offset +44.

### Stable nonzero field

Given both comparisons return nonzero and both field observations are 41, the function returns 41. It does not read the fallback result on this path.

### Stable zero field

Given both comparisons return nonzero and both field observations are zero, the function returns the fallback result as subsequently read.

If that fallback result is zero, the function returns zero normally.

### Field changes from nonzero to zero

Given both comparisons return nonzero, the first field observation is 41, and the test field observation is zero, the function discards the candidate 41 and returns the fallback result.

### Field changes from zero to nonzero

Given both comparisons return nonzero, the first field observation is zero, and the test field observation is 41, the function returns zero: the nonzero test preserves the first observation.

These changing-field examples describe the two-read selection. They do not assert that unsynchronized concurrent mutation is legal in an unknown source-language memory model.

### Dependency changes a later-read global

Given F-44362605ab79 changes the secondary comparison operand before returning, the second comparison uses the value read after that call, not an entry-time snapshot.

Similarly, if the second comparison changes the secondary result and returns zero, the function reads the secondary result after that change.

### Nonzero comparison signs

Given a comparison returns either a positive nonzero value or a negative nonzero value, that gate continues. Only exact zero takes its early-return branch.

## Uncertainties

The following remain **documented_only**:

- Original source-language types and exact ABI.
- Whether the 32-bit results represent integers, addresses, handles, or other tokens.
- Complete contracts and side effects of all dependencies.
- The trusted identity and precise semantics of F-unresolved-01.
- Input object layout beyond the visible offsets and field width.
- Address overflow, alignment, and invalid-memory behavior.
- Global initialization, mutability, and aliasing.
- Ownership and lifetime of values passed to or returned from dependencies.
- Whether the two represented field reads correspond to distinct observable accesses in the underlying executable.
- Concurrency, atomicity, volatility, synchronization, and permitted external mutation.
- Any fault, exception, error-code, or recovery convention beyond the visible zero/nonzero comparison decisions.

No runtime tests have been performed.

## Evidence

**Established static evidence:** The supplied representation contains an initial no-argument dependency call, a comparison against fixed text, a conditional primary-global return, an input-relative dependency call at +40, a second comparison using a global operand, a conditional secondary-global return, and a final selection involving two represented 32-bit reads at +44 and a fallback global.

The visible control flow establishes gate precedence, exact-zero branch conditions, skipped operations on early returns, and the timing of later global reads relative to completed calls.

**Inferred intent:** The operation appears to choose a context-dependent result, with named special cases preceding an input-associated value and fallback. The text operand and comparison shape suggest a name-based match. Neither inference establishes the application meaning or dependency semantics.

**Not established:** Runtime accessibility, actual dependency behavior, ownership, concurrency guarantees, and externally observed results. The acceptance examples are derived conditions for review, not claims of successful execution.
