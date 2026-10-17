<!-- Trusted provenance: F-b767aed951aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b767aed951aa

## Purpose

F-b767aed951aa replaces the low 24 numerical bits of a caller-provided unsigned integer object while retaining that object’s previous numerical bits 24 through 31.

For a 32-bit unsigned object, the result consists of the replacement input’s low 24 bits and the destination’s previous upper eight bits. Replacement zeros clear corresponding destination bits; the operation is replacement, not accumulation.

The supplied source does not define the unsigned integer type’s width. The 32-bit interpretation is therefore conditional. If the type is wider than 32 bits, destination bits above position 31 are cleared, not preserved. If it is narrower, only representable bit positions can appear in the stored result.

The bit fields have no established application meaning. Interpreting them as a payload and a separate attribute is plausible intent, not part of the established contract.

## Interface

### Inputs and output

The function accepts two inputs:

- **Destination:** a pointer to an unsigned integer object, used both to obtain its previous value and to store its replacement value.
- **Replacement:** an unsigned integer supplied by value. Only its numerical bits 0 through 23 can contribute to the result.

Both inputs use the same underlying unsigned integer type: the destination points to an object of that type, and the replacement is a value of that type. Its definition, width, and storage representation are not supplied.

The function returns no value. It exposes no status code or separate result object. The result is the assignment to the destination.

For a 32-bit type, the resulting numerical layout is:

| Bit positions | Width | Source |
|---|---:|---|
| 0–23 | 24 bits | Corresponding replacement bits |
| 24–31 | 8 bits | Corresponding previous destination bits |

Bit position zero is the least-significant bit. This description does not establish byte order or memory layout.

### Preconditions

Defined execution requires the destination pointer to designate an object that:

- Remains live throughout the operation.
- Is legally readable and writable through the supplied pointer type.
- Meets the applicable alignment requirements.
- Contains a value that may legally be read.
- Is accessed under the synchronization rules required by the execution environment.

The function visibly performs no validation of these conditions. A null or otherwise invalid destination is not accepted through a separate error path.

The replacement need not fit within 24 bits. Higher bits are excluded from the result rather than rejected.

The destination and the storage from which the caller obtained the replacement need not be distinct. The supplied interface imposes no non-aliasing requirement between them.

### Ownership and aliasing

The destination remains caller-provided storage. The visible body does not allocate or release storage, retain the pointer for later use, or extend the destination object’s lifetime.

Passing the replacement by value provides no access path to the storage from which that value originated. This does **not** establish that the originating storage is inaccessible through another input. The independently supplied destination pointer may designate that same object.

Consequently, a caller may supply an object as the destination and also supply that object’s current value as the replacement. The function then assigns to the original object through the destination pointer.

For a 32-bit type, such a self-source invocation preserves the object’s numerical value, assuming no intervening modification. For a wider type, it can change the value by clearing bits above position 31. Passing the replacement by value therefore does not imply that its originating object remains unchanged.

## Behavior

### Resulting bit fields

For a 32-bit unsigned destination:

- Each resulting bit in positions 0 through 23 equals the corresponding bit of the replacement value.
- Each resulting bit in positions 24 through 31 equals the corresponding bit of the destination value read by the function.

The previous low field contributes nothing independently of the replacement. A replacement zero clears the corresponding low destination bit; a replacement one sets it.

Replacement bits at position 24 or above cannot affect the retained field. The function does not shift the replacement into another position.

There is no addition or subtraction. No carry or borrow can cross the boundary between positions 23 and 24.

The same transformation applies to all input values. There are no visible value-dependent branches for zero, all-one inputs, equal values, or unchanged results.

### Source-level access and ordering

The function obtains the destination’s previous value, uses its selected bits in forming the result, and assigns the result to that same destination.

The destination value computation precedes the assignment that depends on it. The source does not establish a meaningful ordering between independent evaluation of the replacement contribution and the destination contribution.

