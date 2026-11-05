<!-- Trusted provenance: F-f519a1c22faa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f519a1c22faa

## Purpose

F-f519a1c22faa selects one of two named clock-screen actions from a status obtained through an object interface, performs status-dependent preparation, and dispatches the selected action through a caller-supplied receiver.

The established behavior has three recognized status values:

- **0:** write a zero-valued byte to a state object and select `GoToClockRegionScreen`.
- **1:** write a one-valued byte to that state object and select `GoToClockRegionScreen`.
- **2:** conditionally invoke two preparation dependencies, then select `GoToClockScreenDefault`.

Any other status causes an early normal return without action construction or receiver dispatch.

All normal return paths return the 32-bit value **1**. This value does not distinguish recognized statuses, successful navigation, or the early-return case.

The action names suggest navigation between clock-related screens. That interpretation is inferred intent; the established contract is the sequence of calls, memory accesses, and arguments described below.

## Interface

### Inputs

The function accepts four arguments:

1. **Receiver object address.**  
   An address used for a final indirect method call. The first word at this address supplies a method-table address; the callable entry is obtained at byte offset **0x11c** within that table. The receiver address itself is passed as the first explicit argument to that method.

2. **Unused 32-bit argument.**  
   This argument has no visible use in the supplied body. Its value does not directly affect local branching, memory writes, or dependency arguments.

3. **First auxiliary 32-bit value.**  
   Its complete bit pattern is copied into a local 32-bit slot before any dependency call. The address of that slot is later supplied to F-9b9c1c7483f4 on recognized-status paths.

4. **Second auxiliary 32-bit value.**  
   Its complete bit pattern is copied into a separate local 32-bit slot before any dependency call. The address of this slot is supplied as the first argument to F-9b9c1c7483f4 and, afterward, to F-8a5aadac202f.

The two auxiliary values have no established interpretation as integers, pointers, strings, ownership tokens, or initialized objects. Their widths and address-passing behavior are established; their semantic types are not.

### Output

Every observed normal return produces the 32-bit value **1**.

The function does not return the selected action, the status, the result of preparation, or the receiver method’s result.

### Required object layouts

The visible operations require the following accessible layouts on paths that use them:

| Object | Relative location | Required access |
|---|---:|---|
| Object returned by F-938c950e5046 | First word | Read a method-table address |
| That method table | Byte offset 0x168 | Read and invoke a callable entry |
| State object returned by F-49fc629096f4 | Byte offset 0x2c | Read an unsigned 32-bit field, only for status 2 |
| State object returned by F-49fc629096f4 | Byte offset 0x5d | Write one byte, only for status 0 or 1 |
| Receiver object | First word | Read a method-table address, only for recognized statuses |
| Receiver method table | Byte offset 0x11c | Read and invoke a callable entry |

The supplied representation is consistent with a 32-bit target and four-byte words. The byte offsets above are layout requirements, not field ordinals. In particular, the write at **0x5d** is one byte, not a 32-bit assignment.

No null checks, object-type checks, bounds checks, or callable-entry checks are visible.

## Behavior

### Initial acquisition and status query

The following order is established:

1. Copy the third and fourth arguments into their respective local 32-bit slots.
2. Call F-938c950e5046 with no explicit arguments and retain its returned object address.
3. Call F-49fc629096f4 with no explicit arguments and retain its returned state-object address.
4. Read the first word of the object retained from step 2, obtain the callable entry at method-table byte offset **0x168**, and invoke it with that object address.
5. Interpret the returned status by exact comparison with **2**, **0**, and **1**.

Both acquisition calls occur before the status query. In particular, the state-object acquisition is not deferred until the function knows whether the status is recognized.

The status-query method-table access occurs after F-49fc629096f4 returns. The contract therefore does not assume that this dependency leaves the queried object or its method table unchanged.

The status is queried once. No branch performs a second status query.

### Status 2: default-screen action

For status **2**:

1. Read the unsigned 32-bit field at byte offset **0x2c** of the retained state object.
2. If that field is strictly greater than unsigned **1**:
   - Call F-ff84f6bc341c with the retained state-object address.
   - Pass its 32-bit result, unchanged, as the second argument to F-170ee3da113c.
   - Pass the same retained state-object address as the first argument to F-170ee3da113c.
