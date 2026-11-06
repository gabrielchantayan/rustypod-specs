<!-- Trusted provenance: F-0f8f45a6b6cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0f8f45a6b6cb

## Purpose

F-0f8f45a6b6cb attempts to associate a caller-supplied record with an owner and a shared state object. It performs conditional admission checks, optionally initializes the shared state, invokes a resource-acquisition dependency, and, on success, publishes the record at the front of a state-maintained chain.

The established behavior is the sequence of accesses, calls, status mappings, and record updates described below. Interpreting this operation as registration or attachment is inferred intent, not an established domain meaning.

The operation is not transactional. Some failure paths return immediately; later failure paths invoke cleanup dependencies. No visible mechanism restores all earlier writes or reverses arbitrary dependency side effects.

## Interface

### Arguments and result

The supplied representation uses 32-bit integers, 32-bit pointer-bearing words, and byte-addressed offsets. The contract below describes that represented layout; it does not establish portability to a different pointer width.

The five arguments, in order, are:

1. **Owner address:** a 32-bit address whose word at byte offset +0x04 contains the shared-state address.
2. **Selector:** a 32-bit integer. It is passed to admission checking, written into the record, and compared with exactly 1 for a special eligibility path.
3. **Request flag:** a 32-bit integer. Its full-width nonzero value controls admission checks. On success, its low eight bits are stored in the record.
4. **Attachment value:** an opaque 32-bit value stored in the record on success.
5. **Record address:** a caller-supplied address used for record updates and resource acquisition.

The result is a 32-bit status:

- **0:** the successful publication path completed.
- **8:** a state-byte restriction rejected a nonzero request flag.
- **6:** the admission-check dependency returned any nonzero result.
- **16:** selector 1 reached the special eligibility check, which returned zero.
- **Another nonzero value:** propagated from state initialization or resource acquisition.

These categories describe return origins, not a globally unique error namespace. A dependency may itself return 6, 8, or 16.

### Accessed layouts

Offsets are measured in bytes.

| Object | Offset | Width | Established use |
|---|---:|---:|---|
| Owner | +0x04 | 32 bits | Shared-state address, captured at entry |
| Shared state | +0x00 | 32 bits | Argument to the selector-1 eligibility dependency |
| Shared state | +0x08 | 32 bits | Existing chain head; replaced with the record address |
| Shared state | +0x0C | 32 bits | Zero/nonzero condition controlling initialization |
| Shared state | +0x11 | 8 bits | Restriction byte tested for zero/nonzero |
| Record | +0x00 | 32 bits | Owner address on success |
| Record | +0x04 | 32 bits | Captured shared-state address on success |
| Record | +0x08 | 32 bits | Previously observed chain head |
| Record | +0x0C | 32 bits | Not directly initialized in the new record |
| Record | +0x10 | 32 bits | Attachment value on success |
| Record | +0x14 | 32 bits | Selector, written before resource acquisition |
| Record | +0x18 | Address of storage; at least one 32-bit word is subsequently read on failure | Resource-acquisition destination and cleanup argument source |
| Record | +0x40 | 8 bits | Low eight bits of the request flag on success |
| Record | +0x43 | 8 bits | Cleared after chain publication |

When the existing head is nonzero, its word at +0x0C is overwritten with the new record address. This suggests a reverse link, but the wider chain representation is not established.

### Preconditions and ownership

The function performs no general pointer validation. Every address reached on the selected path must support the indicated access, and dependency-specific preconditions must also hold.

A null record is not generally supported. Although one late cleanup call is guarded by a null check, the record is written before that branch can be reached through ordinary execution.

If the existing chain head is nonzero, it must support a 32-bit write at +0x0C. Required alignment and machine-level fault behavior are not established.

The function receives record storage; it does not visibly allocate or directly release that storage. Resource ownership associated with +0x18, and any reference ownership associated with the shared state, remain dependency-defined.

## Behavior

### 1. Capture the shared-state address

The first direct read obtains the shared-state address from owner offset +0x04.

That address is retained throughout the invocation. Later dependencies may alter owner memory, including its state-address field, but this function does not reload that field. Subsequent direct state accesses and state-related calls use the captured address.

### 2. Apply request-dependent admission checks

If the full-width request flag is nonzero:

