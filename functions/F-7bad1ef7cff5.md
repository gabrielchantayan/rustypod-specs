<!-- Trusted provenance: F-7bad1ef7cff5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7bad1ef7cff5

## Purpose

F-7bad1ef7cff5 processes an indexed sequence whose signed element count is obtained from an input object. For each index, it obtains an element token, obtains a candidate token, initializes or updates local scratch state, and conditionally associates the element with a candidate before invoking an indirect callback.

The established behavior is a sequence of dependency calls, a conditional 32-bit memory write, and callback dispatch. Describing this as filtering, allocation, registration, or ownership transfer would require contracts not present in the supplied evidence.

This contract distinguishes:

- **Established static evidence:** operations and ordering visible in the supplied function.
- **Inferred intent:** plausible interpretations that are not required behavior.
- **Unknown / documented_only:** dependency behavior, memory validity, and other properties not established here.

No runtime tests have been performed.

## Interface

### Inputs

The function accepts four arguments and has no explicit return value.

| Argument | Width and interpretation | Direct use |
|---|---|---|
| Sequence object | A pointer-bearing 32-bit value in the represented interface | Supplies the count at byte offset 12 and an embedded sequence handle at byte offset 8 |
| Context word | Opaque 32-bit value | Initializes the first scratch word and is passed unchanged as a separate argument to F-eecc5603eeba |
| Callback receiver | Pointer to an object with a readable first 32-bit word | Initializes the second scratch word and supplies the receiver for indirect dispatch |
| Initial candidate word | 32-bit value | Initializes the third scratch word, but is replaced before the first per-element initialization call |

The evidence uses 32-bit integer and word accesses. Pointer-bearing values are represented in the same 32-bit environment. A different-width implementation would need to preserve the specified layout and behavior rather than substitute native pointer widths.

No null checks, bounds checks, or object-type checks are visible.

### Sequence-object layout

Only these fields are established:

| Byte offset | Access | Meaning here |
|---|---|---|
| 8 | Address formation | Address passed as the first argument to F-8966abe1b944 |
| 12 | 32-bit read | Signed iteration count, captured once at entry |

The fields may overlap an embedded data structure used by F-8966abe1b944. Its complete layout is unknown.

### Local scratch layout

Three adjacent 32-bit local words form the scratch region exposed through the address of its first word:

| Byte offset | Initial contents | Later role |
|---|---|---|
| 0 | Context word | Scratch state supplied by address to two dependencies |
| 4 | Callback receiver pointer | Additional adjacent scratch state |
| 8 | Initial candidate word | Replaced by each result from F-b2e7bdb1ad7a; subsequently used as the current candidate |

These initializations occur once, before iteration. The first two words are not explicitly restored between iterations.

The function passes the beginning of this region to F-7551853739b9 and F-eecc5603eeba. Their permitted access extent is not established. Their pointer arguments make modification of the adjacent candidate word possible if their contracts allow it; this function supplies no protective copy.

### Candidate and callback layouts

On the accepted path, the current candidate is treated as an address with a writable 32-bit field at byte offset 4. The element token is written to that field.

The callback receiver is treated as an object whose first 32-bit word identifies a dispatch table. The indirect target is loaded from byte offset 28 of that table. It receives:

1. The original callback receiver argument.
2. The address of the current candidate word, corresponding to scratch offset 8.

The callback therefore receives a pointer to a local word, not the candidate value directly.

### Preconditions

For defined execution of the observed path:

- The sequence count must be readable at entry.
- Every dependency invocation must satisfy that dependency’s own, currently unknown contract.
- Any candidate used by the accepted path must identify storage writable at byte offset 4.
- The callback receiver, its dispatch-table pointer, and the target at table offset 28 must be valid when dispatch occurs.
- Dependencies receiving local addresses must not use them beyond their valid lifetime unless some separate, established mechanism makes that safe.

The function does not enforce these conditions or report their violation.

## Behavior

### Entry and loop extent

The function first reads the signed 32-bit count from the sequence object at byte offset 12. This is a snapshot: it is not reread for loop termination.

It then initializes the three local scratch words from the second, third, and fourth arguments.

Indices are processed in ascending order, starting at zero, while the index is less than the captured signed count.

Consequences:

