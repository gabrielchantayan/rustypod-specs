<!-- Trusted provenance: F-2f5fbad4fdef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2f5fbad4fdef

## Purpose

F-2f5fbad4fdef conditionally copies a 32-bit value from an object header into a field of one indexed record.

The condition uses a signed index, a signed 32-bit header value, and a record-storage address held in the object. A write occurs only when the object address is nonzero, the index is nonnegative and strictly less than the header value observed during the bounds decision, and the record-storage address passes a nonzero check.

The apparent intent is to stamp a record with an object-level count or limit. That interpretation is inferred, not established: the supplied material identifies offsets and operations but does not establish the semantic names of either field.

The function returns no value and reports no status. A skipped operation is indistinguishable from a completed write through the return interface.

## Interface

The interface has four positional inputs:

| Position | Contract |
|---|---|
| First | Object address represented by an integer in the supplied interface. Zero causes an immediate return. A nonzero value is not validated before header access. |
| Second | Signed 32-bit record index. Negative values cause a return without accessing object memory. |
| Third | A 32-bit input that does not affect the visible behavior. |
| Fourth | An integer input whose incoming value does not affect the visible behavior. For nonnegative indices, the bounds decision uses the header value read from the object instead. |

There is no return value, output parameter, callback, or explicit error object.

The supplied representation uses 32-bit address-bearing fields. Exact calling convention, native pointer representation, and any relationship between integer addresses and a wider host address space are not established. The memory layout below describes the represented address model; it is not a portable host-language structure definition.

### Required memory layout

Offsets are byte offsets from the object address:

| Object offset | Width | Interpretation |
|---|---:|---|
| `0x0C` | 4 bytes | Header value, interpreted as signed 32-bit for the bounds decision and copied as a 32-bit value if a write occurs. |
| `0x14` | 4 bytes | Address of indexed record storage. Zero disables the write after the bounds decision succeeds. |

Records have a stride of `0x14` bytes, or 20 bytes. The destination field is four bytes wide and begins eight bytes into the selected record.

For an index \(i\) and record-storage address \(B\), the intended destination address is:

\[
B + 20i + 8
\]

The function does not read the prior contents of that destination field and does not explicitly access any other record fields.

### Preconditions for valid memory access

The caller must provide valid memory for every access reached by the selected branch:

- A zero object address requires no object memory.
- A negative index requires no object memory, even when the supplied object address is nonzero.
- A nonzero object address with a nonnegative index requires a readable four-byte field at offset `0x0C`.
- An index that passes the bounds decision additionally requires a readable four-byte address field at offset `0x14`.
- A successful nonzero storage check requires a valid writable four-byte destination and a readable header value for the final copy.

The routine does not establish allocation sizes, alignment suitability, record capacity, or address-arithmetic safety. Those remain caller or environment obligations.

## Behavior

### Early exits and bounds decision

A zero object address causes an immediate return. No object header or record storage is accessed on that path.

For a nonzero object address, a negative index also causes a return without reading either header field.

For a nonnegative index, the routine reads the signed 32-bit header value at object offset `0x0C`. Let \(L\) denote the value observed by this initial read.

The operation proceeds to the storage-address check only when:

\[
0 \leq i < L
\]

The strict upper bound is important:

- An index equal to \(L\) is rejected.
- An index greater than \(L\) is rejected.
- A zero or negative \(L\) permits no index.
- Zero is a valid index when \(L\) is positive.

The comparison is signed. A header bit pattern with its most significant bit set is therefore negative for this decision, not a large unsigned capacity.

When this bounds decision fails, the function returns without reading the storage-address field.

### Storage-address check

After the bounds decision succeeds, the routine reads the address field at object offset `0x14`.

If that observed address is zero, the function returns without an explicit record write. There is no allocation, initialization, retry, or alternate destination.

If that observed address is nonzero, the function proceeds to a single four-byte store into the indexed destination field.

### Value copied and destination selection

The value stored is obtained by reading the header field at offset `0x0C` again. It is not necessarily the same observation used for the earlier bounds decision.

The destination expression also obtains the record-storage address from offset `0x14` again. Consequently, the nonzero storage check and destination selection are distinct source-visible uses of that field.

Under stable, valid memory:

- The same header value supplies both the upper bound and the copied value.
- The same storage address supplies both the nonzero check and the destination base.
- Exactly the selected record’s four-byte field at offset eight is overwritten.

The supplied material does not establish an ordering between the final header read and the final storage-address read. Both contribute to the store, but a contract must not impose a relative order absent additional execution evidence.

The routine does not repeat its bounds decision using the final header read. It also does not repeat its nonzero decision using a separately validated destination address.

### Arithmetic and boundaries

The index and bounds value are signed 32-bit quantities. The record stride is 20 bytes, and the destination field offset is eight bytes.

The bounds decision does not by itself prove that destination arithmetic is safe. For example, a positive signed header value can admit an index for which multiplying by 20 exceeds the signed 32-bit range. A valid destination also depends on the base address, final offset, and four-byte store extent.

The supplied material does not establish whether overflow in these address expressions must wrap, fault, or be excluded by the execution model. Acceptance claims therefore require representable, valid address calculations. No portable behavior is specified for overflowing address arithmetic.

