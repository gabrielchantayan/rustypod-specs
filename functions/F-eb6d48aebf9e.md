<!-- Trusted provenance: F-eb6d48aebf9e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eb6d48aebf9e

## Purpose

F-eb6d48aebf9e attempts to obtain an eight-byte storage block, delegates initialization or processing of its second four-byte region to another function, and publishes the block through a caller-provided output location only when that dependency reports success.

The function returns zero for delegated success, propagates a nonzero status from the processing dependency, or returns −108 when the storage-acquisition dependency returns a null pointer. On the delegated-failure path, it submits the acquired block to a cleanup dependency and then publishes a null pointer.

These are established control-flow and data-flow properties. The interpretation of the block as an object, its first word as a type marker, and −108 as a particular named error are not established.

This contract describes behavior visible in the supplied static evidence. No runtime tests have been performed. Statements about completion assume that each invoked dependency returns normally and that the relevant memory accesses are valid.

## Interface

### Inputs

The function accepts three inputs:

| Input | Established representation | Role |
|---|---|---|
| Output location | Address of a four-byte writable slot in the represented target model | Receives the acquired block’s pointer representation on success, or zero after delegated failure |
| Opaque value | 32 bits | Forwarded unchanged as the first argument to F-40a81407b15f |
| Opaque byte | 8 bits | Forwarded as the third argument to F-40a81407b15f |

The output location is not read before being written. Its prior contents are neither interpreted nor explicitly released by this function.

The visible representation uses four-byte words, a four-byte offset to the block’s second region, and a four-byte output store containing a pointer. A 32-bit pointer model is therefore supported by the supplied representation. A portable contract for wider-pointer targets is not established.

The opaque byte has an eight-bit representation. Its signedness, semantic meaning, and any calling-convention extension are not established.

### Return value

The return value is represented as an integer status:

- **Zero:** F-40a81407b15f returned zero, the first block word was assigned, and the block pointer was stored through the output location.
- **A nonzero dependency status:** F-40a81407b15f returned that status, F-1137792fba7d was subsequently called with the block pointer, and zero was stored through the output location.
- **−108:** F-858f5b8d5984 returned null, and the function returned without directly writing the output location.

The negative constant is −108, not positive 108. The visible evidence does not establish that −108 is reserved exclusively for storage-acquisition failure: F-40a81407b15f could also return −108.

The status interface is consistent with a signed 32-bit integer target model, but the exact integer ABI is not independently established.

### Preconditions

For ordinary completion on any path that reaches output publication, the output location must permit a four-byte write. No null check or other validation of that address is visible.

A successful storage acquisition must provide memory suitable for the direct writes and dependency accesses that follow. The function requests eight bytes, but the actual allocation guarantees, alignment, initialization, and extent are properties of F-858f5b8d5984, not independently verified here.

The caller’s obligations for the opaque value and opaque byte depend on F-40a81407b15f. This function performs no visible validation of either input before forwarding it.

## Behavior

### Storage acquisition

The first visible operation is a call to F-858f5b8d5984 with the numeric argument 8.

If this call returns null:

- F-40a81407b15f is not called.
- F-1137792fba7d is not called.
- No direct read of either shared word described below occurs.
- No direct write through the output location occurs.
- The function returns −108.

Consequently, the function does not guarantee that a caller’s output slot becomes null on storage-acquisition failure. A preexisting value is not overwritten by this function on that path. Whether dependency side effects could independently alter the same storage is unknown.

### Delegated processing

For a non-null acquired block, the function invokes F-40a81407b15f with four arguments in this positional order:

1. The caller’s opaque 32-bit value.
2. The value read from a shared word, designated **shared parameter word** in this contract.
3. The caller’s opaque eight-bit value.
4. The address four bytes after the acquired block’s base.

The shared parameter word is consumed after storage acquisition has returned. It is not visibly captured before the acquisition call. If F-858f5b8d5984 changes that shared word and returns normally, the later read is the relevant one, subject to the target’s memory model.

The fourth argument points to the second four-byte region of the requested eight-byte block. This establishes the address passed to the dependency; it does not establish that the dependency writes exactly four bytes, writes any bytes at all, or confines all accesses to that region.

The dependency’s status controls the next branch. Success is specifically equality to zero. Every nonzero status takes the cleanup branch, regardless of sign or magnitude.

### Zero-status branch

After F-40a81407b15f returns zero, the function reads another shared word, designated **shared header word**, and stores its value into the first four bytes of the acquired block.

This read occurs after the processing call returns. The function does not visibly capture the shared header word before that call. A change made to it by F-40a81407b15f can therefore affect the value stored in the block.

The function then stores the acquired block pointer through the caller’s output location and returns zero.

The established layout at this point, before considering aliasing, is:

| Offset from block base | Size | Established meaning |
|---|---:|---|
| 0 | 4 bytes | Directly assigned the shared header word after delegated success |
| 4 | 4 bytes within the requested extent | Begins at the address passed to F-40a81407b15f; resulting contents are unspecified by this function’s evidence |

There is no visible direct initialization of the block before the processing call. In particular, the first word is not established as initialized when F-40a81407b15f runs.

F-1137792fba7d is not called by this function on the zero-status branch.

### Nonzero-status branch

After F-40a81407b15f returns a nonzero status, the function calls F-1137792fba7d with the acquired block’s base address.

If that cleanup call returns normally, the function:

- Selects null as the value to publish.
- Stores that null value through the output location.
- Returns the original nonzero processing status.

