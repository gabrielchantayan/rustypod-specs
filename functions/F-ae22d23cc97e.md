<!-- Trusted provenance: F-ae22d23cc97e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae22d23cc97e

## Purpose

F-ae22d23cc97e performs an optional special-category lookup followed, when necessary, by a general-category lookup. Each lookup can lead to a call to F-957b25eb2db2. A nonzero result from that dependency during the special-category stage ends processing immediately; a zero result permits the general-category stage.

The function returns a packed 64-bit value containing both a 32-bit working value and a 32-bit result. It does not return only a success indicator.

The supplied static representation establishes the branching, field accesses, dependency calls, and result packing described below. The interpretation of the two lookup regions as collections, and of the dependency result as success or failure, is inferred rather than established.

## Interface

### Inputs

The function has three input values:

| Input | Width | Established use |
|---|---:|---|
| Context address | 32 bits | Supplies two interior addresses and two 32-bit fields to dependencies; is also passed unchanged to F-957b25eb2db2. |
| Original object address | 32 bits | Supplies the category field, an optional nested-object address, and the third argument to F-957b25eb2db2. |
| Initial auxiliary value | 32 bits | Initializes temporary storage that is overwritten before that storage is explicitly passed to a dependency. |

“Unchanged” refers to the address value supplied as an argument, not to preservation of the memory it identifies.

The static representation uses 32-bit address-bearing values. It does not establish compatibility with a wider-pointer environment.

### Referenced layout

The following offsets are byte offsets. Every listed field access transfers 32 bits.

| Base | Offset | Use |
|---|---:|---|
| Original object | +4 | Category; compared with `0x16`, and read again for the general lookup when that stage is reached. |
| Original object | +8 | Nested-object address, read only when the initial category equals `0x16`. |
| Nested object | +4 | Special-stage key. |
| Context | +0x18 | Address passed to F-200598445723 during the special stage. No direct read of a 32-bit field at this offset is shown. |
| Context | +0x28 | Auxiliary value read after the special-stage F-200598445723 call. |
| Context | +0x34 | Address passed to F-200598445723 during the general stage. No direct read of a 32-bit field at this offset is shown. |
| Context | +0x44 | Auxiliary value read after the general-stage F-200598445723 call. |
| Current working address | +0x14 | Value passed as the second argument to F-957b25eb2db2 when that stage’s comparison result is not exactly one. |

The two context interior addresses are not themselves shown being dereferenced by this function. Their required accessible extents depend on F-200598445723.

### Output

The output is a 64-bit bit-pattern concatenation:

- Bits 63–32: the final 32-bit contents of the working-value storage.
- Bits 31–0: the final 32-bit handler-result value.

The lower half is initially zero. It is replaced only by a return value from F-957b25eb2db2. Negative 32-bit results, if produced, retain their 32-bit representation in the lower half; they are not sign-extended over the upper half.

The upper half is not necessarily the original object address. Dependencies receive access to the working-value storage and may replace its contents.

## Behavior

### Initial state and first read

The function first establishes:

- A handler result of zero.
- A working value equal to the original object address.
- An auxiliary temporary equal to the third input.

It then reads the category at original object offset +4.

If that read equals `0x16`, the special stage runs. Any other 32-bit value bypasses the special stage and proceeds to the general stage. No special treatment of negative category values is visible.

### Special-category stage

The special stage has this observable order:

1. Read the nested-object address from original object offset +8.
2. Read a 32-bit key from nested-object offset +4 and place it in key temporary storage.
3. Call F-200598445723 with:
   - The address of the working-value storage.
   - The context interior address at +0x18.
   - The address of the key temporary.
4. After that call returns, read the 32-bit value at context offset +0x28 into the auxiliary temporary.
5. Call F-cb011f129d08 with the addresses of the working-value storage and auxiliary temporary.
6. Compare that dependency’s 32-bit return value with exactly one.

The value read from context offset +0x28 is therefore a post-call read. It is not a value captured before F-200598445723, and a compatible behavioral reproduction must not move it ahead of that dependency call without an independently established equivalence.

If F-cb011f129d08 returns exactly one, the special stage does not call F-957b25eb2db2. Processing continues to the general stage with the handler result still zero.

If it returns anything other than one:

1. Read a 32-bit value at offset +0x14 from the address currently held in the working-value storage.
2. Call F-957b25eb2db2 with the original context address, that newly read value, and the original object address.
3. Store its 32-bit return value as the handler result.

A nonzero handler result ends processing immediately. A zero handler result permits the general stage.

The second argument to F-957b25eb2db2 is obtained after both preceding dependencies have returned. It is not necessarily read from the original object: either dependency has been given access to the working-value storage.

### General-category stage

The general stage runs in all of these cases:

- The initial category was not `0x16`.
- The special comparison returned exactly one.
- The special comparison returned a value other than one, and the special handler returned zero.

It does not run when the special handler returns nonzero.

The stage has this observable order:

1. Read the category again from original object offset +4 and place it in key temporary storage.
2. Call F-200598445723 with:
   - The address of the existing working-value storage.
   - The context interior address at +0x34.
   - The address of the key temporary.
3. After that call returns, read the 32-bit value at context offset +0x44 into the auxiliary temporary.
4. Call F-cb011f129d08 with the addresses of the working-value storage and auxiliary temporary.
5. Compare its 32-bit return value with exactly one.

The working value is not reset to the original object address before this stage. Any changes made during the special stage remain available to the general-stage dependencies.

The general key is a fresh read of the original object’s category. If a special-stage dependency changed that field, the general stage uses the changed value. It does not re-evaluate whether the special stage should run.

If the general comparison returns exactly one, no general handler call occurs. The handler result remains zero.

