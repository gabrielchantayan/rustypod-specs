<!-- Trusted provenance: F-8acbfc8a8dc9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8acbfc8a8dc9

## Purpose

F-8acbfc8a8dc9 replaces a target object's element collection and associated cumulative-value array, derives a classification value from the replacement elements, and conditionally inserts the target at the head of a globally selected linked list.

The established behavior is an ordered sequence of dependency calls, memory reads, and memory writes. The apparent intent is to acquire replacement elements before releasing old elements, rebuild aggregate metadata, and register an eligible object. Acquisition, release, allocation, and registration terminology describes inferred roles; the dependencies’ actual ownership contracts are not supplied.

This operation is not visibly transactional. It changes externally accessible state before requesting replacement storage, provides no local allocation-failure handling, and has no rollback path.

## Interface

### Inputs

The function accepts five 32-bit arguments:

| Input | Interpretation supported by the supplied evidence |
|---|---|
| Target | Address of the mutable object being rebuilt. |
| Initial cumulative value | A 32-bit value placed in the first entry of the replacement cumulative array. |
| Replacement count | An unsigned 32-bit element count. |
| Increment-array address | Address of a sequence of 32-bit increments, one per replacement element. |
| Element-array address | Address of a sequence of 32-bit element addresses, one per replacement element. |

There is no explicit return value. Successful completion is observable through memory changes and dependency calls.

The supplied representation uses four-byte address-bearing values and four-byte array entries. Byte order and alignment requirements are not established independently of that representation.

### Target layout

The function directly accesses these target-relative locations:

| Byte offset | Width | Observed role |
|---|---:|---|
| `+0x08` | 32 bits | Maximum element classification plus one, with 32-bit arithmetic. |
| `+0x0c` | 32 bits | Backward link when inserting into a list. |
| `+0x10` | 32 bits | Forward link when inserting into a list. |
| `+0x14` | 8 bits | Flags byte; its least significant bit controls insertion and is set on insertion. |
| `+0x18` | 32 bits | State field reset to zero. |
| `+0x1c` | 32 bits | Old cleanup counter, subsequently replaced by the replacement count. |
| `+0x20` | 32 bits | Address of the cumulative-value array. |
| `+0x24` | 32 bits | Address of the element-address array. |

Each replacement element is dereferenced at element-relative offset `+0x08` to obtain a 32-bit unsigned classification value.

A global cell contains the address of a table of four-byte list-head entries. The address of that global cell is captured before the first dependency call. Its contents are read initially for a dependency argument and read again after rebuilding the target.

### Preconditions for ordinary completion

The routine performs no explicit validation of:

- The target address or its writable extent.
- Either input-array address.
- Replacement element addresses.
- Existing array addresses or the old count.
- Allocation results.
- List-table extent or existing list links.
- Allocation-size overflow.

For the ordinary non-overlapping case, the caller must provide readable replacement arrays covering the requested count, readable replacement elements through offset `+0x08`, and a writable target covering the accessed fields. An insertion additionally requires a valid table entry and, when nonzero, a writable old head through offset `+0x0c`.

Zero replacement count avoids reads from both input arrays and avoids replacement-element dereferences. It does not avoid dependency calls, storage requests, target writes, or possible list insertion.

## Behavior

### 1. Initial notification and replacement-element calls

The function captures the address of the global table-address cell. It reads that cell and calls F-03b9616af95f with:

1. The value just read from the global cell.
2. The target address.

It then visits replacement element-array entries in ascending index order, from zero through replacement count minus one. For each entry, it reads the four-byte value currently at that input position and passes it to F-2609ef4ae406.

A zero entry is not filtered out at this stage.

All these calls precede cleanup of the old collection. This ordering supports an acquire-before-release interpretation, but does not establish that either dependency implements reference counting.

The input element values are not saved as a complete snapshot. They are read again during reconstruction. If a dependency modifies the input array, the later reconstruction can use different values from those passed to F-2609ef4ae406.

### 2. Old-collection cleanup

Cleanup repeatedly reads the target's current 32-bit count field at `+0x1c`, subtracts one, and writes the result back before checking whether the value originally read was zero.

If the value originally read was zero, cleanup ends immediately. Consequently, even an initially empty collection receives a transient count value of `0xffffffff`.

Otherwise, the decremented value selects a four-byte entry from the array whose address is read from target offset `+0x24`. If that entry is nonzero, the function calls F-029930293dcb with no explicit arguments shown in the supplied evidence. A zero entry suppresses that call.

