<!-- Trusted provenance: F-e4ae6e4922ab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e4ae6e4922ab

## Purpose

F-e4ae6e4922ab coordinates a subject object’s cached state, performs three subject dispatches, searches a globally referenced structure, and conditionally changes a byte in a selected record. A record-byte change triggers two dispatches through a separately obtained service.

The established return value is a 32-bit zero or one. Zero means the final position-check dependency returned nonzero. One means that check returned zero and the subsequent record-processing path completed. The result does not indicate whether the cached state changed, whether the record byte changed, or whether dispatched operations reported success.

State synchronization and consumer notification are plausible intent, not established semantics. The meanings of the state values, identifiers, comparator results, and dispatch operations remain dependency-defined.

## Interface

### Input and representation

The function receives one subject pointer. The supplied representation uses 32-bit pointer-bearing words and 32-bit integer values, with individual byte accesses where specified.

Relevant subject locations are:

| Byte offset | Visible use |
|---|---|
| `0x00` | Dispatch-table pointer |
| `0xBC` | One-byte flag, read only when the initial queried state is exactly one |
| `0xC6` | One-byte cached state, compared with a full-width queried state and conditionally overwritten |

The subject dispatch-table entry at byte offset `0x58` is invoked three times. Each invocation visibly receives the subject and two additional 32-bit arguments.

There are no visible subject-null, object-size, or dispatch-target checks.

### Output

The returned 32-bit value is:

- **Zero** if the final invocation of F-f063d8ade9f7 returns nonzero.
- **One** if that invocation returns zero and execution completes the record-processing path.

Invalid accesses, non-returning dependencies, and exceptional control transfers have no established return contract.

### External state

The function reads a global container pointer and several global 32-bit arguments. This contract uses these explanatory labels:

- **Conversion token**
- **Common subject argument**
- **Subject identifiers A, B, C**
- **Lookup identifier**
- **Notification identifiers A, B**

These labels do not establish string representations, semantic meanings, immutability, or ownership.

The function also obtains services through F-b7af80e829ec and F-9d0ecc5f6adf. Repeated accessor calls are distinct; their results need not denote the same object.

## Behavior

### 1. Perform initial service operations

F-b7af80e829ec supplies a service. The function reads its dispatch table and invokes the entry at byte offset `0xB4`, visibly passing that service and zero.

It obtains a service again through F-b7af80e829ec and invokes the entry at offset `0x108`. That result is the **initial state**, interpreted as an unsigned 32-bit value.

The representation shows no explicit arguments for the `0x108` invocation. Whether an implicit receiver is supplied by the underlying calling convention is unresolved.

### 2. Compare and conditionally replace the cached state

The byte at subject offset `0xC6` is read and zero-extended for comparison with the full 32-bit initial state.

If they differ, the function:

1. Writes the initial state’s low eight bits to subject offset `0xC6`.
2. Obtains a service again through F-b7af80e829ec.
3. Invokes its dispatch-table entry at offset `0x120`.
4. If that result is nonzero, performs the following dependency chain:
   - F-483a717208e6 receives eight.
   - F-17c2f3b940d8 receives the preceding opaque result and the conversion token.
   - F-8e50f265e4fa receives the subject, the preceding opaque result, and zero.

The conversion token is read for that dependency invocation, not captured at function entry. No explicit release or destruction follows the conversion chain.

The comparison and write have different widths. An initial state greater than 255 always differs from the promoted cached byte, including when its low eight bits already equal that byte. Such a mismatch still causes the byte write and mismatch-only service query.

### 3. Conditionally invoke the subject helper

If the initial state is exactly one, the function reads the subject byte at offset `0xBC`. If that byte is zero, it calls F-a886da7a3894 with the subject.

This flag read occurs after any mismatch-only conversion chain. Changes made by those dependencies can therefore affect the flag value observed here.

For every other initial state, the flag is not read and this helper is not called. The return from F-8e50f265e4fa does not control this helper decision: when the initial state is one, the flag read supplies the relevant zero/nonzero value.

### 4. Issue three subject dispatches

The function first captures the common subject argument from its global word.

It then invokes the subject’s dispatch-table entry at offset `0x58` three times:

| Order | First visible argument | Second visible argument | Third visible argument |
|---|---|---|---|
| 1 | Subject | A fresh read of the common-argument global | Subject identifier A |
| 2 | Subject | The previously captured common argument | Subject identifier B |
| 3 | Subject | The previously captured common argument | Subject identifier C |

The capture and the first dispatch’s global read are distinct accesses. No intervening dependency invocation is shown between them. The evidence therefore establishes separate reads, not an observed difference in their values.

The second and third dispatches retain the captured value even if the first dispatch changes the common-argument global. Each subject identifier is obtained for its respective invocation rather than captured as a group before all three.

Each dispatch separately obtains its target through the subject’s current dispatch table. Results are ignored, and there is no result-based cancellation of subsequent dispatches.

### 5. Query the later status and initialize lookup state

After the three subject dispatches, F-b7af80e829ec supplies another service. Its dispatch-table entry at offset `0x120` produces the **later status**, interpreted as unsigned 32-bit.

