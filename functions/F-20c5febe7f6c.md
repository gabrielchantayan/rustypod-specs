<!-- Trusted provenance: F-20c5febe7f6c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-20c5febe7f6c

## Purpose

F-20c5febe7f6c selects or obtains a record, ensures that the record has a suitable storage address according to its own signed-length comparison, invokes a transfer dependency, and updates the record and an input cursor. It returns the selected record on the successful path and a null pointer on its explicit failure paths.

The apparent intent is to construct or replace a buffer-backed value while advancing a source cursor. That interpretation is inferred. The supplied evidence establishes field accesses, dependency calls, branches, and writes, but does not establish the record’s type, the transfer dependency’s semantics, or the ownership rules of the storage.

In particular, the comparison uses a field that is subsequently overwritten with the requested count. That field must not be described unconditionally as allocated capacity.

## Interface

### Inputs and result

The interface has three inputs:

1. **Optional record slot:** a pointer to a writable 32-bit slot holding a record address. The slot pointer itself may be null. When non-null, its current contents are read during selection and may be replaced on success.
2. **Cursor slot:** a pointer to a readable and writable 32-bit integer slot. Its value is captured after a record has been selected. That captured value is passed as the second transfer argument and is later increased by the requested count.
3. **Requested count:** a signed 32-bit integer.

The result is a record pointer, or null.

The supplied representation uses 32-bit record words and 32-bit address-bearing values. A native implementation or binding must preserve those widths and the signed interpretation of the count comparison. The evidence does not independently establish byte order, alignment requirements, or a portable host-language ABI.

### Record layout

Only the following fields are directly relevant:

| Byte offset | Width | Established use |
|---|---:|---|
| +0 | 32 bits | Written as zero after the transfer dependency returns |
| +4 | 32 bits | Written as zero near the end of the successful update |
| +8 | 32 bits | Not directly accessed |
| +12 | 32 bits | Compared as a signed integer against the requested count; overwritten with that count after transfer |
| +16 | 32 bits | Tested for zero, supplied as the transfer destination argument, and overwritten with an allocation dependency’s result |
| +20 | 32 bits | Bit 0 controls eligibility of an initially supplied record; bit 3 is set after an allocation attempt |

The names “storage,” “length,” and “flags” are descriptive conveniences, not established type declarations.

### Preconditions

A non-null record slot must be readable during initial selection. It must also be writable if execution reaches successful publication. On the allocation-failure path, it can be read again after the error dependency returns.

The cursor slot must be readable once selection produces a non-null record, and writable if the operation reaches its final cursor update. A null cursor slot is not explicitly rejected; reaching its read with an invalid address is outside the established valid-input contract.

Any selected record must support the field accesses taken on its path. Dependencies must leave the record and relevant slots usable for subsequent accesses if normal completion is expected. No preservation guarantee for those dependencies is supplied.

No explicit nonnegative-count check exists. Negative counts therefore remain part of the observed control flow, although safe use of them by dependencies is undocumented.

## Behavior

### 1. Record selection

When the optional record slot is non-null, the function reads its current record address.

If that address is non-null, the function reads the record’s flags at offset +20. It retains this record only when bit 0 is set.

In every other case, it invokes **F-c42e42f98364** without explicit arguments:

- the record slot is null;
- the slot contains null; or
- the slot contains a record whose bit 0 is clear.

The returned address becomes the selected record. An initially supplied but ineligible record is not explicitly released, modified, or passed to a dependency by this selection logic.

If F-c42e42f98364 returns null, the function returns null. On that path, it does not read the cursor slot, issue a transfer, or explicitly publish a new record address.

The bit-0 eligibility test is not repeated on a record returned by F-c42e42f98364.

### 2. Cursor capture and storage decision

After selecting a non-null record, the function reads and saves the cursor slot’s 32-bit value. This occurs before any possible call to F-5990a26eaaa3 or F-43da6b23bd44.

It then reads the record’s storage field at offset +16.

If storage is nonzero, it reads the field at offset +12 and compares it with the requested count using signed 32-bit ordering:

- If the requested count is less than or equal to that field, execution proceeds directly to transfer.
- If the requested count is greater, the function calls **F-5990a26eaaa3**, without explicit arguments, and then proceeds to allocation.

If storage is zero, the function proceeds directly to allocation. The field at offset +12 is not needed for that decision.