The assignment is unconditional. There is no comparison that bypasses it when the resulting value equals the previous destination value.

These are source-level observations, not guarantees about generated memory transactions. They do not establish:

- A particular number of machine loads or stores.
- A particular physical access width.
- An indivisible read-modify-write operation.
- A distinct externally observable write when the numerical value is unchanged.

The supplied destination declaration is not explicitly volatile or atomic. No additional device-access or concurrency semantics can be inferred from it.

### By-value replacement and shared originating storage

The replacement is a value supplied to the invocation, not a request to reread the caller’s originating object while computing the result.

If the destination designates that originating object, the destination read and assignment still occur through the destination pointer. Aliasing does not create a second access path through the by-value input.

If the object remains unchanged between obtaining the replacement value and the function’s destination read, both inputs represent the same numerical value. Under that condition:

- With a 32-bit type, the assignment reconstructs that value unchanged.
- With a wider type, positions 0 through 31 remain as supplied, and positions above 31 become zero.

If the replacement value and the subsequently read destination value differ under an otherwise valid execution, the ordinary field rule applies: low bits come from the supplied replacement value, while positions 24 through 31 come from the destination read. The function does not require the two input values to agree.

### Widths and conversions

The selected masks reach numerical bit position 31. This establishes the selected numerical ranges, not the width of the underlying unsigned type.

For a type exactly 32 bits wide, every destination bit belongs to one of the two selected fields.

For a type wider than 32 bits:

- Positions 0 through 23 come from the replacement.
- Positions 24 through 31 come from the destination read.
- All positions above 31 become zero.

The retained field is therefore a fixed numerical range. It is not the highest byte of an arbitrarily wide object.

For a type narrower than 32 bits, only positions representable by that type survive storage:

- Existing positions below 24 come from the replacement.
- Any existing positions from 24 through 31 come from the destination read.
- There are no stored positions outside the type’s width.

These narrower- and wider-type results are conditional consequences of the unsigned type interpretation and applicable integer conversions. They are not evidence that either width is used in the actual environment.

The operation requires no overflow-dependent arithmetic explanation. Concrete ABI widths, integer promotions, padding, and physical representation remain external properties.

### Repeated invocations

With no intervening modification and a fixed type interpretation, repeating the same replacement leaves the destination’s numerical value unchanged after the first invocation.

For a 32-bit destination, successive invocations preserve the same upper eight bits while the low 24 bits reflect the latest replacement alone.

For a wider destination, the first invocation also clears all bits above position 31. Subsequent invocations do not restore them.

These numerical properties do not establish whether a separate machine store occurs on each invocation.

## Side effects

The sole explicit state change is assignment to the destination object. If that object is also the caller’s source of the replacement value, the assignment affects that same storage; by-value argument passing does not prevent this aliasing effect.

The visible body contains no function calls, allocation, deallocation, explicit global-state access, logging, I/O, locking, or explicit memory-order operation. It does not explicitly retain the destination pointer.

There is no error-reporting branch. Invalid access is not converted into a defined status, diagnostic, or fallback result. For a null, dangling, misaligned, inaccessible, or otherwise invalid destination, this contract promises neither a resulting value nor a particular fault.

The operation does not establish atomic preservation of the retained field against another writer. A caller cannot infer that an update made by another actor between the destination read and assignment will survive. Whether a concurrent execution is legal at all depends on external synchronization and language rules.

Likewise, the source does not establish a memory-mapped-device access protocol. Source-level assignment alone does not specify device transactions or their granularity.

## Dependencies

There are no visible called-function dependencies. No dependency function IDs beyond F-b767aed951aa are needed.

The contract relies on execution-environment properties not defined by the supplied body:

- The unsigned integer type’s definition, value width, and representation.
- Integer-promotion and conversion rules.
- Typed-access, alignment, and object-lifetime requirements.
- Synchronization requirements.
- Compiler and target rules governing physical accesses.

