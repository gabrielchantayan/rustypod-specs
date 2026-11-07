<!-- Trusted provenance: F-b3bed2dfc5ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b3bed2dfc5ec

## Purpose

F-b3bed2dfc5ec selects a 32-bit result from object-associated metadata, two caller-supplied values, and conditional interactions with four opaque dependencies.

The established behavior is:

- Derive an initial value from the object-slot address and a metadata offset.
- Store that value in a local field.
- When the third argument is nonzero, obtain a candidate from a temporary context instead of using the initial value as the candidate.
- Return a nonzero candidate immediately.
- Otherwise, apply a continuation gate associated with the second argument.
- If continuation is permitted, select the current contents of the stored-value field or attempt a final operation associated with the third argument.

The stored-value field is not guaranteed to retain its initial contents. It lies at byte offset 40 from the context pointer passed to F-b8520d4f77a0, immediately after the candidate field at offset 36. The dependency’s expected context extent and its effects on this adjacent field are unknown. Later reads must therefore be described in terms of the field’s contents at those reads, not an immutable initial value.

An object lookup or adaptation operation is **inferred intent** only. The evidence does not establish the semantic types of the metadata, derived value, or result.

## Interface

### Arguments

| Argument | Established representation and use |
|---|---|
| Object slot | Pointer to a readable 32-bit word containing a metadata address. The slot’s own address also participates in calculating the initial value. |
| Second value | 32-bit integer compared with the metadata value and supplied to dependencies. |
| Third value | 32-bit integer controlling the initial context call and participating in later comparisons and calls. |

There are no visible null, bounds, alignment, or metadata-validity checks.

### Input memory layout

For each metadata access, the function reads the object slot to obtain an address. Relative to that address:

| Relative byte location | Width | Meaning in this function |
|---|---:|---|
| −8 | 32 bits | Offset used to derive the initial value |
| −4 | 32 bits | Metadata value used in comparisons and dependency arguments |

The object slot is read separately for these two accesses. The evidence does not establish a single consistent metadata-address snapshot if the slot changes between reads.

The initial value is the offset from the first access plus the object-slot address represented in the target’s 32-bit integer/address model. The displacement is in bytes, not 32-bit elements.

### Return value

The result is a 32-bit integer selected from:

- The initial derived value, when the third argument is zero and that value is nonzero.
- The candidate field written or left unchanged by F-b8520d4f77a0.
- The stored-value field’s current contents at a specified later read.
- The output field after a nonzero status from F-509c40235b6d.
- Zero on a rejected continuation gate or a zero final status.

Zero does not uniquely identify failure. The selected stored value or successful-operation output may itself be zero.

### Preconditions and arithmetic

Every object-slot and metadata access actually performed must satisfy the target’s memory-access requirements. Dependency-specific preconditions are unknown.

All displayed integer fields and results are 32 bits. The evidence does not establish portable behavior for signed addition overflow, invalid address formation, or misaligned access. In particular, this contract does not turn the initial addition into a guaranteed portable modulo operation.

## Behavior

### Initial reads and stored value

Before any dependency call, the function performs these operations in order:

1. Reads the object slot and the 32-bit offset eight bytes before the resulting address.
2. Adds that offset to the object-slot address to form the initial value.
3. Reads the object slot again and the 32-bit metadata value four bytes before that resulting address.
4. Initializes the local stored-value field with the initial value.

The metadata value is retained for later comparisons and arguments. The function does not explicitly reread metadata after calling dependencies.

The initial candidate is the initial derived value. The stored-value field is separate from that candidate.

### Optional context call and adjacent local layout

If the third value is zero, F-b8520d4f77a0 is skipped.

If the third value is nonzero, the function initializes local storage and passes its base address to F-b8520d4f77a0. The displayed layout relative to that address is:

| Byte offset | Width | Contents before the call |
|---:|---:|---|
| 0 | 32 bits | First opaque static value, represented as a pointer |
| 4 | 32 bits | Second opaque static value |
| 8 | 32 bits | Zero |
| 12 | 32 bits | Zero |
| 16 | 32 bits | Zero |
| 20 | 32 bits | Third value |
| 24 | 32 bits | Second value |
| 28 | 32 bits | Object-slot pointer |
| 32 | 32 bits | Zero; reused as the final output field if that stage is reached |
| 36 | 32 bits | Zero; candidate field, later reused as the continuation indicator |
| 40 | 32 bits | Stored-value field, already initialized to the initial derived value |

These offsets describe the displayed local-storage arrangement. They do not establish a public structure definition, a 40-byte context boundary, or that the field at offset 40 is outside the dependency’s expected accessible extent.

After initialization of the stored-value field, the visible optional initialization order is: offset 12, offset 0 using the first static value, offset 32, offset 4 using the second static value, offsets 8 and 16, offset 36, then offsets 20, 24, and 28. This identifies the relative order of the two static reads without assuming those reads are interchangeable.

