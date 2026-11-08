<!-- Trusted provenance: F-dd8ffe218bef; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dd8ffe218bef

## Purpose

F-dd8ffe218bef dispatches an unsigned 32-bit selector to polling, status-publication, or control-write operations.

Its return value has selector-dependent meaning:

| Selector | Return value |
|---|---|
| 0 | A five-bit translation of a matching response, or zero after five zero responses. |
| 1 or 2 | Zero after two dependency calls and a control write. |
| 3 | A 32-bit value read from completion base B, independently of the later read used to select the completion-write destination. |
| 4 | Zero after a direct control write. |
| Any other value | The selector unchanged. |

A command/response purpose is inferred from the repeated request calls, response matching, and packed control writes. The supplied representation does not establish a particular device, protocol, or interpretation of the flags.

This contract distinguishes explicit behavior from dependency effects. A dependency may change shared state; preservation of shared addresses, arguments, or storage lifetimes is not established.

## Interface

**Input:** One unsigned 32-bit selector.

**Output:** One unsigned 32-bit value. Selector 3 returns an address-like shared value, not an allocation result. Returning it does not establish ownership transfer.

**Shared inputs and destinations:**

| Shared role | Explicit access |
|---|---|
| Request word A | Read once after the initial selector-0 calls. Its captured 32-bit value is reused throughout polling. |
| Completion base A | Read after selector-0 polling. Selects a full 32-bit write at byte offset 16. |
| Request word B | Read once after the initial selector-3 calls. Its captured 32-bit value is reused throughout polling. |
| Status destination | Read after every selector-3 polling call to capture the pointer used for a possible status test. On a match, read again to select the status-write destination. These reads have distinct roles and need not yield equal pointers. |
| Notification argument | Read only when the selector-3 status test is nonzero; supplied to F-812c4cd96b3e. |
| Completion base B | Read twice after selector-3 polling and any notification: first to capture the return value, then to select the completion-write destination at byte offset 16. The two values need not agree. |
| Control base | Read after the dependency calls for selectors 1 and 2, or directly for selector 4. Selects a full 32-bit write at byte offset 512. |

The saved result of F-475ed60efd45 is an opaque 32-bit quantity. The function writes its bits unchanged; no signed interpretation or semantic meaning is needed.

**Preconditions for ordinary completion:**

- Executed dependencies must accept the indicated arguments and return normally.
- Each actually dereferenced address must support the indicated 32-bit access.
- Address formation at offsets 16 and 512 must be valid in the target environment.
- Polling must eventually encounter a matching nonzero response or accumulate five zero responses.

For selector 3, the status-write destination and status-test destination must each be valid when accessed. Validity of one does not establish validity of the other.

There is no explicit pointer validation, bounds check, alignment check, error conversion, or recovery path. The function does not establish address-width, wraparound, or fault-handling rules for invalid destination arithmetic.

## Behavior

### Common polling rules

Selectors 0 and 3 begin with an allowance of five zero responses. Each polling call invokes F-6e9c50d81ffc with the request captured before the loop.

Responses are unsigned 32-bit words:

- Zero consumes one allowance.
- A nonzero response matches when its value masked by `0x8000FFFF` equals the entire captured request.
- A nonzero nonmatch neither consumes nor replenishes the allowance.
- Polling stops at the first match or immediately after the fifth zero response.

Thus, five is a limit on total zero responses, not total calls or consecutive zero responses. An unlimited sequence of nonzero nonmatches can prevent termination.

Matching compares response bit 31 and bits 0–15. Response bits 16–30 are ignored by that comparison. The request itself is not masked: any set request bit in positions 16–30 makes matching impossible.

Zero is handled before the match comparison. Even a zero request cannot match a zero response. Conversely, a nonzero response whose retained bits are zero can match a zero request.

The captured request is reused even if a dependency changes its shared source during polling.

### Selector 0: poll and translate five flags

The explicit order is:

1. Invoke F-1605e5a48b3c without arguments.
2. Invoke F-475ed60efd45 without arguments and preserve its 32-bit result.
3. Read request word A.
4. Poll using that captured request.
5. After either loop-exit condition, read completion base A.
6. Write the saved 32-bit result at that base plus 16 bytes.
7. Return the translated response value on a match, otherwise zero.

For a matching response, the result contains only the following mapped flags:

| Response bit | Result bit | Result weight |
|---|---:|---:|
| 16 | 0 | 1 |
| 17 | 3 | 8 |
| 18 | 4 | 16 |
| 19 | 2 | 4 |
| 20 | 1 | 2 |

The result is between 0 and 31 inclusive. Response bits 21–30 do not affect it. Bit 31 participates in matching but not in translation.

A match with all five translated flags clear returns zero, so zero does not distinguish exhaustion from a successful match.

