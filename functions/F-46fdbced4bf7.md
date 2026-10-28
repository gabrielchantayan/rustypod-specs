<!-- Trusted provenance: F-46fdbced4bf7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-46fdbced4bf7

## Purpose

F-46fdbced4bf7 conditionally constructs a three-word record and publishes its address through a caller-supplied output slot. A zero construction value selects an empty result. A nonzero construction value requests the record; a separate nonzero option additionally requests a two-word auxiliary block and invokes an opaque dependency before publication.

On normal return, the function returns the original output-slot address, not the constructed record’s address.

“Record,” “auxiliary block,” and “publication” describe observable storage layout and operations. They do not establish an application object type, reference-counting scheme, ownership transfer, or synchronization protocol.

The contract distinguishes explicit operations from final memory contents. Dependency effects and overlapping storage can change the results of earlier stores. In particular, the final publication store can overwrite a word within either constructed block if the output slot overlaps that block.

## Interface

### Inputs

The function accepts three inputs:

| Input | Representation | Observable use |
|---|---|---|
| Output-slot address | Address of a four-byte writable slot | Receives zero initially and the primary-record address after successful completion of the construction path |
| Construction value | 32-bit integer | Tested against zero; if nonzero, stored unchanged in the primary record’s first word |
| Auxiliary option | 32-bit integer | Tested against zero only on the nonzero-construction path |

Both integer decisions distinguish only zero from nonzero. Negative values select the same branches as other nonzero values. No magnitude-dependent allocation size or input range validation is visible.

The function receives no explicit storage length, allocator context, error-output location, destructor, or ownership flag.

### Return value

Every normally returning path returns the original output-slot address.

The returned address alone does not distinguish an empty result from a constructed result. The output slot is the publication location, subject to the validity and dependency conditions described below.

No separate error status is produced.

### Storage representation

The visible layout uses four-byte words and 32-bit address encodings. This contract describes that representation; it does not establish behavior for an address space requiring wider stored addresses.

The primary storage request is 12 bytes. Its explicit initialization stores are:

| Byte offset | Store width | Value written |
|---|---:|---|
| 0 | 4 bytes | Construction value |
| 4 | 4 bytes | Integer 1 |
| 8 | 4 bytes | Zero |

On the auxiliary path, the word at offset 8 subsequently receives the auxiliary-block address.

The auxiliary storage request is 8 bytes:

| Byte offset | Store width | Value written |
|---|---:|---|
| 0 | 4 bytes | Zero |
| 4 | 4 bytes | Zero |

These tables specify writes, not unconditional final-content guarantees. Later writes through overlapping addresses, as well as dependency effects, can alter previously initialized words. No byte order is independently established.

### Preconditions for normal completion

The output slot must support the initial four-byte store. On a construction path, it must also support the final publication store when that store occurs.

Each address returned by F-8157a180d901 must support the writes performed through it: three four-byte words for the primary request and two for the auxiliary request. The primary storage must remain usable for the auxiliary-address store when that branch is selected. Stored addresses must be representable in the evidenced 32-bit fields.

No checks enforce these requirements. The supplied evidence does not establish allocation alignment, distinctness, absence of overlap, or continued validity across opaque calls. Final-content guarantees therefore require additional conditions where stated.

## Behavior

### Initial output clearing and the empty path

The first visible memory operation writes zero to the output slot. This occurs before either branch and before any dependency call.

The function does not first read the slot’s previous contents. It performs no visible release or destruction of a previously stored object.

If the construction value is zero:

- No allocation is requested.
- The auxiliary option has no effect.
- F-b00260d1087a is not invoked.
- The function returns the original output-slot address after the initial clear.

For valid output storage, the only explicit memory modification on this path is the four-byte zero store.

### Primary construction

For a nonzero construction value, the function invokes F-8157a180d901 with a size argument of 12 bytes.

After that call returns, it uses the returned address without a visible null check, capacity check, or other validation. It writes the construction value at offset 0, then integer 1 at offset 4, then zero at offset 8.

If the auxiliary option is zero, the next explicit operation is publication: the function writes the primary-record address into the output slot. It then returns the original output-slot address.

There is no intervening dependency call between the primary initialization stores and publication on this branch. Nevertheless, publication can itself modify record contents if the output slot overlaps the primary storage. Thus, the initialization sequence alone does not guarantee that the three record words retain their initialized values at return.

