<!-- Trusted provenance: F-df261a3fd8d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df261a3fd8d1

## Purpose

F-df261a3fd8d1 sequences operations on a caller-supplied object before a terminal dependency. The supplied static representation has no normal return path. Reaching the terminal dependency is conditional: either of two loops can continue indefinitely even when every dependency invocation returns normally.

The established sequence is:

1. Perform an initial operation at an interior object address, with a one-byte marker written before and after the call. Repeat according to a predicate, replacing the initial operation value with zero on every retry.
2. Invoke a parameterless dependency, then perform a second interior-object operation for each nonzero result from another predicate.
3. Evaluate a final predicate and, on a zero result, invoke an additional interior-object operation and evaluate a global selector.
4. Invoke the same terminal dependency from either final branch.

“Marker,” “predicate,” and “terminal” describe roles in this sequence. They do not establish resource type, success or failure meanings, synchronization semantics, or the mechanism of the terminal action.

## Interface

### Inputs

**Object address.** The first argument supplies the base for every object-relative address used by this function. The representation expresses it as an integer-shaped address, not as a documented object type.

The function performs no explicit validation of this argument. Valid execution requires that each address formation and each resulting access be valid under the target execution model. Null, negative, misaligned, inaccessible, or overflowing address values have no locally defined recovery behavior.

**Initial operation value.** The second argument is a 32-bit quantity. Its signedness and domain interpretation are unspecified. Its full value is supplied to the first invocation of F-fceaba74ea88. Every subsequent invocation by this function receives 32-bit zero.

There is no explicit output parameter.

### Visible object-relative layout

All offsets below are byte offsets from the supplied base.

| Offset | Established use |
|---|---|
| `0x00` | Base address passed to F-80af6764be49 and F-9404073dfe18. |
| `0x04` | Address passed to F-08e7aeef99f4. |
| `0x0c` | Address passed to F-1333331b2da1 on the final zero-result branch. |
| `0x14` | Address passed to F-162a41bac6a7 during the second loop. |
| `0x20` | Address passed to F-fceaba74ea88. |
| `0x26` | Location of the directly written one-byte marker. |

The marker writes replace the entire addressed byte with `1` or `0`; they are not bitwise updates.

For a contiguous object beginning at the supplied base, the direct marker access alone requires storage extending through offset `0x26`, or at least `0x27` bytes. This is not a complete object-size requirement. Dependencies may access storage beyond their supplied addresses or impose additional alignment and lifetime requirements.

The marker is six bytes beyond the address supplied to F-fceaba74ea88. Nothing establishes that these locations belong to separate fields or that the operation cannot access the marker.

### Result and completion

The function has no represented return value. Its terminal dependency, F-feee9156a257, is marked as non-returning.

If both loops terminate and all intervening dependencies return normally, exactly one terminal invocation is reached. Normal returns alone do not establish that either loop terminates.

Reaching the terminal dependency is the last established action, not a normal return to the caller. The evidence does not identify that dependency as process termination, reset, exception delivery, an infinite wait, or another form of control transfer. Behavior after an unexpected return from it is outside the established contract.

## Behavior

### First phase: initial operation and retries

The first phase always attempts an operation before evaluating its repetition predicate. There is no initial predicate that can skip this operation.

Each iteration has this order:

1. Write `1` to the one-byte marker at offset `0x26`.
2. Invoke F-fceaba74ea88 with the address at offset `0x20` and the current 32-bit operation value.
3. If that invocation returns normally, write `0` to the marker.
4. Invoke F-80af6764be49 with the object base.
5. Exit this phase if the predicate returns zero. Otherwise, repeat with zero as the operation value.

The original input value is used only on the first invocation, even if the first predicate result is nonzero. Once a retry is selected, the original value is not restored or resent. An original value of zero follows the same control flow.

The predicate’s result is classified only as zero or nonzero. No positive-versus-negative distinction, retry count, numeric range, or error classification is established.

The marker write of `0` precedes the predicate call. The function does not read the marker to decide whether to repeat.

There is no local iteration bound, timeout, delay, or cancellation branch. If F-80af6764be49 returns nonzero indefinitely and each operation and predicate invocation returns normally, this phase continues indefinitely. The transition dependency, second phase, final predicate, and terminal dependency are then never reached.

The two marker writes bracket a normally returning F-fceaba74ea88 invocation in the caller’s execution order. They do not guarantee that the marker remains `1` while the dependency runs. That dependency or another actor may modify the same byte. If the dependency fails to return, the caller’s subsequent clearing write is not reached.

### Transition between phases

Once the first predicate returns zero, F-338d16f9ce72 is invoked once with no explicit arguments.

No result from this dependency is consumed. The second phase is reached only if this call returns normally. Its domain purpose and effects are not established.

