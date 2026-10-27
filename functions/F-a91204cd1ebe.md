<!-- Trusted provenance: F-a91204cd1ebe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a91204cd1ebe

## Purpose

F-a91204cd1ebe writes two supplied signed integer values into caller-provided storage in ascending order.

For inputs described here as **A** and **B**, the resulting pair is:

- First element: the smaller of A and B.
- Second element: the larger of A and B.

Equal inputs produce two equal output elements. Negative values participate in ordinary signed ordering, not unsigned ordering or ordering by magnitude.

This ordering behavior is established by the supplied static source. A broader purpose, such as normalizing an interval or preparing bounds for another operation, is not established.

## Interface

The function has three inputs:

| Input | Role | Contract |
|---|---|---|
| Destination | Pointer to writable integer storage | Must permit writing two consecutive elements of the function’s integer type. |
| A | First integer value | Passed by value and compared using signed integer ordering. |
| B | Second integer value | Passed by value and compared using signed integer ordering. |

The function returns no value. Its result is communicated through the destination.

### Integer width and representation

Both value inputs and both destination elements use the same signed integer type. The supplied source identifies that type as `int`, but does not independently establish its bit width, byte size, alignment, or object representation.

Accordingly:

- Let **W** denote the target’s width for this signed integer type.
- Each destination element occupies the target’s size for that type.
- The second destination element immediately follows the first according to integer-array element spacing.
- The destination must provide space for two such elements, not merely two bytes.
- Endianness does not affect the mathematical ordering contract, although it affects the byte representation of the stored values.

A 32-bit interpretation is conditional on an external target or ABI declaration. Under that interpretation, A and B range from −2,147,483,648 through 2,147,483,647, and, where the integer type occupies four bytes, the output occupies eight bytes. Those concrete dimensions are not independently established by this source alone.

### Preconditions

For the defined behavioral contract:

- The destination must be non-null.
- It must be correctly aligned for the integer type.
- Both destination elements must be accessible for writing.
- Both writes must be permitted under the applicable object-lifetime, type-access, and memory-access rules.
- Concurrent access must not create an unsynchronized conflicting access.

The previous contents of the destination do not matter. The function does not need either output element to contain an initialized input value.

A and B are already supplied by value when the function executes. They are not fetched through the destination.

## Behavior

### Result relationship

After normal completion:

- The first destination element equals the minimum of A and B.
- The second destination element equals the maximum of A and B.
- The first element is less than or equal to the second.
- The output pair preserves the two input values, including their multiplicity.

The function does not add, subtract, negate, multiply, divide, narrow, or otherwise transform the integer values. It selects and copies them.

As a result, minimum and maximum representable integers require no special arithmetic handling. There is no subtraction-based comparison that could overflow when the inputs are far apart.

### Equal inputs

When A equals B, both output elements receive that common value.

At the source-selection level, equality selects the value supplied as A for both stores. Since the inputs are ordinary integer values of the same type, this does not produce a different numerical result from copying B. It does not establish preservation of distinct object representations, padding bits, or caller-side storage identity.

### Reads and writes

The supplied source establishes the following sequencing:

1. The first result is selected from the supplied integer values.
2. The first destination element is written.
3. The second result is selected from the supplied integer values.
4. The second destination element is written.

Thus, the first destination assignment precedes the second destination assignment in source-level execution.

Neither destination element is read to calculate either result. In particular, the second result is not derived from the newly written first element.

The function retains local copies of the input values. Writing the destination therefore does not change the values used for the subsequent selection. If the destination points to caller storage from which an argument was originally obtained, that storage may be overwritten, but the already-passed argument value remains the value used by this call.

The source-level order does not establish an instruction-level trace, atomic publication of the pair, memory barriers, or the visibility order perceived by an unsynchronized observer.

### Completion and errors

For valid inputs and valid destination storage, the function performs both assignments and returns normally without a result value.

There is no explicit:

- Null-pointer check.
- Destination-length check.
- Alignment check.
- Error code.
- Exception-handling path.
- Recovery or rollback path.

Invalid destination storage is outside the defined contract. A crash, trap, silent corruption, or another particular failure response must not be promised.

The operation is not transactional. If an execution environment permits the first write and then faults on the second, the source contains no mechanism to undo the first write. Behavior under invalid memory access remains outside the normal contract.

