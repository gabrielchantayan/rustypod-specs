<!-- Trusted provenance: F-36f8c0b9b7ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-36f8c0b9b7ad

## Purpose

F-36f8c0b9b7ad conditionally processes an object, may mark its state as `2`, and then performs a shared completion check that can invoke an indirect dependency.

The established behavior is control-flow and memory-access behavior. The meanings of the object, its state values, its control bytes, and the dependent operations are not established. In particular, terms such as destruction, cancellation, flushing, locking, or notification would impose semantics not justified by the supplied evidence.

The function has three broad outcomes:

- Proceed directly to the completion check without an explicit state write.
- Process the object, explicitly write state `2`, and then perform the completion check.
- Invoke a dependency identified as nonreturning, without reaching the state write or completion check through the observed flow.

These outcomes are conditional on values read after earlier dependency calls. Entry-time field values alone do not determine behavior when those calls can change memory.

## Interface

### Input and result

The function accepts one object address. The recovered interface represents this argument as a 32-bit integer used as a base address.

No return value is exposed. On ordinary completion, control returns to the caller. Some paths invoke F-a817d8d58c10, which is identified in the supplied static evidence as nonreturning.

The object is accessed through fixed byte offsets:

| Offset | Access width | Established use |
|---|---:|---|
| `+0x18` | 32-bit pointer in the recovered model | Address of this slot is passed to the first dependency. Its contents are subsequently read during the completion check. |
| `+0x20` | 8 bits | State discriminator; explicitly overwritten with `2` on processing paths that reach that write. |
| `+0xE9` | 8 bits | Zero/nonzero condition controlling a dependency call. |
| `+0xEA` | 8 bits | Zero/nonzero condition controlling processing and a possible nonreturning call. |
| `+0xEC` | 8 bits, interpreted as signed | Selector, including a distinguished value of `−1`. |
| `+0xF0` | 32 bits | Value tested for zero and potentially supplied to a dependency. |

The full object size, field names, alignment requirements, and the meanings of unlisted bytes are unknown. There is no basis for declaring a packed structure or assigning ownership to these fields.

### Preconditions

The function performs no explicit null, range, alignment, or accessibility validation.

For a path to execute without an invalid memory access:

- The object address and any accessed offsets must identify readable storage.
- The state byte must be writable if execution reaches the explicit state update.
- A nonnull pointer obtained from `+0x18` must identify storage readable as a 32-bit word.
- Each invoked dependency must receive whatever additional validity its own contract requires.

The first dependency receives the address of the `+0x18` slot, not the pointer value stored in that slot.

The recovered model uses 32-bit integer and pointer accesses and byte-sized flag accesses. The signed selector is promoted to an integer for one dependency call. Its `−1` sentinel corresponds to byte value `0xFF` in that model. The contract does not generalize these widths to an unrelated execution ABI.

### Ownership

No explicit allocation, deallocation, reference-count adjustment, ownership transfer, or pointer replacement is visible in this function.

Dependencies may have such effects, but their ownership contracts are not supplied. In particular, passing the address of the pointer slot does not by itself establish that the first dependency allocates, releases, or replaces anything.

## Behavior

### 1. Unconditional initial dependency

The first observable operation is a call to F-338d16f9ce72 with the address at object offset `+0x18`.

Only after this call returns does the function read the state byte at `+0x20`. Therefore, all subsequent state decisions use the post-call byte, not necessarily its entry value.

If the dependency does not return, no subsequent behavior described below is reached.

### 2. State-based routing

For a state byte of `0` or `2`, the function goes directly to the completion check. It does not explicitly read `+0xF0`, `+0xE9`, `+0xEA`, or `+0xEC` along this route and does not explicitly update the state.

For every other state byte, it reads the 32-bit value at `+0xF0`.

If that value is zero:

- State `1` goes directly to the completion check, without an explicit state write.
- Any state other than `0`, `1`, or `2` invokes F-a817d8d58c10.

The latter route does not inspect the control bytes at `+0xE9`, `+0xEA`, or `+0xEC` before invoking F-a817d8d58c10.

If the value at `+0xF0` is nonzero, processing continues below. This includes state `1` and every other state not already routed directly to completion.

### 3. Optional preparation and the working value

With a nonzero value read from `+0xF0`, the function reads the byte at `+0xE9`.

- If this byte is nonzero, no preparation dependency is called. The previously read 32-bit value remains the working value.
- If this byte is zero, the function calls F-9050aa7a4a59 with the object address.

