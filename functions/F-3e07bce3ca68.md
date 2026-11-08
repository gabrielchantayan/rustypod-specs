<!-- Trusted provenance: F-3e07bce3ca68; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3e07bce3ca68

## Purpose

F-3e07bce3ca68 performs two ordered operations against a state-bearing object:

1. It conditionally delegates an interval below a supplied lower boundary to F-3ab4d02fd54b.
2. It conditionally processes an interval above a supplied upper boundary, constrained by a dynamically dispatched callback. For each selected index, it performs a lookup, invokes another dependency when the lookup succeeds, and may submit the referenced value to F-33f2303f9bf9.

This is an established description of control flow, not a claim about the application-level meaning of the intervals. A plausible intent is to process entries outside a retained interval, but deletion, eviction, release, and ownership transfer are not established by the supplied evidence.

The two operations are not symmetric. The first delegates an interval directly. The second obtains callback-adjusted bounds and performs indexed processing itself.

## Interface

### Arguments and result

The function has four inputs:

| Input | Established interpretation |
|---|---|
| State address | Base address of a readable object containing fields described below. The supplied representation uses a 32-bit integer to carry this address. |
| Requested lower boundary | Unsigned 32-bit value used by the first operation. |
| Requested upper boundary | Unsigned 32-bit value used by the second operation. |
| Callback seed | Unsigned 32-bit value copied into writable temporary storage before dynamic dispatch. |

There is no explicit return value and no explicit error result.

The requested boundaries are not validated as an ordered pair. In particular, the function does not require or check that the requested lower boundary is less than or equal to the requested upper boundary.

### State and referenced layouts

The following byte offsets are established by the supplied representation:

| Location | Access and role |
|---|---|
| State base + 0x14 | Address passed to F-33f2303f9bf9. Its internal layout is unknown. |
| State base + 0x30 | A referenced-object address, loaded separately at multiple call sites. |
| State base + 0x38 | A 32-bit boundary value. |
| State base + 0x3C | A 32-bit boundary value. |

The referenced object at offset 0x30 supports a dispatch layout in which its first address-sized field points to a table. A callable entry at table byte offset 0x0C is used. The visible representation uses 32-bit addresses and 32-bit scalar fields; broader ABI details are not established.

A successful lookup by F-36b67cc7bb38 returns an address from which a 32-bit value is read. That value is subsequently treated as another address:

- A 32-bit field at that address + 0x40 is compared with zero.
- The value itself may be copied into a temporary and passed by address to F-33f2303f9bf9.

### Preconditions

There are no explicit null, range, accessibility, or alignment checks for the state object, dispatch object, dispatch table, or the value referenced by a successful lookup.

For an execution to complete without an invalid memory access, every address actually dereferenced must remain readable at the time of the access, and every indirectly invoked entry must be callable under the expected ABI.

In particular, a non-null lookup result is not sufficient by itself: its storage and the object address subsequently read from it must still be valid after F-dc07b121bb37 returns.

## Behavior

### 1. Conditionally delegate the lower-side interval

The function first reads the 32-bit boundary at state offset 0x38.

It compares this value with the requested lower boundary using an **unsigned** comparison. If the stored value is not strictly smaller, this stage performs no dependency call and does not read offset 0x3C for this stage.

If the stored value is strictly smaller:

1. Read the 32-bit value at state offset 0x3C.
2. Choose the delegated interval’s upper endpoint:
   - If the requested lower boundary is less than or equal to the just-read value, use the requested lower boundary minus one.
   - Otherwise, use the just-read value.
3. If the original value from offset 0x38 is not 0xFFFFFFFF, invoke F-3ab4d02fd54b with:
   - the state address;
   - that original boundary;
   - the chosen upper endpoint.

The subtraction is 32-bit unsigned arithmetic. Its controlling condition also requires a stored unsigned value to be strictly smaller than the requested lower boundary, so this subtraction cannot execute with a requested lower boundary of zero.

The 0xFFFFFFFF guard is explicit, although it is unreachable under the preceding unsigned strict-less-than condition: no unsigned 32-bit input is larger than 0xFFFFFFFF.

This function does not independently reject a reversed delegated interval. If the stored fields are inconsistent, the selected upper endpoint can be smaller than the selected lower endpoint, and that pair is still passed to F-3ab4d02fd54b.

### 2. Re-read state for the upper-side operation

After the first stage, including any call it made, the function reads offset 0x3C again.

This is a fresh read. Effects of F-3ab4d02fd54b on that field can therefore affect whether the second stage runs.

If the freshly read value is less than or equal to the requested upper boundary, using an **unsigned** comparison, the function returns immediately.

Otherwise:

1. Read offset 0x38 again.
2. If that value is less than or equal to the requested upper boundary, replace the candidate lower endpoint with the requested upper boundary plus one.
3. Otherwise, retain the freshly read value from offset 0x38 as the candidate lower endpoint.
4. If the resulting candidate lower endpoint is 0xFFFFFFFF, return without invoking the callback.

The addition is 32-bit unsigned arithmetic. A requested upper boundary of 0xFFFFFFFF cannot reach this addition because the preceding upper-boundary comparison would already have returned. However, a requested upper boundary of 0xFFFFFFFE can produce 0xFFFFFFFF, causing the explicit sentinel return.

The freshly read value from offset 0x3C becomes the saved upper endpoint for subsequent processing.

### 3. Invoke the dynamic callback

Before dynamic dispatch, the function initializes writable 32-bit temporary storage:

- One word contains the callback seed.
- One word contains the saved upper endpoint.

It then invokes the dispatch-table entry at byte offset 0x0C with these arguments, in order:

1. The referenced-object address obtained through state offset 0x30.
2. The address of the temporary containing the callback seed.
3. The address of the temporary containing the saved upper endpoint.

The callback can modify the two pointed-to words. Their values after return are interpreted as the callback-selected lower and upper endpoints, respectively.

The function’s candidate lower endpoint and saved upper endpoint remain the bounds against which those callback results are checked. The function does not re-read the state boundary fields after this callback.

If the callback leaves either returned endpoint equal to 0xFFFFFFFF, indexed processing is skipped.

The supplied representation also exposes a secondary return-register value from this call. It does not affect the established processing decision: when the callback-selected lower endpoint is valid, the upper endpoint comes from the callback-writable word; when the lower endpoint is the sentinel, processing is rejected.

### 4. Select the indexed interval

Unlike the earlier comparisons, the following endpoint comparisons interpret the 32-bit bit patterns as **signed** values.

Processing requires all of the following:

- The callback-selected lower endpoint is not 0xFFFFFFFF.
- The callback-selected upper endpoint is not 0xFFFFFFFF.
- The candidate lower endpoint is no greater than the callback-selected upper endpoint under signed comparison.
- The callback-selected lower endpoint is no greater than the saved upper endpoint under signed comparison.

The effective starting index is the signed maximum of:

- the candidate lower endpoint;
- the callback-selected lower endpoint.

The effective ending index is the signed minimum of:

- the saved upper endpoint;
- the callback-selected upper endpoint.

The loop executes only while the current index is no greater than the effective ending index under signed comparison.

These rules do not establish one consistent unsigned interval model. Values with bit 31 set participate in the initial stages as large unsigned numbers but participate in this stage as negative signed numbers.

### 5. Process each selected index

For each loop iteration, the following sequence is visible:

1. Read the referenced-object field at state offset 0x30 and call F-36b67cc7bb38 with that value and the current 32-bit index.
2. If the returned address is null, skip all remaining work for that index.
3. Otherwise, read the referenced-object field at state offset 0x30 again and call F-dc07b121bb37 with that freshly read value and the same index.
4. After that call returns, read the 32-bit value through the address returned by F-36b67cc7bb38.
5. Treat the read value as an address and read its 32-bit field at byte offset 0x40.
6. If that field is nonzero, do not call F-33f2303f9bf9 for this index.
7. If that field is zero:
   - Read the 32-bit value through the lookup-returned address again.
   - Copy it into writable temporary storage.
   - Call F-33f2303f9bf9 with the state address plus 0x14 and the address of that temporary.
8. Increment the current index by one using 32-bit arithmetic and repeat the signed loop comparison.

The lookup result is deliberately used **after** F-dc07b121bb37. The evidence does not establish that the dependency preserves that storage. Nor does it establish that the value read through that storage remains unchanged between its two reads.

The referenced-object field is not held constant across dependency calls. A dependency that changes state offset 0x30 can cause the next call to receive a different object.

### Integer and termination boundaries

Unsigned increment wraps modulo 2³², while the continuation test is signed.

If an entered loop has an effective ending index of 0x7FFFFFFF, the visible loop condition cannot become false through index increments alone: every signed 32-bit value is less than or equal to that endpoint. Incrementing past 0x7FFFFFFF produces 0x80000000, interpreted as the most negative signed value, and iteration continues.

Accordingly, finite traversal is not guaranteed for all admitted 32-bit endpoint combinations. No overflow error, iteration limit, or escape check is present.

## Side effects

The function directly writes its own temporary storage. It does not directly assign the state boundary fields or the referenced-object field.

Externally visible mutation can occur through:

- F-3ab4d02fd54b;
- the dynamically dispatched callback;
- F-36b67cc7bb38;
- F-dc07b121bb37;
- F-33f2303f9bf9.