### Second phase: predicate-controlled operations

The second phase evaluates F-9404073dfe18 with the object base before every possible invocation of F-162a41bac6a7.

A zero predicate result ends the phase immediately. A nonzero result causes one invocation of F-162a41bac6a7 with the address at offset `0x14`. After that operation returns normally, the predicate is evaluated again.

Consequently:

- An initial zero result produces no secondary operation.
- Each nonzero predicate result permits exactly one secondary operation before the next predicate evaluation.
- The final zero result does not produce an operation.
- No result from the secondary operation is inspected.

All nonzero predicate results have the same local control-flow effect. The function does not establish that they represent pending work, an error, or another domain condition.

This phase has no explicit iteration bound or timeout. Even with normal returns from every call, an indefinitely nonzero predicate prevents the final predicate and terminal dependency from being reached.

### Final predicate and conditional operation

After the second phase ends, F-08e7aeef99f4 is invoked with the address at offset `0x04`.

For a **nonzero** result, the function proceeds to F-feee9156a257 without invoking F-1333331b2da1 or evaluating the represented global selector.

For a **zero** result, the order is:

1. Invoke F-1333331b2da1 with the address at offset `0x0c`.
2. If it returns normally, evaluate the represented global selector.
3. Invoke F-feee9156a257.

No result from F-1333331b2da1 is consumed.

The selector is evaluated after the conditional operation, not before it. Because the dependency’s global effects are unknown, the selector cannot be assumed to retain any value it had before that call.

The represented dispatch distinguishes values `0` through `9`, but none of those alternatives contains a visible action. No different action is established for values outside that range. All represented selector outcomes lead to the same terminal dependency.

This establishes a selector evaluation in the supplied representation. It does not independently establish a volatile access, an externally observable hardware read, a physical load width, or a distinct storage object unaffected by overlapping global symbols.

### Terminal reachability

The terminal dependency receives no explicit arguments.

Exactly one terminal invocation is reached if:

- the first loop terminates;
- the transition dependency returns normally;
- the second loop terminates;
- the final predicate returns normally; and
- when selected, the conditional operation returns normally.

Earlier calls must also return normally for execution to reach those points. Under the stated non-returning contract of F-feee9156a257, no subsequent action occurs in this function.

A finite sequence of normally returning calls can reach the terminal dependency. An infinite sequence of normally returning calls in either loop cannot. The function provides no independent liveness guarantee.

## Side effects

### Direct memory effects

The only explicit object-memory writes are the marker writes at offset `0x26`: `1` before each first-phase operation and `0` after each normally returning operation.

The marker’s entry value is overwritten without being read, saved, or restored. The function does not explicitly read any other object field.

A clearing write does not guarantee that the marker remains zero. The following predicate and all later dependencies may modify it. Therefore, neither loop exit nor terminal invocation establishes the marker’s final value.

### Dependency effects and ownership

Dependencies receive the base address or fixed interior addresses and may perform additional reads, writes, or resource operations. Those effects are not specified by this caller.

No allocation, release, ownership transfer, lock acquisition, or lock release is explicit in the visible body. Passing an address is not evidence that the callee owns, retains, destroys, or releases the referenced storage.

The caller does not visibly save and restore globals, object contents, or external state around dependencies. A contract for this function must not assume that dependencies preserve those resources. Later operations occur after whatever effects earlier calls actually produced.

### Ordering and concurrency

The stated order is the local execution order of calls and direct writes. No atomic access, memory barrier, mutual exclusion, or reentrancy protection is established.

The marker cannot be treated as a synchronization primitive solely because it brackets an operation. Cross-thread visibility and interference require additional contracts.

### Errors and interrupted execution

There is no explicit error return, rollback, exception handler, or cleanup path.

The loop predicates’ nonzero results mean “repeat” at this caller level. The final predicate’s nonzero result means “skip the conditional operation.” Neither classification establishes success or failure.

If a dependency blocks indefinitely, terminates execution, or transfers control without returning, subsequent steps are not reached. In particular, an interrupted first-phase operation has no visible guaranteed marker cleanup.

## Dependencies

| Opaque function ID | Visible inputs and result use | Caller-established role |
|---|---|---|
| F-fceaba74ea88 | Address at `0x20`; 32-bit value; no result consumed | Initial operation, using the original value once and zero on retries. |
| F-80af6764be49 | Object base; result compared with zero | First-loop predicate: zero exits, nonzero repeats. |
| F-338d16f9ce72 | No explicit arguments; no result consumed | Runs once between loops if reached. |
| F-9404073dfe18 | Object base; result compared with zero | Second-loop predicate: zero exits, nonzero permits one operation. |
| F-162a41bac6a7 | Address at `0x14`; no result consumed | Secondary operation. |
| F-08e7aeef99f4 | Address at `0x04`; result compared with zero | Selects the final conditional path. |
| F-1333331b2da1 | Address at `0x0c`; no result consumed | Runs only after a zero final predicate result. |
| F-feee9156a257 | No explicit arguments; represented as non-returning | Common terminal dependency. |