## Side effects

The intended externally visible state change is replacement of the two destination elements.

For a valid destination:

- The first element is overwritten with the lower value.
- The second element is overwritten with the higher value.
- No additional destination element is explicitly written.
- No destination contents are preserved as part of the result.

The destination remains caller-owned. The function neither takes ownership nor returns a separately allocated object.

The supplied source contains no allocation, deallocation, persistent pointer retention, global-variable access, input/output operation, or external call. Its temporary storage is local to the invocation.

These observations do not prove that arbitrary pointer values are harmless. A pointer into protected storage, device memory, or storage with access side effects requires an additional environment-specific contract. The ordinary writable-memory contract does not establish supported behavior for such uses.

## Dependencies

No dependency function is invoked by F-a91204cd1ebe in the supplied source.

Its behavior depends on language and target rules for:

- Signed integer comparison.
- Copying integer values.
- Addressing consecutive integer elements.
- Writing through the supplied pointer.
- Sequencing the two destination assignments.

There are no unknown callees whose ownership rules, global-state effects, or memory-release behavior must be assumed.

Compiler-generated helpers or instrumentation are not established by the supplied source and are not part of the documented dependency interface.

## Acceptance examples

These examples specify expected behavior; they are not reports of executed tests. Each assumes valid writable storage for two elements of the applicable integer type.

| A | B | First output element | Second output element |
|---:|---:|---:|---:|
| 3 | 8 | 3 | 8 |
| 8 | 3 | 3 | 8 |
| 5 | 5 | 5 | 5 |
| −7 | 2 | −7 | 2 |
| −2 | −9 | −9 | −2 |
| 0 | 0 | 0 | 0 |

### Conditional boundary example

If the target uses 32-bit signed integers with the range stated above:

- A = 2,147,483,647.
- B = −2,147,483,648.
- The first output element must be −2,147,483,648.
- The second output element must be 2,147,483,647.

This outcome follows from signed comparison and copying; no intermediate difference is required.

For a different integer width, the corresponding minimum and maximum representable values must obey the same ordering relationship.

### Existing destination contents

If the two destination elements initially contain 90 and 40, and A and B are 6 and 1, the final pair must be 1 and 6. The previous values must not affect selection.

### Destination backed by a larger array

If the destination points to an element of a larger valid integer array and there is room for the following element, those two elements receive the ordered pair. Other elements are not explicitly written by this function.

### Caller storage used to obtain an argument

If a caller obtains A from storage that is also the first destination element, the result must still use the value passed as A. Replacing that element during the call does not make the second result use the replacement as a new input.

### Insufficient storage

If only one writable integer element is available, the call violates the destination precondition. The function does not promise to detect or reject that condition, and no successful two-element result is specified.

## Uncertainties

The following matters are **documented_only**:

- **Concrete integer width and layout:** Exact width, byte size, alignment, padding, and representation require target information not supplied here.
- **Machine-level access trace:** Source-level assignment order does not establish instruction count, store granularity, atomicity, or externally observed hardware ordering.
- **Invalid-access outcomes:** The source does not define a reliable failure mode for null, misaligned, inaccessible, or undersized destinations.
- **Special memory regions:** Device registers and other access-sensitive memory are not covered by the ordinary-memory assumptions.
- **Calling convention:** Register assignments, stack layout, binary linkage, and ABI-specific return conventions are not established.
- **Application intent:** Interpreting the output as an interval, range, or other domain object is an inference rather than a demonstrated requirement.

No runtime tests have been performed. The acceptance examples remain conditional expectations derived from static evidence.

## Evidence

The supplied source establishes:

- One pointer input and two by-value signed integer inputs.
- Local retention of the supplied integer values.
- Signed comparisons that select the lower value for the first destination element and the higher value for the second.
- Selection of A on equality for both assignments.
- A first-element write preceding a second-element write.
- No destination read used to compute the results.
- No arithmetic transformation of the input values.
- No explicit validation, external dependency call, allocation, deallocation, or global-state access.
- Normal return without a value.

The interpretation as an ascending two-value ordering operation follows directly from those selections. Concrete ABI details, runtime fault behavior, concurrency guarantees, and broader application intent are not established by this evidence.
