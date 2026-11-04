<!-- Trusted provenance: F-aa44afff68ed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa44afff68ed

## Purpose

F-aa44afff68ed conditionally obtains an unsigned sample, updates a shared 32-bit value, maintains a separate 32-bit counter, and returns an unsigned 32-bit result. A final byte-controlled override can replace the return value with 5 without undoing preceding operations.

The established outcomes, before that override, are:

- Return an entry-time value and increment the counter without requesting a sample.
- Accept a sample from 0 through 256, convert it to a value from 0 through 22, optionally limit a downward change, and store the selected result.
- Return a sample greater than 256 unchanged, incrementing the counter without directly writing the shared value.

There are **two separate reads of the global shared-state base reference at entry**. The first captures the base used for subsequent field accesses. The second supplies the address for the entry-time value read. The supplied representation does not establish that these two reference reads identify the same object.

Terms such as “sample,” “gate,” and “downward limiting” describe the mechanics. Their application-level purpose is unknown.

## Interface

### Explicit arguments

| Argument | Width and interpretation | Established use |
|---|---|---|
| First argument | 32-bit quantity; semantic type unspecified | Not read by the visible body. |
| Second argument | Signed 32-bit integer | Compared with zero on the accepted-sample path. Zero enables the downward-change rule; every nonzero value disables it. |

Neither argument is explicitly passed to either dependency. Incidental register contents and undocumented calling-convention effects are not established.

### Return value

The return value is unsigned and 32 bits wide.

Before the final override, it is one of:

- The 32-bit value obtained by the entry-time load.
- The converted or downward-limited value written on the accepted-sample path.
- The raw unsigned sample when that sample exceeds 256.

A nonzero final override byte makes the returned value exactly 5.

### References and field layout

The function accesses a global base reference twice before calling either dependency:

1. Its first read captures the base for subsequent field accesses.
2. Its second read supplies the base for the entry-time 32-bit load at byte offset +8.

This contract calls them the **captured base** and the **entry-load base**, respectively. These names distinguish observations; they do not assert that the addresses differ.

| Address source | Byte offset | Width | Use |
|---|---:|---:|---|
| Entry-load base | +8 | 32 bits | Entry-time value, initially selected for return |
| Captured base | +8 | 32 bits | Later value read and accepted-sample result write |
| Captured base | +12 | 32 bits | Counter reads and writes |

When both reference reads produce the same address, the two +8 accesses concern the same field. Without declarations or synchronization assumptions, that identity is not guaranteed. The entry-time value can therefore come from a different object than the object whose counter or value is subsequently accessed.

A separate global reference identifies a byte read near return. Its zero/nonzero status controls the return override.

### Preconditions for normal completion

The selected execution path requires:

- Both entry reads of the global base reference to be valid.
- A readable 32-bit value at the entry-load base plus 8.
- Valid captured-base storage for every field read or written on the selected path.
- A readable final override reference and referenced byte.
- Dependencies that return normally and leave any storage needed afterward valid.

The captured base’s +8 field is accessed only on the accepted-sample path. The entry-load base’s +8 field is read on every normal execution, even when a later sample replaces its value as the provisional result.

The visible body performs no pointer, extent, alignment, lifetime, or concurrency validation. The representation does not supply enough memory declarations to establish the complete address model or byte order.

## Behavior

### Entry observations and gate call

The function first reads the global shared-state base reference and captures that address.

It then reads the same global reference again and uses this second observation to address a 32-bit load at offset +8. That loaded value becomes the provisional return result. It is not necessarily a load through the captured base.

Next, the function calls F-60e87f3a4344, with no explicit arguments in the supplied representation. The result is tested for zero versus nonzero.

If the gate result is nonzero, the function reads the unsigned 32-bit counter at captured-base offset +12.

Sampling is selected when either:

- The gate result is zero; or
- The gate result is nonzero and the observed unsigned counter exceeds 5.

When the gate result is zero, no counter read is needed to make this decision. The gate result is retained for later decisions; the gate dependency is not called again.

### Sampling skipped

A nonzero gate result and an observed counter from 0 through 5 inclusive cause sampling to be skipped.

The function:

- Writes the observed counter plus one to captured-base offset +12.
- Makes no direct write to either +8 location.
- Retains the entry-time value as the provisional return result.

The counter becomes a value from 1 through 6, so this particular increment cannot wrap.

The result is not refreshed after the gate call. It can differ from the captured object’s current +8 value because the two entry reference reads identified different objects, because the dependency changed storage, or because of other externally permitted changes.

### Sampling selected

The function calls F-5363d29c7612 once, with no explicit arguments in the supplied representation. Its result is treated as an unsigned 32-bit sample and replaces the provisional return result.

Samples from 0 through 256 inclusive are accepted. Samples from 257 through 4,294,967,295 take the above-range path.

