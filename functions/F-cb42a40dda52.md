<!-- Trusted provenance: F-cb42a40dda52; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb42a40dda52

## Purpose

F-cb42a40dda52 accepts a 32-bit value and runs a state machine controlled by externally stored words. Depending on the selected states, it retains the input, passes the retained value to F-5cbb2ab262ad, reads unsigned 16-bit samples from computed addresses, updates arithmetic state, and returns a saved 32-bit result.

The static evidence establishes these operations, not a conventional high-level algorithm. External-word values are unavailable, so no particular state sequence, memory extent, number of calls, or termination guarantee is established.

Structured traversal and sample packing are plausible **inferred intent**, based on the address components, phase bits, and shifted accumulation. They are not required interpretations of the function.

## Interface

**Input:** one signed 32-bit value. It is initially opaque. The function can retain it unchanged, pass that retained value to the dependency, or use it as an address base. Neither a scalar-only nor a pointer-only interpretation is established.

**Output:** a signed 32-bit saved result, returned only when dispatch selects state 5. This field starts at zero and is assigned by the default/outer-processing action. Its meaning as a count, status, pointer, checksum, or packed value is unknown.

**Conditional preconditions:**

- Each execution of state 8 requires its computed address to support a readable unsigned two-byte load.
- Each execution of state 1 requires the retained argument to satisfy F-5cbb2ab262ad’s unknown requirements.
- Arithmetic, address formation, and shifts require the execution environment’s applicable semantics. The representation does not establish defined behavior for every possible external-word configuration.

There is no explicit allocation, release, ownership transfer, or write to caller-provided memory in this function. Those observations do not constrain the dependency’s ownership or memory behavior.

### State and external words

The following are descriptive contract terms, not source identifiers.

Local state includes a dispatch word; retained input; outer and inner indices and aggregates; saved predecessor, result, and seed; phase; row and column address components; and saved scaled inner values. All are initially zero except the dispatch word, which is initialized by reading **Base**.

External words have these roles:

| Contract name | Role |
|---|---|
| Base | Subtracted from the dispatch word to obtain the selector |
| EntryTarget | State 0’s next dispatch word |
| ResetTarget | Next dispatch word in states 1 and 6 |
| InnerTarget | Next dispatch word in states 2 and 8 |
| RelayTarget | State 4’s next dispatch word |
| InnerChoice | Separately observed sources for state 3’s two candidates |
| OuterChoice | Separately observed sources for the default action’s two candidates |
| Width | Half-width, phase, address, and predicate arithmetic |
| Scale | Scaling, correction, and accumulation arithmetic |
| ResultScale | Result, shift, index, and aggregate arithmetic |
| Factor | Outer-predicate and sample-transformation arithmetic |
| Step | Saved-predecessor adjustment and seed factor |
| InnerBias, InnerMask | Inner predicate inputs |
| OuterBias, OuterMask, FinalMask | Outer predicate inputs |
| LoadScale | Multiplier before sample truncation |
| ComplementScale | Multiplier used to construct a complemented sample word |
| SampleScale, Correction | Sample-accumulation inputs |

These names identify storage locations, not configuration snapshots. Each represented reference is a separate observation unless an intermediate explicitly retains its value.

## Behavior

### Arithmetic and dispatch

The representation uses signed and unsigned 32-bit intermediate storage and an unsigned 16-bit load. Hexadecimal masks below describe 32-bit patterns. Unsigned arithmetic has 32-bit modular behavior where the represented operand types establish it; signed operations must not all be silently reinterpreted as wrapping arithmetic.

Before each state action, dispatch reads Base and subtracts it from the current dispatch word. Selectors 0 through 8 are represented. Selector 7 and all unmatched selectors execute the same default action.

If Base is unchanged between initialization and the first dispatch evaluation, the first state is 0. Otherwise, even the first state is not determined.

No iteration limit or independent termination check is visible.

### State 0: retain the input

State 0 first reads EntryTarget into the next dispatch word, then retains the original argument.

It performs no sample load or dependency call. If later states are reached without state 0 having executed, the retained input remains its initial zero value.