The predicate results are integer-shaped in the supplied representation. Their external ABI widths are not independently established. Only zero/nonzero comparisons are visible; no narrowing or sign-based interpretation is established.

No callee implementation is available here. The table specifies how this caller uses each dependency, not each dependency’s complete behavior.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Finite examples assume normal returns from all reached dependencies before the terminal invocation.

### Immediate exit from both loops

Given an initial value of `0x12345678`, a first-loop predicate result of zero, a second-loop predicate result of zero, and a nonzero final predicate result:

- The marker is written to `1`.
- F-fceaba74ea88 receives the address at `0x20` and `0x12345678`.
- The marker is written to `0`, then F-80af6764be49 is called.
- F-338d16f9ce72 runs once.
- F-9404073dfe18 runs once; F-162a41bac6a7 is not called.
- F-08e7aeef99f4 runs.
- F-1333331b2da1 and the selector evaluation are skipped.
- Exactly one terminal invocation is reached.

### Retried values and marker order

Given first-loop predicate results of nonzero, nonzero, then zero, the first-phase operation runs three times. Its values are the original 32-bit input, zero, and zero.

Each operation is preceded by its marker write of `1`. Each normally returning operation is followed by a marker write of `0` before its predicate call. An original input of zero does not change the iteration count selected by those predicate results.

### Secondary operation count

Given second-loop predicate results of nonzero, nonzero, then zero, F-162a41bac6a7 runs twice.

The order is predicate, operation, predicate, operation, predicate. If the remaining finite path returns normally, execution then reaches the final predicate and one terminal invocation.

### Conditional operation and selector

Given terminating loops and a zero final predicate result, F-1333331b2da1 runs before the selector evaluation. If it returns normally, selector values `0`, `9`, and a value outside the displayed alternatives establish no differing action before the terminal call.

If that dependency changes the selector, the caller does not restore its earlier value before the evaluation.

### Infinite first loop despite normal returns

If every first-phase operation returns normally and every first-loop predicate invocation returns nonzero, marker bracketing and retries continue indefinitely. There is no transition call, secondary phase, final predicate, or terminal invocation.

### Infinite second loop despite normal returns

If the first loop terminates and the transition returns, but every second-loop predicate result is nonzero and every secondary operation returns normally, the second phase continues indefinitely. Neither the final predicate nor the terminal dependency is invoked.

### Interrupted marker bracket

If the first-phase operation does not return, the preceding marker write of `1` has occurred, but the caller’s following write of `0` has not. The eventual byte value remains dependent on effects of that dependency and other actors.

## Uncertainties

The following remain **documented_only**:

- Complete object layout, field meanings, required alignment, lifetime, and dependency access bounds.
- Exact address width, integer-address conversion rules, and arithmetic overflow behavior. Valid address formation is required; wrapping arithmetic is not assumed.
- Signedness and domain interpretation of the 32-bit operation value.
- Predicate ABI widths and domain meanings beyond the visible zero/nonzero decisions.
- Ownership, retention, allocation, release, synchronization, and global-preservation behavior of every dependency.
- Whether either loop terminates for any particular object state.
- The global selector’s storage identity, width, signedness, volatility, and physical-read behavior.
- The mechanism and effects of the terminal dependency.
- Behavior if the terminal dependency unexpectedly returns.
- Behavior absent from or omitted by the supplied static representation.

A possible intent is to finish or quiesce operations before a terminal transition. That is an inference, not an established contract. It does not justify describing the operations as flushing, draining, unlocking, destruction, or reset.

## Evidence

**Established static evidence:** The supplied representation shows an unconditional first operation, repeated one-byte marker writes, first-call-only forwarding of the initial 32-bit value, two zero/nonzero-controlled loops, fixed object-relative addresses, a parameterless call between the loops, a final conditional operation, an actionless selector dispatch, and terminal calls to the same non-returning dependency.

**Reachability evidence:** Both loops can select another iteration solely from a nonzero predicate result. Normal returns from their dependencies do not force a zero result. Terminal reachability therefore requires loop termination in addition to normal returns from intervening dependencies.

**Evidence limitations:** The supplied material reports removed unreachable blocks and overlapping global symbols. Those annotations do not establish additional reachable actions or a precise selector storage layout. Dependency implementations and independent ABI or object-layout descriptions were not provided.

**Verification status:** No runtime tests have been performed. The acceptance examples express conditions and consequences supported by the supplied static representation, not observed executions.
