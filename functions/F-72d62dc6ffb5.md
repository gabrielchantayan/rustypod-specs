<!-- Trusted provenance: F-72d62dc6ffb5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-72d62dc6ffb5

## Purpose

F-72d62dc6ffb5 converts a 32-bit input value into a 32-bit result according to a fixed numeric mapping. Inputs from 100 through 121, inclusive, produce consecutive results from 1 through 22. Every other input produces zero.

The established behavior is a total mapping over 32-bit input patterns. It does not require a particular interpretation of the input as an application-level identifier, enumeration member, status, or character. Those meanings are not established by the supplied evidence.

An inferred intent is to translate a contiguous external value range into one-based ordinal values, reserving zero for inputs outside that range. This interpretation is not necessary to implement or use the observable mapping, and it does not establish that zero is an error indicator.

## Interface

**Input:** One 32-bit value, passed by value.

**Output:** One 32-bit value, returned by value.

The supplied representation establishes the widths but does not establish a signed or unsigned source-language type. The contract therefore treats the input as a 32-bit bit pattern. All recognized values are positive integers representable identically under ordinary signed and unsigned 32-bit interpretations. No input whose highest bit is set belongs to the recognized range, so the signedness ambiguity does not change the required mapping.

The possible results are exactly the integers 0 through 22. A conforming result occupies the 32-bit return value; it is not merely a requirement on the lowest byte or another truncated portion.

The interface exposes no output pointer, buffer, array, structure, length field, or ownership-bearing handle. There is consequently no caller-provided memory layout, alignment requirement, capacity requirement, or null-pointer condition to specify.

No special input-value precondition is established. Every 32-bit input has a defined result under the supplied behavior. Requirements for invoking the function through a particular machine-level calling convention remain outside the available evidence.

## Behavior

### Recognized range

For an input in the inclusive range 100 through 121, the result is the input’s numeric value minus 99.

This relationship specifies the mapping rather than prescribing an implementation strategy. An implementation need not perform subtraction or use any particular branching mechanism, provided it returns the required 32-bit result for every input.

The complete recognized mapping is:

| Input, decimal | Result, decimal |
|---:|---:|
| 100 | 1 |
| 101 | 2 |
| 102 | 3 |
| 103 | 4 |
| 104 | 5 |
| 105 | 6 |
| 106 | 7 |
| 107 | 8 |
| 108 | 9 |
| 109 | 10 |
| 110 | 11 |
| 111 | 12 |
| 112 | 13 |
| 113 | 14 |
| 114 | 15 |
| 115 | 16 |
| 116 | 17 |
| 117 | 18 |
| 118 | 19 |
| 119 | 20 |
| 120 | 21 |
| 121 | 22 |

The endpoints are included. There are no gaps, aliases, or duplicate results within this recognized range.

### All other inputs

Any input not equal to one of the recognized values returns zero. In particular:

- Values below 100 do not produce negative results or wraparound-derived results.
- Values above 121 do not continue the consecutive sequence.
- Large 32-bit values do not match merely because their lowest byte resembles a recognized value.
- Bit patterns conventionally interpreted as negative signed integers return zero.

The default result is ordinary return behavior. The supplied evidence does not establish an exception, diagnostic, retry, or separate error channel for an unrecognized input.

### Width and arithmetic constraints

The recognized-range relationship involves values no larger than 121 and results no larger than 22. It therefore requires no overflow or underflow for recognized inputs under either signed or unsigned 32-bit arithmetic.

The subtraction relationship must not be extended to inputs outside the recognized range. For example, input 99 returns zero because it is unrecognized; input 98 also returns zero, not a negative or wrapped result. Input 122 returns zero, not 23.

Recognition concerns the complete 32-bit input value. Narrowing the input before classification would change the contract for some inputs and is not justified by the supplied evidence.

### State and ordering

The supplied behavior contains no externally visible intermediate write. The only specified output is the returned value.

There are no exposed memory reads or writes whose relative order must be preserved. The input is available by value, and the behavior does not dereference it or use it as an address.

No persistent state participates in the mapping. At the supplied behavioral level, successive invocations are independent: an earlier call does not alter which result a later input receives. This is a static conclusion about the represented function, not a runtime concurrency or platform certification.

## Side effects

The supplied representation shows no:

- Global or externally supplied memory access.
- Allocation, release, or transfer of memory ownership.
- Mutation of caller-owned storage.
- Input/output operation.
- Callback or invocation of another function.
- Explicit modification of an error-reporting facility.

No resource is returned for the caller to release, and no caller-owned resource is consumed.

These statements describe the visible function behavior. They do not assert a particular treatment of processor flags, scratch registers, stack storage, instrumentation, or other machine-level details absent from the evidence. Likewise, they do not establish platform-specific exception or asynchronous-interruption behavior.

For unrecognized inputs, the observable treatment remains a zero return. There is no established distinction between different categories of unrecognized input.

## Dependencies

No function dependencies are visible. F-72d62dc6ffb5 neither delegates classification nor obtains its result from an opaque callee.

The mapping does not depend on a supplied table, global configuration, locale, environment value, or previously initialized object in the available representation.

Because no callees are present, this contract makes no assumptions about another function preserving globals, allocating memory, retaining arguments, or releasing resources.

## Acceptance examples

The following are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

| Condition | Required observation |
|---|---|
| The input is 100. | The returned 32-bit value is 1. |
| The input is 110. | The returned 32-bit value is 11. |
| The input is 121. | The returned 32-bit value is 22. |
| The input is 99. | The returned value is zero; recognition does not extend below the lower boundary. |
| The input is 122. | The returned value is zero; the consecutive mapping stops at 121. |
| The input is zero. | The returned value is zero. |
| The input bit pattern is hexadecimal FFFFFFFF. | The returned value is zero, regardless of whether the caller interprets that pattern as signed or unsigned. |
| The input bit pattern is hexadecimal 80000000. | The returned value is zero. |
| The input is 356, whose lowest eight bits equal 100. | The returned value is zero; full-width recognition is required. |

Additional relational acceptance criteria are:

- For any recognized input below 121, increasing the input by one increases the result by one.
- A nonzero result occurs if and only if the input belongs to the inclusive range 100 through 121.
- Each nonzero result identifies exactly one recognized input.
- Calling with 100, then an unrecognized value, then 100 again must yield 1, zero, and 1 respectively, without any specified persistent state change.

These criteria constrain values and observable behavior, not the choice of internal dispatch mechanism or instruction sequence.

## Uncertainties

**documented_only — Application meaning.** The evidence does not identify what the input numbers represent or what consumers do with the returned numbers.

**documented_only — Zero semantics.** Zero is established as the default result. Whether callers interpret it as invalid, absent, unknown, or another application state is not established.

**documented_only — Declared signedness and calling convention.** Both values are represented as 32-bit quantities, but the original public type declarations and machine-level invocation requirements are unavailable.

**documented_only — Timing and machine implementation.** Execution time, instruction selection, dispatch strategy, register use, and constant-time properties are not established.

**documented_only — Runtime confirmation.** No runtime tests have been performed. The acceptance examples specify expected behavior rather than observed executions.

## Evidence

The supplied static representation establishes one by-value 32-bit input, one 32-bit return value, explicit mappings for every integer from 100 through 121, and a zero result for all remaining inputs. It shows no additional calls or explicit memory effects.

The compact recognized-range relationship follows from those mappings. The suggested one-based translation purpose is inferred intent, not an additional requirement.

This contract relies only on the supplied static evidence. It makes no claim of independent implementation inspection, execution, or runtime validation.
