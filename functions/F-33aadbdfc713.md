<!-- Trusted provenance: F-33aadbdfc713; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-33aadbdfc713

## Purpose

F-33aadbdfc713 coordinates a mode change on a receiver with conditional traversal of associated objects. Each object supplied by the traversal receives a virtual operation with a normalized mode value of either zero or one.

The distinguishing behavior is the ordering of the receiver-level operation:

- For any nonzero mode input, the receiver-level operation occurs **before** the eligibility check and any traversal.
- For a zero mode input, the receiver-level operation occurs **after** the eligibility check and any traversal.

The eligibility check controls traversal only. On a normally returning execution, it does not suppress the receiver-level operation.

These are static observations from the supplied representation. Interpreting the two modes as activation and deactivation, or interpreting traversal as ownership of child objects, would be inferred intent rather than an established contract.

## Interface

### Inputs

The function has four inputs and no explicit return value.

| Input | Represented width and role |
|---|---|
| Receiver | A 32-bit integer-valued input used as a receiver argument and as the base of an address calculation. |
| Requested mode | A 32-bit integer. Zero selects one path; every nonzero value selects the other. |
| Auxiliary word | An opaque 32-bit value copied into local storage before dependency calls. No direct interpretation is visible. |
| Initial object pointer | A pointer to integer-addressable storage, copied into a local pointer slot. That slot is subsequently passed by address to the traversal dependency. |

The representation is consistent with 32-bit addresses and four-byte machine words. It is not sufficient to establish a portable ABI, calling convention, alignment rule, or signed-overflow policy.

The receiver is not validated locally. It must support the operations required by the dependencies. When traversal is performed, those requirements include use of an address formed by adding the byte displacement `0xA8` to the receiver.

### Outputs

There is no explicit return value and no explicit status or error result.

Observable results can include:

- Effects of the receiver-level operation.
- Effects of the eligibility check and traversal dependencies.
- Virtual operations on objects delivered by traversal.
- Mutations performed by any of those dependencies or virtual operations.

The supplied initial object pointer is passed by value. The function exposes the address of its **local copy** to the traversal dependency; it does not explicitly write a replacement pointer into caller-owned pointer storage.

### Local state and layout

Before making dependency calls, the function establishes local copies of:

1. The requested mode.
2. The auxiliary word.
3. The initial object pointer.

The address of the first copy is used as an opaque traversal-state handle. The address of the pointer copy is separately used as a traversal output location.

The supplied representation places these copies in successive four-byte stack positions. This is relevant static evidence, but it does not establish a public three-field record format or the full memory extent accessed by the dependencies. In particular, the auxiliary word and initial pointer cannot be declared irrelevant merely because their only visible uses are through local storage.

## Behavior

### Mode selection and normalization

The branch is selected by testing whether the requested mode is zero.

The receiver-level operation and every virtual operation receive the normalized value:

- `1` when the requested mode is nonzero.
- `0` when the requested mode is zero.

The original requested mode is also copied into the traversal-state storage. Thus a nonzero request such as `7` is normalized to `1` for the explicit receiver and object operations, but the initial state copy still contains `7`. The supplied representation does not establish whether any dependency reads that original value.

### Nonzero-mode path

For a nonzero requested mode, the observable call ordering on normal return is:

1. Invoke F-58fdff60ff84 with the receiver and `1`.
2. Invoke F-3fac671648b8 with the receiver.
3. If the eligibility result is zero, return without traversal initialization, traversal steps, virtual operations, or traversal finalization.
4. If the eligibility result is nonzero, perform the traversal described below.
5. Return after traversal finalization.

There is no second receiver-level operation after traversal.

The eligibility check occurs after F-58fdff60ff84 returns. Consequently, the check may observe effects of that call. The function does not snapshot receiver contents before making the call.

### Zero-mode path

For a zero requested mode, the observable call ordering on normal return is:

1. Invoke F-3fac671648b8 with the receiver.
2. If the eligibility result is nonzero, perform the traversal described below.
3. Invoke F-58fdff60ff84 with the receiver and `0`.
4. Return.

If the eligibility result is zero, the receiver-level operation still occurs, immediately after the check.

When traversal occurs, its finalization returns before the receiver-level operation begins. Therefore, F-58fdff60ff84 may observe changes made by traversal or by the object virtual operations.

### Traversal initialization

Traversal begins only after a nonzero eligibility result.