F-b8520d4f77a0 receives, in order:

1. The initial derived value.
2. The retained metadata value.
3. The local-storage base address.
4. Zero.

Its direct return value is ignored. After it returns, the function reads offset 36 and uses that field as the candidate, replacing the initial candidate.

The stored-value field at offset 40 is adjacent to memory exposed through the context pointer. Preservation of that field is not established. A changed stored value does not itself become the candidate: the candidate is read from offset 36.

### Immediate candidate return

A nonzero candidate is returned immediately.

On this path, the function makes no calls to F-f3c8ea0df295, F-f4ffd1965bfa, or F-509c40235b6d.

Consequently, a zero third value and a nonzero initial derived value cause an immediate return without any dependency calls. With a nonzero third value, a zero context candidate enters the fallback path even when the initial derived value was nonzero.

### Continuation gate

The remaining stages are reached only when the candidate is zero.

The function sets the offset-36 continuation indicator to one. If the metadata value equals the second value, it skips F-f3c8ea0df295 and proceeds.

If those values differ, it:

- Writes the address of the continuation indicator into the local pointer field at offset 0.
- Resets the indicator to zero.
- Reads the stored-value field for the first argument to F-f3c8ea0df295.
- Calls F-f3c8ea0df295 with that current stored value, the object-slot pointer, the metadata value, and the second value.
- Reads the continuation indicator after the call.

This argument is not necessarily the original derived value: F-b8520d4f77a0 may already have affected the adjacent stored-value field.

The direct return from F-f3c8ea0df295 is ignored. If the indicator remains zero, the function returns zero. Otherwise, it proceeds.

The indicator’s address is not an explicit argument to F-f3c8ea0df295. The local pointer assignment is visible, but the mechanism by which the dependency could discover or affect that storage is unresolved. The indicator check must not be replaced with a check of the dependency’s return value.

### Stored-value selection and ordered comparison calls

After the continuation gate permits progress, the function reads the stored-value field and uses its current contents as the prospective result.

If the metadata value equals the third value, that result is returned. Neither F-f4ffd1965bfa nor F-509c40235b6d is called.

Otherwise, the function calls F-f4ffd1965bfa twice:

1. First with the metadata value.
2. Then with the third value.

Both calls occur. Their 32-bit results are compared for equality.

After both calls return, the function reads the stored-value field again, replacing the earlier prospective result with the field’s contents at this later read. If the two dependency results are equal, this newly read value is returned.

Thus, the result on the equal-results path is not a snapshot taken before these calls, nor necessarily the original derived value. Whether either call can affect this local field through previously exposed storage or other mechanisms is **documented_only**; preservation is not assumed.

A zero third value suppresses only F-b8520d4f77a0. These comparison calls can still occur with zero as the second call’s argument.

### Final operation

If the two F-f4ffd1965bfa results differ, the function:

- Clears the local pointer field at offset 0.
- Clears the 32-bit field at offset 4.
- Sets the output field at offset 32 to zero.
- Calls F-509c40235b6d with the address of the stored-value field, the address of the output field, the metadata value, and the third value.

The final status controls return selection:

- Zero status selects zero.
- Nonzero status selects the output field’s post-call contents.

The stored-value field is explicitly passed by address and may be changed by this dependency. Its changed contents are not selected as the return merely because the dependency modified them. Likewise, the stored-value read performed after the two comparison calls does not determine the final result once this operation is reached.

A nonzero final status does not guarantee a nonzero return: the output field starts at zero and may remain zero.

## Side effects

### Direct accesses

The function explicitly reads the object slot twice, the corresponding metadata words, the two static values on the optional-context path, local fields, and dependency results.

Its explicit writes target local storage. No direct write through the object-slot pointer or to the metadata words is visible.

The dependency interfaces expose:

- The object-slot pointer through the context and directly to F-f3c8ea0df295.
- The local-storage base address to F-b8520d4f77a0.
- The stored-value field’s address and output field’s address to F-509c40235b6d.

The offset-40 stored-value field’s proximity to the context is relevant exposure, not evidence of guaranteed preservation or guaranteed mutation. Later dependencies’ ability to affect previously exposed storage is also unresolved.

### Ownership and lifetime

No direct allocation, release, reference-count change, or ownership transfer is established.

The context, indicator, stored-value field, and output field are temporary local storage. Their addresses are available during the invocation; later use after the invocation is not established as supported.

No dependency is assumed to preserve globals, object memory, resources, or exposed local storage. Nor is any dependency assumed to allocate or release memory. A nonzero return is not established to be an address or caller-owned resource.

### Errors and abnormal completion

There is no explicit error channel other than the result and no visible recovery for invalid memory accesses or arithmetic conditions.