3. Select the action name `GoToClockScreenDefault`.
4. Continue to common action construction and dispatch.

The comparison is unsigned. Values **0** and **1** skip the two preparation calls; every other 32-bit field value takes them, including values whose highest bit is set.

There is one explicit read of this field in the branch. The visible body does not reread it after either preparation call.

There is no direct write to the byte at **0x5d** in this branch. This does not imply that dependencies cannot change that byte or any other reachable state.

### Status 0: region-screen action with a zero byte

For status **0**:

1. Write exactly one zero-valued byte at byte offset **0x5d** of the retained state object.
2. Select `GoToClockRegionScreen`.
3. Continue to common action construction and dispatch.

The prior byte value is not inspected. The write occurs before F-9b9c1c7483f4 is called.

### Status 1: region-screen action with a one byte

For status **1**:

1. Write exactly one byte with value **1** at byte offset **0x5d** of the retained state object.
2. Select `GoToClockRegionScreen`.
3. Continue to common action construction and dispatch.

As with status 0, there is no read-modify-write operation and no inspection of the prior byte.

### Other statuses

For every status other than **0**, **1**, or **2**, return **1** immediately after classification.

On this path, the visible body does not:

- Read the state field at **0x2c**.
- Write the state byte at **0x5d**.
- Call either conditional preparation dependency.
- Call the action-construction dependency.
- Access the receiver’s method table.
- Invoke the receiver method.
- Call F-8a5aadac202f.

The two initial acquisitions and the status query have nevertheless already occurred, with any effects those operations may have.

### Common action construction and dispatch

For each recognized status, after its branch-specific operations:

1. Call F-9b9c1c7483f4 with these arguments, in order:
   - Address of the local slot initially containing the fourth argument.
   - Address of the selected action-name string.
   - Address of the local slot initially containing the third argument.
2. Retain the dependency’s 32-bit return value.
3. Read the receiver’s first word and obtain the callable entry at method-table byte offset **0x11c**.
4. Invoke that entry with the receiver address and the retained 32-bit result from F-9b9c1c7483f4.
5. Call F-8a5aadac202f with the address of the same local slot passed as the first argument to F-9b9c1c7483f4.
6. Return **1**.

The receiver method’s return value, if any, is not used.

The receiver’s method-table lookup happens after action construction. An implementation must not assume that F-9b9c1c7483f4 preserves receiver memory, other aliased objects, or global state.

F-8a5aadac202f receives the local slot as it exists after action construction and receiver dispatch. Its contents need not still equal the original fourth argument.

## Side effects

### Direct memory effects

The only explicit writes to nonlocal object storage in the supplied body are the single-byte assignments at state-object offset **0x5d** for statuses 0 and 1.

The function also initializes two local four-byte slots. It exposes their addresses to F-9b9c1c7483f4, and exposes one of them again to F-8a5aadac202f. Those calls may read or modify the slots according to contracts not supplied here.

No direct allocation, deallocation, or global-variable assignment is visible.

### Indirect effects and ownership

Every dependency and indirect method may have additional effects. None is assumed to preserve globals or reachable objects.

F-8a5aadac202f appears in a post-dispatch position compatible with cleanup, but its name and placement do not establish destruction, deallocation, ownership release, or reference-count behavior. Those semantics remain unknown.

Likewise:

- The two acquired object addresses are not visibly released.
- Their acquisition does not establish ownership transfer.
- Passing F-9b9c1c7483f4’s result to the receiver does not establish transfer or retention.
- The selected action-name address is passed onward without a visible ownership operation.
- No lifetime guarantee for the local slots extends beyond this invocation.

Aliases among the receiver, acquired objects, and memory reachable through auxiliary values are not excluded. Consequently, a direct byte write or a dependency call can affect memory observed by a later operation if the relevant addresses alias.

### Failure behavior

There is no visible error-return branch for failed acquisitions, invalid objects, failed action construction, or receiver failure.

If an operation does not return normally, the supplied body establishes no subsequent dispatch, cleanup, or return guarantee. In particular, F-8a5aadac202f is visibly sequenced after receiver dispatch; there is no shown exceptional-exit cleanup mechanism.

## Dependencies