The function then saves the global container pointer. This saved pointer is used for comparator context and later header reads.

The **binary target** is derived from the later status:

- Later status zero produces one.
- Every nonzero later status produces zero.

The visible arithmetic subtracts the later status from one at 32-bit width, then replaces the result with zero when the unsigned later status exceeds one. Thus values with the high bit set also produce zero; the result is not an arbitrary truncated subtraction byte.

Initialization then proceeds in this order:

1. Read the lookup-identifier global into the first word of a local lookup area.
2. Reread the global container pointer, read that container’s header pointer at offset `0x10`, and read the initial traversal node from header offset `0x04`.
3. Reread the global container pointer again and obtain the initial candidate from that container’s header pointer at offset `0x10`.
4. Store the binary target in the adjacent second word of the local lookup area.

Consequently, the saved container, the container supplying the traversal start, and the container supplying the initial candidate arise from three distinct global-pointer reads. The representation does not establish that they differ, but it also does not combine them into one captured access. No dependency call is shown between these initialization reads.

The local lookup area contains two adjacent 32-bit words: identifier followed by target. Dependencies receive a pointer to the first word. Which words they inspect or modify is unresolved. The separately retained binary target controls the eventual record comparison and write.

### 6. Traverse the structure

Relevant visible locations are:

| Location | Use |
|---|---|
| Container offset `0x10` | Header pointer |
| Saved container plus `0x19` | Comparator context address |
| Header offset `0x04` | Initial traversal node |
| Traversal node offset `0x08` | Link followed after a zero comparator result |
| Traversal node offset `0x0C` | Link followed after a nonzero comparator result |
| Traversal node plus `0x10` | Address supplied as comparison data |
| Finally accepted position offset `0x14` | Record pointer |

While the traversal pointer is non-null, F-bcefa5957e76 receives, in order:

1. The saved container address advanced by `0x19`.
2. The current traversal node address advanced by `0x10`.
3. The local lookup-area address.

A **zero** result causes two ordered actions:

1. Read the current node’s offset-`0x08` link and make that resulting pointer the traversal pointer.
2. Assign that resulting pointer to the candidate.

The candidate is therefore the linked-to pointer, **not the node just compared**. If the link is null, the candidate becomes null and traversal terminates.

A **nonzero** result follows the current node’s offset-`0x0C` link without changing the candidate.

There is no visible iteration bound or cycle detection. The structure resembles a tree, but conventional lower-bound or equality semantics must not replace these explicit candidate-update rules.

### 7. Check the candidate and select a position

After traversal, the function reads a header pointer from offset `0x10` of the **saved container**. It places that header and the candidate into separate local words and calls F-f063d8ade9f7 with their addresses, candidate first.

If the helper returns zero:

1. F-539f08165386 receives the candidate word’s current value.
2. F-bcefa5957e76 receives the saved-container context, the local lookup-area address, and the opaque value returned by F-539f08165386.
3. If that comparator also returns zero, the candidate local word supplies the selected position.

Otherwise, the function rereads the saved container’s header pointer and uses that newly read value as the selected position.

The two comparator uses have different argument order: traversal supplies node data before the lookup area; candidate acceptance supplies the lookup area before dependency-provided data.

The first position checker receives writable addresses, not merely position values. Its effects on those local words are unknown. Candidate extraction and eventual selection use the candidate word as it exists after that check.

The selected value is copied into another local word. A fresh header read from the saved container supplies a second local word. F-f063d8ade9f7 is called again with their addresses.

- A nonzero result causes an immediate zero return.
- A zero result continues to record processing using the selected-position word as it exists after this second call.

Neither a null candidate nor a header-like position is independently rejected by this function. Their treatment depends on the helpers. A zero final check does not itself prove that the resulting position is safe to dereference.

### 8. Compare the record byte and notify on change

After a zero final check, the function reads a record pointer from selected-position offset `0x14` and reads the byte at record offset `0x64`.

It compares that byte with the retained binary target.

If equal, the function performs no record-byte write and does not obtain the notification service.

If different:

1. It rereads the record pointer from selected-position offset `0x14`.
2. It writes the binary target’s low byte at record offset `0x64`.
3. It obtains a notification service through F-9d0ecc5f6adf.
4. It invokes that service’s dispatch-table entry at offset `0x58` with the service, notification identifier A, and the current lookup-identifier local word.
5. It invokes the same slot again with the service, notification identifier B, and the current lookup-identifier local word.

The record-pointer read used for writing is distinct from the read used for comparison; no dependency call intervenes between them.

The notification service pointer is reused, but each dispatch separately reads its dispatch target. Notification identifiers are read for their respective calls. The lookup identifier comes from the local lookup area, not a fresh read of its original global; earlier dependencies have received that area’s address.

Both notification results are ignored. After either record path completes, the function returns one.

## Side effects

Established direct externally visible writes are:

- One byte at subject offset `0xC6`, on a full-width state mismatch.
- One byte at record offset `0x64`, on a binary-target mismatch.

