<!-- Trusted provenance: F-558efbcbde2c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-558efbcbde2c

## Purpose

F-558efbcbde2c performs an operation through a temporary context, then conditionally evaluates that context against two ordered reference values obtained from shared state. It returns one of three 32-bit status values: 0, 1, or 2.

An optional caller-supplied byte is cleared before any dependency call. If the initial operation succeeds and both reference evaluations return nonzero, the function explicitly sets that byte to 1. This final byte write distinguishes that path from an initial-operation failure, although both paths return 2.

The interpretation of the reference values and the meaning of a zero evaluation result are not established. Describing the function as a parser, classifier, matcher, or validator would be inferred intent, not an established contract.

## Interface

The function takes four arguments:

| Argument | Established representation | Use |
|---|---|---|
| First input | 32-bit opaque value | Passed unchanged to F-2fb582eca75a. |
| Second input | 32-bit opaque value | Passed unchanged to F-2fb582eca75a, after a fixed argument of 8. |
| Optional byte address | Pointer to a one-byte object, or null | Stored in the temporary context and used directly for conditional byte writes. |
| Context seed | 32-bit opaque value | Stored in the second slot of the temporary context before initialization. |

The return value is a 32-bit quantity. On ordinary completion, its value is exactly 0, 1, or 2. No signed interpretation is needed to distinguish these outcomes.

The supplied representation places the context’s pointer slot first and its 32-bit seed slot immediately after it. The depicted layout is consistent with a 32-bit pointer at byte offset 0 and a 32-bit word at byte offset 4, for an eight-byte initial context. The pointer width and exact stack-object boundary are inferred from that representation rather than independently verified ABI metadata.

The context is passed by address to all four dependencies. It is temporary storage associated with this invocation. Its initial contents are specified here; its contents after dependency calls are not.

### Preconditions

For defined execution along the relevant paths:

- A non-null optional byte address must support the initial one-byte write.
- On the path that performs the final explicit byte write, that original address must still designate writable storage.
- Arguments and context contents must satisfy the undocumented requirements of each invoked dependency.
- If the initial operation returns nonzero, the shared-state addresses used for reference lookup must permit the required 32-bit reads.

There is no explicit validation of these conditions. In particular, the function does not check a non-null byte address for validity, and it does not check reference values before passing them onward.

## Behavior

### Initial byte write and context setup

If the optional byte address is non-null, the function first writes the one-byte value 0 through it. This precedes every dependency call.

The function then establishes a tentative return value of 2 and initializes the temporary context:

- The first slot receives the original optional byte address, including null when supplied.
- The second slot receives the original 32-bit context seed.

It calls F-5b24edfb22c0 with the address of that context. The caller does not inspect a return value from this dependency.

The initialization call may modify the context or other state. The wrapper does not restore the original context contents afterward.

### Initial operation

Next, the function calls F-2fb582eca75a with, in order:

1. The original first input.
2. The fixed value 8.
3. The original second input.
4. The temporary context’s address.

The result is tested only for equality with zero. Any nonzero result takes the same branch; there is no special treatment for 1, negative values, or other nonzero representations.

After this call returns, the function reads a shared base value into saved state. In the supplied representation, this read occurs before the branch on the operation result, so it also occurs when that result is zero.

If the operation result is zero:

- Neither reference value is read by the conditional evaluation block.
- F-3805a25164a0 is not called.
- The tentative return value remains 2.
- The function proceeds to finalization.

The initial byte write still occurred when the optional byte address was non-null. Dependencies may nevertheless have changed that byte or other state.

### First reference evaluation

If the initial operation returned nonzero, the function obtains the first reference value by reading a 32-bit word at byte offset 0x40, or 64 bytes, from the shared base selected for this lookup.

The presented expression consults the shared base again for this first lookup rather than explicitly using only the previously saved copy. There is no intervening dependency call between the saved-base read and this lookup. Equality of those two base observations is therefore expected under stable shared state, but is not established in the presence of concurrent or asynchronous modification.

The function calls F-3805a25164a0 with:

1. The temporary context’s address.
2. The first 32-bit reference value.

If this evaluation returns zero:

- The return value becomes 1.
- The second reference value is not read.
- No second evaluation occurs.
- The wrapper does not perform its explicit byte write of 1.
- Execution proceeds to finalization.

A nonzero result continues to the second evaluation.

### Second reference evaluation

The second reference value is a 32-bit word read at byte offset 0x44, or 68 bytes, from the base saved immediately after the initial operation.

This distinction matters: the first evaluation may modify shared state, but replacing the shared base during that call does not redirect the second lookup to the new base. The second lookup uses the saved base. However, the reference word at that saved base is read only after the first evaluation returns; it is not an earlier snapshot of that word.

The function calls F-3805a25164a0 again with:

1. The temporary context’s address.
2. The second 32-bit reference value.

The second evaluation receives the context as it exists after initialization, the initial operation, and the first evaluation. No context reset occurs between these calls.

If the second evaluation returns zero, the return value becomes 0.

If the second evaluation returns nonzero, the return value remains 2. In that case, and only if the original optional byte address is non-null, the wrapper writes the one-byte value 1 through that original address.

The final explicit byte write does not obtain its destination from the context. If a dependency changes the pointer stored in the context, that change does not redirect this write.

### Finalization and return

On every ordinary path described above, the function calls F-289ec4311c64 exactly once with the temporary context’s address.

Finalization occurs after all branch-dependent reference reads and after any explicit byte write of 1. The selected status is returned after finalization returns.

No finalizer result is inspected. There is no subsequent wrapper write to the optional byte. Consequently, the finalizer can affect the caller-visible byte or shared state if its own behavior permits that; the wrapper does not restore either afterward.