F-8ca21d3a8cca receives:

- The address of the local state copy initialized from the requested mode.
- The receiver-derived address at byte displacement `0xA8`.

The function does not explicitly load a pointer from that receiver displacement before calling the dependency. It supplies the computed address itself.

The addition is shown against a 32-bit integer-valued receiver. No explicit range or overflow check appears. For defined behavior, callers must satisfy whatever address-validity and arithmetic requirements apply to the actual target environment and dependency implementation. The supplied representation does not establish wraparound behavior as a supported input case.

No initialization status is tested by this function. If initialization returns normally, traversal steps begin.

### Traversal steps and termination

F-b38a3aba0e18 is called repeatedly with:

- The address of the same local traversal-state storage.
- The address of the local object-pointer slot.

Its result is tested only for zero versus nonzero:

- A zero result terminates the loop without a virtual operation for that step.
- Any nonzero result causes one virtual operation using the object pointer then present in the local slot.

No particular positive value is required. A negative nonzero result is also treated as an instruction to perform the virtual operation.

The function does not explicitly clear the object-pointer slot before each step and does not test whether the dependency wrote a new pointer. Therefore, a nonzero result must leave that slot holding a valid dispatchable object pointer, whether newly written or retained from an earlier value. A zero result may leave any slot value without causing a subsequent object dispatch in this function.

There is no local bound on the number of iterations. If the traversal dependency never returns zero and all calls continue returning normally, the loop does not terminate.

### Per-object virtual operation

For each nonzero traversal result, the function performs indirect dispatch through the current object pointer:

1. Read the pointer currently held in the local object-pointer slot.
2. Read the dispatch-table address from the beginning of the referenced object.
3. Read the callable entry at byte displacement `0xA4` from that table.
4. Invoke that entry with the object pointer and the normalized mode value.

The `0xA4` displacement belongs to the dispatch table. It is not a direct field displacement into the traversed object.

No object-null check, dispatch-table validation, callable-entry validation, or duplicate filtering is visible. A successful traversal step must supply a pointer for which all of these reads and the indirect call are valid.

Each successful step produces one dispatch, including repeated steps that supply the same object pointer. The function neither collects objects in advance nor defers dispatch until traversal completes. Each virtual call returns before the next traversal step is requested.

The virtual operation’s return value, if any, is not inspected.

### Traversal finalization

After a traversal step returns zero, F-d0d3f100effc receives the address of the local traversal-state storage.

Finalization occurs once on the normal traversal-completion path, including when the first traversal step returns zero. It does not occur when the eligibility check rejects traversal.

The supplied representation contains no explicit mechanism that guarantees finalization after an abnormal exit from initialization, a traversal step, or a virtual operation.

## Side effects

### Explicit local effects

The function writes its local copies of the final three inputs before making external calls. It then allows traversal dependencies to access local state and the local object-pointer slot through addresses.

No direct store to receiver fields, traversed-object fields, or global storage is visible in the function itself. This does not imply that execution leaves those locations unchanged: dependencies and virtual operations may mutate them.

### Observable ordering

Dependency calls and virtual operations are interleaved, not batched:

- In nonzero mode, the receiver-level operation precedes every other external call.
- In zero mode, the receiver-level operation follows all normally completed traversal activity.
- Each object operation precedes the next traversal step.
- Traversal finalization follows the terminating step.

The function does not establish that dependencies preserve globals, receiver contents, dispatch tables, collection contents, or other shared state. In particular, the eligibility check is performed once; it is not repeated after object operations change state.

### Ownership and lifetime

No explicit allocation, deallocation, reference acquisition, or reference release is visible.

Calling F-d0d3f100effc establishes a finalization call, not proof that memory is freed. Likewise, no ownership transfer can be assigned to initialization or iteration without their contracts.

The function passes addresses of local storage to dependencies. Those addresses are valid during the enclosing call. Their safe retention or later use is not established.

Receiver and object lifetime requirements across dependency calls are unresolved. The caller and dependencies must collectively ensure that every subsequently used address remains valid.

### Errors and interruption

The function has no explicit error-reporting path.

- A zero eligibility result suppresses traversal.
- A zero traversal-step result terminates traversal.
- Nonzero traversal-step results cause dispatch, without distinguishing success codes from possible error codes.

There are no visible retries, rollback operations, exception handlers, or recovery branches.

