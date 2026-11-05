<!-- Trusted provenance: F-a70effbdb56d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a70effbdb56d

## Purpose

F-a70effbdb56d dispatches calls to F-7439060ab81e according to a selector and a repeatedly read discriminator. It scales an input before dispatch, scales dependency results, and either returns the latest scaled result or continues dispatching.

The function is not necessarily a single-call wrapper. Depending on shared values, it can:

- Return zero without calling the dependency.
- Stage the caller’s inputs, call the dependency once, and return a scaled result.
- Initially call the dependency with zero-valued first, second, and third arguments.
- Make further calls using either freshly staged caller inputs or feedback from an earlier dependency result.
- Continue indefinitely if the relevant values remain stable and dependency calls return normally.

The application-level meanings of the selector, discriminator, and multipliers are unknown. The established behavior is numerical dispatch and argument transformation, not a more specific application operation.

## Interface

### Arguments

| Position | Representation established by the supplied material | Use |
|---|---|---|
| First | Opaque 32-bit word | Becomes the dependency’s first argument after a caller-input staging transition. |
| Second | Opaque 32-bit word | Becomes the dependency’s second argument after a caller-input staging transition. |
| Third | Integer represented as `int` | Scaled once at entry using an independently sampled multiplier; the resulting value is retained for later staging. |
| Fourth | Opaque 32-bit word | Passed unchanged as the dependency’s fifth argument on every call. |

The function does not itself dereference the opaque arguments. Their representation does not establish that they are pointers, handles, numeric quantities, or owned resources.

### Return value

The function returns an integer represented as `int`. A normal return supplies:

- Zero if no dependency call has completed; otherwise
- The most recent dependency result multiplied first by the captured second multiplier and captured result-scaling first multiplier.

There is no separate status output or explicit local error result. Zero and negative dependency results receive the same arithmetic treatment as other results.

### Shared inputs and independent samples

Four shared locations are read:

| Descriptive location | Sampling and use |
|---|---|
| Second-multiplier location | Read once at entry. Its captured value scales every dependency third argument and contributes to every returned-result calculation. |
| First-multiplier location | Read twice at entry. The first sample is retained for result scaling. The second sample scales the caller’s third argument. |
| Selector-source location | Read twice at entry. The first sample plus one becomes the fixed post-call selector; the second sample becomes the initial selector. |
| Live-discriminator location | Read repeatedly during dispatch and again when supplying the dependency’s fourth argument. |

The two first-multiplier samples are distinct. The input-scaling sample need not equal the result-scaling sample. Neither is refreshed during subsequent dispatch.

The two selector-source samples are also distinct. The initial selector need not be one less than the post-call selector.

### Widths, arithmetic, and preconditions

The first, second, and fourth arguments are explicitly four-byte words. The other arguments, shared numerical values, intermediate calculations, dependency result, and function result are represented using integer declarations. The supplied material does not independently establish the target integer width, signed-overflow behavior, or ABI.

The arithmetic described below is established for executions where every relevant operation is representable. Relevant operations include:

- Scaling the third input.
- Adding one to the first selector-source sample.
- Subtracting a discriminator sample from the current selector.
- Scaling a pending numeric value for a dependency call.
- Multiplying the captured result factors and the dependency result.

Overflow-sensitive behavior is **documented_only**. No modular, trapping, saturation, or signed-language-overflow interpretation is selected here.

No explicit input validation or memory-validity precondition appears in this function. Requirements imposed by the dependency remain unknown.

## Behavior

### Entry reads and initialization

The shared reads occur in this order:

1. Read the second-multiplier location and retain that value.
2. Read the first-multiplier location and retain this first sample for result scaling.
3. Read the first-multiplier location again and multiply this second sample by the caller’s third input.
4. Read the selector-source location and add one to obtain the fixed post-call selector.
5. Read the selector-source location again to obtain the initial selector.

The entry-scaled third input is retained for staging. It is not recomputed when staging occurs later.

Initially:

- The accumulated result is zero.
- The pending first and second dependency arguments are zero.
- The pending numeric value from which a dependency third argument is formed is zero.

The live discriminator is first read during dispatch, after these initialization reads.

### Dispatch observations

Each dispatch begins by retaining the current pending argument set for a possible dependency invocation.

It then:

1. Reads the live discriminator and subtracts that sample from the current selector.
2. Reads the live discriminator again and compares that new sample with the current selector.

The subtraction and equality comparison therefore need not describe the same discriminator value.

If the equality comparison succeeds, the dependency-call path is selected regardless of the earlier difference.

If the equality comparison fails, the function performs another live-discriminator read. This read occurs before the difference determines whether to stage, return, or call. It occurs even when the function subsequently returns.

The unsuccessful-equality path also prepares caller-derived local values, but these do not replace the argument set already retained for a call in that dispatch. Their lasting effect depends on the branch described below.

### Equality: call with the pending arguments

