<!-- Trusted provenance: F-7d60babd66f2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7d60babd66f2

## Purpose

F-7d60babd66f2 returns an unsigned integer selected through repeated comparison with a caller-supplied unsigned candidate and, when required, calls to dependency F-2534ccb803dc.

The established selection rule is:

- If the unsigned value read through the supplied pointer is no greater than the current candidate, return a value read through that pointer.
- Otherwise, invoke F-2534ccb803dc with that pointer and the current candidate.
- If the dependency returns a nonzero status, return the current candidate.
- If it returns zero, advance the candidate by one using unsigned arithmetic and repeat.

This establishes a control-flow contract, not a domain-level meaning. The evidence does not establish whether the pointed-to value represents a limit, count, position, resource identifier, or something else. Likewise, a nonzero dependency status selects a return path; calling it “success” would require additional evidence.

## Interface

**Inputs**

1. A pointer to an unsigned integer object.
2. An unsigned integer used as the initial candidate.

**Output**

One unsigned integer, obtained either from the pointed-to object or from the current candidate.

The candidate is passed by value. Changes to it inside this function do not directly update the caller’s original candidate object.

The pointer is also passed by value. The function reads through it and supplies the same pointer value to F-2534ccb803dc. No pointer arithmetic or pointer reassignment is shown.

**Widths and representation**

The supplied source identifies the pointed-to object, candidate, and return value as `uint`, and the dependency result is received in an `int`. Definitions and target ABI information are absent. Consequently:

- The unsigned integer width is **documented_only**. Let its actual width be **W** bits.
- The candidate range is 0 through $2^W-1$.
- Candidate advancement wraps modulo $2^W$.
- The signed status width is **documented_only**. This function distinguishes only zero from nonzero, not positive from negative.

A 32-bit interpretation gives an unsigned range of 0 through 4,294,967,295, but that width is conditional rather than established by the excerpt alone.

**Memory preconditions**

The pointer must designate a suitably aligned, readable unsigned integer object whenever this function dereferences it. The required object extent and alignment depend on the unresolved type definition and ABI.

Additional access requirements imposed by F-2534ccb803dc are unknown. In particular, the visible wrapper does not establish whether the dependency requires writable storage.

There is no null-pointer check, bounds check, lifetime check, or explicit recovery from invalid memory access.

## Behavior

### Initial comparison

The current candidate initially equals the unsigned input value.

Before any dependency call, the function reads the object designated by the pointer and compares that value with the current candidate. The comparison is unsigned and includes equality.

If the read value is less than or equal to the candidate, the function returns through the pointed-value path. F-2534ccb803dc is not called on that iteration.

The comparison and return contain separate source-level dereference expressions. The return expression therefore obtains the result through the pointer rather than returning a saved comparison value. For an ordinary valid object that does not change between these expressions, both reads yield the same value.

This source structure does not establish two separately observable hardware loads. Nor does it establish support for concurrent mutation, volatile access, or memory-mapped I/O.

### Dependency call

If the comparison value is strictly greater than the current candidate, the function calls F-2534ccb803dc with:

- The original pointer value.
- The current unsigned candidate value.

The dependency executes before this function decides whether to advance the candidate. Its result is interpreted through the signed status value received by the wrapper.

If that status is nonzero, the function returns the candidate used for that call. It does not increment the candidate or perform another explicit pointed-object read before returning.

Consequently, dependency-induced changes to the pointed-to object do not replace the return value on this branch. Such changes may still be visible to the caller.

### Zero-status continuation

If the dependency returns zero, the function increments the current candidate by one and repeats the comparison.

The pointed-to object is read again on the next iteration. Its earlier value is not retained as a fixed bound. Any dependency-induced modification can therefore affect the next comparison, provided the object remains valid to read.

There is no explicit attempt counter, retry limit, cancellation check, or special handling of the maximum unsigned candidate.

### Arithmetic boundary and termination

Candidate advancement is unsigned. Advancing $2^W-1$ would produce zero, not a signed-overflow condition.

However, under ordinary defined execution, the function does not reach that increment with the maximum candidate. Every value representable by the pointed-to unsigned type is less than or equal to $2^W-1$. At that candidate, the comparison therefore selects the pointed-value return path before a dependency call or increment.

Accordingly, if every invoked dependency returns normally and every required memory read remains valid, the candidate progression cannot continue indefinitely. From initial candidate $c$, there can be at most:

$$
(2^W-1)-c
$$

dependency calls. Reaching the maximum candidate selects the direct return path.

This bound does not guarantee wall-clock completion. A dependency can fail to return, terminate execution, or invalidate required memory. Those behaviors are outside the visible wrapper’s control.

## Side effects

The function contains no explicit write through the supplied pointer and no explicit global-variable write.

Its visible state changes are limited to its own candidate and stored dependency status. Nevertheless, it is not established as side-effect-free because it invokes F-2534ccb803dc.

