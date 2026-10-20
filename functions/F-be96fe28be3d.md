<!-- Trusted provenance: F-be96fe28be3d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be96fe28be3d

## Purpose

F-be96fe28be3d invokes F-0a08bd5d35c9 with two caller-supplied values and three fixed or externally supplied arguments. It interprets a zero dependency result as permission to derive its own result from two 32-bit fields associated with the first input. Any nonzero dependency result produces a return value of zero.

The derived result is the difference between the field at byte offset 0xEE4 and the field at byte offset 0xEE0, shifted right by three bits. For nonnegative, representable differences, this equals the difference divided by eight with any remainder discarded.

A plausible intent is to report a quantity measured in eight-byte units after a preparatory operation. That interpretation is inferred, not established: the available evidence does not identify the object, the field meanings, or the dependency’s operation.

This contract describes the visible wrapper behavior. It does not assign undocumented memory-management, error-handling, or global-state guarantees to the dependency.

## Interface

### Inputs

The recovered interface exposes two arguments:

| Input | Recovered representation | Visible use |
|---|---|---|
| First input | 32-bit integer used as an address-bearing value | Forwarded to F-0a08bd5d35c9; subsequently used as the base for two field reads if the dependency returns zero |
| Second input | Opaque 32-bit value | Forwarded unchanged to F-0a08bd5d35c9 |

The first input’s integer representation does not establish that arbitrary integer values are valid. When the dependency returns zero, the wrapper requires readable storage at the two field locations described below.

The second input is not directly interpreted, dereferenced, validated, or modified by the visible wrapper. It may nevertheless designate mutable storage or carry other semantics understood by the dependency.

The recovered widths describe the supplied static representation. The evidence does not independently establish a complete calling convention or the original source-language types.

### Output

The function returns a 32-bit integer result:

- If F-0a08bd5d35c9 returns any nonzero value, the wrapper returns zero.
- If F-0a08bd5d35c9 returns zero, the wrapper returns the shifted difference of the two fields.

Zero therefore has at least two observable meanings: a nonzero dependency result, or a field-derived result of zero. The interface provides no separate status output to distinguish them.

### Referenced layout

The visible wrapper uses the following locations relative to the address represented by the first input:

| Byte offset | Access width | Role in the result |
|---|---:|---|
| 0xEE0, decimal 3808 | 32 bits | Subtracted field |
| 0xEE4, decimal 3812 | 32 bits | Field from which the other field is subtracted |

The reads are adjacent and four bytes apart. Their roles do not establish that they are pointers, lengths, counters, or bounds.

For an ordinary contiguous object, both reads require storage extending through byte offset 0xEE7. This is a conditional access requirement, not evidence of the object’s complete size. Alignment and byte order follow the target environment; the wrapper contains no visible handling for unsupported alignment or an alternative encoding.

## Behavior

### Dependency invocation

The wrapper first calls F-0a08bd5d35c9 with five positional arguments:

1. The first caller input.
2. The second caller input.
3. Integer zero.
4. The integer constant 0x20000, decimal 131072.
5. A value represented by an external static datum in the supplied evidence.

The first two inputs are forwarded without a visible conversion or validation step. The third and fourth argument values are fixed for this wrapper.

The fifth argument’s value, width, interpretation, and ownership implications are not established. Its presence must not be replaced with an invented null value, callback, size, address, or flag meaning.

There is one explicit dependency invocation in the wrapper. No retry, fallback invocation, or dependency-status translation is visible.

### Nonzero dependency result

If the dependency returns any nonzero value, the wrapper returns zero immediately.

On this path, the wrapper does not perform either of its two explicit field reads. This restriction applies only to the wrapper’s own accesses: the dependency may already have read or written memory associated with either input.

Positive and negative nonzero dependency results receive the same treatment. Their original values are not returned to the caller.

### Zero dependency result

If the dependency returns zero, the wrapper reads both 32-bit fields using the first input as their common base address.

These reads occur after the dependency returns. Consequently, the result is based on post-call memory, not on a snapshot taken before invocation. Any dependency mutation of those fields can affect the returned value.

The arithmetic uses the value at offset 0xEE4 minus the value at offset 0xEE0. The resulting difference is then shifted right by three bits.

The subtraction precedes the shift. Shifting each field separately and then subtracting would be a different operation and is not supported by the evidence.

The supplied expression does not establish which field is loaded first. It establishes that both wrapper loads belong to the zero-result path after the dependency call. It does not establish an atomic pair read, a coherent concurrent snapshot, or any synchronization.

### Arithmetic boundaries

Both field reads and the returned integer are represented as 32-bit signed quantities in the recovered interface. No explicit widening to a 64-bit intermediate is shown.

For a nonnegative difference representable in signed 32-bit arithmetic, the result is unambiguous: divide that difference by eight and discard the remainder. Thus differences from zero through seven all produce zero.

The evidence is insufficient to prescribe portable behavior for every possible pair of 32-bit field contents:

- Subtraction can exceed the signed 32-bit range.
- No saturation, overflow check, or explicit unsigned conversion is visible.
- A right shift of a negative signed value requires target-specific interpretation not independently established here.
- Address formation can also depend on target integer and pointer behavior when the first input is near an address-space boundary.

A conservative domain for a fully determined numeric result is therefore: valid field addresses, readable signed 32-bit field values, and a mathematical difference between zero and 2,147,483,647 inclusive. Behavior outside that domain is documented_only where target semantics are required.

The wrapper does not verify that the difference is a multiple of eight. It also does not explicitly reject reversed field values.

