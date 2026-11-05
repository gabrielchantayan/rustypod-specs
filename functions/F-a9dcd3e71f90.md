<!-- Trusted provenance: F-a9dcd3e71f90; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a9dcd3e71f90

## Purpose

F-a9dcd3e71f90 conditionally appends a three-word record to a container, creating the container when none is supplied and growing its backing storage when the recorded capacity is not greater than the recorded count.

The function has two distinct successful behaviors:

- If either of the two record inputs is nonzero, it prepares one record and increments the container’s count.
- If both record inputs are zero, it returns the container without appending a record. Container creation and capacity growth can still occur.

An allocation or growth failure takes a common cleanup path: the third input is passed to one dependency, the current container pointer is passed to another dependency, and the function returns a null pointer.

The interpretation of the container as a collection and of its fields as count, capacity, and storage is inferred from their use. The reads, writes, calls, and branch conditions below are established by the supplied static representation. No runtime tests have been performed.

## Interface

### Inputs

The function accepts four inputs:

1. **Context-word address.** An address from which the function reads one 32-bit word immediately upon entry. That word is retained and supplied to allocation, growth, and record-conversion dependencies. The function does not reread this address.
2. **Container pointer.** Either a null pointer or an address of a container header with the layout described below.
3. **Primary record value.** A 32-bit integer-like value. It controls whether an append is requested, is stored in a successful new record, and is passed to a cleanup dependency on allocation or growth failure.
4. **Secondary record input.** A 32-bit integer-like value. It also controls whether an append is requested and is passed to a dependency whose result becomes the new record’s second word.

The available representation uses 32-bit words and integer-like address storage. Pointer arithmetic and record layouts are described for that represented model; a portable ABI or a wider-pointer implementation is not established.

### Container layout

| Byte offset | Width | Observed use |
|---|---:|---|
| 0 | 4 bytes | Signed comparison operand; current record count; incremented when appending |
| 4 | 4 bytes | Signed comparison operand; current capacity; replaced after successful growth |
| 8 | 4 bytes | Not directly read or written by this function |
| 12 | 4 bytes | Backing-storage address; supplied to growth, replaced on successful growth, and used to locate a new record |

The container-creation request is for 16 bytes. This function does not explicitly initialize the header after that request succeeds.

### Record layout

Each record occupies 12 bytes:

| Byte offset within record | Width | New-record behavior |
|---|---:|---|
| 0 | 4 bytes | Cleared, then assigned the primary record value |
| 4 | 4 bytes | Cleared, then assigned the conversion dependency’s result |
| 8 | 4 bytes | Cleared; no subsequent direct write by this function |

These are word-sized stores, not a specified atomic or transactional update.

### Output

The function returns:

- The supplied container pointer on normal success when a container was provided.
- The newly obtained container pointer on normal success when the supplied pointer was null.
- A null pointer after either a container-creation failure or a backing-storage growth failure.

A zero result from the record-conversion dependency does not itself cause failure.

### Preconditions

For defined ordinary operation:

- The context-word address must be readable even if no append will occur.
- A nonnull supplied container must provide readable header fields and writable fields wherever the selected path writes them.
- Its count, capacity, and backing-storage address must describe storage suitable for the selected path.
- The record selected for an append must be writable for 12 bytes.
- Arithmetic used for growth, count increment, and record addressing must have valid target-specific results.

The function does not validate these conditions. It does not reject negative counts or capacities, detect overflow, or verify that the header’s recorded capacity matches an actual allocation.

## Behavior

### 1. Capture the context word

The first observable direct access is a read of the 32-bit word at the first input address.

This occurs before checking whether a container exists and before checking whether either record input is nonzero. A call with two zero record inputs is therefore not a context-free no-op.

Later dependencies receive the captured word. If a dependency changes the memory from which it was read, this function still uses the original captured value.

### 2. Obtain a container when necessary

If the supplied container pointer is nonnull, container creation is skipped.

If it is null, the function calls F-f053f140662a with:

- The captured context word.
- A size of 16 bytes.

A null result immediately selects the common failure path.

A nonnull result becomes the working container pointer. The function proceeds directly to inspect its header. It does not first clear the header, establish a zero count, or initialize the backing-storage address.

Consequently, successful creation depends on the dependency returning a suitably initialized object or otherwise satisfying the subsequent accesses. Merely returning 16 bytes of uninitialized storage would not establish the required header state.

### 3. Check capacity before deciding whether to append

The function compares the capacity field with the count field using a signed comparison.

If capacity is strictly greater than count, growth is skipped.

