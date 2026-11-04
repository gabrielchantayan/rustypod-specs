<!-- Trusted provenance: F-438680ffebfe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-438680ffebfe

## Purpose

F-438680ffebfe selects between two receiver operations according to whether a supplied byte pointer is null or points initially to a zero byte.

For a null pointer or an initially zero byte, it invokes a receiver-selected operation without obtaining a size or invoking the final transfer dependency.

For an initially nonzero byte, it obtains an integer from F-5cb57bbb2e0e, requests a receiver-selected operation using that integer plus one and a zero-valued argument, and conditionally invokes a further indirect dependency with the returned value and the original byte pointer.

**Inferred intent:** this resembles replacing receiver-managed content from a byte string, with a separate clear/reset path and a destination-acquisition step. The evidence does not establish string-length semantics, allocation, copying, termination, or clearing. Those interpretations are not requirements of this contract.

## Interface

### Inputs

The function has two arguments:

1. **Receiver:** a pointer whose first machine word is used as a dispatch-table base.
2. **Source:** a nullable pointer to bytes.

The receiver is not checked for null. Every normal branch eventually accesses its dispatch information. A valid receiver and valid selected dispatch entry are therefore preconditions for ordinary completion, subject to changes made by dependencies before those accesses.

The source has three relevant initial states:

- Null.
- Non-null, with a zero first byte.
- Non-null, with a nonzero first byte.

For a non-null source, its first byte must be readable when the initial classification occurs. This requirement alone is not sufficient for the nonzero branch: subsequent dependencies receive that pointer and may impose additional requirements that are not available here.

No source length, capacity, ownership flag, or encoding is explicitly supplied.

### Result

The recovered interface has no explicit return value. The function returns after its selected indirect operation completes, except that the nonzero-source branch can return immediately when its receiver-selected operation produces zero.

There is no explicit status, error object, or byte count returned to the caller. Return values from the null/empty-path operation and the final indirect dependency are not consumed by this function.

### Representation and widths

The supplied representation uses:

- An eight-bit byte read for initial source classification.
- Integer-valued results from F-5cb57bbb2e0e and the nonzero-path receiver operation.
- Dispatch entries at byte offsets 8 and 12 from a base obtained through the receiver.
- Integer zero and one in the nonzero-path computation.

A 32-bit integer and pointer model is strongly suggested by the recovered representation and dispatch layout, but exact target ABI details are **documented_only**. The offsets 8 and 12 are established; interpreting them as particular numbered entries requires a pointer-size and table-layout assumption.

The first source byte is treated as unsigned before its zero/nonzero classification. Consequently, values from 128 through 255 follow the same branch as values from 1 through 127.

The integer increment is not established to use a size-specific unsigned type. Checked arithmetic, saturation, widening, and a defined wraparound policy must not be inferred.

## Behavior

### 1. Initial classification

If the source is null, no source byte is read. Execution selects the null/empty path.

If the source is non-null, the function reads its first byte before dereferencing the receiver for dispatch:

- A zero byte selects the null/empty path.
- Any nonzero byte selects the nonzero-source path.

This is an initial classification, not a continuing invariant. The function does not visibly repeat the first-byte test after invoking a dependency.

For example, a dependency may change an initially nonzero byte to zero, but that does not cause this invocation to switch to the null/empty path.

### 2. Null/empty path

The function obtains a dispatch-table base through the receiver, selects the indirect entry at byte offset 12, and invokes it with the receiver as its sole explicit argument.

After that operation completes, the function returns.

On this path:

- F-5cb57bbb2e0e is not invoked.
- The dispatch entry at byte offset 8 is not invoked.
- The final indirect dependency is not invoked.
- No arithmetic involving a dependency result occurs.

The source pointer itself is not passed to the selected operation. Thus null and non-null/zero-byte inputs share the same explicit downstream call shape. They differ in that the latter requires a successful initial byte read.

The selected operation may still distinguish surrounding state through mechanisms outside this function’s explicit arguments. Equivalent final receiver state is therefore not guaranteed.

### 3. Nonzero-source path: obtain an integer

