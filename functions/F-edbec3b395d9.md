<!-- Trusted provenance: F-edbec3b395d9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-edbec3b395d9

## Purpose

F-edbec3b395d9 is a classification-and-dispatch wrapper. It examines two input words, conditionally invokes an auxiliary dependency, and either returns a result produced by another dependency or invokes a notification dependency before returning a value from global storage.

Its classification combines the sign-cleared second word with a one-bit indication of whether the first word is nonzero. The classification is fixed before any dependency call. The auxiliary dependency’s return value does not control dispatch.

The masks and thresholds are consistent with inspecting a binary64 floating-point representation supplied as two words. That interpretation is inferred intent, not an established interface requirement. The established behavior is defined by the word operations and call ordering below; no numerical meaning is required to apply it.

## Interface

### Inputs

The interface accepts two scalar words:

| Input | Width and interpretation | Use |
|---|---|---|
| First word | 32-bit signed integer at the shown interface | Tested for equality to zero; otherwise forwarded unchanged |
| Second word | 32-bit unsigned integer | Masked for classification; otherwise forwarded unchanged |

For classification, the first word contributes only whether it is zero. Its sign and magnitude do not otherwise matter. Dependencies receive the original first word, not a normalized zero-or-one replacement.

The second word’s most significant bit is ignored for classification. Dependencies receive the original second word, including that bit.

Neither input is dereferenced by the wrapper. The supplied evidence establishes no pointer, buffer, alignment, or allocation precondition for these inputs. Dependencies may impose requirements that are not visible here.

### Output

The function returns one opaque 32-bit value:

- On delegated paths, it returns the 32-bit result obtained from F-6dafbb420982.
- On the alternate path, it returns the value read from a designated global result location after F-77dd9f40f9b6 returns.

The evidence does not establish whether the result represents an integer, a floating-point fragment, a pointer-sized value, a status, or another encoding. No additional return component is established.

### Global storage roles

Three global storage locations appear in the supplied body:

- **Auxiliary datum A:** supplies the third argument to F-11ed8242c40c.
- **Auxiliary datum B:** supplies the fourth argument to F-11ed8242c40c.
- **Fallback result:** supplies the returned value on the alternate path.

These are descriptive roles, not claims about their contents, mutability, or ownership. The supplied evidence does not include their declarations, initial values, or access widths. The function’s returned value is nevertheless 32 bits.

## Behavior

### Classification

Define a classification quantity, **Q**, as follows:

- Clear the most significant bit of the second word.
- Combine that value by bitwise OR with one if the first word is nonzero, or zero if the first word is zero.

Consequently:

- Q lies between 0 and 0x7FFFFFFF, inclusive.
- A nonzero first word sets Q’s least significant bit; it does not numerically add one.
- If the sign-cleared second word already has its least significant bit set, changing the first word between zero and nonzero does not change Q.
- Changing only the second word’s most significant bit never changes Q.

The wrapper subtracts Q from 0x7FF00000 and uses both the signed sign of that result and bit 30 of its 32-bit representation.

The subtraction’s mathematical range is −0x000FFFFF through 0x7FF00000. Thus, under the 32-bit interpretation shown, the subtraction does not require signed overflow or wraparound to explain the behavior.

The resulting dispatch partitions all possible Q values into three regions:

| Q, inclusive | Auxiliary call | Subsequent action |
|---|---|---|
| 0x00000000–0x3FF00000 | F-11ed8242c40c | Call F-6dafbb420982 and return its result |
| 0x3FF00001–0x7FF00000 | F-11ed8242c40c | Call F-77dd9f40f9b6, then read and return the fallback result |
| 0x7FF00001–0x7FFFFFFF | None | Call F-6dafbb420982 and return its result |

The endpoints are significant. Equality with 0x3FF00000 selects delegation after the auxiliary call. Equality with 0x7FF00000 selects the notification-and-global-result path, not direct delegation.

### Direct delegation

When Q exceeds 0x7FF00000, the subtraction is negative. The wrapper short-circuits directly to F-6dafbb420982.

