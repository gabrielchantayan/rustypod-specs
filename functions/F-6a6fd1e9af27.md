<!-- Trusted provenance: F-6a6fd1e9af27; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6a6fd1e9af27

## Purpose

F-6a6fd1e9af27 conditionally removes a supplied object from an owner-managed structure, decreases owner accounting, and optionally returns the object to a region-specific free chain. If that return makes a region completely free, the function may instead detach and release the region through allocator dependencies.

The observable behavior is conditional at two separate gates:

- A first gate can prevent all object-related reads and modifications.
- A second gate can prevent region processing after removal and owner-accounting changes have already occurred.

A completely free region is retained when it is the sole region in the selected region chain. Other completely free regions enter a release path, subject to an identity check and allocation-header validation.

These are source-derived behavioral findings, not runtime observations. Names such as “object,” “owner,” “region,” and “free chain” describe inferred roles; they do not establish external type definitions or allocator guarantees.

## Interface

**Input:** one pointer to an object record.

**Output:** no explicit return value. Outcomes are observable through dependency calls, memory changes, normal return, or the non-returning failure dependency.

The supplied representation uses 32-bit words and addresses, with selected 16-bit accesses. Offsets below are byte offsets. Pointer comparisons in the region search are treated as inclusive address comparisons in the represented address space; portable language-level ordering between unrelated allocations is not established.

### Object record

| Offset | Width | Observed role |
|---|---:|---|
| +0 | 32 bits | Replaced with a free-chain link on the object-retention path |
| +4 | 32 bits | Owner pointer, captured after the first gate |
| +32 | 32 bits | Conditional callback-associated value |

The object must remain accessible wherever the selected path reads or writes it. There is no initial null-pointer check. A null or otherwise inaccessible object is harmless only when the first gate returns zero before object access.

### Owner and associated records

| Location | Width | Observed role |
|---|---:|---|
| Owner +8 | 32 bits | Pointer to an associated accounting record |
| Owner +34 | 16 bits | Counter decremented after removal |
| Owner +36 | Address | Embedded structure passed to the removal dependency |
| Accounting record +36 | 32 bits | Counter decremented after the owner counter |
| Accounting record +48 | 32 bits | Region-manager pointer, captured before the second gate |
| Region manager +12 | 32 bits | Region-chain head |

The owner is captured once. Its associated accounting pointer is read separately for the counter operation and for selecting the region manager; the contract does not require a single permanently stable accounting pointer across arbitrary dependency effects.

### Region record

| Offset | Width | Observed role |
|---|---:|---|
| +0 | 32 bits | Identity value checked before region release |
| +4 | 32 bits | Pointer to the structure whose +12 field holds a region-chain head |
| +8 | 32 bits | Next-region pointer |
| +12 | 32 bits | Capacity or complete-free threshold |
| +16 | 32 bits | Free-object count |
| +20 | 32 bits | Inclusive lower object-address bound |
| +24 | 32 bits | Inclusive upper object-address bound |
| +28 | 32 bits | Free-object-chain head |

No alignment, stride, allocation-membership, or record-size check accompanies the inclusive address test.

### Allocation header preceding a released region

The release path accesses metadata immediately before the region record:

| Relative offset | Width | Observed role |
|---|---:|---|
| −8 | 32 bits | Header word used for validation and size accounting |
| −4 | 16 bits | First accounting index |
| −2 | 16 bits | Second accounting index |

This metadata must be accessible only on paths that reach region release. The function does not validate that requirement before dereferencing it.

## Behavior

### 1. Initial gate and optional callback

The function first calls F-78d2ee17e68f without arguments.

If its result is zero, the function returns immediately. In this case, the function itself does not access the supplied object. Effects internal to the dependency remain unspecified.

Otherwise, the function reads and retains the owner pointer from object +4.

It then reads object +32. If that value is nonzero:

1. It calls F-3080127849ac with the object pointer.
2. After that call returns, it reads object +32 again.
3. It calls F-808c5d99c3bd with the result of F-3080127849ac as the first argument and the newly read object +32 value as the second.

The second argument is not necessarily the value that caused the conditional branch. Changes made by F-3080127849ac can affect it, including changing it to zero. The owner used subsequently remains the owner captured before these calls.

### 2. Removal and owner accounting

The function calls F-8d6903442911 with:

- the address at captured owner +36;
- the supplied object pointer.

After that dependency returns, the function performs these operations in order:

1. Reads the 16-bit counter at captured owner +34 and writes its value decreased by one back to that field.
2. Reads the accounting-record pointer from captured owner +8 and decreases the 32-bit counter at that record’s +36.
3. Reads captured owner +8 again and obtains the region-manager pointer from that record’s +48.
4. Calls F-8ff94386ac71 without arguments.

If the second gate returns zero, the function returns. Removal and both counter updates are not undone. The object is not linked into a region free chain by this function on this path.

