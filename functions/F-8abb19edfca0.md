<!-- Trusted provenance: F-8abb19edfca0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8abb19edfca0

## Purpose

F-8abb19edfca0 compares a signed value read through a context with a signed 64-bit value supplied as two 32-bit words. The comparison selects between a context-only dependency and a dependency receiving the supplied low word.

When the comparison permits the low-word dependency to run, its signed 32-bit result normally becomes the function’s return value. If that result is zero and the supplied 64-bit value is strictly positive, an additional context-only dependency runs before the function returns zero.

This is a numeric dispatch contract. The supplied evidence does not establish that the values represent sizes, offsets, capacities, timestamps, or any other domain-specific quantity.

## Interface

The function has four inputs and returns a signed 32-bit integer.

| Input | Width and interpretation | Use established by the supplied evidence |
|---|---|---|
| Context | 32-bit address-bearing value | Used for two indirect memory reads and passed unchanged to selected dependencies. |
| Auxiliary word | 32 bits, otherwise uninterpreted | Not read, tested, or forwarded by this function. |
| Low word | Unsigned 32-bit integer | Supplies the low half of the logical signed 64-bit comparison value; also passed alone to F-391e3d30e634. |
| High word | Signed 32-bit integer | Supplies the high half of the logical signed 64-bit comparison value. |

The logical supplied value is:

**requested value = high word × 4,294,967,296 + low word**

This definition spans the entire signed 64-bit range. It is a mathematical interpretation of two arguments, not evidence that the caller stores them in adjacent memory or supplies a native 64-bit argument.

### Required memory layout

The function uses this context-relative layout:

| Location | Access | Interpretation |
|---|---|---|
| Context address plus 24 bytes | Read of one 32-bit word | Address of a secondary object. |
| Secondary-object address plus 80 bytes | Read of one 32-bit word | Stored numeric value. |

The stored numeric word is first available as 32 raw bits. Its interpretation for comparison is **signed 32-bit**, extended to signed 64-bit without changing its numeric value.

Consequently:

- A stored word of `0x7fffffff` represents 2,147,483,647.
- A stored word of `0x80000000` represents −2,147,483,648.
- A stored word of `0xffffffff` represents −1.

The function does not treat the stored word as an unsigned magnitude in the final comparison.

### Preconditions

For ordinary execution to reach a return:

- The context-relative address read must be valid.
- The resulting secondary-object-relative numeric read must be valid.
- The selected dependencies must be callable and return normally.
- The execution environment must support the address representation and memory accesses reflected in the supplied evidence.

There are no explicit null checks, range checks, or memory-access error conversions. Exact behavior for invalid addresses, unsupported alignment, or an abnormal dependency exit is not established.

## Behavior

### Initial reads and comparison

The function first reads the secondary-object address from the context at byte offset 24. It then reads the numeric word from that object at byte offset 80.

These reads occur before any dependency call.

Let **stored value** mean the signed 32-bit interpretation of the numeric word, sign-extended to signed 64-bit. The first branch is governed by:

**stored value ≥ requested value**

Equality takes the same path as a greater stored value.

The supplied arithmetic expresses this comparison using a low-word unsigned borrow, a sign-derived upper word, and signed-overflow predicates. Those predicates are significant: interpreting the intermediate subtractions as ordinary unbounded signed 32-bit arithmetic, or considering only the sign of a wrapped subtraction, would not preserve the branch decision at all boundaries.

The resulting signed 64-bit comparison is the behavioral contract. It does not require an implementation to reproduce the particular arithmetic decomposition.

### When the stored value is smaller

If the stored value is strictly less than the requested value:

1. F-bb73abb30feb is called with the original context input.
2. Its return value, if any, is not used.
3. F-8abb19edfca0 returns zero.

Neither F-391e3d30e634 nor F-251588f6a575 is called on this path.

This path applies to every requested value greater than 2,147,483,647, because the stored value can never exceed that signed 32-bit maximum.

### When the stored value is greater than or equal

If the stored value is greater than or equal to the requested value:

1. F-391e3d30e634 is called with the low word only.
2. Its signed 32-bit result is retained.
3. If that result is nonzero, the function returns it unchanged.
4. If that result is zero, the function additionally evaluates whether the requested value is strictly positive.
5. For a strictly positive requested value, F-251588f6a575 is called with the original context input.
6. The function returns zero.

F-bb73abb30feb is not called on this path.

The context, auxiliary word, and high word are not passed to F-391e3d30e634. In particular, this dependency does not receive the complete logical requested value through the explicit arguments shown.

### Strict positivity

The additional context-only call requires both:

- A zero result from F-391e3d30e634.
- A requested value strictly greater than zero.

The positivity condition is equivalent to either:

- The high word is positive; or
- The high word is zero and the low word is nonzero.

A negative high word always makes the requested value negative, regardless of the low word.

Because this condition is reached only after the stored value has been found greater than or equal to the requested value, an actual call to F-251588f6a575 implies a requested value between 1 and 2,147,483,647, inclusive.

A zero requested value does not trigger this additional call. A negative requested value does not trigger it either, even when its low word is nonzero.

### Return-value contract

There are three return cases:

| Condition | Return |
|---|---|
| Stored value is less than requested value | Zero, after F-bb73abb30feb returns. |
| Stored value is at least requested value and F-391e3d30e634 returns nonzero | That exact signed 32-bit result. |
| Stored value is at least requested value and F-391e3d30e634 returns zero | Zero, after any required F-251588f6a575 call returns. |