- A zero or negative count produces no dependency calls, candidate writes, or callbacks.
- A positive count of \(n\) produces \(n\) iterations if all invoked operations return normally.
- Changes to the stored count during an iteration do not change the captured loop extent.
- The lookup operation still receives the embedded sequence address on each iteration, so changes to sequence contents may affect later results.
- For a maximum positive signed 32-bit count, the last processed index is one less than that count. The final increment reaches the count without requiring signed overflow.

### Per-element operations

For each index, the observable ordering established by this function is:

1. **Obtain an element token.**  
   F-8966abe1b944 receives the sequence-object address plus 8 and the current signed index. Its 32-bit result is retained as the element token for this iteration.

2. **Obtain a candidate token.**  
   F-b2e7bdb1ad7a is called with no explicit arguments. Its 32-bit result replaces the candidate word at scratch offset 8.

3. **Initialize or update scratch state.**  
   F-7551853739b9 receives the scratch-region address followed by two zero arguments. This call occurs even if the candidate-producing call returned zero.

4. **Check the current candidate word.**  
   After F-7551853739b9 returns, the candidate word is compared with zero. A zero value skips the predicate and the remainder of the accepted path.

5. **Evaluate the predicate when the candidate is nonzero.**  
   F-eecc5603eeba receives:
   - The retained element token.
   - The original context argument.
   - The scratch-region address.

   Any nonzero result admits the accepted path. A zero result skips the candidate-field write and callback.

6. **Write the element token on acceptance.**  
   The function writes the retained 32-bit element token at byte offset 4 from the candidate value then present in the candidate word.

7. **Dispatch the callback.**  
   After that write, the function resolves the indirect target through the original callback receiver and invokes it with the receiver and the address of the candidate word.

The next index is considered only after this sequence completes normally.

### Candidate changes across dependency calls

The nonzero candidate check applies to the value present **after** F-7551853739b9 returns, not necessarily the value returned by F-b2e7bdb1ad7a.

There is no second nonzero check after F-eecc5603eeba. If that dependency changes the candidate word and returns nonzero, the accepted-path write uses the changed value.

For example, if the predicate changes a previously nonzero candidate to zero and returns nonzero, the function does not safely skip the operation. Its next direct write targets byte address 4 under the represented address arithmetic. Whether that faults or has some other effect depends on the execution environment.

The callback can modify the candidate word through its second argument. This function does not subsequently use that modified candidate in the same iteration. The next iteration replaces the candidate word with the next result from F-b2e7bdb1ad7a.

### Scratch persistence and original arguments

The first two scratch words are initialized only once. Any modifications to them by dependencies can persist into subsequent iterations.

They must be distinguished from the original arguments:

- F-eecc5603eeba receives the original context word as its second argument, even if scratch offset 0 has changed.
- Indirect dispatch uses the original callback receiver argument, even if scratch offset 4 has changed.

This does not guarantee stable receiver contents or a stable dispatch target. Dependencies may mutate reachable memory or global state; their effects are unknown.

### Termination and errors

Normal completion returns no explicit status or result.

The function contains no visible:

- Error return.
- Retry.
- Rollback.
- Exception handling.
- Cleanup call.
- Rejection diagnostic.
- Release of candidates or element tokens.

A dependency that fails to return normally prevents the remaining operations from completing normally. No recovery behavior is established.

## Side effects

### Directly established effects

The function directly performs:

- One entry-time count read.
- Local scratch initialization and updates.
- One candidate-field write for each accepted iteration.
- Receiver and dispatch-table reads for each callback invocation.
- Calls to the listed dependencies and an indirect callback.

The candidate-field write precedes resolution and invocation of the callback. If that write aliases the receiver or its dispatch table, callback resolution may observe the changed memory. No non-aliasing rule is visible.

### Ownership and lifetime

No ownership transfer is established.

In particular:

- F-b2e7bdb1ad7a is not proven to allocate memory.
- A candidate is not proven to be newly created, uniquely owned, or releasable.
- Rejected candidates receive no explicit release call from this function.
- Writing an element token into a candidate is not proof of retaining or transferring ownership of the element.
- Passing the address of the candidate word to the callback is not proof that the callback consumes or releases the candidate.

The scratch region has local lifetime. Retaining its address or the callback’s candidate-word pointer beyond this invocation is not made safe by the visible behavior.

