<!-- Trusted provenance: F-86cc0abfa18b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-86cc0abfa18b

## Purpose

F-86cc0abfa18b is a configuration-driven integer state machine. Depending on its dispatch path, it captures its input, stages an argument for F-eea26ecf3091, computes a working product, commits a result, or returns the last committed result.

Routing has a distinct destructive effect: **every routing action clears the working product before selecting the next state.** It does not clear the committed result. Consequently, a commit after routing, without an intervening computation, commits zero even if an earlier computation produced a nonzero product.

The supplied static representation establishes these operations but not a universally selected path, a guaranteed dependency invocation, or termination. An interpretation as a conditional arithmetic calculation is inferred intent, not an unconditional input-to-output formula.

## Interface

### Explicit input and output

The function accepts one signed integer by value and returns one signed integer.

No pointer, buffer, length, object handle, or ownership-bearing resource appears in its explicit interface. The input is initially separate from both the captured-input state and the staged dependency argument:

- Capture copies the original input into the captured-input state.
- Routing copies the captured input into the staged dependency argument.
- Computation passes the staged argument to F-eea26ecf3091.

A capture alone therefore does not change the next dependency argument.

The supplied representation uses signed integer types but does not independently establish their bit width. If the signed type is 32 bits, its range is −2,147,483,648 through 2,147,483,647, and the corresponding unsigned type uses arithmetic modulo 4,294,967,296. These numeric limits are conditional, not established ABI facts.

### External configuration

Seven external integer-valued storage locations participate:

| Role label | Use | Source-level accesses |
|---|---|---|
| Multiplier | Working-product and committed-result multiplication | Read once during initialization |
| Dispatch offset | Added to the current dispatch state | Read once during initialization |
| Initial-state source | Establishes fallback and initial dispatch state | Two separate initialization reads |
| Capture successor | Next state during capture | Read on every capture |
| Commit successor | Next state after committing | Read on every commit |
| Product coefficient | Two factors in the working-product expression | Two occurrences on every computation |
| Nonzero successor | Nonzero contribution to routing | Accessed on every routing action, including zero-input routing |

These labels identify roles, not addresses or a contiguous layout. Storage size in bytes, alignment, byte order, aliasing, initialization protocol, and synchronization are not supplied.

The multiplier, dispatch offset, and fallback are cached for the invocation. The successor and coefficient accesses remain live at their respective action sites.

### Preconditions and error model

Defined execution requires valid configuration accesses and defined behavior from F-eea26ecf3091 whenever invoked. Signed additions, subtraction, and intermediate multiplications must remain representable in the actual signed type.

The function performs no visible validation, overflow detection, error classification, or recovery. Zero and negative dependency results are ordinary arithmetic inputs, not recognized error indicators. There is no explicit error-return convention.

## Behavior

### Initialization and dispatch

The established initialization order is:

1. Read and cache the multiplier.
2. Read and cache the dispatch offset.
3. Initialize the working product to zero.
4. Initialize the committed result to zero.
5. Read the initial-state source and subtract two to establish the fallback.
6. Read the initial-state source again to establish the initial dispatch state.
7. Initialize the captured input to zero.
8. Initialize the staged dependency argument to zero.

The two initial-state accesses must not be collapsed into one observation without an additional stability assumption. The fallback can therefore derive from a different observed value than the initial dispatch state.

Subtracting two is signed arithmetic. If the first initial-state observation is the signed minimum or one greater than that minimum, the subtraction exceeds the signed range; no portable outcome is established.

Each iteration selects an action using the signed sum of the current dispatch state and cached dispatch offset:

| Selected sum | Action |
|---|---|
| 0 | Return |
| 1 | Compute |
| 2 | Commit |
| 4 | Capture |
| 3 or any other representable value | Route |

These are selected sums, not necessarily stored dispatch-state values. Overflow in the selection addition is not handled.

### Return

Selection 0 immediately returns the committed result.

The committed result starts at zero and changes only during commit. Capture, computation, and routing do not directly alter it. Therefore:

- Returning before any commit yields zero.
- A newly computed product does not become the return value until committed.
- Routing can discard a working product while preserving an earlier committed result.
- A subsequent return can expose that preserved result.

No cleanup operation is visible on the return path.

### Compute

Selection 1 calls F-eea26ecf3091 with the current staged dependency argument. The argument is zero until routing changes it, regardless of whether capture has already occurred.

