<!-- Trusted provenance: F-c6ebe8e80d24; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c6ebe8e80d24

## Purpose

F-c6ebe8e80d24 obtains an unsigned 32-bit value through a dependency and returns that value divided by 1000, with both quotient and remainder packed into a single 64-bit result.

For the value \(N\) returned by F-d7be9cfb4000, normal completion establishes:

- The low 32 bits contain \(Q=\lfloor N/1000\rfloor\).
- The high 32 bits contain \(R=N-1000Q\).
- \(N=1000Q+R\).
- \(0\le R<1000\).

The arithmetic uses magnitude-dependent paths, but those paths implement the same unsigned quotient-and-remainder operation.

This interpretation is established by static analysis of the supplied body, conditional on the recovered 32-bit operations representing the executable accurately. A possible application-level purpose, such as converting a counter into larger units, is not established.

## Interface

### Explicit and implicit inputs

The recovered interface has no explicit formal arguments. It nevertheless depends on an implicit 32-bit register value identified as register r3 in the supplied representation. That value is supplied as the first argument to F-d7be9cfb4000.

The origin and intended meaning of this register value are unknown. The body does not establish a conventional, self-contained, zero-input API.

The call sequence also supplies:

- An address in the current stack frame to F-4085480a3d85.
- The constant 1000 as the second argument to F-d7be9cfb4000.

The first dependency is called before the second. The evidence does not establish whether the implicit register input is independent of the first call, produced by it, or affected by register-state reconstruction. Callers must not assume that a conventional argument-free invocation supplies all necessary state.

### Normal output

The result is a 64-bit packed integer:

| Bit range | Width | Meaning |
|---|---:|---|
| 63–32 | 32 bits | Unsigned remainder after division by 1000 |
| 31–0 | 32 bits | Unsigned quotient after division by 1000 |

Equivalently, the numerical result is \(R\cdot2^{32}+Q\).

This defines bit positions, not an in-memory byte order. No result buffer is written by the visible arithmetic.

For the complete unsigned 32-bit input domain:

- \(0\le N\le4,294,967,295\).
- \(0\le Q\le4,294,967\).
- \(0\le R\le999\).

There is no separate success flag or ordinary error field in the normal result.

### Preconditions

Normal completion requires both executed dependencies to return normally and to satisfy whatever memory and calling-convention requirements their implementations impose.

The supplied body alone does not establish those requirements. In particular, it does not establish:

- The required initial contents or accessible extent of the stack region passed to F-4085480a3d85.
- The validity constraints on the implicit register input.
- Whether either executed dependency can fail, trap, block, or terminate execution.

Once F-d7be9cfb4000 returns an unsigned 32-bit value, the visible arithmetic accepts its entire range without an additional input-validity check.

## Behavior

### Observable call order

The function first calls F-4085480a3d85 with an address in its own stack frame.

If that call returns, it calls F-d7be9cfb4000 with the implicit register input and the fixed second argument 1000.

The 32-bit return value from the second dependency becomes \(N\), the dividend used by the remaining normal-path arithmetic. The visible body contains no subsequent dependency call on any reachable path under the established unsigned 32-bit bounds.

The first call’s return value is not explicitly consumed as an ordinary scalar result. This does not establish that the call is irrelevant: its memory effects, register effects, or other side effects may matter.

### Unsigned arithmetic model

The dividend, shifted comparisons, subtraction operands, and result components are 32-bit quantities. Right shifts of the dividend are logical unsigned shifts.

All magnitude decisions are unsigned. A value with its most significant bit set is therefore treated as a large nonnegative dividend, not as a negative integer.

The recovered representation includes a signed-looking accumulator initialized with negative constants in the larger-input paths. Its role is bit-level quotient construction and loop control. Its intermediate sign is not an error indication and does not make the output quotient signed.

The contract interprets these operations at their recovered machine widths. It does not rely on treating overflowing signed expressions in the displayed representation as portable language-level arithmetic.

### Small and medium dividends

