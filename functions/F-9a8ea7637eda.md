<!-- Trusted provenance: F-9a8ea7637eda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9a8ea7637eda

## Purpose

F-9a8ea7637eda changes the logical extent recorded in a three-word descriptor. Depending on the requested extent and the descriptor’s current capacity, it may first obtain or replace backing storage. It then passes the affected address range to an external operation and commits the requested extent.

**Established behavior:** the function performs signed comparisons, conditionally calls storage-related dependencies, invokes a range operation on every normally completed path, and updates the descriptor’s first word after that operation returns.

**Inferred intent:** the descriptor resembles a resizable byte-buffer descriptor containing a logical extent, backing address, and capacity. The final external operation may initialize or otherwise process the range added or removed by the extent change. Its actual effect is not established; in particular, clearing, copying, and destruction must not be assumed.

## Interface

### Inputs

The function accepts four arguments:

1. **Descriptor reference.** A reference used to access three consecutive 32-bit words:
   
   | Byte offset | Role evidenced by use |
   |---|---|
   | 0 | Current logical extent, interpreted as a signed 32-bit integer for comparisons and differences |
   | 4 | Backing address represented by a 32-bit word |
   | 8 | Capacity or allocation-size value, interpreted as a signed 32-bit integer for comparison and passed unchanged to a replacement dependency |

2. **Requested extent.** A signed 32-bit integer used as the requested new logical extent.

3. **Auxiliary word A.** A 32-bit value forwarded only to F-870c4fcf7295 when additional capacity is required.

4. **Auxiliary word B.** A 32-bit value forwarded twice to F-870c4fcf7295 when additional capacity is required.

The offset description follows the supplied representation of consecutive 32-bit words. A byte-oriented buffer interpretation is inferred from the arithmetic; no element type is established.

### Output

The return value is a 32-bit integer:

- On normal completion, it equals the requested extent.
- If a storage-related dependency returns zero and the error-reporting dependency returns normally, it is zero.

A zero return is therefore not an unambiguous failure indication: a successful request for extent zero also returns zero.

### Preconditions and validity limits

The function performs no explicit descriptor-null check, bounds validation, sign validation, or consistency validation.

For ordinary defined use, the descriptor must support the reads and writes required by the selected path. The backing address and affected range must meet the external range dependency’s requirements. Those requirements are unavailable.

The supplied representation uses 32-bit integer arithmetic for address construction and size calculation. Inputs that overflow signed addition, subtraction, or left shift do not have a fully established arithmetic contract here. Neither checked arithmetic nor portable wraparound is evidenced.

## Behavior

### 1. Initial observations

The function first reads:

1. The descriptor’s current logical extent.
2. The descriptor’s backing-address word.

Both observations occur before selecting between growth and non-growth behavior. Capacity is not read unless the initially observed extent is less than the requested extent.

The comparison between the current and requested extents is signed.

### 2. Growth within the observed capacity

If the initially observed extent is less than the requested extent, the function next reads the capacity word.

If that capacity is greater than or equal to the requested extent under signed comparison:

- No capacity-calculation dependency is called.
- No storage-related dependency is called.
- The affected address is the initially observed backing address plus the initially observed extent.
- The affected length is the requested extent minus the initially observed extent.

The function then performs the common range operation and final extent commit described below.

This path does not reread the descriptor’s extent or backing-address word before constructing the affected range.

### 3. Growth requiring additional capacity

If both the initially observed extent and the observed capacity are less than the requested extent, the function performs the following sequence.

#### Capacity calculation

It calls F-870c4fcf7295 with five ordered arguments:

1. Requested extent plus 3.
2. The integer constant 3.
3. Auxiliary word A.
4. Auxiliary word B.
5. Auxiliary word B again.

It then shifts the dependency’s returned 32-bit value left by two positions. The resulting value is used as the new storage-size argument and, on success, is stored as the descriptor’s capacity.

The dependency’s transformation is unknown. The visible expression does not establish a rounding rule, minimum capacity, growth factor, or guarantee that the resulting capacity covers the requested extent.

Arithmetic limitations include:

- Requested extent plus 3 must be representable for a straightforward signed interpretation.
- The left shift must have a valid and representable result for a straightforward signed interpretation.
- No overflow or invalid-result check is visible.
- The shifted result is not compared against the requested extent before use.

#### Storage request

