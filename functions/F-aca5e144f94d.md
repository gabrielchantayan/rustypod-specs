<!-- Trusted provenance: F-aca5e144f94d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aca5e144f94d

## Purpose

F-aca5e144f94d repeatedly obtains an object pointer from F-413819d05b57, processes each non-null result through F-531bd09f6472, and then invokes an indirect operation selected from that object. A null result ends iteration and transfers control to F-a817d8d58c10.

This is a behavioral contract derived from the supplied static representation. It does not establish that the routine enumerates a particular container, destroys objects, releases memory, or implements a particular lifecycle protocol. Those interpretations require evidence about its dependencies.

The directly established ordering is:

1. Obtain or update the current candidate and auxiliary state.
2. Test the resulting candidate for null.
3. For a non-null candidate, call F-531bd09f6472.
4. Resolve and invoke the candidate’s indirect operation.
5. Repeat from the candidate-acquisition step.

The terminal dependency is annotated as not returning. No ordinary return value or successful return to the caller is established.

## Interface

The routine has four positional inputs and no declared result.

| Input | Representation | Established role |
|---|---|---|
| First input | 32-bit integer carrying an address-like value | Supplies the base used for an address displaced by 60 bytes, and is passed unchanged to F-531bd09f6472. |
| Second input | Opaque 32-bit value | Has no explicit use in the supplied body. |
| Third input | Opaque 32-bit value | Initializes a writable, invocation-local auxiliary-state cell. |
| Fourth input | Pointer to a word-oriented object, or null | Initializes a writable, invocation-local candidate-pointer cell. |

The representation is consistent with a 32-bit address model: integers are 32 bits, the opaque scalar inputs are four bytes, and indirect dispatch uses a function-pointer slot four bytes beyond the start of a table. The contract should not be mechanically widened to a 64-bit ABI while retaining these offsets.

Two addressable local cells are supplied to F-413819d05b57:

- A candidate-pointer cell, initially containing the fourth input.
- An auxiliary-state cell, initially containing the third input.

These cells are initialized once per invocation, not once per iteration. Their values persist between calls to F-413819d05b57 unless modified by a dependency with access to them.

The third and fourth inputs are passed by value into this routine. Updating the local cells is not, by itself, an update to the caller’s original argument storage.

### Preconditions for defined execution

The routine performs no explicit address validation. Execution requires the following whenever the corresponding operation is reached:

- The first input must support forming the address-like value 60 bytes beyond it.
- F-413819d05b57 must be callable with that displaced value and the addresses of the two local cells.
- Every non-null candidate used after F-531bd09f6472 returns must remain valid for indirect dispatch.
- The candidate’s initial word must supply a usable dispatch-table address.
- The table entry at byte offset 4 must contain a callable target compatible with receiving the candidate pointer.

The source does not establish handling for address overflow, invalid memory, malformed tables, incompatible call targets, or dependency failures.

## Behavior

### Initialization and first acquisition

The routine copies the third input into its auxiliary-state cell and the fourth input into its candidate-pointer cell.

It then calls F-413819d05b57 with three arguments, in this order:

1. The address-like value obtained by adding 60 bytes to the first input.
2. The address of the candidate-pointer cell.
3. The address of the auxiliary-state cell.

This call occurs before any null test on the initial candidate. Consequently, a null fourth input does not independently request immediate termination. F-413819d05b57 receives that initial null value through the writable cell and may replace it.

Similarly, a non-null fourth input is not processed directly merely because it was supplied by the caller. F-413819d05b57 runs first and may replace it with another pointer or with null.

The return value of F-413819d05b57, if any, is not consumed. Its observable contribution to this routine is through the local cells and any other effects it performs.

### Candidate selection

After F-413819d05b57 returns, the routine reads the candidate-pointer cell and retains that pointer as the selected candidate for the current iteration.

