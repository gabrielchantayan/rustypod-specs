<!-- Trusted provenance: F-d43cfe3bcb88; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d43cfe3bcb88

## Purpose

F-d43cfe3bcb88 is a parameterless routine that repeatedly evaluates one externally declared integer object. It derives a fixed reference from the first evaluation, then compares candidates against that reference until a recorded difference is exactly two.

The routine returns no value. Its body contains no explicit writes to external storage, function calls, allocations, or releases. Repeated evaluations of the external object supply its only explicit input.

A polling or waiting purpose is **inferred**, not established. The supplied body does not establish that the object may legally change between evaluations, that each evaluation produces a separate physical load, or that the routine implements a synchronization protocol.

## Interface

### Explicit inputs and output

- **Parameters:** None.
- **Return value:** None.
- **Normal completion:** Returns to the caller after an inner-loop comparison records a difference of exactly two and the remaining operations of that iteration complete.
- **Error result:** No explicit error result, timeout, cancellation result, or recovery path appears.
- **Termination:** Not guaranteed for arbitrary sequences of evaluated values.

Completion does not bypass the final evaluation of the external object. Even after the terminating difference has been recorded, the routine evaluates that object, performs the refresh arithmetic, and resets the candidate before returning.

### Implicit input and layout

The routine depends on one externally declared integer object, called the **shared integer** in this contract. “Shared” identifies the object repeatedly referenced by the body; it does not establish a particular concurrency model.

The containing declaration is unavailable. Consequently, the following remain unknown:

- Declared type, signedness, width, alignment, and storage layout.
- Whether the object is ordinary, volatile, atomic, or associated with device access.
- Whether evaluation has externally visible side effects.
- Whether another execution context may legally modify it.
- Whether an evaluation can establish synchronization.

No address, buffer, length, record layout, or ownership-bearing value crosses the explicit interface.

### Arithmetic widths and conversions

The supplied representation declares the local arithmetic values as signed integers. Their exact width is not established. A 32-bit interpretation is **inferred**, not a requirement supported by the supplied declaration alone.

The missing shared-object declaration also leaves the arithmetic conversions unresolved. In particular, arithmetic involving an evaluated shared value may occur in a type different from the signed local type before assignment to a local.

The ordinary mathematical relationships below therefore apply only when the relevant operations and conversions preserve the stated values. The significant arithmetic operations are:

- Subtracting one from the first evaluated shared value and storing the fixed reference.
- Converting the second evaluated shared value into the initial candidate.
- Subtracting the fixed reference from a candidate.
- Adding one to each later evaluated shared value and storing the replacement candidate.

Representable operands do not by themselves guarantee that a subtraction result is representable.

Under a conditional 32-bit signed interpretation, the range is −2,147,483,648 through 2,147,483,647. Subtracting one from the minimum, adding one to the maximum, or forming an excessively large candidate/reference difference can exceed that range. Defined signed wraparound must not be assumed. Unsigned intermediate arithmetic and conversions into signed locals also must not be guessed from the missing declaration.

### Preconditions for the ordinary arithmetic contract

The behavior described in mathematical terms requires:

1. The shared integer is valid to evaluate whenever reached.
2. Relevant arithmetic and conversions have defined behavior and preserve the stated values.
3. Any changes between evaluations are permitted by the applicable storage and execution model.

These are conditions for the specified ordinary behavior, not evidence that the missing declaration satisfies them.

## Behavior

### Establishing the reference and initial candidate

The routine first evaluates the shared integer, subtracts one, and stores the result as the **fixed reference**.

It then evaluates the shared integer again and stores the resulting value as the **initial candidate**, subject to the applicable conversion.

These are two ordered source-level evaluations. The routine does not explicitly take one snapshot and reuse it for both purposes. If the evaluations may legally produce different values, the first determines the reference and the second independently determines the initial candidate.

The fixed reference does not change during the invocation.

### Inner comparison and unconditional refresh

Each inner-loop iteration performs these operations in order:

1. Records the candidate minus the fixed reference as the **difference**.
2. Records whether that candidate equals the fixed reference.
3. Evaluates the shared integer, adds one, and stores the result as the replacement candidate.
4. Uses the previously recorded equality result to decide whether to repeat the inner loop.

The equality decision concerns the candidate that existed before the refresh. The replacement candidate does not affect that already-recorded decision.

The refresh occurs on every reached inner iteration. It is not skipped when:

- The earlier candidate equals the reference.
- The earlier candidate differs from the reference.
- The earlier difference is already exactly two.