On this path:

1. F-11ed8242c40c is not called.
2. Auxiliary datum A and auxiliary datum B are not read by the wrapper.
3. F-6dafbb420982 receives the original first and second words, in that order.
4. Its result becomes the wrapper’s result.

F-77dd9f40f9b6 is not called, and the wrapper does not read the fallback result on this path.

### Auxiliary call

When Q is at most 0x7FF00000, the wrapper calls F-11ed8242c40c before choosing between its remaining actions.

The six arguments, in positional order, are:

1. Original first word.
2. Original second word.
3. Value obtained from auxiliary datum A.
4. Value obtained from auxiliary datum B.
5. Original first word again.
6. Original second word again.

The duplicate input arguments are part of the visible call contract. There is no evidence of an intervening conversion or normalization of those copies.

Both global argument values are obtained for this call. Their relative read order is not established by the supplied expression. A conforming behavioral description must not require A to be read before B, or B before A.

The auxiliary call’s return value is discarded. Its side effects may matter, but its returned value is not tested.

The classification flag used after this call was computed beforehand. The wrapper does not reread the inputs, recompute Q, or repeat the subtraction after the call.

### Delegation after the auxiliary call

For Q from 0 through 0x3FF00000, inclusive, the wrapper calls F-6dafbb420982 after F-11ed8242c40c returns normally.

F-6dafbb420982 receives the original two input words. Its result is returned unchanged as the wrapper’s 32-bit result.

The wrapper does not call F-77dd9f40f9b6 or read the fallback result on this path. F-6dafbb420982 executes in whatever externally visible state remains after the auxiliary call; no state restoration is shown.

### Notification and fallback result

For Q from 0x3FF00001 through 0x7FF00000, inclusive, the wrapper calls F-77dd9f40f9b6 after F-11ed8242c40c returns normally.

F-77dd9f40f9b6 receives the integer value 1 as its sole shown argument. Its return value is not used.

Only after that call returns does the wrapper read the fallback result and return it. This timing is material: the contract does not substitute the fallback result’s entry-time value or a value captured before either dependency call.

F-6dafbb420982 is not called on this path.

### Abnormal completion

The supplied body contains no explicit recovery, retry, or error-return translation.

The sequences above assume that the invoked dependencies return normally and that the required global accesses complete. If a dependency terminates execution, transfers control elsewhere, or fails to return, the evidence does not establish a normal wrapper result.

The integer argument 1 passed to F-77dd9f40f9b6 must not be assigned a specific error category without additional evidence.

## Side effects

The wrapper’s direct visible operations consist of classification, conditional global reads, dependency calls, and return-value selection. No direct global write, input-memory write, allocation, or release is shown.

The absence of direct writes does not make the function pure:

- F-11ed8242c40c may change state before either subsequent action.
- F-6dafbb420982 may change state while producing the returned result.
- F-77dd9f40f9b6 may change state before the fallback result is read.

No dependency is established to preserve globals, own memory, release memory, or leave error state unchanged. In particular, the fallback result may be affected by either preceding call on its path.

The wrapper performs no visible snapshot or rollback of dependency effects.

No ownership transfer is established for the scalar inputs or global argument values. If a value is interpreted as a resource or address by a dependency, the relevant ownership rules remain unknown.

Concurrency, atomicity, volatile access semantics, and synchronization are not established. The ordered operations described here are the wrapper’s visible sequencing; they are not a guarantee of an atomic transaction.

## Dependencies

| Opaque function ID | Conditions for invocation | Arguments | Result handling |
|---|---|---|---|
| F-11ed8242c40c | Q ≤ 0x7FF00000 | Original input pair, auxiliary datum A, auxiliary datum B, original input pair again | Return value ignored |
| F-6dafbb420982 | Q ≤ 0x3FF00000 or Q > 0x7FF00000 | Original first word, original second word | Returned as the wrapper’s result |
| F-77dd9f40f9b6 | 0x3FF00001 ≤ Q ≤ 0x7FF00000, after the auxiliary call | Integer 1 | Return value ignored; fallback result read afterward |

