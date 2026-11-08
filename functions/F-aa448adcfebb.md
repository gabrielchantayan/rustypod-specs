<!-- Trusted provenance: F-aa448adcfebb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa448adcfebb

## Purpose

F-aa448adcfebb coordinates a word-oriented operation: it initializes caller outputs, obtains working storage, invokes a main dependency, writes a result header, and selects among publication, cleanup, and status-reporting paths.

The established contract is orchestration rather than application semantics. The operation argument, payload contents, auxiliary results, and distinguished status class have no established application-level meaning.

Allocation, resizing, cleanup, release, and diagnostic-construction roles are inferred from dependency usage. Their actual ownership and lifetime guarantees are unknown.

## Interface

### Arguments

The function accepts six arguments:

| Position | Role | Established use |
|---|---|---|
| 1 | Context address | Passed to F-f2daf800ed06. Used for a four-byte status write at byte offset 20 on specified completion paths. |
| 2 | Operation word | An opaque four-byte value passed unchanged to F-f2daf800ed06. |
| 3 | Required primary-output address | Cleared on entry. Receives the payload address on ordinary success. |
| 4 | Optional auxiliary-output address A | Cleared when non-null. Receives auxiliary result A on ordinary success. |
| 5 | Optional auxiliary-output address B | Cleared when non-null. Receives auxiliary result B on ordinary success. |
| 6 | Optional diagnostic-slot address | Passed unchanged to F-f2daf800ed06. Read and replaced by this function on a conditional distinguished-status path. |

The required primary-output address is dereferenced without a null check. Defined use therefore requires writable storage for its four-byte entry store.

Each non-null auxiliary-output address must support four-byte writes. The context must support a four-byte write at byte offset 20 whenever execution reaches a context-status update.

The diagnostic slot is not initialized by this function. When its replacement path is reached, it must support a four-byte read and write. Requirements imposed by F-f2daf800ed06 on that argument, including whether null is acceptable, are unknown.

### Return value

The return value is an unsigned 32-bit status:

- Zero after ordinary successful publication.
- The full main-dependency status after ordinary nonzero failure.
- A separate working status after distinguished completion.
- The constant 7 after initial-storage failure or conditional-resize failure.

Distinguished completion can return zero without publishing a payload. A zero return alone therefore does not establish that the primary output received a result.

### Widths and layout

Explicit data stores described here are four bytes wide. The working block begins with a four-byte header. Its payload address is the block base plus four bytes.

The supplied representation uses four-byte address-bearing output slots and supports a 32-bit pointer model. Equivalent behavior with wider pointers is not established.

The header/count-like working value and threshold are represented as signed 32-bit integers. Their comparison is signed. The resize-size expression uses the count-like value plus one, multiplied by four; the source supplies no explicit range checks or unsigned conversion establishing wraparound semantics.

### Aliasing and access requirements

No overlap checks are present. Caller-provided output addresses may overlap each other or context storage, but independent logical outputs are not preserved in that case. Explicit stores occur in the order described below, and later overlapping stores can replace earlier results.

Dependencies may also access aliased storage. Entry clearing is not a guarantee that an output remains zero after a dependency runs.

F-f2daf800ed06 receives the address of the working-block pointer location. It can replace that pointer during its call. Whether it retains access to that location, directly or through other state, is unknown. Later dependencies cannot be assumed to leave the location unchanged.

## Behavior

### Entry initialization

The function first performs caller-visible initialization in this order:

1. Stores zero to the primary output.
2. Stores zero to auxiliary output B, if present.
3. Stores zero to auxiliary output A, if present.

It does not clear the diagnostic slot.

Before initial storage acquisition, it also initializes internal working state:

| Working role | Initial value |
|---|---:|
| Temporary-resource value | 0 |
| Header/count-like value | 1 |
| Threshold | 20 |
| Distinguished-completion status | 0 |
| Auxiliary result A | 0 |
| Auxiliary result B | 0 |
| Additional opaque working word | 0 |

These are usage-based labels, not established application meanings. In particular, the count-like value is not proven to count payload elements.

The supplied body does not explicitly update these values later. Apart from the working-block pointer, it does not explicitly pass their addresses to the main dependency. Their possible modification through external state or other dependency effects is unresolved; no hidden state structure or adjacent-storage convention is established.

### Initial storage acquisition

F-391e3d30e634 is called with the argument 80. Its result is stored as the working-block pointer.

The function then reads an opaque global value, including when the storage result is null.

If the working-block pointer is null:

- F-f2daf800ed06 is not called.
- No payload-cleanup call is made.
- The context status at byte offset 20 is set to 7.
- The function returns 7.

If the pointer is non-null, the function writes a zero header at that block’s base and calls F-f2daf800ed06 with:

- The context address.
- The operation word.
- The opaque global value read after storage acquisition.
- The address of the working-block pointer location.
- The diagnostic-slot address, including null if supplied.

The global argument is the value already read, not a fresh read at the main call.

### Main-operation completion

Immediately after F-f2daf800ed06 returns, the function writes the current header/count-like working value at the base addressed by the then-current working-block pointer.