After F-9050aa7a4a59 returns, the working value is replaced by a value shown in the supplied evidence as a secondary output register. Its meaning and formal calling convention are not established.

Consequently, a later F-052494b1687e call is not guaranteed to receive the value originally read from `+0xF0`. Nor is its second argument guaranteed to remain nonzero after the preparation call. There is no intervening zero check on the replacement value.

The function then establishes a 32-bit local processing indicator with value `0`. This indicator is later changed to `1` only after F-052494b1687e returns.

### 4. Selector and control-byte processing

The function next reads the signed byte at `+0xEC`.

#### Initial selector is not `−1`

The function reads `+0xEA`.

If `+0xEA` is nonzero:

- It bypasses F-052494b1687e.
- It rereads `+0xEC`.
- It calls F-6fcd85a213a6 with the object address, the newly read signed selector promoted to an integer, and indicator `0`.

This route does not apply the `−1` sentinel check to the reread selector. If that reread yields `−1`, the observed flow passes `−1` to F-6fcd85a213a6.

If `+0xEA` is zero:

- It calls F-052494b1687e with the object address, the working 32-bit value, and a third argument of `0`.
- After that call returns, it changes the processing indicator to `1`.
- It rereads `+0xEA`.

When this reread of `+0xEA` is zero, no F-6fcd85a213a6 call is made, and `+0xEC` is not reread on this route.

When this reread of `+0xEA` is nonzero, the function reads `+0xEC` again:

- If the selector is `−1`, it invokes F-a817d8d58c10.
- Otherwise, it reads `+0xEC` once more and calls F-6fcd85a213a6 with that latest signed value and indicator `1`.

The selector used as an argument is therefore a separate read from the selector used for the sentinel check.

#### Initial selector is `−1`

The function does not call F-052494b1687e. It reads `+0xEA`.

If `+0xEA` is zero, it makes no F-6fcd85a213a6 call.

If `+0xEA` is nonzero, it rereads `+0xEC`:

- A value of `−1` invokes F-a817d8d58c10.
- Any other value permits a further read of `+0xEC`, followed by F-6fcd85a213a6 with that latest signed value and indicator `0`.

As above, the value passed to the dependency comes from a separate read, not necessarily the read that passed the sentinel check.

### 5. State update

Every processing route that reaches this point writes the single byte value `2` to `+0x20`.

This write occurs:

- After any returning F-052494b1687e call.
- After any returning F-6fcd85a213a6 call.
- Even when the selector/control-byte combination caused neither of those dependencies to be called.

The function does not reread the state before overwriting it. A dependency’s earlier change to that byte is therefore overwritten if execution reaches this write.

The direct-to-completion routes do not perform this explicit write.

### 6. Shared completion check

The completion check reads the pointer stored at `+0x18`.

- If it is null, the function returns without dereferencing it or invoking the indirect dependency.
- If it is nonnull, the function reads its first 32-bit word.
- If that word is zero, the function returns.
- If that word is nonzero, the function invokes an indirect target obtained from a separate global slot.

The pointed-to word is a condition, not the function address used for the indirect invocation. The evidence does not establish an object-method dispatch through that word.

No explicit arguments are shown for this indirect call. Its actual ABI requirements, return behavior, and effects are unknown. If it returns normally, this function returns without further explicit object reads or writes.

## Side effects

The only direct object write established by the body is the byte write of `2` at `+0x20`.

Dependency calls can introduce additional effects. The supplied evidence does not establish that they preserve:

- Object fields or reachable storage.
- The pointer slot at `+0x18`.
- The separate global indirect-target slot.
- Other global state.
- Memory ownership or lifetime.

Read order is significant. In particular:

- The first dependency precedes the state read.
- F-9050aa7a4a59 precedes selector/control-byte reads.
- F-052494b1687e precedes renewed reads of `+0xEA` and, conditionally, `+0xEC`.
- The explicit state write precedes the completion pointer read.
- The pointed-to word is read before invoking the global indirect target.

No synchronization, atomic access, memory barrier, exception handling, recovery operation, or explicit error result is established.

Concurrent mutation is not specified as supported. The separate reads described above must not be collapsed into a claim that the function operates on a single coherent object snapshot.

## Dependencies