The 16-bit update stores a 16-bit result; there is no underflow check. The 32-bit counter update also has no underflow check. The represented operations retain the low-width result rather than implementing saturation or reporting a counter error.

### 3. Region search

If the second gate is nonzero, the function reads the region-chain head from captured region manager +12.

It traverses next pointers at region +8 until it finds the first region for which:

- the lower bound at +20 is no greater than the object address; and
- the object address is no greater than the upper bound at +24.

Both endpoints are included.

A null chain head or a traversal that reaches null causes a normal return. Earlier callback, removal, and accounting effects remain in place. There is no fallback insertion when no region matches.

Overlapping ranges are resolved by chain order: the first matching region wins. Cyclic chains are not detected and can prevent termination.

### 4. Free-count update and retention decision

For the matching region, the function reads its 32-bit free count at +16, adds one, and writes the result back. It compares that incremented result with the capacity field at +12.

If the values differ, it returns the object to that region’s free chain:

1. Reads the existing region free-chain head at +28.
2. Writes that head into object +0.
3. Writes the object pointer into region +28.
4. Returns.

No check prevents duplicate insertion or detects a count already above capacity.

If the incremented count equals capacity, the function rereads the selected region manager’s chain head. The region is retained through the same free-chain insertion when both conditions hold:

- it is that chain’s head; and
- its next-region pointer is null.

Thus, the selected manager’s sole region is retained even when its count reaches capacity.

The increment and comparison use a 32-bit result. There is no explicit overflow check. A wrapped result can therefore affect the equality decision.

### 5. Region identity check and detachment

A region that reaches capacity and is not retained as the sole region enters the release path.

The function compares the region’s identity word at +0 with a global expected identity value. If they differ, it returns immediately.

At that return point:

- the free count has already been incremented;
- the object has not been inserted into the region’s free chain;
- no region release has occurred through the subsequent release dependencies.

When identity matches, the function obtains a chain head through the region’s own owner pointer at +4, using offset +12 in that pointed-to record. This is a distinct access from the earlier search through the captured region manager.

If that head is the region being released, the head field is replaced with the region’s next pointer. Otherwise, the function searches for a predecessor whose next pointer equals the region and replaces that predecessor’s next pointer with the region’s next pointer.

If this predecessor search reaches null without finding the region, release processing still continues. Failure to find a predecessor is not reported and does not cancel release.

The function neither verifies that the two chain-owning records are identical nor reconciles differing chains.

### 6. Allocation validation and accounting

After detachment processing, the function calls F-2fd8826557a4 without arguments. Only after this call does it read a global allocator-context reference and capture the context address.

It then reads the 32-bit allocation-header word at region −8. If that word is zero or has its least significant bit set, it calls F-a817d8d58c10. The supplied representation marks that dependency as non-returning.

This failure occurs after the free-count increment and any completed chain detachment. There is no rollback.

For a nonzero header word with its least significant bit clear:

1. The function calls F-c0fc6d033caa with the captured allocator context.
2. After that call, it reads the unsigned 16-bit first index at region −4.
3. It reads the header word at region −8 and masks it with `0x3FFFFFFC` to obtain an accounting amount.
4. It decreases the 32-bit field at context +548 + four times the first index by that amount.
5. It decreases the 32-bit field at context +780 by the same amount.
6. It reads the unsigned 16-bit second index at region −2.
7. It decreases the 32-bit field at context +224 + four times the second index by the same amount.
8. It decreases the 32-bit field at context +544 by the same amount.

The indexes are zero-extended from 16 bits. Each scaled index ranges from zero through 262,140 bytes before the fixed field offset is added. No bounds validation is present.

The accounting amount excludes header bits 0–1 and 30–31. Its possible values are multiples of four, from zero through `0x3FFFFFFC`. Header validation does not require this masked amount to be nonzero.

The header is not revalidated after F-c0fc6d033caa. If that dependency changes it, the accounting amount can differ from what the earlier validation implied.

### 7. Final release operations

The function next calls F-09dcb55e8e0a without arguments. Its result selects a 32-bit context counter at context +784 + four times that result; the function decreases this counter by one.

It then:

1. Decreases the 32-bit counter at context +912 by one.
2. Rereads the allocation-header word at region −8.
3. Masks that new value with `0x3FFFFFFC` and subtracts the result from the 32-bit field at context +8.
4. Calls F-cc6c9b4a0f68 with the captured allocator context and the address of the header at region −8.
5. Calls F-e586bff64fb7 with the captured allocator context.
6. Returns.

The final subtraction does not necessarily use the earlier accounting amount: F-09dcb55e8e0a runs between the two header reads.

The function performs no further direct region-header access after calling F-cc6c9b4a0f68. Whether that dependency releases, transfers, or merely records the allocation is unknown.