For \(N<16,000\), the arithmetic determines the quotient using the contribution weights 8, 4, 2, and 1. Each accepted contribution subtracts the corresponding multiple of 1000 from the current residual.

For \(16,000\le N<256,000\), it first considers contribution weights 128, 64, 32, and 16, in that order, then considers 8, 4, 2, and 1.

Each decision uses the residual left by all earlier accepted contributions. The decisions are therefore ordered; they are not independent comparisons against the original dividend.

The final comparison accepts the last quotient unit exactly when the residual is at least 1000. Thus a residual of 999 remains unchanged, whereas a residual of 1000 becomes zero and adds one to the quotient.

### Larger dividends

For \(N\ge256,000\), the function selects a scaled divisor:

| Dividend range | Initial scaled divisor |
|---|---:|
| 256,000 through 16,383,999 | 64,000 |
| 16,384,000 through 1,048,575,999 | 4,096,000 |
| 1,048,576,000 through 4,294,967,295 | 262,144,000 |

These values are respectively \(1000\cdot64\), \(1000\cdot4096\), and \(1000\cdot262144\).

At each scale, the arithmetic considers six descending multiples of the scaled divisor: 128, 64, 32, 16, 8, and 4. An accepted comparison subtracts that multiple from the residual and contributes the corresponding quotient bit.

If another group is required, the scaled divisor is reduced by a factor of 64 before the next group. The final group uses a scaled divisor of 1000. Two final decisions, for multiples 2 and 1 of that divisor, complete the quotient and remainder.

The three ranges require respectively two, three, and four six-decision groups. The accumulator’s carry behavior controls this fixed, range-dependent progression.

No accepted subtraction underflows the unsigned residual. The comparisons establish that the relevant product fits within the current residual before it is subtracted. This matters because some products would not fit in 32 bits if evaluated indiscriminately; their subtraction is conditional on comparisons that exclude those cases.

### Termination and exceptional-looking paths

For every unsigned 32-bit value returned by F-d7be9cfb4000, the arithmetic reaches a normal packed return after a bounded number of decisions.

The supplied body also contains logic for a further scale selection, a zero-divisor check, a call to F-dd3f3ec55291, and a software-interrupt path. Those paths are not reachable under the established widths:

- A 32-bit unsigned value shifted right by eight is at most 16,777,215.
- The additional scale-selection condition requires that shifted value to be at least 262,144,000.
- Every reachable selected divisor is nonzero.
- Reductions by 64 reach 1000 before the final arithmetic.

Consequently, F-dd3f3ec55291 is not part of normal execution for any representable \(N\).

As a documented-only structural detail, the unreachable zero-divisor branch would call F-dd3f3ec55291 with two arguments both equal to 2. A nonzero low 32-bit return would lead to a software interrupt with immediate value 0x123456 and, if execution continued afterward, an indefinite loop. A zero low word would lead to a return retaining only the dependency result’s upper 32 bits. None of this constitutes a reachable error API for the established input domain.

## Side effects

### Memory access and ownership

The visible body passes a stack-frame address to F-4085480a3d85. The supplied representation places that address eight bytes below its stack reference, but this does not establish the size, alignment, layout, or initialized contents of a complete object.

The address is supplied before F-d7be9cfb4000 is called. Any memory effects of the first dependency therefore precede those of the second.

The arithmetic following the second call has no explicit pointer dereference, heap allocation, heap release, global-memory access, or output-buffer write.

These observations do not prove the function is globally side-effect-free. Either dependency may access global state or memory not shown in the wrapper.

No ownership transfer is established. In particular:

- There is no visible allocation whose ownership can be assigned to the caller.
- There is no visible release operation.
- The stack address does not establish a persistent object or permission to retain the pointer after this invocation.
- The dependency’s actual reads, writes, retention behavior, and ownership requirements remain unknown.

The wrapper does not visibly reread the addressed stack region after the first call. This does not exclude indirect significance through register state, globals, or dependency behavior.

### State changes and failures

The function’s directly established normal result is the packed quotient and remainder. Dependency side effects occur before that result is produced.

There is no visible rollback if a dependency changes state and a later operation fails to return. There is also no visible translation of dependency failures into a status value.

