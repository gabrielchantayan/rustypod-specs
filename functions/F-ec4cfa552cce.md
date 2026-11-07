<!-- Trusted provenance: F-ec4cfa552cce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec4cfa552cce

## Purpose

F-ec4cfa552cce conditionally inserts a caller-supplied record into a shared singly linked list, then performs a separate shared-state check that can invoke an indirect dependency.

The established behavior is narrower than a general scheduling API:

- An entry dependency runs on every invocation.
- A shared marker value is copied into the record before its existing membership state is checked.
- A nonzero membership field causes an immediate return.
- Otherwise, two dependencies run before the record’s interval field is examined.
- A zero interval skips insertion, but does not skip the subsequent completion dependency or indirect-dispatch check.
- A nonzero interval produces a 32-bit computed value and inserts the record at a position selected by a comparison dependency.

A timer or delayed-work interpretation is plausible because the interval is multiplied by 1,000 and added to a value returned by another dependency. That interpretation is inferred intent, not an established unit, clock domain, or deadline policy.

## Interface

### Explicit inputs

The recovered interface has four explicit arguments:

| Argument | Established use |
|---|---|
| Record address | Address of a writable record, with at least nine accessible 32-bit words. |
| Second 32-bit argument | No explicit read or forwarding is visible in this function. |
| Third 32-bit argument | Forwarded to every invocation of F-0d70d6f9010c. |
| Fourth 32-bit argument | Forwarded to every invocation of F-0d70d6f9010c. |

The comparison dependency also receives a fifth value originating in incoming machine-register state. The supplied evidence does not establish that value’s type, origin, validity, or caller-level contract. It must not be described as a reliably initialized fifth explicit argument.

### Record layout

Offsets are byte offsets from the supplied record address. Field names below describe observed roles, not an established public structure definition.

| Offset | Width | Observed role |
|---|---:|---|
| +0 | 32 bits | Next-record link. Written only on the insertion path. |
| +4 | 32 bits | Interval-like value. Loaded after F-add44d4e9aaa returns. Zero skips insertion. |
| +8 | 32 bits | Computed comparison value. Its address is passed to F-0d70d6f9010c. |
| +12 through +20 | Three 32-bit words | No direct accesses shown. Dependencies may access them. |
| +24 | 32 bits | Metadata copied from shared state during insertion preparation. |
| +28 | 32 bits | Membership gate. Any nonzero value causes the early return; successful insertion writes exactly 1. |
| +32 | 32 bits | Marker copied from shared state on every invocation that reaches the first direct write. |

The representation uses 32-bit link values as addresses. A compatible environment must support that representation, along with the alignment and accessibility required by the implementation’s word accesses.

### Result and preconditions

No explicit return value is established. The observable result consists of dependency calls, direct memory writes, and potentially an indirect call.

The record must be valid before the first direct write at offset +32 and the subsequent read at offset +28. Additional requirements depend on the path:

- Insertion requires valid writable fields at offsets +0, +8, +24, and +28.
- Traversal requires readable next links in visited records and accessible comparison data beginning at offset +8.
- Inserting after a predecessor requires that predecessor’s next link to be writable.
- Shared pointer cells and any objects dereferenced during the final dispatch check must be valid when accessed.

There is no visible validation for null record addresses, inaccessible memory, malformed links, or incompatible pointer widths.

## Behavior

### Entry and membership gate

F-936a8728139d is invoked first, with no explicit arguments.

After it returns, the function obtains a shared 32-bit marker and stores it at record offset +32. It then reads record offset +28.

If that field is nonzero, the function returns immediately. In particular, this path does not directly:

- Invoke the admission or preparation dependencies.
- Examine the interval field.
- Compute a new comparison value.
- Traverse or modify the list.
- Invoke the completion dependency.
- Perform the final dispatch check.

This is not a side-effect-free “already present” path: the entry dependency has run, and the marker field has been overwritten.

The membership decision uses the field’s value after the entry dependency and marker write. It is not a snapshot taken at function entry.

### Preparation and interval decision

For a zero membership field, the function invokes F-338d16f9ce72 with a value read from a shared location. That shared location later supplies a pointer used by the final dispatch check, but its value can change in the meantime.

Next, F-add44d4e9aaa is invoked with the record address.

Only after that call returns does this function load the 32-bit interval-like field at offset +4. Consequently, changes made to that field by either preceding dependency affect this invocation.

If the loaded interval is zero, all insertion work is skipped. Execution proceeds directly to F-5bcae2eff9a7 and the final dispatch check.