F-5990a26eaaa3 is not established as a deallocator. Its implicit inputs, global effects, and storage effects are unknown. The function does not explicitly clear the storage field before calling it.

### 3. Allocation path

The function computes the first allocation argument from the requested count:

- A requested count of zero produces an allocation argument of one.
- Every other requested count, including a negative one, is passed unchanged.

It calls **F-43da6b23bd44** with three explicit arguments: that computed value, zero, and zero.

After the dependency returns, the function performs these writes in order:

1. Store the returned 32-bit value in the record’s storage field at offset +16.
2. Read the record’s current flags at offset +20, set bit 3, and write the result back.

The flags update occurs even when the allocation result is zero. Because the flags are read after the dependency returns, dependency-induced flag changes can affect the value retained by this update.

A nonzero allocation result leads to transfer. A zero allocation result leads to error reporting and possible record release.

### 4. Transfer and successful update

The transfer dependency is identified here as **F-unmapped-01**, a document-local opaque identifier because its identity is absent from the supplied dependency map.

It receives three arguments:

1. The selected record’s storage field, read for this call.
2. The cursor value captured earlier.
3. The original requested count.

This call occurs on both the storage-reuse and successful-allocation paths. It also occurs for a requested count of zero.

The transfer dependency’s return value is not inspected. Consequently, reaching the successful update is not conditional on a documented transfer status. If the dependency returns normally, the caller proceeds without checking whether data was transferred.

After it returns, the function performs these explicit writes in order:

1. Write zero at record offset +0.
2. Write the requested count at record offset +12.
3. Write zero at record offset +4.
4. If the optional record slot is non-null, write the selected record address to that slot.
5. Write the saved cursor plus the requested count to the cursor slot.

It then returns the selected record address.

The final cursor is based on the saved value, not a reread of the cursor slot. Changes to that slot made by an intervening dependency do not replace the saved arithmetic input.

The signed addition requires an arithmetic qualification: the supplied expression does not establish a safe result when the mathematical sum exceeds the signed 32-bit range. Wrapping, trapping, or another target-level outcome must not be asserted from this evidence alone.

### 5. Allocation failure

If F-43da6b23bd44 returns zero, the function first calls **F-4fc89ec41a6d** with these five explicit argument values, in order:

- 13
- 147
- 65
- 0
- 0

After that dependency returns, the function decides whether to call **F-d82460213e2d**:

- If the optional record slot is null, it calls F-d82460213e2d with the selected record address.
- If the slot is non-null, it rereads the slot. It calls F-d82460213e2d only if the current slot value differs from the selected record address.

It then returns null.

This comparison uses the slot’s contents after error reporting, not its original contents. Effects of F-4fc89ec41a6d can therefore change whether the release-like dependency is called.

On this path, the function does not explicitly perform the successful record reset, publish a selected record, or advance the cursor. Nevertheless, allocation failure is not a rollback: before reporting the error, it has already stored zero in the storage field and set flag bit 3. Dependencies may introduce additional changes.

## Side effects

### Directly established effects

The function can directly modify:

- The selected record’s storage field.
- The selected record’s flags.
- Record fields at offsets +0, +4, and +12.
- The optional record slot.
- The cursor slot.

It does not directly access the field at offset +8.

The initial record-slot and flags reads precede record acquisition when acquisition is required. Cursor capture occurs only after successful selection. Error reporting precedes the final failure-path record-slot comparison.

### Aliasing and publication order

The evidence does not impose a non-aliasing restriction on the record slot, cursor slot, or record fields.

If these locations overlap, the listed write order matters. For example, publishing the record address into an overlapping cursor slot occurs before the final cursor write, so that publication can subsequently be overwritten. Likewise, an aliased record-field write can affect later reads.

The ordinary interpretation of “publish the record and advance the cursor” assumes distinct, suitably valid locations. More general aliasing must follow the actual ordered accesses, not an atomic-update model.

### Ownership and atomicity

No comprehensive ownership contract is established.

Calling F-d82460213e2d suggests record disposal, but its actual ownership effects are unknown. Calling F-5990a26eaaa3 suggests preparation or cleanup before replacement, but there are no explicit arguments establishing what it operates on.

The function must not be specified as preserving the old storage on allocation failure, freeing it exactly once, or transferring ownership of newly obtained storage. None of those claims follows from the available evidence.

