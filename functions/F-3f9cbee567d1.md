<!-- Trusted provenance: F-3f9cbee567d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f9cbee567d1

## Purpose

F-3f9cbee567d1 computes an unsigned result from a signed integer argument under externally supplied control and arithmetic settings. The settings determine whether the argument is used, which arithmetic observations are made, whether an observation becomes the returned result, and whether execution terminates.

The observable result is **the most recently committed candidate**, not necessarily the latest arithmetic observation. Three distinct roles must therefore remain separate:

- An **observation** records a bit-related arithmetic result.
- A **candidate** holds a value eligible for commitment.
- A **committed result** is available for return.

Updating an observation does not publish it. Updating a candidate does not commit it. Commitment does not itself return.

No fixed mathematical predicate or supported configuration is established. Some configurations return zero without using the input; others execute indefinitely.

This contract separates direct static findings from conclusions conditional on an integer execution model. No runtime tests have been performed.

## Interface

### Argument and result

The function accepts one signed integer by value and returns one unsigned integer by value. The argument is an arithmetic operand, not an address.

There are no explicit buffer, length, pointer, callback, or ownership parameters. No record layout, byte-buffer representation, alignment requirement, or caller-provided output area is exposed.

Under the conditional 32-bit model below, every normal return is either **0** or **1**. That range is not established independently of the model.

### Configuration inputs

Eight distinct external scalar locations have the following roles:

| Role | Contractual use |
|---|---|
| Dispatch base | Establishes the initial execution position and participates in subsequent dispatch and continuation calculations. |
| Post-commit position | Determines where execution continues after a candidate is committed. |
| First multiplier | Controls the first product in a paired arithmetic observation. |
| Second multiplier | Controls the second product in that observation. |
| Alternate arithmetic position | Contributes to the continuation selected after paired-product evaluation. |
| Input multiplier | Scales the argument when input evaluation is selected. |
| Post-input position | Determines the continuation after input scaling. |
| Alternate candidate position | Contributes to the continuation selected when the current observation becomes the candidate. |

These are separate scalar locations, not established fields of a shared structure. Their declarations, widths, signedness, qualifiers, initial values, and accessibility guarantees are unavailable.

Configuration is consulted during execution rather than captured as one entry-time configuration object. A predicted path requires either stable settings or a specified value for every relevant reference.

### Conditional integer model

Numerical examples and Boolean-result claims require all of the following:

- The signed argument and execution position are 32-bit quantities.
- Unsigned working quantities are 32 bits.
- Unsigned arithmetic has modulo-\(2^{32}\) behavior.
- Conversion of a 32-bit unsigned pattern to the corresponding signed quantity preserves that pattern.
- Right shift of a negative signed quantity by 31 produces minus one.
- All arithmetic preceding assignment or conversion is defined.
- Configuration expression types and conversions produce the stated execution positions.

These conditions are not supplied by the unsigned destinations alone. In particular, the argument’s multiplication by an externally typed value may have signed intermediate arithmetic. Signed overflow in that intermediate operation cannot be justified by subsequent storage in an unsigned quantity.

The implementation does not check these prerequisites or reject unsupported settings.

## Behavior

### Dispatch and lifetime

Each invocation begins with zero-valued arithmetic history, observation, candidate, and committed result. Its initial execution position is obtained from the dispatch base. The candidate is initialized after that initial configuration reference; the other six unsigned state quantities are initialized before it.

Execution repeatedly selects one of seven numbered actions or a fallback action:

| Selector | Behavioral action |
|---:|---|
| 0 | Make the current observation the candidate. |
| 1 | Evaluate the scaled input. |
| 2 | Evaluate a pair of products and their bit relationship. |
| 3 | Return the committed result. |
| 5 | Derive a candidate from saved arithmetic history. |
| 6 | Commit the candidate. |
| 4 or any unrecognized value | Apply fallback arithmetic. |

There is no invalid-selector error. An unrecognized selector is an arithmetic action, not a termination condition.

Dispatch uses the current execution position and two references to the dispatch base. One base contribution is shifted right by 24; the other is subtracted from it before the execution position is incorporated. The relative evaluation order of those two references is not established.

For stable configuration and defined arithmetic, dispatch can be described in relative coordinates: an execution position displaced by \(d\) from the base selects the base’s right-shifted value plus \(d\). Thus the initial selector is the base shifted right by 24. This simplification does not apply automatically to changing configuration or invalid intermediate arithmetic.

Every action other than return leads to another dispatch. There is no iteration limit or implicit return after commitment.

### Arithmetic history

The function retains an arithmetic workspace, two products, and a saved offset associated with the first product. These values survive unrelated actions within the invocation.

**Input evaluation** replaces the workspace with the argument scaled by the input multiplier. The continuation comes from the post-input position. Existing products, saved history, observation, candidate, and committed result remain intact.

