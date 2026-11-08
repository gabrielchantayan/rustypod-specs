<!-- Trusted provenance: F-718bce4db40c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-718bce4db40c

## Purpose

F-718bce4db40c performs ordered dispatch of an event-like record against a receiver-like object. It first attempts a specialized handler for selected record kinds and classification results. If no specialized handler applies, or the selected handler returns zero, it invokes a receiver-provided fallback callback. A further receiver callback is eligible only when that fallback returns zero and the then-current record satisfies a receiver-associated identity comparison.

The function returns the first nonzero handler or callback result unchanged. If no invoked handler or callback returns nonzero, it returns zero.

The terms “event,” “receiver,” “classification,” and “handled” describe inferred roles. The supplied static evidence establishes dispatch, memory accesses, argument forwarding, and integer-result propagation; it does not establish the application-level meaning of the numeric values.

## Interface

### Inputs

The function has two inputs:

1. **Receiver address.** A pointer to an integer-addressable object. The function reads its first word to obtain a callback-table address and, on one conditional path, reads a receiver-associated word at byte offset 80.
2. **Event address.** An integer-represented address identifying a record. The function reads a kind field at byte offset 4 and conditionally reads an associated value at byte offset 8.

Both original input values are passed unchanged to every selected specialized handler and indirect callback. The classification dependency receives only the original event address. There is no local copy of the event or receiver.

### Result

The result is an integer:

- A nonzero specialized-handler result is returned unchanged.
- Otherwise, a nonzero fallback-callback result is returned unchanged.
- Otherwise, a nonzero final-callback result is returned unchanged.
- All remaining normal-return paths return zero.

Negative results are nonzero and therefore terminate dispatch just as positive results do. The function does not normalize results to a Boolean value or distinguish success from error within the nonzero set.

### Widths and layouts

The supplied representation uses integer-sized fields and address values consistent with a 32-bit layout. Under that interpretation, the relevant fields are:

| Object | Byte offset | Interpretation in this function |
|---|---:|---|
| Receiver | 0 | Callback-table address |
| Receiver | 80 | Receiver-associated comparison word |
| Event | 4 | Record kind |
| Event | 8 | Nested-record address or comparison word, depending on the path |
| Nested record | 4 | Selector compared with 42 |
| Callback table | 224 | Final conditional callback address |
| Callback table | 268 | Fallback callback for the nested-selector match |
| Callback table | 272 | Other fallback callback |

The receiver offset 80 follows from an index of 20 integer elements in the supplied representation. The intended field width is 32 bits; the supplied material does not independently establish the original platform ABI, pointer width, alignment requirements, or byte order. Binary interoperability requires those facts to be established separately.

All numeric comparisons are equality or inequality comparisons. No signed ordering, multiplication, length calculation, or allocation-size arithmetic occurs. Address formation uses the fixed offsets above; behavior for invalid or wrapping address calculations is not established.

### Preconditions

For a normal return, every access and call actually reached must be valid:

- The event kind field must be readable whenever examined.
- The receiver’s callback-table word and the selected table entry must be readable.
- The selected callback address must identify a callable function with a compatible calling convention.
- Kind 22 at fallback selection requires a readable event-associated address and a readable nested selector at offset 4 from that address.
- Kind 15 at the final eligibility check requires readable comparison words in both the event and receiver.
- Invoked dependencies must return normally for subsequent local behavior to occur.

There are no visible null checks, bounds checks, object-lifetime checks, or callback-validity checks. A null value in a conditionally dereferenced address is not treated as a normal “not handled” outcome.

## Behavior

### 1. Ordered specialized dispatch

The function evaluates specialized cases in the following order. Kind fields are read again at successive conditions; classification results are not cached or reused.

1. **Kind 21:** Invoke F-7d061e6f1d5f with the receiver and event.
2. **Kind 13, classification 111:** Invoke F-eeacd77103f9 with the receiver and event.
3. **Kind 13, classification 110:** Invoke F-1f0d726cee9d with the receiver and event.
4. **Kind 13, classification 89:** Invoke F-ad15199eae9c with the receiver and event.
5. **Kind 13, classification 90:** Invoke F-611acb21394c with the receiver and event.
6. **Kind 8, classification 110:** Invoke F-1f0d726cee9d with the receiver and event.

Each classification test invokes F-468ee00d626a with the event address, but only if that test’s immediately preceding kind read matches the required kind.

