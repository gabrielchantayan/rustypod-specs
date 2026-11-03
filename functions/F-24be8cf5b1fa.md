<!-- Trusted provenance: F-24be8cf5b1fa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-24be8cf5b1fa

## Purpose

F-24be8cf5b1fa produces an unsigned 32-bit result by selecting between a presence-like decision and a repeated comparison of values supplied by dependencies.

The established behavior has three forms:

- When an initial probe returns zero, return whether a second probe returns nonzero.
- When the initial probe returns nonzero but the second probe returns zero, return the unsigned value 4,294,967,295.
- When both probes return nonzero, repeatedly obtain a pair of signed integer results. Return their difference at the first unequal pair, or return zero when both results in a pair are zero.

**Inferred intent:** This resembles an asymmetric comparison operation: the initial probes distinguish absent and present cases, while the loop compares successive values until a difference or a shared terminator appears. The evidence does not establish that the inputs are strings, containers, iterators, or any other particular data type.

This contract describes the supplied static representation. It does not assert runtime observations or undocumented dependency semantics.

## Interface

### Inputs

The function takes two inputs:

1. **Record address.** Represented as a signed integer used for address calculation. The function reads a four-byte field beginning four bytes after this address. That field’s raw value becomes the argument to F-d8cbf9aa79a9.
2. **Opaque four-byte argument.** Passed by value to F-1cb585febd84. The supplied representation does not establish whether these bits identify an object, encode a scalar, or have some other meaning.

The function does not directly pass the record address to a dependency. It passes only the four-byte field read from the record.

### Result

The result is represented as an unsigned 32-bit integer. Normal return paths yield:

- 0;
- 1;
- 4,294,967,295, equivalently hexadecimal FFFFFFFF; or
- a value derived from subtracting the second loop result from the first.

The result is not restricted to a Boolean or to the conventional comparison values −1, 0, and 1. In particular, the unequal-pair path does not normalize the difference.

### Widths and interpretation

The supplied representation uses:

- A four-byte load from the record.
- A four-byte opaque second argument.
- Signed integer interpretations for both probe results and both loop results.
- An unsigned integer return type.

The signed and unsigned integer behavior described below uses the represented 32-bit model. The evidence does not independently establish the target ABI, pointer width, alignment rules, or machine-level overflow behavior.

Zero tests depend only on whether all result bits represent zero. The sign of a nonzero probe result does not change branch selection.

### Preconditions

For ordinary completion through the visible control flow:

- The four-byte field at record offset +4 must be readable when the initial load occurs.
- The record-address calculation and load must be valid under the actual execution environment.
- Each dependency must accept the argument supplied on its selected path.
- Dependencies must return normally for the function to continue to the next visible step.
- Any additional dependency-specific memory, state, or lifetime requirements must hold.

No null check, bounds check, alignment check, or error conversion precedes the record-field load.

## Behavior

### Initial record read and first probe

The function first reads exactly one four-byte field from the record, at byte offset +4. It passes that field’s value to F-d8cbf9aa79a9.

This is the only direct record-field read visible in the supplied body. There is no visible reload of that field later, even if a dependency changes the record.

The returned signed integer selects one of two paths:

- Zero selects the simple decision path.
- Any nonzero value selects the second-probe and possible comparison path.

Only the zero/nonzero classification of this result matters. Its magnitude and sign are otherwise unused.

### Simple decision path

If F-d8cbf9aa79a9 returns zero, the function calls F-1cb585febd84 with the opaque four-byte second input.

It then returns:

- 0 if that call returns zero.
- 1 if that call returns any nonzero signed integer.

There are no calls to either loop dependency on this path.

The result is normalized: a negative dependency result and a positive dependency result both produce 1.

### Second-probe rejection path

If F-d8cbf9aa79a9 returns nonzero, the function also calls F-1cb585febd84 with the opaque second input.

If F-1cb585febd84 returns zero, the function immediately returns 4,294,967,295.

No loop dependency is called on this path.