The storage dependency is selected using the backing-address word read at entry, not a fresh read after F-870c4fcf7295:

- If the observed backing address was zero, F-43da6b23bd44 is called with the computed size, zero, and zero.
- Otherwise, F-06ed623de53f is called with the observed backing address, the observed old capacity, the computed size, zero, and zero.

A zero result from the selected dependency is treated as failure. Any nonzero result proceeds as the new backing-address word; there is no further visible validation.

#### Failure

On a zero storage result, the function calls F-4fc89ec41a6d with these five ordered values:

1. 7
2. 100
3. 65
4. 0
5. 0

If that dependency returns normally, F-9a8ea7637eda returns zero immediately.

On this path, the function itself does not:

- Store a replacement backing address.
- Store the computed capacity.
- Invoke the common range operation.
- Commit the requested extent.

This is not a guarantee that the descriptor, backing storage, or global state remains unchanged. Any invoked dependency may have side effects, and dependency behavior is unavailable.

#### Success

On a nonzero storage result, the function performs these descriptor operations in order:

1. Stores the returned backing-address word at offset 4.
2. Stores the computed capacity at offset 8.
3. Rereads the logical extent at offset 0.

The affected address is then the returned backing address plus that reread extent. The affected length is the requested extent minus that reread extent.

The reread is significant. If a dependency changed the descriptor’s extent during capacity calculation or storage acquisition, the successful growth path uses the changed extent to determine the final range. It does not recheck whether that extent is still below the requested extent.

Conversely, the function does not reread the backing-address word for range construction. It uses the storage dependency’s returned value.

### 4. Equal or smaller requested extent

If the initially observed extent is greater than or equal to the requested extent:

- Capacity is not read.
- No capacity calculation or storage request occurs.
- The affected address is the initially observed backing address plus the requested extent.
- The affected length is the initially observed extent minus the requested extent.

This includes equality. When the extents are equal, the affected length is zero, but the range dependency is still called.

There is no visible capacity reduction or backing-storage release on this path.

### 5. Common range operation and commit

Except after storage failure, the function calls an unmapped external dependency, designated **U-1** in this contract, with:

1. The affected address.
2. The affected length.

Any return value is ignored.

Only after U-1 returns does the function store the requested extent at descriptor offset 0. It then returns that requested extent.

Consequences of this order:

- The requested extent is not committed before U-1 executes.
- On successful capacity growth, the new backing address and capacity have already been committed when U-1 executes.
- If U-1 changes the descriptor’s extent and returns normally, the final extent store overwrites that change.
- If U-1 does not return normally, the final extent store and ordinary return are not reached.

## Side effects

### Direct descriptor writes

The function’s own writes are path-dependent:

| Path | Writes before U-1 | Writes after U-1 |
|---|---|---|
| Growth within capacity | None | Requested extent |
| Growth with successful storage request | Returned backing address, then computed capacity | Requested extent |
| Equal or smaller extent | None | Requested extent |
| Storage failure | None | None |

These are logical access orders established by the supplied representation, not a synchronization, atomicity, or inter-thread memory-ordering guarantee.

### Backing memory and ownership

The storage-related call shapes suggest allocation and replacement operations, but ownership semantics are not established.

In particular:

- No explicit release operation appears in this function.
- Whether F-06ed623de53f releases, retains, or modifies the old allocation is unknown.
- Whether a failed storage request preserves the old allocation is unknown.
- Whether a successful replacement preserves existing contents is unknown.
- U-1’s reads, writes, and ownership effects are unknown.
- The constants passed to dependencies must not be interpreted as ownership or initialization flags without their contracts.

The function does not provide rollback after successful backing-address and capacity writes. No separate failure result from U-1 is inspected.

### Global and indirect effects

No direct global access is visible. This does not establish that the function is globally side-effect-free: every dependency may access globals or indirectly mutate the descriptor or backing memory.

No dependency is assumed to preserve global state, own memory, release memory, or return normally.

## Dependencies

| Opaque identifier | Established use | Unavailable contract |
|---|---|---|
| F-870c4fcf7295 | Produces a value that is shifted left by two to form a storage size | Calculation semantics, valid input domain, side effects, error conventions |
| F-43da6b23bd44 | Called when the initially observed backing address is zero; zero result selects failure | Allocation behavior, initialization, ownership, side effects |
| F-06ed623de53f | Called when the initially observed backing address is nonzero; receives old address, old capacity, and computed size | Replacement behavior, content preservation, release behavior, failure guarantees |
| F-4fc89ec41a6d | Receives the fixed five-value tuple after a zero storage result | Diagnostic meaning, global effects, whether it returns or terminates |
| U-1 | Receives the computed affected address and length on every normally completed path | Operation performed, valid ranges, zero-length behavior, failure and ownership semantics |

