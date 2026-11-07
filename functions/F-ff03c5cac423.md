<!-- Trusted provenance: F-ff03c5cac423; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff03c5cac423

## Purpose

F-ff03c5cac423 stores a caller-supplied 32-bit value in an object. A nonzero value ends processing immediately. A zero value can additionally trigger an allocator-like accounting and release sequence, depending on an object byte and the result of a dependency call.

The accounting sequence validates a preceding metadata word, subtracts a masked quantity from several counters, decrements two allocation-like counts, and invokes two final dependencies.

“Allocator-like,” “allocation,” and “release” describe inferred intent. The established behavior is the sequence of memory accesses and dependency calls below. The supplied evidence does not establish that memory is actually freed, that a lock is acquired or released, or that ownership transfers.

## Interface

**Inputs**

- **Object address:** An address represented by the first integer-valued argument. The function accesses fields at fixed byte offsets from this address.
- **Stored value:** A 32-bit value represented by the second argument. Its full representation is written to the object; zero versus nonzero controls further processing.

**Output**

- No explicit return value.
- Normal completion is by return after the applicable stores and calls.
- One validation-failure path invokes a dependency marked as nonreturning in the supplied representation.

**Object layout used**

| Byte offset | Width | Observed role |
|---|---:|---|
| `0x9c` | 32 bits | Destination for the supplied value |
| `0xa0` | 8 bits | Enables the additional sequence when nonzero |

No other object fields are directly accessed.

**Additional storage used**

A dependency can supply a nonzero address, referred to here as the **candidate address**. Metadata is accessed immediately before it:

| Offset relative to candidate address | Width | Observed role |
|---|---:|---|
| `-8` | 32 bits | Validation and masked accounting quantity |
| `-4` | 16 bits, unsigned | First accounting-table index |
| `-2` | 16 bits, unsigned | Second accounting-table index |

A separate global pointer slot supplies a **manager address**. Manager-relative storage is described under Behavior.

**Preconditions**

The object address must permit a 32-bit write at `+0x9c`, regardless of the supplied value. When the supplied value is zero, it must also permit an 8-bit read at `+0xa0`.

A zero object address is not safely rejected: the write occurs before any null comparison, and the byte read also precedes that comparison. There is no established null-input error result.

If the accounting path proceeds, the candidate metadata, global pointer slot, and all selected manager locations must be valid for the accesses described below. No local bounds, alignment, or overflow checks establish these conditions.

## Behavior

### 1. Store the supplied value

The first direct memory action is a 32-bit write of the supplied value to object offset `0x9c`.

If the supplied value is nonzero, the function returns. It does not directly read the byte at `0xa0`, obtain a candidate, or perform manager accounting on this path.

### 2. Decide whether to obtain a candidate

For a zero supplied value:

1. Read the byte at object offset `0xa0`.
2. If that byte is zero, return.
3. If that byte is nonzero, evaluate whether the object address is nonzero.
4. If the object address is zero, do not proceed to candidate acquisition.

The address comparison does not protect either preceding object access.

When the byte is nonzero and the address comparison succeeds, call **F-fb5a2b34c007** with no explicit arguments. Retain its result as the candidate address.

If the result is zero, return. The object’s `0x9c` store has already occurred; there is no direct manager access on this path.

### 3. Perform preparation and capture the manager address

For a nonzero candidate:

1. Call **F-2fd8826557a4** with no explicit arguments.
2. Read the global pointer slot and retain its value as the manager address.

The manager address is captured **after** F-2fd8826557a4 returns. The function therefore uses the slot value present after that call, not a value sampled before it.

Later direct manager accesses use this retained address. The function does not explicitly reload the global pointer slot during the remaining sequence.

### 4. Validate the candidate metadata word

At candidate offset `-8`:

1. Read the 32-bit metadata word and compare it with zero.
2. If it is nonzero, read that word again and test bit 0.
3. If the first read was zero, or bit 0 was set in the second read, invoke **F-a817d8d58c10** with no explicit arguments.

The supplied representation marks this dependency as nonreturning. There is no represented normal continuation from that call.

For stable metadata, validation succeeds exactly when the word is nonzero and even. The two-read structure matters if the storage changes between reads: a single immutable snapshot is not established.

This validation occurs after preparation and manager capture, but before the direct accounting writes.

### 5. Enter the manager operation and update four accounting locations

