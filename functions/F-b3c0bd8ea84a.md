<!-- Trusted provenance: F-b3c0bd8ea84a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b3c0bd8ea84a

## Purpose

F-b3c0bd8ea84a searches a word-organized bit region, skips set bits, and then traverses consecutive clear bits while setting those bits in a working word. It attempts to write processed words back to acquired storage and reports a candidate starting position and a count.

**Established static behavior:** the function uses 32-bit positions and words, traverses storage through opaque dependencies, performs ordered output and storage writes, and returns a signed 32-bit status. Explicit failure paths store zero into both outputs. Earlier storage writes are not rolled back.

**Inferred intent:** clear bits denote available positions, set bits denote unavailable positions, and the requested length limits an allocation. These domain meanings are not established by the supplied function alone.

Two limitations constrain the contract:

- Writeback calls to F-b5c826d2ab04 have no explicit argument in the supplied representation. The function visibly stores their returned values, but the effective input and resulting stored bit pattern are unresolved.
- Dependencies may modify memory through context, acquired storage, or retained references. Consequently, this contract distinguishes the function’s own output stores from output values surviving until return.

No runtime tests have been performed.

## Interface

The interface accepts six arguments and returns a signed 32-bit status.

| Argument | Interpretation |
|---|---|
| Context | Opaque 32-bit value passed to F-1f96b6eaec84 |
| Initial position | Unsigned 32-bit position at which searching begins |
| Exclusive end | Unsigned 32-bit search limit and initial marking boundary |
| Requested length | Unsigned 32-bit value used to calculate the effective length |
| Start output | Pointer to writable unsigned 32-bit storage |
| Count output | Pointer to writable signed 32-bit storage |

The source establishes four-byte storage words and four-byte word strides. It does not establish a portable pointer width or a public structure layout for acquired storage.

### Arithmetic and layout

Let \(S\) denote the initial position, \(E\) the supplied end, and \(R\) the requested length. The effective length is calculated before the first dependency call:

\[
L=\min\bigl((E-S)\bmod 2^{32},R\bigr).
\]

Position additions, subtractions, comparisons, and masks use unsigned 32-bit arithmetic. An end below the start therefore produces a wrapped difference, not an immediate interval error.

The initial word index within acquired storage is:

\[
(S\mathbin{\&}\mathrm{0xFE0})\mathbin{\gg}5.
\]

This selects one of 128 words. The traversal treats each block as 128 words, corresponding to 4,096 bit positions. How an absolute position selects a particular acquired block is delegated to F-1f96b6eaec84.

Within a transformed word, the initial mask is:

\[
\mathrm{0x80000000}\mathbin{\gg}(S\mathbin{\&}31).
\]

Increasing positions therefore proceed from the most-significant bit toward the least-significant bit.

The count calculation subtracts a reloaded start-output value from the current position using unsigned 32-bit arithmetic, then stores that result into signed 32-bit storage. The supplied representation does not establish a portable interpretation for results above the signed positive range.

### Preconditions

No explicit validation of arguments, pointer alignment, aliasing, or acquired-storage size is visible.

Defined, usable access requires:

- Writable output locations whenever the function reaches an output store.
- Acquired storage that supports every word access actually reached.
- Valid memory for preparation and finalization calls.
- A dependency contract that keeps any accessed storage usable for the required duration.

An ordered, nonwrapping interval, a positive requested length, and disjoint output storage are appropriate restrictions for ordinary allocation-like use. They are not checks performed by this function.

## Behavior

### Initial acquisition and read

The function initializes a block-pointer slot to null and passes its address, the context, and the initial position to F-1f96b6eaec84.

A nonzero acquisition result selects failure handling. A zero result permits the following sequence:

1. Capture the acquired pointer for word addressing.
2. Call F-d5b4823d3579 with the acquired pointer.
3. Address the initial word using the calculated word index.
4. Read that 32-bit word and pass its value to F-b5c826d2ab04.
5. Use the returned 32-bit value as the working word.

The captured data pointer and the block-pointer slot are distinct state. A later dependency could change the slot through a retained reference without changing the already captured data pointer.

Acquisition, preparation, and the initial word read occur before checking whether the initial position has reached or exceeded the supplied end.

### Searching past set bits

Searching continues while the current position is unsigned-less-than the supplied end and the selected bit in the working word is set.

Each advance shifts the mask right and increments the position. When the mask becomes zero, traversal advances to the next word and restores the most-significant-bit mask.

At the end of a 128-word block, the function:

1. Calls F-1e9d170e476a with the pointer currently obtained from the block-pointer slot and a zero second argument.
2. Takes the failure path if that call returns nonzero.
3. Otherwise explicitly clears the block-pointer slot.
4. Calls F-1f96b6eaec84 for the incremented position.
5. Takes the failure path if acquisition returns nonzero.
6. On success, captures the new data pointer, calls F-d5b4823d3579, and resumes at the first word.