| Opaque ID | Observed interface and role |
|---|---|
| F-338d16f9ce72 | Unconditionally receives the address of the object’s `+0x18` slot before other explicit reads. Return value is not used. |
| F-9050aa7a4a59 | Receives the object address when `+0xF0` was nonzero and the subsequently read `+0xE9` byte is zero. A secondary output register supplies the later working value. |
| F-052494b1687e | Receives the object address, the working 32-bit value, and `0`. Called only through the initial non-sentinel-selector and zero-control-byte route. |
| F-6fcd85a213a6 | Receives the object address, a sign-extended selector byte, and indicator `0` or `1`. Indicator `1` means F-052494b1687e returned earlier in this invocation. |
| F-a817d8d58c10 | Invoked on the specified invalid-state/value or sentinel/control combinations. Identified as nonreturning in the supplied evidence; error mechanism is unknown. |
| Unidentified indirect dependency | Invoked through a separate global target when the completion pointer and its first word are both nonzero. No opaque ID or argument contract is supplied. |

The supplied mapping also associates F-599da3979baf with a source-level label. That label marks the shared completion region in the provided body; it does not establish a separate call to F-599da3979baf. No dependency semantics are assigned to it here.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless otherwise stated, dependencies return normally, accessed storage is valid, and field values remain stable between the reads relevant to each example.

1. **State `0`, null completion pointer.**  
   After F-338d16f9ce72 returns, the state is `0` and the pointer at `+0x18` is null. The function performs the completion pointer read and returns. It does not explicitly read the later processing fields or write state `2`.

2. **State `2`, enabled completion invocation.**  
   The post-initial-call state is `2`; the completion pointer is nonnull and its first word is nonzero. The function invokes the global indirect target. No explicit state write occurs.

3. **State `1`, zero processing value.**  
   The state is `1` and `+0xF0` contains zero. Processing dependencies are bypassed. The completion check still occurs.

4. **Other state, zero processing value.**  
   The state is `3` and `+0xF0` contains zero. F-a817d8d58c10 is invoked without reading the three processing-control bytes. The observed flow does not reach completion.

5. **Non-sentinel selector with control byte staying zero.**  
   A processing route has a nonzero `+0xF0`, nonzero `+0xE9`, selector `5`, and zero `+0xEA`. F-052494b1687e receives the value read from `+0xF0` and third argument `0`. If `+0xEA` remains zero after that call, F-6fcd85a213a6 is skipped. State `2` is then written, followed by completion.

6. **First processing dependency enables the second.**  
   Under the preceding initial conditions, F-052494b1687e changes `+0xEA` to nonzero and leaves the selector at `5`. F-6fcd85a213a6 receives selector `5` and indicator `1`. State `2` is written only after it returns.

7. **Initially enabled control byte.**  
   A processing route has selector `5` and nonzero `+0xEA`. F-052494b1687e is skipped. F-6fcd85a213a6 receives the reread selector and indicator `0`.

8. **Sentinel with disabled control byte.**  
   A processing route has selector `−1` and zero `+0xEA`. Neither processing dependency is invoked at this stage. The function still writes state `2` and performs completion.

9. **Sentinel with enabled control byte.**  
   A processing route has selector `−1`, nonzero `+0xEA`, and the sentinel reread remains `−1`. F-a817d8d58c10 is invoked before the state write.

10. **Preparation replaces the working value.**  
    A nonzero `+0xF0` and zero `+0xE9` cause F-9050aa7a4a59 to run. If its secondary output is zero and subsequent conditions select F-052494b1687e, that dependency receives zero as its second argument. The original nonzero test is not repeated.

## Uncertainties

The following are **documented_only**, not inferred guarantees:

- Semantic meanings of all object fields and state values.
- Valid state-byte range beyond the comparisons explicitly observed.
- The formal ABI and meaning of F-9050aa7a4a59’s secondary output.
- Dependency effects on globals, memory, ownership, and object lifetime.
- Identity and ABI of the indirect completion target.
- Whether F-a817d8d58c10 terminates execution, transfers control, raises an error, or otherwise fails to return.
- Behavior if a dependency unexpectedly returns contrary to a nonreturning annotation.
- Behavior under invalid storage, address overflow, misalignment, concurrent mutation, or unsupported ABI assumptions.

**Inferred intent:** writing state `2` after processing suggests a transition marker, and the final pointer/word check suggests conditional completion signaling. Neither interpretation establishes resource-management or lifecycle semantics.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes the fixed offsets, recovered access widths, comparisons, conditional dependency calls, separate repeated reads, explicit state write, and shared completion check. The nonreturning characterization and secondary-register output originate from that static representation and retain the limitations stated above.

No dependency implementations, independent ABI documentation, or runtime observations were supplied. No runtime tests have been performed.