Negative results from F-391e3d30e634 are nonzero results and are returned unchanged. The evidence does not classify zero or nonzero as success or failure.

## Side effects

### Direct memory access

The function directly performs two memory reads, in this order:

1. The context’s secondary-object address.
2. The secondary object’s stored numeric word.

No direct write to the context, secondary object, or other caller-visible memory is shown.

The numeric value is read once for the branch decision. It is not reread after a dependency call.

### Dependency effects and ordering

Only these normal call sequences are possible:

- F-bb73abb30feb alone.
- F-391e3d30e634 alone.
- F-391e3d30e634 followed by F-251588f6a575.

The function does not invoke both context-only dependencies during the same execution.

The supplied evidence does not establish what any dependency reads or writes. A dependency may affect globals, shared state, the context, or resource lifetime according to its own contract. No preservation guarantee is implied merely because this function shows no direct write.

In the two-call sequence, F-251588f6a575 receives the original context argument after F-391e3d30e634 has completed. There is no intervening context reread or validity check. Any lifetime requirement across those calls is therefore an external precondition, not a guarantee enforced here.

### Ownership

No direct allocation, release, ownership transfer, or reference-count operation is established.

Passing the context to a dependency does not, by itself, establish whether that dependency borrows, retains, consumes, or releases it. Likewise, F-391e3d30e634’s integer return must not be described as an owned resource or address without additional evidence.

## Dependencies

| Opaque function ID | Invocation condition | Explicit input | Use of result |
|---|---|---|---|
| F-391e3d30e634 | Stored value is at least requested value | Low word, 32 bits | Signed 32-bit result is tested for zero and returned unchanged. |
| F-251588f6a575 | F-391e3d30e634 returned zero and requested value is positive | Original context | Result is not used. |
| F-bb73abb30feb | Stored value is less than requested value | Original context | Result is not used. |

Their internal semantics, global-state interactions, error behavior, and ownership rules are **documented_only** unknowns.

## Acceptance examples

These are conditional acceptance examples derived from the static behavior. They are not reported runtime observations. Each assumes valid memory accesses and normally returning dependencies.

| Stored word | High word | Low word | Required behavior |
|---|---:|---:|---|
| `0x00000005` | 0 | 6 | Stored value 5 is below requested value 6. Call F-bb73abb30feb with the context; return zero. |
| `0x00000005` | 0 | 5 | Equality permits F-391e3d30e634, receiving 5. Return its nonzero result unchanged; if it returns zero, call F-251588f6a575 and return zero. |
| `0x00000000` | 0 | 0 | Call F-391e3d30e634 with zero. A zero result does not trigger F-251588f6a575. |
| `0xffffffff` | 0 | 0 | Stored value −1 is below zero. Call F-bb73abb30feb; return zero. |
| `0xffffffff` | −1 | `0xffffffff` | Both values are −1. Call F-391e3d30e634 with `0xffffffff`. A zero result does not trigger F-251588f6a575 because the requested value is negative. |
| `0x80000000` | −1 | `0x80000000` | Both values are −2,147,483,648. Call F-391e3d30e634 with `0x80000000`. |
| `0x7fffffff` | 0 | `0x80000000` | Requested value is positive 2,147,483,648, not negative. Call F-bb73abb30feb; return zero. |
| `0x80000000` | −1 | 0 | Requested value is −4,294,967,296. The stored value is greater, so call F-391e3d30e634 with zero. Its zero result does not trigger F-251588f6a575. |
| Any valid stored word | 1 | 0 | Requested value is 4,294,967,296. Call F-bb73abb30feb; return zero. |
| Any valid stored word | −2,147,483,648 | 0 | Requested value is the signed 64-bit minimum. Call F-391e3d30e634 with zero; no additional positivity-triggered call occurs. |

Additional observable requirements:

- If F-391e3d30e634 returns −7 on a permitted path, the function returns −7 and does not call F-251588f6a575.
- Changing only the auxiliary word does not change this function’s explicit reads, branch arithmetic, or dependency arguments.
- A mutation made by a dependency does not cause the initial numeric comparison to be repeated.

## Uncertainties

### Established static behavior

The supplied evidence establishes the explicit access offsets, numeric widths, signed comparison, call ordering, argument forwarding, and return selection described above.

### Inferred intent

The structure suggests a numeric eligibility check followed by an operation on the low word, with separate context notifications for different outcomes. This is only an interpretation. Terms such as “capacity check,” “allocation,” “cleanup,” and “error handler” are not established and are not part of this contract.

### Documented-only unknowns

The following remain **documented_only**:

- The domain meaning and units of both numeric values.
- The semantic role of the unused auxiliary word.
- Dependency side effects, ownership rules, and global-state preservation.
- Whether dependency results have an application-specific success convention.
- Memory byte order, required alignment, and detailed address-fault behavior.
- Concurrency, atomicity, and synchronization guarantees.
- Behavior if a dependency does not return normally.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

The comparison interpretation follows from the stored word’s sign-derived upper half, the unsigned low-word borrow, and the overflow-corrected upper-half subtraction. The positivity interpretation follows from the corresponding overflow-corrected comparison against zero.

No runtime tests have been performed. No dependency implementation, external interface documentation, or platform ABI evidence was supplied. The acceptance examples are derived obligations, not execution results.