After a successful word transition, the next word is read and its value is passed to F-b5c826d2ab04.

Word and block transitions occur before the search condition is reevaluated. Thus, consuming a set bit immediately before the end can cause a read of the following word, or finalization and acquisition of the following block, even though the incremented position has reached the end. An error during that transition takes precedence over the later exhausted-search check.

After searching, equality between the current position and the supplied end selects status **−34** and failure handling. The check is equality, not greater-than-or-equal. An initial position above the end bypasses searching but is not rejected merely for being above the end.

### Publishing a candidate and choosing the marking boundary

If the current position is not equal to the supplied end, the function stores that position into the start output.

Let \(P\) denote that candidate position. The supplied end remains the marking boundary unless:

\[
P < (E-L)\bmod 2^{32}.
\]

When this comparison is true, the boundary becomes:

\[
(P+L)\bmod 2^{32}.
\]

For ordered, nonwrapping inputs with positive effective length, this selects the earlier of the supplied end and the candidate plus the effective length. That interpretation is not generally valid for malformed or wrapping inputs.

The candidate store precedes marking-related dependency calls and word stores. It does not guarantee that a later read of the output will still obtain the candidate.

### Traversing clear bits and writing words

While the selected working-word bit is clear, the function increments the position and sets that bit in the working word.

It then checks whether the incremented position equals the marking boundary. Equality stops traversal immediately, before shifting the mask or entering another word. Otherwise, traversal advances the mask.

A set bit encountered next stops marking without consuming that position. This is ordinary completion, not an error; the traversed run may be shorter than requested.

When the mask is exhausted without reaching the boundary:

- F-b5c826d2ab04 is called without an explicit argument in the supplied representation.
- Its returned 32-bit value is stored into the word just processed.
- Traversal advances to the next word.

At a block boundary, that completed-word store occurs before F-1e9d170e476a. Successful boundary finalization is followed by explicit slot clearing, acquisition, preparation, and reading of the next block. Any nonzero boundary-finalization or acquisition status selects failure handling.

After ordinary termination of the marking loop, another argument-unresolved call to F-b5c826d2ab04 supplies the value stored into the current word. This final word store occurs even if the marking loop executes zero iterations.

The calculation of the working word is established: each traversed clear bit is set. The relationship between that calculated word and the values returned for writeback remains **documented_only**.

### Output stores and status selection

After ordinary marking completion and its final word store, the function reloads the start output, subtracts that value from the current position modulo \(2^{32}\), and stores the result into the count output.

Under noninterference and nonaliasing assumptions, this count is the number of positions traversed during marking. Without those assumptions, it is specifically the subtraction using the value read at that time.

The ordinary completion status is zero.

Explicit failure handling stores zero into the start output, then zero into the count output. It is reached for:

- Initial acquisition failure.
- Boundary-finalization failure during either phase.
- Subsequent acquisition failure.
- Exhausted search, with status −34.

Previously performed word stores remain earlier observable effects. There is no visible rollback.

### Final cleanup

After success or failure output stores, the function reads the current block-pointer slot. If its value is non-null, it calls F-1e9d170e476a using that value and a zero second argument.

The cleanup return value is ignored and does not replace the selected status. Its memory effects are not established. In particular, output stores made before cleanup are not unconditional guarantees about output values at return.

## Side effects

### Established ordering

The function’s own operations establish these orderings:

- Initial acquisition precedes preparation and the initial word read.
- The candidate-start store precedes marking word stores.
- A completed-word store precedes marking-boundary finalization.
- Successful boundary finalization precedes explicit slot clearing and the next acquisition.
- The final marking-word store precedes the start-output reload and count store.
- Failure clearing stores start first and count second.
- Final cleanup follows either the success count store or the failure clearing stores.

On boundary-finalization failure, the function itself does **not** clear the block-pointer slot. This does not establish that the slot remains non-null or retains its earlier value. F-1f96b6eaec84 received its address, and retention of that address is unresolved. Later dependencies cannot be assumed to preserve the slot.

Cleanup therefore acts on the slot’s current value: it may make no call, call with the previously used pointer, or call with another pointer. The supplied function alone does not select among those outcomes.

Likewise, failed acquisition does not establish whether the slot is null at cleanup.

### Ownership and persistence

The function acquires access to blocks and passes pointers to preparation and finalization dependencies. The evidence does not establish whether these operations allocate, retain, lock, map, flush, unlock, unmap, release, or free anything.

No repeat-call safety or persistence guarantee is established. A successful finalization status does not by itself establish that a prior word store is durable.

### Aliasing and dependency interference

The function does not enforce separation between outputs, acquired storage, or other dependency-accessible memory.

Consequences include:

- A word store may overwrite the start output before the count calculation.
- A dependency may alter either output through context or a retained reference.
- Overlapping output locations may cause the later count store to overwrite the earlier start store.
- Cleanup may change outputs after all explicit output stores are complete.