The dependency may modify the pointed-to object or other accessible state. The supplied evidence does not establish whether it allocates memory, releases memory, performs I/O, acquires resources, changes globals, or invokes other code. None of those effects may be assumed absent.

**Ordering**

The source establishes this order for each iteration:

1. Evaluate the pointed-object comparison.
2. If the comparison selects the direct return path, evaluate the pointed-object return expression.
3. Otherwise, complete the dependency call before inspecting its returned status.
4. On nonzero status, return the current candidate.
5. On zero status, advance the candidate before the next comparison.

There is no explicit read of the pointed-to object between the dependency’s return and the nonzero-status return.

**Ownership**

The wrapper contains no explicit allocation, deallocation, ownership transfer, or lifetime extension. Supplying the pointer to the dependency does not, by itself, establish any ownership contract.

If the dependency ends the object’s lifetime, a subsequent zero-status iteration can make the next dereference invalid. The wrapper does not protect against that condition. A nonzero-status return requires no further explicit dereference by this wrapper, but the caller’s remaining ownership obligations are still unknown.

**Errors**

No dedicated error result is defined. Every output is an unsigned integer selected by the ordinary return rules. The dependency status is not returned directly.

## Dependencies

### F-2534ccb803dc

**Invocation condition:** The latest comparison read is strictly greater than the current candidate.

**Arguments:** The original unsigned-object pointer and the current unsigned candidate, in that order.

**Consumed result:** A signed integer status, interpreted only as zero or nonzero.

**Control effect:**

- Zero requests continuation with the next candidate.
- Nonzero selects immediate return of the candidate supplied to that invocation.

These descriptions concern the wrapper’s interpretation, not the dependency’s intended semantics.

The dependency’s implementation, declared interface, memory requirements, ownership behavior, global effects, and termination properties are unavailable. Its full result range and any ABI-level conversion into the wrapper’s signed status storage are also **documented_only**.

## Acceptance examples

These examples are conditional behavioral checks derived from static evidence. They have not been executed. Small numeric values fit any unsigned width capable of representing them.

### Initial value below the candidate

Given a stable pointed-to value of 4 and initial candidate 7:

- The initial comparison selects the direct return path.
- The result is 4.
- There are no dependency calls.

The function does not necessarily return its input candidate.

### Equality at entry

Given a stable pointed-to value of 7 and initial candidate 7:

- Equality selects the direct return path.
- The result is 7.
- There are no dependency calls.

The comparison is inclusive, not strictly less-than.

### Nonzero dependency status

Given a pointed-to value of 10 and initial candidate 3, suppose F-2534ccb803dc returns a representable nonzero status on its first invocation.

The dependency receives the original pointer and candidate 3. The function returns 3 without another explicit pointed-object read. A negative nonzero status has the same branch effect as a positive nonzero status.

### Continuation until equality

Given a pointed-to value that remains 5 and initial candidate 3, suppose the dependency returns zero on each invocation.

The dependency is called with candidates 3 and 4, in that order. The next comparison occurs at candidate 5 and selects the direct return path. The result is 5; there is no dependency call at candidate 5.

### Dependency lowers the pointed-to value

Given initial pointed-to value 10 and candidate 3, suppose the dependency changes the object to 2 and returns zero.

The candidate advances to 4. The next comparison reads 2 and selects the direct return path, returning 2. Thus, the result need not be at least the initial candidate.

### Mutation with nonzero status

Given initial pointed-to value 10 and candidate 3, suppose the dependency changes the object to 1 and returns nonzero.

The result is 3, not 1. The object remains subject to whatever mutation the dependency actually performed; the wrapper performs no restoration.

### Maximum candidate

For an unsigned width of W bits, an initial candidate of $2^W-1$ selects the direct return path for every valid pointed-to unsigned value. No dependency call or candidate increment occurs.

Conditionally, for a 32-bit unsigned type, candidate 4,294,967,295 with a stable pointed-to value of 12 returns 12 without invoking the dependency.

## Uncertainties

- Exact unsigned and signed widths, object layout, alignment, and calling convention are **documented_only**.
- Domain meaning and intended algorithm are unknown.
- Dependency mutation, ownership, global-state effects, and failure behavior are unknown.
- No concurrency or asynchronous-update contract is established.
- Separate source dereference expressions do not prove separate observable machine reads.
- No special handling for invalid pointers or dependency-induced lifetime violations is shown.
- No runtime tests have been performed.

## Evidence

Established static evidence consists of the supplied function body: an unsigned pointer input, an unsigned candidate input and result, an inclusive unsigned comparison, a pointed-object return expression, a conditional dependency invocation, a zero/nonzero status branch, and unsigned candidate advancement before repetition.

The call-count bound and maximum-candidate behavior follow from those operations over a finite unsigned range, subject to valid reads and normally returning dependency calls.

Any characterization as a search, allocation routine, bounded retry mechanism, or resource-selection operation would be inferred intent, not established behavior. No such domain interpretation is required by this contract.