When the equality comparison succeeds, the function calls F-7439060ab81e with the pending argument set that existed at the start of this dispatch.

The earlier subtraction result does not prevent that call, even if it indicated a difference of one or two using a different discriminator sample.

### Inequality and difference of one: stage caller inputs

When equality fails and the earlier difference is one:

- The pending first argument becomes the caller’s first input.
- The pending second argument becomes the caller’s second input.
- The pending numeric value becomes the entry-scaled third input.
- The current selector becomes the fresh discriminator sample taken on the unsuccessful-equality path.

No dependency call occurs in this transition. The accumulated result is unchanged.

The staged numeric value uses the **second** entry sample of the first-multiplier location. It does not use the sample captured for result scaling and does not reread either multiplier location.

Dispatch then repeats. If the discriminator remains unchanged, the next equality comparison selects a dependency call.

### Inequality and difference of two: return

When equality fails and the earlier difference is two, the function returns the accumulated result.

The unsuccessful-equality path’s additional discriminator read still occurs before this return. It does not cancel the return merely because it observes a new value.

If no dependency call has completed, the accumulated result remains zero.

### Other inequalities: call with the previously pending arguments

When equality fails and the earlier difference is neither one nor two, the function calls the dependency using the pending argument set retained at the start of the dispatch.

This call does not use the caller-derived values temporarily prepared on the unsuccessful-equality path. In particular, an initial dispatch through this branch supplies zero-valued first and second arguments and a third argument calculated from a zero pending numeric value.

After the call, the pending first and second arguments are restored to the values actually supplied to that call. The selector sampled on the unsuccessful-equality path is replaced by the fixed post-call selector.

### Dependency-call arguments

Every call to F-7439060ab81e has five arguments:

| Position | Value |
|---|---|
| First | Pending first opaque word retained for this dispatch |
| Second | Pending second opaque word retained for this dispatch |
| Third | Captured second multiplier multiplied by the pending numeric value retained for this dispatch |
| Fourth | A fresh live-discriminator sample |
| Fifth | Original fourth input |

The fourth argument is sampled separately from the discriminator values used for subtraction, equality, and the unsuccessful-equality transition.

Thus the shared-read sequence for an equality-selected call contains two dispatch discriminator reads followed by the call-argument read. An inequality-selected call contains the additional unsuccessful-equality-path read before the call-argument read.

### Processing a dependency result

After a normal dependency return:

- The captured second multiplier is multiplied by the first-multiplier sample retained for result scaling.
- That product is multiplied by the dependency result.
- The resulting scaled value replaces the accumulated result.
- The same scaled value becomes the pending numeric value for a possible later call.
- The pending first and second words remain those supplied to this call.
- The current selector becomes the fixed post-call selector captured at entry.

The accumulated result is replaced, not added to an earlier result.

A subsequent dependency call may therefore receive a feedback-derived third argument: the captured second multiplier multiplied by the previous scaled result. A difference-of-one transition instead replaces this feedback value with the entry-scaled caller input.

Dependency-side changes to either multiplier location cannot replace the already sampled factors or the retained entry-scaled input. Changes to the live discriminator can affect subsequent dispatch. Changes to the selector-source location are not reread by this invocation.

### Stable-state termination

The following deductions require both selector-source reads to agree, the discriminator to remain constant, all arithmetic to remain representable, and every dependency invocation to return normally.

| Initial selector minus discriminator | Behavior |
|---|---|
| Two | Return zero without a dependency call. |
| One | Stage caller inputs, call once, then return the scaled dependency result. |
| Zero | Call initially with zero-valued pending inputs; thereafter repeatedly stage caller inputs and call. |
| Any other difference | Call initially with zero-valued pending inputs; thereafter repeatedly call using feedback-derived numeric values. |

The last two cases have no normal return under these conditions. This does not establish nontermination when the discriminator changes, a dependency fails to return, or arithmetic leaves the representable domain.

## Side effects

### Direct effects

The function directly reads the four shared locations identified above. It performs no explicit writes to those locations.

Its explicit writes are to invocation-local state: retained samples, staged arguments, the selector, and the accumulated result. No allocation, deallocation, buffer traversal, or externally addressed write is shown.

Repeated reads must not be collapsed in this contract. The supplied material shows distinct samples, but does not establish whether the storage is volatile, atomic, synchronized, or concurrently modified.

### Effects through the dependency

F-7439060ab81e is an opaque side-effect boundary. This contract does not assume it:

- Preserves shared state.
- Preserves objects represented by the opaque arguments.
- Owns, borrows, allocates, or releases memory.
- Accepts zero-valued arguments.
- Returns normally.
- Uses a particular error convention.

The first two caller words cross this boundary only after a staging transition has made them pending. The original fourth word crosses it on every call.

There is no established ownership transfer, borrowing duration, or cleanup obligation. Those matters are **documented_only** pending the dependency’s contract.

## Dependencies

### F-7439060ab81e

This is the sole explicit called dependency.