If the selected candidate is null, no per-candidate processing or indirect dispatch occurs in that iteration. Iteration ends, and the terminal dependency is called.

If the selected candidate is non-null, the routine calls F-531bd09f6472 with:

1. The original first input.
2. The selected candidate pointer.

No return value from F-531bd09f6472 is used.

The routine does not establish that this dependency preserves object contents, global state, memory ownership, or the candidate-pointer cell through possible aliases. It does establish that subsequent dispatch uses the selected candidate pointer retained before the call, rather than intentionally selecting a new pointer from the candidate cell afterward.

### Indirect dispatch

Only after F-531bd09f6472 returns does the routine obtain the indirect call target.

The dispatch layout is:

| Location | Interpretation |
|---|---|
| Byte offset 0 in the selected candidate | Address of a dispatch table |
| Byte offset 4 in that table | Function pointer to invoke |
| Indirect-call argument | The selected candidate pointer |

These accesses are ordered after F-531bd09f6472. The table pointer and function pointer are not established as having been read or cached before that dependency runs. A valid modification of the candidate’s dispatch-table pointer, or of the table entry, can therefore affect the target invoked in the current iteration.

The indirect operation receives the selected candidate pointer as its explicit argument. No explicit deletion flag, size, ownership marker, or auxiliary-state argument is supplied.

The indirect operation’s return value, if any, is ignored. If it returns normally, the next iteration begins with another call to F-413819d05b57. The candidate-pointer cell is not explicitly cleared, and the auxiliary-state cell is not reset.

There is no established per-object uniqueness guarantee. If F-413819d05b57 repeatedly leaves the same non-null candidate in the cell, that candidate is processed repeatedly.

### Termination

When the post-acquisition candidate is null, the routine calls F-a817d8d58c10 with no explicit arguments.

The supplied representation annotates this terminal dependency as not returning. Thus, normal return from F-aca5e144f94d is not an established outcome.

The annotation does not identify the terminal mechanism. It does not establish whether the dependency terminates a process, transfers control through an exception-like mechanism, exits a thread, or does something else.

No finite iteration bound is present. If every acquisition returns normally and leaves a non-null candidate, and every per-candidate operation returns normally, the routine continues indefinitely.

## Side effects

### Direct local writes

The routine directly initializes the two local cells. Their addresses are then exposed to F-413819d05b57.

No direct store to the memory addressed by the first input or to a selected object is visible in the supplied body. This does not imply that those regions are unchanged: dependencies may mutate them.

### Dependency-mediated effects

Observable effects may arise from:

- F-413819d05b57, including changes to either supplied cell.
- F-531bd09f6472, including changes relevant to the following dispatch.
- The indirect operation selected from each candidate.
- F-a817d8d58c10 on termination.

No dependency is assumed to preserve global state or unrelated reachable memory.

The displaced address supplied to F-413819d05b57 is formed from the original first input for each iteration. The routine does not retain a one-time snapshot of the memory at that address.

### Ownership and lifetime

The routine does not explicitly allocate or release memory.

In particular, the indirect call must not be described as a destructor or deallocator solely because of its table position or its placement after F-531bd09f6472. Ownership transfer and release behavior are unknown.

If F-531bd09f6472 invalidates the selected candidate before returning, the subsequent dispatch still attempts to use that candidate. No guard or replacement lookup is visible.

The local cells are invocation-local storage. They are valid during the call to F-413819d05b57. The evidence does not establish that retaining their addresses for use after this invocation ends is valid.

### Failure propagation

There is no visible local recovery, retry-on-error branch, cleanup handler, or error translation.

If a dependency does not return normally, the later operations in that iteration are not established as occurring. If a memory access or indirect call is invalid, no defined recovery behavior is supplied by this routine.

## Dependencies

