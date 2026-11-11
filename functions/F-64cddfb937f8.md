<!-- Trusted provenance: F-64cddfb937f8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-64cddfb937f8

## Purpose

Construct and register a composite object whose subordinate components are selected by two configuration masks. Publish the object through shared state early in construction, populate two indexed component groups, invoke three final registration operations, and return the object pointer.

The established behavior is structural. The supplied material does not establish the application-level meaning of the object, its configuration values, its components, or its registrations. Describing these operations as construction and registration is an inference from their use, not an independently verified dependency contract.

No runtime tests have been performed.

## Interface

### Inputs

The interface has four arguments under the supplied 32-bit representation:

| Argument | Width | Established use |
|---|---:|---|
| First | 32 bits | Its value plus 4 is passed to F-e3199032a3a0. |
| Second | 32 bits | Not read by this function. |
| Third | 32 bits | Copied into a mutable local word whose address is passed to a configuration callback. |
| Fourth | 32 bits, treated as unsigned | Copied into another mutable local word whose address is passed to that callback. Its post-callback low 16 bits select a configuration row. |

The first argument is address-like in use, but its actual type and valid range are not established. The supplied representation does not establish whether overflow in its addition is valid wrapping behavior. Callers must provide values for which the required address arithmetic and subsequent dependency accesses are valid.

The callback receives the address of the fourth-argument copy before the address of the third-argument copy. Changes to these copies do not directly overwrite the caller’s argument storage.

### Output

The function returns the object pointer produced by F-99684221ccfe.

There is no explicit error return, status result, or null-result branch. Successful return requires the intervening dependencies and indirect calls to complete and all direct memory accesses to be valid.

### Directly accessed object layout

Offsets below are byte offsets from the returned object’s base. Pointer-sized and word-sized fields are 4 bytes in the supplied representation.

| Offset | Width | Direct behavior |
|---|---:|---|
| 0x00 | 4 bytes | Written with a shared base value. |
| 0x04 | Address only | Passed as the receiver/context for both indexed component-installation dependencies. The extent of that embedded region is unknown. |
| 0xD0 | 4 bytes | Written with shared base value + 0x48; later its address is passed to a registration callback. |
| 0xD4 | 4 bytes | Written with shared base value + 0x74; later its address is passed to another registration callback. |
| 0xD8 | 1 byte | Cleared before initial publication. |
| 0xD9 | 1 byte | Cleared after the final registration callback. |
| 0xDC | 4 bytes | Receives the result of F-b9b333969cc4. |
| 0xE0 | 4 bytes | Receives the result of F-5b8f958fe5c4. |

These accesses require writable storage through offset 0xE3. They do not prove the allocation size, total object size, or contents of intervening bytes.

## Behavior

### 1. Obtain and initially publish the object

The function performs these operations in order:

1. Call F-e3199032a3a0 with the first argument plus 4.
2. Subtract 4 from that result and pass it to F-99684221ccfe.
3. Use the latter result as the object pointer, without a null check.
4. Obtain the shared publication-state address and a shared base value.
5. Write the base value to object offset 0x00.
6. Write base value + 0x48 to offset 0xD0.
7. Write base value + 0x74 to offset 0xD4.
8. Clear the byte at offset 0xD8.
9. Write the object pointer to publication-state offset 0x04.
10. Write 1 to publication-state byte 0x00.
11. Load a shared default first-group mask.
12. Set local defaults for the second-group mask to 7 and the auxiliary configuration word to 0.
13. Write 1 to publication-state byte 0x01.

The pointer is therefore published before either component group is populated and before several later object fields are initialized. This is not an atomic construction-and-publication sequence.

The function next calls F-f9aeac43f002 with the object pointer. It then calls F-b9b333969cc4 with that pointer and stores the result at offset 0xDC. Next it calls F-5b8f958fe5c4 with the pointer and stores the result at offset 0xE0.

These dependency calls may have additional effects. Their placement does not establish that previously written object or shared-state values remain unchanged afterward.

### 2. Resolve configuration

The function obtains a service object from F-cdd33263ee89. It invokes the indirect method found at byte offset 0x34 in that service object’s method table, supplying:

- The service object as receiver.
- The address of the mutable fourth-argument copy.
- The address of the mutable third-argument copy.

After the callback returns, only the low 16 bits of the updated fourth-argument copy are used here. The updated third-argument copy is not subsequently read by this function.

The low-16-bit value is first written as a zero-extended 32-bit value at publication-state offset 0x08.

If that value is less than 26, it selects one of 26 configuration records. Each record is 12 bytes:

| Record offset | Width | Meaning within this function |
|---|---:|---|
| 0x00 | 32 bits | Second-group selection mask |
| 0x04 | 32 bits | First-group selection mask |
| 0x08 | 32 bits | Auxiliary configuration word |

The selected record replaces all three local defaults.

If the low-16-bit value is 26 or greater:

- No configuration record is read.
- Publication-state offset 0x08 is overwritten with 0xFFFFFFFF.
- The previously captured default first-group mask is retained.
- The second-group mask remains 7.
- The auxiliary word remains 0.

Finally, the selected first-group mask is written at publication-state offset 0x0C, followed by the auxiliary word at offset 0x10. The second-group mask is not directly published in this state block.

For an invalid selection, the initial selection write and the subsequent 0xFFFFFFFF overwrite are distinct ordered writes; the function does not directly write only the sentinel.

### 3. Populate the first indexed group

The group’s installation context is the object address plus 0x04.

Index 0 is unconditional:

1. Call F-8157a180d901 with 0x2D4.
2. Call F-7d2442257269.
3. Pass that result to F-e0339bb8a995 with the group context and index 0.

There is no explicit null fallback for index 0.

Indices 1 through 12 are processed in ascending order. For each index, the corresponding bit of the selected first-group mask determines whether to attempt a specialized component.

If the bit is clear, the function calls F-8157a180d901 with 0x2D0 and then calls F-30edc55ccfb9 to obtain the component.

If the bit is set, the specialized operation is:

| Index | Preceding F-8157a180d901 argument | Specialized dependency |
|---:|---:|---|
| 1 | 0x2FC | F-22f2544a79a9 |
| 2 | 0x2D0 | F-75e3172e621d |
| 3 | 0x2EC | F-2689cf259307 |
| 4 | None | F-b596b3636e04 |
| 5 | 0x2DC | F-3c6d4cc848e0 |
| 6 | None | F-63d95206852d |
| 7 | None | F-58eea5cc5d8a |
| 8 | 0x2E4 | F-16bf4c670950 |
| 9 | 0x308 | F-0f5da409f9f7 |
| 10 | None | F-eab7f4d39e5b |
| 11 | 0x2D0 | F-0e5c9f46d8d2 |
| 12 | 0x308 | F-1d90215cca8b |

A zero specialized result triggers the same fallback operation used when the bit is clear: F-8157a180d901 with 0x2D0, followed by F-30edc55ccfb9.

Each resulting value is passed to F-e0339bb8a995 with the context and its index. A zero fallback result is passed through without another attempt or explicit error.

On a path that reaches the end of this phase, F-e0339bb8a995 is called exactly 13 times, at indices 0 through 12.

Only mask bits 1 through 12 affect specialization. Bit 0 and bits 13 through 31 do not affect this phase. The represented index conversion through an 8-bit character does not change these values because all are between 1 and 12.

### 4. Populate the second indexed group

After all first-group installation calls, the function calls F-8157a180d901 with 0x10. Unlike the preceding first-group sequences, this call’s result is explicitly passed as an argument to F-eceb0ddf4208, together with 0. The result of F-eceb0ddf4208 becomes a shared fallback value for this entire group.

Indices 0, 1, and 2 are then processed in ascending order:

- **Index 0:** Always uses the shared fallback. Setting mask bit 0 does not select another dependency.
- **Index 1:** If mask bit 1 is set, call F-8157a180d901 with 0x18, then pass its result and the value 1 to F-a4551691f121. Use the specialized result if nonzero; otherwise use the shared fallback.
- **Index 2:** If mask bit 2 is set, call F-8157a180d901 with 0x44, then call F-94a214cc3a9e. Use the specialized result if nonzero; otherwise use the shared fallback.

A clear selection bit also selects the shared fallback.

For each index, the selected value is passed to F-a26e140da40e with the object-plus-0x04 context and the index. The same fallback pointer or word may therefore be installed at multiple indices.

There is no null check on the shared fallback and no retry if it is zero. Mask bits 3 through 31 do not affect this group.

### 5. Perform final registrations and return

After the three second-group installation calls:

1. Obtain a service object from F-b7af80e829ec and call its method at method-table byte offset 0x00, passing the constructed object pointer.
2. Obtain a service object from F-15643c36365d and call its method at method-table byte offset 0x08, passing the address of object field 0xD0.
3. Obtain a service object from F-5fd97c30efdc and call its method at method-table byte offset 0x08, passing the address of object field 0xD4.
4. Clear object byte 0xD9.
5. Return the original object pointer.

The last byte clear occurs after all three callbacks. The function does not republish the object pointer or reset the other publication-state fields before returning.

## Side effects

The function directly modifies the object and shared publication state. Dependency calls may additionally allocate storage, mutate shared state, retain pointers, register objects, or perform other operations; those effects are not established here.

The publication-state direct-write layout is:

| Offset | Width | Written value |
|---|---:|---|
| 0x00 | 1 byte | 1 |
| 0x01 | 1 byte | 1 |
| 0x04 | 4 bytes | Object pointer |
| 0x08 | 4 bytes | Configuration selection, then optionally 0xFFFFFFFF |
| 0x0C | 4 bytes | Selected first-group mask |
| 0x10 | 4 bytes | Selected auxiliary word |

Bytes 0x02 and 0x03 are not directly written.

The function retains a local copy of the publication-state address and selected masks across dependency calls. Later direct writes use that retained address; they do not establish that a shared location identifying the publication block still holds the same address.

