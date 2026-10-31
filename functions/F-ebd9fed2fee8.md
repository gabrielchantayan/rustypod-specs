<!-- Trusted provenance: F-ebd9fed2fee8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ebd9fed2fee8

## Purpose

F-ebd9fed2fee8 conditionally invokes F-7cb60db72009 and records its 32-bit result in a globally based table-like memory region.

The established selection rule is unusual: the operation proceeds for every negative signed 32-bit selector and for selectors 1 and 2. It does not proceed for selector 0 or any selector greater than or equal to 3. There is no observed lower-bound check.

On the selected path, the function calls the dependency, reads a global base value, and writes three consecutive 32-bit fields at offsets 4, 8, and 12 within the selected record location. The first field receives the dependency’s result; the other two receive zero. The record stride is 276 bytes.

The function returns the 32-bit value 0 after these actions complete normally. On the unselected path, it returns the 32-bit value 9 without calling the dependency or accessing the table.

These return values are established numeric outcomes, not established semantic names such as “success” and “invalid argument.”

## Interface

### Inputs

The function has two explicit inputs:

- **Opaque argument:** A 32-bit value passed unchanged as the first argument to F-7cb60db72009. Its interpretation is unknown. The available evidence does not establish whether it represents an integer, pointer, handle, or encoded value.
- **Selector:** A signed 32-bit integer used both to select the execution path and to calculate a record displacement.

There is no explicit table pointer, capacity, length, or output-buffer argument.

### Implicit state

A global 32-bit base value supplies the base used to calculate the write addresses. It is read only on the selected path, after F-7cb60db72009 has returned.

The available evidence does not establish:

- How the base is initialized.
- What allocation, mapping, or object it identifies.
- How many records are available.
- Whether the base is mutable by other functions or execution contexts.
- Whether negative selectors designate valid storage.

### Output

The function returns a 32-bit value:

| Selector | Return after normal completion | Dependency call and table writes |
|---|---:|---|
| 0 | 9 | None |
| 1 or 2 | 0 | Performed |
| Any negative signed 32-bit value | 0 | Performed |
| Any signed 32-bit value at least 3 | 9 | None |

The dependency’s result is stored in memory rather than returned directly.

### Preconditions for normal selected-path completion

For a selected invocation to complete normally:

- F-7cb60db72009 must return normally.
- The global base must be readable after that return.
- Each of the three calculated 32-bit write locations must be writable under the execution environment’s addressing and alignment rules.

These are environmental requirements, not checks performed by this function. The function does not validate the base, the selector’s relationship to an allocation, or the write destinations.

## Behavior

### Selector decision

The observable branch condition is equivalent to:

- Proceed when the selector is nonzero and is signed-less-than 3.
- Otherwise return 9.

This equivalence accounts for the signed-overflow indication accompanying the 32-bit subtraction used in the supplied representation. It must not be simplified to a test of the wrapped subtraction result’s sign alone.

In particular:

- Zero is excluded.
- The positive values 1 and 2 are included.
- Every negative signed 32-bit value is included, including the minimum representable value.
- Values from 3 through the maximum signed 32-bit value are excluded.

The special treatment of zero is material. A plain signed comparison against 3 would incorrectly include zero.

### Unselected path

For selector 0 or any selector at least 3:

1. No dependency call occurs.
2. The global base is not read.
3. No table fields are written.
4. The function returns the 32-bit value 9.

Consequently, this path does not require a valid table base for its own observed operations. No interpretation of the opaque argument is established on this path.

### Selected path and observable order

For a negative selector, selector 1, or selector 2, the established sequence is:

1. Invoke F-7cb60db72009 with three arguments: the opaque argument unchanged, the selector unchanged, and a 32-bit zero.
2. Retain the dependency’s 32-bit return value.
3. Read the global base.
4. Calculate the selected record location using a 276-byte stride.
5. Write the retained dependency result to the 32-bit field at record offset 4.
6. Write a 32-bit zero to the field at record offset 8.
7. Write a 32-bit zero to the field at record offset 12.
8. Return the 32-bit value 0.

The ordering of the call and base read is significant. A base value observed before the dependency call is not an adequate substitute for the value this function uses. If the dependency changes the global base, the subsequent address calculation uses the changed value.

The record address is calculated before the three stores. The supplied representation does not show a fresh base read between stores. Accordingly, if a store aliases the global base’s storage, that store does not cause the remaining field addresses to be recalculated.

The dependency result is not checked. Every 32-bit result, including zero and values an external convention might designate as errors, is written into the first destination field. Normal completion still returns 0.

### Address calculation and layout

Let the base mean the global value read after the dependency returns. The selected record location is the base plus the selector multiplied by 276 bytes.

Only these fields are directly written:

| Byte offset from selected record location | Width | Written value |
|---:|---:|---|
| 4 | 32 bits | F-7cb60db72009 result |
| 8 | 32 bits | Zero |
| 12 | 32 bits | Zero |

There is no direct write at record offset 0. The function does not clear a complete 276-byte record. Apart from the three specified fields, no additional record writes are shown.

All three stores are full 32-bit stores. Their byte-level encoding depends on the target’s byte order, which is not established here.

The supplied representation uses 32-bit integer quantities for the stride calculation and the address-bearing intermediate. Under that recovered 32-bit model, multiplication and addition retain their low 32 bits. Address calculations therefore must not be treated as unbounded integer arithmetic.