### Auxiliary construction

For a nonzero auxiliary option, the function requests 8 bytes from F-8157a180d901 after completing the three primary initialization stores.

After the second request returns, the function:

1. Writes zero to the auxiliary storage at offset 0.
2. Writes zero to the auxiliary storage at offset 4.
3. Writes the auxiliary address into the primary record at offset 8.
4. Invokes F-b00260d1087a.
5. If that invocation returns normally, writes the primary-record address into the output slot.
6. Returns the original output-slot address.

No explicit argument is visible at the F-b00260d1087a call site, and no result from that call is consumed. This does not establish that the dependency cannot access the output slot, either block, or global state.

The function performs no dedicated restoration of the primary record’s first two words after the second allocation or the final dependency call. It likewise performs no dedicated restoration of the auxiliary words after F-b00260d1087a. This is an operation-level statement, not a guarantee that changed values remain changed: the final publication store may overwrite overlapping storage and may coincidentally restore a prior value.

### Observable ordering and aliasing

For a normally completing auxiliary invocation, the explicit sequence is:

- Clear the output slot.
- Request 12 bytes.
- Initialize the primary words in ascending offset order.
- Request 8 bytes.
- Clear the auxiliary words in ascending offset order.
- Install the auxiliary address in the primary record.
- Invoke F-b00260d1087a.
- Publish the primary address.

There are no explicit reads of the previous contents of the output slot or either constructed block. Addresses and input values are used to perform stores.

This order does not imply that the output slot stays zero until publication. A dependency may modify it, and a store into overlapping constructed storage may also modify it.

Similarly, the absence of dedicated restoration does not imply that an earlier value can never reappear. Every later store must be interpreted at its actual destination:

- Auxiliary clearing can overwrite primary fields if the two blocks overlap.
- Installing the auxiliary address can overwrite part of the auxiliary block if its storage overlaps the primary record’s third word.
- Final publication can overwrite primary or auxiliary contents if the output slot overlaps them.

The available evidence neither requires nor excludes these overlaps. Claims about independently retained field values require appropriate nonoverlap assumptions.

### Integer behavior

The construction value is copied without visible normalization or arithmetic transformation. In particular, it is not reduced to a Boolean value.

The auxiliary option controls branching but is not stored in either block.

Allocation sizes are fixed at 12 and 8 bytes. The function contains no input-dependent size calculation, multiplication, overflow check, or integer-error branch. Field accesses use fixed four-byte offsets.

Address publication and the auxiliary-address field use 32-bit encodings. Portability to wider stored addresses is not established.

## Side effects

The function explicitly writes:

- Zero to the output slot on every invocation.
- Three initialization words to the primary storage on nonzero-construction paths.
- Two zero words to auxiliary storage when selected.
- The auxiliary address to the primary record’s third word when selected.
- The primary address to the output slot on normally completed construction paths.

It invokes F-8157a180d901 zero, one, or two times, according to the selected path. It invokes F-b00260d1087a exactly once on the auxiliary path if execution reaches that call.

No explicit release, rollback, destruction, reference-count adjustment, logging, or error reporting is visible. Dependencies may perform effects not established here.

The evidence does not identify the owner of either returned block, the caller’s eventual release obligations, or whether the final dependency changes storage lifetime. Overwriting the output slot does not visibly dispose of its former contents.

If a dependency does not return normally, no cleanup or recovery path is visible in this function. If an address is unsuitable for a subsequent store, there is no checked conversion to an empty result or error status. The resulting failure behavior is unresolved.

No synchronization operation is directly visible. The publication store alone does not establish atomicity, inter-thread ordering, or safe concurrent access.

## Dependencies

### F-8157a180d901

**Established use:**

- Receives 12 on every nonzero-construction path.
- Receives 8 on the auxiliary path, after primary initialization.
- Returns a value immediately used as a storage address.
- Has no visible return-value validation at these call sites.

**Inferred intent:** a storage-allocation service.

**Unresolved guarantees:** failure behavior, initialization policy, alignment, overlap, distinctness, ownership, lifetime, and effects on existing memory or globals.

The second call occurs while the primary record already contains initialized words. The caller does not protect those words against dependency effects or reinitialize the entire record afterward.

### F-b00260d1087a

**Established use:**

