<!-- Trusted provenance: F-48a3dae42ee4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-48a3dae42ee4

## Purpose

F-48a3dae42ee4 is a two-stage adapter. It captures three incoming 32-bit values in temporary local storage, passes the address of the first stored value to F-3b3feb156f36, and then passes that same address and a Boolean condition to F-ef010853c3a4. The condition distinguishes a zero selector from a nonzero selector.

The established behavior is the storage preparation, dependency order, pointer reuse, and zero-versus-nonzero classification. The supplied evidence does not establish what either dependency accomplishes, what the three stored values represent, or whether the second dependency consumes changes made by the first.

An interpretation such as “prepare a record, then apply an operation in a selected mode” is inferred intent only. It is not a substitute for the observable contract below.

## Interface

### Inputs

The function has four incoming arguments:

| Position | Role | Established representation |
|---|---|---|
| First | Selector used to form the second dependency’s Boolean argument | An integer, compared against zero |
| Second | Initial value of the first temporary slot | 32-bit value |
| Third | Initial value of the second temporary slot | 32-bit value |
| Fourth | Initial value of the third temporary slot | 32-bit value |

The three value arguments are copied without an explicit conversion, mask, arithmetic operation, or interpretation. Their role in this function is to supply 32-bit stored values. Whether any of those values denotes an address, handle, flags, numeric quantity, or other representation is unknown.

The selector’s signedness does not change the relevant distinction: zero produces false, and every nonzero value produces true. The evidence does not support reducing the selector to a particular bit, testing positivity, or forwarding its original magnitude.

The source representation identifies the selector as an integer but does not independently provide a target data model or calling convention. A 32-bit selector is consistent with the representation, but its exact machine width remains documented_only. The three other arguments explicitly have four-byte representations.

### Output

There is no explicit return value.

On the ordinary returning path, completion means both dependency calls have returned and this function has then returned to its caller. There is no explicit success result, failure result, status conversion, or result aggregation.

Any dependency return value is unused by this function. That does not imply the dependencies have no return values or that their results are semantically unimportant elsewhere.

### Preconditions

The caller must satisfy the actual calling convention and argument representations. No additional input validation is visible.

The temporary storage must be accessible for the dependency calls. Requirements imposed by the dependencies on the three stored values are unknown. Consequently, an arbitrary combination of 32-bit values cannot be declared safe merely because this wrapper performs no validation.

## Behavior

### Temporary-state preparation

Before either dependency is invoked, the function initializes three distinct local 32-bit objects:

1. The first receives the second incoming argument.
2. The second receives the third incoming argument.
3. The third receives the fourth incoming argument.

These initializations occur in that order in the supplied representation. All three precede the first dependency call.

The function takes the address of the first local object. The supplied representation does not explicitly construct an array or declare a record containing all three values. It therefore establishes three stored values and an address to the first, but not a portable aggregate layout.

The three objects contain 12 bytes of value storage in total. That does not establish a 12-byte buffer beginning at the address passed to the dependencies. Padding, alignment, relative placement, and the validity of accessing the other two objects through that address are not independently established by the supplied evidence.

### First dependency call

F-3b3feb156f36 is invoked with one visible argument: the address of the first temporary object.

The wrapper does not pass the selector, a length, a capacity, or a separate address for either of the other two temporary objects to this dependency.

The first dependency runs before the second. If it does not return normally, the ordinary sequence does not reach the second call. The wrapper contains no visible mechanism for recovering from a dependency failure or forcing the later call to occur.

No visible dependency result is inspected.

### Transition between calls

After F-3b3feb156f36 returns, the wrapper does not explicitly restore or reinitialize any of the three temporary values. It also does not copy the temporary state into another object before the next call.

Thus, the second dependency receives the same first-object address that was supplied to the first dependency. If the first dependency changes storage accessible to the second, and those changes remain present when the second call begins, this wrapper does not undo them.

This is a statement about the wrapper’s actions, not a claim that the first dependency actually writes anything. Its reads, writes, and accessible memory range are unknown.

### Second dependency call

F-ef010853c3a4 is invoked with two visible arguments:

- The same first-object address used for F-3b3feb156f36.
- The result of comparing the selector with zero for inequality.

The comparison appears after the first dependency call in the supplied sequence. Its semantic result is false for zero and true for nonzero. There is no arithmetic on the selector and no possibility of arithmetic overflow in the visible classification.

The Boolean argument expresses a logical condition. Its ABI carrier width is not established here. At the source-expression level, the condition is normalized to false or true rather than preserving the selector’s original integer value.

No visible result from F-ef010853c3a4 is checked or transformed. If that call returns normally, the wrapper returns without further explicit work.

## Side effects

### Local writes and exposed memory

The wrapper explicitly writes the three incoming values into local storage before invoking external behavior.

The address passed to both dependencies identifies temporary local storage, not an allocation visibly obtained from a heap allocator. The wrapper itself performs no visible allocation, deallocation, ownership transfer, or release operation.

