<!-- Trusted provenance: F-ec82dd9bf147; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec82dd9bf147

## Purpose

F-ec82dd9bf147 retrieves an object from a state-local cache, obtaining and optionally initializing an object when the selected cache slot is empty.

Selection has two stages: the requested index selects a byte from a selector array; that byte selects a descriptor from an external registry. The descriptor’s first byte identifies the cache slot. Different indices or descriptors can therefore resolve to the same cached pointer.

On a cache miss, the function calls F-3ce007b692cc, writes the selected descriptor pointer into the returned object when status permits, and optionally invokes an initialization dependency. Successful initialization permits cache publication. Failed initialization selects a cleanup path.

The result consists of a 32-bit status return and a final four-byte write to the caller’s output location. Neither a nonzero status nor a zero status independently determines whether the output is null. In particular, initialization failure clears the candidate pointer, but cleanup dependencies may change the status before it is returned.

## Interface

### Arguments and return value

| Item | Established representation and role |
|---|---|
| State pointer | Pointer to a structure containing a context pointer, unsigned index bound, selector-array pointer, and inline cache-pointer slots |
| Requested index | Unsigned 32-bit integer |
| Output location | Address receiving one four-byte pointer representation |
| Return value | 32-bit integer read from the function’s status storage at return |

The supplied representation uses four-byte pointer storage and four-byte words. Endianness is not established.

Status starts at zero. An out-of-range request explicitly sets it to 6. Other status values can originate from F-3ce007b692cc, an initialization result, or subsequent modification of status storage whose address was exposed to F-3ce007b692cc. No broader status-code vocabulary is established.

### State layout

Offsets are measured in bytes from the state pointer.

| Offset | Width | Meaning established by access |
|---|---:|---|
| 0 | 4 bytes | Context pointer |
| 4 | 4 bytes | Bound used in an unsigned comparison |
| 8 | 4 bytes | Selector-array pointer |
| 12 + 4 × slot | 4 bytes | Cached object pointer |

The slot identifier is an unsigned byte, ranging from 0 through 255. The cache-pointer starting offsets therefore range from 12 through 1032. Accessing slot 255 requires valid storage through byte offset 1035.

These are possible access locations, not proof that every state allocation contains all 256 slots.

### Selection data and descriptor layout

An in-range requested index addresses one byte in the selector array. That byte selects a four-byte descriptor-pointer entry in an external registry.

The descriptor fields used here are:

| Offset | Width | Meaning established by access |
|---|---:|---|
| 0 | 1 byte | Cache slot identifier |
| 8 | 4 bytes | Argument to F-3ce007b692cc |
| 12 | 4 bytes | Optional initialization dependency pointer |
| 20 | 4 bytes | Optional failure-cleanup dependency pointer |

The selected descriptor pointer is retained for the remainder of this invocation. The function does not explicitly repeat the registry lookup after calling a dependency. Fields within that descriptor may nevertheless change.

### Context and object requirements

On a cache miss, the function reads a four-byte value at context offset 100. It retains that value for the call to F-3ce007b692cc and, if initialization fails, the later call to F-bf7242c98d06.

A zero status after F-3ce007b692cc permits a four-byte descriptor-pointer write at offset zero of the returned object. The function does not first check whether that returned pointer is null.

### Indirect dependency interfaces

The initialization dependency receives the candidate object pointer and the context pointer read from the state for that invocation. Its 32-bit result is assigned to status storage. That assigned result is checked immediately: zero selects publication; nonzero selects failure cleanup.

The initialization result is not necessarily the eventual function return. F-3ce007b692cc has already received the address of status storage. Its implementation and any retention or disclosure of that address are unknown. Later cleanup dependencies may therefore change the stored status.

The optional failure-cleanup dependency receives only the candidate object pointer. Its return value, if any, is not used.

Opaque IDs for these descriptor-selected indirect dependencies were not supplied.

### Preconditions