A selected specialized handler is invoked once. If it returns nonzero, dispatch ends immediately with that exact result. If it returns zero, execution proceeds directly to fallback dispatch. It does not resume searching the remaining specialized cases.

If no specialized case matches, execution also proceeds to fallback dispatch.

#### Repeated classification is observable

For an event whose kind remains 13 throughout this phase:

- Classification 111 on the first call selects F-eeacd77103f9 after one classification call.
- Missing 111, then obtaining 110, selects F-1f0d726cee9d after two classification calls.
- Missing the first two targets, then obtaining 89, selects F-ad15199eae9c after three classification calls.
- Missing the first three targets, then obtaining 90, selects F-611acb21394c after four classification calls.
- Missing all four targets produces no specialized handler.

These statements do not require successive classification results to be identical. Each test compares that invocation’s result with its own target.

If F-468ee00d626a changes the event kind, later tests use subsequent kind reads. For example, if the first classification call returns a value other than 111 and changes the kind from 13 to 8, the remaining kind-13 tests can be skipped and the kind-8 classification test can still occur.

Consequently, replacing the sequence with one classification call or one initial kind snapshot would not preserve the established behavior without an additional immutability and purity contract.

### 2. Fallback callback selection

Fallback dispatch begins with a fresh read of the event kind.

- If that kind is **22**, the function reads the event word at offset 8 as a nested-record address, then reads the nested word at offset 4.
  - If the nested word equals **42**, it selects the callback at receiver-table offset **268**.
  - Otherwise, it selects the callback at receiver-table offset **272**.
- For every other kind, it selects the callback at receiver-table offset **272**, without reading the nested-record address for this selection.

The nested selector is read before the receiver’s callback-table address is loaded for this dispatch. The selected callback address is then loaded from that table and invoked with the receiver and event.

If the callback returns nonzero, that exact result is returned immediately.

If it returns zero, the function proceeds to the final conditional dispatch. There is no second fallback attempt: returning zero from the callback at offset 268 does not cause the callback at offset 272 to be invoked.

### 3. Final conditional callback

After a zero fallback result, the function reads the event kind again.

Only if this fresh kind equals **15** does it:

1. Read the event word at offset 8.
2. Read the receiver word at offset 80.
3. Compare those two words for equality.

If they are equal, the function reloads the receiver’s callback-table address, loads the callback at table offset **224**, and invokes it with the receiver and event.

A nonzero result is returned unchanged. A zero result produces a final result of zero.

If the fresh kind is not 15, or the two comparison words differ, this callback is not invoked and the function returns zero.

The comparison is exact word equality. The function does not dereference the compared values or establish that they are valid addresses. Two zero words satisfy the comparison, provided the receiver and event field reads themselves are valid.

### 4. Mutation between dispatch stages

Dependencies may change memory subsequently examined by this function; no preservation contract is supplied.

In particular:

- A zero-returning specialized handler may change which fallback callback is selected.
- Classification calls may change which later specialized conditions are evaluated.
- The fallback callback may change the event kind or either comparison word used by the final eligibility check.
- The fallback callback may replace the receiver’s callback-table address. The final callback lookup uses the reloaded table address, not a retained earlier table value.

These are consequences of the visible rereads, not claims that any dependency actually performs such mutations.

No inter-thread atomicity or synchronization guarantee is established.

## Side effects

The visible function body contains no explicit writes to the receiver, event, nested record, callback table, or global storage. Its direct effects are reads and dependency invocations.

Invoked functions may have additional effects, including memory mutation, resource operations, or changes to global state. Their effects are not constrained by the supplied material.

There is no visible allocation, deallocation, ownership transfer, reference-count adjustment, or object retention. Passing the receiver and event to a dependency does not establish that the dependency preserves, retains, consumes, or releases them.

If a dependency invalidates an object needed by a later reached read, normal continuation is not guaranteed. Conversely, after a dependency returns nonzero, this function immediately returns that result without further visible object reads.

No explicit diagnostic, exception translation, retry, rollback, or recovery action is present. Zero means that dispatch may continue or ultimately return zero; the evidence does not prove that zero denotes an application error. Likewise, nonzero may represent handling, failure, or another status defined elsewhere.

## Dependencies

