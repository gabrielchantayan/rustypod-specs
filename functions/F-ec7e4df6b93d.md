<!-- Trusted provenance: F-ec7e4df6b93d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec7e4df6b93d

## Purpose

F-ec7e4df6b93d processes a zero-terminated byte sequence using mutable context and shared state. It obtains temporary parser storage, requests tokenization, delivers selected categories to a parser dependency, handles diagnostics, and performs conditional cleanup.

Its result is a displayed 64-bit composition, not simply the context status:

- The low 32 bits contain a local outcome accumulator.
- The high 32 bits contain the input address initially, or the context address after particular parser-delivery paths.

“Parser,” “diagnostic,” and “release-like” describe apparent roles. They do not establish dependency semantics or ownership contracts.

This contract distinguishes explicit static behavior from inferred intent and unresolved behavior. No runtime tests have been performed.

## Interface

### Arguments and address model

The supplied representation exposes four 32-bit arguments:

1. **Context address:** mutable storage containing the fields below.
2. **Input address:** a byte-oriented, zero-terminated sequence, despite its displayed pointer type.
3. **Diagnostic-slot address:** optional address of a mutable 32-bit value.
4. **Uninterpreted value:** initially assigned to temporary storage that is overwritten by the allocation result before any visible use. No behavioral dependence is established.

Addresses and stored pointer values are 32 bits. Input advancement is measured in bytes.

### Referenced context layout

| Byte offset | Width | Observed role |
|---|---:|---|
| 0x000 | 32 bits | Shared-state address |
| 0x004 | 32 bits | Status |
| 0x008 | 32 bits | Pending diagnostic |
| 0x00c | 32 bits | Resource-active indicator |
| 0x013 | 8 bits | Cleanup-suppression flag |
| 0x040 | 32 bits | Resource-associated value, tested for signed positivity |
| 0x13c | 32 bits | Field cleared after cleanup involving 0x140 |
| 0x140 | 32 bits | Value supplied to F-94cb53877c15 |
| 0x158 | 32 bits | Value supplied to F-94cb53877c15 |
| 0x170 | 32 bits | Current token-start address |
| 0x174 | 32 bits | Packed token metadata |
| 0x178 | 32 bits | Input-start address |
| 0x17c | 32 bits | Input boundary/progress address |
| 0x180 | 32 bits | Conditional argument to F-da1a289a3277 |
| 0x184 | 32 bits | Argument to F-188f252dfa34 |
| 0x198 | 32-bit field, low byte tested | Controls the call involving 0x180 |
| 0x1a0 | 32 bits | Value supplied to F-94cb53877c15 |

The shared-state object has these referenced fields:

| Byte offset | Width | Observed role |
|---|---:|---|
| 0x01e | 8 bits | Stop/failure flag |
| 0x054 | 32 bits | Captured position limit |
| 0x090 | 32 bits | Controls interruption reset |
| 0x0d8 | 32 bits | Interruption indicator |

The context’s shared-state address is captured at entry. Later shared-state accesses use that captured address; changing context offset 0x000 does not itself redirect them.

### Preconditions and output

Every address actually accessed must permit the corresponding read or write. This includes the input byte inspected after scanning stops. There are no explicit null checks for context, shared state, or input.

The diagnostic slot is optional for some branches, but its address, including zero, is passed directly to F-2cb24bca4183 on the interruption path.

A nonzero allocation result must permit a 32-bit write at its start and subsequent dependency operations. Required allocation size, alignment beyond visible accesses, dependency null tolerance, and aliasing restrictions are unresolved.

The result accumulator starts at zero. Allocation failure sets it to 7; an unrecognized category sets it to 1; handling a nonzero pending diagnostic increments it once. Context status is maintained separately.

## Behavior

### Entry initialization and allocation

The following observable order is established:

1. Read the shared-state address from context offset 0x000.
2. Read and retain the limit at shared offset 0x054.
3. Read shared offset 0x090; if zero, clear shared offset 0x0d8.
4. Clear context status at 0x004.
5. Write the input address to context offset 0x178.
6. Read and retain a global configuration value.
7. Write the input address to context offset 0x17c.
8. Call F-391e3d30e634 with the retained configuration value.

In particular, the configuration read precedes the boundary write. The allocator receives that captured value, not a later reread. This ordering matters if storage aliases or dependencies modify global state.

