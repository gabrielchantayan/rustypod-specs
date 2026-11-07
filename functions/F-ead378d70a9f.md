<!-- Trusted provenance: F-ead378d70a9f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ead378d70a9f

## Purpose

F-ead378d70a9f inserts a four-byte value into storage described by a three-word mutable header. Its two paths are:

- **Existing storage:** extend the logical end, shift the occupied suffix one word toward higher addresses, and overwrite the insertion position.
- **Replacement storage:** choose a larger capacity, request storage, invoke a transfer dependency for the prefix and suffix, write the inserted word between them, release the previous storage through another dependency, and replace the header.

This interpretation is supported by the address arithmetic and writes. It is not evidence that the routine implements a particular language-level container, allocator contract, or element ownership model.

The direct element operations copy four-byte representations. The replacement path additionally depends on an unidentified transfer routine whose behavior is not supplied. Consequently, full sequence-preservation semantics for that path remain conditional.

## Interface

### Inputs

The routine accepts three address-valued arguments:

| Argument | Required representation | Role |
|---|---|---|
| Header | Address of at least three consecutive four-byte fields | Mutable storage descriptor |
| Position | Address used as an insertion location or transfer boundary | Determines the insertion point |
| Value source | Address from which one four-byte value can be read | Supplies the inserted representation |

The header fields, in address order, are interpreted as:

1. **Begin:** address of the first element.
2. **End:** address immediately after the occupied elements.
3. **Storage limit:** address immediately after the available storage.

The supplied representation uses 32-bit words and 32-bit address arithmetic. An element occupies four bytes. No element tag, alignment metadata, destructor, or ownership flag is consulted directly.

### Output

There is no explicit return value. Observable results are conveyed through:

- Header writes.
- Element-storage writes.
- Calls to dependencies.
- Any effects those dependencies produce.

### Preconditions for ordinary insertion behavior

The following are necessary conditions for the conventional insertion interpretation, not validations performed by this routine:

- The header is readable and writable.
- Occupied storage forms a coherent, four-byte-stepped range.
- Begin, end, storage limit, and position have compatible alignment.
- The value source remains readable when the actual value load occurs.
- Address and capacity calculations do not overflow in ways that invalidate the ranges.
- Dependency behavior is compatible with the intended storage operation.

The existing-storage path has an additional important restriction: position must be an occupied element address, from begin through end minus four. It does not directly support insertion at end or insertion into an empty range.

The replacement path’s accepted position range depends in part on the transfer dependency. There is no explicit position validation in this routine.

## Behavior

### 1. Select a path

The routine first reads and retains the header’s begin and storage-limit fields. It then calls F-0c14ee61f55d with the header.

The capacity-like quantity used for the branch is obtained from the retained address difference:

- Subtract begin from storage limit.
- Apply a signed right shift by two.
- Interpret the result as an unsigned 32-bit quantity.

The replacement-storage path is selected when that quantity is less than or equal to the unsigned result of F-0c14ee61f55d. Otherwise, the existing-storage path is selected.

Thus:

- Equality selects replacement storage.
- A helper result larger than the capacity-like quantity also selects replacement storage.
- A helper result smaller than that quantity selects existing storage.

The compared address fields were captured before the helper call. A helper that changes the header does not change those retained operands. Later operations, however, read the header again.

Calling the helper result an “element count” is inferred intent. Its actual behavior is not established by the supplied material.

### 2. Replacement-storage path

#### Capacity selection

The routine calls F-0c14ee61f55d again. This second result, not the first result, determines the requested capacity.

For a second result \(n\), the two candidates are:

\[
n + 32
\]

and

\[
n + \lfloor n/2 \rfloor + \lfloor n/8 \rfloor.
\]

The shifts used to obtain the fractional terms are unsigned. Additions use the 32-bit unsigned representation; overflow can therefore change both candidate values and their unsigned ordering.

The larger unsigned candidate becomes the selected capacity. The routine requests storage through F-871e64ddc6de with:

- First argument: selected capacity shifted left by two, producing a 32-bit byte-size argument.
- Second argument: zero.

There is no explicit overflow rejection and no allocation-failure branch.

#### Prefix transfer

After the allocation call returns, the routine invokes F-unmapped-01 with four arguments:

1. The header’s then-current begin.
2. Position.
3. The returned allocation address.
4. The header address.

F-unmapped-01 is a local opaque designation for the transfer dependency whose published function ID was not provided.

The routine does not establish whether this dependency copies, moves, initializes, destroys, validates, or changes header fields.

#### Inserted-word write