The function performs no explicit pointer validation beyond null tests on optional dependency pointers and the cached object pointer.

Normal completion requires all actual accesses to be valid, including:

- Reading the state bound, even when the request is rejected.
- Writing four bytes to the output location on every normally completed path.
- Reading the selector byte, registry entry, descriptor slot byte, and selected cache slot for an in-range request.
- Reading the context pointer and context offset-100 field on a cache miss.
- Writing the object’s first four bytes after a zero-status result from F-3ce007b692cc.
- Calling each non-null indirect dependency with its established arguments.

No non-aliasing, synchronization, or exclusive-access guarantee is enforced.

## Behavior

### Range decision

The function starts with a null candidate pointer and zero status. It compares the requested index with the state bound as unsigned 32-bit values.

If the index is greater than or equal to the bound, the function sets status to 6 and proceeds to the final output write with a null candidate. This path does not explicitly access the selector array, descriptor registry, cache, context contents, or dependencies.

A bound with its high bit set remains a large unsigned bound; it is not treated as negative. With the maximum bound of 4,294,967,295, index 4,294,967,294 passes the comparison, while index 4,294,967,295 does not.

Passing the comparison establishes no backing-storage guarantee.

### Descriptor selection and cache hit

For an in-range request, the function follows this data-dependent access sequence:

1. Obtains the selector-array pointer from the state.
2. Reads the selector byte at the requested index.
3. Obtains the descriptor pointer from the registry entry selected by that byte.
4. Reads the descriptor’s slot byte.
5. Reads the corresponding cached object pointer from the state.

A non-null cached pointer becomes the candidate. The function then proceeds directly to the final output write.

On this path, it does not explicitly:

- Access the context’s offset-100 field.
- Call F-3ce007b692cc or F-bf7242c98d06.
- Read or invoke the optional dependency fields.
- Rewrite or validate the cached object’s descriptor word.
- Verify that the cached object belongs to the currently selected descriptor.

Without interfering mutation, status remains zero on a cache hit.

### Obtaining an object on a cache miss

For a null cached pointer, the function reads the state’s context pointer and captures the four-byte value at context offset 100.

It calls F-3ce007b692cc with:

- The captured context-field value.
- The descriptor’s four-byte field at offset 8.
- The address of the function’s status storage, initially containing zero.

The returned pointer becomes the candidate. The function then checks the status storage, not the candidate pointer, to determine the next branch.

If status is nonzero at that check, the function skips the object-header write, initialization, explicit cache publication, and failure cleanup. It proceeds to the final output write with the pointer returned by F-3ce007b692cc.

Consequently, a nonzero status can accompany a non-null output. The usability, ownership, and cleanup obligations of that pointer are not established.

If status is zero, the function writes the selected descriptor pointer into the candidate’s first four bytes. A null candidate with zero status therefore causes an attempted write through null rather than a locally generated error result.

The supplied representation does not establish a total evaluation order among independent argument reads within a dependency call. The captured context-field value is obtained before F-3ce007b692cc is invoked.

### Initialization and publication

After the object-header write, the function tests the descriptor’s initialization dependency pointer.

If it is null, the function skips initialization and takes the publication path.

If it is non-null, the function invokes the initialization dependency with the candidate and a context pointer read from the state for that call. This context pointer can differ from the one used earlier to obtain the captured offset-100 value: F-3ce007b692cc may have changed state.

The initialization result is stored in status storage. A zero result selects publication; a nonzero result selects failure cleanup.

On the publication path, the function reads the descriptor’s slot byte again and writes the candidate pointer into that state cache slot. It does not repeat the original cache lookup or verify that the destination is still empty.

Thus, mutation of the descriptor’s slot byte can redirect publication. An intervening write to the eventual destination can be overwritten.

The optional dependency pointer is read for its null test and referenced again as the indirect call target. The body establishes no immutable snapshot of a non-null callback field. Concurrent mutation of that field is not given a safe execution contract.