No exceptional cleanup mechanism is shown. A dependency that does not return, or an invalid memory access, prevents the ordinary completion sequence from being established.

## Side effects

The wrapper directly performs these externally relevant memory accesses:

- An initial one-byte write of 0 through the optional byte address, when non-null.
- A read of the shared base after the initial-operation call, including when that call returns zero.
- On initial-operation success, a further use of the shared base to obtain a 32-bit word at offset 64.
- If the first evaluation returns nonzero, a 32-bit read at offset 68 from the saved base.
- If both evaluations return nonzero, a one-byte write of 1 through the original optional byte address, when non-null.

The reference offsets are byte displacements, not element indexes. The reference values themselves are 32 bits wide. The wrapper performs no arithmetic on those values and does not normalize them before passing them to F-3805a25164a0.

All dependency calls may have additional effects not specified by the available evidence. In particular, they are not assumed to preserve shared globals, context contents, or the optional byte.

The wrapper contains no explicit allocation or direct release of caller memory. Calling a finalizer-like dependency does not establish that it owns or frees anything. Ownership transfer, reference-count changes, resource acquisition, and resource release inside dependencies remain unknown.

Aliasing can affect behavior. For example, if the optional byte address overlaps storage later read as a reference value, the initial byte write may change that value before lookup. The wrapper provides no overlap check.

## Dependencies

| Opaque function ID | Invocation and role supported by evidence |
|---|---|
| F-5b24edfb22c0 | Called once with the initialized temporary context before the initial operation. Its result is not inspected. Its initialization or preparation role is inferred from ordering. |
| F-2fb582eca75a | Called once with the first input, fixed value 8, second input, and context address. Zero skips both reference evaluations; nonzero enables them. |
| F-3805a25164a0 | Called zero, one, or two times with the context and a 32-bit reference value. Each result is distinguished only as zero or nonzero. |
| F-289ec4311c64 | Called once on every ordinary completion path, after branch-specific work. Its cleanup or finalization role is inferred from ordering. |

The same context address is supplied throughout. No independent copy is constructed for either evaluation.

The dependencies’ internal contracts are unavailable. Their names here identify call targets, not semantic guarantees.

## Acceptance examples

These examples are conditional on valid memory, dependency calls returning normally, and the stated dependency outcomes. Byte-value expectations additionally require that dependencies do not overwrite the byte after the wrapper’s relevant write.

| Initial-operation result | First evaluation | Second evaluation | Returned status | Wrapper’s explicit byte writes for a non-null address |
|---|---|---|---|---|
| Zero | Not called | Not called | 2 | 0 at entry only |
| Nonzero | Zero | Not called | 1 | 0 at entry only |
| Nonzero | Nonzero | Zero | 0 | 0 at entry only |
| Nonzero | Nonzero | Nonzero | 2 | 0 at entry, then 1 before finalization |

Additional acceptance conditions:

- **Null optional address:** Neither explicit byte write occurs. Initialization, the initial operation, eligible evaluations, and finalization still occur. Null is stored in the context; whether dependencies accept it remains undocumented.
- **First evaluation returns zero:** The second reference word is not read, even if reading it would fail. The returned status is 1 after ordinary finalization.
- **Initial operation returns zero:** Both reference-word reads are skipped, but the post-operation shared-base read shown in the supplied representation still occurs.
- **Nonzero values other than 1:** They follow the nonzero branches. The wrapper does not require a canonical Boolean result.
- **First evaluation changes the shared base:** The second lookup still uses the saved earlier base. This does not imply that the storage at that base remains valid.
- **First evaluation changes the second reference word:** If the saved base remains valid, the second lookup observes the word when that lookup occurs, not a value captured before the first evaluation.
- **A dependency changes the context’s pointer slot:** The wrapper’s final explicit write, if reached, still targets the original optional byte address.
- **Finalization changes the output byte:** The byte visible after return reflects that later change. The wrapper’s write of 1 is an ordered event, not an unconditional post-return value guarantee.
- **Both evaluations return zero-capable outcomes:** A zero first result takes precedence because the second evaluation is never reached.

These are contract examples derived from static evidence, not reports of executed tests.

## Uncertainties

The following are **documented_only**:

- The semantic meanings of the first input, second input, context seed, fixed value 8, and reference values.
- Whether zero from F-3805a25164a0 means equality, success, absence, or another condition.
- The intended names or domain meanings of return statuses 0, 1, and 2.
- Exact ABI details beyond the depicted widths and inferred context layout.
- Alignment requirements for the context and reference-word reads.
- Validity guarantees, lifetime, synchronization, and mutability of the shared base and its referenced storage.
- Whether dependencies retain the context address or any supplied pointer. The wrapper supplies temporary storage but establishes no safe retention lifetime after return.
- Whether dependencies allocate, release, transfer ownership, or modify unrelated state.
- Dependency error handling, exceptions, nonlocal exits, and behavior on invalid inputs.
- Whether optimized execution or platform memory rules alter externally observable access behavior relative to the supplied representation.

The initial byte clear and later conditional byte set suggest a supplementary outcome indicator, but that intent is inferred. The evidence establishes the writes and their ordering, not a broader meaning.

## Evidence

The contract is based solely on the supplied static function representation. Established evidence includes the four-argument interface, the 32-bit return and opaque word values, the byte-sized stores, dependency-call order, zero-versus-nonzero branch conditions, reference offsets 0x40 and 0x44, use of a saved base for the second lookup, and finalization on every ordinary return path.

The context’s eight-byte layout and the preparation/finalization descriptions are identified as inferences where applicable. No dependency implementation, external API documentation, ABI confirmation, or runtime observations were provided.

No runtime tests have been performed.