This header write occurs before any status classification. The replacement pointer is not checked for null or validity. Safe continuation requires it to identify writable header storage at this point, even when the main dependency reports failure.

The main status is classified in this order:

1. If its least significant eight bits equal 4, use distinguished completion.
2. Otherwise, make the ordinary temporary-resource release-like call.
3. If the full status is nonzero, use ordinary failure.
4. Otherwise, proceed toward ordinary successful publication.

Thus, a status of hexadecimal 00000104 takes the distinguished path rather than the ordinary nonzero-failure path.

### Distinguished completion

The function first calls F-9c8ac0a31144 with the payload address obtained from the current working-block pointer plus four bytes.

After that call, handling depends on the temporary-resource value:

- If it is zero, no diagnostic replacement or temporary-resource release-like call is performed on this path.
- If it is nonzero and the diagnostic-slot address is null, F-94cb53877c15 is called with the temporary-resource value.
- If it is nonzero and the diagnostic-slot address is non-null:
  1. The current diagnostic-slot contents are read and passed to F-94cb53877c15.
  2. F-e5083a6dd5b5 is called with a fixed opaque template address and the temporary-resource value.
  3. Its result is written to the diagnostic slot.
  4. F-94cb53877c15 is called with the temporary-resource value.

There is no null-result check before storing the diagnostic-construction result.

Finally, the working distinguished-completion status is written to the context field at byte offset 20 and returned. The original main status is not returned on this path.

Payload cleanup precedes any diagnostic cleanup or replacement. Context-status reporting follows those calls. Dependencies may change accessible state between these operations; preservation of globals or working state is not established.

### Ordinary failure

For a main status whose low byte is not 4, F-94cb53877c15 is called with the temporary-resource value before the full-status zero test. This call occurs even when that value is zero.

If the full main status is nonzero, the function then:

- Calls F-9c8ac0a31144 using the then-current working-block pointer plus four bytes.
- Returns the full main status.

The wrapper performs no explicit context-status write on this path. It does not publish a primary or auxiliary result.

The pointer used for payload cleanup is evaluated after the temporary-resource call. It must not be assumed to remain identical to the pointer used for the earlier header write if a dependency can modify the exposed pointer location.

### Ordinary success and conditional resizing

After a zero main status and the ordinary temporary-resource call, the function compares the signed threshold with the signed header/count-like value.

- If the threshold is less than or equal to the count-like value, resizing is skipped.
- If the threshold is greater, F-0ee6114cab5e is called.

The resize-like call receives the working-block pointer evaluated for that call and a size equal to four times one more than the count-like value. With unchanged initialized values, the comparison is 20 greater than 1 and the requested size is 8.

This is not a conventional “grow when full” condition: values below the threshold request resizing; values at or above it skip resizing.

When resizing is skipped, the current pointer is selected for publication.

When F-0ee6114cab5e returns non-null, its returned pointer is installed as the working-block pointer. That explicit assignment selects the returned address even if dependency activity changed the pointer location during the call.

When F-0ee6114cab5e returns null:

1. The function reads the working-block pointer location after the call.
2. It calls F-9c8ac0a31144 with that then-current pointer plus four bytes.
3. It writes 7 to the context status field.
4. It returns 7 without wrapper publication.

The failed resize result is not installed into the working-block pointer. Nevertheless, the cleanup pointer is not necessarily the pointer passed to F-0ee6114cab5e. The main dependency previously received the address of that location, and retained access or subsequent dependency interference is unresolved.

Only if the working-block pointer remains unchanged across the resize call does failure cleanup address the same block that was passed to it.

No bounds or overflow check precedes the resize arithmetic. Behavior for negative values or signed addition or multiplication outside representable range is not established by the supplied representation. A portable contract must not replace this uncertainty with assumed modulo arithmetic.

### Publication

Once ordinary success selects a usable block, stores occur in this order:

1. The primary output receives the selected block base plus four bytes.
2. Auxiliary output B, if present, receives its working result.
3. Auxiliary output A, if present, receives its working result.
4. The function returns zero.

Publication is sequential, not atomic. The wrapper performs no context-status write and makes no subsequent payload-cleanup call on this path.

## Side effects

### Explicit memory effects

The function directly writes:

- Entry zeros to the primary and present auxiliary outputs.
- A zero block header before the main call.
- The current working header after the main call.
- Successful primary and auxiliary results.
- A diagnostic replacement on the qualifying distinguished path.
- Context status on distinguished completion and storage/resize failure.

It reads the opaque global after initial storage acquisition, even when that acquisition fails. It conditionally reads the diagnostic slot before diagnostic replacement.

The function does not restore entry zeros on failure. Dependencies may change outputs through aliases or other access.

### Resource and ownership effects

The call pattern suggests that F-391e3d30e634 acquires storage, F-0ee6114cab5e resizes it, F-9c8ac0a31144 cleans up a payload-bearing object, and F-94cb53877c15 releases resources. These are inferred roles, not established ownership guarantees.

