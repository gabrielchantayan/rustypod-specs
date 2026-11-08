<!-- Trusted provenance: F-8bc9ecd2a67f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8bc9ecd2a67f

## Purpose

F-8bc9ecd2a67f searches for a 32-bit target value over a sequence of indices associated with a supplied object. It returns the first matching index reached by its traversal, or the 32-bit sentinel `0xFFFFFFFF` when traversal finishes without a match.

The object selects between two access modes:

- **Count-checked mode:** validates the starting index and requested length against separately obtained counts, then retrieves pointers to values.
- **Unchecked mode:** performs no explicit count validation and retrieves values directly through a repeatedly resolved interface.

The requested interval is not necessarily a conventionally valid slice. In particular, count-checked mode checks the starting index and length separately; it does not establish that their sum is within the collection. Both modes use a signed comparison against a 32-bit endpoint, which can prevent traversal when the endpoint has its high bit set.

These behaviors are established by the supplied static representation. Describing the operation as a collection search, or the surrounding calls as synchronization, is inferred intent rather than an independently established dependency contract.

## Interface

### Inputs

The function accepts four machine-width arguments:

1. **Object address.** An address used as the base for fixed-offset field reads and embedded-interface addresses.
2. **Target value.** A 32-bit value compared for equality with retrieved values. Its displayed signed type does not affect equality: matching depends on the same 32-bit representation.
3. **Starting index.** A 32-bit unsigned input, except where explicitly interpreted as signed for validation or traversal.
4. **Requested length.** A 32-bit unsigned input used both in validation and in endpoint arithmetic.

If either the starting index or requested length equals `0xFFFFFFFF`, both supplied range arguments are discarded. The effective starting index becomes zero, and the effective requested length becomes the result of F-28fd04957d82.

That substitution occurs once. There is no second sentinel check on the value returned by F-28fd04957d82.

### Output

On normal return, the result is a 32-bit unsigned value:

- The index at which the first equality is observed.
- `0xFFFFFFFF` if no equality is observed before traversal terminates.

There is no ordinary error result distinct from the no-match sentinel. Count-validation failure instead invokes a dependency represented as non-returning.

### Relevant object layout

The supplied representation uses these object-relative locations:

| Offset | Access or role |
|---|---|
| `0x038` | Address passed to the final dependency in unchecked mode. |
| `0x32C` | Address passed to F-56a66efac853 during each unchecked-mode iteration. |
| `0x330` | Embedded interface address used in count-checked mode. Its first word supplies an indirect-dispatch table address. |
| `0x5F0` | Four-byte field loaded separately before each count query in count-checked mode. |
| `0x5F8` | One-byte mode field; only bit zero is tested. |

For the count-checked interface, the indirect entry at table byte offset `0x10` is called with the embedded interface address and current index. Its result is treated as a pointer to a 32-bit value.

For the unchecked interface, F-56a66efac853 returns an interface address. The indirect entry at table byte offset `0x0C` is called with that address and the current index. Its return is compared directly with the target.

The source representation supports a 32-bit address and integer model. It does not establish byte order, the complete object layout, or contracts for the two indirect methods.

## Behavior

### Range normalization and mode selection

The function first examines the two range arguments for the sentinel.

If either is the sentinel, it sets the effective starting index to zero and calls F-28fd04957d82 to obtain the effective length. This dependency receives no explicit arguments in the supplied representation. Its access to ambient state is unknown.

Only after normalization does the function read the mode byte at object offset `0x5F8`.

- Bit zero clear selects count-checked mode.
- Bit zero set selects unchecked mode.

Other bits do not participate in this decision. The selected mode is not re-evaluated during traversal. If normalization changes object state indirectly, the mode read occurs after those changes.

### Count-checked mode

The observable sequence is:

1. Form the embedded interface address at object offset `0x330`.
2. Call F-b7cd573c0b4c with no explicit argument shown.
3. Read the four-byte field at object offset `0x5F0` and pass that value to F-edd755fc1cdd.
4. Compare the starting index with the returned count using unsigned ordering.
5. If the starting index is less than that count, additionally require its signed 32-bit interpretation to be nonnegative.
6. Only if both starting-index conditions pass, reload the field at offset `0x5F0` and call F-edd755fc1cdd again.
7. Require the requested length to be less than or equal to this second result using unsigned ordering.

The two count results are not required by this function to be equal. The field supplying their argument is also read twice rather than captured once. No assumption that either dependency preserves globals or object state is part of this contract.

Validation therefore establishes only:

- The starting index is below the first count.
- The starting index is in the nonnegative signed 32-bit range.
- The length is no greater than the second count.

It does **not** establish that starting index plus length is no greater than either count.

If validation fails, the function calls F-a817d8d58c10. That dependency is represented as non-returning. No call to F-d49b39475920 appears on this path before that transfer.

After successful validation, the function forms the 32-bit endpoint from starting index plus requested length. It then traverses while the signed interpretation of the current index is less than the signed endpoint.

For each entered iteration:

- It reads the indirect-dispatch table through the embedded interface at offset `0x330`.
- It calls the entry at byte offset `0x10`, passing the embedded interface address and current index.
- It reads the 32-bit value at the returned address.
- If that value equals the target, it stops and retains the current index as its result.
- Otherwise it increments the index and repeats.

After either a match or normal exhaustion, it calls F-d49b39475920 with the embedded interface address, then returns the retained result.

The successful validation of a zero length still requires a valid starting index. In particular, a zero-length request does not bypass the first count query or its starting-index checks.

### Unchecked mode

The observable sequence is:

1. Call F-b7cd573c0b4c with no explicit argument shown.
2. Form the 32-bit endpoint from starting index plus requested length.
3. Traverse while the signed interpretation of the current index is less than the signed endpoint.

There is no explicit count query, unsigned range check, or nonnegative-start check in this mode.

For every entered iteration:

- Call F-56a66efac853 with the address at object offset `0x32C`.
- Treat its return as an interface address.
- Read that interface’s indirect-dispatch table.
- Call the entry at table byte offset `0x0C`, passing the returned interface address and current index.
- Compare the returned 32-bit value directly with the target.
- Stop at the first equality; otherwise increment the index.

F-56a66efac853 is called again for every iteration. Its result is not cached across indices, so different iterations can use different interfaces if the dependency returns them.

After a match or normal exhaustion, the function calls F-d49b39475920 with the address at object offset `0x038`, then returns the match index or no-match sentinel.

### Integer and termination semantics

The endpoint is a 32-bit quantity interpreted as signed for the loop comparison. The index increment is 32-bit unsigned arithmetic. The static representation combines unsigned addition with storage in a signed 32-bit endpoint; the behavioral interpretation here uses the low 32 bits of that sum as a signed value. The original language-level conversion rules are not independently available.

Consequences under that interpretation include:

- A zero length causes no iteration after any required validation.
- An endpoint with bit 31 set is negative in the loop comparison.
- In count-checked mode, a nonnegative starting index cannot enter the loop when the endpoint is negative.
- In unchecked mode, an input with bit 31 set can participate as a negative signed index.
- Traversal is not an unsigned circular-interval search. Signed comparison, rather than merely the requested length, determines whether it runs.

At each loop-condition evaluation, the prospective result is first set to `0xFFFFFFFF`. A match replaces it with the current index. Exhaustion therefore returns the sentinel, including when no iteration was entered.

## Side effects

The function directly performs reads of object fields, interface tables, and retrieved values. The supplied representation shows no direct writes into the object or collection.

All externally visible mutation may nevertheless include effects of:

- Range normalization.
- The initial F-b7cd573c0b4c call.
- Count queries.
- Interface resolution.
- Indirect element access.
- The final F-d49b39475920 call.
- The non-returning validation-failure dependency.

The initial and final dependencies suggest an enter/leave or acquire/release protocol, but that interpretation is not established. In particular, the initial call has no explicit receiver in the representation, and the final receiver differs between the two modes.

No allocation, deallocation, ownership transfer, or reference-count operation is explicit in this function. Returned interface and element pointers are used but not explicitly released here. Their validity and ownership remain dependency obligations; their absence from explicit cleanup does not establish that they are borrowed.

A normal empty traversal still performs the initial and final dependency calls. Count-checked mode additionally performs the applicable validation calls.

The retained return value is selected before the final dependency call. That call can still affect external state, fail, or fail to return; its behavior is not specified by this function.

There are no visible null checks for the object, count-related field, interface addresses, dispatch tables, indirect entries, or returned element pointer. Valid memory and callable targets are preconditions for an ordinary successful execution.

## Dependencies