The position and accumulator start at zero. A reusable 32-bit category slot initially contains the diagnostic-slot address. The tokenizer later receives the address of this slot. It is not reset between tokenization calls.

The high-result address initially holds the input address. The pending diagnostic at context offset 0x008 is not cleared.

If allocation returns zero, the function writes byte value 1 to shared offset 0x01e, sets the accumulator to 7, and returns immediately. No parser disposal, diagnostic reconciliation, or remaining cleanup occurs. This branch does not write status 7; the allocator could nevertheless have modified status after its entry reset.

Otherwise, the first 32-bit word of temporary storage is initialized to all ones, interpreted as signed −1. The category-history value also starts as all ones.

### Scanning and token metadata

Each iteration first reads the shared stop byte. If nonzero, scanning ends without that iteration’s input-byte check. Otherwise, the current input byte is read; zero ends scanning.

For a nonzero byte:

- Write its address to context offset 0x170.
- Call F-86b0b8e613c4 with that address and the address of the reusable category slot.
- Read context metadata at 0x174 after the call and retain its low bit.
- Form 32-bit metadata from the tokenizer’s return value shifted left by one, combined with the retained bit.
- Advance the 32-bit position by the return value masked with 0x7fffffff.
- Store the packed metadata at 0x174.
- Compare the resulting position with the captured limit using signed 32-bit interpretation.

The return value’s top bit contributes neither to advancement nor to the stored shifted metadata. The function establishes no meaning for that bit.

If the signed position exceeds the signed limit, status becomes 0x12 and scanning stops before category handling. Equality does not trigger this branch. The position and metadata have already changed.

There is no local requirement that advancement be positive. There is no explicit overflow guard. The arithmetic is 32-bit, not an unbounded mathematical sum; behavior outside valid, representable input traversal is unresolved.

The category slot is consumed whether or not the tokenizer writes it. On the first call, an unchanged slot still contains the diagnostic-slot address. On later calls, it retains its prior contents unless modified.

### Category handling and ordinary parser delivery

Category handling follows the limit check.

**Category 1:** write the advanced input address to context offset 0x17c, then perform ordinary parser delivery.

**Category 0x90:** stop with an unrecognized-token condition. If a diagnostic slot exists:

1. Read its current value and pass it to F-94cb53877c15, including when that value is zero.
2. Call F-1983453691c7 with the captured shared-state address, a diagnostic template, and the descriptor beginning at context offset 0x170.
3. Store the low 32 bits of that dependency’s displayed result in the caller’s diagnostic slot.

The accumulator becomes 1 whether or not a diagnostic slot exists. This branch does not itself assign a context status.

**Categories 0x91 and 0x93:** do not perform ordinary parser delivery. Read shared offset 0x0d8. If zero, continue scanning. Otherwise, set status to 9, call F-2cb24bca4183 with the diagnostic-slot address, an interruption-related template, and zero, then stop scanning.

**Every other category:** perform ordinary parser delivery.

Ordinary delivery has four displayed arguments to F-be5a383a3711:

1. Temporary parser-storage address.
2. Current category-slot value.
3. Context token-start value at 0x170.
4. The packed metadata computed for this token.

Immediately before this call, the high-result address becomes the context address.

After the call, category history receives the category slot’s **then-current** value, followed by a context-status read. Nonzero status stops scanning.

Consequently, history is not unconditionally the category passed to the parser. The tokenizer has received the slot’s address; a dependency retaining that address could modify the slot during parser delivery. Categories skipped without parser delivery do not directly update history.

### End-of-input delivery

After scanning ends, the function reads the current input byte even if scanning stopped because the shared stop byte was nonzero.

Finalization is eligible only when:

- This byte is zero.
- The accumulator is zero.
- Context status, read after those conditions succeed, is zero.

The stop byte is not independently rechecked as a finalization condition.

If history is not 1, F-be5a383a3711 receives a synthetic delivery with **five displayed arguments**:

1. Temporary parser-storage address.
2. Category 1.
3. Current context token-start value at 0x170.
4. Current context metadata at 0x174.
5. Context address.

After that call, context offset 0x17c is set to the current input address.

Every eligible finalization then delivers category 0 through a four-argument call: temporary storage, zero, current token-start value, and current metadata. Immediately before this end-marker call, the high-result address becomes the context address.