U-1 is a document-local opaque label. No published function ID for that external dependency was supplied.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume ordinary representable arithmetic, valid dependency inputs, and normal dependency returns unless otherwise stated.

### Growth within capacity

Given an initial extent of 8, backing address of 4096, capacity of 32, and requested extent of 12:

- The capacity word is read.
- No storage-related dependency is called.
- U-1 receives address 4104 and length 4.
- After U-1 returns, the extent becomes 12.
- The return value is 12.
- The function itself leaves the backing address and capacity unchanged.

This does not specify what U-1 does to those four addressed units.

### Reduction

Given extent 12, backing address 4096, and requested extent 5:

- Capacity is not read.
- U-1 receives address 4101 and length 7.
- The extent becomes 5 after U-1 returns.
- The return value is 5.
- No explicit backing-storage release occurs.

### Equal extents

Given extent 12, backing address 4096, and requested extent 12:

- Capacity is not read.
- U-1 is called with address 4108 and length zero.
- The extent is stored as 12 after that call.
- The return value is 12.

Skipping U-1 solely because the length is zero would not match the visible behavior.

### Successful growth from a zero backing address

Given extent 0, backing address zero, capacity 0, and requested extent 5:

- F-870c4fcf7295 receives 8, 3, auxiliary word A, auxiliary word B, and auxiliary word B.
- If it returns 4, the computed storage size is 16.
- F-43da6b23bd44 receives 16, zero, and zero.
- If it returns address 8192, the function stores backing address 8192 and capacity 16, then rereads the extent.
- If the reread extent remains zero, U-1 receives 8192 and 5.
- After U-1 returns, the extent becomes 5 and the return value is 5.

The assumed dependency result of 4 is illustrative, not a claimed rounding rule.

### Extent changed by a dependency

Given initial extent 8, capacity 8, nonzero backing address, and requested extent 12, suppose the capacity calculation yields a computed size of 16. Suppose the replacement dependency returns address 8192 and, before returning, changes the descriptor extent to 10.

Then:

- The returned address and computed capacity are stored.
- The extent reread observes 10.
- U-1 receives address 8202 and length 2.
- The final extent becomes 12.

Using the original extent of 8 for this range would not match the successful-storage path.

### Storage failure

If either selected storage dependency returns zero:

- F-4fc89ec41a6d receives 7, 100, 65, zero, and zero.
- If it returns, the function returns zero.
- U-1 is not called.
- No descriptor write is performed directly by F-9a8ea7637eda on that path.

Descriptor preservation remains conditional on dependency behavior.

### Negative requested extent

Given extent 4, backing address 4096, and requested extent −2:

- Signed comparison selects the non-growth path.
- The computed address is 4094.
- The computed length is 6.
- If U-1 accepts that range and returns, the function stores and returns −2.

This establishes the absence of local negative-input rejection, not the validity or safety of the resulting memory range.

## Uncertainties

The following remain **documented_only**:

- The semantic operation performed by U-1.
- The exact capacity-calculation rule.
- The valid descriptor invariants, including nonnegative extents and capacity relationships.
- The target’s overflow and address-wrap semantics.
- Storage ownership, content preservation, and release behavior.
- Dependency effects on globals, the descriptor, and backing memory.
- The diagnostic meaning of the fixed failure tuple.
- Whether any dependency can terminate execution or otherwise fail to return.
- Whether descriptor storage may alias backing storage, and the consequences of such aliasing.
- Concurrency, reentrancy, and synchronization requirements.

The buffer-resizing interpretation is inferred intent. It must not strengthen the established contract into claims of zero initialization, safe shrinking, transactional failure, or automatic memory reclamation.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping.

That evidence establishes the branch conditions, represented integer widths, call argument order, zero-result failure selection, descriptor access order, and final return behavior described above. Dependency bodies and external runtime contracts were not supplied.

No runtime tests have been performed. The acceptance examples are conditional consequences of the visible control flow and arithmetic, not observed executions.