The function first invokes F-5cb57bbb2e0e with the original source pointer.

Its integer result is increased by one for use as an argument to the next operation. No range check, negative-result check, or overflow check is visible.

The available evidence does not establish what this integer measures. In particular, it does not independently establish that the result is a string length, that a terminating zero was found, or that adding one reserves terminator space.

F-5cb57bbb2e0e completes before the function reads the receiver’s dispatch information for the next operation. Its effects on reachable memory or global state cannot be assumed absent.

### 4. Nonzero-source path: receiver operation

After F-5cb57bbb2e0e returns, the function obtains the receiver’s dispatch-table base and selects the indirect entry at byte offset 8.

It invokes that entry with these explicit arguments, in order:

1. The original receiver pointer.
2. The integer result from F-5cb57bbb2e0e plus one.
3. Integer zero.

The receiver pointer is the same pointer value supplied by the caller. Its contents, including its dispatch-table base, need not remain unchanged across the preceding dependency call.

The selected operation returns an integer-valued result. This result is tested for equality with zero.

- If zero, F-438680ffebfe returns immediately.
- If nonzero, execution continues to the final indirect dependency.

Zero is the only result rejected by this test. No positivity check, pointer validation, capacity check, or comparison with the requested quantity is visible.

### 5. Nonzero-source path: final indirect dependency

When the receiver operation produces a nonzero result, the function invokes a further indirect dependency through a separately stored target.

Its explicit arguments are:

1. The nonzero result from the receiver operation.
2. The original source pointer.

No explicit length is passed at this stage.

The dependency’s result is not used. The function returns after that invocation completes.

The target is selected at this later stage. Nothing in the supplied evidence establishes that earlier dependencies preserve its stored value.

The source argument retains its original pointer value, but its contents are not snapshotted. Any changes made by earlier dependencies may therefore affect what the final dependency observes.

## Side effects

### Direct accesses and ordering

The function itself visibly performs reads, dispatches, and arithmetic. No direct store to the receiver, source, returned destination, or global state appears in the supplied representation.

The established high-level access and call ordering is:

- Initial source-byte classification, when the source is non-null.
- Either the offset-12 receiver operation, or F-5cb57bbb2e0e followed by the offset-8 receiver operation.
- The final indirect dependency only after a nonzero result from the offset-8 operation.

On the nonzero path, receiver dispatch information is read after F-5cb57bbb2e0e completes. The final indirect target is used after the offset-8 operation completes. Neither must be treated as a target captured at function entry.

The source is not directly scanned by this function beyond the initial byte read. Any further source reads belong to dependencies.

### Ownership and lifetime

No ownership transfer is explicitly expressed by the interface.

The evidence does not establish whether any dependency:

- Allocates, resizes, reuses, or releases storage.
- Retains the receiver or source pointer.
- Copies from or writes through the source pointer.
- Treats the receiver-operation result as an address, handle, or another value.
- Requires the source and destination to be disjoint.

The string-copy-like interpretation of the final call is insufficient to establish any of those guarantees.

The function performs no visible release or rollback after the offset-8 operation returns zero. Any state changes already made by that operation remain subject to its own behavior.

### Errors and abnormal completion

There is no visible recovery for:

- An unreadable non-null source.
- An invalid receiver or dispatch table.
- A missing or invalid indirect target.
- Invalid memory passed onward to a dependency.
- Arithmetic outside the defined range of the recovered integer operation.
- A dependency that faults, does not return, or otherwise exits abnormally.

The zero-result branch is ordinary control flow, not proof that zero specifically denotes allocation failure.

## Dependencies

Only F-5cb57bbb2e0e has a supplied resolved identity. The other opaque IDs below are local labels for unresolved dependency roles, not claims of distinct implementation identities.