After the prefix call, the routine reads begin again. It subtracts that current begin from position and clears the low two bits of the 32-bit result. Adding this rounded offset to the allocation address produces the inserted-word destination.

If that computed destination is nonzero, the routine reads four bytes from the value source and writes those bytes to the destination.

If the computed destination is zero, this particular read and write are skipped. This is not a general allocation-failure check: a zero allocation address with a nonzero offset can still produce a nonzero destination.

The source value is not captured before allocation or prefix transfer. Effects of those earlier calls can therefore influence the value eventually loaded.

#### Suffix transfer

The routine invokes F-unmapped-01 a second time with:

1. Position.
2. The header’s then-current end.
3. A destination obtained from the allocation address plus the position-relative rounded offset plus four.
4. The header address.

The position-relative offset is recomputed using a fresh header-begin read. It is not necessarily the offset used for the preceding inserted-word write.

Accordingly, if the prefix dependency or inserted-word write changes the header, the second transfer can use different boundaries or a different relative destination.

Reads needed to construct arguments to one dependency call are not given a stronger relative order here than the supplied representation establishes. The two calls themselves, and the intervening inserted-word operation, have the sequence described above.

#### Previous-storage release and header replacement

After the suffix transfer, the routine:

1. Calls F-0c14ee61f55d a third time and retains its result.
2. Reads the current begin and retains it as the previous-storage address.
3. Traverses addresses from that retained begin toward the header’s end in four-byte increments.
4. Calls F-ff1bdc376dcd.
5. Writes the replacement header fields.

The traversal has no explicit element read, element write, or destructor call. Its termination condition consults the header’s end. A malformed range that cannot be reached by four-byte increments can prevent progress to release and header replacement.

F-ff1bdc376dcd receives:

- The retained previous-storage address.
- The signed right-shift-by-two result of the current storage limit minus that retained address.
- Zero.

The second argument is an element-count-like quantity, not the byte-size argument passed to the allocation dependency. The routine does not establish why these dependency interfaces use different units.

After the release call returns, the routine writes the header in this order:

1. Begin becomes the returned allocation address.
2. End becomes that address plus four times the third helper result, plus four.
3. Storage limit becomes that address plus four times the selected capacity.

The new end therefore depends on the third helper result. It is not derived from either earlier helper result.

### 3. Existing-storage path

This path performs no allocation, transfer, or release calls after the initial F-0c14ee61f55d call.

The routine reads the header’s current end and immediately advances the header’s end by four bytes. This header write occurs before any element copy.

If the previous end address is nonzero, the routine copies the four-byte word immediately before that address into the word at that address. There is no separate check that the occupied range was nonempty.

It then walks backward through the occupied suffix, copying words toward higher addresses until its backward source cursor reaches position. For a valid occupied position:

- The old last word is copied into the newly occupied final slot first.
- Remaining affected words are copied in descending address order.
- The original word at position is copied to position plus four.
- The source value is read and written to position last.

The descending direction permits the ordinary overlapping right shift without overwriting a word before it is needed.

The value source is read only after the shift. If it aliases an overwritten location, the inserted representation can differ from the source’s entry value.

Position equal to the previous end is not handled as an append operation. The backward cursor begins one word below that end and moves downward, away from it. Empty storage likewise causes an attempted read before begin when the end address is nonzero.

## Side effects

### Direct memory effects

On the existing-storage path:

- End is written before element movement.
- The newly occupied final slot is written first, subject to the nonzero-address condition.
- Suffix writes proceed toward lower addresses.
- Position is written last.
- Begin and storage limit are not directly changed.

On the replacement-storage path:

- The inserted-word destination is directly written between the two transfer calls, unless that destination is zero.
- The previous-storage release dependency runs before the final header replacement.
- Begin, end, and storage limit are written in that order.

Aliasing can make these effects interact. For example, a value source that points to the header’s end field observes the already-advanced end on the existing-storage path. Aliasing header fields with element destinations can also invalidate the ordinary container interpretation.

### Ownership and lifetime

The replacement path passes previous storage to F-ff1bdc376dcd and installs the returned allocation address in the header. This supports an inferred ownership-transfer interpretation, but does not establish:

- That allocation succeeded.
- That the release dependency actually frees memory.
- That transfer calls preserve or relinquish ownership.
- That elements have trivial lifetime semantics.
- That the caller retains valid aliases into previous storage.

The routine does not directly release the value source or position.

### Errors and interruption

There is no explicit status result, validation error, rollback operation, or recovery branch.

If a dependency fails to return normally, execution does not reach later operations in the sequence. The resulting state depends on effects already performed. In particular:

- Existing-storage end advances before the shift completes.
- Replacement storage can be partly populated before release.
- Release occurs while the header still contains its prior fields.
- The three final header writes are not an atomic update.

No concurrency, reentrancy, or failure-atomicity guarantee is established.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-0c14ee61f55d | Receives the header; called once on the existing-storage path and three times on the replacement path; results govern branch selection, capacity growth, and final end respectively | Whether it computes size, changes the header or globals, or returns consistent results |
| F-871e64ddc6de | Receives a 32-bit byte-size expression and zero; its result is used as a storage base | Allocation policy, alignment, failure signaling, ownership, or global effects |
| F-ff1bdc376dcd | Receives a previous-storage address, an element-count-like expression, and zero before final header replacement | Whether it frees storage, requires a specific size unit, or changes other state |
| F-unmapped-01 | Called twice with source boundaries, a destination, and the header | Transfer semantics, overlap rules, element lifetime effects, header mutation, and failure behavior |

F-unmapped-01 is only a document-local opaque identifier. It is not a claimed published mapping.

No dependency is assumed to preserve globals, preserve the header, or manage memory in a particular way.

## Acceptance examples

These are conditional acceptance examples derived from the static operation order. They are not runtime test results.

### Existing storage, middle insertion

Assume occupied words are `[10, 20, 30]`, storage has room for another word, the first helper result selects existing storage, position addresses `20`, and a separate stable source contains `99`.

Expected occupied words are `[10, 99, 20, 30]`. End advances by four; begin and storage limit remain unchanged.

### Existing storage, last occupied position

Under the same assumptions, let position address `30`.

The initial copy places `30` in the next slot. No additional backward-copy iteration is needed. The final write produces `[10, 20, 99, 30]`.

This differs from insertion at the previous end, which is not supported by this path’s backward traversal.

### Value source aliases a shifted slot

Assume occupied words are `[10, 20, 30]`, position addresses `10`, and the value source addresses the original `20` slot.

The shift overwrites that source slot with `10` before the final value load. The resulting sequence is `[10, 10, 20, 30]`, not `[20, 10, 20, 30]`.

This demonstrates that the contract does not promise entry-time capture of the source value.

### Replacement capacity without overflow

Assume all helper calls preserve the header, report a count of eight, and the initial capacity-like quantity is eight.

Equality selects replacement storage. Growth candidates are 40 and 13, so the selected capacity is 40 words and the allocation request is 160 bytes.

If the transfer dependency performs ordinary range copying, allocation succeeds, and release returns normally, the final header describes nine occupied words in storage for 40 words.

### Different helper results

Assume the first helper result selects replacement, the second result is 64, and the third result is 7.

The capacity candidates are 96 and 104, so the requested size is 416 bytes. The final end is allocation base plus 32 bytes, because it uses the third result plus one word.

The routine does not reconcile that final length with the ranges actually transferred.

### Zero computed insertion destination

If the replacement path computes an inserted-word destination of zero, it skips that source load and destination write. It still proceeds to the suffix transfer and subsequent operations if they return normally.

This behavior does not constitute successful empty-container insertion or successful allocation-failure handling.

## Uncertainties

### Established static evidence

The supplied material establishes the direct four-byte reads and writes, branch comparison, growth expressions, dependency-call sequence, header-write sequence, and absence of explicit validation or rollback.

### Inferred intent

The three-field header resembles a contiguous-storage descriptor. F-0c14ee61f55d appears size-like, F-871e64ddc6de allocation-like, F-ff1bdc376dcd release-like, and F-unmapped-01 transfer-like.

These interpretations explain the normal insertion shape but are not independent dependency contracts.

### documented_only

The following remain unknown:

- The transfer dependency’s actual behavior and published opaque ID.
- Dependency effects on globals, header fields, element storage, and ownership.
- The actual runtime’s treatment of signed subtraction or multiplication overflow, invalid pointer arithmetic, and invalid accesses.
- Whether signed right shifts in the supplied representation exactly reflect target execution for negative values.
- Alignment and allocation guarantees.
- Dependency failure mechanisms and partial effects.
- Supported caller invariants, including whether callers exclude append and empty-range cases on the existing-storage path.
- Whether unusual aliasing is permitted by the wider program.

Unsigned growth and shift results must not be silently replaced with unbounded mathematical arithmetic. Signed address expressions likewise require valid-range assumptions before ordinary arithmetic reasoning is safe.

## Evidence

This contract is based solely on the supplied static function representation and dependency mapping. No implementation access, dependency bodies, runtime observations, or tests were available.

No runtime tests have been performed. Acceptance examples describe conditional consequences of the supplied operations, not observed executions.
