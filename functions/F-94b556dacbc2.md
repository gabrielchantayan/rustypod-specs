<!-- Trusted provenance: F-94b556dacbc2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-94b556dacbc2

## Purpose

F-94b556dacbc2 conditionally updates one indexed record within a caller-supplied state region. Its behavior depends on a state byte:

- When the byte equals 2, it either decrements a nonzero counter or, when the counter is zero, attempts a record-position transition followed by counter initialization and a dependent operation.
- For every other byte value, it increments the counter, obtains a comparison threshold, and attempts a record-position transition when the updated counter reaches or exceeds that threshold.

The function returns either 0 or 3. These values describe the selected control path, not whether state changed: a return of 3 can follow writes or dependency-induced changes, and a return of 0 need not involve a position transition.

Descriptions such as “position,” “counter,” and “threshold” identify operational roles. Their application-level meanings are inferred, not established.

## Interface

### Inputs

The interface contains four argument slots:

| Argument | Established role |
|---|---|
| State base | A 32-bit address-like value used to access the state region and indexed records. |
| Record index | A signed 32-bit value multiplied by 80 to select a record-relative address. |
| Unused argument | A 32-bit value that is not read or forwarded by this function. |
| Initial threshold | A signed 32-bit value copied into private writable storage before any dependency call. A dependency may replace it. |

No argument validation is visible. The function does not reject a null base, a negative index, an out-of-range index, or invalid stored values before accessing memory.

### Result

The result is a 32-bit value:

- **0:** a branch-specific success or continuation condition was met.
- **3:** a branch-specific refusal or failure condition occurred.

The function does not propagate dependency return values. It tests them for zero versus nonzero and maps the relevant outcomes to 0 or 3.

### State layout

Let **B** denote the state base, **I** the record index, and **R** the derived address B + 80 × I. The offsets below are byte offsets.

| Address | Width | Operational role |
|---|---:|---|
| B + 0x1060 | 32 bits | Context value passed to position-transition dependencies. |
| B + 0x1393 | 8 bits | Mode selector; only the value 2 selects the special branch. |
| R + 0x1240 | 32 bits | Boundary position used in the mode-2 zero-counter path. |
| R + 0x124c | 32 bits | Current position; also passed by address to transition dependencies. |
| R + 0x1250 | 32 bits | Signed counter, read and written directly. |
| R + 0x1280 | 32 bits | Comparison position used to decide whether to set a marker. |
| R + 0x1285 | 8 bits | Marker that this function can set to 1. |

These are relative accesses, not a complete structure definition. The 80-byte index stride does not establish record-count limits, allocation boundaries, ownership, or disjointness between all indexed fields and base-relative fields.

### Preconditions

For defined execution, all accesses taken on the selected path must address valid memory of suitable width and alignment. Dependency calls must also accept the supplied values and addresses.

The visible operations use 32-bit address and integer quantities. Address formation, counter increment or decrement, and threshold-minus-one calculations require particular care at representational limits. The supplied static representation does not establish whether overflowing operations wrap, trap, or have another language-level interpretation. Acceptance claims below therefore assume representable arithmetic unless stated otherwise.

## Behavior

### Entry sequence

The function first derives R, then reads the 32-bit counter at R + 0x1250. It copies the fourth argument into its private threshold storage and reads the mode byte at B + 0x1393.

The counter read therefore occurs before the mode read. The selected branch is not reconsidered if a dependency later changes the mode.

The derived record address is retained for subsequent direct accesses; it is not recomputed from dependency-modified state.

### Mode equals 2, entry counter is nonzero

If the initially read counter is nonzero:

1. The function subtracts one from that captured value.
2. It writes the result to R + 0x1250.
3. It returns 0.

This branch makes no dependency calls and does not directly read the current position, boundary position, comparison position, or context value.

The condition is **nonzero**, not positive. A negative entry counter also takes this path and is decremented, subject to the arithmetic limitations above.

### Mode equals 2, entry counter is zero

The function reads the current position at R + 0x124c and compares it with the boundary position at R + 0x1240.

#### Current position equals the boundary

The function returns 3 immediately.

It makes no dependency calls and performs no direct state writes on this path. It does not inspect the comparison position or marker.

#### Current position differs from the boundary