There is no explicit release, rollback, destruction, or ownership transfer in the supplied body. In particular:

- Early publication is not undone if a later operation fails to complete.
- A failed specialized component can be followed by another preparation/allocation-like call without any explicit release of the earlier result.
- Reusing one fallback across several indices does not establish shared ownership, reference counting, or permission to release it repeatedly.
- Passing the addresses of fields 0xD0 and 0xD4 does not establish whether the recipients retain those addresses.

## Dependencies

All dependency behavior beyond the arguments, ordering, and result use described above is **documented_only**.

- **Object acquisition:** F-e3199032a3a0, F-99684221ccfe.
- **Initial setup and field results:** F-f9aeac43f002, F-b9b333969cc4, F-5b8f958fe5c4.
- **Configuration-service acquisition:** F-cdd33263ee89.
- **Repeated size-bearing operation:** F-8157a180d901.
- **First-group unconditional component:** F-7d2442257269.
- **First-group fallback:** F-30edc55ccfb9.
- **First-group installation:** F-e0339bb8a995.
- **First-group specialized components:** The twelve dependencies listed in the first-group table.
- **Second-group fallback and specialized components:** F-eceb0ddf4208, F-a4551691f121, F-94a214cc3a9e.
- **Second-group installation:** F-a26e140da40e.
- **Final service acquisition:** F-b7af80e829ec, F-15643c36365d, F-5fd97c30efdc.

The size-bearing operation appears allocation-like, but this is inferred intent. In many first-group sequences its result is not explicitly forwarded to the next dependency in the supplied representation. Those sequences establish call order, not a verified allocation-to-constructor argument relationship.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. They assume valid memory, returning dependencies, and no dependency side effects that invalidate subsequent accesses.

1. **Callback chooses a valid row.**  
   If the callback leaves the fourth-argument copy equal to 0xABCD0005, record 5 supplies both masks and the auxiliary word. The direct selection write is 5, not the full callback-produced word.

2. **Selection reaches the invalid boundary.**  
   If the callback leaves low 16 bits equal to 26, no record is read. The selection field is written with 26 and then 0xFFFFFFFF. The previously captured default first-group mask, second-group mask 7, and auxiliary value 0 are used.

3. **High bits alone do not invalidate selection.**  
   A post-callback word of 0xFFFF0000 selects record 0. A word of 0xFFFFFFFF is invalid because its low 16 bits are 65535.

4. **No first-group specializations selected.**  
   If the first-group mask is zero, index 0 still uses F-7d2442257269. Each index from 1 through 12 independently uses the fallback sequence. No specialized dependency from the table is called.

5. **Specialized first-group creation returns zero.**  
   If bit 9 is set, the function calls F-8157a180d901 with 0x308 and then F-0f5da409f9f7. If that returns zero, it calls F-8157a180d901 with 0x2D0 and F-30edc55ccfb9 before installing index 9.

6. **Second-group fallback is reused.**  
   With a second-group mask of zero, all three indices receive the single result obtained from F-eceb0ddf4208. The fallback is not constructed three times.

7. **Second-group bit 0 has no selection effect.**  
   Masks 0 and 1 produce the same second-group dependency choices. With mask 7, index 0 still uses the fallback, while indices 1 and 2 attempt their specialized dependencies.

8. **Final callback observes the pre-clear byte.**  
   The function does not directly clear object byte 0xD9 until after the final callback returns. Its value during that callback depends on prior storage contents and dependency effects.

## Uncertainties

- **documented_only:** Actual object types, allocation sizes, method signatures beyond observed argument use, and application-level semantics.
- **documented_only:** Configuration-record contents, the shared default mask’s value, and the shared base value’s meaning.
- **documented_only:** Null, exception, fault, or non-returning behavior of dependencies. The function supplies no local recovery contract.
- **documented_only:** Whether dependencies preserve object fields, shared globals, configuration storage, or previously installed components.
- **documented_only:** Ownership, lifetimes, aliasing permissions, and release obligations for every returned or passed pointer.
- **documented_only:** Thread safety, reentrancy, callback retention of local addresses, and synchronization of early publication. Ordered direct writes are not a concurrency guarantee.
- **documented_only:** Target endianness and arithmetic behavior outside valid pointer/address ranges. The observed widths and offsets do not establish either.
- **Inferred intent:** The size-bearing calls prepare storage, the indexed dependencies install components, and the final callbacks register the object or embedded fields.

## Evidence

Established claims derive solely from the supplied static function body: direct accesses, branch conditions, loop bounds, call arguments, dependency-result use, and indirect-call offsets.

The fixed loop ranges establish 13 first-group installation calls and 3 second-group installation calls on a path that completes. The low-16-bit extraction and strict comparison establish the configuration boundary. The placement of shared-state writes establishes early publication and the invalid-selection overwrite sequence.

No external dependency implementations, runtime observations, tests, or independently authenticated provenance were available. No snapshot hash is asserted.