All normal-path descriptions assume dependencies return normally. Exception behavior, unwinding, and nonlocal control transfer are unresolved.

## Dependencies

| Opaque ID | Established interaction | Unresolved behavior |
|---|---|---|
| F-b8520d4f77a0 | Optional call when the third value is nonzero. Receives the initial value, metadata, context pointer, and zero. Offset 36 is read afterward. | Context extent, significance of static fields, effects on adjacent offset-40 storage, direct return meaning, external effects, and resource behavior. |
| F-f3c8ea0df295 | Called on a zero-candidate path when metadata differs from the second value. Receives the stored-value field’s current contents. Its direct return is ignored. | How the continuation indicator could be affected, effects on local or external state, and resource behavior. |
| F-f4ffd1965bfa | Called first for metadata and then for the third value when they differ. Equal results suppress the final operation; the stored-value field is reread after both calls. | Meaning of the result, purity, and ability to affect previously exposed local storage or external state. |
| F-509c40235b6d | Receives stored-value and output addresses plus metadata and third values. Zero status selects zero; nonzero status selects output. | Status meaning beyond zero/nonzero, output interpretation, other mutations, and resource behavior. |

## Acceptance examples

These examples are conditional consequences of the static representation, not executed tests. They assume valid accesses and normal returns.

1. **Immediate derived-value result.**  
   The third value is zero and the initial derived value is 0x1000. The result is 0x1000, with no dependency calls, regardless of the second value.

2. **Context candidate takes precedence.**  
   The third value is nonzero. F-b8520d4f77a0 leaves 0x2000 at offset 36. The result is 0x2000, regardless of any change it makes to offset 40. No fallback dependencies are called.

3. **Zero context candidate overrides a nonzero initial candidate.**  
   The initial value is 0x1000, but F-b8520d4f77a0 leaves offset 36 zero. The continuation gate is entered rather than immediately returning 0x1000.

4. **Adjacent stored-value mutation affects the equality path.**  
   Conditionally, F-b8520d4f77a0 leaves offset 36 zero and changes offset 40 from 0x1000 to 0x4100. Metadata equals both caller-supplied values. The gate skips F-f3c8ea0df295, and the post-gate stored-value read returns 0x4100. This example does not establish whether the dependency actually performs or supports that mutation.

5. **Gate call receives the current stored value.**  
   Following the same conditional offset-40 change, metadata differs from the second value. F-f3c8ea0df295 receives 0x4100 as its first argument, not 0x1000.

6. **Ignored gate-call return.**  
   The candidate is zero and metadata differs from the second value. F-f3c8ea0df295 leaves the continuation indicator zero. The function returns zero even if that dependency’s direct return is nonzero.

7. **Post-gate read controls direct metadata equality.**  
   The gate permits continuation, metadata equals the third value, and the stored-value field contains 0x5200 at the read after the gate. The result is 0x5200, without either comparison call or the final operation.

8. **Reload after both comparison calls.**  
   The gate permits continuation and metadata differs from the third value. The two ordered F-f4ffd1965bfa calls return equal results. If the stored-value field contains 0x6300 after those calls, the function returns 0x6300, even if the earlier post-gate read obtained a different value. The mechanism for such a change remains unresolved.

9. **Final failure discards output.**  
   The final operation is reached. F-509c40235b6d writes a nonzero output but returns zero status. The function returns zero.

10. **Final success selects output, not stored value.**  
    F-509c40235b6d returns nonzero status, sets output to 0x3000, and changes the stored-value field to 0x4000. The result is 0x3000. If output instead remains zero, the result is zero.

11. **Zero third value does not suppress all later work.**  
    The third value and initial derived value are zero; metadata is nonzero. If the continuation gate permits progress and the two comparison results differ, F-509c40235b6d is called with zero as its fourth argument.

## Uncertainties

The following remain **documented_only**:

- Semantic types and meanings of the object, metadata, derived value, and result.
- Interpretation of the two static context values.
- The dependency’s expected context extent and whether it includes offset 40.
- Dependency-mediated mutation of the adjacent stored-value field, including possible effects between its distinct reads.
- How F-f3c8ea0df295 could affect the continuation indicator.
- Ownership, aliasing restrictions, resource behavior, concurrency, and reentrancy.
- Arithmetic-overflow, invalid-address, and alignment behavior in the original execution environment.
- Any semantic meaning of zero beyond the explicit branch and return rules.

## Evidence

This contract uses only static inspection of the supplied function representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent evidence of behavior.

Established evidence includes the displayed widths and local offsets, initialization order, metadata accesses, call arguments and order, ignored direct returns, branch conditions, stored-value reads after the gate and after both comparison calls, and final output selection.

High-level intent, dependency implementations, and mutation mechanisms remain separate from those observations. No implementation access, dependency-body inspection, runtime tests, or execution observations have been used.
