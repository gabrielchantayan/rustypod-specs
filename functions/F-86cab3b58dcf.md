<!-- Trusted provenance: F-86cab3b58dcf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-86cab3b58dcf

## Purpose

F-86cab3b58dcf searches an integer-slot container for a requested value and, if that value is absent, attempts to place it in the first zero-valued slot.

The established behavior is narrower than a general insertion operation:

- An existing equal value causes an immediate return of −1.
- Equality between two header fields triggers dependency F-e3ce423fc856 before the zero-slot search.
- A successful zero-slot search increments the third header word, writes the requested value into the selected slot, and returns its index.
- Failure to find a zero-valued slot returns −1 without a direct slot write.
- The function does not directly append an element or increment the second header word.

The two −1 outcomes are not distinguishable through the return value alone. They can differ in side effects because the dependency may run before the no-slot outcome.

The interpretation of zero as an available-slot marker is inferred intent. The comparison and replacement of zero-valued slots are established static behavior.

## Interface

### Inputs

The function receives:

1. A pointer to a mutable header containing at least three consecutive integer words.
2. An integer value to search for and potentially store.

The supplied representation uses four-byte integer slots and signed integer comparisons. This contract describes its 32-bit integer model. The exact machine-level treatment of invalid addresses and signed arithmetic overflow is not established by the supplied material.

### Header layout

Offsets are relative to the beginning of the header.

| Offset | Width | Established use |
|---|---:|---|
| 0 bytes | 32 bits | Address-bearing word used to locate the slot array |
| 4 bytes | 32 bits | Signed scan bound |
| 8 bytes | 32 bits | Compared with the scan bound before the dependency call; incremented on a successful placement |

Each slot occupies four bytes. Slot index *k* is accessed at the slot-array base plus four times *k* bytes.

The second and third words are not assigned stronger semantic names here. In particular, calling the third word “capacity” would imply a relationship that the supplied behavior does not establish.

### Output

The return value is a signed integer:

- **−1:** an equal value was found during the initial scan, or no zero-valued slot was found during the later scan.
- **A nonnegative index:** the selected zero-valued slot was written with the requested value.

A nonnegative result identifies the index selected by the second scan. It is not a newly computed element count.

### Preconditions for defined, ordinary execution

The caller must provide readable header storage. Any slot accessed by either scan must be readable, and a selected slot must be writable. Successful placement also requires the third header word to be writable.

The address-bearing word must designate accessible storage under the target address model. Header and slot accesses must satisfy the platform’s alignment requirements.

The required accessible slot range depends on the scan bound actually used at each stage, including any changes made by F-e3ce423fc856. The function performs no established bounds, null-pointer, alignment, or overflow validation.

For a straightforward container interpretation, the header and slot array should not overlap. Overlap is not explicitly rejected, however, and can make the ordering of writes observable.

## Behavior

### Initial duplicate search

The function begins at slot index zero and searches for a slot equal to the requested value.

The second header word supplies the signed upper bound. That word is consulted again as the loop proceeds rather than being captured once for the entire initial scan. For each visited index, the address-bearing header word is used to locate the slot.

Under stable memory and a positive bound, indices are examined in ascending order, starting at zero and stopping before the bound.

If a visited slot equals the requested value:

- The function returns −1 immediately.
- F-e3ce423fc856 is not called.
- This function performs no header or slot write.

If the initial bound is zero or negative, the initial scan visits no slots.

There is no special exception for a requested value of zero. A zero-valued slot encountered during this search is an existing equal value and causes immediate rejection.

### Conditional dependency call

After the duplicate search finishes without finding an equal value, the function compares the third header word with the second header word.

If they are equal, it calls F-e3ce423fc856 with the original header pointer.

If they differ, it does not call that dependency.

No dependency result is used to choose a subsequent branch. The visible caller does not check a success indicator or implement a recovery path.

This call occurs before looking for a zero-valued slot, even when such a slot was already present during the duplicate search. Equality of the header words is the entire visible trigger.

The dependency’s effects are unknown. It must not be assumed to preserve the header, slot storage, unrelated global state, or memory ownership. Subsequent behavior uses the state that exists after it returns.

### Zero-slot search

Following the optional dependency call, the function reads the second header word and retains that value as the bound for the second scan.

It then examines slots in ascending index order, beginning at zero, and selects the first slot whose value equals zero.

Unlike the initial scan, this scan uses the retained bound throughout. Its slot accesses still obtain their base address from the address-bearing header word.

If the retained bound is zero or negative, no slots are examined.

If no zero-valued slot is found, the function returns −1. It performs no direct header or slot writes on that path, although the dependency may already have changed state.

There is no fallback placement at the bound, no direct expansion, and no second dependency call.

### Successful placement

For ordinary, non-overflowing loop execution, a selected index lies between zero and one less than the retained bound. Every such selected index follows the placement path, including index zero.

The observable direct operations are ordered as follows:

1. Read the third header word.
2. Increase that value by one.
3. Write the incremented value to the third header word.
4. Read the address-bearing header word.
5. Write the requested value to the selected slot.
6. Return the selected index.

The function does not directly update the second header word.

The header increment precedes the slot write. Consequently, if the slot write faults, the third header word may already have changed. The function has no established rollback mechanism.

The destination address is obtained after the header increment. If storage aliases in a way that makes the header write affect later addressing, the later slot write follows that changed state; it is not guaranteed to use an earlier base address.

### Behavior across dependency-induced changes

The initial duplicate search is not repeated after F-e3ce423fc856 returns.

