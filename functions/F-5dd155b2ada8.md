<!-- Trusted provenance: F-5dd155b2ada8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5dd155b2ada8

## Purpose

F-5dd155b2ada8 conditionally replaces three adjacent four-byte state fields with a common bit pattern. Its action depends on an integer selector and, for one selector value, the state field at byte offset 8 from a caller-supplied base address.

Two selector values are active:

- Selector `0` writes `0xFFFFFFFB` to all three fields.
- Selector `1` writes `0xFFFFFFFF` to all three fields unless the field at offset 8 matches either of two excluded values, `-2` or `-3`.

Every other selector value causes an immediate return without an explicit state access.

The established behavior is field replacement, not arithmetic adjustment. The supplied evidence does not establish what the fields or negative values mean to the surrounding application. Descriptions such as “reset,” “disable,” or “error state” would be inferred intent, not part of this contract.

## Interface

### Inputs

The function accepts two arguments:

| Argument | Role |
|---|---|
| Base address | Identifies storage containing the affected fields. The supplied representation carries this address in an integer-typed argument. |
| Selector | An integer compared with exactly `1` and `0`. Other values are not accepted as alternate spellings of either active value. |

There is no explicit buffer length, object type, ownership marker, or validity indicator.

The pointer width and address-carrier width are not independently established by the supplied representation. A reimplementation must use an address representation suitable for its environment rather than infer a portable pointer conversion from the displayed integer type.

### Referenced layout

All offsets below are byte offsets relative to the base address.

| Offset | Established use |
|---:|---|
| 4 | Destination of the first four-byte write |
| 8 | Conditional integer read location and destination of the second four-byte write |
| 12 | Destination of the third four-byte write |

The three write regions occupy bytes 4 through 15, inclusive. The function does not explicitly access bytes 0 through 3 or any location beyond byte 15.

Each write is exactly four bytes. The two replacement patterns have these signed 32-bit interpretations:

| Four-byte pattern | Signed 32-bit interpretation |
|---|---:|
| `0xFFFFFFFF` | `-1` |
| `0xFFFFFFFB` | `-5` |

The exclusion comparisons are expressed as signed integer comparisons against `-2` and `-3`. Under a 32-bit interpretation of that field, their patterns are `0xFFFFFFFE` and `0xFFFFFFFD`. Four-byte store width is explicit; the width of the integer type used for the reads is not independently declared in the supplied material.

### Output

There is no return value. Normal completion includes both a successful three-field replacement and an early return that performs no explicit writes.

The function provides no status distinguishing those outcomes.

### Preconditions

For an execution that writes, all three four-byte destinations must be valid and writable. For selector `1`, the field at offset 8 must also be valid for the integer reads shown by the supplied representation.

Alignment requirements, address overflow behavior, and the containing object’s actual size are not established. Valid access to the referenced locations is therefore a caller precondition, not something the function checks.

For selectors other than `0` and `1`, the supplied body does not dereference the base address. That observation does not establish any broader calling-convention or language-level guarantee for arbitrary address values.

## Behavior

### Selector `0`

Selector `0` bypasses the exclusion checks. The existing contents of the three destination fields do not influence the replacement pattern.

The function writes `0xFFFFFFFB` in this order:

1. Four bytes at offset 4.
2. Four bytes at offset 8.
3. Four bytes at offset 12.

It then returns.

In particular, values `-2` and `-3` at offset 8 do not prevent replacement on this path. Their exclusion applies only to selector `1`.

### Selector `1`

Selector `1` first examines the integer field at offset 8.

The exclusion condition uses short-circuit evaluation:

- The first comparison checks whether the field equals `-2`.
- If that comparison is true, the function returns without writing.
- If that comparison is false, the field expression is evaluated for a comparison against `-3`.
- If the second comparison is true, the function returns without writing.
- Otherwise, the function proceeds to replacement.

On the replacement path, it writes `0xFFFFFFFF` in this order:

1. Four bytes at offset 4.
2. Four bytes at offset 8.
3. Four bytes at offset 12.

It then returns.

The supplied expression contains two field-read evaluations when the first comparison is false. This contract does not convert those evaluations into a guarantee of two physical memory transactions: volatility, optimization constraints, and generated instructions are not provided.

### Other selectors

Any selector other than exactly `0` or exactly `1` causes a return without explicit reads or writes to the referenced storage.

The selector is not treated as a Boolean. In particular, a nonzero value other than `1` does not select the `0xFFFFFFFF` replacement.

### Read and write ordering

For selector `1`, the exclusion evaluation precedes every explicit write. If an exclusion comparison succeeds, none of the three writes is reached.