After the dependency returns, its result is used to calculate a new working product. The multiplication is grouped left-to-right:

- The first coefficient occurrence is multiplied by the cached multiplier.
- That intermediate product is multiplied by the second coefficient occurrence.
- That intermediate product is multiplied by the dependency result.

Both coefficient occurrences access the same external storage. Their relative evaluation order is not established, and equality of the observations must not be assumed without an appropriate stability condition.

The dependency call completes before the product expression. A returning dependency can therefore affect coefficient values subsequently used. Changes to the external multiplier do not change the multiplier already cached by this invocation.

Every signed intermediate multiplication must fit. A mathematically zero final product does not excuse overflow in an earlier multiplication.

After the working product is assigned, the dispatch state becomes the cached fallback. Computation neither commits its result nor refreshes the staged argument.

### Commit

Selection 2 multiplies the cached multiplier by the current working product and assigns that product to the committed result.

Only after this assignment does it read the commit successor and install that value as the next dispatch state.

A commit before any computation commits zero. A commit after routing also commits zero unless a computation intervened, because routing resets the working product. Repeated commits use the working product, not the previously committed result.

The multiplication remains signed and requires a representable result.

### Capture

Selection 4 first reads the capture successor and assigns it as the next dispatch state. It then copies the original function input into the captured-input state.

Capture does not change the working product, committed result, or staged dependency argument. It does not invoke F-eea26ecf3091.

Repeated captures copy the same original input; there is no external input refresh.

### Route

Selection 3 and every default selection perform the same action, in this order:

1. Reset the working product to zero.
2. Evaluate the routing expression and assign its result as the next dispatch state.
3. Copy the captured input into the staged dependency argument.

The committed result is unchanged throughout routing.

Routing distinguishes zero from nonzero captured input. At the arithmetic-selection level, zero chooses the fallback contribution and nonzero chooses the nonzero-successor contribution. The input’s magnitude and sign do not otherwise affect this selection.

The expression is not a short-circuit conditional. The nonzero-successor storage participates even when the captured input is zero and its contribution is multiplied by zero. The product reset precedes evaluation of that expression.

An unsigned constant causes the fallback contribution and the final addition to use the corresponding unsigned integer type. The nonzero-successor contribution is a signed multiplication by either zero or one before conversion for the addition; that multiplication does not overflow for a representable successor.

Assignment of the resulting unsigned value back to the signed dispatch type requires care:

- A selected nonnegative value within the signed range is represented without an out-of-range conversion.
- A selected negative fallback or successor is represented through unsigned arithmetic before conversion back to signed.
- The supplied material does not establish the platform behavior for an unsigned result outside the signed range.

A common two’s-complement conversion may recover the intended negative state, but this contract does not assume it.

The staged argument receives the full captured integer, not a Boolean normalization. A later computation can therefore receive a negative input unchanged.

### Progress

Only selection 0 returns normally. There is no iteration bound, cycle detection, timeout, or progress check.

Repeated routing continually clears the working product. Repeated computation can replace it repeatedly. Repeated commits can overwrite the committed result, including replacing a previously nonzero result with zero after routing.

Configuration can produce nonterminating cycles. A nonreturning dependency also prevents normal completion. No universal call count or closed-form result is established.

## Side effects

The visible function reads the seven configuration locations and makes no explicit external-storage writes.

Its internal state changes affect the dispatch state, working product, committed result, captured input, and staged dependency argument. In particular, clearing the working product during routing is an internal state change, not an external write or a reset of the committed result.

Every compute action invokes F-eea26ecf3091. Its effects are unknown. This contract does not assume that it preserves globals, avoids allocation or release, preserves ownership, avoids reentrancy, or returns normally.

Changes made by a returning dependency can affect subsequent live coefficient and successor accesses. They do not refresh this invocation’s cached multiplier, offset, or fallback.

The visible function performs no allocation, deallocation, explicit ownership transfer, logging, or failure reporting. This establishes no ownership rule for resources touched by the dependency.

The stated access order is source-level sequencing. It does not establish volatile transactions, atomicity, memory barriers, or inter-thread visibility. Unsynchronized concurrent mutation is not made safe by this function.

## Dependencies

### F-eea26ecf3091

**Established interface in the supplied representation:** one integer argument and an integer result consumed by signed multiplication.

**Invocation condition:** each visit to compute, selected by sum 1.

**Argument:** the captured-input value copied by the most recent routing action, or zero before any such copy.