The function reads the comparison position at R + 0x1280. If it equals the captured current position, the function writes the byte value 1 to R + 0x1285.

That marker write occurs before any dependency call. There is no corresponding marker-clearing operation anywhere in this function.

The function then calls **F-ce5add3994af**, supplying:

- The context value read from B + 0x1060.
- The boundary position read for the call from R + 0x1240.
- The current-position value captured before the comparison and possible marker write.
- The address R + 0x124c as a writable argument.

The boundary is read for the earlier comparison and again as a call argument. Likewise, the current-position argument is a captured value, not a fresh read at the call. These distinctions can matter if fields alias or memory changes independently.

If F-ce5add3994af returns nonzero, F-94b556dacbc2 returns 3. It performs no rollback of the marker or any dependency writes.

If F-ce5add3994af returns zero, the function calls **F-5a8b0ee5bea0** with:

- The state base.
- A fresh 32-bit read of the current position.
- The address of the private threshold storage, initially containing the fourth argument.

Thus, this second dependency receives the position present after the transition call, not necessarily the entry position.

If F-5a8b0ee5bea0 returns nonzero, the function returns 3. It does not directly initialize the counter on this path.

If F-5a8b0ee5bea0 returns zero:

1. The function reads its private threshold storage.
2. It writes threshold minus one to the counter.
3. It calls **F-d3ad26dbcfec** with the state base and original record index.
4. It returns 0 if that dependency returns zero, otherwise 3.

The counter write precedes F-d3ad26dbcfec. A nonzero result from that dependency does not undo the write. Conversely, there is no counter write after that dependency, so any counter change it makes is not overwritten by this function.

### Mode is any value other than 2

The function first writes the captured entry counter plus one to R + 0x1250.

It then calls **F-5a8b0ee5bea0**, supplying the state base, the current position read for that call, and the address of the private threshold storage.

If the dependency returns nonzero, the function returns 3. The earlier counter increment is not rolled back.

If the dependency returns zero, the function rereads the counter from memory and compares it with the threshold storage using a signed 32-bit less-than comparison.

This reread is significant: the comparison uses the counter present after the dependency call, not necessarily the incremented value.

#### Counter is less than the threshold

The function returns 0 without a position-transition call or any further direct write.

#### Counter is greater than or equal to the threshold

The function calls **F-8fce2b40d724**, supplying:

- The context value read from B + 0x1060.
- The current position read for this call.
- The address R + 0x124c as a writable argument.

If this dependency returns nonzero, the function returns 3. It does not directly reset the counter.

If the dependency returns zero, the function writes 0 to the counter and returns 0. This reset occurs after the dependency returns and overwrites any counter value the dependency may have installed.

The mode-2 marker and boundary checks do not occur in this branch.

## Side effects

### Direct writes

Depending on the path, this function directly performs:

- A one-byte marker write of 1.
- A counter decrement.
- A counter increment.
- A counter assignment to threshold minus one.
- A counter reset to zero.

It does not directly assign a new current-position value. Instead, transition dependencies receive that field’s address.

No direct allocation, deallocation, resource acquisition, or resource release is visible.

### Dependency-mediated changes

Dependencies receive either the state base, a context value, a writable current-position address, or combinations of these. Their full side effects are not established.

In particular:

- A zero dependency result does not establish that unrelated state was preserved.
- A nonzero dependency result does not establish that no state changed.
- Context-value ownership and whether it denotes an owned resource are unknown.
- The position address does not, by itself, establish a transfer of ownership.
- Dependencies are not known to preserve global state.

The threshold address refers to invocation-local storage. It is available during the dependency call; this function provides no persistent storage for that object after return. Whether F-5a8b0ee5bea0 retains the address is unknown.

### Failure persistence

Return value 3 does not imply transactional failure. Examples of changes that may already have occurred include the marker write, counter increment, threshold-minus-one write, and mutations made by dependencies.

There is no visible rollback or compensating call.

## Dependencies