After validation succeeds, call **F-c0fc6d033caa**, passing the retained manager address.

After that dependency returns:

1. Read the unsigned 16-bit index at candidate offset `-4`.
2. Select the 32-bit manager location at `0x224 + 4 × first index`.
3. Read the metadata word at candidate offset `-8` and retain its value masked by `0x3ffffffc`.
4. Subtract that masked quantity from the selected location and write the 32-bit result.
5. Subtract the same retained quantity from manager offset `0x30c` and write the result.
6. Read the unsigned 16-bit index at candidate offset `-2`.
7. Subtract the same retained quantity from manager offset `0xe0 + 4 × second index` and write the result.
8. Subtract the same retained quantity from manager offset `0x220` and write the result.

Each subtraction is a read-modify-write of its destination. These four writes occur in the order listed.

The mask retains bits 2 through 29. Its result is a multiple of four in the range zero through `0x3ffffffc`. Bits 0, 1, 30, and 31 do not contribute to this quantity.

The quantity is sampled after F-c0fc6d033caa returns. Validation does not establish that this later sample is equal to either earlier metadata read. The function does not repeat the validation after that dependency call.

The second index is read after the first two accounting writes. It is not captured together with the first index.

### 6. Update indexed and aggregate counts

Next, call **F-09dcb55e8e0a** with no explicit arguments. Use its integer result as an index into manager storage:

1. Read the 32-bit location at manager offset `0x310 + 4 × returned index`, decrement it by one, and write it back.
2. Read the 32-bit location at manager offset `0x390`, decrement it by one, and write it back.

There is no local range check on the returned index. The evidence does not establish that it is unsigned or nonnegative.

F-09dcb55e8e0a is called after the four preceding accounting writes. Those changes have therefore already occurred if this dependency fails to return normally.

### 7. Update the final quantity and invoke final dependencies

After the two count decrements:

1. Read the metadata word at candidate offset `-8` again.
2. Mask this newly read word with `0x3ffffffc`.
3. Subtract that newly sampled quantity from the 32-bit location at manager offset `0x08` and write the result.
4. Call **F-cc6c9b4a0f68**, passing the retained manager address and the address of the metadata word at candidate offset `-8`.
5. Call **F-e586bff64fb7**, passing the retained manager address.
6. Return normally if both calls return normally.

The second argument to F-cc6c9b4a0f68 points to the metadata word, not to the candidate address itself.

The quantity used for the final subtraction is not necessarily the quantity used for the earlier four subtractions. In particular, F-09dcb55e8e0a runs between those samples.

## Side effects

### Direct writes

Every invocation that reaches its first instruction attempts the object store at `+0x9c`.

On the complete accounting path, seven additional 32-bit read-modify-writes occur:

1. Manager `+0x224 + 4 × first index`: subtract the first masked quantity.
2. Manager `+0x30c`: subtract that same quantity.
3. Manager `+0xe0 + 4 × second index`: subtract that same quantity.
4. Manager `+0x220`: subtract that same quantity.
5. Manager `+0x310 + 4 × dependency-selected index`: decrement by one.
6. Manager `+0x390`: decrement by one.
7. Manager `+0x08`: subtract the newly sampled masked quantity.

The function does not directly write candidate metadata or reset the object byte at `+0xa0`.

### Arithmetic and aliasing

The quantity-subtraction results are stored as 32-bit values. There is no saturation, underflow rejection, or rollback. Interpreted as unsigned 32-bit accounting, these subtraction results wrap modulo \(2^{32}\).

The two count decrements are represented using signed integer accesses. The evidence supplies no explicit behavior for signed overflow at the minimum representable value. Acceptance examples below avoid that boundary.

The 16-bit indices range from 0 through 65,535, giving scaled displacements from zero through `0x3fffc` before adding the respective base offset. The function does not establish that the manager owns storage covering that range.

No disjointness guarantee is supplied. Indexed locations can coincide with other manager locations, and external aliasing is not excluded. When locations overlap, the listed write order governs the direct effects; they must not be treated as independent simultaneous updates.

### Dependency effects and ownership

Dependency effects are additional to the direct effects above. The evidence does not establish that any dependency preserves globals, manager fields, candidate metadata, or other shared storage.

There is no explicit local allocation. Ownership of the candidate is unknown. Passing its metadata address to F-cc6c9b4a0f68 may be part of a release operation, but ownership transfer and reclamation are not established.

