<!-- Trusted provenance: F-4e2fc8e8f90e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4e2fc8e8f90e

## Purpose

F-4e2fc8e8f90e searches an index range derived from global configuration. For each attempted index, it obtains a candidate, processes that candidate using temporary local state, obtains a reference value from the second input, and compares that reference with the temporary state’s second word.

A comparison result of exactly zero selects the current index. The function obtains an index-associated result, saves it across a cleanup call, and returns it. If the range is exhausted without selection, the function obtains and returns a fresh reference value.

This describes observable control flow, not domain meaning. The supplied static evidence does not establish whether the values represent strings, resources, handles, addresses, or other objects.

## Interface

### Inputs and output

The function accepts four 32-bit quantities and returns one 32-bit quantity.

| Input | Established use |
|---|---|
| First | No explicit use in the visible body. |
| Second | Forwarded unchanged to F-44362605ab79 for each attempt and again on exhaustion. |
| Third | Written into the first word of a temporary local region before dependency calls. |
| Fourth | Written into the second word of that region before dependency calls. |

The input widths do not establish semantic types. In particular, a 32-bit quantity may encode an address rather than an ordinary numeric value.

Reusing the second input means reusing its original bits. It does not imply that successive F-44362605ab79 calls return equal results or observe unchanged external state.

The returned quantity is:

- The saved result of F-c0b020cd3882 when a comparison returns zero.
- The result of an additional F-44362605ab79 call when the range is exhausted.

There is no separate success indicator. A zero returned quantity has no established special meaning.

### Temporary-state layout

The supplied representation identifies two adjacent 32-bit local words:

| Byte offset from the region’s beginning | Initial value |
|---|---|
| 0 | Third input |
| 4 | Fourth input |

F-7afe554bcf62 and F-289ec4311c64 receive the address at offset 0. The word at offset 4 supplies the first argument to F-4a6c0bf0740c.

These observations establish the visible layout, not a complete object format or an access-permission contract. No length, capacity, ownership field, type tag, or permitted extent of dependency accesses is established.

The word at offset 4 is read **after F-44362605ab79 returns on that attempt**. Its value is not necessarily the fourth input or the value most recently written by F-7afe554bcf62.

### Operational preconditions

Normal execution requires:

- F-17fe47085641 to return a receiver usable for the subsequent indirect call.
- The receiver’s first word to identify a readable dispatch structure with a callable entry at byte offset 228.
- Each dependency to accept the supplied arguments and return normally.
- The relevant index arithmetic and memory accesses to have defined behavior in the execution environment.

The function does not check these prerequisites or expose a local recovery path when they fail. They describe requirements imposed by unchecked operations, not guarantees supplied by the function.

## Behavior

### Initialization and acquisition

The function first writes the third and fourth inputs into the temporary region, at offsets 0 and 4 respectively.

It then performs the following ordered acquisition sequence:

1. Calls F-17fe47085641 without arguments.
2. Uses that result as a receiver, reads its dispatch information, and invokes the dispatch entry at byte offset 228 with the receiver and literal 8.
3. Calls F-682adfd5a561 with the indirect call’s result and literal 5.

The result of F-682adfd5a561 becomes the candidate-source value used throughout this invocation. There is no explicit reacquisition between attempts.

The indirect target’s identity and internal behavior are unavailable. The dispatch offset identifies a location in the supplied layout, not a portable interface for other pointer widths.

### Configuration reads

After acquisition, the function performs three configuration reads in this order:

1. Reads a selector and retains it for candidate lookup.
2. Reads a signed base and adds 2 to establish the retained inclusive upper bound.
3. Reads the same base location again to establish the starting index.

The index and bound are represented as signed 32-bit values. The continuation comparison is signed.

There is no explicit dependency call between these configuration reads. Nevertheless, the evidence does not establish synchronization, atomicity, or equality of the two base observations. Later dependency effects on configuration do not cause an explicit reload by this function.

With stable configuration and defined arithmetic, the ordinary candidate sequence is the base, base plus 1, and base plus 2. That statement is conditional: the initial addition and any executed increments must have defined behavior.

### Each attempted index

The function enters the attempt body before checking the index against the upper bound. Reaching the body therefore causes at least one attempt, even when the starting index exceeds the retained bound.

Each attempt has this observable order:

1. F-5710a8628cba receives the retained candidate source, retained selector, and current signed index.
2. F-7afe554bcf62 receives the temporary-region address and the candidate returned by F-5710a8628cba.
3. F-44362605ab79 receives the original second input and returns a reference value.
4. The temporary region’s second word is read.
5. F-4a6c0bf0740c receives that word value as its first argument and the newly obtained reference as its second argument.
6. Its signed 32-bit result is tested for equality to zero.