Therefore:

- A requested value introduced by the dependency does not itself prevent subsequent placement into another zero-valued slot.
- A replacement slot-array address is used by the later scan.
- A changed second header word determines the later scan’s retained bound.
- The value incremented on success is the third header word as read after the dependency, not necessarily its entry value.

These are consequences of the caller’s ordering, not guarantees about what the dependency actually does.

## Side effects

### Direct writes

On a successful placement, this function directly performs:

- One write to the third header word.
- One write to the selected slot.

It performs no direct write to the second header word or to the address-bearing header word.

On duplicate rejection, it performs no direct writes and makes no dependency call.

On failure to find a zero-valued slot, it performs no direct writes, but may already have called F-e3ce423fc856.

### Ownership and lifetime

The visible function contains no direct allocation or deallocation operation.

That fact does not establish that storage ownership or lifetime is unchanged: F-e3ce423fc856 may have effects not described by the supplied material. No ownership-transfer or release contract for that dependency is available.

### Errors and partial effects

There is no explicit error reporting for invalid memory, exhausted storage, arithmetic overflow, or dependency failure.

Returning −1 is not evidence of allocation failure. It is the visible result of either duplicate detection or failure to locate a zero-valued slot.

Access faults and arithmetic outside the valid target model are outside the established normal-return contract. No transactional guarantee is present.

No synchronization is visible. Repeated header reads do not establish support for concurrent mutation or atomic container updates.

## Dependencies

### F-e3ce423fc856

**Invocation condition:** the third and second header words compare equal after the initial duplicate scan has completed without a match.

**Argument:** the original header pointer.

**Use of result:** no returned value is consumed by the visible caller.

**Established ordering:** the call occurs before the second scan’s bound is captured.

**Unknown contract:** allocation behavior, failure behavior, header modifications, slot modifications, global-state effects, ownership changes, and guarantees on return.

The dependency may plausibly support storage preparation or expansion, but that is inferred intent only. This function’s contract does not require that interpretation.

## Acceptance examples

The following examples assume valid, non-overlapping storage, stable memory outside dependency effects, and representable arithmetic. Header values are stated as “scan bound” and “third word”; they do not imply additional field semantics.

1. **Existing nonzero value is rejected.**  
   With scan bound 3, slots [8, 12, 0], and requested value 12, the result is −1. No dependency call or direct write occurs.

2. **Index zero is a valid successful result.**  
   With scan bound 3, third word 7, slots [0, 12, 19], and requested value 8, the result is 0. The third word becomes 8, slot zero becomes 8, and the scan bound remains 3.

3. **The earliest available slot is selected.**  
   With scan bound 4, third word 9, slots [5, 0, 0, 6], and requested value 7, the result is 1. The third word becomes 10; the later zero-valued slot remains unchanged.

4. **No available slot means no direct insertion.**  
   With scan bound 2, third word 5, slots [4, 6], and requested value 7, the result is −1. The function makes no dependency call and directly changes nothing.

5. **Equality triggers the dependency even when a zero slot exists.**  
   With scan bound 2, third word 2, slots [0, 6], and requested value 7, the dependency is called. If it returns without changing relevant state, the result is 0, the third word becomes 3, and slot zero becomes 7.

6. **Dependency changes govern the later search.**  
   If the dependency returns with scan bound 3, third word 6, and accessible slots [4, 0, 9], a requested value of 7 is placed at index 1. The third word becomes 7.

7. **Zero can be rejected before any placement attempt.**  
   With scan bound 2, slots [5, 0], and requested value zero, the result is −1 during the initial scan. No dependency call occurs.

8. **Zero placement is conditionally possible after dependency mutation.**  
   Suppose the requested value is zero, the initial scan finds no zero, and equal header words trigger the dependency. If that dependency introduces a zero-valued slot within the later bound, the function selects it, increments the third word, writes zero back to it, and returns its index.

9. **A duplicate introduced by the dependency is not rechecked.**  
   If the initial scan finds no 7, but the dependency returns with slots [7, 0] and bound 2, the function can place another 7 at index 1.

10. **An empty range does not imply successful append.**  
    With scan bound zero and a different third word, both scans visit no slots and the result is −1. If both words are zero, the dependency runs first; the outcome then depends on its effects.

## Uncertainties

- **documented_only — Target arithmetic:** four-byte slots and signed comparisons are visible; exact executable overflow behavior is not established. In particular, incrementing the maximum signed third-word value has no specified normal result here.
- **documented_only — Address arithmetic:** valid address range, pointer encoding details, and behavior when four-times-index or base-plus-offset arithmetic exceeds the target range are unknown.
- **documented_only — Dependency semantics:** no preservation, ownership, allocation, or failure guarantees are available.
- **documented_only — Field meaning:** the third word’s intended meaning is unknown. Its increment is established; a capacity or occupancy interpretation is not.
- **documented_only — Invalid state:** no general behavior is promised for inaccessible storage, misalignment, races, or overflow.
- **Inferred intent:** zero denotes an available slot, and the operation attempts a uniqueness-preserving insertion. Uniqueness is checked only before the dependency call and is not independently enforced afterward.

## Evidence

This contract is based solely on static inspection of the supplied representation of F-86cab3b58dcf and its supplied dependency mapping.

The established evidence covers branch conditions, scan ordering, four-byte slot addressing, direct writes, dependency-call placement, and return values. It does not establish the dependency’s implementation or higher-level container invariants.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static behavior, not observed executions.
