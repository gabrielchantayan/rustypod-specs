<!-- Trusted provenance: F-d9e78f45a0bc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d9e78f45a0bc

## Purpose

F-d9e78f45a0bc updates shared byte-sized state, conditionally invokes an operation using a temporary buffer, and, when that operation returns nonzero, waits for readiness before dispatching an eight-bit value.

The early-return decision occurs only after three dependency calls and an initial state write. It is not a side-effect-free duplicate-request check. When execution continues, a second state write occurs before the operation’s result is known; there is no visible rollback.

**Inferred intent:** the two state bytes may describe requested and remembered modes, and the final dispatch may apply an operation-produced value. These interpretations are not established requirements.

This contract describes static evidence from the supplied representation. It does not establish runtime behavior of dependencies, machine-level memory-load counts, or synchronization guarantees.

## Interface

The recovered interface has four inputs and no returned value.

| Input | Recovered width and interpretation | Visible use |
|---|---|---|
| First | Signed 32-bit integer | Converted for byte-sized state writes; compared as a full signed value against 3 and 4. |
| Second | Opaque 32-bit value | Not read by the visible body. |
| Third | Opaque 32-bit value | Not read by the visible body. |
| Fourth | Unsigned 32-bit integer | Initializes word 0 of a temporary two-word buffer. |

These widths follow the recovered representation, not an independently supplied ABI specification. The second and third inputs have no visible influence on this body’s decisions or dependency arguments.

### Shared storage

Two shared values provide access to state:

- **State-base global:** a pointer-valued shared object. One source-level reference captures its value for later offset-1 accesses. A subsequent, separate source-level reference supplies the address for the initial offset-0 write.
- **Readiness-base global:** a shared value captured after the buffer operation returns. On the nonzero-result path, the byte at offset 2 from that captured base controls waiting.

For the captured state pointer, the only directly accessed field is byte offset 1. The initial offset-0 write instead uses the state-base global expression directly. The supplied representation does not establish that these two references produce distinct machine loads or different pointer values.

The full layouts, ownership, allocation sizes, lifetimes, and application meanings of the referenced objects are unavailable.

### Temporary buffer

The operation receives the address of a local buffer containing two 32-bit words, occupying eight bytes under the recovered widths.

- Word 0 is initialized from the fourth input.
- Word 1 is not initialized by this body.
- Only word 0 is read by this body after the operation.
- Buffer word indexing is distinct from byte indexing. Selecting the low eight bits of word 0 does not establish that the first byte in memory is selected on every target byte order.

### Preconditions

Defined execution requires valid storage and compatible dependency contracts for every reached access:

- The state-base value captured for offset-1 accesses must remain valid whenever those accesses occur.
- The state-base global expression used for the offset-0 write must identify writable byte storage at that point.
- On the waiting path, the captured readiness base must support the address calculation and byte access at offset 2.
- Dependencies must accept the supplied arguments and respect the temporary buffer’s bounds, initialization, and lifetime constraints.
- Concurrent access must satisfy the target’s memory and synchronization rules.

The body performs no visible null, bounds, lifetime, or overflow checks.

## Behavior

### Initial call, pointer capture, and first write

The first visible action is a call to F-c979b3c48e40 with no explicit arguments.

After that call returns, the body performs these source-level actions in order:

1. It obtains the state-base global’s value and captures that pointer for subsequent offset-1 accesses.
2. It separately references the state-base global expression and writes a character-converted form of the first input to byte offset 0 through that expression.
3. It calls F-fa1e7c5a0d15 with the Boolean result of comparing the full signed first input with 3.

F-fa1e7c5a0d15 therefore receives 1 when the first input is greater than 3, and 0 otherwise. This comparison does not use the byte written to state.

The two state-base references must remain distinct in the source-facing contract. The offset-0 write is not explicitly made through the previously captured pointer. Conversely, this distinction does not prove separate machine loads, an intervening change, or different destination objects.

F-c979b3c48e40 can affect the shared state-base value before both references. No intervening dependency call is visible between the pointer capture and the offset-0 write.

The character conversion used for the write has implementation-dependent aspects for first-input values outside the representable range of the target’s plain character type. The apparent intent is to retain a byte-sized representation, but exact out-of-range conversion semantics are not independently established.

### Classification using captured state

After F-fa1e7c5a0d15 returns, the body reads byte offset 1 through the captured state pointer. It interprets that byte through the recovered plain-character type, converts it to an integer, and passes the result to F-c39b71dff8cc.

For high-bit-set byte values, the argument depends on plain-character signedness. For example, raw byte `0xFF` can produce −1 under signed eight-bit character semantics or 255 under unsigned semantics.