The completion-base read occurs after polling and therefore after all executed polling calls. Changes to that shared base during those calls can affect the destination. If polling never exits, the explicit completion write never occurs.

### Selector 3: poll, publish status, and optionally notify

The initial order is:

1. Invoke F-external-1 with the numeric argument 25.
2. Invoke F-1605e5a48b3c without arguments.
3. Invoke F-475ed60efd45 without arguments and preserve its 32-bit result.
4. Read request word B.
5. Enter polling with the captured request.

Every iteration has an additional shared-state access: after F-6e9c50d81ffc returns, the function reads the status-destination global and captures that pointer for a possible status test. This pointer capture occurs even when the response is zero or a nonmatch. Those two outcomes do not cause a dereference of the captured status pointer in the explicit polling logic.

For a matching nonzero response, the order is:

1. Read the status-destination global again, obtaining the destination for the status write.
2. Extract response bits 16–30 as an unsigned value.
3. Write that value as a full 32-bit word through the second pointer.
4. Read a full 32-bit word through the pointer captured immediately after the polling call.
5. If that read is nonzero, read the current notification argument and invoke F-812c4cd96b3e with it.
6. Exit polling.

The extracted status ranges from 0 through 32,767. Response bit 31 is excluded, and there is no sign extension.

The write pointer and test pointer come from separate global reads. The function does not explicitly compare them or require equality. Consequently:

- A zero extracted status does not necessarily suppress notification.
- A nonzero extracted status does not necessarily cause notification.
- Notification depends on the actual word read through the earlier pointer.

Even if both pointer reads yield the same address, memory behavior or intervening changes can distinguish the status test from simply testing the extracted value.

If the zero allowance is exhausted, the polling logic performs neither a status write nor a status dereference nor a notification call. It still captures the status pointer after every polling call, including the fifth zero response.

After either loop-exit condition, and after any notification, completion occurs in this order:

1. Read completion base B and capture its 32-bit value for return.
2. Read completion base B again to select the completion-write destination.
3. Write the saved result of F-475ed60efd45 at the second base plus 16 bytes.
4. Return the value captured by the first base read.

The completion write does not necessarily use the returned base. A dependency invoked before these reads may affect their values. If the final store aliases the shared storage containing completion base B, it does not replace the already captured return value.

### Selectors 1 and 2: identical control operations

Both selectors execute:

1. F-17d6434f9fa3 with ordered numeric arguments 116, 1, and 0.
2. F-external-1 with numeric argument 1.
3. A read of the control base.
4. A full 32-bit write of `0x000E020E` at that base plus 512 bytes.
5. Return zero.

The written word contains 14 in bits 16–31, 2 in bits 8–10, and 14 in bits 0–7. All other bits are zero.

The control base is read after both calls. Neither dependency is established to preserve it. The arguments to the first dependency do not establish any particular internal write or hardware operation.

### Selector 4: direct control operation

Selector 4 makes no explicit dependency call. It reads the control base, writes `0x000E0402` as a full 32-bit word at byte offset 512, and returns zero.

This word contains 14 in bits 16–31, 4 in bits 8–10, and 2 in bits 0–7. All other bits are zero.

The control-word calculations use small nonnegative quantities: the relevant right shifts produce 14, the masked three-bit fields produce 2 or 4, and the shifted fields do not overlap. No arithmetic overflow is needed to obtain either control word.

### Other selectors

Any selector outside 0–4 is returned unchanged. This path has no explicit dependency call or shared-state access.

## Side effects

All explicit destination writes are full 32-bit stores, not byte writes or read-modify-write operations.

| Path | Explicit writes, in order |
|---|---|
| Selector 0, either exit | Saved opaque value at completion base A plus 16 bytes. |
| Selector 1 or 2 | `0x000E020E` at control base plus 512 bytes. |
| Selector 3, match | Extracted status through the second status-pointer read; later, saved opaque value at the second completion-base read plus 16 bytes. |
| Selector 3, exhaustion | Saved opaque value at the second completion-base read plus 16 bytes. |
| Selector 4 | `0x000E0402` at control base plus 512 bytes. |
| Other selector | None. |

These are logical word values. Their byte order is not established.

Aliasing is not excluded. A status write may affect another shared value that is read later, including the notification argument or completion base B. The completion write may affect shared storage without changing the previously captured return value.

Dependency effects are additional and unknown. No allocation, release, ownership acquisition, or ownership transfer is explicit in this function, but that does not constrain dependency behavior. No lifetime protection for shared destinations is visible.

No lock, atomic access, memory barrier, rollback, or exception handler is explicit. The stated access sequence describes the supplied representation; it does not establish platform-level concurrency or device-ordering guarantees.

## Dependencies

