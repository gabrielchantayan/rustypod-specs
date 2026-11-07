<!-- Trusted provenance: F-f2de6e8bcf21; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2de6e8bcf21

## Purpose

F-f2de6e8bcf21 prepares a context-referenced object and conditionally inserts it into a linked structure. Insertion depends on an active marker, a nonzero interval-like field, and comparison results against existing nodes. After either skipping or completing insertion, it performs a trailing dependency call and conditionally invokes an indirect target.

Initial setup occurs even when the object is already marked active. An active object returns before insertion preparation and trailing dispatch. An inactive object with a zero interval skips insertion but still reaches trailing dispatch.

The apparent purpose is activation or scheduling: an interval is multiplied by 1,000 and added to an external result to form a comparison key. Scheduling, time units, ordering direction, and synchronization roles are inferred intent, not established contracts.

## Interface

### Inputs

The recovered interface exposes four 32-bit arguments:

| Argument | Established use |
|---|---|
| Context address | Supplies an object reference through a four-byte slot at byte offset +176. |
| Second argument | No explicit use appears in the supplied body. |
| Third argument | Passed to the comparison dependency on each comparison. |
| Fourth argument | Passed to the comparison dependency on each comparison. |

The comparator also receives a fifth 32-bit value originating from unrecovered incoming register state. Its meaning and relationship to the apparent interface are unknown. Comparison behavior therefore cannot be specified solely from the four explicit arguments.

The second invocation of F-936a8728139d has no recovered explicit arguments. This does not establish that it has no machine-level inputs.

### Object and node layouts

All listed slots are four bytes wide. Offsets are bytes from the relevant object address.

| Candidate offset | Established role |
|---:|---|
| +0 | Next-node reference, written during insertion. |
| +4 | Interval-like value; zero suppresses insertion. |
| +8 | Computed comparison key; its address is passed to the comparator. |
| +24 | Receives an external global value during insertion preparation. |
| +28 | Active marker; zero permits further processing, and insertion writes 1. |
| +32 | Receives an external global value before the active-marker check. |

Each traversed node must provide a readable four-byte next reference at +0. Its address at +8 is passed to the comparator; the comparator’s access width and interpretation are not established.

Addresses and stored node references are represented as 32-bit values in the supplied representation. No null, bounds, alignment, or lifetime validation is visible for the context or retained object.

### Shared preparation and dispatch state

One global location connects preparation to trailing dispatch. This contract calls it the **shared control-cell global**:

- The shared control-cell global contains a 32-bit address.
- That address identifies a **control cell**.
- Reading the control cell yields a 32-bit **dispatch-state pointer**.
- A non-null dispatch-state pointer identifies storage whose first four-byte word controls indirect dispatch.

Preparation passes the value held in the shared control-cell global to F-338d16f9ce72. It does not visibly read the control cell at that point.

Trailing dispatch reads the same global again, then reads the control cell addressed by that newly obtained value. The earlier value is not retained for this purpose.

### Output

No return value or status result is established. Normal completion is by return after the active-marker early exit, after a suppressed dispatch, or after the indirect target returns.

## Behavior

### Initial setup and object selection

The function performs three separate reads of the context’s object-reference slot at +176:

1. The first value is passed to F-936a8728139d.
2. After that call returns, a fresh value is passed to F-6904f7699031 together with a value read from an external global.
3. After that call returns, another fresh value becomes the retained candidate address.

These reads must not be collapsed into a single lookup. Either preceding dependency may change the context slot.

After retaining the candidate address, the function invokes F-936a8728139d again, with explicit arguments unrecovered. It does not refresh the retained candidate address after this invocation.

### Unconditional update and active-marker gate

After the second invocation of F-936a8728139d returns, the function reads an external global value and writes it to candidate offset +32. It then reads the marker at +28.

If the marker is nonzero, the function returns immediately. This path includes the initial dependency calls and the +32 write, but excludes:

- F-338d16f9ce72 and F-add44d4e9aaa.
- The interval read and all direct insertion work.
- F-5bcae2eff9a7.
- The trailing shared-global read and indirect-dispatch checks.

Thus, an already-active object is not a side-effect-free no-op.

If the marker is zero, processing continues. The marker is not visibly checked again before insertion.

### Preparation and interval selection

For a marker observed as zero, the function:

1. Reads the shared control-cell global and passes its value to F-338d16f9ce72.
2. Calls F-add44d4e9aaa with the retained candidate address.
3. Reads the four-byte interval-like field at candidate offset +4.

The interval decision therefore uses the field value after both dependencies return, not its entry value. Neither dependency is assumed to preserve that field, the marker, the shared global, or other state.

No locking or unlocking contract is established for these calls.

### Zero interval

If the observed interval is zero, the body skips:

- The external call supplying the key’s base value.
- Direct writes to candidate offsets +8 and +24.
- Reading the list-head location for insertion.
- Traversal and comparator calls.
- Link publication.
- Setting the marker to 1.
- Writing the candidate’s next reference.

It proceeds to F-5bcae2eff9a7 and the trailing dispatch sequence.

These are exclusions of direct actions on this path, not guarantees that dependencies leave the corresponding memory unchanged.

### Nonzero interval and key preparation

Any nonzero 32-bit interval representation enters this branch. There is no visible positive-only check or range validation.

The function calls an external dependency whose opaque ID is unavailable in the supplied mapping. Its recovered result is a 32-bit integer-like value used as the key’s base.

After that call returns, the function captures an address from an external global. That captured address identifies the list-head slot. It then performs these operations in order:

1. Stores at candidate +8 the result of adding the base value to the captured interval multiplied by 1,000.
2. Reads another external global value and stores it at candidate +24.
3. Reads the current head reference through the captured list-head-slot address.

The interval operand was captured before the base-producing call. Changes to candidate +4 during that call do not replace the captured operand.

The stored key is 32 bits wide. For inputs whose multiplication and addition remain within the applicable recovered signed 32-bit range, the mathematical calculation describes the result. There is no visible overflow check, saturation, rejection, or explicit wider intermediate. Exact overflow behavior remains documented_only; this contract does not assume portable signed wraparound.

The base value’s meaning and units are unknown.

### Comparison-driven traversal

Traversal begins with the head reference just read and no predecessor.

For each non-null current node, F-0d70d6f9010c receives:

- The address at current-node offset +8.
- The address at candidate offset +8.
- The third incoming argument.
- The fourth incoming argument.
- The unrecovered incoming register value.

These are address-based operands, not copied key values.

Only zero versus nonzero in the comparator’s recovered result controls traversal:

- Zero advances past the current node.
- Nonzero stops at the current node.
- A null current node stops without a comparison.

When advancing, the function reads the current node’s next reference after the comparator returns and records the current node as predecessor. A comparator mutation of that next reference can therefore affect traversal.

The insertion position is before the first encountered node producing a nonzero result, or after the traversed prefix if traversal reaches null. Numeric ordering, equality treatment, and stability are not established.

There is no visible cycle detection or traversal bound. Cyclic links or comparator-driven mutations can prevent termination.

### Insertion writes

Once traversal stops, the direct writes occur in this order:

1. Publish the candidate address through the captured head slot if there is no predecessor; otherwise publish it through predecessor offset +0.
2. Write the 32-bit value 1 to candidate offset +28.
3. Write the stopping-node address, possibly null, to candidate offset +0.

The forward link is finalized after the candidate becomes reachable through the head or predecessor. The body therefore does not establish atomic publication of a fully linked node. Whether dependencies or surrounding execution prevent observation of the intermediate state is unknown.

No check confirms that the candidate is absent from the list. The earlier zero-marker observation is not revalidated after preparation or comparison calls. Avoiding duplicate membership or self-links requires suitable surrounding state and dependency behavior.

### Trailing call and indirect dispatch

The zero-interval path and the completed-insertion path both call F-5bcae2eff9a7 with a value read from another external global.

After that call returns, the function performs the following reads and decisions:

1. Rereads the shared control-cell global to obtain a control-cell address.
2. Reads the four-byte control cell at that address to obtain the dispatch-state pointer.
3. If the dispatch-state pointer is non-null, reads its first four-byte word.
4. If the dispatch-state pointer is non-null and that first word is nonzero, reads an indirect target from a separate external location and invokes it.

The control-cell read is unconditional on this trailing path. There is no visible null check on the control-cell address obtained from the shared global. A null dispatch-state pointer suppresses the first-word read and indirect invocation; a null control-cell address does not receive that protection.

The shared global is reread after F-5bcae2eff9a7. Its value may differ from the value passed earlier to F-338d16f9ce72, and the newly addressed control cell may contain a different state pointer. No intervening dependency is assumed to preserve either location.

The indirect target is not visibly checked for null or validity. No explicit call arguments or use of its return value are recovered.

## Side effects

The direct body writes candidate +32 whenever execution reaches the active-marker gate, including the active-object early-return path.

Insertion additionally writes candidate +8, candidate +24, the head or predecessor link, candidate +28, and candidate +0 in the order described above.

The body contains no visible allocation, deallocation, whole-object copy, or explicit ownership transfer. Linking creates reachability; it does not establish responsibility for storage release.

Dependencies may modify globals, context contents, object fields, list links, or memory lifetime. No preservation or ownership contract is inferred for them. Normal execution requires that every subsequently accessed address remain valid despite intervening calls.