## Side effects

Direct effects can include:

- Replacing object +0 with a free-chain link.
- Decreasing owner and accounting-record counters.
- Increasing a matching region’s free count.
- Replacing a region free-chain head.
- Removing a region from a chain.
- Decreasing allocator byte-accounting fields and allocation counters.
- Invoking a non-returning failure dependency.

These effects are not transactional. Several normal-return and failure paths leave earlier changes committed without inserting the object into a free chain.

No explicit allocation, synchronization primitive, error return, or rollback appears in the supplied function. Dependency effects may add any of those behaviors; their absence cannot be inferred from this body.

On the retention path, the function stores the object pointer in region-managed state. On the release path, it passes preceding allocation metadata to a dependency. Neither fact alone establishes a caller-visible ownership rule.

## Dependencies

| Opaque ID | Observed use |
|---|---|
| F-78d2ee17e68f | Initial zero/nonzero gate |
| F-3080127849ac | Computes the first callback argument from the object |
| F-808c5d99c3bd | Conditional two-argument callback |
| F-8d6903442911 | Called with owner +36 and object before counter decrements |
| F-8ff94386ac71 | Second zero/nonzero gate, after owner accounting |
| F-2fd8826557a4 | Called after region-detachment processing and before context capture |
| F-a817d8d58c10 | Non-returning allocation-header failure path |
| F-c0fc6d033caa | Called with captured context before accounting-header reads |
| F-09dcb55e8e0a | Supplies the final indexed-counter selector |
| F-cc6c9b4a0f68 | Receives captured context and allocation-header address |
| F-e586bff64fb7 | Final context operation |

No dependency is assumed to preserve globals, preserve object fields, acquire a lock, or own or release memory. Statements about later behavior are conditional on earlier dependencies returning and leaving subsequently accessed storage usable.

## Acceptance examples

These are conditional acceptance cases derived from the supplied representation, not executed tests.

1. **Initial rejection.** If F-78d2ee17e68f returns zero, the function returns without directly reading the object, including when the supplied pointer is null.

2. **Callback argument changes.** If object +32 is initially nonzero and F-3080127849ac changes it, F-808c5d99c3bd receives the changed value. Subsequent owner operations still use the previously captured owner.

3. **Second-gate rejection.** If the initial gate succeeds and F-8ff94386ac71 returns zero, removal and the two counter decrements occur, but no region traversal or object free-chain insertion follows.

4. **Inclusive boundary match.** An object address exactly equal to either region bound is eligible for that region. With overlapping eligible regions, the first one encountered is selected.

5. **Ordinary free-chain return.** If a region’s free count changes from two to three while capacity is five, object +0 receives the previous free-chain head and region +28 becomes the object pointer.

6. **Sole-region retention.** If the selected manager has exactly one region and its free count reaches capacity, that region remains linked and receives the object on its free chain. The release-path identity check is not required on this branch.

7. **Identity mismatch.** If a non-retained region reaches capacity but its identity does not match, its increased count remains stored. The object is not inserted by the final free-chain operation, and release processing stops.

8. **Missing predecessor.** If identity matches but the region is absent from the chain selected through its own +4 field, reaching the end of that search does not prevent the subsequent release dependencies.

9. **Invalid allocation header.** If the header is zero or odd at validation, the failure dependency is invoked after any detachment. No later accounting or release call follows under its stated non-returning behavior.

10. **Minimum accepted header.** A header value of two passes the explicit validation but yields a masked accounting amount of zero. Provided dependencies leave it unchanged, byte-accounting subtractions use zero while allocation-count decrements still occur.

## Uncertainties

- **documented_only — semantic roles:** Removal, free-list, allocator-context, and release terminology is inferred from access patterns and call placement.
- **documented_only — dependency contracts:** Return-value meanings beyond zero/nonzero tests, synchronization, reentrancy, exceptions, and memory ownership are unavailable.
- **documented_only — valid storage:** Required object extents, allocator-context extent, chain invariants, index bounds, and address validity are not supplied.
- **documented_only — selector range:** F-09dcb55e8e0a’s result range is unknown. No range check precedes its use in address calculation.
- **documented_only — concurrency:** No concurrency safety follows from the visible ordering alone.
- **documented_only — arithmetic interpretation:** Access widths and truncating stored results are visible in the supplied representation. Original signedness and source-language overflow rules are not established.
- **documented_only — failure mechanism:** The invalid-header dependency is marked non-returning; termination, diagnostic output, or another mechanism is not specified.

## Evidence

The supplied function representation establishes the listed field offsets, access widths, conditional branches, direct writes, dependency arguments, and call order. The opaque dependency map supplies dependency identities, not their behavioral contracts.

Role names and intended lifecycle are inferences kept separate from those operations. No external implementation, dependency body, runtime trace, or test result was available. No runtime tests have been performed.