Consequently, evaluating the input again is not a reset.

**Paired-product evaluation** uses the workspace as it stood on entry to that action for both multiplications. The second product does not use the newly computed first product.

The bit relationship being observed is the absence of set bits from both products: the resulting workspace has a set bit exactly where neither product has that bit set. Under the conditional 32-bit model, the resulting observation is:

- **1** when both products have bit 31 clear;
- **0** when either product has bit 31 set.

The action also retains the first product offset downward by two, including unsigned wraparound under the model. This saved value is distinct from the first product itself.

For Boolean observations, continuation is:

| Observation | Continuation |
|---:|---|
| 0 | Alternate arithmetic position |
| 1 | Dispatch base minus one |

The continuation calculation is weighted arithmetic, not a short-circuit branch between configuration references. Both configuration contributions remain part of the expression even when one has zero weight.

The workspace no longer contains the scaled input after this action. Repeating paired-product evaluation without another input evaluation therefore operates on the preceding bit-relationship pattern.

**Fallback arithmetic** advances a different part of the history. The retained first product decreases by one. The observation is then taken from bit 31 of the pattern absent from both that updated first product and the second product offset upward by one.

The upward offset of the second product is temporary; the retained second product does not change. The workspace receives the corresponding absent-bit pattern. Continuation is the dispatch base minus one.

Fallback preserves the saved first-product offset, candidate, and committed result. Repeated fallback therefore accumulates changes in the retained first product without bringing the saved offset up to date.

### Candidate production and commitment

There are two candidate-producing actions.

**Using the current observation** copies the complete observation into the candidate. It is not a separate extraction of bit 31.

For a Boolean observation, continuation is the alternate candidate position for zero and the dispatch base plus five for one. Continuation is determined before the candidate is replaced.

For non-Boolean observations, the continuation cannot be treated as an ordinary true/false choice. Its weights come from conversion through plain character width and then signed integer form; one weight toggles the low bit, while the other retains the converted value. The candidate still receives the original observation, not that narrowed conversion.

**Using saved arithmetic history** derives a candidate from the saved first-product-minus-two value and a temporary upward offset of two from the retained second product. Under the 32-bit model, the candidate is one exactly when both of those patterns have bit 31 clear.

This action also replaces the workspace with the associated absent-bit pattern. Continuation is the dispatch base plus five and is assigned before the candidate’s bit extraction. The current observation is unchanged.

Before any paired-product evaluation, the saved offset is zero. After fallback, it remains the value saved by the most recent paired-product evaluation, even though the retained first product has changed.

**Commitment** replaces the committed result with the candidate, then obtains the continuation from the post-commit position. Execution may subsequently produce another candidate, change arithmetic history, commit again, or return.

**Return** delivers the committed result without further arithmetic or configuration references belonging to a continuation. Return before any commitment yields the initialized zero. Neither an uncommitted candidate nor a newer observation changes that result.

### Conditional invariants

Under the stated 32-bit model:

- Both observation-producing actions produce only zero or one.
- Both candidate-producing actions produce only zero or one.
- Commitment preserves that range.
- The returned value is the latest committed candidate, or zero if no commitment occurred.

Outside that model, the Boolean argument is unresolved. In particular, paired-product observation formation includes unsigned-to-signed conversion, signed shifting by 31, and negation. Different valid execution semantics may invalidate the Boolean interpretation.

## Side effects

### Configuration access and ordering

The visible body contains reads of configuration but no explicit writes to external storage. Its explicit state updates are invocation-local.

Established ordering constraints include:

- Input scaling completes before the post-input position is referenced.
- The first product is assigned before the second product is evaluated.
- Paired-product history and its observation are established before the associated continuation assignment.
- Observation-based candidate production assigns its continuation before replacing the candidate.
- Saved-history candidate production assigns its continuation before extracting the candidate.
- Commitment replaces the committed result before referencing the post-commit position.

Within either weighted continuation expression, the relative order of configuration references is not established. The same limitation applies to the two dispatch-base references within a dispatch expression.

A zero arithmetic weight does not establish that the corresponding configuration reference is absent. Conversely, expression-level references do not establish a physical load count: qualifiers, optimization rules, and storage semantics are unavailable.

There is no visible atomic-access or synchronization protocol. Validity of concurrent configuration changes is **documented_only**.

### Ownership, resources, and errors

No function calls, allocations, releases, I/O operations, or caller-buffer writes are visible. No ownership transfer is established.

There is no distinguished error return. Under the conditional model, zero and one are ordinary committed results.

Nontermination is possible and produces no reported error. Inaccessible configuration, invalid shifts, overflowing signed intermediates, unsupported conversions, and other execution faults have no established recovery behavior or default result.