On ordinary success, the function publishes an interior pointer four bytes beyond the selected base. The caller’s required destruction operation and the published pointer’s lifetime are unknown.

On resize failure, safe cleanup requires the **then-current** working-block pointer to identify storage suitable for F-9c8ac0a31144 after adding four bytes. Two separate unknowns matter:

- Whether the pointer location remains unchanged during the resize call.
- Whether the object identified by the pointer used for cleanup remains valid after resize failure.

The wrapper establishes neither. An assertion that cleanup targets the original resize input is valid only under an explicit unchanged-pointer condition.

Diagnostic replacement passes the previous diagnostic value to a release-like dependency before storing a replacement. It then passes the temporary resource to the same release-like dependency. Whether the replacement copies, retains, or otherwise depends on that resource is unknown.

## Dependencies

| Opaque ID | Observed interaction | Unknown contract |
|---|---|---|
| F-391e3d30e634 | Called with 80; result becomes the initial block pointer. | Allocation units, alignment, ownership, initialization, other effects. |
| F-f2daf800ed06 | Receives context, operation word, captured global value, address of the working-block pointer, and diagnostic-slot address. | Status meanings, pointer replacement rules, retained access, layout, state effects. |
| F-9c8ac0a31144 | Receives a payload address on distinguished completion and selected failures. | Whether it destroys contents, releases the enclosing block, or performs other effects. |
| F-94cb53877c15 | Receives temporary-resource values and conditionally the existing diagnostic value. | Null acceptance, release semantics, ownership, interference. |
| F-e5083a6dd5b5 | Receives a fixed template address and temporary-resource value; its result replaces the diagnostic slot. | Construction semantics, failure representation, allocation, lifetime. |
| F-0ee6114cab5e | Receives a block pointer and calculated size; a non-null result is installed for publication. | Storage preservation, validity on failure, ownership, interference with the exposed pointer location. |

No dependency is established to preserve globals or other accessible state.

## Acceptance examples

These are conditional static expectations, not executed tests. They assume valid memory and dependencies that return normally.

### Initial storage failure

If F-391e3d30e634 returns null:

- Outputs are cleared in primary, B, A order.
- The opaque global is read.
- The main dependency and payload cleanup are not called.
- Context status and return value are both 7.

### Ordinary success with unchanged working values

If initial storage succeeds, the main status is zero, and working values retain their initialized values:

- The post-main header is 1.
- F-94cb53877c15 receives zero.
- Resizing is requested with size 8.
- A non-null resize result becomes the selected base.
- The primary output receives that base plus four bytes.
- Present auxiliary outputs receive zero in B-then-A order.
- The return value is zero, with no explicit wrapper context-status update.

### Ordinary failure

If the main status is 2:

- The header write precedes status handling.
- The temporary-resource call precedes payload cleanup.
- Cleanup uses the working pointer current after that call, plus four bytes.
- The return value is 2.
- The wrapper performs no publication or context-status write.

### Distinguished status

If the main status is hexadecimal 00000104 and the temporary-resource value remains zero after payload cleanup:

- Payload cleanup occurs.
- No diagnostic replacement or temporary-resource release-like call follows.
- The working distinguished status is written to the context and returned.
- If that status remains zero, the function returns zero without publishing a payload.

### Resize failure without pointer interference

If resizing returns null and the working-block pointer remains unchanged across the resize call:

- Cleanup receives the resize-input block address plus four bytes.
- No payload is published by the wrapper.
- Context status and return value are 7.

### Resize failure with pointer interference

If resizing returns null but, during that call, dependency activity changes the exposed working-block pointer to a different valid block:

- Cleanup receives the replacement block address plus four bytes, not necessarily the resize-input address plus four bytes.
- The wrapper does not restore the earlier pointer.
- Context status and return value are 7.

This example is conditional on such interference; it does not establish that any dependency actually performs it.

### Overlapping outputs

On ordinary success, if the primary output and auxiliary output B share an address, the B store overwrites the published pointer. If auxiliary output A shares it too, the A store is last.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose, operation meanings, and payload layout.
- Whether the header represents a count, size, index, or another quantity.
- How working values other than the exposed block pointer might change.
- Whether the main dependency retains access to the working-block pointer location.
- Whether later calls modify that location, globals, or other accessible state.
- Dependency ownership, null handling, storage lifetime, and failure guarantees.
- Valid ranges and overflow behavior for signed resize arithmetic.
- Context layout beyond the four-byte status field.
- Diagnostic construction and resource-retention semantics.
- Behavior after invalid pointers, dependency nonreturn, or storage invalidation.
- The caller’s required release procedure for a published payload.

## Evidence

Established static evidence consists of the supplied body’s access widths, initialization values, call arguments, exposed pointer location, status tests, signed comparison, resize expression, pointer offsets, store ordering, and return paths.

In particular, failed-resize cleanup obtains its payload address from the working-block pointer location after the resize call; the source does not establish that this location still contains the call’s input pointer.

Resource-role labels are usage-based inferences. Dependency implementations and runtime behavior are unavailable.

No runtime tests have been performed.