### Invalid inputs and abnormal completion

No explicit null check, address-range check, object-type check, field-order check, or field-alignment check appears in the visible wrapper.

An invalid first input is not guaranteed to be harmless on the nonzero-result path because the dependency receives it before the wrapper chooses a branch.

If the dependency returns zero while either required location is unreadable, no defined wrapper-level error return is established. A memory fault or another target-dependent failure is possible, but the supplied evidence does not specify fault handling.

If the dependency does not return normally, the wrapper has no visible recovery path. This contract does not assume that every invocation reaches either return branch.

## Side effects

The wrapper contains no explicit store to the caller-associated fields and no visible direct allocation or release operation.

Those observations do not make the operation pure or read-only. F-0a08bd5d35c9 may have side effects that are not described by the supplied evidence, including changes to caller-accessible memory or global state.

In particular:

- The dependency may change either field before the wrapper reads it.
- The wrapper does not cache the field values before calling the dependency.
- No guarantee is established that the dependency preserves unrelated globals.
- No ownership transfer, retention, allocation, or deallocation contract is established for either caller input or the fifth dependency argument.
- No rollback is visible when the dependency returns nonzero.
- Returning zero does not establish that memory or external state remained unchanged.

On the zero-result path, the storage used for both field reads must remain readable after the dependency returns. The wrapper performs no visible lifetime check before accessing it. Whether the dependency guarantees that lifetime is unknown.

The wrapper provides no synchronization around these accesses. Concurrent mutation of the fields is outside the established stable-input behavior; no particular concurrent observation is promised.

## Dependencies

### F-0a08bd5d35c9

This is the sole explicit function dependency shown.

Its observable relationship to F-be96fe28be3d is limited to:

- Receiving the five arguments in the positions described above.
- Running before the wrapper’s explicit field reads.
- Returning a value compared with zero.
- Selecting the wrapper’s zero-return path when that value is nonzero.
- Selecting the field-derived return path when that value is zero.

Calling the zero result “success” is an inferred convention, not an established dependency contract. The wrapper only demonstrates a zero-versus-nonzero decision.

The dependency’s accepted input domain, memory access patterns, global effects, ownership behavior, exceptions or faults, and numeric status meanings remain unknown.

### External static datum

The fifth dependency argument is represented by a static datum whose contents and declaration are unavailable. No fixed value or semantic role can be assigned to it from this evidence.

The wrapper contains no explicit assignment to that datum. This does not establish that it is immutable or preserved by the dependency.

## Acceptance examples

These examples are conditional behavioral checks, not reports of executed tests. Unless stated otherwise, assume valid addresses, ordinary readable storage, and no concurrent field mutation.

| Conditions | Required wrapper outcome |
|---|---|
| The dependency returns 1 | Return zero; perform neither explicit field read |
| The dependency returns −1 | Return zero; treat the negative nonzero result like any other nonzero result |
| The dependency returns zero; the post-call fields at 0xEE0 and 0xEE4 are both 100 | Return zero |
| The dependency returns zero; the post-call fields are 100 and 108 respectively | Return 1 |
| The dependency returns zero; the post-call fields are 100 and 115 respectively | Return 1; the seven-unit remainder is discarded |
| The dependency returns zero; the post-call fields are 100 and 116 respectively | Return 2 |
| The dependency returns zero; the post-call fields are −16 and 0 respectively | Return 2; the representable difference is positive despite a negative field value |

Additional boundary conditions:

- **Dependency mutation:** If the dependency changes the fields from equal values to 200 and 224, then returns zero, the required result is 3. Pre-call equality must not determine the result.
- **Small positive difference:** If the dependency returns zero and the post-call difference is seven, the required result is zero. The wrapper does not round upward.
- **Negative difference:** For post-call fields 16 and 8 respectively, the difference is −8. If the target uses arithmetic signed right shift, the result is −1. That shift rule remains conditional rather than an unconditional cross-platform guarantee.
- **Overflowing subtraction:** For post-call fields −1 and 2,147,483,647 respectively, the mathematical difference is 2,147,483,648. No portable result is specified here because the recovered operation has no explicit wider intermediate.
- **Unreadable post-call storage:** A zero dependency result does not turn invalid field addresses into a defined zero return. No wrapper-level recovery is established.

## Uncertainties

The following remain documented_only:

- The first input’s object type and full layout.
- The meanings and validity constraints of the two fields.
- The second input’s semantics.
- The fifth dependency argument’s value, type, and role.
- The dependency’s ownership and lifetime guarantees.
- The dependency’s global and external side effects.
- The meaning of individual dependency return values beyond zero versus nonzero.
- The order of the two field reads.
- Target behavior for negative signed right shifts and overflowing signed subtraction.
- Address-overflow, misalignment, and invalid-memory behavior.
- Any higher-level interpretation of the result as an element count, distance, capacity, or other unit.

An eight-byte-unit interpretation is plausible because of the three-bit shift. It is not evidence of an eight-byte element layout or a guarantee that valid differences are eight-byte aligned.

## Evidence

Established static evidence consists of the supplied recovered body for F-be96fe28be3d and the supplied opaque dependency mapping. It shows one dependency call, fixed third and fourth arguments, an externally represented fifth argument, a zero-result branch with two offset-based 32-bit reads, subtraction followed by a three-bit right shift, and a zero return for nonzero dependency results.

Inferred intent is limited to a possible preparation-then-measure operation and a possible eight-byte-unit interpretation.

No dependency implementation, object declaration, complete target arithmetic specification, or runtime observation was provided. No runtime tests have been performed.