If the earlier candidate equals the reference, the inner loop repeats and compares the replacement candidate during the next iteration.

### Inner-loop exit, candidate reset, and outer decision

When the earlier candidate differs from the reference, the inner loop exits. The routine then resets the candidate to the fixed reference. This discards the candidate produced by the immediately preceding refresh.

The outer continuation condition contains two alternatives: the recorded difference equals one, or it differs from two. Together, these continue execution for every difference other than two.

Therefore:

- A recorded difference of exactly two permits normal return.
- Every other recorded difference starts another outer iteration.

The candidate reset occurs before this outer decision, including on the terminating path.

After an outer retry, the candidate equals the fixed reference. The next inner iteration consequently records a zero difference and an equal comparison. It still performs a fresh shared-integer evaluation and refresh, then necessarily repeats the inner loop.

This forced equal iteration is significant. An unsuccessful outer iteration does not carry its last refreshed candidate into the next comparison; that candidate has already been discarded.

### Values that can produce completion

There are two routes to a terminating difference under the ordinary arithmetic conditions.

**Initial-candidate route.** The second initialization evaluation supplies the candidate directly. A terminating difference is recorded on the first comparison if that value is two greater than the reference. Since the reference is the first evaluated value minus one, this means the second evaluated value is one greater than the first.

A third evaluation and its addition still occur before return.

**Refreshed-candidate route.** Later candidates come from an evaluated shared value plus one. Such a candidate produces a difference of two when the evaluated value is one greater than the reference—equivalently, equal to the first initialization value.

This route applies only when the refreshed candidate survives to another comparison. A refresh performed during an unequal comparison is discarded at inner-loop exit, even if its numerical result would otherwise be a successful candidate.

### Progress and nontermination

With a constant shared value and defined, value-preserving arithmetic throughout, the routine returns after five source-level evaluations of the shared integer.

Other legal value sequences can prevent return:

- The inner loop can continue indefinitely if each refreshed candidate equals the reference.
- Outer iterations can repeat indefinitely if each unequal comparison records a difference other than two.

No explicit delay, yielding operation, iteration limit, or timeout appears.

These are source-level control-flow properties. They do not independently establish behavior under unsynchronized mutation, particular optimization choices, or a missing atomic-access model.

## Side effects

### Explicit state changes

The routine evaluates the shared integer twice during initialization and once per inner-loop iteration.

It updates local state consisting of the fixed reference, candidate, recorded difference, and equality result. Only the reference remains fixed after initialization.

There are no explicit writes to the shared integer or other external objects. There are no explicit function calls or resource-management operations.

These statements concern operations present in the supplied body. They do not exclude effects inherent in evaluating an object whose declaration and access semantics are unavailable.

### Read ordering and observability

The evaluation order described above is source-level order. Whether each evaluation constitutes a distinct observable read depends on the declaration and execution model.

In particular:

- Ordinary storage must not automatically be treated as requiring a distinct physical load for every source evaluation.
- Volatile, atomic, or device-associated evaluation semantics cannot be inferred or ruled out from this body alone.
- The final refresh remains part of the source-level behavior even though its resulting candidate is discarded.

No explicit locking or barrier primitive appears in the supplied body. Whether the shared-object evaluations are atomic or establish synchronization depends on the missing declaration and execution model.

Accordingly, neither a synchronization guarantee nor a categorical absence of synchronization is established.

### Ownership and lifetime

No pointer, handle, or owned object is passed or returned. The body explicitly allocates and releases nothing and performs no ownership transfer.

The routine relies on the shared integer remaining valid for every reached evaluation. Its lifetime and any effects associated with its access are external prerequisites, not managed by this routine.

## Dependencies

No function-call dependency appears in the supplied body. No additional opaque function ID is invoked.

The routine depends on:

- The existence and readability of the shared integer.
- Its declared type and access semantics.
- Local integer widths and arithmetic conversions.
- The execution model governing repeated evaluations and external changes.

There is no callee whose global-state preservation or memory-management behavior can be assumed. Any read-side effects arise from the unresolved semantics of the shared object, not from an explicit dependency call shown in the body.

## Acceptance examples

These examples are **conditional static acceptance examples**, not observed executions. Each assumes valid evaluations and defined, value-preserving arithmetic unless a boundary case explicitly states otherwise. Evaluation counts refer to source-level evaluations, not guaranteed hardware loads.

### Constant value: five evaluations

Suppose every evaluation yields 10.

