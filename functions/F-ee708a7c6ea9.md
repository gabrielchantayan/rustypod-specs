<!-- Trusted provenance: F-ee708a7c6ea9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ee708a7c6ea9

## Purpose

F-ee708a7c6ea9 performs an ordered, two-stage indirect dispatch using an input record and a dispatch-base value captured from a related object.

The first stage is optional: a callback selected through the input record is invoked only when its tested slot is nonzero. The second stage is unconditional on ordinary completion of the first stage: a target is selected through the previously captured dispatch base and invoked without an explicit argument in the supplied representation.

The established behavior is dispatch, not a particular lifecycle operation. The evidence does not establish that this function destroys an object, releases storage, finalizes a resource, or transfers ownership.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Input

The function accepts one pointer to a record. The following names describe roles in this contract, not recovered type identities:

| Role | Location | Interpretation |
|---|---|---|
| Input record | Argument value | Address used to access the initial object reference and the optional-callback selector |
| Related-object reference | Byte offset 0 in the input record | Address of an object containing the dispatch-base field |
| Optional-callback selector | Byte offset 12 in the input record | Address of a structure containing the optional callback slot |
| Captured dispatch base | Byte offset 100 in the related object | Address used later to locate the final target |
| Optional callback slot | Byte offset 8 in the selector structure | Nullable callable target |
| Final target slot | Byte offset 8 in the captured dispatch structure | Callable target, with no null check shown |

These offsets describe the supplied 32-bit representation: integer-indexed record elements occupy four bytes, address-bearing fields occupy four bytes, and the fourth record element is therefore at byte offset 12. A related-object offset of 100 bytes is decimal, equivalent to hexadecimal 0x64.

The source-level integer spelling alone does not establish a portable language ABI. These layouts must not be generalized to a platform with different integer or pointer widths.

### Output

No result value is exposed. Values returned by either indirect target are not consumed by the shown function.

A normal return is possible only if all preceding accesses and invoked targets complete in a way that allows control to return through this function. Faults, exceptions, nonlocal control transfers, and nonreturning callees are not handled in the supplied body.

### Preconditions for defined ordinary execution

At entry:

- The input record must be nonnull and readable at the required fields.
- Its related-object reference must identify storage readable at byte offset 100.
- Its optional-callback selector must identify storage readable at byte offset 8, even when the callback stored there is null.
- The captured dispatch base must ultimately permit a readable final-target slot at byte offset 8.
- Each target actually invoked must be callable under the applicable machine-level calling convention.

The late null comparison does not make null input supported. The input has already been dereferenced before that comparison.

The evidence supplies no length parameters, bounds validation, alignment checks, or overflow handling. Address formation and access must be valid under the actual target environment; this contract does not introduce wrapping-address semantics.

## Behavior

### Capture the final-dispatch base first

The function first reads the related-object reference from the beginning of the input record. It then reads the four-byte value at byte offset 100 in that related object.

That second value becomes the captured dispatch base for the final stage.

This capture precedes any optional callback invocation. Consequently, the optional callback cannot redirect the final stage merely by replacing the related-object reference or changing the related object’s field at offset 100. The final stage uses the earlier captured value rather than rereading either field.

The capture does **not** capture the final callable target. That target is read later from the captured base.

### Inspect the optional callback

After capturing the dispatch base, the function reads the optional-callback selector from byte offset 12 in the input record and examines the callable slot at byte offset 8 in the selected structure.

If the examined slot is zero, no optional callback is invoked.

If it is nonzero, the function invokes the callback with the original input-record pointer as its one explicit argument.

The supplied representation expresses the slot lookup again for the invocation rather than explicitly saving the tested callable target. It therefore does not establish a guarantee that a target tested as nonzero is independently snapshotted and reused. In ordinary stable memory, the tested target and invoked target coincide. Exact machine-load multiplicity, concurrent mutation, and volatile-memory behavior are not established by this representation.

No callback result is inspected.

### Apply the late null comparison

Following the optional-callback stage, the function compares the input-record pointer against zero. A zero value causes an immediate return instead of final dispatch.

This comparison occurs too late to protect the initial accesses. For ordinary execution with a valid nonnull input, the comparison does not suppress final dispatch.

Passing the pointer to the optional callback does not, by itself, permit that callback to replace the caller’s argument value. The callback can potentially change reachable record contents, but that is different from assigning a new value to this invocation’s pointer argument. Stack corruption or other out-of-contract interference is not a supported mechanism for making this comparison meaningful.

### Resolve and invoke the final target

If execution proceeds beyond the late comparison, the function reads the callable target at byte offset 8 relative to the captured dispatch base and invokes it.

There is no shown null check for either the captured base or the target loaded from it. A null optional callback is therefore supported by the branch, but a null final target is not given a corresponding skip behavior.