At this call site it accepts two opaque words, an integer product, a discriminator sample, and the original fourth input. Its result is consumed as an integer.

No result classification occurs before scaling. Zero does not cause an early exit; a negative result is not specially treated as an error. Continued dispatch depends on the selector and discriminator observations, not on a success/failure interpretation of the result.

Its implementation, validity requirements, resource behavior, shared-state effects, and failure behavior are **documented_only**.

## Acceptance examples

These are conditional expectations derived from the supplied control flow, not executed tests. Unless stated otherwise, dependency calls return normally and all arithmetic is representable.

### Immediate return without a call

Both selector-source reads produce 6, and every discriminator read produces 4.

The initial difference is two and equality fails. One further discriminator read occurs on the unsuccessful-equality path. The function returns zero without calling F-7439060ab81e.

The entry multiplier reads and input-scaling operation still occur before this decision.

### Different first-multiplier samples

Assume:

- Captured second multiplier: 3.
- First-multiplier samples, in order: 2 for result scaling, then 5 for input scaling.
- Third input: 7.
- Both selector-source samples: 5.
- Stable discriminator: 4.
- Dependency result: 4.

The retained entry-scaled input is 35. The initial difference of one stages the caller’s first two words and this value.

The dependency receives third argument 105, fourth argument 4, and the original fourth input as its fifth argument.

The scaled result is 24, using result factors 3 and 2 rather than the input-scaling factor 5. The fixed post-call selector is 6. Its difference from the discriminator is two, so the function returns 24.

### Equal first-multiplier samples

With the same conditions as the preceding example, except that both first-multiplier samples are 2, the dependency’s third argument is 42 and the return value remains 24.

This equality of samples is an example condition, not a requirement.

### Initial zero arguments followed by caller-input calls

Both selector-source reads produce 10, and all discriminator reads produce 10.

The first dependency call receives zero for its first three arguments, 10 for its fourth argument, and the original fourth input for its fifth argument.

After the call, the selector is 11. A difference-of-one transition stages the caller’s first two inputs and the entry-scaled third input, then sets the selector to 10. The following dispatch calls the dependency.

This staging-and-call cycle repeats under the stated stable-state and representability conditions. Every staging transition reuses the same entry-scaled input, even if the two first-multiplier samples originally differed.

### Feedback after a discriminator change

Use the differing-samples example: second multiplier 3, result-scaling sample 2, input-scaling sample 5, third input 7, and selector-source samples 5.

The initial dependency call receives third argument 105. Suppose it returns 4 and changes the discriminator to 6.

The accumulated result becomes 24 and the post-call selector is 6. If the following discriminator reads remain 6, a second call occurs with the same first two pending words and third argument 72: the captured second multiplier 3 multiplied by feedback value 24.

If that call returns 5 and changes the discriminator to 4, subsequent stable reads of 4 cause a return of 30.

### Independent discriminator observations

Suppose the current selector is 6. The subtraction read produces 4, but the equality read produces 6.

The computed difference is two, yet equality succeeds. The function calls the dependency rather than returning. The dependency’s fourth argument comes from another discriminator read and need not be either 4 or 6.

### Multiplier changes after entry

Suppose the captured second multiplier is 3, the result-scaling first sample is 2, and the input-scaling first sample is 5.

Later changes to either multiplier location do not change these retained values. Result scaling continues to use 3 and 2. Future caller-input staging continues to use the third input already scaled by 5. Feedback calls use the retained second multiplier and the previous scaled result.

## Uncertainties

- **Integer model — documented_only:** exact integer width, overflow semantics, and overflow-sensitive comparisons are not independently established.
- **Shared-memory model — documented_only:** volatility, atomicity, synchronization, and legal concurrent modification are unknown.
- **Dependency contract — documented_only:** argument validity, ownership, resource effects, error reporting, global-state effects, and termination are unknown.
- **ABI and layout — documented_only:** argument order and opaque-word widths are visible; register placement, stack layout, alignment, and calling convention are not established. No aggregate memory layout is exposed.
- **Application purpose — inferred intent:** the structure resembles state-controlled dispatch with input scaling and result feedback. A more specific purpose is unsupported.
- **Failure behavior:** no local validation or recovery branch is shown. This does not establish that every input is valid or every dependency failure is recoverable.
- **Resource consequences:** dispatch can cause unbounded calls under the stated conditions. Their resource cost depends on the unknown dependency.

## Evidence

The supplied static body establishes the five entry shared reads in their stated order, including the two distinct first-multiplier samples. The first sample contributes to result scaling; the second scales the third input.

It also establishes the two selector-source samples, repeated discriminator observations, branch precedence, preservation of the pre-dispatch pending arguments for calls, five dependency argument positions, result replacement, feedback behavior, and return path.

Stable-state termination cases and acceptance examples are deductions under their explicit conditions, not runtime observations. Application meaning and dependency semantics are not established by this body.

No runtime tests have been performed.
