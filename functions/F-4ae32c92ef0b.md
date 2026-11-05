<!-- Trusted provenance: F-4ae32c92ef0b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4ae32c92ef0b

## Purpose

F-4ae32c92ef0b conditionally updates two state flags associated with an owner object, notifies a dependency when an update is required, and then performs a conditional sequence of indirect operations on a separately obtained receiver.

The requested state has three effective categories:

- **Disabled:** the request equals zero.
- **Enabled, ordinary:** the request is nonzero and does not equal two.
- **Enabled, special:** the request equals two.

These category names describe the observed encoding, not an established application-level meaning. The source does not establish whether the flags represent visibility, selection, activation, or another domain concept.

A matching state normally produces an immediate return without writes or dependency calls. One important exception to full state normalization is that an already-disabled object returns without inspecting or clearing its secondary flag.

## Interface

### Inputs

The recovered interface accepts two 32-bit integer-sized arguments:

| Input | Interpretation |
|---|---|
| Owner reference | Address of an object containing a four-byte state-object reference at byte offset `0xF00`. |
| Requested state | A 32-bit integer interpreted through the predicates “nonzero” and “equal to two.” |

The owner argument is used as an address, not as an ordinary numeric value. The recovered representation uses four-byte address-bearing fields. This contract describes that representation; it does not establish portability to a different pointer width.

All requested-state values are handled by the visible control flow. No range validation restricts the input to zero, one, or two. Negative values and positive values other than two fall into the enabled, ordinary category.

### Referenced layouts

Only the following portions of the owner and state object are established:

| Object | Byte offset | Width | Role |
|---|---:|---:|---|
| Owner | `0xF00` | 4 bytes | State-object reference |
| State object | `0x1C` | 1 byte | Primary flag |
| State object | `0x8E` | 1 byte | Secondary flag |

Flag reads are evaluated by zero-versus-nonzero truth. Flag writes store Boolean results: zero for false and one for true.

The later receiver has a four-byte dispatch-table reference at its beginning. The visible indirect operations use dispatch-table byte offsets `0x1A4`, `0x78`, and `0x88`. These are byte offsets, not entry numbers.

### Output

There is no explicit return value. Observable results are:

- Whether the function returns without changing state.
- The flag stores on the update path.
- Calls to direct dependencies.
- The conditional sequence of indirect receiver operations.
- Any effects or failures originating in those dependencies.

No status code or recoverable error result is provided.

### Preconditions

Successful execution requires the memory accesses and indirect calls actually reached to be valid. In particular:

- The owner reference must permit reading the field at `0xF00`.
- Each state-object reference used must permit the particular byte access reached.
- On the update path, the notification dependency must accept the reference passed to it.
- The receiver returned by its provider must support the reached dispatch-table reads and calls.

There are no visible null checks, bounds checks, pointer validation, or fallback paths. A valid primary-flag address does not imply that the secondary-flag address must be readable on an early-return path, because that path may never access it.

## Behavior

### Initial observation and no-change decision

The function first reads the state-object reference from the owner. It then reads the primary flag from that initially selected state object.

The primary flag’s truth value is compared with whether the requested state is nonzero.

If those truth values differ, the function takes the update path immediately. It does not first read the old secondary flag.

If those truth values match:

- When the observed primary flag is zero, the function returns immediately.
- When the observed primary flag is nonzero, the function reads the secondary flag from the same initially selected state object.
- For that enabled case, it returns immediately if the secondary flag’s truth value matches whether the requested state equals two.
- Otherwise, it takes the update path.

Thus, the disabled no-change decision depends only on the primary flag. A stale nonzero secondary flag does not prevent an immediate return when the primary flag and requested state are both zero.

The decision compares Boolean meanings rather than requiring stored flag bytes to be canonical zero or one. A nonzero primary or secondary byte can satisfy the corresponding true condition without being rewritten.

### Ordered flag updates

The update path performs two distinct byte stores in this order:

1. It writes the truth value of “requested state is nonzero” to the primary flag of the state object selected by the initial owner-field read.
2. It reads the owner’s state-object reference again and writes the truth value of “requested state equals two” to that newly selected object’s secondary flag.

The owner’s state-object reference is not treated as a single immutable snapshot across these operations. The first store uses the initially selected object; the second store uses a fresh read of the owner field.

If the owner field remains stable, the resulting stored pair is:

| Requested state | Primary flag | Secondary flag |
|---|---:|---:|
| Zero | 0 | 0 |
| Two | 1 | 1 |
| Any other nonzero value | 1 | 0 |

This table describes completed update-path stores under a stable owner reference. It does not describe every early-return result, because the disabled early-return case leaves the secondary byte untouched.

The two writes are not presented as an atomic transaction. No rollback is visible if a later access or dependency fails.

### Notification

After the stores, the function reads the owner’s state-object reference a third time and passes that value to F-e0ba31379901.

Consequently, the notification argument is not guaranteed to equal either object reference previously used for a store if the owner field can change between observations.

F-e0ba31379901 is called only on the update path. It is not called for any early-return case. Its result, if any, is not used by the visible caller.

### Receiver acquisition and indirect operations

After F-e0ba31379901 returns, the function calls F-87be4aa684d1 with no explicit arguments. Its result supplies the receiver for the remaining operations.

The function reads that receiver’s dispatch-table reference and invokes the operation at byte offset `0x1A4`, passing the receiver as the explicit object argument. The result is consumed as a 32-bit integer.

- If the result is nonzero, F-4ae32c92ef0b returns.
- If the result is zero, it invokes the operation at byte offset `0x78`, passing the same receiver and the integer value three.
- It then invokes the operation at byte offset `0x88`, passing the same receiver, and completes.

The receiver’s dispatch-table reference is read again for each visible indirect operation. In particular, the function does not establish that the operation at `0x78` leaves the dispatch-table reference unchanged before the operation at `0x88`.

The final indirect transfer has an explicit recovery warning in the supplied source. The recovered representation shows it as an invocation followed by completion, but the exact underlying transfer form remains uncertain.

## Side effects

### Established direct effects

On the update path, the function directly performs:

- One byte store to a primary flag.
- One byte store to a secondary flag.
- One call to F-e0ba31379901.
- One call to F-87be4aa684d1.
- One receiver operation at dispatch-table offset `0x1A4`.
- If that operation returns zero, two additional receiver operations, in order: `0x78` with argument three, then `0x88`.

On an early-return path, it performs no visible stores and invokes none of those dependencies.

The ordering above is the ordering expressed by the recovered source. It is not a claim about synchronization, atomicity, or hardware ordering under concurrent access.

### Ownership and lifetime

No explicit allocation, deallocation, ownership transfer, or reference-count adjustment appears in this function.

That absence does not establish ownership behavior inside any dependency. In particular:

- The notification dependency may have effects beyond the state object passed to it.
- The receiver provider’s ownership and lifetime contract is unknown.
- Indirect receiver operations may mutate the receiver or other state.
- No dependency is known to preserve global state or retain or release memory.

Continued use of the receiver after the query operation, and again after the operation receiving three, requires the relevant receiver and dispatch information to remain usable. The visible caller does not enforce that requirement.

### Errors and interruption

There is no explicit error handling. The nonzero result from the query operation selects a normal return; it is not established as an error indication.

Behavior after invalid memory access, an invalid indirect target, dependency failure, nonreturning dependency behavior, or lifetime invalidation is outside the established successful-execution contract. The function contains no visible recovery mechanism for those cases.

## Dependencies

The following identifiers distinguish established dependencies from unresolved indirect targets. The unresolved identifiers are document-local opaque labels, not recovered symbol identities.