The operation is not established as atomic, thread-safe, or reentrant.

## Dependencies

| Opaque ID | Explicit call interface observed here | Established caller behavior |
|---|---|---|
| F-c42e42f98364 | No explicit arguments; address-like result | Used when no eligible initial record is retained; null causes an immediate null return |
| F-5990a26eaaa3 | No explicit arguments | Called when existing storage is nonzero and the signed requested count exceeds the field at +12 |
| F-43da6b23bd44 | Adjusted count, zero, zero; 32-bit result | Result replaces storage; zero triggers the allocation-failure path |
| F-4fc89ec41a6d | 13, 147, 65, zero, zero | Called after a zero allocation result and before the failure-path slot comparison |
| F-d82460213e2d | Selected record address | Called on allocation failure unless a non-null record slot currently contains that same address |
| F-unmapped-01 | Current storage, saved cursor, original count | Called before successful field updates; return value ignored |

These interfaces describe only calls made by this function. They do not establish full dependency signatures, global-state preservation, allocation units, exception behavior, or memory-release responsibilities.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, assume valid distinct locations, normally returning dependencies, no dependency mutation of relevant state, and a representable final cursor sum.

### Eligible record with sufficient existing storage

Given a slot containing a record with bit 0 set, nonzero storage, field +12 equal to 16, saved cursor 100, and requested count 6:

- Neither acquisition nor allocation is called.
- F-5990a26eaaa3 is not called.
- F-unmapped-01 receives the existing storage, 100, and 6.
- The fields at +0 and +4 become zero; +12 becomes 6.
- The slot contains the selected record, the cursor becomes 106, and the result is that record.

### Existing storage with insufficient recorded length

Given an eligible record with nonzero storage, field +12 equal to 4, and requested count 9:

- F-5990a26eaaa3 runs before F-43da6b23bd44.
- Allocation receives 9, zero, zero.
- A nonzero allocation result is stored, bit 3 is set, and transfer precedes the successful updates.

### Zero count with missing storage

Given a selected record with zero storage and requested count zero:

- Allocation receives one, zero, zero.
- If allocation succeeds, transfer still receives a count of zero.
- Field +12 becomes zero.
- The final cursor equals its saved value.

A zero count does not bypass allocation when storage is absent.

### Ineligible initial record

Given a non-null initial record whose bit 0 is clear:

- F-c42e42f98364 is called.
- If it returns null, the cursor is not read.
- If it returns another record, that record is used without another bit-0 check.
- The original record is not explicitly disposed of merely because it was rejected.

### Allocation failure with retained slot identity

Given an allocation result of zero and a record slot still containing the selected record after F-4fc89ec41a6d returns:

- F-d82460213e2d is not called.
- No transfer occurs.
- The function returns null.
- The function has already written zero to storage and set bit 3.

If error reporting changes the slot to another value, the disposal decision changes accordingly.

### Negative count

Given nonzero storage, field +12 equal to 5, and requested count −1:

- The signed comparison selects reuse.
- Transfer receives −1 as its third argument.
- If it returns, field +12 becomes −1 and the saved cursor is decreased by one, provided the arithmetic is representable.

This establishes the caller’s lack of rejection, not the safety of the dependency call.

## Uncertainties

The following remain **documented_only**:

- The record’s semantic type and the meanings of its reset fields.
- Whether field +12 denotes length, capacity, or another limit.
- The actual transfer operation, overlap rules, count units, and negative-count behavior.
- Allocation units, initialization guarantees, and negative-size handling.
- Global state read or modified by dependencies.
- Whether either release-like dependency releases memory, changes reference counts, or performs unrelated work.
- Dependency failure modes, including exceptions or non-returning error handling.
- Arithmetic behavior outside the representable signed 32-bit sum range.
- ABI, alignment, byte order, synchronization, and supported aliasing constraints beyond the displayed accesses.

The inferred buffer-management intent does not resolve these unknowns.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the six-word record access pattern, signed comparison, zero-to-one allocation adjustment, ordered field writes, error arguments, conditional disposal call, cursor snapshot, and explicit return paths.

Buffer copying, capacity management, and ownership semantics are inferred intent or documented unknowns, not established dependency behavior.

No implementation execution, runtime tests, or external dependency inspection has been performed.