The zero-interval path does not directly clear the next link, computed comparison value, metadata, or membership field. Their resulting values can nevertheless depend on the dependencies already invoked.

### Computed comparison value

For a nonzero interval, an unmapped value-producing dependency is invoked. Its recovered result is a 32-bit integer.

The function then reads the shared address of the list-head cell and retains that address for this insertion. It computes a value corresponding to:

- The loaded interval multiplied by 1,000.
- The returned integer added to that product.
- Storage of the resulting 32-bit value at record offset +8.

The interval was loaded before the value-producing call. A dependency changing record offset +4 during that call does not change the interval already selected for this computation.

The recovered representation treats the operands as 32-bit integers and the destination as a 32-bit word. It does not establish overflow handling. There is no visible widening operation, saturation branch, range check, or error result. Exact signed-overflow semantics require additional evidence; an unbounded mathematical result must not be promised.

After writing offset +8, the function reads shared metadata and stores it at offset +24. It then reads the current head link through the retained list-head-cell address.

### Traversal and insertion position

Traversal begins with the current head as the candidate and no predecessor.

For each non-null candidate, F-0d70d6f9010c receives:

1. The candidate address plus eight bytes.
2. The incoming record address plus eight bytes.
3. The third explicit argument.
4. The fourth explicit argument.
5. The additional value derived from incoming register state.

A zero comparison result means “advance past this candidate.” A nonzero result means “stop before this candidate.” No distinction between positive and negative nonzero results is made.

When advancing, the function reads the candidate’s next link after the comparison dependency returns. Therefore, a comparison dependency that modifies that link affects the subsequent traversal.

Traversal stops at either:

- The first candidate producing a nonzero comparison result; or
- A null candidate reached at the end of the chain.

No comparison occurs for a null candidate.

The evidence does not establish an ascending or descending ordering, equality handling, or a tie-breaking rule. Those properties depend on F-0d70d6f9010c and the objects it examines.

### Insertion writes

Once traversal stops, the function performs these direct writes in order:

1. If there is no predecessor, replace the retained list-head cell with the incoming record address. Otherwise, replace the predecessor’s next link with that address.
2. Write exactly 1 to incoming record offset +28.
3. Write the stopped candidate address, possibly null, to incoming record offset +0.

This order publishes the incoming record through the list before writing its membership flag and final next link. The implementation does not provide evidence of an atomic insertion transaction or safe observation by concurrent readers.

The function does not verify that a zero membership field accurately represents absence from the list. If the record is already reachable despite that field being zero, traversal or insertion can create invalid topology, including cycles.

### Completion and indirect dispatch

Both the successful insertion path and the zero-interval path invoke F-5bcae2eff9a7 with a value read from another shared location.

After that dependency returns:

1. The function rereads the shared location previously used to obtain the argument for F-338d16f9ce72.
2. It treats the current value as the address of a pointer cell and reads that cell.
3. If the resulting object pointer is non-null, it reads the object’s first 32-bit word.
4. If both the object pointer and that word are nonzero, it invokes a shared indirect target with no explicit arguments.

A null object pointer suppresses the object-word read and the indirect invocation. A non-null object whose first word is zero suppresses the invocation but not the object-word read.

The pointer-cell address itself is dereferenced without a visible null check. The indirect target is not visibly checked for null or validity.

The indirect target is consulted only after the eligibility check. The supplied representation models the transfer as a call followed by return, but includes an unresolved control-transfer indication. Exact call-versus-tail-transfer mechanics are not established.

## Side effects

### Direct memory effects

Depending on the path, this function directly modifies:

- Record offset +32 on every invocation that returns from F-936a8728139d and reaches that write.
- Record offsets +8 and +24 when the loaded interval is nonzero.
- Either the shared list-head cell or a predecessor’s next link during insertion.
- Record offsets +28 and +0 after publishing the inserted record.

There is no direct allocation, deallocation, record copying, or removal from the list in the supplied body.

### Ownership and lifetime

Insertion makes the supplied address reachable through shared list state. The body does not establish an ownership transfer or release policy. At minimum, any use of that shared link requires the record to remain valid for the relevant access.

No dependency may be assumed to preserve global state, refrain from modifying the record, or avoid acquiring or releasing resources. The apparent admission/completion pairing is insufficient evidence of locking or ownership semantics.

### Errors and interruption

There is no explicit error status, rollback, exception recovery, or traversal limit.