If a call fails to return normally, later operations are not guaranteed. For example, interruption during traversal on the zero-mode path can prevent both traversal finalization and the final receiver-level operation.

## Dependencies

| Opaque ID or mechanism | Established use | Not established |
|---|---|---|
| F-58fdff60ff84 | Receives the receiver and normalized mode; called before traversal eligibility in nonzero mode and after traversal in zero mode. | Exact receiver mutation, error semantics, ownership effects, global effects. |
| F-3fac671648b8 | Receives the receiver once; zero suppresses traversal and nonzero permits it. | Meaning of eligibility, purity, stability, or preservation of receiver state. |
| F-8ca21d3a8cca | Initializes or otherwise prepares traversal using local state and the receiver-derived address at `0xA8`. | State extent, collection layout, allocation, ownership, failure behavior. |
| F-b38a3aba0e18 | Receives local state and a pointer-output slot; zero stops iteration, nonzero triggers dispatch. | Object selection order, mutation tolerance, pointer ownership, error-code interpretation. |
| F-d0d3f100effc | Receives local state after normal loop termination. | Whether it destroys state, releases resources, or changes shared state. |
| Dynamic dispatch at table displacement `0xA4` | Receives the current object pointer and normalized mode once per nonzero traversal result. | Concrete target identity, full signature, semantics, lifetime effects, return convention. |

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not runtime test results. Except where stated otherwise, assume valid addresses and normally returning callees.

1. **Nonzero request, traversal rejected.**  
   Given requested mode `9` and a zero eligibility result, the receiver-level operation receives `1` before the eligibility check. No traversal dependency or object virtual operation is invoked.

2. **Zero request, traversal rejected.**  
   Given requested mode `0` and a zero eligibility result, the eligibility check occurs first, followed by the receiver-level operation with `0`. No traversal initialization or finalization occurs.

3. **Eligible but empty traversal.**  
   Given nonzero eligibility and a first traversal-step result of zero, initialization occurs, one traversal step occurs, and finalization occurs. No object dispatch occurs. The receiver-level operation retains its mode-dependent position before or after this sequence.

4. **Two delivered objects.**  
   Given step results of nonzero, nonzero, then zero, with valid objects supplied for the first two steps, exactly two object operations occur. The first operation completes before the second step; the second completes before the terminating step. Finalization follows termination.

5. **Negative requested mode.**  
   Given requested mode `-1`, the nonzero path is selected and explicit mode arguments are `1`. The local state is initially seeded with the original 32-bit requested-mode value, not replaced with `1`.

6. **Negative traversal result.**  
   Given a negative nonzero step result and a valid current object pointer, an object operation occurs. The function does not treat the result as an error solely because it is negative.

7. **Repeated object delivery.**  
   Given two successful steps supplying the same valid object pointer, that object receives two operations. No uniqueness requirement or deduplication is imposed locally.

8. **Zero result with an invalid output pointer.**  
   If a step returns zero while leaving an invalid pointer in the output slot, this function performs no object dispatch through that pointer. This does not establish that finalization is independent of that storage.

9. **Nonzero result without a valid pointer.**  
   If a step returns nonzero while the output slot is null or otherwise invalid, the function has no defensive path. Defined completion is outside the established preconditions.

## Uncertainties

The following remain **documented_only**:

- The domain meaning of either mode.
- The meaning of the receiver-associated region at `0xA8`.
- The complete traversal-state layout and the role of the auxiliary word.
- Whether dependencies read adjacent local storage.
- Ownership, allocation, release, and reference-counting behavior.
- Traversal order, mutation safety, reentrancy, and concurrency requirements.
- Concrete virtual targets and their effects.
- Platform-specific address arithmetic and ABI details.
- Error propagation and abnormal-exit cleanup guarantees.

A plausible intent is ordered propagation of a mode across related objects, with opposite receiver ordering for the two modes. That interpretation is not required to explain the established behavior.

## Evidence

This contract is based solely on static inspection of the supplied function representation and dependency-ID mapping. Embedded naming and comments are not treated as authoritative behavioral guarantees.

Static evidence establishes the mode branches, normalized arguments, dependency-call order, receiver-derived displacement, loop termination condition, indirect-dispatch displacement, and absence of explicit local validation or error handling.

Dependency implementations, concrete dispatch targets, surrounding callers, and runtime observations were not supplied. No runtime tests have been performed.
