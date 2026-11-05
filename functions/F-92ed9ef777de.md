<!-- Trusted provenance: F-92ed9ef777de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-92ed9ef777de

## Purpose

F-92ed9ef777de advances the availability state of a storage pool whose records occupy 20 bytes each.

It gives priority to an existing linked list of available records. When that list is empty, it advances a cursor through the current storage region. If the cursor equals the region’s end, it first requests a new region and a 12-byte descriptor, links the descriptor into the pool, and then advances the new cursor by one record.

The function returns no value. In particular, it does not directly return the record removed from the available-record list or the record preceding the advanced cursor.

The interpretation as a pool is inferred intent. The field accesses, allocation requests, arithmetic, and write ordering below are established by the supplied static evidence.

## Interface

### Inputs

The function receives:

1. **State address:** a pointer to a writable structure containing at least four consecutive 32-bit words.
2. **Allocation-mode value:** a 32-bit integer tested only for equality with zero. Its magnitude and sign otherwise have no effect.

The state structure has the following observed layout:

| Byte offset | Width | Behavioral role |
|---|---:|---|
| 0 | 32 bits | Address of the current descriptor, or zero |
| 4 | 32 bits | Address of the first available record, or zero |
| 8 | 32 bits | Current storage cursor |
| 12 | 32 bits | Current storage limit |

These are behavioral names, not evidence of declared public field names.

A descriptor has this observed layout:

| Byte offset | Width | Behavioral role |
|---|---:|---|
| 0 | 32 bits | Previous descriptor address |
| 4 | 32 bits | Region capacity, in 20-byte records |
| 8 | 32 bits | Storage-region base address |

For an available record, only its first 32-bit word is accessed. That word supplies the address of the next available record.

### Outputs

There is no explicit return value or status result. Observable outputs are:

- Updates to the state structure.
- Initialization of a new descriptor when storage growth is required.
- Calls to F-871e64ddc6de.
- Any additional effects of that dependency.

No output pointer is written with the address of a selected record.

### Preconditions and representation

The supplied representation uses 32-bit words and address-bearing words. It is not a portable specification for an arbitrary host pointer width.

The state address must permit the accesses selected by the branch taken. Additional required accessible memory is branch-dependent:

- A nonzero available-record address must permit a 32-bit read at its beginning.
- During automatic growth, a nonzero descriptor address must permit a 32-bit capacity read at descriptor offset 4.
- A newly returned descriptor address must permit writes to all 12 descriptor bytes.
- For ordinary pool use, the storage allocation must accommodate the intended record count and cursor arithmetic.

The function performs no explicit validation of these conditions.

The straightforward pool interpretation also requires that the state, descriptor, and available-record storage do not overlap in ways that interfere with the specified reads and writes. Non-overlap is not checked. Aliasing can change the consequences of later accesses, so the ordered operations below take precedence over a simplified final-state description.

## Behavior

### 1. Prefer the available-record list

The function first reads the state word at offset 4.

If it is nonzero:

1. It uses the available-record address held in that state field to read the record’s first 32-bit word.
2. It writes that value back to the state word at offset 4.
3. It returns immediately.

Under ordinary noninterfering storage, this removes the first available record from the list.

This path does not:

- Inspect the allocation-mode value.
- Compare or advance the storage cursor.
- Request an allocation.
- Change the descriptor-head field.
- Clear or otherwise initialize the removed record.

The removed record’s first word is read, not overwritten. No release operation is present.

The condition and the record access are separate accesses in the supplied representation. No atomicity or concurrent-modification guarantee is established.

### 2. Decide whether storage growth is required

When the available-record field is zero, the function compares the state cursor at offset 8 with the limit at offset 12.

Only equality triggers growth.

If they differ, the function skips all capacity selection and allocation, then advances the cursor as described in step 5. It does not check whether the cursor is below the limit, above it, aligned to a record boundary, or within a valid region.

Consequently, unequal cursor and limit values do not establish that a full record remains available.

### 3. Select the new capacity

This step occurs only when the available-record field is zero and the cursor equals the limit.

The selected capacity is a 32-bit unsigned quantity representing a number of 20-byte records.

#### Nonzero allocation-mode value

Any nonzero allocation-mode value selects a capacity of exactly one record.

The existing descriptor’s capacity is not consulted for this selection.

#### Zero allocation-mode value, no descriptor

If the allocation-mode value is zero and the state’s descriptor-head word is zero, the selected capacity is 32 records.

The requested storage size is therefore 640 bytes.