The function has no represented transaction, compensating write, or local recovery path.

## Dependencies

| Opaque ID | Explicit interface observed here | Established role in this function |
|---|---|---|
| **F-fb5a2b34c007** | No explicit arguments; address-like result | Supplies the candidate; zero stops processing |
| **F-2fd8826557a4** | No explicit arguments | Runs before the global manager slot is read |
| **F-a817d8d58c10** | No explicit arguments; marked nonreturning | Invoked for zero metadata or metadata with bit 0 set |
| **F-c0fc6d033caa** | Manager address | Runs before indices and accounting quantity are sampled |
| **F-09dcb55e8e0a** | No explicit arguments; integer result | Selects the count location at `0x310 + 4 × result` |
| **F-cc6c9b4a0f68** | Manager address; metadata-word address | Runs after all seven direct accounting writes |
| **F-e586bff64fb7** | Manager address | Final dependency on the normal accounting path |

Whether these calls lock, unlock, inspect execution context, release storage, or report a fatal error remains unknown.

## Acceptance examples

These are conditional static acceptance examples, not observed executions. Unless stated otherwise, assume valid memory, non-overlapping destinations, stable metadata, normally returning dependencies, and no dependency modifications to the illustrated fields.

### Nonzero supplied value

Given a supplied value of 7:

- Object offset `0x9c` becomes 7.
- The byte at `0xa0` is not directly read.
- No dependency is called.

### Zero supplied value with a disabled byte

Given a supplied value of zero and a zero byte at `0xa0`:

- Object offset `0x9c` becomes zero.
- No candidate is requested.
- No direct manager accounting occurs.

### Candidate acquisition returns zero

Given a zero supplied value, a nonzero enabling byte, a valid nonzero object address, and a zero result from F-fb5a2b34c007:

- The object store occurs.
- Candidate acquisition occurs.
- F-2fd8826557a4 is not called.
- The global manager slot is not directly read.

### Invalid metadata

Given a nonzero candidate and stable metadata equal to zero:

- Preparation and manager capture occur.
- F-a817d8d58c10 is invoked.
- None of the seven direct manager accounting writes occurs.

Stable metadata equal to 5 also invokes that dependency because bit 0 is set.

### Valid metadata with zero accounting quantity

Given stable metadata equal to 2:

- Validation succeeds.
- The masked quantity is zero.
- The five quantity-subtraction locations still undergo their specified read-modify-writes, although their values remain unchanged under these assumptions.
- Both count locations are decremented.
- Both final dependencies are called.

Thus, nonzero validated metadata does not imply a positive accounting quantity.

### Ordinary accounting

Given stable metadata `0x25`, the failure path is taken because bit 0 is set.

Given stable metadata `0x24` instead:

- Validation succeeds.
- The masked quantity is 36.
- Each of the five quantity locations loses 36.
- The two count locations each lose one.
- The final call order is F-cc6c9b4a0f68, then F-e586bff64fb7.

### Metadata changes during the sequence

Suppose the quantity sampled after F-c0fc6d033caa is 36, but F-09dcb55e8e0a changes the metadata so that its masked quantity becomes 68:

- The first four quantity subtractions use 36.
- The final subtraction at manager `+0x08` uses 68.
- No local consistency check rejects this difference.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the object fields and the global manager slot.
- The complete valid layout and extent of manager storage.
- Valid ranges for either metadata index and the dependency-selected index.
- The allocation or ownership relationship between candidate metadata and candidate storage.
- Whether preparation and final calls implement synchronization.
- The concrete failure mechanism used by F-a817d8d58c10.
- The effects of dependencies on globals, metadata, and manager storage.
- Endianness and external calling-convention details.
- Concrete target behavior for invalid addresses, address-calculation overflow, signed count overflow, or misaligned accesses.
- Behavior under concurrent unsynchronized access; no atomicity or memory-ordering guarantee is established.

The allocator-like interpretation is **inferred intent**, not a substitute for the direct behavioral contract.

## Evidence

The supplied static function representation establishes the field offsets, access widths, branch conditions, direct write order, dependency order, mask, and explicit dependency arguments described here.

Repeated metadata reads are treated as separate observations. Manager capture is placed after the preparation call. Unknown dependency behavior is not replaced with preservation, locking, or ownership assumptions.

No runtime tests have been performed. The acceptance examples describe consequences of the static evidence under their stated conditions; they are not execution results.
