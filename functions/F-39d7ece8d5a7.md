<!-- Trusted provenance: F-39d7ece8d5a7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-39d7ece8d5a7

## Purpose

F-39d7ece8d5a7 conditionally produces a 32-bit integer result from an operation code and a selector. It returns a separate 32-bit status indicating whether it handled that combination.

Four operation codes are recognized: 0x8401, 0x8402, 0x8403, and 0x8404. Handling also requires the selector to match one of two values read from external storage. A match against the first selector value always produces zero. A match against the second selector value produces an operation-specific result, potentially involving an object field or dependency F-27c28941448c.

The established behavior is dispatch and result production. The application-level meanings of the operation codes, selector values, object fields, and resulting integers are unknown. In particular, a result of zero is not the same as an unhandled request: successful handling can produce zero.

## Interface

### Inputs

The function has four arguments:

| Argument | Interpretation |
|---|---|
| Object address | Address used as the base for conditional field reads. |
| Selector | A 32-bit value compared for equality against two externally stored selector values. |
| Operation | A 32-bit operation code. |
| Result address | Address at which a handled request stores one 32-bit integer. |

The supplied representation uses 32-bit integer and word operations. The object address is represented through an integer-like argument, while the result address is represented as an integer pointer. This establishes the accessed data widths for this contract, but does not independently establish a portable host ABI or native pointer width.

Two external storage locations participate in dispatch:

- **Primary selector cell:** the value tested first for each recognized operation.
- **Secondary selector cell:** the value tested only when the primary comparison does not match.

These names identify their roles in this contract, not source identifiers or application semantics. Their concrete values are unavailable.

### Object layout accessed

Only the following object-relative locations are directly accessed:

| Byte offset | Access width | Condition for access |
|---|---:|---|
| 0x18 | 4 bytes | Operation 0x8403 or 0x8404, with a secondary-selector match after a primary-selector mismatch. |
| 0x1C | 1 byte | Operation 0x8401, with a secondary-selector match after a primary-selector mismatch. |

The byte at offset 0x1C is tested only for zero versus nonzero. Its signedness does not affect this decision.

The four-byte value at offset 0x18 is passed to F-27c28941448c. Its meaning is unknown; it must not be assumed to be a count, pointer, handle, or owned allocation.

No minimum complete object size, additional fields, or object type is established beyond the storage needed for the accesses actually taken.

### Outputs

The return value is a 32-bit status:

- **1:** the request was handled, and the function performed a four-byte result store on normal completion.
- **0:** the request was not handled, and the function performed no result store.

For a handled request, the stored value is one of:

- Zero.
- One.
- The 32-bit result of F-27c28941448c decremented by one.

The output address need not refer to initialized data: this function does not directly read the previous result value.

### Preconditions

Storage must be valid for every access reached by the selected path:

- Recognized operations require readable selector cells.
- Every successful path requires a writable four-byte result destination.
- The object-byte path requires a readable byte at object offset 0x1C.
- The dependency path requires a readable four-byte field at object offset 0x18 and satisfaction of any requirements imposed by F-27c28941448c.

These requirements are path-dependent. An unsupported operation does not directly access the selector cells, object, or result destination. A recognized operation with neither selector match does not directly access the object or result destination.

No explicit checks for invalid addresses, alignment, or dependency-specific validity are established.

## Behavior

### Dispatch and precedence

An operation outside the four recognized codes returns zero without a direct memory access to either selector cell, the object, or the result destination.

For each recognized operation, the function first reads the primary selector cell and compares its value with the selector argument.

If that comparison matches:

1. The function stores zero at the result address.
2. The function returns one.

On this path it does not directly read the secondary selector cell or either object field, and it does not call F-27c28941448c.

If the primary comparison does not match, the function reads the secondary selector cell and compares that value with the selector argument. If this comparison also fails, it returns zero without a result store.

Primary matching therefore has precedence. If the two selector cells contain equal values and the selector argument matches them, the result is zero for every recognized operation. The secondary behavior is not also performed.

### Secondary-selector behavior

After a primary mismatch and a secondary match, behavior is:

| Operation | Additional direct action | Stored result | Status |
|---|---|---:|---:|
| 0x8401 | Read the byte at object offset 0x1C. | Zero if the byte is zero; otherwise one. | 1 |
| 0x8402 | No object access or dependency call. | One. | 1 |
| 0x8403 | Read the four-byte field at offset 0x18 and call F-27c28941448c with that value. | Dependency result minus one. | 1 |
| 0x8404 | Same dependency path as 0x8403. | Dependency result minus one. | 1 |

For operation 0x8401, all nonzero byte values produce the same result of one. The original byte value is not copied to the destination.

Operations 0x8403 and 0x8404 have identical established behavior. No operation-specific difference in dependency arguments or output adjustment is visible.

### Observable access order

The direct read, call, and write order is significant:

- **Primary match:** primary selector read, result store, return.
- **Neither selector matches:** primary selector read, secondary selector read, return without a result store.
- **Secondary match for 0x8401:** primary selector read, secondary selector read, object-byte read, result store, return.
- **Secondary match for 0x8402:** primary selector read, secondary selector read, result store, return.
- **Secondary match for 0x8403 or 0x8404:** primary selector read, secondary selector read, object-word read, dependency invocation and normal return, decrement, result store, return.

These are the accesses established by the supplied representation. They are not a concurrency, atomicity, or memory-ordering guarantee.

The wrapper does not re-read selector cells after invoking the dependency. It also does not re-read the object field before storing the adjusted result.

### Integer arithmetic