If capacity is less than or equal to count, the function:

1. Calculates a requested new capacity as twice the old capacity plus four.
2. Calculates a requested storage size as that new capacity multiplied by 12.
3. Calls F-eb8708ae5ace with the captured context word, the current backing-storage address, and the calculated byte count.

A zero result selects the common failure path.

A nonzero result is stored into the container’s backing-storage field. The calculated new capacity is then stored into the capacity field, in that order.

There is only one growth attempt. The function does not repeatedly grow until capacity exceeds count. For malformed input such as a count far above capacity, the calculated capacity can remain insufficient.

The growth decision precedes the append decision. Therefore, a call with both record inputs zero can still allocate backing storage, replace the backing-storage field, update capacity, or fail.

### 4. Decide whether to append

An append occurs if at least one of the primary record value and secondary record input is nonzero.

If both are zero, the function performs no direct record initialization and no direct count increment. It returns the working container pointer after any creation or growth described above.

The conversion dependency is not called on this zero/zero path.

### 5. Reserve and clear the new record

For an append, the function reads the current count and writes back that count plus one.

It then obtains the backing-storage address and selects the record at:

- The backing-storage base, plus
- Twelve times the count read before the increment.

The count increment precedes the record stores and the conversion dependency call. The function does not defer the count update until the record is complete.

It clears the selected record’s three words in ascending field order: first word, second word, third word.

There is no rollback path for this count increment or these stores within the function.

### 6. Obtain and store the secondary value

After the three zero stores, the function calls F-e7c4e7dd076d with:

- The captured context word.
- The secondary record input.

This call occurs whenever an append is selected, including when the secondary input is zero and only the primary value is nonzero.

After the dependency returns, its 32-bit result is written to the selected record’s second word. The primary record value is then written to the selected record’s first word.

Thus, absent dependency-induced changes, the resulting record contains:

1. The primary record value.
2. The conversion result.
3. Zero.

The record address is determined before the conversion call and retained across it. The function does not reload the container’s backing-storage address or recalculate the record address afterward.

No contract for the conversion dependency establishes that it preserves the container, backing storage, or global state. If it changes or invalidates the selected storage, the subsequent stores still target the previously selected address.

### 7. Return success

After either the zero/zero path or a completed append, the function returns the working container pointer.

It does not examine the conversion result for an error indicator. In particular, a zero conversion result is stored as a normal second-word value.

### 8. Common failure path

Container-creation failure and backing-storage growth failure use the same sequence:

1. Call F-841dc33ee157 with the primary record value.
2. Call F-e5380ae57ea1 with the current working container pointer.
3. Return a null pointer.

Both calls occur regardless of whether the primary value or container pointer is zero.

On creation failure, the container argument to F-e5380ae57ea1 is null.

On growth failure, it is the supplied or newly obtained container pointer. The function has not directly replaced the backing-storage field or capacity with the failed growth result.

The secondary record input is not passed to either cleanup dependency. No append conversion occurs on these failure paths.

## Side effects

### Direct memory effects

Depending on the path, the function directly:

- Reads the context word.
- Reads count and capacity.
- Reads the backing-storage address.
- Replaces backing-storage and capacity after successful growth.
- Increments count before initializing a new record.
- Clears three record words.
- Stores the conversion result and primary record value.

The header word at byte offset 8 is untouched by direct accesses in this function. This does not establish preservation across dependency calls.

For ordinary distinct, valid storage, the direct stores occur in this order:

1. Successful growth: backing-storage field, then capacity field.
2. Append: count field.
3. New record: first word zero, second word zero, third word zero.
4. After conversion: second-word result, then first-word primary value.

This is an ordering description, not a concurrency guarantee. Atomicity, memory barriers, and thread safety are not established.

### Ownership and lifetime

The static call pattern does not establish ownership contracts.

- The primary value is stored directly on success; no separate retain or duplication call is visible.
- On allocation or growth failure, it is passed to F-841dc33ee157.
- The container pointer is passed to F-e5380ae57ea1 on failure, including when it was supplied by the caller.
- The growth dependency receives the existing backing-storage address.
- The conversion dependency’s result is placed into the record without further validation.

These facts do not prove that any dependency frees, retains, transfers, duplicates, or preserves memory. In particular, callers cannot infer that a container remains usable after a null return merely because it existed before the call.

### Partial visibility and aliasing

During conversion, the function has already incremented count and cleared the new record. An observer reached through that dependency may therefore encounter an incomplete appended record.