| Opaque ID | Established invocation |
|---|---|
| F-1605e5a48b3c | No arguments. Called once for selectors 0 and 3 before obtaining the saved value. Its result is unused. |
| F-475ed60efd45 | No arguments. Called once for selectors 0 and 3. Supplies the opaque 32-bit value later written on completion. |
| F-6e9c50d81ffc | Receives the captured 32-bit request on each polling call. Supplies the 32-bit response. |
| F-812c4cd96b3e | Receives the then-current notification argument only when the selector-3 post-write status read is nonzero. Its result is unused. |
| F-17d6434f9fa3 | Receives ordered numeric arguments 116, 1, and 0 for selectors 1 and 2. Its result is unused. |
| F-external-1 | Document-local opaque ID for the unmapped external dependency. Receives 25 for selector 3 or 1 for selectors 1 and 2. Its result is unused. |

The external argument values do not establish a delay operation or time unit. Dependency duration, failure behavior, internal semantics, ownership effects, and preservation of shared state remain unknown. Exact dependency parameter types are not established beyond the values and 32-bit request/result uses shown here.

## Acceptance examples

These are conditional deductions, not executed tests. Unless an example specifies changing reads, dependencies return normally and relevant storage remains stable.

1. **Identity path.** Selector `0xFFFFFFFF` returns `0xFFFFFFFF`, with no explicit shared-state access or dependency call.

2. **Flag translation.** For selector 0, request `0x00001234` and response `0x00151234` match. Response bits 16, 18, and 20 produce result 19. The saved opaque value is written at completion base A plus 16 before return.

3. **Successful zero result.** For selector 0, request and response `0x00001234` match and return zero after the completion write. Five zero responses are not required.

4. **Zero response precedence.** A zero request followed by five zero responses exhausts the allowance; none is a match. Conversely, a first response of `0x00010000` matches a zero request and produces selector-0 result 1.

5. **Nonconsecutive exhaustion.** Responses zero, nonmatch, zero, nonmatch, zero, zero, zero cause exhaustion on the seventh call. Nonmatches do not reset the allowance.

6. **Impossible request match.** With request `0x00011234`, no response can satisfy the masked comparison because request bit 16 is set. Five zero responses can still terminate polling.

7. **Unbounded polling.** If every response is a nonzero nonmatch, neither polling selector reaches its final completion write.

8. **Stable selector-3 status.** Request `0x00001234` and response `0x00021234` match and produce status 2. If both status-pointer reads select the same stable word, the status test reads 2 and notification occurs.

9. **Distinct status destinations.** On a match producing status 2, suppose the first status-pointer read selects location P and the second selects location Q. If P contains zero, the function writes 2 to Q, reads zero from P, and skips notification. It does not test Q merely because Q received the write.

10. **Zero status with notification.** On a match producing status zero, suppose the write selects Q while the earlier captured pointer selects P containing 9. The function writes zero to Q, reads 9 from P, and invokes F-812c4cd96b3e.

11. **Distinct completion bases.** After selector-3 polling, suppose the first completion-base read yields R and the second yields W. The saved value is written at W plus 16 bytes, and R is returned. This applies after a match or exhaustion.

12. **Selector-3 exhaustion.** Five zero responses cause five status-pointer captures, but no explicit status dereference, status write, or notification. The two completion-base reads and completion write still occur.

13. **Control paths.** Selectors 1 and 2 make their two specified calls before writing `0x000E020E`. Selector 4 makes no explicit dependency call and writes `0x000E0402`.

Examples involving differing repeated reads are conditional access-sequence cases, not evidence that concurrent mutation is safe or guaranteed observable on an unspecified platform.

## Uncertainties

The following remain **documented_only**:

- Shared addresses, initial contents, pointer representation, alignment guarantees, and alias relationships.
- Whether shared storage is ordinary memory, volatile storage, or device memory.
- Platform byte order and concurrency, atomicity, or memory-order rules.
- Whether distinct reads of a shared pointer can observe changes in the actual execution environment.
- Dependency internals, failure modes, blocking behavior, storage releases, and global mutations.
- The external dependency’s published opaque ID.
- Meanings of the response flags, status, saved value, notification argument, and packed control fields.
- Whether callers rely on indefinite polling or selector-specific return meanings.

The apparent preservation and later publication of an opaque value is inferred intent. The established behavior is the saved bit pattern and the explicit completion-write sequence, not a named save/restore protocol.

## Evidence

This contract is based on static inspection of the supplied function representation and dependency mapping. Branches, constants, masks, argument order, explicit shared reads, full-word stores, and return expressions support the stated behavior.

In particular, selector 3 contains two distinct status-destination reads on a match: the earlier read supplies the status-test pointer, and the later read supplies the status-write pointer. It also contains two distinct completion-base reads: the earlier value is returned, while the later value selects the completion-write destination.

Comments and prior prose do not establish additional behavior. Dependency internals and hardware semantics are not available.

No runtime tests have been performed. Acceptance examples are conditional static deductions, not observed execution results.