Under stable, valid, nonnegative old state, entries are examined in descending index order. For an old count of three, the entry order is two, one, zero, followed by a final count read of zero and write of `0xffffffff`.

This is not a snapshot-based traversal. Each iteration rereads the target count and array address. A dependency that changes either can alter subsequent traversal, termination, or memory accesses. In particular, the evidence does not justify treating the nonzero entry as an explicit argument to F-029930293dcb.

Negative old counts have no visible rejection path. Their decremented bit patterns are used in address calculation. Such inputs are outside the ordinary valid-state interpretation and can cause invalid accesses or unexpectedly long traversal.

### 3. Old-storage calls and replacement allocations

After cleanup ends, the function performs these operations in order:

1. Reads the address at target offset `+0x24` and passes it to F-58ba61416cc9.
2. Reads the address then present at target offset `+0x20` and passes it to F-58ba61416cc9.
3. Calls F-fab3799b3190 with the replacement count multiplied by four.
4. Stores that call's result at target offset `+0x24`.
5. Calls F-fab3799b3190 with the replacement count plus one, multiplied by four.
6. Stores that result at target offset `+0x20`.
7. Writes the initial cumulative value into the first four-byte entry of this second result.

The second old-storage address is read after the first F-58ba61416cc9 call. It must not be assumed to equal its value before that call if the dependency can modify target state.

The first replacement-array address is installed before the second allocation call. A dependency observing the target at that point may therefore see partially replaced storage.

Both allocation-size expressions are 32-bit computations. Their mathematical sizes are not checked for representability. For example, replacement count `0x40000000` produces a first requested size of zero and a second requested size of four. Replacement count `0xffffffff` produces a second requested size of zero. No local guard prevents later accesses based on those undersized results.

The function checks neither returned address. The second result is written through immediately, including when the replacement count is zero.

### 4. Target initialization and reconstruction

After writing the first cumulative entry, the function writes, in order:

1. Zero to target offset `+0x18`.
2. The replacement count to target offset `+0x1c`.
3. Zero to target offset `+0x08`.

It then processes replacement positions in ascending order. For each position:

- It rereads the corresponding input element address and writes it into the target's current element array.
- It writes the next cumulative entry as the preceding cumulative entry plus the corresponding 32-bit increment.
- It reads the stored element address through the target's element array, then reads that element's unsigned classification at offset `+0x08`.
- It compares that classification with the unsigned value currently at target offset `+0x08`, replacing the target value only when the element value is greater.

Thus, under non-overlapping stable storage, cumulative entry zero is the supplied initial value, and each subsequent entry incorporates the next increment. There are replacement count plus one entries.

The cumulative arithmetic has no explicit overflow check or error branch. The supplied representation uses signed 32-bit operands for these additions, but does not establish a portable language-level overflow guarantee. Any specification requiring defined behavior beyond the signed range needs an additional arithmetic assumption.

After all elements are processed, the function increments the target's classification field by one. At the represented 32-bit width, a maximum classification of `0xffffffff` becomes zero.

For a zero-element replacement, the pre-increment maximum remains zero, so the resulting field is one.

The reconstruction includes no dependency calls. Aliasing can nevertheless change the result: writes to the target or replacement arrays can overlap input entries, metadata, elements, or other destinations. The ordinary aggregate description assumes these regions do not overlap destructively.

### 5. Conditional list insertion

After reconstruction, the function reads the table address again from the previously captured global cell. This read occurs even if insertion will subsequently be skipped.

It computes an index by subtracting one from the target's current classification field using 32-bit arithmetic. In the ordinary non-overlapping case, this recovers the unsigned maximum element classification, or zero for an empty replacement.

Insertion is skipped when either:

- The index has its high bit set, making its signed 32-bit interpretation negative.
- The index is nonnegative and the least significant bit of the target's flags byte is already set.

When the index is negative, the flags byte is not needed for that decision. No table entry is accessed on this skip path.

When insertion proceeds, observable operations occur in this order:

1. Write zero to the target's backward link at `+0x0c`.
2. Read the selected table entry and store it in the target's forward link at `+0x10`.
3. Write the target address into that table entry.
4. Read the target's flags byte and write it back with bit zero set, preserving the other bits.
5. Read the target's forward link.
6. If that link is nonzero, write the target address to the linked object's backward link at `+0x0c`.

There is no visible list-length validation, duplicate detection beyond the flag test, synchronization, or rollback.

If insertion is skipped, this final stage does not write either target link or the flags byte. Earlier dependencies may still have changed them.

## Side effects

Established direct side effects include:

- Incremental mutation of the old count during cleanup.
- Replacement of both array-address fields.
- Initialization of replacement-array contents.
- Reset of the field at `+0x18`.
- Replacement of the count and classification fields.
- Conditional modification of the target's links and flags.
- Conditional modification of one global table entry and one previous head's backward link.

Dependency side effects are unknown. They may affect the target, input arrays, global table cell, storage validity, or other state. The contract does not assume that calls preserve globals or target fields.

Ownership is only partially observable. Old array addresses are passed to F-58ba61416cc9, and addresses returned by F-fab3799b3190 are installed in the target. The function does not itself establish allocator compatibility, reference ownership, lifetime guarantees, or whether input-array storage remains caller-owned.

There is no explicit error result or local failure recovery. An inaccessible address or unusable allocation result leads to whatever failure the execution environment produces; no specific exception, signal, or diagnostic is established.

## Dependencies

| Opaque ID | Established use | Unestablished interpretation |
|---|---|---|
| F-03b9616af95f | Called once initially with the current global table address and target. | Possibly detaches or prepares the target; exact behavior unknown. |
| F-2609ef4ae406 | Called once per replacement element-array entry, in ascending order. | Possibly acquires or retains an element. |
| F-029930293dcb | Called without visible explicit arguments for each nonzero old-array entry encountered. | Possibly releases old element state; argument association and ownership effects unknown. |
| F-58ba61416cc9 | Called for the old element-array address, then the old cumulative-array address. | Possibly releases storage; null handling and failure behavior unknown. |
| F-fab3799b3190 | Called twice with 32-bit byte-size expressions; returned values become array addresses. | Possibly allocates storage; zero-size behavior, initialization, and failure semantics unknown. |

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests. They assume valid non-overlapping storage, usable allocation results, and dependencies that do not unexpectedly mutate relevant state.

### Ordinary replacement and insertion

Given replacement count three, initial cumulative value 10, increments 2, 5, and −1, and element classifications 4, 2, and 7:

- The cumulative entries become 10, 12, 17, and 16.
- The target count becomes three.
- The state field at `+0x18` becomes zero.
- The classification field becomes eight.
- If flag bit zero is clear, insertion selects table entry seven.
- The target becomes that entry's head, its backward link becomes zero, and the previous nonzero head receives the target as its backward link.

### Empty replacement

Given replacement count zero:

- No replacement-element acquisition calls occur.
- Neither input array is read.
- Allocation requests are zero bytes and four bytes, respectively.
- The cumulative array's first entry receives the initial cumulative value.
- The count becomes zero and classification becomes one.
- If flag bit zero is clear, insertion uses table entry zero.

Completion depends on the dependencies' zero-size and returned-address behavior.

### Old entries containing zero

Given an old count of three and old entries, from lowest to highest index, of nonzero, zero, nonzero:

- Cleanup examines indices two, one, zero.
- F-029930293dcb is called for indices two and zero only.
- The count is decremented before each examination.
- A final zero-to-`0xffffffff` count write occurs before the old-storage calls.

### Already flagged target

Given a nonnegative insertion index and flag bit zero already set when the final decision is made:

- Reconstruction still completes.
- The global table-address cell is still reread.
- The final insertion stage leaves the target links, flags, and table entries unchanged.

This does not constrain changes made by the initial dependency.

### High-bit classification

Given maximum classification `0x80000000`, the stored classification becomes `0x80000001`, and final insertion is skipped because the recovered index is negative under signed interpretation.

Given maximum classification `0xffffffff`, the stored classification wraps to zero at 32 bits, the recovered index is `0xffffffff`, and insertion is likewise skipped.

## Uncertainties

The following remain **documented_only**:

- Exact dependency contracts, including ownership, reentrancy, exceptions, and global-state preservation.
- Meaning of the initial notification and the no-explicit-argument cleanup call.
- Allocator behavior for zero requests or failure.
- Signed cumulative-overflow semantics beyond what the supplied representation establishes.
- Required alignment, byte order, table capacity, and valid classification range.
- Whether callers prohibit overlapping storage, malformed old counts, cycles, or inconsistent membership flags.
- Thread-safety and visibility to concurrent observers.

The inferred acquire/release and list-membership roles do not replace these missing contracts.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Evidence establishes explicit call order, field widths and offsets, loop conditions, memory accesses, unsigned classification comparisons, allocation-size expressions, and conditional link writes.

No dependency implementations, caller contracts, runtime traces, or independent layout definitions were supplied. No runtime tests have been performed. Acceptance examples are conditional consequences of the static evidence, not observed results.