If a dependency does not return, subsequent operations do not occur. If an invalid access interrupts insertion after the publication write, the record can already be reachable without its later fields having been finalized.

A cyclic chain can prevent termination when every comparison returns zero. No cycle detection is visible.

## Dependencies

Mapped identifiers below use the supplied opaque function IDs. The two unmapped identifiers are document-local labels, not claims about external identity.

| Opaque ID | Invocation and established role |
|---|---|
| F-936a8728139d | First operation; no explicit arguments. Semantics otherwise unknown. |
| F-338d16f9ce72 | Called only after the membership gate passes; receives a shared value. |
| F-add44d4e9aaa | Receives the incoming record address before its interval is loaded. |
| F-0d70d6f9010c | Comparison dependency. Zero advances traversal; any nonzero result stops it. Receives two offset addresses and three further values. |
| F-5bcae2eff9a7 | Called after optional insertion, including the zero-interval case; receives a shared value. |
| F-unmapped-01 | Document-local ID for the value-producing dependency used in the 32-bit computation. No explicit arguments are shown. |
| F-unmapped-02 | Document-local ID for the shared indirect target. Invoked only when the final object pointer and its first word are nonzero. |

The shared marker, metadata, list-head-cell address, admission-state value, completion argument, and indirect target are distinct observed data sources. Their application meanings and initialization rules remain unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless otherwise stated, dependencies return normally and do not modify relevant state.

### Existing membership

Given any nonzero value at record offset +28:

- F-936a8728139d runs.
- Offset +32 receives the shared marker.
- The function returns without invoking the remaining dependencies or performing list operations.

This applies equally to a membership value of 1 and to any other nonzero 32-bit pattern.

### Zero interval after preparation

Given zero membership and an interval that is zero when loaded after F-add44d4e9aaa:

- Admission and preparation dependencies run.
- F-unmapped-01 and the comparator are not invoked.
- No insertion-related direct writes occur.
- F-5bcae2eff9a7 and the final dispatch check still occur.

An initially nonzero interval changed to zero by preparation follows this same path.

### Empty list

Given zero membership, interval 2, a value-producing result of 500, and a null list head:

- Offset +8 receives 2,500.
- Offset +24 receives shared metadata.
- The comparator is not invoked.
- The head cell receives the record address.
- Offset +28 becomes 1.
- Offset +0 becomes null.

The example avoids overflow and therefore does not depend on unresolved overflow semantics.

### Insert before the current head

Given a nonempty list and a nonzero result from the first comparison:

- Traversal stops at the original head.
- The head cell is replaced with the incoming record.
- The incoming record’s membership becomes 1.
- Its next link becomes the original head.

Both positive and negative nonzero comparator results select this position.

### Insert between records

Given consecutive candidates A and B, a zero comparison for A, and a nonzero comparison for B:

- A’s next link is read after A’s comparison.
- A becomes the predecessor.
- A’s next link is replaced with the incoming record address.
- The incoming record’s membership becomes 1.
- Its next link becomes B.

### Final dispatch uses post-completion state

Given that F-5bcae2eff9a7 changes the shared dispatch-related pointer cell:

- The final check uses the state read after that dependency returns.
- A null resulting object pointer causes no object-word read and no indirect invocation.
- A non-null object with a zero first word causes no indirect invocation.
- A non-null object with a nonzero first word causes invocation of the shared indirect target, assuming that target is valid.

## Uncertainties

The following are **documented_only**:

- The application meaning of the record, interval, marker, and metadata.
- The unit, epoch, and behavior of F-unmapped-01.
- Exact arithmetic behavior when the multiplication or addition exceeds signed 32-bit range.
- Comparator ordering, equality behavior, access extent, and mutation effects.
- The provenance and ABI significance of the comparator’s additional register-derived argument.
- Initialization, aliasing, synchronization, and ownership rules for shared data.
- Whether any dependency acquires or releases memory or synchronization resources.
- The ABI and exact control-transfer behavior of the indirect target.
- Behavior under invalid pointers, concurrent mutation, reentrant invocation, or corrupt list topology.

These unknowns do not justify assuming memory preservation, lock pairing, overflow safety, or recovery behavior.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the explicit 32-bit field accesses, branch conditions, dependency-call order, comparator zero/nonzero handling, insertion-write order, and final pointer-check sequence.

The delayed-work interpretation is inferred from arithmetic and list usage, not established by dependency contracts. Overflow semantics, hidden calling-convention details, and the unresolved indirect transfer remain uncertain.

No runtime tests have been performed.