No guarantee is established that either dependency preserves globals, preserves unrelated registers beyond its actual calling convention, or leaves memory ownership unchanged.

## Dependencies

| Opaque ID | Established interaction | Unestablished behavior |
|---|---|---|
| F-4085480a3d85 | Called first with an address in the current stack frame; no explicit scalar return is subsequently consumed. | Purpose, memory layout, access extent, output data, register effects, global effects, ownership behavior, and failure behavior. |
| F-d7be9cfb4000 | Called second with an implicit 32-bit register input and 1000; its unsigned 32-bit return is the dividend. | Meaning of its inputs, relationship between input and return, global effects, ownership behavior, and failure behavior. |
| F-dd3f3ec55291 | Referenced by a zero-divisor branch with arguments 2 and 2. | Its semantics are unknown; its call is unreachable under the established 32-bit arithmetic bounds. |

The fixed argument 1000 to F-d7be9cfb4000 must not be confused with proof of that dependency’s operation. The division specified here is established by the arithmetic performed on its returned value.

## Acceptance examples

These examples are conditional static acceptance criteria, not recorded test results. Each assumes that the two executed dependencies return normally and that F-d7be9cfb4000 returns the listed \(N\).

| \(N\) | Low 32 bits: quotient | High 32 bits: remainder | Boundary or behavior |
|---:|---:|---:|---|
| 0 | 0 | 0 | Zero dividend |
| 999 | 0 | 999 | Largest dividend below one unit |
| 1000 | 1 | 0 | Exact first unit |
| 1001 | 1 | 1 | Residual retained |
| 15,999 | 15 | 999 | Last value on the smallest path |
| 16,000 | 16 | 0 | First expanded small-path value |
| 255,999 | 255 | 999 | Last value before scaled division |
| 256,000 | 256 | 0 | First scaled-path value |
| 16,383,999 | 16,383 | 999 | Last value using initial scale 64,000 |
| 16,384,000 | 16,384 | 0 | First value using initial scale 4,096,000 |
| 1,048,575,999 | 1,048,575 | 999 | Last value before the highest reachable scale |
| 1,048,576,000 | 1,048,576 | 0 | First value using initial scale 262,144,000 |
| 4,294,967,295 | 4,294,967 | 295 | Maximum unsigned 32-bit dividend |

Additional conditional criteria:

- For \(N=1001\), the numerical 64-bit return is 4,294,967,297: remainder 1 occupies the upper word and quotient 1 the lower word.
- An input above 2,147,483,647 remains an unsigned dividend; it must not be rejected or divided as a negative number.
- On every normal completion, the remainder is below 1000 and reconstructing \(1000Q+R\) yields the dependency-returned dividend exactly.
- For every representable dividend, the visible call sequence contains F-4085480a3d85 followed by F-d7be9cfb4000, without a subsequent call to F-dd3f3ec55291.

## Uncertainties

The following remain documented_only:

- The intended external API and the provenance of the implicit register input.
- Whether the first dependency supplies data or state needed by the second.
- The precise stack-object layout and permitted access extent.
- All dependency-internal memory effects, ownership rules, global changes, and failure modes.
- The platform’s concrete return-register assignment and any memory serialization byte order.
- The reason unreachable exceptional logic appears in the supplied representation.

It is inferred, not established, that the function participates in a unit conversion or counter-reading operation. No such interpretation is required for the arithmetic contract.

## Evidence

The supplied body provides static evidence for:

- The order and arguments of the two reachable dependency calls.
- The unsigned 32-bit dividend.
- The magnitude thresholds and scaled-divisor progression.
- Conditional subtraction and quotient-bit accumulation.
- Packing the residual into the upper 32 bits and quotient into the lower 32 bits.
- The bounds that make the additional scale and zero-divisor branches unreachable.

The division identity, output bounds, iteration counts, and acceptance examples are mathematical consequences of those operations.

Comments and recovery warnings are not treated as independent proof of behavior. No implementation access, runtime execution, or runtime tests have been performed.