Once replacement begins, the explicit write order is offset 4, then offset 8, then offset 12. The previous contents of offsets 4 and 12 are not read by the supplied body.

These are source-level access-order properties. The evidence does not establish atomicity, barriers, inter-thread visibility, or transactional behavior. It also does not establish rollback if a memory access fails after an earlier write has completed.

## Side effects

The only explicit externally visible state changes are the three four-byte writes on a replacement path.

An excluded selector-`1` execution leaves all three destination fields unchanged by this function. An unsupported selector likewise performs no explicit modification.

The function does not explicitly:

- Allocate or release memory.
- Transfer ownership of the addressed storage.
- Retain the base address.
- Call another function.
- Access a named global object.
- Log, notify, or return an error code.

These statements describe the supplied body, not a guarantee about operating-system reactions to invalid memory accesses or about external agents sharing the same storage.

The function is not a read-modify-write operation on the destination values. It does not preserve selected bits or combine old state with new state; it replaces each destination with a fixed pattern.

For ordinary stable memory, repeating a completed replacement with the same active selector leaves the same final field values. This final-value property does not imply that repeated calls omit their writes.

## Dependencies

No callable dependencies are present in the supplied body. Consequently, there are no dependency function IDs to enumerate beyond F-5dd155b2ada8 itself.

Execution relies on the surrounding environment for valid address interpretation and readable or writable storage. Those requirements do not establish any allocator, object-lifetime manager, synchronization service, or exception handler.

No assumptions about an unknown callee preserving globals or releasing memory are needed because the supplied body contains no calls.

## Acceptance examples

These examples are conditional specification cases, not runtime observations. Unless stated otherwise, they assume valid, ordinarily accessible storage, no concurrent modification, and the conventional 32-bit interpretation of the guard field.

1. **Selector `0` replaces ordinary values.**  
   Given selector `0` and arbitrary initial contents in the three destination fields, completion leaves each field holding `0xFFFFFFFB`. Explicit writes occur at offsets 4, 8, and 12, in that order. No existing destination value is consulted.

2. **Selector `0` overrides an exclusion value.**  
   Given selector `0` and `-2` at offset 8, all three destination fields still become `0xFFFFFFFB`. The same outcome applies when offset 8 initially holds `-3`.

3. **Selector `1` excludes `-2`.**  
   Given selector `1` and a stable value of `-2` at offset 8, the first comparison succeeds. No explicit writes occur, and the second comparison is not evaluated.

4. **Selector `1` excludes `-3`.**  
   Given selector `1` and a stable value of `-3` at offset 8, the first comparison fails and the second succeeds. All three destination fields remain unchanged.

5. **Selector `1` replaces a nonexcluded value.**  
   Given selector `1` and a stable value of `0`, `-1`, or `-5` at offset 8, neither exclusion comparison succeeds. Each destination becomes `0xFFFFFFFF`, in the established write order.

6. **Other selectors do nothing explicitly.**  
   Given selector `2` or `-1`, the function returns without explicitly reading or writing the referenced storage. Initial field contents do not affect this outcome.

7. **An already matching state does not suppress replacement.**  
   Given selector `1` and `0xFFFFFFFF` already present in all three fields, the exclusion condition does not match. The function still reaches all three writes, although their final values remain unchanged.

## Uncertainties

The following matters are **documented_only**, not requirements filled in by assumption:

- **Field meanings:** No application-level interpretation of the three fields or sentinel values is established.
- **Guard-read width:** The reads use an integer-typed access, but the supplied material does not independently declare that type’s width. The four-byte write width is explicit.
- **Address model:** Native pointer width, integer-to-address rules, overflow behavior, and required alignment are unspecified.
- **Byte order:** The stored four-byte patterns are established; their byte ordering in memory is not.
- **Concurrency:** There is no synchronization contract. Behavior under concurrent modification cannot safely be reduced to a single stable guard value.
- **Physical memory transactions:** Logical read evaluations and explicit write order do not establish exact machine transactions or hardware visibility.
- **Invalid accesses:** No checked error path, recovery behavior, or partial-update guarantee is provided.

## Evidence

Established static evidence consists of the supplied body of F-5dd155b2ada8: two exact selector comparisons, a short-circuit exclusion condition reading offset 8, two fixed four-byte replacement patterns, and three ordered writes at offsets 4, 8, and 12.

The body contains no explicit calls, allocation, deallocation, validation, or returned result. These absences support the limited side-effect and dependency descriptions above.

The interpretation of the routine as a state-setting operation is inferred intent. Application semantics, platform details, and concurrency guarantees remain unknown.

No runtime tests have been performed. The acceptance examples describe conditional expected behavior derived from static evidence, not measured results.