These possibilities affect memory observations, not the fact that the function performs the described stores and ignores the cleanup return value. No general preservation of globals or shared memory is assumed.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-1f96b6eaec84 | Receives context, position, and the address of the block-pointer slot; returns a status | Mapping, ownership, failure-output behavior, retention of the slot address, and other memory effects |
| F-d5b4823d3579 | Called after successful acquisition and before reading the acquired block | Preparation semantics, storage mutation, resource effects, and indirect memory effects |
| F-b5c826d2ab04 | Receives an explicit word value on reads; returns values used for working words and writeback | Transformation semantics, effective input on writeback calls, and side effects |
| F-1e9d170e476a | Receives a slot-derived block pointer and zero at boundaries and cleanup | Lifetime, persistence, release behavior, repeat-call safety, slot mutation, and output mutation |

Acquisition and boundary-finalization results control explicit error propagation. Final-cleanup results do not.

## Acceptance examples

These are conditional static expectations, not executed tests.

Unless stated otherwise, examples assume valid disjoint output storage, stable usable acquired storage, successful dependencies, and no dependency modification of either output location—including through context, aliases, or retained references and during final cleanup. They also assume that dependency effects do not disrupt the described traversal.

Examples use logical bits as returned by the read transformation. Persisted marked-bit claims additionally require the unresolved writeback behavior to encode the calculated working word.

### Set prefix and a short clear run

For initial position 100, end 120, and requested length 5, suppose positions 100–101 are set, 102–104 are clear, and 105 is set.

The function stores start 102 and count 3 and returns zero. Under the stated noninterference assumptions, those are also the output values at return. Positions 102–104 are set in the working calculation; position 105 is not consumed.

### Request-limited run

For initial position 100, end 120, and requested length 5, suppose the first clear position is 102 and positions 102–106 are clear.

The function stores start 102 and count 5 and returns zero. It stops at position 107 without consuming that position.

### End-limited run

For initial position 100, end 104, requested length 10, and clear bits throughout the interval, the effective length is 4.

The function stores start 100 and count 4. The final word store occurs before the count store and cleanup.

### Exhausted search and an empty interval

For initial position 100, end 104, and set bits throughout the interval, the function selects −34 and stores zero into both outputs. No explicit marking-word store occurs.

For initial position equal to the end, successful initial acquisition is still followed by preparation and a word read before −34 is selected.

Both expectations require any reached transition dependencies to succeed; a transition failure can select its own status first.

### Zero requested length

For an ordered nonempty interval, suppose searching finds a clear candidate. A zero requested length makes the marking boundary equal to that candidate.

The marking loop nevertheless increments the position and sets the selected working bit before checking equality. Its first increment therefore misses that boundary. Zero length must not be accepted as a guaranteed zero-count or no-write operation. Later behavior depends on encountered bits, dependency results, and possible wraparound.

### Boundary-finalization failure

Suppose marking stores a completed word and boundary finalization then returns a nonzero status.

The function preserves that status for return and stores zero into each output. It does not roll back the earlier word store and does not itself clear the block-pointer slot on this failure path.

Cleanup then examines the current slot. A second call with the same pointer is expected only if the slot still contains that pointer. A null slot suppresses cleanup; a changed non-null slot supplies the changed pointer.

### Cleanup status and output mutation

If ordinary marking completes and cleanup returns nonzero, that return value does not replace status zero.

If cleanup leaves outputs unchanged, the previously stored outputs remain observable at return. If cleanup modifies an output through an accessible reference, its modified value may instead be observed. Ignoring cleanup’s status does not imply preservation of memory.

## Uncertainties

The following remain **documented_only**:

- Effective inputs and exact results of writeback transformation calls.
- Whether the transformation performs byte-order conversion.
- Domain meanings of set and clear bits.
- Block mapping, ownership, lifetime, persistence, and release semantics.
- Retention and later mutation of the block-pointer slot.
- Dependency modification of outputs, shared memory, or globals.
- Synchronization, reentrancy, and concurrent mutation behavior.
- Caller enforcement of ordered intervals, positive lengths, valid storage, nonaliasing, and signed-count limits.
- Termination and useful range semantics for malformed or wrapping inputs.

Static evidence establishes unchecked wrapping arithmetic and the absence of a zero-request no-op guard. It does not justify repairing those cases into an intended allocation contract.

## Evidence

This specification is based solely on the supplied static function representation and opaque dependency mapping.

Direct evidence supports the 32-bit arithmetic, four-byte word layout, 128-word traversal, most-significant-bit-first masks, exact end-equality check, −34 status, ordered stores, start-output reload, dependency-status propagation, and ignored final-cleanup result.

The distinction between captured data pointers and the mutable block-pointer slot follows from passing the slot’s address to acquisition and subsequently reading the slot for finalization. Output preservation cannot be established because dependency memory effects are unavailable.

Allocation intent and storage-format conversion remain interpretations rather than established dependency contracts. No runtime execution, tests, dependency implementation inspection, or external validation has been performed.