The pointer used here is the captured pointer. A dependency’s later reassignment of the state-base global does not, by itself, redirect this access. Dependencies can nevertheless modify the captured object or invalidate its storage; pointer capture does not preserve its contents or lifetime.

F-c39b71dff8cc’s result is treated as an unsigned 32-bit value.

### Early-return decision

The first condition compares the complete unsigned classification result with the Boolean result of the signed comparison “first input greater than 4.”

Consequently:

- For first input at most 4, only classification result 0 satisfies this equality.
- For first input greater than 4, only classification result 1 satisfies it.
- Other results do not qualify merely because they are nonzero.
- The classification result is not compared with the numeric value 4.

Only when this equality succeeds does the body make a fresh read of byte offset 1 through the captured pointer and compare that byte expression with −1. If this comparison is unequal, the function returns immediately.

The second offset-1 read occurs after F-c39b71dff8cc returns. It may therefore observe a value different from the byte supplied to that dependency. If the classification equality fails, this second read is skipped.

**Unresolved sentinel semantics:** the classification argument explicitly passes through a plain-character interpretation, but the later sentinel comparison does not establish the same interpretation. If its byte expression is unsigned and promotes to a wider signed integer, raw `0xFF` becomes 255 and compares unequal to −1. If it is interpreted as signed eight-bit data, raw `0xFF` can compare equal to −1 and prevent early return. The supplied representation does not settle this distinction.

An early return skips the second state write, temporary-buffer operation, readiness-base capture, waiting, and dispatch. It does not undo or skip the preceding calls and offset-0 write.

### Continuing execution and buffer operation

If the early return is not taken, the body writes the character-converted first input to byte offset 1 through the captured state pointer.

This access has the explicit captured-pointer guarantee. Reassigning the state-base global since capture does not alone change its destination. The same character-conversion uncertainty applies as for the offset-0 write.

Next, the body initializes word 0 of the temporary buffer from the fourth input, leaves word 1 uninitialized, and calls F-f864d4b0c549 with:

1. The integer 4.
2. The address of the two-word buffer.

The meaning of 4 is unknown. Its presence does not establish a byte count, word count, operation code, or permitted buffer extent.

The offset-1 state write precedes this call. There is no visible restoration if the dependency returns zero, fails to return, or leads to indefinite waiting.

### Result handling and readiness capture

After F-f864d4b0c549 returns, the body obtains and captures the readiness-base global. This source-level access occurs before the branch on the operation’s result, including when that result is zero.

The operation result is represented as a signed 32-bit integer and is used only for a zero/nonzero decision:

- Zero: return without reading the readiness byte or dispatching.
- Nonzero, including a negative result: select the low eight bits of post-call buffer word 0 and proceed to readiness checking.

Because the dependency receives the buffer’s address, word 0 after the call need not equal the fourth input. The selected dispatch value lies between 0 and 255 inclusive. Higher bits of the word do not contribute.

This selection occurs after readiness-base capture and before the readiness loop. The value is not selected again after waiting calls.

### Readiness loop and dispatch

The body reads the byte at offset 2 from the captured readiness base.

- If it is nonzero, execution proceeds directly to dispatch.
- If it is zero, the body calls F-unmapped-01 with arguments 0 and 1, then checks the readiness byte again.
- Each observed zero therefore leads to one such call before the next check, provided the call returns normally.

F-unmapped-01 is an opaque identifier assigned here to the dependency absent from the supplied mapping. It conveys no known external identity or semantics.

The loop uses the captured readiness base. A later change to the readiness-base global does not, by itself, redirect it. The address calculation includes an offset of 2; the representation does not independently establish overflow or address-wrap behavior.

No timeout, iteration bound, cancellation check, or alternate exit is visible. Readiness observations that remain zero can prevent normal return indefinitely.

After observing a nonzero readiness byte, the body calls F-b4c8a64376c1 with the previously selected value in the range 0–255 and the integer 0. It then returns without exposing a success status or visibly processing a result from that call.

## Side effects

### Direct writes and observations

The direct shared-state writes are:

- An offset-0 write through the state-base global expression, after a separate capture of that global’s value.
- An offset-1 write through the captured pointer, only if early return is not taken.

These writes are not guaranteed by the representation to target the same underlying object, although no difference between their bases is established either.

The offset-1 write is not conditional on operation success. Both direct state writes precede any readiness waiting.

Other visible observations include the classification read, the conditionally evaluated sentinel read, the readiness-base capture even after a zero operation result, and repeated readiness-byte checks on the nonzero path.

### Dependency effects and ownership

No direct allocation or release appears in this body. That does not establish that dependencies avoid allocation, release, callbacks, or other state changes.

No dependency is shown to preserve globals, referenced storage, or temporary-buffer contents. No dependency is guaranteed to return normally.

