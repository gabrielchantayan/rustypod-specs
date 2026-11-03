<!-- Trusted provenance: F-d8cb7bd4f85b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d8cb7bd4f85b

## Purpose

F-d8cb7bd4f85b conditionally copies four-byte values toward the beginning of descriptor-addressed storage, then updates a descriptor count. Under a stable, non-overlapping sequence layout, its effect is to discard a requested number of leading elements without releasing or resizing the backing allocation.

That sequence interpretation is inferred intent. The established behavior consists of descriptor reads, conditional forward transfers, and a count-field store. Backing storage is not required by the supplied body to be separate from the descriptor. Consequently, transfers can change metadata, and a count-field store can also change backing elements.

A null descriptor, an initially zero count, or a zero request causes no writes. A nonzero request at least as large as the initially observed nonzero count causes a zero count-field store without element-copy accesses.

## Interface

### Arguments

The function accepts three arguments:

| Argument | Representation | Established role |
|---|---|---|
| Descriptor reference | Nullable pointer to a record of 32-bit words | Supplies the address-bearing field and count field |
| Removal request | Unsigned 32-bit quantity | Selects the initial source index and amount subtracted from the final count |
| Supplied quantity | Unsigned 32-bit quantity | Has no effective influence on the established behavior |

For a nonnull descriptor, the stored count replaces the supplied quantity before any decision uses that quantity. For a null descriptor, the function performs no work regardless of it.

### Descriptor and storage layout

The accessed descriptor fields are:

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | Address-bearing word used to form transfer addresses |
| 4 | 32 bits | Not explicitly accessed |
| 8 | 32 bits | Count, interpreted as unsigned for decisions |

The complete descriptor size and any additional fields are unknown.

Each transferred value occupies four bytes. Values are copied without element-specific interpretation, conversion, destruction, or callbacks.

The address-bearing field is a 32-bit integer word, not an established native pointer field. Its suitability for addressing depends on the target execution model. Substituting a wider host pointer layout would not preserve the established interface.

### Result

There is no return value, status code, or returned element. Observable outcomes are memory accesses, resulting memory contents, and any execution failure imposed by the environment.

### Preconditions

For ordinary sequence-removal behavior:

- A nonnull descriptor must support reading the count field.
- A path that reaches the final count store must support writing that field.
- Copying requires every selected four-byte source and destination access to be valid.
- Address calculations and signed arithmetic must be valid in the target execution model.
- The descriptor and relevant storage must remain stable except for the function’s intended writes.
- Backing-storage writes must not interfere with descriptor fields.
- Any intended ownership semantics must permit copying raw four-byte representations and retaining old representations in the unused tail.

These conditions are not validated by the function. Nullness, initial count zero, and request zero are the only explicit guards before selecting a work path.

## Behavior

### Entry and no-write paths

For a null descriptor, the function returns without accessing descriptor fields or backing storage.

For a nonnull descriptor, the first established descriptor access is a read of the count at byte offset 8. This read occurs even when the request is zero.

If that count is zero, or the request is zero, the function returns without writing memory. Neither condition requires reading the address-bearing field. A zero request does not trigger self-copying or rewrite an unchanged count.

The supplied third argument cannot enable work for a null descriptor or override a zero stored count.

### Request at least equal to the initial count

When the initially observed count and request are both nonzero, the request is compared with the count as an unsigned 32-bit quantity.

If the request is greater than or equal to that count:

1. No element-copy source load or destination store is performed.
2. The address-bearing field is not needed or explicitly read.
3. The count field is written as zero.
4. The function returns.

This branch does **not** guarantee that all backing bytes remain unchanged. If backing storage overlaps the count field, the count-field store also modifies those overlapping backing bytes. For example, if backing element zero occupies the count word, writing the count to zero also writes that element to zero.

Backing bytes outside the count-field store remain unchanged by this branch. There is no separate clearing pass, element destruction, or resource release.

An unusable value in the address-bearing field does not itself require a backing-storage access on this path. The count-field access must still be valid.

### Request smaller than the initial count

For a nonzero request unsigned-less-than the initially observed nonzero count, copying begins with:

- A source index equal to the request.
- A destination index equal to zero.

Before each attempted transfer, the function reads the descriptor’s current count and compares the source index with that value using unsigned interpretation. If the source index is not smaller, copying ends.

For each successful continuation check, one four-byte value is loaded from the source-index address and stored at the destination-index address. The destination index then advances by one, followed by the source index advancing by one before the next continuation check.