| Opaque ID or dispatch site | Established use | Unestablished contract |
|---|---|---|
| F-28fd04957d82 | Called once when either range argument is the sentinel; supplies effective length. | Meaning of length, state effects, ownership effects, and possible sentinel return. |
| F-b7cd573c0b4c | Called once after mode selection, before validation or traversal. | Synchronization semantics, implicit receiver, failure behavior, and state preservation. |
| F-edd755fc1cdd | Returns a 32-bit count-like value from the separately loaded field at offset `0x5F0`; called once or twice in count-checked mode. | Stability, correspondence to indexed storage, and side effects. |
| F-a817d8d58c10 | Invoked on count-checked validation failure; represented as non-returning. | Whether it terminates, raises an exception, transfers control otherwise, or performs cleanup. |
| F-56a66efac853 | Resolves an interface from the address at offset `0x32C` on every unchecked-mode iteration. | Result lifetime, identity stability, mutation, and ownership. |
| F-d49b39475920 | Called on normal traversal completion with the mode-specific address. | Release semantics, state changes, and whether it always returns. |
| Count-checked indirect dispatch, table offset `0x10` | Receives interface and index; returns a pointer whose first 32-bit value is compared. | Bounds behavior, pointer validity, lifetime, and side effects. |
| Unchecked indirect dispatch, table offset `0x0C` | Receives interface and index; returns the value compared. | Bounds behavior, side effects, and error handling. |

## Acceptance examples

These examples are conditional static expectations, not executed tests. They assume invoked dependencies return normally unless stated otherwise and provide the described values and valid memory.

1. **First match in a conventional interval.**  
   With starting index 2, length 3, and successful count validation, candidate indices are 2, 3, and 4. If index 2 does not match and index 3 does, access stops at 3. The final dependency is called and the return value is 3.

2. **No match.**  
   With starting index 1 and length 2, where traversal enters normally, mismatches at indices 1 and 2 produce `0xFFFFFFFF`. There is no access at index 3.

3. **Either sentinel replaces both arguments.**  
   If the starting index is 7 and length is `0xFFFFFFFF`, and F-28fd04957d82 returns 4, the effective request starts at zero with length 4. The original starting index 7 is not retained. The same normalization occurs when only the starting index is the sentinel.

4. **Valid empty range in count-checked mode.**  
   With starting index 2, length zero, and both count queries returning 5, validation succeeds. No element access occurs. The final dependency receives the embedded interface address at offset `0x330`, and the result is `0xFFFFFFFF`.

5. **Empty collection rejects a zero-length request in count-checked mode.**  
   With starting index zero, length zero, and the first count query returning zero, the starting-index check fails. There is no second count query, no element access, and no explicit final-dependency call before F-a817d8d58c10.

6. **Separate checks do not protect the full interval.**  
   With starting index 4, length 3, and both counts equal to 5, validation passes. If accesses return normally and no earlier value matches, the function attempts indices 4, 5, and 6. Whether indices 5 and 6 are safe is not established.

7. **Two count observations can differ.**  
   With starting index 4, length 3, a first count of 5 and second count of 3 both satisfy their respective checks. A second count of 2 instead invokes the non-returning failure dependency.

8. **Negative signed endpoint suppresses traversal.**  
   In unchecked mode, starting index zero and length `0x80000000` yield a negative signed endpoint. No interface resolution or element access occurs. The initial and final dependencies are still called, and the result is `0xFFFFFFFF`.

9. **Unchecked mode can pass high-bit indices to its accessor.**  
   With starting index `0x80000000` and length 1, the signed loop comparison admits the starting index. If its retrieved value matches, the function returns `0x80000000`. No count validation precedes that access.

## Uncertainties

The following are **documented_only**:

- Complete ABI details, byte order, and original language-level integer-conversion rules.
- The meaning of the mode bit and the relationship between the two storage-access mechanisms.
- Whether count queries and indexed accesses refer to the same logical collection.
- Synchronization, reentrancy, and concurrent mutation guarantees.
- Dependency effects on globals, object fields, memory ownership, and pointer lifetime.
- Exact consequences of invalid addresses, invalid indices, dependency failures, or unexpected returns from the dependency represented as non-returning.
- Whether omitted or removed regions in the supplied representation affect the fidelity of this behavioral model.

No thread-safety, memory-safety, or ownership guarantee is inferred merely from the ordering of dependency calls.

## Evidence

This contract is based solely on the supplied static function representation and the supplied opaque dependency mapping. No implementation access, dependency-body inspection, runtime execution, or tests were performed.

Direct static evidence establishes the sentinel substitution, mode-bit selection, field offsets, short-circuit validation order, separate count-field reads, indirect-dispatch offsets, comparison forms, normal completion calls, and failure transfer.

Collection-search terminology and possible synchronization intent are interpretations. Dependency semantics, ownership, and guarantees beyond the visible operations remain unknown.