The final third word is only guaranteed to remain zero if no intervening dependency or aliased access changes it. This function does not clear it again after conversion.

No alias restrictions are supplied. Claims about distinct header fields and record contents assume valid storage without destructive overlap.

## Dependencies

| Opaque ID | Observed invocation | Established interpretation limits |
|---|---|---|
| F-f053f140662a | Captured context word and 16-byte request, only when the container input is null | Null means creation failure to this function. Initialization and ownership behavior are unknown. |
| F-eb8708ae5ace | Captured context word, existing backing-storage address, calculated byte count | Zero means growth failure. Preservation, relocation, allocation, and release semantics are unknown. |
| F-e7c4e7dd076d | Captured context word and secondary record input, once per append | Its result is stored as a 32-bit record value, including zero. Other effects are unknown. |
| F-841dc33ee157 | Primary record value on either allocation failure | Called before container cleanup. Release or disposal semantics are not established. |
| F-e5380ae57ea1 | Current container pointer on either allocation failure | May receive null. Destruction, null tolerance, and ownership semantics are unknown. |

Successful completion assumes the invoked dependencies return normally. Behavior after a dependency traps, aborts, or performs a nonlocal transfer is outside the established contract.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. Unless stated otherwise, they assume valid, nonoverlapping storage, representable arithmetic, normal dependency returns, and no unexpected dependency mutation of relevant memory.

1. **Append with spare capacity.** Given count 2, capacity 5, and a nonzero primary value, no creation or growth call occurs. Count becomes 3. The record beginning 24 bytes after the backing-storage base is cleared, conversion is called even if the secondary input is zero, and the resulting record receives the primary value and conversion result. The original container pointer is returned.

2. **Secondary-only append.** Given spare capacity, a zero primary value, and a nonzero secondary input, an append still occurs. The first record word ends as zero; the second receives the conversion result.

3. **Successful zero conversion result.** Given an append request and a conversion result of zero, the function returns the container rather than entering cleanup. The new record’s second word is zero.

4. **Zero/zero call with spare capacity.** Given count 1 and capacity 3, both record inputs zero, and an existing container, the function reads the context word and checks capacity but performs no direct header or record writes and makes no dependency call.

5. **Zero/zero call requiring growth.** Given count 2, capacity 2, and both record inputs zero, requested capacity is 8 and requested storage size is 96 bytes. On growth success, backing storage and capacity are updated, but count remains 2 and conversion is not called.

6. **Container-creation failure.** Given a null container and a null creation result, F-841dc33ee157 receives the primary value, then F-e5380ae57ea1 receives null. The function returns null without calling growth or conversion.

7. **Growth failure with an existing container.** Given count equal to capacity and a zero growth result, the primary cleanup call precedes the container cleanup call. No direct count increment or record initialization occurs. The function returns null.

8. **No repeated growth for inconsistent metadata.** Given count 100 and capacity 1, the single requested capacity is 6 and the byte request is 72. The function does not retry growth merely because 6 remains below 100. An append under those conditions is not established as memory-safe.

9. **Conversion-time visibility.** Given an append from count 0, conversion is invoked only after count has become 1 and all three selected record words have been cleared. The primary value is not written into that record until conversion returns.

## Uncertainties

The following are **documented_only**:

- Exact external ABI, pointer encoding, and allocator family.
- Whether container creation initializes the header and, if so, to what values.
- Whether the growth dependency preserves old storage on failure or initializes newly available storage.
- Ownership and null-handling rules of both cleanup dependencies.
- Meaning, lifetime, and ownership of the conversion result.
- Dependency effects on globals, the container, backing storage, and aliases.
- Error behavior for invalid pointers, inaccessible storage, or inconsistent metadata.
- Exact overflow behavior for signed arithmetic.

The represented operations use signed integer expressions for doubling capacity, adding four, multiplying by 12, incrementing count, and forming the record offset. No explicit overflow checks are present. It would be unsupported to promise either safe saturation or a particular wraparound result for out-of-range inputs.

A resizable record collection is the inferred intent. Cleanup names, allocation guarantees, and safe handling of malformed states are not inferred from that intent.

## Evidence

The supplied static representation establishes the branch conditions, 16-byte creation request, four-word header accesses, 12-byte record stride, growth formula, direct store ordering, dependency arguments, and shared failure path described above.

It also establishes two important absences: no explicit header initialization after creation and no conversion-result failure check.

The representation does not provide dependency implementations, caller contracts, runtime traces, or authoritative ownership annotations. No runtime tests have been performed; all acceptance examples are conditional specifications derived from static evidence.