The return value of F-7afe554bcf62 is ignored. Its effects on the temporary region or other state are dependency-defined.

Crucially, candidate processing and the comparison’s local-word read are separated by F-44362605ab79. Although that dependency is not explicitly passed the temporary-region address, an earlier dependency could have retained the address or otherwise made the region reachable. Its unknown effects therefore cannot be assumed to preserve the region.

The comparison receives the second word’s value when it is read after F-44362605ab79 returns. A value written by F-7afe554bcf62 supplies that argument only if no intervening effect changes it.

The temporary region is initialized only once. Neither word is explicitly restored from the inputs before subsequent attempts. Any reset, replacement, release, or preservation performed by dependencies is outside the visible contract.

### Selection on zero

If F-4a6c0bf0740c returns exactly zero:

1. F-9d0ecc5f6adf is called without arguments.
2. F-c0b020cd3882 receives its result and the selected index.
3. The result of F-c0b020cd3882 is saved.
4. F-289ec4311c64 receives the temporary-region address.
5. The saved result is returned.

Result construction precedes cleanup. The cleanup dependency’s ordinary return value is ignored and does not replace the saved return quantity.

This branch performs no index increment, no further attempt, and no exhaustion call to F-44362605ab79.

Saving the return quantity does not establish the continued validity of any storage it might identify. Cleanup may affect aliased storage; no alias-safety check is visible.

### Continuation on nonzero

Every nonzero comparison result follows the same branch. Positive and negative values have no distinct treatment.

The function:

1. Calls F-289ec4311c64 with the temporary-region address.
2. Increments the signed index by 1.
3. Continues if the incremented index is less than or equal to the retained upper bound.

Cleanup thus occurs before both the increment and the range test.

An initially out-of-range index is still attempted. If its comparison is zero, it is selected without an increment or bound check. If its comparison is nonzero, cleanup and increment occur before deciding whether another attempt is allowed.

### Exhaustion

If the post-increment continuation test fails, the function calls F-44362605ab79 once more with the original second input and returns that invocation’s result.

The fallback is a fresh call, not reuse of the final attempt’s reference. It occurs after the final cleanup, so its result can reflect effects of that cleanup or any earlier dependency.

### Arithmetic boundaries

The visible arithmetic consists of signed base-plus-2 and signed index-plus-1 operations. Overflow behavior is not established.

The final failed attempt still increments the index before testing the bound. Consequently, even a representable upper bound does not by itself establish defined exhaustion behavior. For example, a stable base of 2,147,483,645 gives a representable upper bound of 2,147,483,647, but a nonzero comparison at that upper bound requires an increment beyond the signed 32-bit maximum.

No wraparound, saturation, trap, or fixed attempt count is promised where an executed operation overflows.

## Side effects

### Visible local operations

The function explicitly:

- Initializes the two local words before external calls.
- Passes the region’s address to candidate processing and cleanup.
- Reads its second word after each attempt’s reference-generation call.

No direct write to the global configuration is visible. The selector and bound are retained after setup rather than explicitly refreshed during the search.

### Dependency effects and pointer escape

All other effects are delegated. The supplied evidence does not establish whether dependencies allocate memory, release memory, retain addresses, modify shared state, perform I/O, or invoke other operations.

Passing the temporary-region address to F-7afe554bcf62 does not establish borrowing, copying, ownership transfer, or non-escape. Likewise, F-289ec4311c64 is called “cleanup” here because of its position in the control flow, not because deallocation or destruction is proven.

No dependency should be assumed to preserve globals or previously exposed local storage merely because its explicit arguments omit those locations.

### Ownership and lifetime

The temporary region is local to this invocation. Its address is not directly returned, but dependencies receive it and may retain it; no non-retention guarantee is supplied.

The visible body contains no explicit release operation for the acquisition receiver, indirect-call result, candidate source, candidates, reference values, or selected return value. This does not establish a leak. These values may be non-owning quantities, or ownership operations may occur inside dependencies.

Any validity requirement for a returned address or handle after cleanup remains dependency-defined.

### Errors and abnormal exits

There are no explicit null checks, dependency-status checks, retries, or exception handlers.

Only the comparison result controls selection. A nonzero result is treated as continuation, not as a separately recognized error condition.