| Opaque function ID or callback slot | Role established here |
|---|---|
| F-468ee00d626a | Classifies the event for each reached specialized classification test. Receives only the event address. |
| F-7d061e6f1d5f | Specialized handler selected by kind 21. |
| F-eeacd77103f9 | Specialized handler selected by a reached kind-13 classification result of 111. |
| F-1f0d726cee9d | Specialized handler selected by a reached classification result of 110 under kind 13 or kind 8. |
| F-ad15199eae9c | Specialized handler selected by a reached kind-13 classification result of 89. |
| F-611acb21394c | Specialized handler selected by a reached kind-13 classification result of 90. |
| Receiver-table offset 268 | Fallback callback selected for kind 22 with nested selector 42. |
| Receiver-table offset 272 | Fallback callback selected for all other fallback cases. |
| Receiver-table offset 224 | Final callback selected after a zero fallback result when kind 15 and the comparison words match. |

The concrete identities of the indirect callback targets are unknown.

F-d5dd567991e2 appears in the supplied mapping but has no demonstrated function invocation in the supplied body. The transfer to fallback processing is local control flow, not an established call to that dependency.

## Acceptance examples

These are conditional contract examples, not executed tests. All reached memory accesses and callback targets are assumed valid, and all invoked functions are assumed to return normally.

1. **Immediate specialized result.**  
   The initial kind is 21. F-7d061e6f1d5f returns −7. The function returns −7. It does not classify the event or invoke any fallback callback.

2. **Zero specialized result does not restart specialized search.**  
   The initial kind is 21. F-7d061e6f1d5f returns zero after changing the kind to 13. Execution proceeds to fallback selection using the current kind. The kind-13 classification sequence is not started.

3. **Second classification test succeeds.**  
   The kind remains 13. The first classification result is 0, and the second is 110. F-1f0d726cee9d is selected. No third classification call occurs.

4. **An earlier result is not reused.**  
   The kind remains 13. The first classification result is 110, the second is 111, the third is 0, and the fourth is 0. None matches its particular test target. No specialized handler is selected; fallback dispatch follows.

5. **Classification mutation changes the later route.**  
   The first kind-13 classification call returns 0 and changes the kind to 8. The intervening kind-13 tests do not classify. If the kind-8 classification call returns 110, F-1f0d726cee9d is selected.

6. **Nested-selector fallback.**  
   No specialized case is selected. At fallback selection, the kind is 22 and the nested selector is 42. Only the callback at offset 268 is selected. If it returns 5, the function returns 5.

7. **No alternate fallback after zero.**  
   The callback at offset 268 returns zero without changing kind 22. The callback at offset 272 is not tried. The final kind check fails, and the function returns zero.

8. **Fallback enables final dispatch.**  
   The fallback callback returns zero after setting the kind to 15 and making the event word at offset 8 equal the receiver word at offset 80. The callback at offset 224 is invoked. Its nonzero result becomes the function result.

9. **Updated callback table is honored.**  
   A zero-returning fallback callback replaces the receiver’s callback-table address and leaves the final eligibility condition satisfied. The final callback is obtained from offset 224 in the replacement table.

10. **Identity mismatch prevents final invocation.**  
    After a zero fallback result, the kind is 15 but the comparison words differ. No callback at offset 224 is invoked, and the result is zero.

11. **Invalid nested address is not a normal mismatch.**  
    At fallback selection, the kind is 22 and the nested-record address is invalid. The function attempts the nested selector read. The contract does not require a zero result or any other normal return.

## Uncertainties

The following items are **documented_only**:

- Original ABI, byte order, pointer width, alignment, and exact correspondence between the supplied representation and machine-level accesses.
- Application meanings of kinds 8, 13, 15, 21, and 22; classifications 89, 90, 110, and 111; and nested selector 42.
- Dependency purity, determinism, global-state preservation, ownership behavior, and resource effects.
- Whether nonzero results denote successful handling, errors, or another status.
- Object aliasing constraints, lifetime guarantees, and supported concurrent access.
- Concrete indirect callback identities and behavior.
- Fault, exception, or nonlocal-transfer behavior when dependencies or memory accesses do not complete normally.

Event dispatch is inferred intent. The ordered calls, comparisons, conditional reads, and result propagation are established static behavior.

## Evidence

This contract is based solely on the supplied function body and opaque dependency mapping. The body supports the described dispatch order, repeated classification calls, fixed-offset reads, callback selection, fresh reads after callbacks, and unchanged propagation of nonzero results.

No dependency implementations, independent type definitions, ABI documentation, runtime traces, or caller contracts were supplied. No runtime tests have been performed. The acceptance examples are deductions from static control flow, not observed executions.