### Indirect effects

Every dependency may have additional effects not shown here, including mutation of globals or reachable memory. This contract does not assume that any unknown callee preserves globals, receiver state, sequence state, or memory ownership.

## Dependencies

| Opaque ID | Established invocation | Result used by this function |
|---|---|---|
| F-8966abe1b944 | Embedded sequence address and current index | 32-bit element token |
| F-b2e7bdb1ad7a | No explicit arguments | 32-bit candidate token |
| F-7551853739b9 | Scratch-region address, zero, zero | No result is consumed |
| F-eecc5603eeba | Element token, original context word, scratch-region address | Zero versus nonzero controls acceptance |

The indirect callback has no supplied opaque function ID. Its identity is determined at runtime through the receiver’s dispatch table at byte offset 28.

All four named dependencies are invoked in the order described above. The predicate is conditional; the first three calls occur on every normally reached iteration.

The callback’s return value, if any, is not consumed.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests.

### Nonpositive count

**Given:** The entry count is zero or negative, and the entry count read is valid.

**Required:** No dependency calls, candidate-field writes, or callback dispatches occur. The function returns normally.

### Candidate remains zero after initialization

**Given:** A positive count, and on one iteration the candidate word is zero after F-7551853739b9 returns.

**Required:** That iteration still performs the lookup, candidate-producing call, and initialization call. It does not invoke F-eecc5603eeba, write the candidate field, or invoke the callback.

### Initialization supplies a nonzero candidate

**Given:** F-b2e7bdb1ad7a returns zero, but F-7551853739b9 changes the candidate word to a valid nonzero address.

**Required:** The predicate is invoked. Its result controls acceptance. The initial zero result alone does not force rejection.

### Predicate rejects

**Given:** The candidate is nonzero at the gate, and F-eecc5603eeba returns zero.

**Required:** No direct candidate-field write or callback occurs for that iteration. No direct cleanup or release follows.

### Accepted element

**Given:** The candidate is nonzero at the gate, the predicate returns nonzero, and all required memory and callback accesses are valid.

**Required:** The retained element token is written as a 32-bit word at candidate offset 4. Only afterward is the callback resolved and invoked with the original receiver and the address of the candidate word.

### Predicate substitutes a candidate

**Given:** The candidate is initially a valid address A. The predicate changes the candidate word to a different valid address B and returns nonzero.

**Required:** The direct field write targets B plus 4, not A plus 4.

### Count changes during processing

**Given:** The captured count is two, and a dependency changes the object’s stored count to zero during the first iteration.

**Required:** A second iteration remains required if the first completes normally. Its lookup uses index one; the count is not reread to suppress it.

### Scratch context changes

**Given:** A dependency changes scratch offset 0.

**Required:** The modified scratch value can persist into the next iteration. Nevertheless, the predicate’s separate context argument remains the original context word.

## Uncertainties

The following are **documented_only**:

- The semantic type of the sequence, element token, context word, and candidate.
- The full layout and invariants of the embedded sequence structure.
- Whether the scratch region is a formal three-word record or merely storage interpreted by dependencies.
- Which scratch words each dependency reads or writes.
- Whether any dependency allocates, retains, releases, or invalidates memory.
- Whether dependency calls preserve global state or receiver contents.
- Whether reentrancy, concurrency, or exceptional control flow is supported.
- Whether receiver and candidate aliasing is valid in intended use.
- The indirect callback’s identity, return convention, and ownership contract.
- Platform behavior for invalid, misaligned, wrapped, or otherwise unusable addresses.

**Inferred intent:** The operation resembles per-element acceptance followed by attachment to a candidate and notification through a receiver. That interpretation does not establish allocation, successful registration, or ownership transfer.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

The evidence establishes:

- A single signed 32-bit count snapshot at object offset 12.
- Ascending zero-based iteration.
- A sequence address formed at object offset 8.
- Three adjacent 32-bit scratch words.
- The dependency order and conditional predicate invocation.
- A zero/nonzero predicate decision.
- A 32-bit accepted-path write at candidate offset 4.
- Callback dispatch through receiver table offset 28.
- The absence of visible cleanup, error reporting, and explicit return data.

Dependency implementations and callback behavior were not supplied. No runtime tests have been performed.