1. Read the restriction byte at shared-state offset +0x11.
2. If that byte is nonzero, return 8 immediately.
3. Otherwise, call F-fa3b853f090d with the owner address, selector, and a zero third argument.
4. If that dependency returns any nonzero value, return 6.

The dependency’s particular nonzero status is discarded.

If the request flag is zero, both this initial restriction-byte check and the admission-check call are skipped.

These immediate-return paths perform no direct record writes and do not invoke the late cleanup dependencies. Calls already made may nevertheless have their own side effects.

### 3. Optionally initialize the captured shared state

Read the 32-bit word at shared-state offset +0x0C.

- If it is nonzero, skip initialization and proceed.
- If it is zero, call F-9a463b3fd053 with the owner address.

A nonzero initialization result is returned unchanged, without the late cleanup sequence.

After a zero initialization result, read the restriction byte again. If it is nonzero and the original full-width request flag is nonzero, return 8.

This post-initialization byte read occurs even when the request flag is zero; in that case, its value cannot cause rejection. If initialization was skipped because the state word was already nonzero, there is no corresponding second restriction-byte check.

The function does not itself set the initialization word or verify that initialization made it nonzero. Initialization is called with the owner, but the subsequent byte read still uses the captured shared-state address.

### 4. Store the selector and apply special eligibility

Write the original selector into record offset +0x14.

If the original selector is exactly 1, read the word at shared-state offset +0x00 and pass that value to F-f3192f39e4f6.

- A zero result selects failure status 16 and skips resource acquisition.
- Any nonzero result permits resource acquisition.

For every selector other than exactly 1, this dependency is not called.

The comparison uses the original argument, not a later reload from record storage. By contrast, the acquisition selector described next is loaded from the record. Thus, a dependency that changes record offset +0x14 can affect acquisition without changing whether the selector-1 eligibility branch was taken.

### 5. Acquire or prepare record-associated resources

Unless the special eligibility check selected status 16, call F-6293178c6e41 with:

1. The captured shared-state address.
2. The current 32-bit value at record offset +0x14.
3. The address of record offset +0x18.
4. Zero.

The third argument identifies storage within the supplied record, rather than passing the word currently stored there. The extent and format of storage used by the dependency are unknown.

A nonzero result selects that status for the late failure path. A zero result selects the success path.

### 6. Complete and publish the record

After successful resource acquisition, perform these direct operations in order:

1. Store the captured shared-state address at record +0x04.
2. Store the owner address at record +0x00.
3. Store the attachment value at record +0x10.
4. Store the low eight bits of the request flag at record +0x40.
5. Read the current head from shared-state +0x08.
6. Store that captured head at record +0x08.
7. If that captured head is nonzero, store the record address at the old head’s +0x0C.
8. Store the record address at shared-state +0x08.
9. Store zero in the byte at record +0x43.
10. Return 0.

The head used here is read after resource acquisition returns. It is not captured at entry.

The new record’s own +0x0C word is not directly initialized by this sequence. Bytes at +0x41 and +0x42 are also not directly written.

Publication of the new head precedes the final byte clear. There is no visible synchronization establishing atomic publication of a fully finalized record.

### 7. Perform late failure cleanup

For special eligibility failure or nonzero resource-acquisition status:

1. If the record address is nonzero, read its 32-bit word at +0x18 and call F-6cc778185718 with that value.
2. Call F-ce2b2a7a0121 with the captured shared-state address.
3. Return the selected failure status.

There is no check that the word passed to F-6cc778185718 is nonzero.

The same cleanup sequence is used when selector-1 eligibility failed before acquisition was called. Consequently, record +0x18 must already be suitable for cleanup on that path, or a dependency must have made it so. This function does not directly initialize that word first.

## Side effects

Direct record mutation starts with the selector write at +0x14. That write occurs before either special eligibility failure or resource-acquisition failure and is not visibly rolled back.

On late failure, the function does not directly clear the resource slot, unlink the record, restore overwritten fields, or release the record allocation. Cleanup dependencies may perform additional effects, but none are established here.

On success, the function updates both the supplied record and the captured shared state. A nonzero old head receives a further link write.

The following ordering is observable in the represented control flow:

- Initial state-address capture precedes every dependency call.
- The selector write precedes special eligibility and acquisition.
- Resource cleanup precedes shared-state cleanup.
- The old head’s +0x0C write precedes replacement of the state’s head.
- Head replacement precedes clearing record +0x43.