| Opaque ID or interface | Visible role and contract boundary |
|---|---|
| **F-938c950e5046** | Called once without explicit arguments before state acquisition. Returns the object used for the status query. |
| **F-49fc629096f4** | Called once without explicit arguments before the status query. Returns the state-object address retained throughout the invocation. |
| **Status interface, offset 0x168** | Invoked once with the first acquired object. Its result selects the branch. No opaque function ID is supplied for the concrete target. |
| **F-ff84f6bc341c** | Called only for status 2 with the unsigned field at 0x2c greater than 1. Receives the state object and returns a 32-bit value. |
| **F-170ee3da113c** | Immediately follows F-ff84f6bc341c on that conditional path. Receives the retained state-object address and the preceding result. Its result is unused. |
| **F-9b9c1c7483f4** | Called once for each recognized status with two local-slot addresses and the selected action-name address. Its 32-bit result is forwarded to the receiver. |
| **Receiver interface, offset 0x11c** | Invoked once for each recognized status, after action construction. Receives the caller-supplied receiver and the construction result. |
| **F-8a5aadac202f** | Called after normal receiver-method completion with the address of the local slot initialized from the fourth argument. Its result is unused. |

No additional contracts for these dependencies are established by the supplied material.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not records of runtime tests. They assume each required memory access is valid and each preceding call returns normally.

1. **Status 0 with any initial state byte.**  
   The byte at **0x5d** is assigned **0** before action construction. The action name is `GoToClockRegionScreen`. The two conditional preparation dependencies are not called. Dispatch precedes the final F-8a5aadac202f call, and the function returns **1**.

2. **Status 1 with any initial state byte.**  
   The byte at **0x5d** is assigned **1**, and the selected name is `GoToClockRegionScreen`. Even an already-one byte does not eliminate the explicit write.

3. **Status 2 with the field at 0x2c equal to 0 or 1.**  
   The field is read, but F-ff84f6bc341c and F-170ee3da113c are skipped. The action name is `GoToClockScreenDefault`. There is no direct assignment to the byte at **0x5d**.

4. **Status 2 with the field equal to 2.**  
   F-ff84f6bc341c is called before F-170ee3da113c. The latter receives the former’s unchanged 32-bit result. Both precede action construction.

5. **Status 2 with the field equal to 0xffffffff.**  
   The preparation pair is invoked because the comparison is unsigned and this value exceeds 1.

6. **Status outside the recognized set.**  
   After both acquisitions and the status query, the function returns **1** without inspecting the receiver. An invalid receiver alone therefore does not affect this path, provided the earlier operations themselves complete normally.

7. **Action construction returns an arbitrary 32-bit pattern.**  
   The same pattern is passed to the receiver method without a visible validity test or conversion. A zero result is not specially rejected.

8. **Action construction modifies its first local slot.**  
   The final F-8a5aadac202f call receives that same slot’s address, not a fresh copy of the original fourth argument.

## Uncertainties

### Inferred intent

The action names suggest clock-screen navigation. The state byte may represent a regional-selection mode, and the status-2 preparation may reconcile existing state. These interpretations are not necessary to implement the established behavior and must not replace its exact branching rules.

### documented_only

The following remain undocumented by the supplied evidence:

- Concrete object types, complete layouts, and allowable alias relationships.
- The semantic meaning and full return domain of the queried status.
- The meanings of the state field at **0x2c** and byte at **0x5d**.
- The semantic types of the auxiliary arguments and construction result.
- Allocation, retention, destruction, and ownership rules of every dependency.
- Whether any dependency mutates globals, object tables, or aliased memory.
- Thread-safety, reentrancy, synchronization, and concurrent-mutation guarantees.
- Exception behavior and abnormal-exit cleanup outside the visible body.
- Independent confirmation of the target ABI beyond the supplied 32-bit representation.

## Evidence

This contract is based exclusively on the supplied static representation of F-f519a1c22faa and the supplied opaque dependency mapping.

Established evidence includes argument use, local-slot initialization, dependency order, indirect-call offsets, exact status comparisons, unsigned field comparison, byte-write widths, selected action names, result forwarding, and the constant normal return value.

Semantic interpretations and unavailable dependency contracts are separated above from those established operations. No implementation access, runtime execution, or runtime tests have been performed.
