<!-- Trusted provenance: F-dd1caeeeac00; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dd1caeeeac00

## Purpose

F-dd1caeeeac00 synchronizes a stored 32-bit value in a caller-supplied object, invokes a change-handling dependency when that value differs from the requested value, and then selects one of three output paths using a byte in the object.

The output paths are:

- **Mode 1:** publish the address of an internal object region through the first output location, then invoke F-627515f2f187 with that internal region and the second output location.
- **Mode 2:** delegate output handling to F-f2b4805e6e7b.
- **Any other mode byte:** write the two supplied values directly to the two output locations.

“Mode,” “stored value,” and “internal region” are descriptive labels. Their application-level meanings are not established. In particular, the supplied evidence does not establish that the internal region is a string, array, cache, or independently owned allocation.

## Interface

The function has five inputs and no explicit return value:

| Input | Contract role |
|---|---|
| Object address | Base address used for object-relative reads, writes, and dependency arguments. |
| First supplied value | A 32-bit value used directly in the fallback output path and passed to the selected output dependency otherwise. |
| Requested value | A 32-bit value compared against the object’s stored value, potentially stored there, and used as the second supplied output value. |
| First output address | Address of a location receiving a 32-bit store in the fallback and mode-1 paths; passed to F-f2b4805e6e7b in mode 2. |
| Second output address | Address of a location receiving a 32-bit store in the fallback path; passed to either selected output dependency in modes 1 and 2. |

The supplied representation uses 32-bit integer quantities for the stored value, supplied values, and direct output stores. It also represents object addresses and the internal-region address using integer-width values. A native pointer ABI, alignment rules, and address-overflow behavior are not independently established.

The object layout directly accessed by this function is:

| Object-relative offset | Width | Use |
|---|---:|---|
| `0x18` | 32 bits | Stored value compared with, and potentially replaced by, the requested value. |
| `0x44` | 8 bits | Output-path selector. |
| `0x907c` | Address only in this function | Beginning of the internal region supplied to F-627515f2f187 and published through the first output address. |

The internal-region offset is 36,988 bytes. This function does not directly read or write that region’s contents, so its required extent cannot be determined without the dependency contract.

### Preconditions

For a defined, successfully completing call:

- The object-relative stored-value location must support the initial 32-bit read and, when the values differ, a 32-bit write.
- The selector location must support the required byte reads.
- Each direct output location used by the selected path must support its 32-bit store.
- Any selected dependency’s additional memory and argument requirements must hold.
- Address calculations must produce valid addresses under the actual target’s addressing rules.

There is no explicit null check, size check, alignment check, output-pointer validation, or error return. The function does not establish a non-aliasing requirement, but aliasing can change both final memory contents and the state visible to dependencies.

## Behavior

### 1. Synchronize the stored value

The function first reads the 32-bit stored value at object offset `0x18` and compares it with the requested value.

If they are equal:

- This phase performs no store.
- F-358767ca9a05 is not invoked.

If they differ:

1. The requested value is written to the stored-value location.
2. F-358767ca9a05 is invoked with the object address.
3. If that invocation returns normally, output-path selection proceeds.

The store precedes the dependency invocation. The dependency can therefore observe the requested value already installed, unless some intervening external effect changes it.

The function does not recheck the stored value after F-358767ca9a05 returns. If that dependency changes the stored value again, this function does not restore the requested value as part of its synchronization phase.

The comparison is equality, not an ordering or range test. No signed comparison, clamping, or interpretation of negative values is visible.

### 2. Select the output path

Output-path selection occurs after the optional synchronization dependency returns. Consequently, changes that dependency makes to the selector can affect the current call.

The selector is not unconditionally captured once:

1. A byte is read from object offset `0x44` and compared with 1.
2. If that observation is 1, the mode-1 path is selected.
3. Otherwise, the selector byte is read again and compared with 2.
4. If the second observation is 2, the mode-2 path is selected.
5. Otherwise, the fallback path is selected.

There is no explicit store or dependency call between these two selector reads. Nevertheless, the represented behavior contains two reads on the non-mode-1 path; a single stable snapshot should not be substituted in a contract intended to preserve those observations.

With an ordinarily stable selector, the paths correspond to byte values 1, 2, and all other values. Byte signedness does not change those stable-selector classifications.

### 3. Mode-1 path

When the first selector observation equals 1:

1. The object address plus `0x907c` is stored through the first output address as a 32-bit address representation.
2. F-627515f2f187 is invoked with these arguments, in order:
   - Object address.
   - First supplied value.
   - Requested value.
   - Address of the internal region at object offset `0x907c`.
   - Second output address.
3. The function returns if the dependency returns normally.

The first output store happens before the dependency call. The dependency therefore runs after any effects of that store, including effects caused by aliasing with object storage.

The internal-region argument is derived from the object address. It is not obtained by reading back the first output location. The two contain corresponding address values immediately after the explicit store, but the dependency receives the internal-region address independently of the output location’s subsequent contents.

This function does not directly initialize the second output location in this path. It also does not restore the first output location after the dependency returns.

### 4. Mode-2 path

When the first selector observation is not 1 and the second is 2, F-f2b4805e6e7b is invoked with:

