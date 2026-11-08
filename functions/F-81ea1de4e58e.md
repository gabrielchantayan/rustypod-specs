<!-- Trusted provenance: F-81ea1de4e58e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-81ea1de4e58e

## Purpose

F-81ea1de4e58e processes entries in a supplied container, recursively processes entries marked as nested containers, and resolves entries with one particular accepted classification to objects. When a current object is supplied, the function rejects a matching identity and may pass regions of the resolved and current objects to a relationship dependency.

Processing stops at the first visible rejection or unsuccessful nested result. The function returns a 32-bit status and, on paths admitted by the container gate, invokes a finalization dependency before returning.

“Container,” “identity,” “current object,” “activation,” and “relationship” describe apparent roles rather than established application semantics. This contract distinguishes visible control flow from unknown dependency behavior. In particular, an earlier dependency call is not a guarantee that its effects persist through later calls.

## Interface

### Arguments

| Position | Width | Visible role |
|---|---:|---|
| First | 32 bits | Opaque context passed to resolution and recursive invocations. |
| Second | 32 bits | Optional current-object address. Zero disables current-object identity comparison and relationship processing. |
| Third | 32 bits | Opaque container value supplied to traversal dependencies. |
| Fourth | 32 bits | Depth bit pattern, checked and incremented using unsigned 32-bit arithmetic. |

The function does not validate object addresses, container validity, dependency-produced addresses, or the initialization of dependency outputs.

Defined execution requires readable storage for every field actually accessed and valid arguments for every dependency actually called. The supplied representation uses 32-bit addresses and values; it does not establish behavior for invalid addresses or address calculations outside valid object storage.

### Return value

| Status | Visible triggering conditions |
|---:|---|
| 1 | Traversal completes without a rejecting condition, or the container gate returns nonzero and traversal is skipped. |
| 2 | An entry read returns nonzero, or an ordinary entry fails the classification-mask test. |
| 3 | Object resolution returns zero. |
| 4 | Depth admission fails, a converted identity matches the current object, or the first state read after activation observes 6. |
| 5 | The first post-activation state read is not 6 and the second state read is not 1. |

A recursive result other than 1 becomes the parent’s result and stops its traversal. These numbers are established return values, not established public error names.

The return and finalization rules below assume invoked dependencies return normally. No exception, fault, or other nonlocal-exit handling is visible.

### Relevant layouts

Offsets are byte offsets from the indicated base.

**Entry descriptor**

| Offset | Width | Use |
|---|---:|---|
| `+0` | 32 bits | Entry value used for the sentinel check and classification. |
| `+4` | 32 bits | Present in the supplied storage representation; not directly interpreted here. |
| `+8` | 8 bits | Zero selects ordinary processing; nonzero selects nested traversal. |
| `+12` | 32 bits | Nested-container value, used only on the nested branch. |

The descriptor is temporary storage supplied to F-21c1171f3c98. Dependencies receiving it may inspect contents beyond the fields directly interpreted by this function.

**Converted identity**

| Offset | Width | Use |
|---|---:|---|
| `+0` | 32 bits | First identity component. |
| `+4` | 32 bits | Second identity component. |

F-145ff188377a receives an output address for this storage. No conversion status is checked. Both components must be initialized for paths that subsequently use them.

**Current object**

- `+0x30`: first 32-bit identity component.
- `+0x34`: second 32-bit identity component.
- `+0x1d8`: beginning of a region whose address is passed to F-6caf95f9c95b.

**Resolved object**

- `+0x48`: beginning of a region whose address is passed to F-6caf95f9c95b.
- `+0x1ac`: flag byte; only bit 0 is tested.
- `+0x1ae`: state byte, inspected only when flag bit 0 was set.

The region offsets identify addresses within the objects, not pointer fields loaded from those offsets. Full object sizes and layouts remain unknown.

## Behavior

### 1. Depth admission

Before invoking any dependency, the function adds one to the depth using unsigned 32-bit arithmetic. Execution is admitted only if that result is less than 21.

The admitted input bit patterns are exactly:

- `0xffffffff`, whose increment wraps to zero;
- `0x00000000` through `0x00000013`, representing 0 through 19.

Every other depth bit pattern returns 4 without dependency calls.

This is not equivalent to a signed comparison against 20. For example, the bit pattern representing signed −1 is admitted, while the bit pattern representing signed −2 is rejected.

Nested traversal receives the parent depth incremented by one with the same arithmetic. An admitted parent at depth 19 therefore invokes a nested child at depth 20, which immediately returns 4.

### 2. Container gate and captured count

After depth admission, the provisional result is 1. F-f3fb3c9e6fe1 receives the container.

If the gate returns nonzero, the function returns 1 without querying the count, reading entries, or calling the finalizer.

If the gate returns zero, F-3b3cb2970a08 supplies an unsigned 32-bit entry count. This count is obtained once per invocation. The function visits indices in ascending unsigned order, beginning at zero and stopping before the captured count.

A zero count causes no entry reads but still leads to finalization.

For each visited index, F-21c1171f3c98 receives the container, index, and descriptor output address. A nonzero result selects status 2 and stops traversal without inspecting the descriptor for that entry.

The function contains no explicit rollback operation for entries already processed. This establishes only its own control flow: subsequent dependencies, including the finalizer, may reverse, discard, or overwrite earlier effects.

### 3. Nested entries

After a successful entry read, the descriptor discriminator at `+8` is inspected.

A nonzero discriminator selects a recursive invocation with:

- the same context;
- the same current-object address;
- the nested-container value at descriptor offset `+12`;
- the incremented depth.

The first descriptor word is not classified on this branch.

A child result of 1 permits the parent to continue. Any other child result stops the parent and is returned after the parent’s applicable finalization.

Each child independently applies depth admission, its own container gate, and its own finalization rules.

### 4. Ordinary entries and classification

A zero discriminator selects ordinary processing.

If the descriptor’s first word is `0x3b`, the entry is accepted without classification, identity conversion, resolution, flag inspection, or relationship processing.

Every other ordinary entry is passed to a classification dependency whose opaque ID was not supplied. Its result is interpreted as a 64-bit value.

Acceptance is controlled by two external 32-bit masks:

1. The classification’s upper 32 bits are intersected with the upper-word mask.
2. Only if that intersection is zero are the lower 32 bits intersected with the lower-word mask.
3. If both intersections are zero, processing stops with status 2.

The lower-mask access is short-circuited when the upper intersection is nonzero. The mask values are not supplied, so the full accepted classification set cannot be enumerated.

An accepted classification other than `0x0000100000000000` completes processing of that entry without identity conversion or resolution.

The exact classification `0x0000100000000000` selects identity processing, but only after passing the mask test. Its upper word is `0x00001000` and its lower word is zero. Consequently, it reaches identity processing only if the upper-mask read includes bit `0x00001000`.

### 5. Identity conversion and comparison

For the special accepted classification, F-145ff188377a receives the descriptor address and identity output address. Its visible conversion return status, if any, is not checked.

When the current-object address is nonzero:

1. The current object’s second identity component is read at `+0x34`.
2. That value is compared with the converted second component.
3. Only if they match is the current object’s first component read at `+0x30` and compared with the converted first component.

If both components match, processing stops with status 4. Resolution is not called for that entry.

A mismatch in the second component skips the current object’s first-component read. Either mismatch permits resolution.

When the current-object address is zero, neither current-object identity field is read.

### 6. Resolution

F-fed763d460a8 receives four 32-bit arguments, in order:

1. Context.
2. An auxiliary value.
3. Converted first identity component.
4. Converted second identity component.

With a nonzero current object, the auxiliary argument is the second identity component already read from current-object offset `+0x34`.

With a zero current object, the auxiliary argument comes from an additional register value exposed after F-145ff188377a. Its semantic meaning and ABI guarantee are unresolved. The available evidence does not justify substituting zero, the context, or either converted identity component.

A zero resolution result selects status 3 and stops traversal. A nonzero result is treated as a resolved-object address.