Otherwise, the function reads a 32-bit value at the current working address plus +0x14 and calls F-957b25eb2db2 with the original context address, that value, and the original object address. Its return value becomes the final handler result, whether zero or nonzero. There is no subsequent lookup stage.

### Completion and arithmetic

At completion, the function packs the working value into the upper half of the output and the handler result into the lower half.

The shown offsets are address calculations in a 32-bit setting. No explicit range check, overflow check, null check, or bounds check precedes them. This contract does not grant defined behavior to overflowing or otherwise invalid address calculations.

Equality with one and equality with zero are exact 32-bit comparisons. The function does not classify positive and negative dependency results separately.

## Side effects

The function explicitly writes its own temporary storage. It does not directly assign to the original object or context fields.

That restriction does not imply caller-visible immutability. Dependencies receive:

- Writable addresses of temporary storage.
- Interior context addresses.
- Original context and object addresses.

Their memory effects are not established here. In particular, the function performs later reads after dependency calls, so those reads may observe dependency-induced changes to shared or aliased memory.

No explicit allocation, release, ownership transfer, logging, or global-state update appears in this function. Whether any dependency performs those actions is unknown. The packed upper-half value has no established ownership or lifetime guarantee.

The temporary-storage addresses are valid during the active invocation. The supplied representation does not establish a safe retention or post-return-use contract for those addresses.

The third input has no demonstrated influence through the function’s explicit reads: its temporary is overwritten with a context field before being passed explicitly to F-cb011f129d08. This is not proof that arbitrary dependency behavior cannot observe adjacent storage or otherwise depend on the calling environment.

## Dependencies

### F-200598445723

Called once per reached lookup stage, before the corresponding context auxiliary-field read.

Its arguments are the address of the working-value storage, a context interior address, and the address of a 32-bit key temporary. No return value is consumed by this function.

Its internal behavior, mutations, and ownership effects are unknown. Treating it as a lookup or search initializer is inferred intent, not an established dependency contract.

### F-cb011f129d08

Called after each reached F-200598445723 call and its following context auxiliary-field read.

It receives the addresses of the working-value and auxiliary storage. Exactly one suppresses that stage’s handler call; every other return value enables the handler path.

The semantic meaning of one is unknown. In particular, it must not be relabeled “found,” “not found,” or “success” without additional evidence.

### F-957b25eb2db2

Called only after a comparison result other than one.

Its arguments are the original context address, a 32-bit value read from the current working address plus +0x14, and the original object address.

Its return becomes the handler-result word. During the special stage, any nonzero result prevents the general stage. During the general stage, the result is returned without further branching.

The function does not establish whether this dependency consumes, retains, releases, or mutates any referenced resource.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. Each assumes that the required memory accesses and dependency calls complete.

1. **Ordinary category, comparison suppresses handling.**  
   The initial category is not `0x16`. The general F-cb011f129d08 call returns one. There is one F-200598445723 call, one comparison call, and no handler call. The output’s lower half is zero; its upper half is the final working value.

2. **Special comparison suppresses only the special handler.**  
   The initial category is `0x16`, and the special comparison returns one. The general stage still runs. A comparison result of one is not an immediate function return.

3. **Special handler terminates processing.**  
   The initial category is `0x16`; the special comparison returns zero; the special handler returns `0xFFFFFFFE`. The general stage is skipped, and the lower output half is `0xFFFFFFFE`.

4. **Special handler declines to terminate.**  
   The initial category is `0x16`; the special comparison returns two; the special handler returns zero. The general stage runs. If its comparison returns one, the final lower half is zero, despite an earlier handler call.

5. **Both handlers run.**  
   The initial category is `0x16`; both comparisons return values other than one; the special handler returns zero. The general handler’s return supplies the final lower half. At most two handler calls occur through the visible control flow.

6. **Working value persists between stages.**  
   A special-stage dependency replaces the working value with another valid address, and processing reaches the general stage. The general F-200598445723 call receives access to that retained value, not a freshly restored original object address.

7. **General key observes mutation.**  
   A special-stage dependency changes the category in the original object. If execution reaches the general stage, its key is the newly read category. The initial category remains relevant only to the already-completed special-stage decision.

8. **Auxiliary reads occur after initialization calls.**  
   F-200598445723 changes the relevant context auxiliary field and returns. The following F-cb011f129d08 receives temporary storage initialized from the changed field, because the explicit read occurs after that call.

## Uncertainties

The following are **documented_only**:

- Actual object types and field meanings.
- Collection structure at the two context interior addresses.
- Dependency implementations, reentrancy, global effects, and aliasing behavior.
- Pointer alignment requirements and the target’s behavior on invalid memory access.
- Object lifetime and ownership of the returned upper-half value.
- Whether dependency calls always return normally.
- Whether the third input has significance under a broader, unstated calling convention.
- Error meanings associated with zero, one, or other dependency results.

Valid original-object storage is required for every category read reached. Nested-object storage is required only on the special path. A valid current working address for the +0x14 read is required only when the corresponding comparison returns something other than one. Dependency-specific preconditions may be stronger.

There is no visible error recovery for an invalid address, an invalid working value, or a dependency failure that does not return normally.

## Evidence

The established behavior comes solely from the supplied static representation: 32-bit field accesses, explicit ordering around dependency calls, exact comparisons, two-stage control flow, and the final 64-bit concatenation.

The category value `0x16`, context offsets, object offsets, and dependency argument ordering are directly represented. Lookup terminology and handler terminology are descriptive interpretations, not claims about hidden implementations.

No implementation access, runtime execution, or tests were used. The acceptance examples describe required outcomes conditional on their stated memory and dependency behavior; they are not observed results.