| Opaque ID | Visible interaction |
|---|---|
| **F-ce5add3994af** | Called only in mode 2 with a zero entry counter and unequal current and boundary positions. Receives context, boundary, captured current position, and a writable current-position address. Zero permits further processing; nonzero leads to 3. |
| **F-5a8b0ee5bea0** | Receives state base, current-position value, and writable threshold storage. Called after a successful mode-2 transition or after the non-mode-2 counter increment. Zero permits further processing; nonzero leads to 3. |
| **F-d3ad26dbcfec** | Called only after the mode-2 transition and threshold calls both return zero and the counter has been set to threshold minus one. Receives state base and index. Zero maps to 0; nonzero maps to 3. |
| **F-8fce2b40d724** | Called only outside mode 2 after a successful threshold call and a signed comparison showing counter ≥ threshold. Receives context, current position, and its writable address. Zero causes a subsequent counter reset; nonzero leads to 3 without that direct reset. |

No stronger dependency contracts are established. Exact evaluation order among separate arguments of a single call is not established; the call ordering and surrounding direct writes described above are established.

## Acceptance examples

These are conditional acceptance cases, not observed executions. Each assumes valid memory, representable arithmetic, and no unmentioned dependency mutations or concurrent changes.

1. **Simple mode-2 decrement.**  
   Mode is 2 and the entry counter is 5. The function writes 4, calls no dependencies, and returns 0.

2. **Negative mode-2 counter.**  
   Mode is 2 and the entry counter is −2. The function writes −3 and returns 0. It does not treat the negative value as a request to transition.

3. **Mode-2 boundary refusal.**  
   Mode is 2, the counter is zero, and current position equals boundary position. The function returns 3 without direct state writes or dependency calls.

4. **Marker persists after transition refusal.**  
   Mode is 2, the counter is zero, current differs from boundary, and current equals comparison position. The marker is set to 1 before F-ce5add3994af is called. If that dependency returns nonzero, the function returns 3 with the marker still set unless the dependency changed it.

5. **Successful mode-2 initialization.**  
   The zero-counter transition path is eligible. F-ce5add3994af returns zero; F-5a8b0ee5bea0 returns zero with threshold 7. The function writes counter 6 before calling F-d3ad26dbcfec. Its final result is 0 or 3 according to whether that final dependency returns zero or nonzero.

6. **Threshold zero is not rejected.**  
   In the preceding path, a successful threshold call leaves threshold 0. The function writes −1 to the counter. There is no positivity check.

7. **Increment survives threshold failure.**  
   Mode is not 2 and the entry counter is 4. The function writes 5 before calling F-5a8b0ee5bea0. If that call returns nonzero, the result is 3 and no direct rollback occurs.

8. **Below-threshold continuation.**  
   Mode is not 2, the counter becomes 5, and the threshold call succeeds with threshold 6. The function returns 0 without calling F-8fce2b40d724.

9. **Equality triggers transition.**  
   After a successful threshold call, counter and threshold are both 6. The function calls F-8fce2b40d724. If it returns zero, the function writes counter 0 and returns 0.

10. **Dependency mutation affects comparison.**  
    Outside mode 2, the increment produces 5, but F-5a8b0ee5bea0 changes the counter to 9 and supplies threshold 8. The subsequent comparison uses 9, so the transition call is attempted.

## Uncertainties

### Inferred intent

The two transition dependencies may represent opposite traversal directions, and the threshold may represent a duration or span. These interpretations are plausible from control flow but are not required by the contract.

### documented_only

The following remain unknown:

- The permitted index range and complete state layout.
- Endianness, alignment guarantees, and arithmetic-overflow semantics.
- Dependency effects beyond the visible arguments and return-value tests.
- Whether dependencies always write threshold storage on success.
- Ownership and lifetime rules for the context and position values.
- The semantic meanings of status 3 and the marker.
- Concurrency, synchronization, reentrancy, and asynchronous mutation guarantees.
- Fault behavior for invalid addresses or dependency failures outside normal returns.

If F-5a8b0ee5bea0 leaves threshold storage unchanged, the fourth argument remains the threshold. This follows from local initialization; it is not a claim that the dependency actually behaves that way.

## Evidence

The established behavior comes solely from static inspection of the supplied function representation: field access widths and offsets, branch conditions, direct writes, dependency arguments, call order, signed comparison, and zero/nonzero return handling.

No dependency implementations were supplied. No runtime tests have been performed. The acceptance examples specify conditional outcomes rather than reporting executed results.