This describes the recovered machine-width behavior, not a claim that overflowing signed arithmetic would be defined in an independently compiled source-language implementation. The underlying mapping and validity of a resulting address remain environmental questions.

### Abnormal completion

There is no observed recovery path for a dependency failure to return normally, an invalid global read, or an invalid destination write.

If an execution environment reports such a failure, the function is not shown converting it into return value 9 or another status. The final return of 0 occurs only after the dependency call and all three stores have completed normally.

The stores are sequential, not an established transaction. If a later store fails, the supplied evidence does not provide rollback of earlier stores or of the dependency’s effects.

## Side effects

### Direct effects

On the selected path, the function directly:

- Reads one global base value after the dependency call.
- Stores the dependency’s result into one 32-bit field.
- Clears two further 32-bit fields.

On the unselected path, none of those accesses occurs.

The function contains no observed allocation, deallocation, ownership transfer, reference-count adjustment, or explicit release operation. Storing the dependency result does not establish whether that value represents owned memory or any other resource.

### Indirect effects

F-7cb60db72009 may have effects not visible in the supplied evidence. It must not be assumed to:

- Preserve globals, including the base.
- Preserve the destination region.
- Allocate or release memory.
- Transfer ownership of its returned value.
- Return normally for every selected input.

The function does not inspect or undo any such effects.

### Aliasing and concurrency

The supplied evidence establishes no disjointness between the destination fields, global storage, and memory used by the dependency.

No locking, atomic operation, or synchronization mechanism is shown. The ordered stores must not be described as an atomic three-field update. Concurrent observation, racing modification, and memory-order guarantees are documented_only.

## Dependencies

### F-7cb60db72009

This dependency is called exactly once on a normally progressing selected invocation and not at all on the unselected path.

Its argument contract as used here is:

| Argument position | Value supplied |
|---:|---|
| 1 | Original opaque 32-bit argument |
| 2 | Original signed 32-bit selector |
| 3 | 32-bit zero |

Its 32-bit result becomes the value written at selected record offset 4.

No further behavioral contract for this dependency is established. In particular, the meaning of its result and its treatment of negative selectors are unknown. Selection by this caller does not prove that the dependency supports those inputs safely.

The global base is a separate implicit dependency. Its initialization, lifetime, allocation bounds, and permitted mutation are not established.

## Acceptance examples

These examples are conditional static acceptance criteria, not runtime observations. Selected-path examples assume the dependency returns normally and all required memory accesses are valid.

### Zero selector

Given selector 0 and any opaque argument:

- Return 9.
- Do not call F-7cb60db72009.
- Do not read the global base.
- Do not perform the three field writes.

A rule that accepts every selector less than 3 would fail this example.

### Positive selected selectors

Given selector 1, dependency result 0x12345678, and post-call base B:

- Call the dependency with the original opaque argument, selector 1, and zero.
- Write 0x12345678 at B + 280.
- Write zero at B + 284.
- Write zero at B + 288.
- Return 0.

For selector 2, the corresponding write displacements are 556, 560, and 564 bytes from the post-call base.

### Rejected upper range

Given selector 3, or any larger signed 32-bit value:

- Return 9.
- Do not call the dependency or access the table.

The maximum signed 32-bit value follows this same path.

### Negative selector

Given selector −1:

- Take the selected path.
- Pass −1 unchanged to the dependency.
- Under ordinary non-wrapping address arithmetic, target fields 272, 268, and 264 bytes before the post-call base.
- Return 0 if the call and stores complete normally.

This example establishes the absence of a negative-selector rejection. It does not establish that those addresses belong to valid records.

### Minimum signed selector

Given selector −2,147,483,648:

- Take the selected path despite overflow in the intermediate subtraction used for the decision.
- Under the recovered 32-bit arithmetic model, multiplying the selector by 276 produces a low-32-bit displacement of zero.
- Consequently, the three field addresses are the post-call base plus 4, 8, and 12, modulo the recovered address width.

This example distinguishes the established signed comparison from an incorrect wrapped-sign-only interpretation.

### Dependency changes the base

Given a selected selector and a dependency that changes the base from one value to another before returning:

- Use the post-call base for all three destinations.
- Do not use the earlier base.
- Do not reload the base between field writes.

### Uninterpreted dependency result

Given a selected selector and dependency result 0xFFFFFFFF:

- Store that exact 32-bit value in the first field.
- Clear the following two fields.
- Return 0 after normal completion.

No result-dependent error branch is established.

## Uncertainties

The following remain documented_only:

- The business purpose of the operation and meanings of return values 0 and 9.
- Whether negative selectors are intentional, erroneous, or meaningful only under hidden caller constraints.
- The opaque argument’s type and ownership implications.
- The dependency’s semantics, resource behavior, and failure modes.
- The table’s allocation bounds, initialization, lifetime, and record meanings.
- The meaning of the untouched field at offset 0 and the rest of each 276-byte record.
- Target byte order, permitted alignment, and concrete address mapping.
- Concurrency guarantees and the externally visible handling of memory-access failures.

A table-record update is an inference from the regular stride and fixed field offsets. It is not evidence of a particular object type or application-level state transition.

## Evidence

This contract is based solely on the supplied static function representation and dependency mapping.

Established static evidence includes the 32-bit values and operations, overflow-aware selector decision, dependency arguments, post-call global-base read, 276-byte stride, three ordered field stores, and final return values.

The branch truth table and acceptance examples are deductions from that evidence. Record terminology is descriptive inferred intent; dependency semantics and environmental safety requirements remain unknown.

No runtime tests have been performed.
