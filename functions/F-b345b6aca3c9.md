<!-- Trusted provenance: F-b345b6aca3c9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b345b6aca3c9

## Purpose

F-b345b6aca3c9 operates on a shared control record using an integer operation selector. It returns the record’s entry status value, changes status bits and a counter according to the selected operation, and invokes external dependencies in a defined order.

The established behavior supports four operations:

- **0:** replace status with zero and clear the counter.
- **1:** replace status with three and clear the counter.
- **2:** when status bit 0 is set, decrement a nonzero counter; when that decrement reaches zero, set status bit 1 and issue an additional dependency call.
- **3:** when status bit 0 is set, conditionally perform a refresh sequence and then increment the counter.

Other selector values perform no direct record writes, but still invoke the common entry and exit dependency calls.

Interpreting these operations as disabling, enabling, releasing, and acquiring is plausible intent, not an established interface guarantee. The meanings of the dependency calls and sampled values are unknown.

## Interface

### Input and result

The function accepts one integer selector. The supplied representation uses a signed integer input and an unsigned integer result. This contract describes their presented 32-bit interpretation; the underlying calling convention and original type declarations are not independently established.

Only exact selector values 0, 1, 2, and 3 receive special handling. Negative values and positive values above 3 take the default path. No selector conversion, range correction, or validation is visible.

The result is the unsigned 32-bit status word read before the first dependency invocation. It is not the resulting status and is not a success or failure indicator.

### Shared record

A global pointer supplies access to a record. The function captures a record pointer before invoking dependencies and uses that captured pointer for its subsequent record accesses.

The relevant fields are:

| Byte offset | Width | Contract name | Use |
|---|---:|---|---|
| 0 | 32 bits | Status | Returned entry snapshot; operation controls and bit updates |
| 20 | 32 bits | Counter | Cleared, decremented, tested, or incremented |
| 24 | 32 bits | Stored sample | Compared with and replaced by dependency results |

Offsets are relative to the captured record pointer. No accesses to the intervening fields are established.

Status bit 0 has mask 1. Status bit 1 has mask 2. These names identify bit positions, not externally established meanings.

The pointer must identify storage that supports the reads and writes taken by the selected path. Null-pointer handling, bounds checks, allocation checks, and record-version checks are absent. Alignment, byte order, and the full record extent are not established by the supplied evidence.

### Arithmetic

Counter operations use unsigned 32-bit arithmetic:

- Decrement occurs only after observing a nonzero counter.
- Increment wraps from 4,294,967,295 to zero.
- No overflow indication or saturation is provided.

The bit-clearing operation clears only status bit 1; its 32-bit mask is 4,294,967,293.

## Behavior

### Common entry and exit

The observable entry order is:

1. Capture the shared record pointer.
2. Read the entry status for the eventual result.
3. Invoke F-890414d1fd99 with arguments **(9, 20, 0, 0)**.
4. Perform selector-specific behavior.

Every normally completing path then invokes F-890414d1fd99 with **(10, 20, 0, 0)** and returns the saved entry status.

The entry status is captured before the entry call. By contrast, the operation’s status and counter decisions occur after that call. If the dependency modifies the record, those modifications can therefore affect the operation without changing the saved return value.

The common exit call occurs after all direct record writes for that operation. Its effects are not reversed or checked.

### Selector 0: replace status and clear counter

After the common entry call, the function:

1. Writes zero to status.
2. Writes zero to the counter.
3. Performs the common exit call.

This replaces the entire status word. It does not preserve higher bits.

The function does not directly read or write the stored sample on this path.

### Selector 1: replace status and clear counter

After the common entry call, the function:

1. Writes three to status.
2. Writes zero to the counter.
3. Performs the common exit call.

This sets status bits 0 and 1 and clears all other status bits. It is not an operation that merely sets two bits in the previous status value.

The stored sample is not directly accessed.

### Selector 2: conditional decrement

After the common entry call, the function reads status and tests bit 0.

If bit 0 is clear, it makes no direct counter access and no direct record write on this path. It proceeds to the common exit call.

If bit 0 is set, it reads the counter:

- If the observed counter is zero, it makes no direct record write and proceeds to the common exit call.
- If the observed counter is nonzero, it writes that observed value minus one back to the counter.

The next decision uses the decrement result derived from that observed counter, not a fresh counter read.

If the decrement result is nonzero, there are no further direct writes before the common exit call.