### Initialization failure and status after cleanup

A nonzero initialization result selects this sequence:

1. Test the descriptor’s failure-cleanup dependency pointer.
2. Invoke that dependency with the candidate if its pointer is non-null.
3. Call F-bf7242c98d06 with the previously captured context-field value and the candidate.
4. Set the candidate pointer to null.
5. Perform the final output write and return the status then remaining.

The branch is selected by the initializer’s nonzero result. The eventual return is **not unconditionally that result**.

F-3ce007b692cc received the address of status storage. Nothing supplied establishes that it discards that address or prevents later dependencies from obtaining it. The optional failure-cleanup dependency or F-bf7242c98d06 may therefore modify status through a retained or shared address. The returned status can differ from the initialization result, including becoming zero.

If cleanup preserves status, the initialization result remains the returned status. Regardless of such status changes, this already-selected failure path does not subsequently publish the candidate merely because status becomes zero. After returning cleanup calls, it clears the candidate and writes null to the output.

No explicit cache publication occurs on this branch. Dependencies and aliases may still modify cache storage.

### Final output and return

Every normally completed path writes the current candidate pointer to the supplied output location and then returns the current status.

There is no preliminary output initialization. During dependency calls, the output location retains its previous contents unless an intervening dependency or aliased write changes it.

The final output write is unconditional with respect to status. If that write aliases other relevant storage, its effects occur before the status return.

## Side effects

### Direct writes and ordering

The function’s explicit externally relevant writes are:

- The descriptor pointer into the candidate object after the zero-status allocation check.
- The candidate pointer into a cache slot on the publication path.
- The candidate pointer into the caller’s output location on normal completion.

It also initializes and updates status storage and exposes its address to F-3ce007b692cc.

For successful creation, the object-header write precedes initialization, any initialization call precedes publication, and publication precedes the final output write.

For initialization failure, the object-header write and initialization precede optional failure cleanup; optional failure cleanup precedes F-bf7242c98d06; that call precedes candidate clearing and the final null output write.

These orderings concern the function’s own operations. They do not restrict additional effects inside dependencies.

### Aliasing and mutation

No checks prevent the output location from overlapping the state, a descriptor, the context, or object storage. Its final four-byte write may therefore alter those structures, including on error paths.

The captured context-field value is retained independently of later context changes. In contrast, the initialization context pointer and publication slot byte are read later.

Dependencies may modify globals, state, descriptors, cache entries, objects, and exposed status storage. No preservation guarantee can be derived merely because an item is not an explicit argument to a later call.

### Ownership and lifetime

A cache hit returns the existing pointer without any visible retain operation.

Successful creation stores and outputs the same candidate pointer. The body does not establish whether the caller borrows that pointer, owns it, or shares ownership with the cache.

Initialization failure passes the candidate to cleanup dependencies and subsequently outputs null. Neither cleanup dependency is established to free memory. Conversely, calling them both does not establish that both may safely release the same allocation; that compatibility is an external requirement.

Allocation-status failure outputs the dependency’s pointer without invoking cleanup. Ownership for that case is unresolved.

Exposure of status storage does not establish permission to use its address after this function returns.

## Dependencies

| Dependency | Established interaction | Unresolved contract |
|---|---|---|
| F-3ce007b692cc | Called on an observed cache miss with the captured context-field value, descriptor offset-8 value, and writable status address; returns the candidate pointer | Allocation mechanism, argument units, pointer validity, ownership, status-address retention, shared-state preservation |
| F-bf7242c98d06 | Called on the initialization-failure branch after optional failure cleanup; receives the captured context-field value and candidate | Resource release, status preservation, global effects, ownership |
| Descriptor-selected initialization dependency; opaque ID unavailable | Optional call after the object-header write; receives candidate and current state context pointer; result selects publication or cleanup | Internal effects, synchronization, ownership changes |
| Descriptor-selected failure-cleanup dependency; opaque ID unavailable | Optional call after nonzero initialization result and before F-bf7242c98d06; receives candidate | Resource effects, access to escaped status storage, shared-state preservation |