Their effects are not constrained by the supplied body. In particular, they must not be assumed to preserve global state, preserve the state object, leave lookup storage valid, or avoid allocation or deallocation.

The addresses passed for callback output and submission storage designate temporaries whose lifetime does not extend beyond this invocation. No persistent ownership transfer of those temporary addresses is established. Whether a dependency copies their contents, retains them, or releases anything referenced by them is unknown.

There is no visible rollback. Effects performed by an earlier dependency remain possible even if a later guard causes an immediate return.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-3ab4d02fd54b | Receives the state address and two 32-bit endpoints during the lower-side stage. Its return value is not used. |
| F-36b67cc7bb38 | Receives a freshly loaded referenced-object value and an index. Its result is tested for null and otherwise used as readable storage containing an address-valued word. |
| F-dc07b121bb37 | Receives a freshly loaded referenced-object value and the current index after a successful lookup, before the lookup result is dereferenced. Its return value is not used. |
| F-33f2303f9bf9 | Receives the state subobject address at offset 0x14 and a pointer to a temporary containing a value obtained through the lookup result. Its return value is not used. |

The dynamic callback has no supplied opaque function ID. Its identity is determined by the referenced object’s dispatch table, not by a fixed named dependency.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, assume valid addresses, normally returning dependencies, no dependency mutation of relevant state, and small nonnegative endpoints.

1. **No interval requires processing.**  
   Stored boundaries are 10 and 20; requested boundaries are 10 and 20. No lower-side delegation occurs. The upper-side comparison returns immediately. No callback or indexed dependency is invoked.

2. **Both sides require processing.**  
   Stored boundaries are 10 and 20; requested boundaries are 13 and 17. F-3ab4d02fd54b receives endpoints 10 and 12. The second-stage candidate interval is 18 through 20. If the callback returns endpoints 19 and 20, lookups occur at indices 19 and 20, in that order.

3. **Delegation can cover the entire stored interval.**  
   Stored boundaries are 10 and 20; the requested lower boundary is 30 and the requested upper boundary is 40. F-3ab4d02fd54b receives 10 and 20. The second stage returns without dispatching the callback.

4. **State changes from the first dependency are observed.**  
   Initial stored boundaries are 10 and 20; requested boundaries are 13 and 17. If F-3ab4d02fd54b changes offset 0x3C to 17, the second-stage fresh read causes an immediate return.

5. **Either callback sentinel suppresses indexed work.**  
   With a candidate interval of 18 through 20, callback output of 0xFFFFFFFF for either endpoint produces no lookups.

6. **Reversed callback bounds can produce an empty traversal.**  
   With a candidate interval of 18 through 20, callback endpoints 20 and 19 lead to no loop iterations.

7. **Null lookup suppresses later per-index calls.**  
   If the lookup at index 19 returns null, neither F-dc07b121bb37 nor F-33f2303f9bf9 is invoked for index 19. Traversal continues to the next index if the loop condition permits.

8. **The post-call field controls submission.**  
   After a successful lookup and the call to F-dc07b121bb37, a zero field at the referenced value’s offset 0x40 causes submission to F-33f2303f9bf9. A nonzero field suppresses submission, but does not undo the preceding dependency call.

9. **Sentinel candidate prevents dispatch.**  
   If the stored upper boundary is 0xFFFFFFFF and the requested upper boundary is 0xFFFFFFFE, a qualifying stored lower boundary causes the candidate to become 0xFFFFFFFF. The function returns without invoking the callback.

## Uncertainties

**Inferred intent:** The boundary fields and indexed calls suggest processing entries outside a requested interval. This interpretation is not required to reproduce the established behavior.

**documented_only:**

- The application-level meaning and invariants of the boundary fields.
- Whether 0xFFFFFFFF is a universal invalid-index convention or only a local sentinel.
- Valid endpoint ranges that exclude mixed signedness and nontermination cases.
- The callback’s identity and full calling convention.
- Mutation, ownership, allocation, release, and failure behavior of all dependencies.
- Whether F-dc07b121bb37 preserves the lookup-returned storage.
- The meaning of the field at referenced-value offset 0x40.
- Exception, fault, concurrency, reentrancy, and nonlocal-exit behavior.
- Whether observed reversed intervals or extreme signed endpoints are reachable in valid application states.

No runtime tests have been performed.

## Evidence

The established contract derives solely from the supplied static representation of F-3e07bce3ca68 and the supplied opaque dependency mapping.

That evidence establishes explicit field offsets, scalar widths, unsigned and signed comparisons, sentinel checks, dependency argument order, repeated reads, callback-writable temporaries, and loop progression.

It does not establish dependency implementations, broader object definitions, runtime invariants, or successful execution on any concrete input. No external implementation access or runtime validation was used.