#### Zero allocation-mode value, existing descriptor

If the allocation-mode value is zero and the descriptor-head word is nonzero, the function reads the unsigned 32-bit capacity at descriptor offset 4.

It forms two candidate capacities:

- The old capacity plus 32.
- The old capacity plus its value shifted right by one bit, plus its value shifted right by three bits.

Each right shift is an unsigned shift and therefore corresponds to division with truncation toward zero. For ordinary values, the second candidate is approximately 1.625 times the old capacity, but the separate truncations matter.

The function selects the greater of the two candidates using an unsigned comparison. A tie selects the shifted-sum candidate, although the numeric capacity is identical.

The additions are unsigned 32-bit arithmetic. They can wrap modulo \(2^{32}\). The comparison is applied to the wrapped candidate values, not to unbounded mathematical results.

There is no validation that:

- The old capacity is positive.
- The new capacity exceeds the old capacity.
- The new capacity is nonzero.
- The resulting byte count can represent the intended storage size.

### 4. Allocate and publish a new region

With the selected capacity fixed, the function makes two calls to F-871e64ddc6de, in this order:

1. Request size 12, with second argument zero.
2. Request the selected capacity multiplied by 20, with second argument zero.

The multiplication is unsigned 32-bit arithmetic. A large capacity can therefore produce a wrapped byte request, including zero.

The first result is treated as a descriptor address. The second result is treated as a storage-region address. Neither result is explicitly checked before subsequent use.

After both calls return, the function performs these writes and reads in order:

1. Writes the second call’s result to descriptor offset 8.
2. Reads the state’s current descriptor-head word and writes it to descriptor offset 0.
3. Writes the selected capacity to descriptor offset 4.
4. Writes the new descriptor address to state offset 0.
5. Reads the storage-region address back from descriptor offset 8.
6. Writes that reloaded address to the state cursor at offset 8.
7. Writes the reloaded address plus the selected capacity multiplied by 20 to the state limit at offset 12.

The descriptor is thus initialized before its address is published in the state’s descriptor-head field. The cursor and limit are updated after that publication.

The previous-descriptor value is read **after both dependency calls**, not saved before them. If a dependency changes that state field, the newly created descriptor links to the value present at this later read.

Likewise, the payload address used for the state cursor and limit is reloaded from descriptor memory after the descriptor-head update. It is not merely assumed to equal the second allocation result under arbitrary aliasing.

The selected capacity remains the locally computed value. The limit calculation does not reread the descriptor’s capacity field.

The end-address expression uses unsigned 32-bit arithmetic through its unsigned capacity-derived operand. It does not detect address wraparound.

### 5. Advance the cursor

After either:

- Finding unequal cursor and limit values, or
- Completing the growth sequence,

the function reads the current cursor and writes back that value plus 20.

Under ordinary nonoverlapping storage and successful allocation, growth therefore leaves:

- The cursor one record beyond the new storage base.
- The limit at the storage base plus the region’s total byte size.
- The descriptor head pointing to the newly initialized descriptor.
- The new descriptor linked to the descriptor-head value read after the dependency calls.

A one-record region is immediately exhausted by this advance: its cursor equals its limit on return.

The final cursor increment is represented as signed integer addition in the supplied source form. Unlike the unsigned capacity calculations, defined modulo arithmetic for signed overflow is not established by this evidence. Normal operation requires a representable result; exact overflow behavior is documented_only.

## Side effects

### Direct memory effects

Depending on the path, the function directly writes:

- Only the available-record field; or
- Only the cursor field; or
- A new descriptor’s three words, followed by the state’s descriptor head, cursor, limit, and final cursor advance.

It does not explicitly write record payload contents or clear newly allocated storage.

No descriptor or storage region is explicitly released.

### Allocation and ownership

The function retains references to the new descriptor and its storage region through the state and descriptor layouts. It also retains the previous descriptor address in the new descriptor.

This establishes linkage, not a complete ownership contract. The evidence does not establish:

- Who must eventually release either allocation.
- Which release operation would be appropriate.
- Whether F-871e64ddc6de can reclaim or move existing allocations.
- Whether allocation results are fresh, disjoint, aligned, or initialized.
- Whether the dependency modifies other state.

### Failure and partial state

There is no explicit error return, null-result branch, rollback, or cleanup path.

If the first dependency call returns an unusable descriptor address and returns normally, the second dependency call is still attempted before descriptor initialization begins.

If the second call fails, the consequence depends on how that dependency reports failure:

- A returned address value is stored without validation.
- An exception, termination, or other nonlocal exit prevents later operations according to the dependency’s behavior.
- A zero result is not explicitly rejected.

If execution stops during descriptor initialization or state publication, earlier writes and dependency effects may already be visible. The update is not transactional.

## Dependencies

### F-871e64ddc6de

Observed use:

| Call order | First argument | Second argument | Result interpretation |
|---|---:|---:|---|
| First | 12 | 0 | Descriptor address |
| Second | Selected capacity × 20, modulo \(2^{32}\) | 0 | Storage-region address |

The dependency’s apparent allocation role is inferred from the requested sizes and subsequent pointer use.

Its full contract is unavailable. In particular, zeroing, alignment, failure signaling, ownership, synchronization, reentrancy, and preservation of externally accessible state are documented_only.

No other function dependency appears in the supplied body.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless noted otherwise, they assume valid nonoverlapping memory, normal dependency returns, no external interference, and representable final cursor addition.

### Available record takes priority

Given:

- Available-record field points to a record whose first word is address \(N\).
- Cursor equals limit.
- Either allocation-mode value.

Expected behavior:

- Available-record field becomes \(N\).
- Cursor, limit, and descriptor head remain unchanged.
- No dependency call occurs.
- The removed record is not cleared.

### Existing region has unused cursor space

Given:

- Available-record field is zero.
- Cursor is address \(B\).
- Limit differs from \(B\).

Expected behavior:

- Cursor becomes \(B + 20\).
- No dependency call occurs.
- Descriptor head and limit are unchanged.

This expectation does not imply that the cursor was valid or below the limit.

### First automatic region

Given:

- Available-record field is zero.
- Cursor equals limit.
- Descriptor head is zero.
- Allocation-mode value is zero.
- Dependency calls return descriptor address \(D\) and storage address \(B\).
- The dependency does not change the descriptor-head field.

Expected behavior:

- Calls request 12 bytes and 640 bytes, in that order.
- Descriptor \(D\) contains previous address zero, capacity 32, and base \(B\).
- State descriptor head becomes \(D\).
- State cursor becomes \(B + 20\).
- State limit becomes \(B + 640\).

### Forced one-record region

Given an exhausted region, an empty available-record list, and any nonzero allocation-mode value:

- Capacity is one, irrespective of the old descriptor capacity.
- Calls request 12 bytes and 20 bytes.
- With returned storage base \(B\), both cursor and limit become \(B + 20\).

Absent intervening changes, another call with an empty available-record list encounters exhaustion again.

### Automatic growth from capacity 64

Given automatic growth from an existing capacity of 64:

- Additive candidate: 96.
- Shifted-sum candidate: \(64 + 32 + 8 = 104\).
- Selected capacity: 104.
- Storage request: 2,080 bytes.

### Truncation affects growth

Given automatic growth from an existing capacity of 33:

- Additive candidate: 65.
- Shifted-sum candidate: \(33 + 16 + 4 = 53\).
- Selected capacity: 65.
- Storage request: 1,300 bytes.

### Dependency changes the descriptor head

If either dependency changes the state’s descriptor-head word to \(H\), and it remains \(H\) until the post-call linkage read, the new descriptor’s previous-address field receives \(H\), not necessarily the descriptor address present on entry.

## Uncertainties

The following are documented_only:

- The public meaning of the allocation-mode value beyond zero versus nonzero.
- How callers identify or retain the record whose availability is consumed.
- The dependency’s allocation and failure contract.
- Lifetime and eventual release of descriptors and regions.
- Whether callers enforce valid capacities, cursor bounds, alignment, and non-overlap.
- Whether concurrent access is possible or externally synchronized.
- Exact execution behavior on invalid addresses or signed overflow.
- Whether the recovered 32-bit address representation matches a broader published ABI.

“Pool,” “available record,” and “region” describe inferred intent. They do not add validation, memory-management guarantees, or operations absent from the observed behavior.

## Evidence

Established static evidence consists solely of the supplied function body and its opaque dependency mapping.

That evidence supports:

- The four-word state access layout.
- The three-word descriptor layout.
- The 20-byte cursor stride.
- Available-list precedence and early return.
- Equality-only growth detection.
- Zero-versus-nonzero mode selection.
- Unsigned capacity calculations and byte-size multiplication.
- Two ordered dependency calls.
- Descriptor initialization and state-update ordering.
- The absence of explicit validation, release, rollback, and status reporting.

No dependency implementation, caller behavior, runtime traces, or runtime tests were provided. No runtime tests have been performed.