On normal control-flow paths, each completed attempt invokes cleanup once. This is not an abnormal-exit guarantee: if any dependency fails to return normally—including either success-only dependency before cleanup—the supplied evidence does not establish that cleanup runs.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-17fe47085641 | Called without arguments; returns the dispatch receiver. |
| F-682adfd5a561 | Receives the indirect-call result and literal 5; returns the retained candidate source. |
| F-5710a8628cba | Receives candidate source, selector, and current index; returns the candidate. |
| F-7afe554bcf62 | Receives temporary-region address and candidate; ordinary return value ignored. |
| F-44362605ab79 | Receives the second input; returns each attempt’s reference and the fresh exhaustion result. |
| F-4a6c0bf0740c | Receives the second local word’s post-reference-call value and the reference; exactly zero selects. |
| F-9d0ecc5f6adf | Called without arguments after selection. |
| F-c0b020cd3882 | Receives the preceding dependency’s result and selected index; returns the saved selected result. |
| F-289ec4311c64 | Receives temporary-region address after each normally completed attempt; ordinary return value ignored. |

The additional indirect dependency is selected at dispatch byte offset 228 and receives the receiver and literal 8. No opaque ID for its target is supplied. Its identity and behavioral contract remain **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, dependencies return normally and executed arithmetic is defined. Call counts refer to invocations directly made by this function, not unknown nested calls.

### First index selected

With stable base 20 and a zero comparison at index 20:

- Only index 20 is attempted.
- F-44362605ab79 is called once.
- F-9d0ecc5f6adf and F-c0b020cd3882 run before cleanup.
- Cleanup runs once.
- The saved F-c0b020cd3882 result is returned.
- No increment or fallback occurs.

### Third index selected

With stable base 20 and nonzero, nonzero, then zero comparison results:

- Indices 20, 21, and 22 are attempted in order.
- Each attempt calls F-44362605ab79 once.
- Cleanup runs once per attempt.
- Only index 22 is supplied to F-c0b020cd3882.
- No exhaustion call occurs.

### All three comparisons nonzero

With stable base 20 and nonzero comparisons at 20, 21, and 22:

- Cleanup runs three times.
- The final increment produces 23, which exceeds the retained bound of 22.
- F-44362605ab79 is called a fourth time after cleanup.
- Its fourth result is returned, regardless of whether earlier results differed.
- Neither success-only dependency is called.

### Different base observations

If the bound calculation observes base 20 and the starting-index read observes 24:

- The retained bound is 22.
- Index 24 is nevertheless attempted.
- A zero comparison selects 24.
- A nonzero comparison causes cleanup, increment to 25, a failed continuation test, and fallback.

This illustrates separate reads; it does not establish support for concurrent configuration changes.

### Temporary-state writes and intervening effects

Suppose F-7afe554bcf62 writes value A into the second local word.

- If F-44362605ab79 returns without any intervening effect changing that word, F-4a6c0bf0740c receives A as its first argument.
- If an intervening effect changes that word to B—for example, through an address retained by an earlier dependency—the comparison receives B.
- The candidate-processing write alone is insufficient to determine the comparison argument.

On later attempts, the original fourth input is not automatically restored. The same post-reference-call read rule applies every time.

### Signed upper boundary

With stable base 2,147,483,645, defined initial addition, and nonzero comparisons on the first two attempts, index 2,147,483,647 is reached.

A zero comparison there takes the selection branch without incrementing. A nonzero comparison invokes cleanup and then requires an unestablished overflow operation. The supplied evidence does not determine subsequent behavior.

## Uncertainties

- **Semantic types — documented_only:** Domain meanings of inputs, candidates, configuration, and results are unavailable.
- **Dependency preservation — documented_only:** No global-state or exposed-memory preservation guarantees are supplied, including for F-44362605ab79.
- **Ownership and aliasing — documented_only:** Allocation, release, retention, pointer escape, and returned-value validity are unresolved.
- **Dispatch target — documented_only:** The indirect target’s identity and contract are unavailable.
- **Configuration consistency — documented_only:** No synchronization or stable-snapshot guarantee is established.
- **Overflow — documented_only:** Signed overflow semantics are unresolved.
- **Portability — documented_only:** The represented widths and dispatch layout do not establish equivalent layouts on other execution environments.
- **Cleanup intent — inferred:** Placement suggests temporary-state finalization; actual destruction, release, or reset is unproven.
- **Comparison intent — inferred:** Zero serves as this function’s selection condition; a broader equality or ordering contract is unknown.
- **Abnormal execution — documented_only:** Faults, exceptions, nonlocal exits, and recovery behavior are unavailable.

## Evidence

Established claims derive from the supplied static function representation and opaque dependency mapping: input forwarding, represented widths and local layout, ordered configuration reads, dependency-call order, the local-word read after reference generation, zero-result selection, cleanup placement, saved selected result, and fresh exhaustion call.

The possibility of intervening mutation follows from the absence of dependency contracts and the earlier exposure of the temporary-region address; such mutation is not asserted to occur.

Domain semantics, dependency internals, ownership rules, synchronization, and overflow behavior are not established by the supplied evidence.

No runtime tests have been performed.