Any return value from F-1137792fba7d is not used to select the result. The visible function does not translate the processing status into a generic error.

The shared header word is not directly read or assigned to the block on this branch. The cleanup dependency may have its own accesses; those are unknown.

### Ordering and aliasing

The significant established order is:

- Storage acquisition precedes the shared-parameter read and processing call.
- Processing completes before either the header assignment or cleanup call.
- On success, the header assignment precedes the output-pointer store.
- On delegated failure, cleanup precedes the output-null store.
- The final output store precedes normal return on either processing-result branch.

No non-aliasing precondition is checked. If the output location overlaps the acquired block, the final output store can overwrite bytes established earlier. For example, an output location equal to the block base would cause the published pointer representation to replace the header word on success.

If the output location refers to memory made invalid by cleanup, the final null store is not made safe merely because it follows a failure. Defined completion requires that location to remain writable.

No byte-order conversion, variable-size calculation, or arithmetic derived from the opaque inputs is visible. The only block offset is the fixed four-byte displacement.

## Side effects

### Direct memory effects

The function directly:

- Reads the shared parameter word on the non-null acquisition path.
- Reads the shared header word only after delegated success.
- Writes the first four bytes of the acquired block only after delegated success.
- Writes the caller’s output slot after either processing outcome.
- Leaves the output slot without a direct write when acquisition returns null.

It performs no visible direct write to either shared word.

### Dependency effects and ownership

All three dependencies may have additional effects not described by the available evidence. They must not be assumed to preserve shared state, avoid retaining pointers, or obey ownership rules merely from their placement in this function.

On success, the block pointer is exposed to the caller and this function makes no cleanup call. This supports an inferred intent of handing the block to the caller, but does not establish exclusive ownership, the eventual release mechanism, or the block’s lifetime.

On delegated failure, the block is submitted to F-1137792fba7d and is not published as a non-null output. That call does not, by itself, prove that the storage is released or that resources created by F-40a81407b15f are fully reclaimed.

The function does not explicitly release any pointer previously present in the output slot. Overwriting such a pointer may lose the caller’s access to an existing resource.

## Dependencies

| Opaque ID | Visible invocation | Contract boundary |
|---|---|---|
| F-858f5b8d5984 | Called once with 8 before other dependency calls | Returns a pointer-like value; null selects immediate −108 return. Allocation semantics and other effects are unknown. |
| F-40a81407b15f | Called once only after non-null acquisition, with the four arguments specified above | Returns a status tested against zero. Its writes, retained references, input interpretation, and shared-state effects are unknown. |
| F-1137792fba7d | Called once only after nonzero processing status, with the acquired block base | Cleanup-like role is inferred. Release behavior, other effects, and exceptional behavior are unknown. |

The supplied evidence establishes no retry, fallback acquisition, or second processing attempt.

## Acceptance examples

These examples are conditional static acceptance criteria, not reported executions.

### Acquisition returns null

Given that F-858f5b8d5984 returns null and returns normally:

- The function returns −108.
- Neither other dependency is called.
- The function does not directly overwrite an existing output-slot value.

This case must not be interpreted as guaranteeing a null output.

### Processing succeeds

Given valid, nonoverlapping storage, acquisition of a suitable block, and a zero result from F-40a81407b15f:

- The processing dependency receives the opaque inputs, the shared parameter word, and the block-base-plus-four address.
- The block’s first word receives the shared header word read after processing.
- The output slot receives the block pointer.
- Cleanup is not called.
- The function returns zero.

The second word’s contents are not independently guaranteed.

### Processing returns a negative error

Given a processing result of −7 and normal cleanup return:

- Cleanup receives the original block base.
- The output slot is written with zero after cleanup.
- The function returns −7, not −108.

### Processing returns a positive status

Given a processing result of 3 and normal cleanup return, the behavior is still failure handling: cleanup, null output publication, and return of 3. Positive status is not treated as success.

### Shared state changes during dependencies

If acquisition changes the shared parameter word before returning a block, the processing call uses the subsequently read value.

If processing changes the shared header word before returning zero, the success-path header assignment uses the subsequently read value.

Neither example assumes that the dependencies normally make such changes.

### Output overlaps the header

If the output location is the acquired block’s base and all accesses are otherwise valid, successful processing is followed first by a header store and then by a pointer store to the same location. Acceptance must account for that final overwrite rather than requiring the header value to remain there.

## Uncertainties

The following remain **documented_only**:

- Exact pointer and integer ABI beyond the supplied four-byte representation.
- Alignment and byte-order requirements.
- Meaning of either opaque input or shared word.
- Symbolic interpretation of −108.
- Whether acquisition returns initialized, uniquely owned, or reclaimable storage.
- Whether processing initializes the second word, accesses other memory, or retains the supplied address.
- Whether cleanup releases storage or preserves any shared state.
- Ownership and destruction rules for a successful result.
- Thread safety, synchronization, and concurrent access rules.
- Behavior after invalid memory accesses, dependency non-return, or abnormal control transfer.

Allocator-like, initializer-like, header-like, and cleanup-like interpretations are inferred intent, not additional guarantees.

## Evidence

The supplied static body establishes the fixed acquisition argument of eight, null-acquisition return of −108, four-argument processing call, four-byte interior address, exact zero/nonzero status split, success-only first-word assignment, failure-only cleanup call, and final output publication after either processing outcome.

It also establishes that acquisition failure bypasses the output store and that shared header consumption occurs after successful processing.

No dependency implementation, runtime trace, allocator specification, ownership contract, or concurrency model was supplied. No runtime tests have been performed.