Important ordering constraints are:

- The cache write precedes the mismatch-only query and conversion chain.
- The flag read and possible subject helper precede all three subject dispatches.
- All three subject dispatches precede the later-status query and container initialization.
- Traversal precedes candidate checking and final position checking.
- The record write precedes notification-service acquisition and both notifications.
- Notification A precedes notification B.
- Final-check rejection does not undo earlier cache changes or dispatches.

Dependencies can have additional effects. Their preservation of globals, subject fields, local words passed by address, container links, services, and object lifetimes is not established.

The function contains no explicit deallocation, cleanup, rollback, null recovery, or allocation-failure handling. This establishes neither ownership nor freedom from allocation inside dependencies.

Normal completion requires usable dispatch tables, valid memory for all performed accesses, suitable dependency behavior, and a terminating traversal. If the final check accepts without changing an unusable selected position, record processing can attempt an invalid access rather than returning a defined error.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-b7af80e829ec | Supplies services used for dispatch slots `0xB4`, `0x108`, and `0x120` |
| F-483a717208e6 | Receives eight in the mismatch-only conversion branch |
| F-17c2f3b940d8 | Receives an opaque result and the conversion token |
| F-8e50f265e4fa | Receives the subject, converted opaque result, and zero |
| F-a886da7a3894 | Receives the subject when initial state is one and the later-read flag is zero |
| F-bcefa5957e76 | Controls traversal and candidate acceptance through zero/nonzero results |
| F-f063d8ade9f7 | Checks two local position/header words through their addresses |
| F-539f08165386 | Produces candidate-associated comparison data |
| F-9d0ecc5f6adf | Supplies the service used for the two notifications |

Dispatch operations are identified only by table offsets and their visible arguments. Their full signatures, implicit receivers, return semantics, and ownership effects are unresolved.

## Acceptance examples

These are conditional static expectations, not executed tests. They assume normal dependency returns and usable accessed memory unless stated otherwise.

1. **Matching cache and unchanged record.**  
   Initial state and cached byte are both zero. The mismatch-only query, conversion chain, and subject helper are skipped. All three subject dispatches still occur. With later status zero, the target is one. If final checking accepts a usable position whose record byte is one, the return is one without notification-service acquisition.

2. **Wide initial state.**  
   Initial state is 257 and the cached byte is one. The full-width comparison detects a mismatch, writes one back to the cache, and performs the mismatch-only query. The state-one helper is not invoked.

3. **Flag observed after conversion.**  
   Initial state is one and the mismatch branch executes. If the flag at `0xBC` is zero when subsequently read, the subject helper runs regardless of the conversion chain’s return value. If an earlier dependency changes that flag to nonzero, the helper is skipped.

4. **Captured common argument survives a callback.**  
   Suppose the common-argument capture and first-dispatch reread both observe the same value. If the first dispatch changes that global, the second and third dispatches still receive the captured value. This does not imply that the initial two reads were merged.

5. **Nonbinary later status.**  
   Later status two or `0xFFFFFFFF` produces target zero. Neither yields a wrapped nonbinary record byte.

6. **Zero traversal result with a null link.**  
   A comparison returns zero and the compared node’s offset-`0x08` link is null. Both traversal and candidate become null. If the first checker returns zero without changing the candidate, F-539f08165386 receives null. The function does not substitute the previously compared node.

7. **Distinct container observations.**  
   If the three initialization reads observe different containers, comparator context and later headers come from the saved container, the traversal starts through the second observed container, and the initial candidate comes from the third. No consistency check reconciles them.

8. **Final rejection or record change.**  
   A nonzero final check returns zero without record access or notifications. Conversely, if the check accepts a usable position, target zero differs from record byte one, and dependencies return normally, the function writes zero before acquiring the notification service, sends A then B, and returns one.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of states, flags, identifiers, and dispatches.
- Comparator ordering and which lookup-area words it reads or modifies.
- The structure’s actual type and required invariants.
- Whether null or header-like candidates are accepted, rejected, or rewritten.
- Full calling conventions and implicit arguments of dispatch operations.
- Dependency mutation of passed local words, globals, fields, or links.
- Ownership, allocation, retention, and release behavior inside dependencies.
- Outcomes for invalid pointers, malformed structures, exceptions, or non-returning calls.
- Endianness and ABI details beyond the visible widths and offsets.

No preservation or ownership guarantee is inferred from an opaque call. In particular, the traversal’s linked-to candidate update remains the established behavior even if it appears unconventional for an ordered search.

## Evidence

This contract derives solely from the supplied static function representation and opaque dependency mapping. Established evidence consists of visible accesses, widths, argument positions, branches, arithmetic, pointer forwarding, dispatch order, writes, and return paths.

The three distinct initialization reads of the global container, the separate common-argument capture and first-call reread, and the candidate assignment after following the offset-`0x08` link are explicit static observations.

Structural labels and synchronization intent are explanatory interpretations, not recovered semantic guarantees. No dependency implementations, runtime traces, or independent ABI evidence were supplied. No runtime tests have been performed.