There is no intervening status check. A synthetic delivery that changes status does not by itself suppress the end marker. Descriptor fields are read for the later call rather than assumed unchanged across the synthetic call.

On initially empty input, this function has not initialized offsets 0x170 or 0x174 before these deliveries. Their existing values, including dependency modifications, are used.

### Temporary parser disposal

After scanning and any finalization, a global indirect disposal target is captured **before** parser draining.

While the first 32-bit word of temporary storage is signed-nonnegative, F-552df11c9525 is called with that storage address. The word is reread to decide whether to repeat.

Once the word is signed-negative, the previously captured indirect target is called with the storage address. Changes to the global function-pointer location during draining do not select a new target for this invocation.

No iteration bound exists. Progress depends on dependency behavior. A null-storage guard is displayed here, although successful allocation was required to reach this phase.

### Diagnostic reconciliation

After disposal, read the shared stop byte again. If nonzero, overwrite context status with 7.

If status is neither zero nor 0x65 and the pending diagnostic is zero, call F-d5aed6d90713 with the current status. Pass its result to F-2cb24bca4183 with the address of context offset 0x008 and zero.

A nonzero pending diagnostic is then handled regardless of status:

- If the caller supplied no diagnostic slot, pass the pending value to F-94cb53877c15.
- If the caller slot currently contains a nonzero value, likewise pass the pending value to F-94cb53877c15.
- Otherwise, write the pending value into the empty caller slot.

After that operation, increment the accumulator and clear context offset 0x008.

Thus, an existing caller diagnostic takes precedence at this decision point. Release-like calls are not assumed to preserve either slot, but the explicit pending-field clear still follows the call. Diagnostic construction is not guaranteed to produce a nonzero value.

### Remaining cleanup and status normalization

The conditional cleanup has two distinct suppression checks when F-e9635d699624 is called.

If the active indicator at 0x00c is nonzero and the field at 0x040 is signed-positive, read the suppression byte at 0x013:

- If nonzero, skip both this conditional cleanup call and the cleanup involving 0x140/0x13c.
- If zero, call F-e9635d699624, then explicitly clear the active indicator. **Read the suppression byte again.** Only if this second read is zero, pass the current 0x140 value to F-94cb53877c15, then clear 0x140 followed by 0x13c.

If the active indicator is zero, or 0x040 is zero or signed-negative, do not call F-e9635d699624. Instead, read the suppression byte directly to decide whether to perform the 0x140/0x13c cleanup.

The first zero suppression read does not guarantee the later cleanup: F-e9635d699624 may change that byte.

The following operations then occur in order:

1. Pass the current 0x1a0 value to F-94cb53877c15.
2. Read the low byte of the field at 0x198; if zero, pass the current 0x180 value to F-da1a289a3277.
3. Pass the current 0x184 value to F-188f252dfa34.
4. Pass the current 0x158 value to F-94cb53877c15.

These fields are not explicitly cleared. Earlier calls may affect values or conditions read later.

Finally, if the accumulator is nonzero and status is zero or 0x65, write status 1. Return the composed 64-bit result.

## Side effects

Explicit writes affect context fields, shared-state fields, temporary storage, and potentially the caller’s diagnostic slot. No direct input-buffer write is visible.

Temporary storage is obtained, initialized, drained, and handed to an indirect dependency. Several values are supplied to F-94cb53877c15 on release-like paths. These observations do not prove allocation provenance, deallocation, reference counting, or null safety.

Writing a pending diagnostic into an empty caller slot suggests ownership transfer, but its external ownership contract remains unresolved. Only the explicitly identified fields are cleared.

Dependencies may mutate accessible context, shared state, globals, input, or retained storage. No preservation guarantee is established. Aliasing can make an explicit write affect another described field or later read; the contract does not silently exclude it.

A status-only failure can leave the accumulator zero if diagnostic construction leaves no pending value. Conversely, a pending diagnostic can increment the accumulator even with status zero or 0x65.

## Dependencies