If the decrement result is zero, the function:

1. Reads status again.
2. Writes that status value with bit 1 set.
3. Invokes F-890414d1fd99 with **(10, 27, 0, 0)**.
4. Performs the common exit call.

Thus, the zero-transition write to the counter precedes the status update, and the status update precedes the additional dependency call.

The status bit-0 test is not repeated before setting bit 1. The later status update preserves other bits as observed by its own read, rather than rebuilding status from the saved entry snapshot.

The stored sample is not directly accessed on this operation.

### Selector 3: conditional refresh and increment

After the common entry call, the function reads status and tests bit 0.

If bit 0 is clear, it performs no further direct record access on this path and proceeds to the common exit call.

If bit 0 is set, it reads the counter to determine whether a refresh sequence is required.

#### Deciding whether to refresh

If the counter read is zero, refresh is required immediately. There is no comparison-sample call before the refresh sequence.

If the counter read is nonzero:

1. Invoke F-7eadd9e17dc6.
2. Read the stored sample after that invocation.
3. Compare the dependency’s returned value with that stored sample.

Refresh is required if those values differ. If they are equal, the function skips the refresh sequence and proceeds to the increment.

The order matters: a dependency-induced change to the stored sample can affect this comparison. The comparison is not necessarily against the value present before the dependency call.

#### Refresh sequence

When required, refresh consists of the following ordered actions:

1. Invoke F-890414d1fd99 with **(10, 20, 0, 0)**.
2. Invoke F-890414d1fd99 with **(9, 27, 0, 0)**.
3. Invoke F-890414d1fd99 with **(9, 20, 0, 0)**.
4. Read status and write it back with bit 1 cleared.
5. Invoke F-7eadd9e17dc6.
6. Write that invocation’s result to the stored sample.

The three dependency invocations precede the status update. The status update preserves all bits other than bit 1 as observed after those invocations.

For a nonzero counter with a mismatching comparison sample, F-7eadd9e17dc6 is called twice during the operation. The second result is stored; the first is used only for deciding whether to refresh. The two results need not be equal.

For a zero counter, F-7eadd9e17dc6 is called only once during the operation, to obtain the replacement stored sample.

#### Increment

After either skipping or completing refresh, the function reads the counter again, adds one using unsigned 32-bit arithmetic, and writes the result back.

This is a fresh read. The increment need not use the counter value that determined whether refresh was required. Changes made by intervening dependencies can affect the incremented value.

Status bit 0 is not rechecked after the initial eligibility test. A dependency clearing that bit does not, by itself, cancel the remaining refresh actions or increment.

### Other selectors

All other selectors perform only the common entry and exit calls and return the saved entry status.

They perform no direct counter or stored-sample access and no direct status write. This does not establish that shared state remains unchanged: the dependencies may have side effects.

## Side effects

Direct writes are limited to the three described record fields:

- Status replacement for selectors 0 and 1.
- Counter clearing for selectors 0 and 1.
- Conditional counter decrement and status-bit setting for selector 2.
- Conditional status-bit clearing and stored-sample replacement, followed by a counter increment, for an eligible selector 3.

No allocation, deallocation, ownership transfer, or explicit resource release is visible. That absence does not establish the memory-management behavior of either dependency.

The function continues using its captured record pointer across dependency calls. A dependency changing the global pointer does not redirect those subsequent direct accesses. Safe completion therefore requires the captured record storage to remain valid for the accesses that follow.

No locking, atomic operations, memory barriers, retry mechanism, or rollback is established. A read-modify-write of status or counter must not be treated as an atomic transaction.

There is no visible error-reporting branch. Results from F-890414d1fd99 are not inspected. Results from F-7eadd9e17dc6 are treated as ordinary 32-bit values; no value is reserved or checked as an error.

If a dependency fails to return normally, the evidence does not establish completion of later actions, execution of the common exit call, or rollback of earlier writes.

## Dependencies

### F-890414d1fd99

Receives four integer arguments. The established argument tuples and uses are:

| Arguments | Invocation condition |
|---|---|
| (9, 20, 0, 0) | Common entry; also the third call within refresh |
| (10, 20, 0, 0) | Common exit; also the first call within refresh |
| (10, 27, 0, 0) | Selector 2 decrements a nonzero counter to zero |
| (9, 27, 0, 0) | Second call within refresh |

A refresh therefore includes an invocation identical to the common exit call before later state changes and the actual common exit invocation. These are separate calls and must not be collapsed.