## Dependencies

No invoked function is visible, so there are no additional opaque function dependency IDs.

Execution depends on:

- The eight external scalar locations and their values at executed references.
- Their types, qualifiers, representation, and accessibility.
- Integer promotion, arithmetic, conversion, and shift semantics.
- Configuration stability for any acceptance claim expressed as a fixed path.

No initializer, validator, or configuration provider is supplied. None is assumed.

There are no visible callees whose global-state preservation or memory ownership can be characterized. The absence of calls does not resolve the external configuration’s own storage semantics.

## Acceptance examples

These are conditional static derivations, not observed runs. Each assumes stable, accessible configuration and defined arithmetic throughout the entire path under the conditional 32-bit model.

### Immediate return without argument use

With dispatch base \(0x03000000\), the initial selector is 3.

Required outcome: return **0**, without input evaluation, commitment, or references to the other seven configuration locations. This outcome must not be replaced by unconditional evaluation of an input predicate.

### Commitment before arithmetic

With dispatch base \(B=0x06000000\) and post-commit position \(B-3\), the selector sequence is:

**6 → 3**

Required outcome: commit the initial candidate zero, then return **0**. No input scaling or product evaluation occurs. The post-commit position is consulted after commitment, not at entry.

### Saved-history boundary

Use these settings:

| Configuration role | Value |
|---|---:|
| Dispatch base | \(B=0x01000000\) |
| Input multiplier | 1 |
| Post-input position | \(B+1\) |
| First multiplier | 1 |
| Second multiplier | 0 |
| Alternate arithmetic position | \(B+3\) |
| Alternate candidate position | \(B+4\) |
| Post-commit position | \(B+2\) |

Required conditional outcomes:

| Argument | Selector sequence | Result |
|---:|---|---:|
| 0 | 1 → 2 → 0 → 6 → 3 | 1 |
| −2147483648 | 1 → 2 → 4 → 0 → 6 → 3 | 1 |
| −2147483647 | 1 → 2 → 4 → 0 → 5 → 6 → 3 | 1 |
| −2147483646 | 1 → 2 → 4 → 0 → 5 → 6 → 3 | 0 |

For the minimum signed argument, fallback moves the first product from \(0x80000000\) to \(0x7fffffff\). That change permits a one-valued observation to become the candidate and then the result.

For argument −2147483647, the saved offset is \(0x7fffffff\). Its highest bit is clear, as is that of the second product’s temporary offset of two. Saved-history candidate production therefore yields one.

For argument −2147483646, the saved offset is \(0x80000000\), whose highest bit forces the candidate to zero.

The last two cases require preservation of the saved offset across fallback. Recomputing it from the fallback-adjusted first product would change the final boundary outcome.

### Nontermination without an error result

With dispatch base zero and alternate candidate position four, execution begins at selector 0 with observation zero. It then reaches selector 4 and fallback.

Fallback selects execution position minus one. Subsequent selectors remain unrecognized and continue to invoke fallback. The first product repeatedly decreases with unsigned wraparound; no commitment or return occurs.

Required conditional behavior: continued execution, not an invented failure return.

## Uncertainties

The following remain **documented_only**:

- Exact widths of the declared integer types.
- External scalar types, qualifiers, representation, and initialization.
- Signedness and behavior of the dispatch-base right shift.
- Unsigned-to-signed conversion behavior for high-bit-set patterns.
- Signed right-shift semantics in paired-product observation formation.
- Intermediate expression types and signed-overflow boundaries.
- Conversion of continuation values to the signed execution position.
- Plain-character signedness, width, and narrowing behavior outside Boolean observations.
- Hardware access counts and ordering beyond the stated expression sequencing.
- Valid concurrent mutation and synchronization requirements.
- Supported configurations and termination guarantees.
- The intended mathematical application.

The literal shift counts remain 24 and 31; they are not established to mean “highest byte” or “sign bit” on every possible integer width.

Without the conditional model, an observation may fail to be Boolean, weighted continuations may cease to behave as binary choices, and the return-range proof may fail. These cases are not replaced by assumed platform behavior.

## Evidence

**Established static evidence:** The supplied body exposes one signed argument, an unsigned return, eight external scalar references, zero-initialized unsigned state, repeated dispatch, and separate observation, candidate, commitment, and return behavior. Selector 4 and every unrecognized selector share fallback behavior. No calls or explicit external writes are visible.

**Conditional derivations:** Relative dispatch coordinates, modulo-\(2^{32}\) arithmetic, bit-31 interpretations, Boolean-result invariants, and the acceptance examples depend on the stated execution assumptions.

**Inferred intent:** The product-bit observations suggest a configurable integer predicate. Its intended domain, configuration scheme, and mathematical meaning are not established.

**Runtime evidence:** No runtime tests have been performed.
