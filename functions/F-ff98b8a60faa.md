<!-- Trusted provenance: F-ff98b8a60faa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff98b8a60faa

## Purpose

F-ff98b8a60faa performs a prerequisite operation on a caller-supplied object and, if that operation reports zero, derives an integer result from the difference between two 32-bit fields in that object.

The established nonnegative computation is division by 24 with the remainder discarded. The negative-difference path computes an unsigned magnitude quotient and contains a sign-restoration step, but the supplied representation does not establish that step’s behavior reliably: it includes a signed 32-bit shift by 32. Consequently, signed division toward zero is an inferred intent, not an unconditional contract for negative differences.

A return value of zero is ambiguous. It can indicate that the prerequisite operation reported nonzero, or that the subsequent arithmetic produced a zero quotient.

## Interface

### Inputs

The function accepts two arguments:

| Input | Established use |
|---|---|
| Object address | A recovered 32-bit address-like value used as the base for fixed-offset memory reads and passed to F-0a08bd5d35c9. |
| Auxiliary value | An opaque 32-bit value passed unchanged to F-0a08bd5d35c9. This function does not otherwise inspect it. |

The object must provide readable storage at these offsets:

| Offset from object base | Width | Interpretation in this function |
|---|---:|---|
| `0xEF9` | 1 byte | Selects one of two prerequisite-operation modes by zero versus nonzero. |
| `0xEB0` | 4 bytes | Subtrahend in the post-operation difference. |
| `0xEB4` | 4 bytes | Minuend in the post-operation difference. |

The offsets establish access locations, not a complete object layout. No field names, object type, allocation size, byte order, or general alignment guarantee is supplied. The caller must satisfy the target’s requirements for these accesses and any additional requirements imposed by F-0a08bd5d35c9.

### Output

The recovered return type is a signed 32-bit integer.

For a successful prerequisite operation followed by a nonnegative 32-bit difference \(d\), the established result is:

\[
\left\lfloor \frac{d}{24} \right\rfloor
\]

For negative differences, the unsigned magnitude division is identifiable, but the final sign is subject to the uncertainty described below.

The function exposes no separate error code, remainder, status structure, or output pointer. It does not forward a nonzero prerequisite result to its caller.

### Ownership and lifetime

The visible body neither allocates nor explicitly releases storage. It passes the object address and auxiliary value to F-0a08bd5d35c9.

No ownership transfer, retention policy, or release behavior can be established for that dependency. In particular, successful return from the dependency must leave the two arithmetic fields accessible if this function is to complete normally. The visible body does not protect against the dependency invalidating that storage.

## Behavior

### 1. Select the prerequisite-operation mode

The function first reads the byte at object offset `0xEF9`.

- If the byte is zero, the selected 32-bit mode is `0x00000002`.
- If the byte is nonzero, the selected 32-bit mode is `0x80000000`.

All nonzero byte values select the same mode. The byte is not treated as a count, and no individual bit has a separately visible meaning.

The selected mode is determined before the dependency is called. A change to the flag during that call does not trigger another mode selection in the visible body.

### 2. Invoke the prerequisite operation

The function calls F-0a08bd5d35c9 with four arguments, in this logical order:

1. The object address.
2. The unchanged auxiliary value.
3. The selected 32-bit mode.
4. Zero.

It then examines the dependency’s returned integer.

If that result is nonzero, F-ff98b8a60faa returns zero immediately. On this path, its visible body does not read either arithmetic field. The dependency may have read or modified those fields internally; its implementation is not supplied.

If the dependency returns zero, processing continues. There is no visible retry, rollback, or compensation for effects of that call.

### 3. Read the post-operation arithmetic fields

After the zero result, the function reads the 32-bit fields at offsets `0xEB4` and `0xEB0` and forms the difference:

\[
d = \text{field at }0xEB4 - \text{field at }0xEB0
\]

These are post-call reads. The function does not capture the fields before invoking F-0a08bd5d35c9. Changes made by that dependency can therefore affect the result.

The supplied expression establishes that both reads occur after the dependency returns successfully, but does not establish their relative order. It also does not show a lock, atomic pair load, or consistency check. If another actor can modify the fields concurrently, the body does not establish that they represent a coherent snapshot.