Its semantic meaning, side effects, error behavior, and memory-ownership behavior are unknown.

### F-7eadd9e17dc6

Takes no explicit arguments in the supplied evidence and returns a value used as an unsigned 32-bit sample.

It is called only on an eligible selector-3 path:

- Once for a zero-counter refresh.
- Once for a nonzero-counter comparison that matches.
- Twice for a nonzero-counter comparison that mismatches and causes refresh.

No purity, stability, monotonicity, or global-state preservation guarantee is established.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Except where stated otherwise, assume dependencies return normally and do not mutate the record or invalidate its storage.

1. **Selector 0, entry status 37, counter 8.**  
   Returns 37. Writes status zero before writing counter zero. The dependency-call sequence is (9, 20, 0, 0), then (10, 20, 0, 0). The stored sample is unchanged by direct actions.

2. **Selector 1, entry status 128, counter 8.**  
   Returns 128. Final status is three and final counter is zero. The previous high status bit is not preserved.

3. **Selector 2, status 4, counter 9.**  
   Returns four. Because bit 0 is clear, the function does not directly read or modify the counter. Only the common dependency calls occur.

4. **Selector 2, status 1, counter zero.**  
   Status and counter remain unchanged by direct actions. In particular, an already-zero counter does not cause bit 1 to be set or cause the additional (10, 27, 0, 0) call.

5. **Selector 2, status 1, counter two.**  
   Final counter is one and status remains one. No additional dependency call occurs.

6. **Selector 2, status 5, counter one.**  
   Counter becomes zero, then status becomes seven. The additional (10, 27, 0, 0) call occurs after both writes and before the common exit call. The result is five.

7. **Selector 3, status 3, counter zero; sample result 42.**  
   No comparison-sample call occurs before refresh. Refresh performs its three ordered calls, clears status bit 1, obtains 42, and stores it. The counter then becomes one. Final status is one; the result is three.

8. **Selector 3, status 3, counter five, stored sample 42; comparison result 42.**  
   Refresh is skipped. Status and stored sample are unchanged by direct actions. Counter becomes six. There is one sample invocation.

9. **Selector 3, status 3, counter five, stored sample 42; sample results 43 and 99.**  
   The comparison mismatches, causing refresh. Final status is one, stored sample is 99, and counter is six. The first sample result is not stored.

10. **Selector 3, status 1, maximum counter, matching comparison sample.**  
    Refresh is skipped and the increment wraps the counter to zero. No decrement-to-zero notification is generated by this increment path.

11. **Selector 3; refresh dependency changes the counter to 100.**  
    Provided later dependencies leave it at 100, the final increment writes 101. It does not increment the earlier counter observation.

12. **Unrecognized selector; entry dependency changes status.**  
    The return remains the pre-call entry status. The function does not directly restore that status or otherwise undo the dependency’s changes.

## Uncertainties

The following remain **documented_only**:

- The domain meanings of the selector values, status bits, counter, stored sample, and dependency argument tuples.
- Whether the common calls implement synchronization, device control, accounting, or another protocol.
- Whether dependencies mutate the record, global pointer, other shared state, or memory ownership.
- Whether dependencies can reenter this function.
- The complete record layout, initialization requirements, alignment, byte order, and external synchronization rules.
- The precise ABI and independent confirmation of the presented integer widths.
- Behavior under invalid storage, concurrent unsynchronized access, dependency failure, or nonlocal control transfer.

The global pointer is obtained for the captured access base and also used for the entry-status read before any dependency invocation. The evidence does not define asynchronous pointer replacement between those entry accesses. Treating the saved result as the same record’s entry status requires pointer stability over that interval.

No runtime tests have been performed. The acceptance examples describe consequences of the supplied static behavior under their stated conditions.

## Evidence

Established static evidence consists of the supplied body of F-b345b6aca3c9 and its opaque dependency mapping. It supports the selector branches, field offsets, bit masks, unsigned arithmetic, direct read/write ordering, dependency-call ordering, short-circuit sample behavior, and saved entry-status return described above.

The operational labels used here are descriptive conveniences. Any interpretation as a reference count, timing token, enable protocol, or paired resource operation is inferred intent rather than an established guarantee.

No dependency implementations, caller contracts, runtime observations, or independent ABI declarations were supplied. Their absence is retained as uncertainty rather than replaced with assumed behavior.