The supplied evidence does not establish non-aliasing among external storage, object fields, and list storage. Distinct-role examples below assume the stated locations do not overlap in ways that change their outcomes; the observable operation order remains the contract if aliasing exists.

No error reporting, rollback, or recovery mechanism is visible. A dependency that fails to return prevents the later sequence from being established. Invalid addresses or invalid indirect targets have no demonstrated recovery path.

## Dependencies

| Dependency | Established interaction |
|---|---|
| F-936a8728139d | First receives a context-derived object reference; invoked again after candidate retention with explicit arguments unrecovered. |
| F-6904f7699031 | Receives a freshly read context-derived reference and an external global value. |
| F-338d16f9ce72 | Receives the value read from the shared control-cell global after the zero-marker observation. |
| F-add44d4e9aaa | Receives the retained candidate address before the interval read. |
| F-0d70d6f9010c | Receives two operand addresses and three additional values; zero advances traversal, nonzero stops it. |
| F-5bcae2eff9a7 | Receives an external global value before the shared control-cell global is reread for dispatch. |

Two dependencies have no supplied opaque IDs: the base-producing call and the conditional indirect target. Their identities and wider contracts remain documented_only; no substitute function identifiers are assigned.

## Acceptance examples

These are conditional requirements derived from static evidence, not executed tests. Unless an example specifies mutations, dependencies return normally and preserve the storage and conditions needed for its stated result.

### Already-active candidate

If the retained candidate’s marker is nonzero when checked, the initial three dependency invocations and the +32 write occur. The function then returns without preparation, insertion, F-5bcae2eff9a7, or trailing dispatch-state reads.

### Context replacement during setup

If the first dependency changes the context’s object-reference slot, F-6904f7699031 receives the replacement reference. If that call changes the slot again, the third read selects the later reference as the retained candidate. The references need not identify the same object.

### Preparation changes the interval

If F-add44d4e9aaa changes the candidate interval to zero, the following interval read selects the zero-interval path. No direct insertion occurs, but F-5bcae2eff9a7 and the trailing dispatch sequence still occur.

### Empty-list insertion

For a nonzero interval and a null head after key preparation, no comparator call occurs. The function writes the candidate into the head slot, writes marker 1, and then writes a null candidate next reference.

For a captured interval of 2 and base result of 7, with no arithmetic overflow, the key written before those insertion writes is 2,007.

### Insert after a traversed prefix

For distinct nodes where the first comparison returns zero and the second returns nonzero, the first node becomes the predecessor. Its next reference receives the candidate, then the candidate marker becomes 1, then the candidate next reference receives the second node’s address.

If every comparison returns zero and the final next reference is null, the corresponding insertion is after the last traversed node.

### Shared-global replacement before dispatch

Suppose preparation reads a control-cell address from the shared global and passes it to F-338d16f9ce72. If a later dependency replaces the shared-global value, trailing dispatch uses the replacement after F-5bcae2eff9a7 returns. It reads the replacement control cell rather than reusing the earlier address.

### Dispatch gating and dereferences

After F-5bcae2eff9a7 returns:

- A valid control cell containing a null state pointer suppresses the state-word read and indirect call.
- A valid control cell containing a non-null state pointer whose first word is zero suppresses the indirect call.
- A valid control cell containing a non-null state pointer whose first word is nonzero causes the indirect invocation, provided the target is valid.
- A null or invalid control-cell address is not a supported suppression case: the body attempts the control-cell read without a preceding check.

## Uncertainties

The following remain documented_only:

- Exact calling convention and unrecovered machine-level inputs.
- Arguments to the second F-936a8728139d invocation and indirect target.
- Meaning of external global values and the shared control-cell mechanism.
- Dependency synchronization, mutation, lifetime, and ownership contracts.
- Base-producing dependency identity, result meaning, and units.
- Exact overflow behavior of key arithmetic.
- Comparator operand widths, ordering semantics, and side effects.
- Aliasing relationships among accessed storage.
- Indirect target identity and effects.
- Termination under cycles or dependency-driven link changes.

Absent checks do not establish support for malformed contexts, invalid pointers, duplicate membership, or cyclic lists.

## Evidence

This contract uses only the supplied static body and opaque dependency mapping. The prior draft and reviewer feedback are not treated as independent behavioral evidence.

Established evidence includes the distinct context-slot reads, four-byte field accesses, call order, active-marker early return, post-preparation interval read, key construction, comparison-controlled traversal, ordered insertion writes, and trailing dispatch chain.

In particular, the same shared control-cell global supplies the preparation argument and is reread after F-5bcae2eff9a7. The trailing path then dereferences the resulting control-cell address to obtain a state pointer, conditionally reads that state’s first word, and conditionally invokes the indirect target.

Scheduling and synchronization interpretations remain inference. No runtime tests, execution traces, or independent implementation inspection have been performed.