The external registry is a data dependency. Its allocation size, initialization, mutability, and synchronization are not supplied.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless specified otherwise, assume valid storage, returning dependencies, stable shared data, no conflicting aliases, and no status mutation beyond the described writes.

1. **Index equals bound.** With bound 4 and index 4, return 6 and write null. No selector or registry access occurs.

2. **Zero bound.** Every unsigned index is rejected with status 6 and null output.

3. **Cache hit.** If selection reaches a slot containing pointer P, return zero and output P. No allocation or indirect dependency is invoked, and P’s first word is not checked.

4. **Creation without initialization.** For an empty slot, if F-3ce007b692cc returns writable P with zero status and the initialization pointer is null, write the descriptor pointer into P, publish P, output P, and return zero.

5. **Successful initialization.** If initialization returns zero, it is invoked after the descriptor-pointer write. Publication occurs only after it returns, using the slot byte read at publication.

6. **Initialization failure with status-preserving cleanup.** If initialization returns 9, optional failure cleanup and F-bf7242c98d06 both return without altering status, call them in that order, output null, and return 9.

7. **Initialization failure with changed status.** If initialization returns 9 and F-bf7242c98d06 changes exposed status storage to zero, the function still completes the failure path: no explicit publication, null output, and return zero.

8. **Allocation-status failure with a pointer.** If F-3ce007b692cc returns P and leaves status 12, output P and return 12. No object-header write, initialization, or failure cleanup occurs.

9. **Shared slot.** If two descriptors identify slot 7 and that slot contains P, either descriptor can lead to P. Descriptor identity is not validated against the object.

10. **Slot changes before publication.** If the initial lookup checks empty slot 2, initialization changes the descriptor’s slot byte to 5 and returns zero, publication writes P to slot 5 without checking whether slot 5 is empty.

11. **Unsigned upper boundary.** With bound 4,294,967,295, index 4,294,967,294 passes the range check; index 4,294,967,295 fails. The passing case remains conditional on valid address calculations and storage.

## Uncertainties

**documented_only — Escaped status storage.** F-3ce007b692cc receives its address. Retention, disclosure, and later access are unknown. In particular, cleanup cannot be assumed to preserve the initializer’s nonzero status.

**documented_only — Resource semantics.** The captured context field and descriptor offset-8 value have unknown types and units. Allocation size, alignment, lifetime, ownership, and release behavior are not established.

**documented_only — Address arithmetic.** Selector and slot bytes limit registry scaling to 1020 bytes and cache starting offsets to 1032 bytes. A passing requested index can reach 4,294,967,294. Pointer validity, address wrapping, and backing extent are not checked; their platform-specific consequences are unresolved.

**documented_only — Mutation and reentrancy.** No synchronization or reentrancy guard is visible. Dependencies may change descriptors, state, globals, cache contents, or escaped status storage. Stable-data assumptions in the examples are conditions, not guarantees.

**documented_only — Abnormal completion.** Invalid memory, invalid call targets, nonreturning dependencies, or a null zero-status candidate may prevent the final output write and return. Exact fault behavior is not supplied.

**Inferred intent.** The structure resembles lazy cached-object construction with failure teardown. That interpretation does not establish atomicity, allocation semantics, or ownership.

## Evidence

Established behavior comes solely from the supplied static body and opaque dependency mapping: the unsigned range check, byte-based selection, four-byte layouts, status-controlled allocation branch, object-header write, optional indirect calls, publication-slot reread, failure-cleanup order, final output write, and final status read.

The exposure of status storage to F-3ce007b692cc is explicit. Whether any dependency retains or later modifies that storage is unknown; preserving it cannot be assumed.

Dependency implementations, caller contracts, full allocation layouts, and runtime observations were not supplied. No runtime tests have been performed.