### Accepted sample: conversion

For an accepted sample, the converted value is:

\[
\left\lfloor \frac{22 \times \text{sample}}{256} \right\rfloor
\]

The represented operation multiplies by 22 and shifts the unsigned product right by eight bits. Because the range check precedes this computation, the largest product is 5,632. No 32-bit multiplication overflow occurs on this path.

The converted value is between 0 and 22 inclusive. In particular:

- Sample 0 converts to 0.
- Sample 255 converts to 21.
- Sample 256 converts to 22.

### Accepted sample: counter and value update

After conversion, a zero gate result causes a 32-bit zero write to captured-base offset +12. A nonzero gate result causes no direct counter write on this path.

The function then reads the 32-bit value at captured-base offset +8. This read occurs:

- After both dependency calls.
- After the counter reset, if that reset occurs.
- Independently of the entry-time value load.

This later value supplies the comparison for downward limiting. The selected result becomes the later value minus one only when all of the following hold:

- The later value is not 23.
- The second argument is zero.
- The converted value is strictly less than the later value, using unsigned comparison.

Otherwise, the selected result remains the converted value.

The function writes the selected result to captured-base offset +8, including when the write leaves the field numerically unchanged. That same selected result becomes the provisional return value.

The subtraction cannot underflow: its enabling comparison requires the later value to be at least 1. A downward-limited result need not remain in the conversion range. A later value of 100 and converted value of zero produce 99.

The exception for 23 bypasses downward limiting; it does not force the stored result to 23.

### Sample above 256

An above-range sample is not converted and does not cause a direct +8 write.

Instead, the function:

- Reads the counter at captured-base offset +12 after the sample dependency returns.
- Writes that freshly observed counter plus one to the same field.
- Retains the raw sample as the provisional return result.

This post-sample counter read is required even if a counter was previously read to select sampling. Dependency changes to the counter can therefore affect the increment.

A zero gate result does not reset the counter on this path.

The increment uses unsigned 32-bit arithmetic. A counter value of 4,294,967,295 becomes zero; other values increase by one.

### Final override

After the selected path’s field operations, the function reads the separate override reference and the byte it designates.

- A zero byte leaves the provisional result unchanged.
- Any nonzero byte replaces the result with 5.

The override does not cancel dependency calls, reverse counter changes, or undo a shared-value write. It does not itself write 5 to the +8 field.

## Side effects

### Observable access order

The source-level sequence is:

1. Read the global shared-state base reference and capture it.
2. Separately read that global reference again.
3. Read the entry-time 32-bit value at the second observed base plus 8.
4. Call F-60e87f3a4344.
5. If its result is nonzero, read the counter at the captured base plus 12.
6. If sampling is selected, call F-5363d29c7612.
7. Perform the selected path’s field accesses.
8. Read the override reference, then its designated byte.
9. Return the selected unsigned 32-bit result.

The path-specific accesses in step 7 are:

| Path | Ordered field accesses through the captured base |
|---|---|
| Sampling skipped | Write the incremented counter at +12 |
| Accepted sample, zero gate | Write zero at +12; read value at +8; write selected result at +8 |
| Accepted sample, nonzero gate | Read value at +8; write selected result at +8 |
| Above-range sample | Read counter at +12; write incremented counter at +12 |

All post-entry field accesses use the captured base. Replacing the global base reference during a dependency call does not redirect those accesses. The function does not read that global reference again for later field addressing.

This ordering describes the supplied representation. It does not establish atomicity, a concurrency memory model, or hardware-level synchronization.

### Ownership and lifetime

No direct allocation, deallocation, or ownership transfer is visible.

Neither dependency is assumed to preserve global references, preserve field values, or retain ownership of referenced memory. If a dependency invalidates captured-base storage, the function still uses the captured address for any subsequent field operations. Normal completion requires those particular accesses to remain valid.

The override reference is obtained late. Its contents may have been changed by a dependency. If the override byte aliases storage written earlier, the final check observes it according to the stated access order; independence is not guaranteed.

### Error behavior

There is no visible explicit error status, retry, recovery mechanism, or exception handler.

An above-range sample becomes a raw provisional return value and causes a counter increment. The evidence does not establish that this value is an application-level error code.

Memory faults, nonreturning dependencies, and dependency failure mechanisms are outside the established normal-return behavior. No rollback is visible.

## Dependencies

| Opaque ID | Invocation | Result usage | Unresolved behavior |
|---|---|---|---|
| F-60e87f3a4344 | Once, after the two base-reference reads and entry value load | Zero/nonzero selects sampling eligibility and controls accepted-sample counter reset | Global mutation, storage mutation, lifetime effects, failure behavior |
| F-5363d29c7612 | Once when the gate is zero or the observed unsigned counter exceeds 5; otherwise not called | Supplies the unsigned 32-bit sample | Global mutation, storage mutation, lifetime effects, failure behavior |