With stable metadata and valid arithmetic, transfers therefore proceed in this order: source element at the request to destination zero, the next source element to destination one, and so on.

The source remains ahead of the destination by the positive request. Under an ordinary non-overlapping descriptor layout, forward copying does not overwrite values needed by later transfers.

### Repeated metadata observations

The initial count selects the branch; it is not a fixed bound retained throughout copying.

Every continuation decision reads the current count, including the decision that terminates the loop. If a transfer changes the count through aliasing, subsequent decisions use the changed value.

The address-bearing word is referenced in both source-address and destination-address expressions. The supplied body does not establish one immutable backing address for the entire operation. A changed address field can therefore affect later transfers.

The source value must be available before the corresponding destination store completes. However, the source representation does not establish a complete ordering between the evaluations that form the source and destination addresses. This contract does not promise a particular machine-level load count, volatile-access trace, or synchronization behavior.

### Final count update

After copying stops, the function reads the count field again, subtracts the request, converts the result to the signed 32-bit word representation used for the assignment, and writes the count field.

For stable metadata and ordinary representable values, the final count is the initial count minus the request.

If a transfer has changed the count, the subtraction uses that changed value, not the original count. The final store can itself modify backing storage where that storage overlaps the count field.

There is no clamping operation on the copying path. If the later count, interpreted as unsigned, is smaller than the request, the subtraction wraps modulo \(2^{32}\) before conversion for storage. The exact signed conversion and resulting stored representation for an out-of-range result depend on the target model.

### Width and arithmetic boundaries

The request, source index, and count comparisons use unsigned 32-bit quantities. A count word with its high bit set is therefore not treated as a negative count by the branch or continuation comparisons.

The source index advances with unsigned arithmetic. Its multiplication by four and addition to the address-bearing word follow the unsigned arithmetic of the represented 32-bit model. Wraparound arithmetic does not establish that the resulting address is accessible or meaningful.

The destination index and its multiplication by four use signed 32-bit arithmetic in the supplied representation. Its address calculation also includes signed arithmetic. Thus, avoiding destination-index increment overflow is insufficient: scaling or adding the address can become invalid earlier.

For example, a nonnegative destination index above \(536{,}870{,}911\) cannot be multiplied by four within a signed 32-bit range. Even smaller indices can encounter address-addition limits.

The function therefore does not establish safe support for every unsigned 32-bit count. Validity requires both accessible memory and valid arithmetic; a mathematically nonnegative final count does not establish either.

## Side effects

### Descriptor and backing-storage writes

The count field is the only descriptor field explicitly assigned as metadata.

It is explicitly written only when all three conditions hold:

- The descriptor is nonnull.
- The initially read count is nonzero.
- The request is nonzero.

On the copying path, element stores can overwrite any aliased descriptor bytes, including the address-bearing word, the otherwise unaccessed word at offset 4, or the count. “Not explicitly assigned” does not mean protected from aliased stores.

Conversely, on either working path, the final count-field store can alter backing elements that overlap it. This remains true when no element-copy accesses occur.

### Ordinary non-overlapping result

For stable metadata, valid arithmetic, initial count \(N\), and request \(K\) with \(0<K<N\):

- Exactly \(N-K\) four-byte transfers occur.
- Destinations are elements zero through \(N-K-1\), in increasing order.
- The final count is \(N-K\).
- The live prefix contains the original elements \(K\) through \(N-1\).

There is no separate operation to erase the unused tail. Under these non-overlap assumptions, tail elements retain their previous bytes.

### Ownership and lifetime

No allocation, deallocation, capacity adjustment, destructor invocation, reference-count operation, or external call appears in the supplied body.

The function does not establish ownership of the descriptor or storage. Copying a four-byte representation does not establish transfer of ownership of any resource it might identify. Removed representations are not explicitly released, and duplicated representations may remain beyond the new logical count.

### Errors and partial effects

There is no explicit error reporting, rollback, or recovery.

Invalid memory, unsuitable alignment, invalid signed arithmetic, or other execution-model violations have no function-defined recovery behavior. If execution fails after some stores, earlier effects may remain while the final count update has not occurred.

Concurrent mutation is not synchronized here. Repeated reads must not be interpreted as a guarantee that unsynchronized concurrent access is supported.

## Dependencies

No calls to other functions are present. There are no established callee dependencies to identify with additional opaque function IDs.

Behavior depends on the execution environment for:

- Interpretation of the 32-bit address-bearing word.
- Four-byte access validity and alignment.
- Signed arithmetic and out-of-range unsigned-to-signed conversion.
- Memory-access failures and their consequences.
- Compiler and memory-model treatment of aliasing and concurrent access.

No global object is explicitly referenced. Descriptor-addressed memory may nevertheless be shared or global. The absence of external calls does not imply isolation from such storage.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not observed executions. Unless an example explicitly states overlap, assume valid arithmetic and addressing, stable metadata, and descriptor storage disjoint from backing storage.

### Proper-prefix removal

Given count five, elements `[11, 22, 33, 44, 55]`, and request two:

- Transfers occur from source 2 to destination 0, source 3 to destination 1, then source 4 to destination 2.
- The count becomes three.
- The live prefix becomes `[33, 44, 55]`.
- Full backing storage becomes `[33, 44, 55, 44, 55]`.

Changing only the third argument must not change this outcome.

### Equal or larger request without overlap

Given count three and request three or four:

- The count becomes zero.
- There are no element-copy accesses.
- Backing bytes remain unchanged because this example excludes overlap with the count-field store.

An unsigned request of \(2^{32}-1\) follows the same branch for count three. It is not treated as a negative request.

### Equal request with count-backed storage

Assume the address-bearing word designates the descriptor count word as backing element zero. Let the initial count and request both be three, and assume the descriptor count is readable and writable.

Then:

- No element-copy accesses occur.
- The address-bearing word need not be read.
- The zero count-field store also changes backing element zero from three to zero.
- Backing bytes not overlapping the count-field store remain unchanged.

This outcome is incompatible with an unconditional promise that the no-copy branch preserves every backing element.

### Zero request

Given a nonnull descriptor with a readable nonzero count and request zero:

- The count is read.
- No memory is written.
- The address-bearing field is not needed.

The descriptor count need not be writable for this path.

### Zero count

Given a nonnull descriptor with a readable zero count and any request:

- No count store or element-copy access occurs.
- A nonzero third argument does not replace the zero count.

### Null descriptor

Given a null descriptor and arbitrary remaining arguments, no descriptor or backing-storage access occurs and no memory is written.

### Copy changes the continuation bound

Assume backing element zero is the descriptor count word, initially three. The next four-byte backing element contains one. Let the request be one, with all involved accesses and arithmetic valid.

Then:

1. The first continuation check compares source index one with count three.
2. The transfer loads one from backing element one and stores it into backing element zero, changing the count to one.
3. The next continuation check compares source index two with count one and stops copying.
4. The final subtraction reads one and subtracts one.
5. The final count is zero.

An implementation that instead uses the original count for every decision would not satisfy this aliasing example.

## Uncertainties

### Established static behavior

The supplied body establishes the guards, unsigned comparisons, four-byte transfers, forward index progression, repeated count observations, final count assignment, and absence of external calls.

It also establishes that the no-copy branch still writes the count field. Whether that store changes backing storage depends on overlap, not on whether element copying occurred.

### Inferred intent

The likely purpose is prefix removal from a compact sequence while retaining its allocation. The layout and transfer direction support that interpretation but do not establish a public container contract or resource-ownership policy.

### documented_only

The following remain unresolved:

- Original public types and calling convention.
- Full descriptor size and unused-field meanings.
- Exact target address interpretation and alignment requirements.
- Permitted aliasing under any external caller contract.
- Valid maximum counts and storage sizes.
- Target-specific out-of-range conversion behavior.
- Whether concurrency is prohibited or externally synchronized.
- Ownership rules for four-byte element representations.
- Exact generated access ordering beyond the stated control and value dependencies.

Aliasing examples describe the supplied body’s accesses conditionally; they do not establish that every source-language or target environment permits those layouts.

No runtime tests have been performed.

## Evidence

This contract is based solely on the supplied static body for F-d8cb7bd4f85b. Prior prose and reviewer feedback are not independent evidence.

The body supports:

- A count read before the zero-request decision for nonnull descriptors.
- Behavioral irrelevance of the third argument.
- No writes for null descriptors, initial zero counts, and zero requests.
- Unsigned request/count decisions.
- Forward four-byte transfers with fresh continuation-count reads.
- A separate count read for subtraction after copying.
- A zero count-field store, without element-copy accesses, for an equal or larger request.
- Possible backing-storage modification by that store when storage overlaps the count field.
- No explicit allocation, release, element cleanup, or external dependency calls.

The acceptance examples are analytical consequences under their stated preconditions, not runtime verification.