### State 1: dependency call and outer reset

State 1 calls F-5cbb2ab262ad with the retained input. Its return value is unused.

After normal return, the outer index is cleared, then the outer aggregate is cleared, then ResetTarget is read into the dispatch word. Saved results, seeds, inner state, phase, and address components are not cleared by this action.

The dependency may affect external words. In particular, the ResetTarget observation occurs after the call, not before it. No preservation guarantee is available.

### State 2: initialize inner processing

State 2 clears the inner index, reads InnerTarget into the dispatch word, and then copies the saved seed into the inner aggregate.

It does not reset outer state, the saved result, the saved predecessor, phase, or address components.

### State 3: derive inner state and choose a transition

This action has the following statement-level order:

1. Read Width and derive a signed half-width by right-shifting it by one.
2. Read Width again and form an inner-predicate intermediate: the inner index multiplied by the difference between the retained half-width and this new Width observation.
3. Assign the unsigned phase from the half-width multiplied by the sum of the inner and outer indices.
4. Evaluate and retain the inner predicate, including its further external-word reads.
5. Save the half-width-scaled inner aggregate.
6. Save the row component as half-width times inner index.
7. Save the column component as half-width times outer index.
8. Save the half-width-scaled saved predecessor.
9. Read InnerChoice and form the adjusted candidate by subtracting two.
10. Read InnerChoice again to obtain the unadjusted candidate.
11. Combine the retained candidates according to the predicate.

Thus, the predicate’s external observations precede assignment of the scaled aggregate and address components. The two InnerChoice observations occur afterward, in the stated order. They need not have equal values.

For compact specification of the predicate, define the width-sensitive expression

\[
\Phi(t,z,m)=
\left[-4-\left((t-1)\mathbin{\mathrm{OR}}\mathtt{FFFFFFFC}\right)\right]
+
\left[
\left((t+1)-\left(z\mathbin{\mathrm{OR}}(t+2)\right)\right)
\mathbin{\mathrm{XOR}}
\mathop{\mathrm{NOT}}\left((t+2)\mathbin{\mathrm{OR}}m\right)
\right].
\]

In state 3:

- \(t\) is the retained intermediate from step 2.
- \(z\) is \(t-\text{InnerBias}\), plus the OR of a fresh Width-times-inner-index value plus one with InnerMask.
- \(m\) is a separate InnerMask observation, shifted right by one and ANDed with `FFFFFFFC`.

These expressions specify bitwise dependencies, not unbounded mathematical arithmetic. The two InnerMask references are not a single retained read. Their relative evaluation order within the predicate expression is not established.

### States 4 and 5: relay and return

State 4 reads RelayTarget into the dispatch word without another represented state update.

State 5 immediately returns the saved result. It performs neither a dependency call nor a sample load.

### State 6: promote scaled inner state

State 6:

1. Reads Scale and assigns the outer index from Scale times the saved scaled predecessor.
2. Reads Scale again and assigns the outer aggregate from Scale times the saved scaled inner aggregate.
3. Reads ResetTarget into the dispatch word.

The Scale observations are separate and ordered. If state 3 has not populated the saved scaled fields, state 6 uses their initial zeros or values retained from an earlier visit.

### Default action: outer processing

This action applies both to selector 7 and to unmatched selectors.

Its statement-level order is:

1. Read Factor and retain Factor times the outer index as the initial outer-predicate intermediate.
2. Read Step and save outer index minus Step as the predecessor.
3. Read ResultScale and save ResultScale times the outer aggregate as the result.
4. Save a seed equal to the saved result times the difference between Step shifted right by 29 and Step. The two Step references in this expression are separate observations.
5. Evaluate the rest of the outer predicate.
6. Read OuterChoice and form the adjusted candidate by adding two.
7. Read OuterChoice again to obtain the unadjusted candidate.
8. Combine the retained candidates according to the predicate.

The initial Factor observation therefore precedes the predecessor, result, and seed updates. The remainder of the predicate is evaluated after those updates.

The predicate is \(\Phi(t,z,m)\), where:

- \(t\) is the Factor-times-outer-index intermediate retained in step 1.
- \(z\) is \(t+\text{OuterBias}\), plus the OR of Width times outer index plus one with OuterMask.
- \(m\) is FinalMask.

The saved result is a snapshot. Later outer-aggregate changes do not alter it unless this action executes again. Unmatched selectors do not produce an invalid-state error.

### Predicate-based candidate combination

Both predicate-producing actions store the predicate in signed 32-bit storage and form a selector from its signed right shift by 31, plus one. Candidate combination uses that selector and its XOR with one.

Under arithmetic signed right shift:

- A negative predicate selects the adjusted candidate.
- A nonnegative predicate selects the unadjusted candidate.

For state 3, these are respectively the **first InnerChoice observation minus two** and the **second InnerChoice observation**. For default processing, they are the **first OuterChoice observation plus two** and the **second OuterChoice observation**.

Both target reads occur regardless of which candidate is ultimately selected. Describing them as arithmetic adjustments of one shared target value is valid only with an additional stability assumption.

The sign interpretation remains conditional on signed-shift and conversion semantics.

### State 8: sample read and inner accumulation

State 8 uses previously saved address components. Its sample address is

\[
\text{retained input}
+6\,\text{row component}
+2
+2\,\text{column component}.
\]

It performs one unsigned 16-bit load, with no visible bounds check or pointer validation.

The action proceeds in this order:

1. Evaluate the load-and-transform expression: multiply the loaded sample by LoadScale and retain the low 16 bits as unsigned word \(u\).
2. Read ComplementScale and form unsigned word
   \[
   v=\mathop{\mathrm{NOT}}(u\cdot\text{ComplementScale})
   \mathbin{\mathrm{OR}}\mathtt{0000FFFF}.
   \]
3. Form unsigned correction \(w\) from Scale XOR a separate Scale observation interpreted as signed and shifted right by 28.
4. Form an unsigned shift word from ResultScale, Scale, the low two phase bits, and eight.
5. Update the inner index from Scale times ResultScale times the difference between its old value and \(w\).
6. **Read InnerTarget and save it as the next dispatch word.**
7. Calculate and assign the inner aggregate, performing that expression’s additional external-word reads.
8. Resume dispatch, which reads Base again.

The InnerTarget read is after the index update but before the aggregate calculation. It is not deferred until all state-8 arithmetic finishes.

The effective shift count is the low eight bits of the shift word:

\[
n=
\left(
\text{ResultScale}\cdot\text{Scale}\cdot
(\text{phase}\mathbin{\mathrm{AND}}3)\cdot8
\right)\mathbin{\mathrm{AND}}255.
\]

The aggregate calculation uses fresh external observations to form a sample-dependent word:

\[
q=
\text{Factor}\left(
\text{Scale}\cdot\mathop{\mathrm{NOT}}v
+u\cdot\text{SampleScale}
-\left(\text{Correction}\mathbin{\mathrm{XOR}}
(\text{signed Correction}\gg28)\right)
+\text{Correction}\cdot v
\right)-1.
\]

Each Correction occurrence is a separate observation. The Scale and ResultScale occurrences in this aggregate calculation are also distinct from their earlier state-8 observations.

The new inner aggregate is represented by

\[
\text{Scale}\cdot\text{ResultScale}\cdot
\left(
\text{old inner aggregate}
+w\cdot(q\ll n)
+w\cdot(1\ll n)
\right).
\]

The two shifted contributions must remain distinct. The sample-dependent expression includes unsigned operands, while the unit-shift expression starts from signed one. Their validity and overflow behavior are not interchangeable. The count is masked to eight bits, not five; counts at least 32 are possible. Even a smaller count can raise signed-shift representability concerns for the unit contribution.

The low 16 bits of \(v\) are always ones, so the low 16 bits of its 32-bit complement are zero. This fact does not eliminate \(v\)’s other contributions to the aggregate.

## Side effects

Directly represented effects are external-word reads, dependency calls, two-byte sample reads, local state changes, and eventual return. No explicit external-memory write, allocation, or release appears.