| Dependency | Established interaction | Not established |
|---|---|---|
| F-413819d05b57 | Called at the start of every iteration with the first input displaced by 60 bytes, a writable candidate-pointer cell, and a writable four-byte auxiliary-state cell. | Enumeration rules, state encoding, termination policy, ownership effects, and failure behavior. |
| F-531bd09f6472 | Called once for each non-null selected candidate, before dispatch, with the original first input and that candidate. | Whether it detaches, registers, mutates, releases, or otherwise manages the candidate. |
| Indirect operation, opaque identity unavailable | Resolved after F-531bd09f6472 from the selected candidate’s dispatch table at byte offset 4; called with that candidate. | Semantic purpose, ownership effects, additional ABI requirements, and failure behavior. |
| F-a817d8d58c10 | Called without explicit arguments after acquisition produces null; annotated as non-returning. | Exact terminal mechanism and external effects. |

The second input is not forwarded explicitly to any of these dependencies.

## Acceptance examples

These are conditional static acceptance cases, not reports of executed tests. They assume valid addresses, compatible calling conventions, and normal returns except where stated otherwise.

### Immediate termination after acquisition

Given any initial candidate, if the first call to F-413819d05b57 leaves the candidate-pointer cell null:

- F-531bd09f6472 is not called.
- No candidate dispatch-table access occurs.
- F-a817d8d58c10 is called.

The initial candidate alone does not determine this outcome.

### Null initial candidate replaced by a valid candidate

Given a null fourth input, if F-413819d05b57 writes a valid non-null candidate into the cell:

- That candidate is passed to F-531bd09f6472.
- Its indirect operation is resolved afterward and invoked.
- Another acquisition follows if the indirect operation returns normally.

Treating the initial null as an early exit would violate the established ordering.

### Two candidates followed by termination

If successive acquisition calls leave the candidate cell containing candidate A, candidate B, and null, respectively, the observable call order is:

1. Acquisition.
2. F-531bd09f6472 for A.
3. Indirect operation for A.
4. Acquisition.
5. F-531bd09f6472 for B.
6. Indirect operation for B.
7. Acquisition.
8. F-a817d8d58c10.

There is no per-candidate call for the final null result.

### Persistent auxiliary state

If the first acquisition changes the auxiliary-state cell from its initial value to another four-byte value, that changed value remains available to the next acquisition unless an intervening effect changes it. The routine does not restore the third input between iterations.

### Dispatch changed during processing

If F-531bd09f6472 validly changes the selected candidate’s table pointer or its table entry at byte offset 4, the subsequent dispatch uses the values read after that call. Dispatching through an entry cached before the call would not match the observed ordering.

### Repeated candidate

If acquisition repeatedly leaves the same valid non-null candidate and all invoked operations return, processing and dispatch repeat for that candidate. The routine performs no visible duplicate suppression.

## Uncertainties

The following are **documented_only**:

- The meanings of the first input’s referenced region, the 60-byte displacement, and the auxiliary-state value.
- Whether the observed 32-bit address model is the complete external ABI.
- Address-wrap behavior when forming the displaced argument. No checked arithmetic is visible, but a portable overflow rule is not established.
- The identity and semantics of the indirect operation.
- Ownership and lifetime guarantees supplied by each dependency.
- Whether dependencies retain addresses of the local cells.
- The exact non-returning behavior of F-a817d8d58c10.
- Synchronization, reentrancy, and behavior under concurrent mutation.
- Any externally documented significance of the unused second input.

An iterative traversal followed by lifecycle processing is a possible interpretation, but it remains inferred intent rather than an acceptance requirement.

## Evidence

Established static evidence consists of the supplied routine body, dependency-ID mapping, four-byte scalar representations, address displacements, indirect-call layout, branch structure, and terminal non-return annotation.

The central observable constraints are acquisition before the null test, F-531bd09f6472 before dispatch-table reads, retained-candidate dispatch, persistent local state across iterations, and terminal invocation after a null acquisition result.

Dependency implementations, runtime traces, and authoritative type or ownership declarations were not supplied. No runtime tests have been performed.