Subsequent arithmetic operates on a 32-bit difference representation. There is no visible widening to a 64-bit difference. For inputs whose mathematical signed subtraction exceeds the signed 32-bit range, the precise subtraction semantics remain a target-level uncertainty; the examples below avoid relying on such overflow.

### 4. Compute the quotient

The arithmetic separates differences with a clear sign bit from those with a set sign bit. An OR with the constant 24 participates in this decision, but does not change the sign bit and therefore does not change that distinction.

#### Nonnegative differences

For a nonnegative 32-bit difference, the visible arithmetic computes the quotient by 24 through unsigned comparisons, shifts, and conditional subtraction.

Observable result boundaries include:

- Differences from 0 through 23 return 0.
- Differences from 24 through 47 return 1.
- Every further complete group of 24 increases the result by one.
- A remainder does not round the quotient upward.
- Larger differences enter a general division sequence rather than being capped at the range of the shorter sequence.

The largest nonnegative signed 32-bit difference, 2,147,483,647, produces a quotient of 89,478,485, with remainder 7. That quotient fits in the signed 32-bit return type.

The division has no visible memory accesses or dependency calls on its normal path.

#### Negative differences

When the difference has its sign bit set, the body conditionally negates its 32-bit unsigned representation to obtain a magnitude, then applies a general division-by-24 sequence.

Interpreted as 32-bit unsigned arithmetic, this magnitude formation also represents the magnitude 2,147,483,648 for the most-negative signed 32-bit difference. Its quotient by 24 is 89,478,485, with remainder 8. Thus the quotient magnitude fits within the signed 32-bit range even for this boundary case.

The body subsequently contains a conditional negation intended to restore a negative sign. However, the value used to control that negation originates in a signed 32-bit right shift by 32 in the supplied representation. That shift cannot be assigned portable 32-bit integer semantics from this evidence.

The following distinctions are therefore required:

- **Established:** negative-bit-pattern inputs enter magnitude division.
- **Established:** the visible terminal arithmetic conditionally negates the quotient.
- **Inferred intent:** return signed division by 24, truncating toward zero.
- **Documented only:** whether the actual target reliably selects the negative result.

A negative input whose quotient magnitude is zero is unaffected by this sign uncertainty. Either sign choice yields zero.

### 5. Exceptional terminal path

The supplied representation includes a branch for a zero working divisor. That branch calls F-dd3f3ec55291 with arguments `(2, 2)`.

If the dependency returns zero, the function returns zero. If it returns nonzero, the body invokes a software interrupt with immediate `0x123456`. If execution returns from that interrupt, it enters an infinite loop with no visible exit.

The normal arithmetic initializes its divisor from nonzero constants and reduces it through a bounded sequence. No ordinary input is established as reaching the zero-divisor branch. This branch is therefore a documented-only exceptional path, not evidence that a caller can request division by zero or trigger a particular error response through the object fields.

## Side effects

### Direct effects

The visible body performs:

- One byte read before the prerequisite call.
- Two 32-bit reads after that call, only if it returns zero.
- No explicit writes to the object.
- No explicit writes to global storage.
- No explicit allocation, deallocation, or ownership bookkeeping.

Arithmetic temporaries are not externally exposed.

### Dependency effects

F-0a08bd5d35c9 is invoked on every ordinary entry that reaches the dependency call. Its effects are unknown. The contract does not assume that it preserves:

- Object fields.
- Global state.
- External resources.
- Memory ownership or validity.

In particular, returning zero from F-ff98b8a60faa does not imply that nothing changed. The prerequisite dependency may already have performed externally observable work.

The exceptional path may invoke F-dd3f3ec55291 and a software interrupt. Their effects are unknown. If execution reaches the final loop, the visible body makes no further progress toward a return.

### Invalid access and dependency failure

The body has no visible null check, bounds check, address validation, or access-fault recovery. Invalid object storage can fail at the initial byte read or, after a zero dependency result, at either arithmetic-field read.

If a dependency fails to return normally, this function has no visible mechanism that converts that event into a normal integer result.

## Dependencies