**Result treatment:** incorporated into the working product without validation or special handling for zero, negative values, or apparent status codes.

**Unknowns:** exact ABI, argument restrictions, return guarantees, exceptional behavior, configuration mutations, allocation, release, and ownership effects.

No other explicit function calls appear in the supplied representation.

## Acceptance examples

These are conditional static acceptance cases, not observed executions. Unless otherwise stated, configuration is stable, dependency calls return, and all signed arithmetic is representable.

### Immediate return

With initial-state source 0 and dispatch offset 0, the first selected sum is 0.

Expected behavior: return zero without calling F-eea26ecf3091. The original input does not affect the result.

### Commit before computation

With initial-state source 2, offset 0, and commit successor 0, the first action commits and the next returns.

Expected behavior: return zero without a dependency call, for any representable multiplier.

### Captured negative argument and committed computation

Assume:

| Setting | Value |
|---|---:|
| Initial-state source | 4 |
| Dispatch offset | 0 |
| Capture successor | 3 |
| Nonzero successor | 1 |
| Commit successor | 0 |
| Multiplier | 2 |
| Product coefficient | 3 |
| Original input | −5 |

Suppose F-eea26ecf3091 returns 7 for argument −5.

The actions are capture, route, compute, commit, return. Routing first clears the working product and stages −5. Computation then replaces the zero product with 126. The fallback is 2, leading to a commit of 252.

Expected behavior: exactly one dependency call with argument −5; return 252.

### Zero input skips computation

Use the preceding configuration with original input zero.

Routing clears the working product, selects fallback 2, and stages zero. Commit stores zero, then execution returns.

Expected behavior: no dependency call and return zero. The routing expression still includes the nonzero-successor access.

### Routing discards a computed product before another commit

Use the negative-input example, but set the commit successor to 3. During its single invocation, let F-eea26ecf3091 change the nonzero successor to 2 and return 7.

The execution prefix is capture, route, compute, commit, route, commit.

Expected state transitions:

- Computation produces working product 126.
- The first commit stores 252.
- The next routing action clears the working product to zero while leaving the committed result at 252.
- The next commit replaces the committed result with zero.

If the resulting configuration remains unchanged, routing and committing repeat indefinitely. No normal return is asserted.

### Routing preserves a committed result for return

Use the same setup, except the dependency changes the nonzero successor to 0 before returning.

After computation and commit, routing clears working product 126 but preserves committed result 252. It selects state 0.

Expected behavior: the next action returns 252, not zero.

### Stable routing cycle

With initial-state source 5 and offset 0, the fallback is 3. No capture occurs, so the captured input remains zero.

Selection 5 routes to 3; selection 3 routes to itself. Every routing action clears the working product.

Expected behavior: no dependency invocation and no normal return under these assumptions.

### Dependency changes a coefficient

If a compute action is reached and F-eea26ecf3091 changes the coefficient before returning, subsequent coefficient accesses need not use its pre-call value.

Expected behavior: derive the working product from the post-call observations, subject to defined intermediate arithmetic. No exact result follows without those observations.

## Uncertainties

The following remain **documented_only**:

- Actual integer widths, ABI details, and out-of-range unsigned-to-signed conversion behavior.
- Configuration contents, initialization, layout, alignment, and byte order.
- Volatility, atomicity, aliasing, and permitted external mutation.
- Whether the two initial-state observations agree.
- Whether the two coefficient observations agree.
- Dependency domain, termination, exceptional behavior, and resource effects.
- Guaranteed termination, iteration bounds, and dependency-call counts.
- Defined outcomes after invalid access or signed overflow.

Under stable coefficient values, a computation followed by a commit with no intervening routing can apply the coefficient twice and cached multiplier twice to the dependency result. This is a conditional arithmetic interpretation. Routing between those actions destroys the working product and invalidates that result expectation.

## Evidence

The supplied static representation establishes the initialization order, dispatch cases, configuration-access sites, dependency call, arithmetic grouping, routing product reset, successor assignments, and staged-argument update.

Specifically, the routing reset precedes its next-state expression and argument staging. The committed result is assigned only during initialization and commit, so routing preserves it while discarding the working product.

The supplied material does not establish configuration contents, platform type definitions, dependency implementation, synchronization guarantees, or runtime behavior.

No runtime tests have been performed. All acceptance examples are conditional deductions from the supplied static representation.