| Opaque ID | Visible role |
|---|---|
| F-391e3d30e634 | Receives captured global configuration; returns temporary storage |
| F-86b0b8e613c4 | Receives input position and writable category-slot address; returns packed advancement information |
| F-be5a383a3711 | Receives ordinary, synthetic category-1, and category-0 deliveries |
| F-94cb53877c15 | Receives diagnostic/resource values on release-like paths |
| F-1983453691c7 | Produces a displayed result whose low half replaces a caller diagnostic |
| F-2cb24bca4183 | Receives interruption or status-derived diagnostic requests |
| F-552df11c9525 | Processes storage while its first word remains signed-nonnegative |
| F-d5aed6d90713 | Produces a value used in status-derived diagnostic construction |
| F-e9635d699624 | Conditional cleanup call with no displayed explicit arguments |
| F-da1a289a3277 | Conditionally receives the field at 0x180 |
| F-188f252dfa34 | Receives the field at 0x184 |

A further indirect target receives temporary storage after draining. No opaque function ID is supplied for that target.

## Acceptance examples

These are conditional static expectations, not executed tests. Except where stated, dependencies preserve relevant state, storage does not alias, and input traversal remains valid.

1. **Allocation failure:** allocation returns zero. The stop byte becomes 1; the low result is 7 and the high result is the input address. No later cleanup occurs. Status remains zero only if allocation did not modify it.

2. **Configuration capture:** the configuration location aliases context offset 0x17c. The allocator receives the value observed after the 0x178 write but before the 0x17c write, not necessarily the input address subsequently stored there.

3. **Empty input:** allocation succeeds, stop and status remain zero, and the first input byte is zero. The parser receives synthetic category 1 with the context as its fifth displayed argument, followed by category 0. Existing descriptor fields supply the token data.

4. **Category slot not written:** the tokenizer leaves its writable slot unchanged. The first category decision uses the diagnostic-slot address as a 32-bit category value; later decisions reuse the retained slot value.

5. **History changed during delivery:** category 1 is passed to ordinary delivery, but a dependency changes the exposed category slot to 2 before that call returns. History becomes 2. At otherwise eligible end of input, a synthetic category 1 is delivered.

6. **Trailing category 1:** ordinary delivery leaves the category slot equal to 1. Subsequent skipped categories do not change history. Eligible finalization sends category 0 without a synthetic category 1.

7. **Limit boundary:** a representable signed position equal to the captured limit passes the limit check. A signed-greater position sets status 0x12 and prevents that token’s delivery.

8. **Diagnostic replacement:** category 0x90 with a caller slot causes its old value to reach F-94cb53877c15 before the replacement is constructed and stored. The accumulator becomes 1; subsequent pending-diagnostic handling can increase it to 2.

9. **Captured disposal target:** the global target is A when captured. A draining call changes the global location to B. Once draining terminates, A is called.

10. **Suppression changes during cleanup:** active is nonzero, 0x040 is signed-positive, and suppression initially equals zero. F-e9635d699624 changes suppression to nonzero. The active indicator is still cleared, but 0x140 is not subsequently passed to F-94cb53877c15 and 0x140/0x13c are not explicitly cleared.

11. **Interruption gating:** category 0x91 or 0x93 with nonzero interruption sets status 9 and invokes F-2cb24bca4183 before stopping. The same indicator alone does not establish a check for another category.

12. **Caller diagnostic precedence:** with a nonzero pending diagnostic and an occupied caller slot, reconciliation passes the pending value to F-94cb53877c15, increments the accumulator, and clears the pending field rather than explicitly replacing the caller slot.

## Uncertainties

The following remain **documented_only**:

- Public API meaning of the upper result half and whether callers consume all 64 bits.
- Actual calling conventions beyond the displayed four- and five-argument parser calls.
- External token names, diagnostic text, and dependency semantics.
- Whether the tokenizer always writes its category slot or retains its address.
- Dependency preservation, ownership, allocation, release, and null-argument contracts.
- Temporary-storage layout beyond the first word.
- Intended handling of zero advancement, arithmetic overflow, invalid input storage, aliasing, or concurrent mutation.
- Whether parser draining terminates.
- Whether the signed-positive field at 0x040 represents an address constraint or another convention.

These unknowns do not erase explicit argument values, read ordering, repeated suppression checks, captured-target behavior, or post-call category-slot reads.

## Evidence

Established evidence consists of the supplied static representation: access widths and offsets, argument values, conditional branches, arithmetic widths, call ordering, and 64-bit result composition. The supplied dependency mapping supports the opaque IDs used above.

Purpose and resource terminology are inferred intent. Dependency implementations, public ownership rules, and runtime behavior are not established.

No implementation access, dependency inspection, runtime execution, or tests were used. The acceptance examples express conditional consequences of static evidence, not observed verification results.