The dependency result is adjusted by subtracting one in a 32-bit result domain. There is no visible range check, saturation, clamping, or rejection of zero or negative results.

Under a 32-bit machine-word interpretation:

- A dependency result of 1 produces 0.
- A dependency result of 0 produces the all-ones word, conventionally interpreted as signed −1.
- A dependency result with bit pattern 0x80000000 produces 0x7FFFFFFF after decrement.

The supplied representation does not establish a portable source-language rule for signed overflow. The boundary example describes the machine-word interpretation, not a claim that signed overflow is defined in every possible reimplementation language.

Recognition of operation 0x8404 is expressed through an arithmetic comparison in the supplied representation. In the 32-bit word domain, it identifies 0x8404; it does not establish a wider family of recognized operation values.

## Side effects

### Direct effects

On a normally completed handled path, the function directly writes exactly one four-byte result. It does not directly modify either object field or either selector cell.

On an unhandled path, it performs no direct write and makes no dependency call.

No direct allocation, release, ownership transfer, resource registration, or cleanup operation is visible.

### Dependency effects

For the secondary-selector paths of operations 0x8403 and 0x8404, F-27c28941448c executes before the wrapper’s result store.

Its effects are unknown. It must not be assumed to preserve globals, object storage, the result destination, or resource ownership. If it modifies the result destination through some independently available reference, the wrapper’s subsequent four-byte store overwrites that destination on normal return.

If the dependency does not return normally, the wrapper’s final result store and success return are not established.

### Aliasing

No non-aliasing requirement or alias detection is visible.

If the result destination overlaps object storage, the required object read occurs before the result store. The store can therefore alter that storage after its value has already been consumed.

If the result destination overlaps a selector cell, selection occurs before the result store. The current invocation follows the values already read; the write may affect later invocations.

Such cases still require valid underlying accesses. The supplied representation does not establish that every possible overlap is valid under an implementation language’s alignment and aliasing rules.

### Errors

The only explicit failure status is zero for an unsupported operation or unmatched selector.

There is no separate explicit status for an invalid address, invalid object contents, an unsuitable dependency argument, or dependency failure. No fault recovery, rollback, or cleanup behavior is shown. A negative dependency-derived result is not itself treated as failure: on normal completion, the wrapper still returns one.

## Dependencies

### F-27c28941448c

**Invocation condition:** operation 0x8403 or 0x8404, a primary-selector mismatch, and a secondary-selector match.

**Input:** one four-byte value read from object offset 0x18.

**Result use:** the returned 32-bit value is decremented by one and stored at the result address.

**Call count:** exactly one invocation on that path; none on other paths.

**Unknown contract:** accepted argument values, semantic meaning, errors, global accesses, mutation, allocation, ownership effects, and exceptional or nonreturning behavior.

### External selector storage

Two readable selector cells determine whether recognized operations are handled. Their initialization, mutability, synchronization, lifetime, and application meanings are unknown.

The function reads them in precedence order rather than establishing a simultaneous snapshot. No atomic relationship between the two reads is established.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless otherwise stated, all reached accesses are valid and any dependency call returns normally.

1. **Unsupported operation:** With operation 0x8400, the status is zero. The function does not directly read the selector cells or object and does not write the result destination.

2. **Recognized operation with no selector match:** For any recognized operation, if the selector differs from both values as read, the status is zero and an initial destination value remains unchanged by this function.

3. **Primary selector match:** For any recognized operation, a primary match produces status one and result zero. No object read or dependency call occurs.

4. **Equal selector values:** If both cells contain the same value and the selector matches it, operation 0x8403 produces zero without calling F-27c28941448c. Primary precedence controls the outcome.

5. **Zero object byte:** With operation 0x8401, primary mismatch, secondary match, and a zero byte at offset 0x1C, the result is zero and status is one.

6. **Nonzero object byte:** Under the same selection conditions, byte values 0x01, 0x80, and 0xFF each produce result one.

7. **Constant secondary result:** With operation 0x8402, primary mismatch, and secondary match, the result is one without an object read.

8. **Dependency-derived result:** With operation 0x8403 or 0x8404 and secondary selection, a dependency result of 7 produces result 6 and status one.

9. **Dependency result zero:** On that same path, a dependency result of zero produces the 32-bit all-ones result under machine-word arithmetic, while status remains one.

10. **Dependency changes a selector:** If the dependency changes a selector cell before returning, the wrapper does not repeat selection. It stores the adjusted dependency result and returns one.

11. **Dependency does not return normally:** No completed result store or success return from the wrapper is required by the established normal-return behavior.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the operation codes, selectors, fields, and result values.
- Concrete selector values and whether they can change between reads.
- Full object layout, native ABI, pointer provenance, alignment requirements, and byte order.
- The complete contract and side effects of F-27c28941448c.
- Portable source-language behavior at signed arithmetic boundaries.
- Fault, interruption, exception, and concurrency behavior.
- Whether aliasing examples occur in intended use.

**Inferred intent:** the zero/nonzero byte mapping resembles Boolean normalization, and decrementing the dependency result may translate between value conventions. Neither interpretation is required by the established behavior.

## Evidence

The behavioral contract is based solely on static inspection of the supplied restricted representation of F-39d7ece8d5a7.

Established evidence includes four recognized operation codes, ordered selector comparisons, conditional one-byte and four-byte object reads, one dependency call path, a decrement of its returned value, one four-byte store on successful normal completion, and status values limited to zero and one.

No runtime tests have been performed. The acceptance examples describe conditional obligations, not observed executions.