| Opaque ID | Established interaction | Unknowns |
|---|---|---|
| F-e0ba31379901 | Called after both stores, with the most recently read owner-held state reference. | Meaning, ownership effects, global effects, failure behavior, and any return contract. |
| F-87be4aa684d1 | Called afterward with no explicit arguments; its result is used as a receiver. | Selection policy, nullability, ownership, lifetime, and other side effects. |
| F-unresolved-01 | Receiver operation selected at table byte offset `0x1A4`; receives the receiver and returns a value tested against zero. | Domain meaning, mutation effects, and interpretation beyond zero versus nonzero. |
| F-unresolved-02 | Receiver operation selected at table byte offset `0x78`; receives the receiver and integer three. | Meaning of three, ownership effects, and all other effects. |
| F-unresolved-03 | Receiver operation selected at table byte offset `0x88`; receives the receiver after F-unresolved-02. | Exact transfer form, domain meaning, return behavior, and all other effects. |

The visible caller does not use results from F-unresolved-02 or F-unresolved-03.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Unless stated otherwise, assume valid memory, a stable owner-held reference, returning dependencies, and a usable receiver.

1. **Already disabled with a stale secondary flag.**  
   Initial flags are primary zero and secondary one; the request is zero. The function reads the primary flag and returns. It does not read or clear the secondary flag and calls no dependencies.

2. **Already enabled in ordinary mode.**  
   Initial flags are primary one and secondary zero; the request is one. Both flags are read, no stores occur, and no dependencies are called.

3. **Equivalent nonzero requests.**  
   Initial flags are primary one and secondary zero; the request is negative one. The function returns without updates. The visible behavior does not distinguish this request from one.

4. **Already enabled in special mode.**  
   Initial flags are primary one and secondary one; the request is two. Both flags are read, and the function returns without updates.

5. **Transition from ordinary to special.**  
   Initial flags are primary one and secondary zero; the request is two. The function writes primary one, then secondary one, and proceeds through notification and receiver acquisition.

6. **Disable an enabled object.**  
   The initial primary flag is nonzero; the request is zero. The old secondary flag is not read before updating. The function writes primary zero, then secondary zero, and continues through the dependency sequence.

7. **Noncanonical true bytes.**  
   Initial primary and secondary bytes are both nonzero values other than one; the request is two. Their truth values match the request, so the function returns without canonicalizing either byte.

8. **Nonzero query result.**  
   After an update, F-unresolved-01 returns any nonzero 32-bit result, including a negative result. Neither F-unresolved-02 nor F-unresolved-03 is invoked.

9. **Zero query result.**  
   After an update, F-unresolved-01 returns zero. F-unresolved-02 is invoked with three, followed by F-unresolved-03, subject to the earlier invocation returning normally and leaving the subsequent access usable.

10. **Owner reference changes between observations.**  
    If the execution environment permits such a change, the primary store can target one state object, the secondary store another, and the notification a third. The visible function does not consolidate these observations into one reference.

## Uncertainties

### Inferred intent

The flag pairing suggests a disabled state and two enabled submodes. The subsequent notification and receiver operations suggest propagation or reconciliation of a state change. These are interpretations only; application-specific meanings are not established.

### documented_only

The following remain unknown:

- Actual object types, complete layouts, allocation sizes, and lifetime rules.
- Whether the surrounding system permits concurrent or externally visible mutation of the owner field.
- Whether noncanonical flag bytes occur in valid application states.
- Dependency effects on globals, memory ownership, receiver validity, or dispatch tables.
- The semantic meaning of the query result and the constant three.
- Whether the final indirect transfer is an ordinary call, a tail transfer, or another recovered control-flow form.
- Behavior outside the recovered memory-access and calling assumptions.

No concurrency guarantee follows from the repeated reads. Those reads establish separate observations in the recovered expression of the function, not a safe concurrent-access protocol.

## Evidence

Established claims come from static examination of the supplied recovered function: its field widths and offsets, Boolean comparisons, conditional secondary read, ordered stores, repeated owner-reference reads, direct dependency calls, indirect dispatch offsets, and zero-result branch.

The supplied recovery warning limits confidence in the exact final indirect-transfer form. Dependency bodies and runtime observations were not supplied, so their internal semantics are not incorporated into this contract.

No runtime tests have been performed. The acceptance examples describe expected behavior under their stated preconditions; they are not reports of observed execution.