| Opaque ID | Selection | Explicit inputs | Result used here |
|---|---|---|---|
| F-5cb57bbb2e0e | Direct dependency on the nonzero-source path | Original source pointer | Integer, increased by one |
| F-unknown-01 | Receiver dispatch entry at byte offset 8 | Receiver, incremented integer, zero | Integer tested for zero; forwarded if nonzero |
| F-unknown-02 | Receiver dispatch entry at byte offset 12 | Receiver | Not consumed |
| F-unknown-03 | Separately stored indirect target | Nonzero result from F-unknown-01, original source pointer | Not consumed |

No preservation guarantees are supplied for these dependencies. In particular, none may be assumed to preserve globals, receiver contents, source contents, or storage ownership.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence. They have not been executed.

### Null source

Given a valid receiver and a null source:

- No source byte is read.
- F-unknown-02 is invoked with the receiver.
- F-5cb57bbb2e0e, F-unknown-01, and F-unknown-03 are not invoked.
- If F-unknown-02 returns normally, this function returns.

No particular cleared or empty receiver state is required without a contract for F-unknown-02.

### Non-null source beginning with zero

Given a readable source whose first byte is zero:

- The first byte is read.
- The same explicit receiver-operation path as the null-source case is selected.
- Bytes beyond the first are not directly examined by this function.

This does not guarantee that the selected dependency avoids all other memory reads.

### Nonzero source and zero receiver-operation result

Given an initially nonzero first byte, and assuming F-5cb57bbb2e0e returns 7:

- F-unknown-01 receives the receiver, 8, and zero.
- If it returns zero, F-unknown-03 is not invoked.
- The function returns without a visible cleanup operation.

### Nonzero source and nonzero receiver-operation result

Given an initially nonzero first byte, a helper result of 3, and a nonzero result from F-unknown-01:

- F-unknown-01 receives the receiver, 4, and zero.
- F-unknown-03 receives that operation’s result and the original source pointer.
- No explicit size accompanies this final invocation.

### High-bit first byte

Given a first byte of 255, the nonzero-source path is selected. The byte’s high bit does not make the input empty or invalid within this function’s initial test.

### Source mutation during processing

Given an initially nonzero byte that F-5cb57bbb2e0e changes to zero:

- Execution remains on the nonzero-source path.
- If F-unknown-01 returns nonzero, F-unknown-03 receives the original pointer to the now-modified source storage.

### Receiver dispatch mutation

If F-5cb57bbb2e0e changes the receiver’s dispatch-table base and returns normally, the subsequent offset-8 dispatch uses the receiver state read after that call. The initial dispatch base is not visibly cached.

### Integer boundaries

If F-5cb57bbb2e0e returns −1 and the recovered integer interpretation applies, the next operation receives zero as its second argument. There is no visible rejection of that helper result.

If it returns the maximum representable signed integer, the increment’s outcome is **documented_only**. This contract does not choose wraparound, trapping, or another overflow behavior.

## Uncertainties

- **documented_only — ABI:** exact pointer width, integer width, calling conventions, and integer-to-address interpretation require target-level confirmation.
- **documented_only — helper semantics:** F-5cb57bbb2e0e is not established to compute a length or validate a string.
- **documented_only — receiver operations:** reservation, allocation, reset, release, and logical-size effects are unknown.
- **documented_only — final dependency:** copying, termination, overlap handling, and destination capacity requirements are unknown.
- **documented_only — indirect control flow:** the supplied representation treats unresolved indirect transfers as calls. Their exact low-level return mechanics are not established.
- **documented_only — arithmetic overflow:** no checked or portable overflow policy is demonstrated.
- **documented_only — dependency effects:** mutation, retention, ownership transfer, reentrancy, and abnormal completion are unspecified.

The inferred content-replacement purpose must not be used to fill these gaps.

## Evidence

Established static evidence consists of the nullable source check, one-byte initial classification, the direct invocation of F-5cb57bbb2e0e on the nonzero path, the addition of one to its integer result, receiver-selected operations at byte offsets 8 and 12, the zero-result early return, and the conditional final indirect invocation.

The supplied representation also establishes the explicit argument order and the absence of visible direct stores or cleanup operations in this function. It does not establish the internal behavior of any dependency.

No runtime tests have been performed. Acceptance examples describe conditional consequences of the static evidence, not observed execution results.