These are program-order observations, not guarantees of cross-thread visibility. No locking, atomic operation, or memory barrier is visible.

Aliasing can affect the result. For example, record writes could overlap state fields if the supplied addresses alias. No non-overlap checks are present. The ordinary interpretation of the layout assumes distinct, valid objects; unusual aliasing must be analyzed using the actual access order rather than presumed object roles.

## Dependencies

| Opaque function ID | Arguments supplied | Result handling |
|---|---|---|
| F-fa3b853f090d | Owner, original selector, zero | Zero continues; any nonzero becomes status 6 |
| F-9a463b3fd053 | Owner | Zero continues; nonzero returned unchanged |
| F-f3192f39e4f6 | Current word at captured state +0x00 | Zero selects status 16; nonzero continues |
| F-6293178c6e41 | Captured state, current record selector, address of record +0x18, zero | Zero succeeds; nonzero enters late cleanup |
| F-6cc778185718 | Current word at record +0x18 | Called during late cleanup; its result does not determine this function’s status |
| F-ce2b2a7a0121 | Captured state | Called after resource cleanup; its result does not determine this function’s status |

No dependency is assumed to preserve globals, preserve reachable memory, allocate resources, release resources, or transfer ownership merely because of its position in this sequence.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Where relevant, they assume valid addresses and dependencies that return normally.

1. **Immediate restriction rejection.** With a nonzero request flag and a nonzero initial restriction byte, return 8 before admission, initialization, acquisition, or any direct record write.

2. **Admission error normalization.** With a nonzero request flag, a zero initial restriction byte, and a nonzero admission result, return 6 regardless of that dependency’s particular nonzero status. Do not invoke late cleanup.

3. **Initialization failure.** If admission permits continuation, the initialization word is zero, and initialization returns −3, return the represented 32-bit value −3 unchanged. Do not invoke late cleanup.

4. **Restriction introduced by initialization.** With a nonzero request flag, successful initialization, and a subsequently nonzero restriction byte, return 8 without writing the record selector.

5. **Zero request flag.** Skip initial admission checking. If initialization runs successfully, read the restriction byte but do not reject because of it.

6. **Selector-1 ineligibility.** After prerequisites succeed, store selector 1. If eligibility returns zero, skip acquisition, select status 16, call resource cleanup with the current record +0x18 word, then call state cleanup.

7. **Acquisition failure.** If acquisition returns 23, invoke the two cleanup dependencies in order and return 23. Success-only publication writes are not performed by this function.

8. **Success into an empty chain.** If acquisition succeeds and the head read is zero, write zero to record +0x08, publish the record as the state head, clear record +0x43, and return 0. Do not write through an old-head address.

9. **Success into a nonempty chain.** If the head read identifies another record, save that address at the new record +0x08, set the old record’s +0x0C to the new record, then replace the state head.

10. **Full-width truth differs from stored byte.** A request flag of 256 takes the nonzero admission path but stores byte zero at record +0x40 on success.

11. **Dependency mutation affects acquisition.** If selector-1 eligibility returns nonzero after changing record +0x14 to 7, acquisition receives 7, while the eligibility decision remains based on the original selector 1.

12. **Owner state replacement does not redirect captured accesses.** If a dependency changes owner +0x04, subsequent direct state accesses still use the address captured at entry.

## Uncertainties

The following are **documented_only**:

- Domain meanings of the selector, request flag, attachment value, restriction byte, initialization word, and status codes.
- Dependency internals, memory effects, ownership rules, and behavior on invalid arguments.
- Required initialization and cleanup safety of record +0x18.
- The full record size and acquisition output layout.
- The intended invariant for the new record’s +0x0C word.
- Whether aliasing, reentrancy, concurrent access, or repeated registration is permitted.
- Platform-specific pointer validity, alignment, fault handling, and signed-byte representation.
- Whether dependencies can terminate, unwind, or otherwise prevent normal return.

The chain interpretation and registration purpose are inferred intent. The direct offsets, branch conditions, call arguments, status handling, and access ordering are established by the supplied static representation.

## Evidence

This contract derives solely from the supplied representation of F-0f8f45a6b6cb and the provided opaque dependency mapping. No implementation outside that representation was available.

No runtime tests, dependency executions, memory observations, or concurrency experiments have been performed. The acceptance examples are consequences of the represented control flow under their stated conditions, not observed executions.