| Opaque function ID | Invocation condition | Supplied arguments | Locally observed result |
|---|---|---|---|
| F-0a08bd5d35c9 | After reading the mode byte | Object address, auxiliary value, selected mode, zero | Nonzero causes immediate return of zero; zero permits arithmetic. |
| F-dd3f3ec55291 | Documented-only zero-divisor branch | `2`, `2` | Zero causes return of zero; nonzero leads to the software interrupt. |

These are the only named function dependencies visible in the supplied body.

The software interrupt is an additional external control-transfer mechanism, not a mapped function dependency. Its service meaning and platform effects are unknown.

No assumption is made that either dependency is pure, preserves globals, retains or releases memory, or interprets its arguments according to a particular library convention.

## Acceptance examples

These are static acceptance conditions, not executed tests. Unless stated otherwise, the object storage is valid, F-0a08bd5d35c9 returns normally with zero, and the stated field values describe the reads made after that return.

| Condition | Required observable behavior |
|---|---|
| Mode byte is zero | Call F-0a08bd5d35c9 with mode `0x00000002` and fourth argument zero. |
| Mode byte is any nonzero value | Call F-0a08bd5d35c9 with mode `0x80000000` and fourth argument zero. |
| F-0a08bd5d35c9 returns any nonzero value | Return zero without direct arithmetic-field reads by this function. |
| Post-call fields are equal | Return zero. |
| Post-call difference is 23 | Return zero. |
| Post-call difference is 24 | Return one. |
| Post-call difference is 47 | Return one. |
| Post-call difference is 48 | Return two. |
| Post-call difference is 6,143 | Return 255. |
| Post-call difference is 6,144 | Return 256; crossing to the general division sequence does not change the quotient rule. |
| Post-call difference is 2,147,483,647 | Return 89,478,485. |
| Post-call difference is −23 | Return zero under either possible final sign choice. |

Additional conditional cases:

- **Dependency mutation:** If the fields would have produced a difference of zero before the prerequisite call, but their post-call values produce 48, the arithmetic result is two. The pre-call values are not the inputs to the division.
- **Mode mutation:** If the mode byte is zero when read and becomes nonzero during the dependency call, that invocation still receives `0x00000002`. The visible body does not reread the mode.
- **Negative quotient:** For a difference of −25, magnitude division yields one. If target behavior establishes correct sign restoration, the return is −1. The supplied representation alone does not establish that condition.
- **Most-negative difference:** For a difference of −2,147,483,648, magnitude division yields 89,478,485. With correct sign restoration, the return is −89,478,485. The sign qualification remains necessary.
- **Exceptional branch:** If execution reaches the zero-divisor branch and F-dd3f3ec55291 returns nonzero, a normal return is not established. The interrupt is invoked, followed by an infinite loop if the interrupt returns.

## Uncertainties

### Documented only

1. **Negative-result sign restoration.** The supplied signed shift by 32 prevents an unconditional negative-input return contract.
2. **Overflowing field subtraction.** The representation shows 32-bit arithmetic, but does not independently establish target behavior for subtraction outside the signed 32-bit range.
3. **Zero-divisor branch reachability.** The branch exists in the supplied representation; no normal input reaching it is established.
4. **Dependency semantics.** Their application-level purpose, side effects, ownership rules, and failure behavior are unavailable.
5. **Interrupt semantics.** The immediate is visible, but its service meaning and return behavior are unknown.
6. **Memory environment.** Byte order, alignment rules, synchronization requirements, and access-fault behavior are not supplied.
7. **Representation fidelity.** Recovery warnings identify unreachable-block removal, overlapping global symbols, and restarted dead-code processing. They limit confidence in exceptional control flow and unusual arithmetic expressions; they do not supply replacement semantics.

### Inferred intent

The arithmetic strongly indicates a difference measured in units of 24, with signed truncation toward zero intended for negative values. Neither the meaning of those units nor the application-level meaning of the two fields is established.

## Evidence

This contract is based solely on the supplied static representation of F-ff98b8a60faa and the supplied opaque dependency mapping.

The evidence establishes the field offsets and widths, mode selection, prerequisite-call arguments, early-return condition, post-call field reads, nonnegative quotient calculation, negative-magnitude calculation, and documented exceptional branch.

No implementation access, dependency bodies, target instruction trace, caller analysis, or runtime tests were available. Acceptance examples are derived conditions, not observed executions. No snapshot hash or runtime provenance is asserted.