These identifiers describe call targets only. Their names do not imply mathematical operations, exception policies, validation guarantees, or memory-management behavior.

Exactly one of F-6dafbb420982 and F-77dd9f40f9b6 is reached on a normally progressing invocation. F-11ed8242c40c precedes that choice except in the upper classification region, where it is skipped.

## Acceptance examples

The following are conditional acceptance criteria derived from static evidence, not executed tests. Each assumes normally returning dependencies and valid global accesses.

### Zero classification

Given a first word of zero and a second word of 0x00000000:

- Q is zero.
- F-11ed8242c40c is called with the original input pair in both duplicated positions.
- F-6dafbb420982 is called afterward with zero and 0x00000000.
- The wrapper returns its result.
- The notification dependency and fallback-result read do not occur.

A second word of 0x80000000 selects the same path, but both dependencies receive 0x80000000 as the second word.

### Lower boundary

Given a first word of zero and a second word of 0x3FF00000:

- Q is exactly 0x3FF00000.
- The auxiliary call occurs.
- Delegation follows.

Changing only the first word to any nonzero 32-bit value sets Q to 0x3FF00001:

- The auxiliary call still occurs.
- F-77dd9f40f9b6 is then called with 1.
- The fallback result is read afterward and returned.
- F-6dafbb420982 is not called.

This boundary distinguishes bitwise OR from an interpretation that ignores the first word.

### Already-set least significant bit

Given a second word of 0x3FF00001, both a zero first word and a nonzero first word produce Q = 0x3FF00001.

Both select the notification-and-fallback path. The actual first-word value must still be forwarded to F-11ed8242c40c.

### Upper boundary

Given a first word of zero and a second word of 0x7FF00000:

- Q is exactly 0x7FF00000.
- The auxiliary call occurs.
- Notification and the subsequent fallback-result read occur.

Changing the first word to nonzero produces Q = 0x7FF00001:

- The auxiliary call is skipped.
- Neither auxiliary datum is read by the wrapper.
- Direct delegation occurs.

### Ignored auxiliary return

For Q in either region that invokes F-11ed8242c40c, changing only that dependency’s returned value must not change the wrapper’s selected branch.

This criterion does not require independence from the dependency’s side effects.

### Fallback timing

On the notification-and-fallback path, if F-77dd9f40f9b6 changes the fallback result before returning, the wrapper reads the resulting value afterward.

Likewise, a change made by F-11ed8242c40c may affect the returned fallback value if it remains in effect when that final read occurs.

## Uncertainties

### Inferred intent

The sign mask, exponent-like threshold, and first-word nonzero test are consistent with treating the second word as the high word of a binary64 encoding and the first as its low word.

Under that interpretation, the regions resemble:

- Magnitudes at most one: auxiliary call followed by delegation.
- Magnitudes greater than one, including infinities: auxiliary call followed by notification and a global result.
- NaN encodings: direct delegation.

This interpretation does not establish that the interface accepts a language-level floating-point value, that the result is floating-point, or that any dependency implements a particular mathematical function.

### documented_only

The following remain unresolved:

- Global datum values, declarations, storage widths, and mutability.
- The fallback result’s semantic meaning.
- Dependency implementations and complete side effects.
- Calling convention and physical argument or return registers.
- Memory byte order for any wider representation assembled from the inputs.
- Error-state behavior, exception behavior, and nonlocal control transfer.
- Resource ownership and lifetime rules inside dependencies.
- Concurrency and synchronization guarantees.

These unknowns do not change the established word-level dispatch contract.

## Evidence

The supplied static body establishes the input mask, nonzero-bit combination, subtraction, pre-call flag computation, short-circuit condition, dependency argument positions, ignored dependency returns, and final global-read placement.

The three classification regions follow arithmetically from those operations. They are derived behavior, not runtime observations.

The supplied material does not include dependency bodies, global declarations, or execution traces. No runtime tests have been performed.