**Statement-level order is established as described above. Expression-level evaluation order is not generally established.** In particular:

- The sample load and LoadScale observation belong to one expression; their internal evaluation order is unspecified here.
- Repeated external references within a predicate, correction, seed, or aggregate expression remain separate observations without an established internal order.
- InnerChoice and OuterChoice candidate reads are separate statements with an established first-then-second order.
- State 8’s InnerTarget observation precedes all external-word observations belonging to the subsequent aggregate expression.

These facts do not establish atomicity, synchronization, or a concurrency protocol. Changes by the dependency or other environmental activity must not be excluded without evidence.

There is no visible error-return path for invalid addresses, arithmetic failures, malformed control values, or nontermination. Faults and dependency failures are not visibly caught.

## Dependencies

**F-5cbb2ab262ad**

- Receives the retained 32-bit input when state 1 executes.
- Has an unused return value.
- Has unknown memory-access, ownership, global-state, failure, and normal-return behavior.
- Is not established as a validator, initializer, allocator, or destructor.

External words and readable sample storage are additional environmental dependencies. Their values and accessibility are necessary to derive a concrete execution.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **Stable initial dispatch.** If Base is unchanged between initialization and the first dispatch, state 0 executes first. EntryTarget is observed before the input is retained.

2. **Immediate return.** If stable control values route state 0 directly to state 5, the result is zero. Neither the dependency nor the sample load executes.

3. **Dependency mutation.** If state 1 executes and the dependency changes ResetTarget before returning normally, the subsequent ResetTarget read observes the post-call environment. The contract does not preserve a pre-call target.

4. **Distinct inner candidates.** Suppose state 3’s two InnerChoice reads yield 12 and 30. Subject to defined arithmetic and arithmetic signed right shift, a negative predicate selects dispatch word 10; a nonnegative predicate selects 30. These are dispatch words, not necessarily state selectors.

5. **Distinct outer candidates.** If the two OuterChoice reads yield 20 and 40, the corresponding adjusted and unadjusted dispatch candidates are 22 and 40. Both reads occur after the saved predecessor, result, and seed updates and after predicate evaluation.

6. **State-8 target timing.** If InnerTarget changes only after state 8 saves its next dispatch word, during the subsequent aggregate calculation, that later change does not replace the saved word. The next selector still also depends on the later Base observation.

7. **Zero address components.** If state 8 executes with row and column components zero, it reads two bytes at retained input plus two. Address validity remains a precondition.

8. **Sample truncation.** If the sample-times-LoadScale expression has low 32-bit pattern `12345678`, \(u\) is `5678`, interpreted unsigned.

9. **Saved return.** If default processing saves a result and the outer aggregate later changes without another default action, state 5 returns the earlier saved result.

10. **Large shift.** A derived count of 32 is not established as equivalent to zero. Acceptance is **documented_only** pending execution semantics.

## Uncertainties

The following remain **documented_only**:

- Concrete external values, state reachability, termination, call counts, and sample counts.
- Intended data layout, dimensions, memory extent, and result meaning.
- Sample byte order, alignment requirements, and address-space interpretation.
- Signed-overflow behavior, implementation-dependent conversions and right shifts, address overflow, and invalid or nonrepresentable left shifts.
- External-word declarations and any volatility, atomicity, or stability guarantees absent from the supplied representation.
- Expression-internal read ordering beyond the stated statement boundaries.
- Dependency ownership, global mutations, failures, and termination.

No assumption that repeated reads agree, or that the dependency preserves globals or releases memory, resolves these unknowns.

## Evidence

**Established static evidence:** zero-initialized local state; externally based dispatch; explicit states and shared default action; ordered target observations; one dependency-call site; one unsigned 16-bit load site; width-sensitive arithmetic; and return of a saved result.

**Inferred intent:** structured traversal or packing suggested by paired indices, scaled addresses, phase bits, and shifted accumulation.

**Unavailable evidence:** external-word contents and declarations, dependency implementation, complete target execution semantics, and runtime observations.

No runtime tests have been performed.