No explicit arguments are supplied to these calls in the representation.

The sample comparison is unsigned. A bit pattern of 0xFFFFFFFF denotes 4,294,967,295 for this decision and is therefore above range, regardless of a possible signed interpretation elsewhere.

## Acceptance examples

These are conditional consequences of static evidence, not execution reports. Unless stated otherwise, both entry reference reads identify the same valid object, dependencies return normally without changing relevant storage, the override byte is zero, and required storage remains valid.

1. **Sampling skipped at the threshold.**  
   Entry value 12, nonzero gate result, counter 5. The sample dependency is not called. The counter becomes 6, no direct +8 write occurs, and the result is 12.

2. **Distinct entry bases remain distinct.**  
   The first reference read captures one object; the second identifies another. The captured object has value 18 and counter 2. The entry-load object has value 12. With a nonzero gate result, sampling is skipped: the captured object’s counter becomes 3, its value remains 18, the other object is not written, and the result is 12.

3. **Accepted sampling uses the captured object, not the entry-load object.**  
   The two entry bases differ. The entry-load object supplies value 4. The captured object’s later value is 100. With a zero gate result, sample 0, and zero second argument, the captured counter is reset before the later value read. The captured value becomes 99 and the result is 99. The entry-load object receives no direct write.

4. **Counter above the threshold permits sampling.**  
   Value 7, nonzero gate result, counter 6, sample 256. Conversion produces 22. The value becomes 22 and the result is 22 for either zero or nonzero second argument. No direct counter write occurs.

5. **Zero gate resets only on acceptance.**  
   Zero gate result, counter 9, sample 0, nonzero second argument. The counter becomes zero, then the value is read and written as zero. The result is zero.

6. **Downward limiting is conditional.**  
   Sampling is selected, sample 0, later captured value 10. A zero second argument produces a stored and returned value of 9. A second argument of 1 or −1 instead produces zero.

7. **Special value 23 bypasses limiting.**  
   Sampling is selected, sample 0, later captured value 23, second argument zero. The stored and returned value is zero, not 22.

8. **Conversion and acceptance boundaries differ.**  
   With limiting disabled, samples 255 and 256 produce stored results 21 and 22. Sample 257 instead causes no direct value write and returns 257 before any override.

9. **Above-range sampling increments rather than resets.**  
   Zero gate result, sample 257, post-sample counter 9. The counter becomes 10, no direct value write occurs, and the result is 257.

10. **The above-range increment uses the post-call counter.**  
    A nonzero gate and counter 6 select sampling. The sample dependency changes the counter to 40 and returns 257. The subsequent read observes 40, so the function writes 41 rather than 7.

11. **Counter arithmetic wraps.**  
    Sampling returns an above-range sample and the post-sample counter read is 4,294,967,295. The counter is written as zero. The raw sample remains the provisional result.

12. **A dependency cannot redirect the captured field base merely by replacing the global reference.**  
    Both entry reads initially identify one object. The gate dependency replaces the global reference with another object, returns nonzero, and leaves the original counter at 2. Sampling is skipped and the original object’s counter becomes 3.

13. **The override changes the return, not preceding operations.**  
    Any otherwise-valid example runs with a final override byte of 2. Its described writes and calls still occur, but the result is 5.

## Uncertainties

### Inferred intent

The threshold and downward-change rule suggest delayed sampling and smoothing. These are interpretations, not established application semantics.

The value 23 may represent a distinguished state. Only its exemption from downward limiting is established.

### documented_only

The following remain unresolved:

- Semantic names, units, initialization, and intended ranges of the shared fields.
- Whether the two entry base-reference reads always yield the same address in the actual environment.
- Memory declarations, address representation, alignment requirements, and byte order.
- Mechanisms that could change the reference or storage between observations.
- Dependency side effects, ownership behavior, and failure conventions.
- Lifetime and meaning of the final override reference.
- Aliasing between the override byte and other accessed storage.
- Concurrency, reentrancy, and external synchronization requirements.
- Application interpretation of above-range raw samples.

The contract does not assume stable references, independent storage, or well-behaved dependency ownership. Conditional behavior must be evaluated using the separate observations and access order above.

## Evidence

This contract is based solely on static inspection of the supplied function representation and opaque dependency mapping.

Established evidence includes:

- Two distinct entry reads of the global base reference.
- An entry +8 load addressed through the second reference observation.
- Subsequent +8 and +12 accesses addressed through the first, captured observation.
- Unsigned thresholds of 5 and 256.
- Conversion by multiplication by 22 and an eight-bit right shift.
- The comparison against 23 and the zero test of the second argument.
- Path-dependent counter resets and increments.
- The final byte-controlled return override.

No dependency implementations, callers, complete memory declarations, or runtime observations were supplied. No runtime tests have been performed.