Passing the address does expose that storage to the dependencies during their execution. The evidence does not establish whether either dependency:

- Reads or writes the first object.
- Reaches the other two local objects.
- Treats the values as a structured object.
- Interprets a stored value as a pointer.
- Retains or publishes the supplied address.
- Reads or writes global state.
- Performs I/O, allocation, release, callbacks, or synchronization.

None of those behaviors may be assumed absent merely because they are not visible inside this wrapper.

### Lifetime and ownership

The temporary objects have local lifetime. This wrapper does not establish a caller-owned output buffer or persistent storage for dependency use after the function returns.

There is no visible ownership annotation or explicit transfer protocol. In particular, the pointer argument alone does not establish that a dependency may release the supplied storage.

If a dependency retains the address, its later use would require a lifetime guarantee not supplied by this function. Whether retention occurs is documented_only.

The wrapper also does not establish ownership of anything that a 32-bit stored value might reference. A value that happens to encode an address is still only copied here; pointee ownership and release obligations remain unknown.

### Errors and abnormal control flow

There is no visible validation, error branch, cleanup handler, retry, rollback, or diagnostic operation.

Dependency failure semantics are unknown. A dependency might return normally, terminate execution, or use another control-transfer mechanism; this evidence does not determine which outcomes are possible.

The ordinary ordering guarantee is conditional on return: the second call follows the first only when control returns from the first. The wrapper returns only when control also returns from the second.

No atomicity guarantee is established. If the first dependency produces externally visible effects and later execution fails, no compensating action is visible.

## Dependencies

| Opaque function ID | Arguments supplied by this function | Established role |
|---|---|---|
| F-3b3feb156f36 | Address of the first temporary 32-bit object | First external operation after all three initializations |
| F-ef010853c3a4 | Same address, followed by the selector’s nonzero condition | Second external operation, conditional on the first returning |

Dependency names do not supply behavioral evidence. Their algorithms, memory-access contracts, ownership rules, global-state effects, and failure behavior are unknown.

The wrapper establishes an ordered dependency relationship, not independence. Effects of the first call may matter to the second through the supplied address or through other state, but the existence and nature of such effects are not established.

No claim is made that either dependency preserves globals, preserves the temporary contents, or owns or releases any referenced resource.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests.

### Zero selector

Given a zero selector and three valid incoming 32-bit values, with both dependencies returning normally:

- The three temporary objects are initialized from the value arguments in argument order.
- F-3b3feb156f36 is called first.
- F-ef010853c3a4 is called next with the same first-object address and a false condition.
- The wrapper then returns without an explicit result.

### Nonzero selector

Given a selector equal to one, two, or any other representable nonzero value, the second dependency receives a true condition.

If the selector representation supports negative values, a negative nonzero value also produces true. Positivity is not the criterion.

### Full-width stored values

Given incoming value arguments containing all-zero bits, all-one bits, and a mixed 32-bit pattern, respectively, the visible initializations preserve those values in their corresponding local objects.

There is no visible truncation to a byte or halfword, sign-dependent conversion, or special treatment of a zero value argument. Subsequent interpretation by the dependencies is outside the established contract.

### First-call mutation

Assume F-3b3feb156f36 changes the first temporary object and returns normally. The wrapper must not explicitly restore the original incoming value before invoking F-ef010853c3a4.

The second call must receive the same object address. Whether it reads the changed value depends on its own behavior.

### First call does not return

Assume F-3b3feb156f36 does not return control normally. The ordinary sequence must not be described as guaranteeing a later F-ef010853c3a4 call or a wrapper return.

### Second call returns a status

Assume F-ef010853c3a4 returns a status through its actual interface. This wrapper does not visibly branch on that status, return it, or translate it into another explicit result.

## Uncertainties

The following remain documented_only:

- Exact calling convention, selector width, and Boolean argument carrier width.
- Exact alignment and relative layout of the three local objects.
- Whether the first-object address designates any larger valid region.
- Meaning and valid domains of the three 32-bit values.
- Read and write ranges of both dependencies.
- Whether either dependency retains the supplied address.
- Ownership or release behavior for resources represented by stored values.
- Global-state changes, callbacks, I/O, synchronization, and reentrancy.
- Dependency return types and failure semantics.

The strongest supported interpretation is an ordered, two-call adapter over temporary state. More specific domain semantics would require evidence about the dependencies or their callers.

## Evidence

Established static evidence consists of three four-byte local initializations, two ordered dependency invocations using the same first-object address, a selector-versus-zero inequality condition supplied to the second invocation, and a return without an explicit value.

The supplied representation contains no visible validation, allocation, release, restoration between calls, result handling, or error recovery. This absence describes the wrapper only; it does not constrain hidden dependency behavior.

No runtime tests have been performed. The acceptance examples specify expected behavior under their stated conditions and do not claim execution evidence.