There is no special handling for the largest signed index. Because the header value cannot exceed `2,147,483,647` as a signed 32-bit value, that index cannot satisfy the strict upper bound.

## Side effects

The only explicit memory modification is one four-byte store on the successful path.

Under stable, nonoverlapping memory, the function does not:

- Change the header value used as the bound.
- Change the object’s storage-address field.
- Increment or decrement any count.
- Initialize other record fields.
- Allocate or release memory.
- Transfer ownership.
- Perform I/O or invoke another function.

These are statements about the visible operations, not guarantees that distinct logical fields occupy distinct physical memory. The routine imposes no non-aliasing check. If the calculated destination overlaps an object header field or another caller-visible location, the store modifies that overlapping memory. No restoration follows.

The routine retains no explicitly represented pointer or state for later use.

There is no explicit synchronization, atomic operation, lock, or memory-ordering protocol. The repeated reads must not be treated as a consistent snapshot. Behavior under concurrent unsynchronized mutation is not established.

Invalid addresses, inaccessible memory, unsuitable alignment, and arithmetic overflow have no explicit error-handling path. A platform fault is possible, but its precise form is outside the supplied evidence. The function provides no recoverable error result.

## Dependencies

No external function dependencies are visible in F-2f5fbad4fdef.

Its behavior depends directly on:

- Readable object-header memory.
- The signed interpretation of the initial four-byte bounds value.
- A readable record-storage address field after the bounds decision passes.
- Valid address arithmetic and writable destination memory for the successful path.
- The execution environment’s memory-access and fault behavior.

There is no visible callee whose ownership, global-state preservation, or cleanup behavior needs to be assumed.

## Acceptance examples

These examples are conditional expectations derived from the supplied static material. They are not runtime test results. Unless stated otherwise, memory is stable, addresses are valid, arithmetic is representable, and the destination does not overlap the object header.

1. **Zero object address.**  
   Given a zero first input, any index, and arbitrary third and fourth inputs, the function returns without reading object or record memory and without writing memory.

2. **Negative index with a nonzero address.**  
   Given a nonzero object address and index `−1`, the function returns without reading either header field. The address need not be dereferenced on this path.

3. **Index equal to the bound.**  
   Given an initial header value of `3` and index `3`, the function reads the header value and returns. It does not read the storage-address field or write a record.

4. **Index greater than the bound.**  
   Given an initial header value of `3` and index `4`, the operation is skipped without consulting record storage.

5. **Zero or negative bound.**  
   Given index `0` and a header value of `0` or `−1`, the function returns after the bounds read. No nonnegative index is accepted for either header value.

6. **Null record storage after a valid index.**  
   Given header value `3`, index `1`, and a zero storage-address field, the function reads the bound and storage-address field, then returns without a record write.

7. **First record.**  
   Given header value `3`, index `0`, and valid nonzero storage address \(B\), the four bytes beginning at \(B + 8\) are replaced with the 32-bit header value `3`. Other nonoverlapping record bytes remain unchanged.

8. **Last accepted record.**  
   Given header value `3` and index `2`, the destination begins at \(B + 48\): two 20-byte strides plus the eight-byte field offset. The stored value is `3`.

9. **Unused input independence.**  
   With the first two inputs and all accessed memory unchanged, varying the third or incoming fourth input does not change which branch is taken or which value is written.

10. **Distinct header observations.**  
    If an execution environment validly permits the initial header read to observe `3` and the final copy read to observe `7`, index `2` passes using `3`, and the stored value is `7`. This illustrates the separate reads; it does not authorize unsynchronized concurrent mutation.

11. **Insufficient actual storage.**  
    A positive header value and nonzero storage address do not establish that the selected record exists. If the destination is outside writable storage, successful completion is not guaranteed and no explicit error result is supplied.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the header value and destination field.
- Whether the storage represents an array, a subregion of another allocation, or another layout with the same stride.
- Allocation ownership, lifetime rules, and actual record capacity.
- Exact target ABI, byte order, alignment requirements, and fault behavior.
- Address-overflow semantics.
- Whether repeated source-visible loads correspond one-for-one to physical memory transactions.
- Any valid synchronization protocol allowing the header or storage address to change during execution.
- The relative evaluation order of the final value read and destination-base read.
- Whether callers intentionally permit header/destination aliasing.

The names “bound,” “record,” and “storage” describe observed roles. They do not establish a higher-level data model.

## Evidence

Established static evidence consists of the supplied body for F-2f5fbad4fdef:

- An immediate return for a zero object address.
- No object dereference for negative indices.
- A signed header comparison using offset `0x0C`.
- A strict accepted index range from zero through one less than the initially observed header value.
- A zero-address check using offset `0x14`.
- A four-byte store at an address formed with a 20-byte stride and an eight-byte field offset.
- A fresh header-field reference for the copied value and a fresh storage-field reference for the destination.
- No visible calls, allocation, release, status return, or additional explicit writes.

The count-or-limit interpretation is inferred intent. Platform semantics and concurrency guarantees remain unknown.

No runtime tests have been performed.