### 7. Flag, activation, and state handling

The resolved object’s flag byte at `+0x1ac` is read.

If bit 0 is clear, F-eddc89ee2a0a is skipped and no state byte is inspected.

If bit 0 is set:

1. F-eddc89ee2a0a receives the resolved-object address and zero.
2. After it returns, the state byte at `+0x1ae` is read and compared with 6.
3. If that read observes 6, status 4 is selected.
4. Otherwise, the state byte is read again and compared with 1.
5. If the second read observes anything other than 1, status 5 is selected.

There is no pre-call state check. The dependency’s visible return value is unused.

The supplied representation contains two distinct state reads. It does not establish an immutable snapshot. For example, first observing 2 and then 1 passes these checks, while first observing 1 and then 6 selects status 5, not status 4. Whether such changes can occur in a supported execution environment is unknown.

### 8. Relationship call

If resolution and state handling have not rejected the entry, and the current-object address is nonzero, F-6caf95f9c95b receives:

1. Resolved-object address plus `0x48`.
2. Current-object address plus `0x1d8`.

Its visible return value is ignored. The operation performed on those regions, mutation direction, duplicate handling, and failure reporting are unknown.

A zero current-object address skips this call but does not skip resolution or flag/state handling.

### 9. Finalization and return

Every invocation whose container gate returned zero calls F-e7fa8e2c3df9 with its container after traversal stops, before returning the selected status.

This includes an empty container, successful traversal, entry-read failure, classification rejection, identity rejection, resolution failure, state rejection, and a non-1 child result.

When a child fails, its applicable finalization precedes its return; the parent’s applicable finalization follows that return.

Depth rejection and a nonzero container-gate result skip finalization. The finalizer’s visible return value is ignored.

Finalization’s effects are unknown. Its placement does not establish that earlier changes are committed, retained, undone, or released.

## Side effects

Directly visible activity consists of dependency calls, output storage supplied to dependencies, and reads of descriptor fields, identity components, object fields, and external classification masks. The function does not visibly assign object fields itself.

Dependency calls may modify objects, containers, globals, ownership state, or other storage. No general preservation guarantee is established.

Important ordering guarantees are:

- Entry reading precedes inspection of that entry.
- Classification precedes identity conversion.
- Conversion and any current-object comparison precede resolution.
- The flag read precedes activation.
- Activation precedes the state reads.
- Successful state handling precedes any relationship call.
- A relationship call for an earlier entry precedes a later entry read.
- Applicable finalization follows the stopping condition.

These are call-order and access-order guarantees, not persistence guarantees. Later calls—including entry reads, recursive processing, relationship calls, and finalization—may reverse, discard, or overwrite effects of earlier calls. Persistence across those calls is documented_only.

The masks are consulted during individual entry processing rather than captured once for the invocation. Dependency behavior may therefore affect later mask observations.

Descriptor and identity output addresses refer to temporary storage. Whether dependencies retain these addresses is unknown; safe retention would require a compatible lifetime contract not established here.

No allocation, ownership transfer, reference-count adjustment, or release is established by the gate/finalizer pairing alone. In particular, the finalizer must not be assumed either to destroy the container or to preserve it.

## Dependencies

| Opaque ID | Visible interface or role |
|---|---|
| F-f3fb3c9e6fe1 | Container gate. Zero enables count retrieval, traversal, and eventual finalization; nonzero produces status 1 without those steps. |
| F-3b3cb2970a08 | Receives the container and supplies the unsigned 32-bit count. |
| F-21c1171f3c98 | Receives container, unsigned index, and descriptor output address. Zero permits descriptor inspection; nonzero selects status 2. |
| F-145ff188377a | Receives descriptor and identity output addresses. No conversion status is checked. An additional post-call register value supplies the resolution auxiliary argument when the current object is zero. |
| F-fed763d460a8 | Receives context, auxiliary value, and two identity components. Returns zero or a value used as an object address. |
| F-eddc89ee2a0a | Receives resolved-object address and zero when flag bit 0 is set. State reads follow its return. |
| F-6caf95f9c95b | Receives resolved-object region address followed by current-object region address. Visible return value unused. |
| F-e7fa8e2c3df9 | Receives the container after every normally completed traversal path enabled by a zero gate result. Visible return value unused. |
| F-81ea1de4e58e | Recursive nested traversal. A non-1 result stops the parent. |