The final target is resolved after the optional callback returns. If the optional callback changes the target slot within the already captured dispatch structure, the later lookup can observe that change. This differs from replacing the related object’s dispatch-base field, which is not reread.

The supplied representation shows no explicit arguments for the final invocation. It also warns that this transfer was represented as a call despite uncertainty about an indirect jump. Therefore, the established interface is limited: no source-level argument expression is shown. The exact register inputs, calling convention, stack behavior, and whether the transfer is a tail dispatch remain unresolved.

## Side effects

### Direct effects

The shown body performs memory reads and indirect control transfers. It contains no explicit writes to the input record, related object, callback structure, dispatch structure, or global storage.

There is no explicit allocation, deallocation, reference-count update, lock operation, or ownership transfer.

### Effects through invoked targets

Either invoked target may have effects not described by this body. The evidence does not establish that either target:

- Preserves global state.
- Leaves the input record or related storage unchanged.
- Preserves the final dispatch structure.
- Retains, transfers, or releases ownership.
- Returns normally.
- Avoids reentrancy.

The optional callback receives the input-record pointer explicitly. Its access to other state depends on its implementation and execution environment.

The final target’s accessible state cannot be bounded from the empty explicit argument list. Ambient registers, globals, and calling-convention details remain unknown.

### Lifetime requirements

Readable storage must remain valid until its last required access. In particular, retaining the dispatch-base address does not retain ownership of the addressed storage.

If the optional callback releases or invalidates the captured dispatch structure, the final target lookup has no shown protection against that invalidation. The evidence does not establish that such release occurs, only that the function provides no visible lifetime safeguard.

## Dependencies

There are two dynamically selected dependency roles:

| Dependency role | Selection | Explicit input |
|---|---|---|
| Optional callback | Slot at byte offset 8 in the structure selected from input offset 12 | Input-record pointer |
| Final dispatch target | Slot at byte offset 8 in the captured dispatch structure | None shown |

Neither target has a resolved opaque function ID in the supplied dependency map. They remain unresolved indirect dependencies; assigning them fabricated IDs or attributing particular implementations would add unsupported provenance.

F-ee708a7c6ea9 is the only resolved function identity supplied.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests.

### Null optional callback

Given valid input, related-object storage, selector storage, captured dispatch storage, and final target:

- If the optional slot is zero, the optional callback is skipped.
- The dispatch base is still captured before that slot is inspected.
- The final target is still resolved and invoked.

A zero optional slot does not suppress final dispatch.

### Nonnull optional callback

Given stable valid structures and a nonzero optional callback:

- The dispatch base is captured first.
- The optional callback receives the original input-record pointer.
- If that callback returns normally, the final target is subsequently resolved through the captured base and invoked.

Any callback return value has no shown influence on this sequence.

### Callback replaces the dispatch-base field

Suppose the initially captured base identifies structure A. During its invocation, the optional callback changes the related object’s offset-100 field to identify structure B.

Provided structure A remains valid, final dispatch still uses the target slot in A. The updated field pointing to B is not reread.

### Callback changes the captured structure’s target

Suppose the captured base identifies structure A, and the optional callback replaces A’s target at byte offset 8.

Provided the update is visible and the new target is valid, the final lookup uses the updated slot. Capturing A’s address does not freeze its contents.

### Null input or invalid intermediate storage

A null input is not accepted as a harmless no-op: the first record access precedes the null comparison.

Likewise, a null selector does not become acceptable merely because an absent optional callback was intended. The selector’s slot must be accessed to determine whether that callback is absent.

### Nonreturning optional callback

If the optional callback does not return, this invocation does not reach the later null comparison or final target lookup through the shown path.

No fallback final dispatch is established.

## Uncertainties

The following items are **documented_only**:

- Exact machine-level ABI, pointer alignment requirements, and platform behavior for invalid accesses.
- Exact number of machine reads used to test and invoke the optional callback.
- Concurrency guarantees and behavior under unsynchronized mutation.
- Final-transfer calling convention, register arguments, and call-versus-tail-jump mechanics.
- The identities, semantics, ownership rules, and global-state effects of both indirect targets.
- Whether the late null comparison reflects meaningful original behavior or a representation artifact.
- Any domain-specific purpose such as destruction, cleanup, or resource release.

The repeated dispatch patterns suggest staged processing, but that is inferred intent rather than an established lifecycle contract.

## Evidence

The supplied static representation establishes the ordered dispatch-base capture, nullable optional-callback branch, explicit callback argument, late input-pointer comparison, and final indirect transfer through the captured base.

It also supplies a warning that recovery of the final indirect control transfer was incomplete. That warning limits claims about the final target’s interface and control-flow mechanics; it does not justify assigning a specific target or lifecycle meaning.

No runtime observations, execution traces, target implementations, or external ABI documentation were supplied. All acceptance examples above are conditional consequences of the static evidence, not claims of verified execution.