The temporary buffer belongs to this invocation’s local storage. Passing its address does not establish ownership transfer or permission to retain it beyond its valid lifetime. Any retention or asynchronous access would require a separate compatible contract.

Neither captured base establishes ownership or keeps referenced storage alive.

### Errors and progress

There is no explicit error output or local recovery path.

A zero operation result suppresses waiting and dispatch, but whether it denotes failure, refusal, or another application outcome is unknown. Invalid storage, incompatible buffer access, dependency failure, and permanent nonreadiness are not locally handled.

Repeated reads and external calls do not by themselves establish atomicity, synchronization, visibility across threads, or eventual progress.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-c979b3c48e40 | Called first without explicit arguments, before both source-level state-base references. |
| F-fa1e7c5a0d15 | Receives 1 exactly when the full signed first input exceeds 3; called after the offset-0 write. |
| F-c39b71dff8cc | Receives the integer conversion of a plain-character interpretation of captured-state byte offset 1. Its unsigned 32-bit result participates in early return. |
| F-f864d4b0c549 | Receives 4 and the temporary buffer address. Word 0 is initialized; word 1 is not. Its zero/nonzero result controls further processing. |
| F-unmapped-01 | Receives 0 and 1 once for each observed zero readiness byte, assuming execution reaches and returns from the call. |
| F-b4c8a64376c1 | Receives the selected post-operation low-byte value and 0 after readiness is observed nonzero. |

## Acceptance examples

These are conditional static examples, not executed tests. They assume valid storage and normally returning dependencies unless stated otherwise.

1. **Distinct state-base expressions.**  
   F-c979b3c48e40 returns; the body captures the state-base value, then uses a separate state-base global expression for the offset-0 write. Subsequent offset-1 accesses use the captured pointer. Acceptance does not require two machine loads or different pointer values.

2. **Later global reassignment.**  
   If F-fa1e7c5a0d15 changes only the state-base global to another valid object, the classification read still uses the previously captured object. This does not establish that the original object’s contents or lifetime were preserved.

3. **Input 3 and early return.**  
   F-fa1e7c5a0d15 receives 0. If classification returns 0 and the fresh sentinel comparison is unequal to −1, the function returns without its direct offset-1 write or buffer-operation call. The initial offset-0 write has already occurred.

4. **Input 4 uses different thresholds.**  
   F-fa1e7c5a0d15 receives 1, while the classification equality still requires result 0.

5. **Input 5 requires exact result 1.**  
   Classification result 1 satisfies the equality portion. Result 2 does not, so it skips the sentinel read and continues to the offset-1 write.

6. **Fresh sentinel observation.**  
   If classification matches and its dependency changes captured-state byte offset 1, the sentinel check uses the post-call byte. For raw `0xFF`, whether this prevents early return remains conditional on unresolved byte-promotion semantics.

7. **Zero operation result.**  
   If the operation returns zero, the offset-1 write has already occurred and the readiness-base global is still accessed. The readiness byte is not examined, and neither waiting nor final dispatch occurs.

8. **Post-call value and negative result.**  
   If the operation leaves word 0 equal to `0x1234ABCD`, returns −1, and readiness is immediately nonzero, final dispatch receives 205 and 0. The negative result follows the nonzero path.

9. **Delayed readiness.**  
   Readiness observations of zero, zero, then nonzero produce two calls to F-unmapped-01 before dispatch. The dispatch value remains the one selected before those calls.

10. **Persistent nonreadiness.**  
    If every readiness observation is zero and every waiting call returns normally, final dispatch and normal return are never reached.

## Uncertainties

The following remain **documented_only**:

- Exact ABI details beyond the recovered widths.
- Plain-character signedness and out-of-range conversion behavior.
- The type and promotion rules governing the −1 sentinel comparison.
- Whether the two state-base references correspond to separate machine loads or can observe different pointer values.
- Shared-object layouts, validity, ownership, lifetime, aliasing, and synchronization.
- Address arithmetic and overflow behavior for the readiness offset.
- Dependency side effects, failure conventions, and termination.
- Whether the buffer operation reads the uninitialized second word.
- Buffer retention permissions and the meaning of argument 4.
- The identity and behavior of F-unmapped-01.
- The mechanism that changes readiness and any visibility or progress guarantees.

## Evidence

The supplied restricted representation and dependency-ID mapping establish the source-level call order, two ordered state-base references, captured-pointer offset-1 accesses, comparison thresholds, conditional sentinel read, buffer initialization, readiness-base capture, low-eight-bit selection, waiting structure, and final argument values.

The initial offset-0 write is explicitly associated with the separate state-base global expression, not attributed to the captured pointer. No machine-load count or pointer-value difference is inferred from that distinction.

Application meanings are identified as inferred intent. Unavailable type, dependency, ownership, and runtime properties remain documented uncertainties.

No runtime tests have been performed.