- The first evaluation establishes reference 9.
- The second supplies initial candidate 10.
- The first comparison records difference 1 and inequality. The third evaluation forms candidate 11, but that candidate is discarded.
- The candidate resets to 9 and the outer loop retries.
- The forced equal iteration uses the fourth evaluation to form candidate 11.
- The next iteration records difference 2 and performs the fifth evaluation before leaving the inner loop.
- The candidate resets to 9, and the routine returns.

Expected result: normal return without a value after five evaluations, with no explicit external writes.

### Immediate success still requires a third evaluation

Suppose the first three evaluations yield 10, 11, and 40.

The reference is 9 and the initial candidate is 11. The first comparison records difference 2, but the routine still evaluates 40 and forms 41. It then discards that replacement candidate, resets the candidate to 9, and returns.

Expected result: three evaluations. Recording the terminating difference does not skip the refresh.

### Equality is determined before refreshing

Suppose successive evaluations yield 10, 9, 8, 10, and 20.

The reference and initial candidate are both 9.

- The first equal iteration refreshes from 8, producing candidate 9.
- The second equal iteration refreshes from 10, producing candidate 11.
- The next iteration records difference 2 and refreshes from 20 before returning.

Expected result: normal return after five evaluations. Neither refresh retroactively changes the equality decision made earlier in its iteration.

### A numerically successful refresh can be discarded

Suppose successive evaluations yield 10, 10, 10, 8, 10, and 50.

The reference is 9 and the initial candidate is 10.

- The first comparison records difference 1. The third evaluation forms candidate 11, which would yield difference 2 if compared, but it is discarded.
- After the reset, the forced equal iteration refreshes from 8, producing candidate 9.
- Another equal iteration refreshes from 10, producing candidate 11.
- The next comparison records difference 2 and performs the final refresh from 50.

Expected result: normal return after six evaluations. The third evaluation does not itself establish successful completion.

### Persistent equality prevents return

Suppose the first evaluation yields 10, the second yields 9, and every later evaluation yields 8.

The reference is 9. The initial candidate equals it, and every refresh produces 9 again.

Expected result under the stipulated evaluation model: the inner loop continues indefinitely.

### Repeated outer retries prevent return

Suppose the first two evaluations yield 10 and every later evaluation yields 9.

The reference is 9. The initial comparison records difference 1. Thereafter, each outer retry starts with candidate 9, refreshes to 10 during an equal iteration, and then records difference 1 during the next iteration.

Expected result under the stipulated evaluation model: repeated outer retries without return.

### Final-refresh overflow is not excused by success

Conditionally assume that the refresh addition uses a 32-bit signed type. Suppose the initial evaluations yield 10 and 11, and the final refresh evaluates 2,147,483,647.

The first comparison records difference 2, but the subsequent addition of one exceeds the assumed signed range.

Expected result: this contract does not specify portable normal-return behavior for that case. The fact that the replacement candidate would be discarded does not establish defined wraparound or justify omitting the arithmetic.

## Uncertainties

The following are **documented_only**:

- The shared integer’s declaration, width, signedness, alignment, and layout.
- Exact local integer widths.
- Arithmetic conversions involving shared values.
- Behavior at unresolved overflow or conversion boundaries.
- Volatility, atomicity, device semantics, and read-side effects.
- Whether evaluations establish synchronization.
- Legal sources of changes between evaluations.
- Physical load count and optimization behavior.
- Fault behavior if an evaluation is invalid.
- Scheduling, timing, and termination under concurrent activity.
- The intended application-level purpose.

A polling interpretation and a 32-bit local-integer interpretation remain inferences. Neither is an unconditional acceptance requirement.

## Evidence

**Established static evidence:** The supplied body has no parameters and no returned value. It performs two ordered initialization evaluations, derives an unchanged reference by subtracting one from the first value, and obtains the initial candidate from the second. Each inner iteration records a difference and equality before evaluating the shared integer again and adding one. After inner-loop exit, it resets the candidate before evaluating the outer continuation condition. No explicit external write, function call, lock, barrier primitive, allocation, or release appears.

**Derived static consequences:** The outer condition continues for every recorded difference except two. Every refresh on an unequal iteration is discarded. Every outer retry begins with a forced equal iteration. Under defined, value-preserving arithmetic, constant input returns after five source-level evaluations.

**Inferred intent:** The repeated evaluations may serve a polling or waiting purpose, but the body alone does not establish that interpretation.

**Evidence limits:** The shared-object declaration, target arithmetic model, callers, and concurrency context are unavailable. Atomicity and synchronization cannot be settled from the supplied body. No runtime tests have been performed.