Thus, subject to dependencies returning normally, F-1cb585febd84 is called exactly once per invocation regardless of the first probe’s result. Its outcome has different meanings on the two paths: Boolean normalization after a zero first probe, but either rejection or loop entry after a nonzero first probe.

### Repeated comparison path

The loop is entered only when both probe results are nonzero.

Each iteration performs the following externally relevant sequence:

1. Call F-d3922c7fe0ec with the address of one local stack location.
2. Call F-5afffd4d8a24 and retain its signed integer result as the first comparison value.
3. Call F-d3922c7fe0ec with the address of a distinct local stack location.
4. Call F-5afffd4d8a24 and retain its signed integer result as the second comparison value.
5. Compare the two retained values.

The calls are ordered. In particular, the second call to F-d3922c7fe0ec occurs after the first value has been obtained, not before both value-producing calls.

If the values differ, the function returns the first value minus the second, converted to the unsigned result type.

If the values are equal and zero, the function returns zero.

If the values are equal and nonzero, the function starts another iteration using the same two local address expressions.

Consequences of this order include:

- A zero first value does not short-circuit acquisition of the second value.
- Every completed iteration obtains both values before deciding whether to return.
- An unequal pair is handled before the shared-zero termination check.
- Equal negative values continue the loop just as equal positive values do.
- There is no visible iteration limit.
- An indefinitely continuing sequence of equal, nonzero pairs produces no normal return.

### Difference arithmetic

On an unequal pair, the subtraction uses the signed integer results in first-minus-second order. The resulting value is then returned as unsigned.

Where the signed subtraction is representable in 32 bits:

- A positive difference is returned unchanged.
- A negative difference is converted to its unsigned 32-bit value, mathematically adding 4,294,967,296.
- The magnitude of the difference is preserved rather than reduced to a sign indicator.

For example, a representable signed difference of −3 returns 4,294,967,293.

**documented_only:** The supplied representation expresses signed subtraction before unsigned conversion. If the mathematical difference falls outside the signed 32-bit range, the evidence does not establish a portable result. Unsigned return conversion alone does not prove that the preceding subtraction wraps. A machine-level wrapping rule must not be assumed without additional evidence.

## Side effects

### Direct memory access

The visible body directly reads the four-byte record field at offset +4.

It does not visibly write to the record or explicitly dereference the opaque second input. It also does not visibly allocate, free, acquire, or release caller-owned resources.

These observations do not establish that the inputs or reachable memory remain unchanged: dependencies may have additional effects.

### Local stack addresses

The loop passes two distinct local stack addresses to F-d3922c7fe0ec. Their represented locations are four bytes apart.

That spacing does not establish:

- The size of either object.
- That either location contains a valid four-byte object.
- Their initial contents.
- Their initialization history.
- Whether the dependency reads them, writes them, or merely uses their addresses.
- Whether adjacent storage is accessible to the dependency.

No explicit initialization of these locations appears before their first use in the supplied body. The function does not visibly consume their contents afterward.

The two argument locations are not interchangeable in the established call trace: the first is always supplied before the first value-producing call, and the second before the second value-producing call.

### Dependency-mediated state

The function makes no visible direct global-state accesses. Nevertheless, every dependency may read or modify state not shown here.

In particular, it is not established that:

- F-d3922c7fe0ec preserves global state.
- F-5afffd4d8a24 is independent of preceding calls.
- The first and second value-producing calls use separate underlying objects.
- Either probe is side-effect-free.
- Any dependency preserves caller-owned memory reachable through its arguments.

The ordering of calls is therefore part of the contract, even where argument values are not visibly used afterward.

### Ownership and lifetime

The body shows no explicit ownership transfer or cleanup operation.

Passing local addresses establishes that those addresses are supplied during the calls. It does not establish whether F-d3922c7fe0ec retains them. Those locations cannot be assumed to remain valid after this invocation returns.

No claim is made that any dependency acquires or releases memory.

### Errors and abnormal execution

There is no visible recovery path for an invalid record read, dependency fault, nonlocal exit, or other abnormal execution.

The all-ones result is an explicit ordinary return on one branch. It is not established as a general error indicator and is not unique to that branch: a representable difference of −1 also converts to the same unsigned value.