- Called only on the auxiliary path.
- Called after auxiliary clearing and installation of the auxiliary address.
- Called before final publication.
- Receives no visible explicit argument.
- Produces no visibly consumed result.

Its purpose is unresolved. No preservation, ownership, release, synchronization, or global-state guarantee may be inferred from its position or apparent lack of arguments.

## Acceptance examples

These are conditional consequences of the static evidence, not executed tests.

### Empty construction

Given writable output storage, construction value zero, and any auxiliary option:

- The function writes zero to the output slot.
- Neither dependency is invoked.
- The returned address equals the supplied output-slot address.

A preexisting slot value is overwritten without a visible read or release.

### Primary-only construction with independent storage

Given construction value 27 and auxiliary option zero, suppose F-8157a180d901 returns normally with valid 12-byte storage that does not overlap the output slot.

The primary words are written as 27, 1, and 0, in that order. Publication then places the primary address in the output slot without changing those words. The function returns the output-slot address and does not invoke F-b00260d1087a.

### Negative inputs

Given construction value −1 and auxiliary option zero, construction occurs. Under the preceding storage conditions, the first primary word contains the 32-bit representation of −1, not integer 1.

Given any nonzero construction value and auxiliary option −1, the auxiliary branch is selected just as it is for auxiliary option 1.

### Auxiliary construction with preserving dependencies

Given construction value 27 and a nonzero auxiliary option, suppose:

- Both requests return valid storage and all calls return normally.
- The blocks do not overlap each other or the output slot.
- Dependencies preserve previously initialized storage and keep it valid.

After completion, the primary words contain 27, 1, and the auxiliary address. Both auxiliary words contain zero. The output slot contains the primary address, and the return value is the output-slot address.

F-b00260d1087a is invoked before the final publication store. Its actual purpose remains unspecified.

### A dependency changes a nonoverlapping word

Suppose F-b00260d1087a changes an initialized word, keeps the relevant storage valid, and returns normally. If that word does not overlap the output slot, the function performs no subsequent store to it. The changed value therefore persists through this function’s remaining explicit operations.

Without that nonoverlap condition, persistence is not guaranteed: final publication can overwrite the changed word.

### Publication can restore an overlapping word

Suppose the output slot coincides with the primary record’s first word, all required accesses are valid, and the nonzero construction value equals the primary address’s 32-bit encoding.

Primary initialization writes that encoding into the first word. Suppose F-b00260d1087a later changes the word and returns normally. Final publication writes the primary address into the same location, restoring the value originally written during initialization.

This is not a dedicated restoration operation. It is a consequence of aliasing and the ordinary publication store.

### Unsuitable returned storage

If either storage request returns an address unsuitable for the ensuing writes, the function does not detect that condition and select a clean empty result. It attempts the corresponding unchecked stores. The concrete failure outcome is not established.

## Uncertainties

The following remain **documented_only**:

- Application meaning of the construction value and constant second word.
- Semantic role of the auxiliary block.
- Dependency implementations, failure modes, and nonlocal control flow.
- Allocation ownership, release obligations, and lifetime.
- Alignment, allocation distinctness, and overlap guarantees.
- Dependency access to the output slot, constructed storage, or globals.
- Purpose and effects of F-b00260d1087a.
- Concurrency requirements and memory-ordering guarantees.
- Byte order and behavior outside the evidenced 32-bit representation.
- Concrete consequences of invalid addresses or invalidated storage.

The constant 1 does not by itself establish reference counting. An address-bearing field does not establish a list or ownership relationship. A dependency call before publication does not establish synchronization.

## Evidence

**Established static evidence:** the supplied function establishes the initial output clear, zero-versus-nonzero decisions, fixed storage requests, four-byte field layout, ordered stores, optional dependency invocation, final primary-address publication, and return of the original output-slot address.

**Static consequences:** later stores can overwrite earlier stores when their destinations overlap. In particular, final publication can change or restore a word also belonging to constructed storage. This follows from the specified store destinations; it does not assert that an actual allocation service returns overlapping storage.

**Inferred intent:** F-8157a180d901 appears to provide storage, and the initialized blocks appear to represent a primary object with optional auxiliary state. These interpretations do not supply missing dependency guarantees.

**Unknown:** external implementations, runtime allocation behavior, ownership rules, and application semantics are not supplied.

No runtime tests have been performed.