No unknown callee is assumed to preserve globals, retain pointers, or own or release memory: no callee appears in the supplied implementation.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless otherwise stated, each example assumes a 32-bit unsigned type, valid readable and writable destination storage, and no intervening or concurrent modification.

| Previous destination | Replacement | Expected destination after assignment | Acceptance condition |
|---|---|---|---|
| 0xAB123456 | 0x00FEDCBA | 0xABFEDCBA | Low field replaced; upper field retained |
| 0xAB123456 | 0x98FEDCBA | 0xABFEDCBA | Replacement bits above 23 have no effect |
| 0xABFFFFFF | 0x00000000 | 0xAB000000 | Replacement zeros clear old low bits |
| 0x12000000 | 0xFFFFFFFF | 0x12FFFFFF | Every low-field bit can be set |
| 0x00123456 | 0xFF000001 | 0x00000001 | Replacement cannot set the retained field |
| 0xFFFFFFFF | 0xFFFFFFFF | 0xFFFFFFFF | Assignment may leave the numerical value unchanged |

Additional conditional cases:

- **Ignored replacement field:** From destination 0x7E123456, replacements 0x00123456 and 0xFF123456 both produce 0x7E123456.
- **Successive replacements:** From destination 0xAB000000, replacement 0x00112233 followed by replacement 0x00445566 produces 0xAB445566. No low bits from the first replacement independently survive.
- **Repeated replacement:** Applying replacement 0x00445566 to destination 0xAB445566 leaves its numerical value unchanged. This does not establish the presence or absence of a distinct physical store.
- **Aliased originating storage, 32-bit type:** An object contains 0xAB123456. The destination designates that object, and the replacement is its current value. The function assigns to the object, whose resulting value remains 0xAB123456.
- **Wider destination:** If external type information establishes a 64-bit unsigned type, destination 0x11223344AB123456 with replacement 0x0000000098FEDCBA becomes 0x00000000ABFEDCBA.
- **Aliased originating storage, 64-bit type:** An object contains 0x11223344AB123456. The destination designates that object, and the replacement is its current value. Its resulting value is 0x00000000AB123456. The originating object changes despite the replacement being passed by value.
- **Narrow destination:** If external type information establishes a 16-bit unsigned type, destination 0xABCD with replacement 0x1234 becomes 0x1234. No representable destination bits lie in the retained range.

No acceptance case promises a recoverable error or defined destination value for an invalid pointer.

## Uncertainties

### Established static evidence

The supplied body establishes:

- One destination value read in the source expression.
- Selection of replacement bits 0 through 23.
- Retention of previous destination bits 24 through 31.
- An unconditional assignment to the destination.
- No returned value and no visible function calls.

The interface provides no non-aliasing restriction that would exclude the destination from designating the replacement’s originating object.

### Inferred intent

The selected ranges suggest a 24-bit field adjacent to an eight-bit field in a 32-bit word. The source does not establish what either field means, or that 32 bits is the actual underlying width.

### documented_only

The supplied material does not establish:

- The underlying unsigned type definition or ABI width.
- Byte order, addressable-unit size, padding, or machine access granularity.
- Application meanings for the selected fields.
- Which objects actual callers supply.
- Whether actual callers use shared originating storage.
- Any atomicity, concurrency, or device-memory protocol.
- Behavior outside valid memory-access preconditions.
- Generated instructions or optimization decisions.

These remain unresolved rather than being filled in from a familiar platform or application.

## Evidence

This contract derives solely from static examination of the supplied body and declaration for F-b767aed951aa. The source establishes the selected numerical fields and the destination read-to-assignment relationship.

The aliasing account follows from the independent pointer and by-value inputs: obtaining a value does not itself provide access to its originating storage, but the destination pointer may separately designate that storage.

No runtime tests, compilation, external implementation inspection, or generated-instruction inspection have been performed. All acceptance examples are deductions under their stated width, validity, and stability assumptions.