The classification dependency has no supplied opaque ID. Its visible boundary is a 32-bit entry value producing a result interpreted as 64 bits. Its implementation and full contract remain documented_only.

## Acceptance examples

These are conditional static expectations, not executed tests. Calls are assumed to return normally, and accessed storage is assumed valid.

1. **Depth rejection:** Depth 20 returns 4 without dependency calls. Depth bit pattern `0xfffffffe` is also rejected.

2. **Wrapped depth:** Depth `0xffffffff` passes admission. A nested child receives depth zero.

3. **Gate bypass:** At admitted depth, a nonzero gate result returns 1 without querying the count or calling the finalizer.

4. **Empty traversal:** A zero gate result and count zero produce one finalizer call and return 1.

5. **Entry-read failure:** A nonzero entry-read result at index zero selects status 2 without descriptor inspection, then finalizes.

6. **Sentinel entry:** A zero discriminator and first word `0x3b` bypass classification and resolution. Traversal continues unless this was the final entry.

7. **Mask rejection:** If both applicable masked classification halves are zero, identity conversion is skipped, the container is finalized, and status 2 is returned.

8. **Special classification blocked:** If classification is `0x0000100000000000` but the upper-mask read lacks bit `0x00001000`, processing returns 2 after finalization. The special value does not override mask rejection.

9. **Identity match:** If conversion produces the current object’s identity, the second current-object component is read before the conditional first-component read. A complete match selects status 4 without resolution.

10. **Resolution failure:** If comparison does not reject the entry and resolution returns zero, finalization occurs and status 3 is returned.

11. **Clear flag:** A successful resolution with flag bit 0 clear skips activation and state reads. Relationship processing occurs only if a current object exists.

12. **State decisions:** A first post-activation state read of 6 selects status 4. A first read other than 6 followed by a second read other than 1 selects status 5.

13. **Nested depth failure:** A parent at depth 19 invokes a nested child at depth 20. The child returns 4 without container access; the parent then finalizes its own container and returns 4.

14. **Earlier call before later failure:** Suppose one entry reaches F-6caf95f9c95b and the next entry read returns nonzero. The relationship call precedes that failed read, which precedes finalization and return of status 2. No explicit rollback operation appears in this function. Whether any earlier relationship effect persists after the failed read or finalization is unknown.

## Uncertainties

### Established static evidence

The supplied representation establishes the status values, unsigned depth rule, captured-count traversal, branch precedence, field offsets, visible dependency arguments, conditional reads, and finalization placement.

It establishes that no explicit rollback operation appears in this function. It does not establish preservation of earlier effects across subsequent dependency calls.

### Inferred intent

Recursive container traversal, identity-based resolution, rejection of a current-object match, object activation, and relationship recording are plausible interpretations. They are not independent guarantees about domain meaning or dependency semantics.

### documented_only

Unresolved matters include:

- Classification-mask values, mutability, and synchronization.
- The classification dependency’s opaque ID and full behavior.
- Meaning and ABI reliability of the null-current-object auxiliary argument.
- Initialization guarantees for descriptor and identity outputs.
- Ownership, allocation, retention, release, and temporary-address lifetime contracts.
- Whether later dependencies preserve, reverse, discard, or overwrite earlier effects.
- Container mutation and index stability relative to the captured count.
- Concurrency, reentrancy, and validity of repeated field reads.
- Exact meanings of the flag, state values, and object regions.
- Dependency failures outside their visible return values.
- Whether dependencies return normally on all relevant paths.

## Evidence

This contract is based solely on the supplied function representation and dependency mapping. No runtime execution or tests have been performed.

Dependency implementations, external mask contents, surrounding type definitions, and ABI documentation were not supplied. Claims requiring that information remain documented_only.