- Object address.
- First supplied value.
- Requested value.
- First output address.
- Second output address.

The function then returns if the dependency returns normally.

There are no direct output stores in this path. Any output values, memory mutations, or failures attributable to this phase depend on F-f2b4805e6e7b.

### 5. Fallback path

When neither selector condition selects a dependency:

1. The first supplied value is stored through the first output address.
2. The requested value is stored through the second output address.
3. The function returns.

The order matters when the output locations overlap. If both output addresses identify the same valid 32-bit location, the second store overwrites the first, leaving the requested value there.

## Side effects

The directly established effects are limited to:

- Reading the stored-value location.
- Conditionally writing that location.
- Reading the selector once or twice.
- Performing the path-specific output stores.
- Invoking the selected dependencies.

No direct allocation, release, reference-count adjustment, global-variable access, or resource registration is shown. This does not constrain the dependencies: their global effects, allocation behavior, and ownership behavior are unknown.

Publishing an address inside the supplied object is not evidence of an ownership transfer. This function performs no explicit lifetime extension. Whether the published address remains usable depends on the object’s lifetime and the internal region’s actual contract.

There is no explicit rollback. If a dependency does not return normally, or an access fails, earlier stores may already have occurred. The supplied evidence specifies neither a fault-recovery mechanism nor an error-reporting convention.

Aliasing is observable:

- An output store may modify object state if an output address points into the object.
- A mode-1 output store may modify state that F-627515f2f187 subsequently reads.
- A fallback output store may replace the synchronized stored value without causing another F-358767ca9a05 invocation.
- Dependencies may modify output locations or object state; this function does not generally repair those modifications.

## Dependencies

| Opaque function ID | Invocation condition | Established interface and ordering |
|---|---|---|
| F-358767ca9a05 | Initial stored value differs from the requested value. | Receives the object address after the requested value has been stored. Runs before selector reads. |
| F-f2b4805e6e7b | First selector observation is not 1; second observation is 2. | Receives the object address, both supplied values, and both output addresses. |
| F-627515f2f187 | First selector observation is 1. | Receives the object address, both supplied values, the internal-region address, and the second output address, after publication through the first output address. |

No dependency return value is used by this function. This does not establish that the dependencies themselves have no return value.

The supplied evidence does not define dependency output formats, preservation guarantees, memory ownership, reentrancy, global-state behavior, or failure modes.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume valid memory, normal dependency returns, and no unmentioned external interference.

1. **Equal stored value, stable fallback selector.**  
   The stored value and requested value are both 7; the selector is 0; output locations are distinct and do not overlap the object. F-358767ca9a05 is not invoked. The first output receives the first supplied value, followed by 7 being written to the second output.

2. **Changed stored value, then fallback.**  
   The stored value is 4 and the requested value is 9. F-358767ca9a05 must be invoked after the stored-value location receives 9. If the dependency leaves a stable selector other than 1 or 2, the direct output stores follow.

3. **Synchronization dependency changes the path.**  
   The initial stored value differs from the request. If F-358767ca9a05 returns with selector 1, mode 1 is selected regardless of the selector’s value before that invocation.

4. **Stable mode 1.**  
   After synchronization, the first selector read yields 1. The first output receives the internal-region address before F-627515f2f187 runs. No direct second-output store is required. Final output contents cannot be specified without the dependency’s behavior.

5. **Stable mode 2.**  
   Both selector reads yield 2. F-f2b4805e6e7b receives the original supplied values and output addresses. This function performs no direct output store.

6. **Selector observations differ.**  
   If the first selector read yields 0 and the second yields 2, mode 2 is selected. If the first yields 2 and the second yields 1, fallback is selected. If the first yields 1, there is no second selector read.

7. **Identical fallback output addresses.**  
   Both outputs designate the same valid 32-bit location, outside the object. With first supplied value 11 and requested value 23, the stores occur in that order and leave 23 at that location.

8. **Fallback output aliases stored state.**  
   The stored value already equals the request, and the first output designates that stored-value location. A fallback first-output store can replace the stored value with the first supplied value. No new synchronization check or change-handling call follows that store.

## Uncertainties

The following are **documented_only**:

- Exact native ABI, pointer width, integer representation, alignment requirements, and invalid-access behavior.
- Address arithmetic behavior when adding object offsets would exceed the target address range.
- Required object size beyond the directly accessed fields.
- Internal-region layout, capacity, contents, and lifetime guarantees.
- Dependency contracts, including ownership and preservation of object or global state.
- Whether concurrent or externally changing selector storage is supported.
- Application-level meanings of the supplied values and selector modes.
- Whether failures propagate through faults, exceptions, termination, or another mechanism.

An **inferred intent** is that F-358767ca9a05 refreshes state after a stored-value change, while the selector chooses an output representation. Neither interpretation is needed to establish the observable contract, and neither supplies missing dependency guarantees.

## Evidence

Established static evidence consists of the supplied function representation: its five inputs, object-relative accesses, conditional store, three dependency invocation sites, selector-read ordering, output-store ordering, and return paths.

No runtime tests have been performed. Acceptance examples are consequences of the visible operations under their stated conditions, not observed executions. Dependency internals and target-platform details were not supplied.