## Dependencies

| Opaque function ID | Visible interaction | Established use of result | Unknowns |
|---|---|---|---|
| F-d8cbf9aa79a9 | Called first with the four-byte record field | Zero/nonzero selects the top-level path | Argument meaning, side effects, ownership, failure behavior |
| F-1cb585febd84 | Called once with the opaque four-byte second input, after the first probe returns | Zero/nonzero produces a Boolean result or selects rejection versus loop entry | Input interpretation, side effects, ownership, failure behavior |
| F-d3922c7fe0ec | Called twice per completed loop iteration, each time with a different local address | No returned value is visibly consumed | Reads or writes through the address, retained pointers, state changes, return signature |
| F-5afffd4d8a24 | Called twice per completed loop iteration, with no explicit arguments shown | Signed integer results are compared and possibly subtracted | Value domain, state source, termination guarantees, side effects |

The lack of explicit arguments to F-5afffd4d8a24 does not establish statelessness or independence from previous calls.

## Acceptance examples

These examples are conditional static expectations, not executed tests. They assume valid accesses, normal dependency returns, and the represented 32-bit integer model.

### Probe combinations

| First probe | Second probe | Expected unsigned return | Loop entered |
|---:|---:|---:|---|
| 0 | 0 | 0 | No |
| 0 | 7 | 1 | No |
| 0 | −7 | 1 | No |
| 4 | 0 | 4,294,967,295 | No |
| −4 | 2 | Determined by loop | Yes |

In each row, the second probe follows the first. Its call is not omitted when the first probe returns zero.

### Shared terminator

Given two nonzero probe results and a first loop pair of 0 and 0:

- Both local-address calls occur.
- Both value-producing calls occur.
- The function returns 0 after that pair.
- No second iteration occurs.

### Immediate unequal pair

Given two nonzero probe results and a first loop pair of 9 and 4, the function returns 5.

For a pair of 4 and 9, it returns 4,294,967,291, corresponding to the representable signed difference −5 converted to unsigned.

### Zero on only one side

For a pair of 0 and 3, the function returns 4,294,967,293.

For a pair of 3 and 0, it returns 3.

Neither case is shared-zero termination. Both use the unequal-pair path.

### Equal nonzero continuation

Given successive pairs of −2 and −2, followed by 6 and 1:

- The first pair causes another iteration.
- The second pair causes return of 5.
- There are four calls to F-d3922c7fe0ec and four calls to F-5afffd4d8a24.

### Overlapping return meanings

A nonzero first probe followed by a zero second probe returns 4,294,967,295.

Entering the loop and obtaining 8 followed by 9 also returns 4,294,967,295.

A consumer cannot distinguish those paths from the return value alone.

### Arithmetic boundary

A pair of −2,147,483,648 and 0 has a representable signed difference and converts to unsigned 2,147,483,648.

A pair of 2,147,483,647 and −1 requires a mathematical signed difference of 2,147,483,648. The result for that case is **documented_only** because signed-overflow behavior is not established.

## Uncertainties

The following remain **documented_only**:

- Concrete input types, record extent, and field meaning.
- Actual pointer width and ABI requirements.
- Alignment and address-arithmetic behavior.
- Semantic meaning of either probe.
- Contents, extent, and initialization of the two local stack locations.
- State relationships between the loop dependencies.
- Memory ownership and retention behavior of all dependencies.
- Dependency error conventions and non-returning behavior.
- Whether zero represents an end marker, a valid data value, or both.
- Termination guarantees for equal nonzero pairs.
- Overflow behavior for out-of-range signed subtraction.
- Thread safety, reentrancy, and concurrent mutation behavior.

These gaps do not change the visible branch structure, call order, or representable-arithmetic results.

## Evidence

The supplied static body establishes the record-field load, dependency arguments, call order, zero/nonzero decisions, loop continuation rule, and explicit return expressions.

The comparison-like purpose is an inference, not an established data-model contract. No dependency implementation or independent ABI evidence was supplied.

No runtime tests have been performed. The acceptance examples describe conditional consequences of the static evidence, not observed execution.
