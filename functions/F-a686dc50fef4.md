<!-- Trusted provenance: F-a686dc50fef4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a686dc50fef4

## Purpose

F-a686dc50fef4 consumes a pending-work indicator and performs mode-dependent dependency calls. In nonzero mode, it additionally obtains a context and either processes an existing resource or obtains a replacement resource that may be formatted and registered with an associated object.

Established behavior includes the order of direct memory accesses, dependency calls, resource-field updates, region metadata writes, and conditional indirect invocation. It does not establish allocation, deallocation, ownership transfer, synchronization, or notification semantics.

“Resource,” “region,” and “registration” are descriptive terms for the visible data flow. Their broader meaning remains **documented_only**.

## Interface

### Input and completion

The sole explicit input is a 32-bit integer, called **mode**:

- Zero selects the short path.
- Every nonzero value selects the extended path, including negative values.

There is no explicit return value. Normal completion occurs through an early return or after the final conditional indirect call.

An integrity-failure dependency is represented as nonreturning. Behavior when other dependencies fail, do not return, or transfer control unexpectedly is not established.

### External state and widths

The function uses:

- A global pointer to a one-byte pending indicator.
- A global 32-bit region extent, read at two distinct points on the replacement path.
- A global indirect-call target.
- A context address supplied by F-fdd8589eca2d.
- An associated object, candidate region, and optional callback-gating record.

The supplied representation uses 32-bit addresses and four-byte integer fields except where a byte access is explicitly identified. Byte order is not established.

For region calculations below, **B** is the retained candidate address and **L** is the first extent value read after F-1375398b8c0d returns. A later read of that same global supplies an argument to F-4e23d415fadf; it does not replace L.

### Context fields

| Offset | Access width | Role |
|---|---:|---|
| +0x14 | 32 bits | Signed status; −1 selects existing-resource processing |
| +0x18 | 32 bits | Existing resource or newly published dependency result |
| +0x1C | 32 bits | Associated-object address |

The context is dereferenced without an explicit null check.

### Associated-object fields

| Offset | Access width | Role |
|---|---:|---|
| +0x04 | 32 bits | Accumulated registered extent |
| +0x10 + 8 × index | 32 bits | Registered region address |
| +0x14 + 8 × index | 32 bits | Registered region extent |
| +0xB0 | 32 bits | Registration count; capacity test is unsigned |
| +0xB4 | 8 bits | Final-call gate, enabled only by exactly 1 |
| +0xB8 | 32 bits | Address of a record whose first word is tested |
| +0xC4 | 32 bits | Alternate region address in an unreachable selection |
| +0xC8 | 32 bits | Alternate region extent in that selection |
| +0xCC | 8 bits | First condition of that selection |

These are observed field accesses, not a complete object definition. Neither non-aliasing nor preservation across dependency calls is established.

## Behavior

### Pending indicator and initial calls

The function reads the byte addressed by the pending-indicator global.

If it is zero, the function returns without any dependency call or direct write.

Otherwise, it writes zero to the pending byte before calling any dependency. The particular nonzero byte value has no further significance. No later direct write restores the indicator.

After consuming the indicator:

- Zero mode calls F-da86f56f2dd4 once.
- Nonzero mode calls F-da86f56f2dd4 twice.
- Both modes then call F-65576a99eee9 once.

The initial dependency results do not contribute to subsequent resource processing.

Zero mode returns after F-65576a99eee9. It does not obtain a context. Nonzero mode continues.

These call counts assume the dependencies return normally. Their effects on globals or reachable memory are unknown.

### Context selection

The extended path obtains a context address from F-fdd8589eca2d.

It first reads the associated-object address at context offset +0x1C:

- If zero, it selects existing-resource processing without reading the status field.
- If nonzero, it reads the signed 32-bit status at +0x14.
- Status −1 selects existing-resource processing.
- Every other status selects replacement processing.

The associated-object address used later in replacement processing is reloaded after the replacement result has been published. It need not equal the address used for this initial selection.

### Existing-resource processing

The function reads the resource field at context offset +0x18.

A zero value causes immediate return. Otherwise, it calls F-1375398b8c0d with the integer argument 1.

If that dependency returns zero, the function returns without explicitly clearing the resource field.

If it returns nonzero, the function:

1. Reads the context resource field again.
2. Calls F-88503fb8bb83 with the nonzero dependency result as its first argument and the freshly read resource as its second argument.
3. Writes zero to the context resource field after that call returns.
4. Returns.

The second resource read is significant. A dependency-induced change between the first and second reads affects the value passed to F-88503fb8bb83. That argument can even be zero despite the initial nonzero test.

The final zero write occurs after F-88503fb8bb83 and overwrites any intervening change to that field. The call itself does not establish release, transfer, or retention of the resource.

### Replacement processing and separate extent reads

The function calls F-1375398b8c0d with argument 1. The following order is established:

1. Immediately after that call returns, the function reads the global 32-bit extent and retains this first value as L.
2. It tests the dependency result.
3. If the result is zero, it returns.
4. If the result is nonzero, it reads the extent global again to supply the second argument to F-4e23d415fadf.
5. It calls F-4e23d415fadf with the nonzero dependency result and that second extent value.

Thus, the first extent read occurs even when F-1375398b8c0d returns zero. The second occurs only on the nonzero-result path.

The two reads must not be treated as one snapshot. Equality requires an explicit stability assumption. The extent passed to F-4e23d415fadf can differ from L, while subsequent region calculations still use L.

The result from F-4e23d415fadf is written to context offset +0x18 before it is tested.

- A zero result therefore explicitly clears the context resource and then causes return.
- A nonzero result is retained as candidate address B.

No visible step processes the old resource value before this overwrite.

For a nonzero B, the function reloads the associated-object address from context offset +0x1C and then calls F-c0fc6d033caa with no explicit arguments.

B, L, and the reloaded associated-object address are retained across this call. Later object-field reads use that retained object address. The call can affect the contents subsequently read; preservation of those contents is not assumed.

### Alternate-region selection

After F-c0fc6d033caa returns, the function reads the byte at object offset +0xCC.

The represented alternate selection requires:

- A nonzero byte at +0xCC.
- B equal to zero.
- L equal to zero.

If selected, it would read an alternate address at +0xC4 and alternate extent at +0xC8.

This selection is unreachable under the supplied scalar control flow: a zero replacement result already caused return, and B is not reassigned before the condition.

The +0xCC byte read nevertheless occurs on the reachable path. No reachable fallback to +0xC4 or +0xC8 is established.

### Capacity and region calculations

The function reads the count at object offset +0xB0 and compares it as an unsigned 32-bit value with 20.

A count of 20 or greater skips region formatting, table writes, accumulator update, and count increment. It does not undo the earlier context-resource assignment or skip the final indirect-call checks.

For a count below 20, the function derives:

- **P:** add 15 to B using the represented 32-bit arithmetic, then clear the lowest three bits.
- **U:** subtract P − B and a further 8 from L using the represented 32-bit arithmetic, then clear the lowest three bits.

P and U are numerically divisible by eight. Without address wraparound, P is between 8 and 15 bytes after B; this is not ordinary rounding of B to the nearest following eight-byte boundary.

The unsigned operations retain the low 32 bits. Insufficient L can produce a large wrapped U rather than a rejected request. No minimum-size or wraparound check precedes the writes.

### Region writes and integrity checks

For an admitted count, the visible direct-write order is:

1. Write a zero four-byte word at P − 4.
2. Write a zero four-byte word at P + U.
3. Read the current count and write P into the corresponding address slot.
4. Read the current count again and write U into the corresponding extent slot.
5. Write U at P.
6. Write zero at P + 4.
7. Write zero at P + 8.

The capacity test and the two slot selections use separate count reads. Describing both slot writes as a single coherent entry requires the count to remain stable across those accesses. The representation does not provide synchronization enforcing that stability.

The function then reads the word at P + U. A nonzero word causes an integrity failure. Only if that word is zero does it evaluate the represented pointer-order condition comparing the end marker with B + L − 4.

An end marker greater than B + L − 4 also causes failure. Failure invokes F-a817d8d58c10, represented as nonreturning.

These checks follow the metadata and table writes. They neither prevalidate the writable region nor provide rollback. Overlapping addresses can cause later writes to change the earlier end marker before it is checked.

After successful checks, the function:

1. Reads the accumulator at object offset +0x04, adds U, and stores the 32-bit result.
2. Calls F-cc6c9b4a0f68 with the retained associated-object address.
3. Reads the count after that call, adds one, and stores it back.

The increment therefore uses the post-call count, not a saved table index. No overflow rejection is shown for the accumulator or count.

### Final indirect-call gates

The final checks are reached after successful registration or after registration was skipped for capacity.

The function reads the byte at object offset +0xB4. Unless it equals exactly 1, it returns. Other nonzero values do not enable the call.

When the byte equals 1:

- It reads the record address at +0xB8.
- A zero address causes return without dereferencing the record.
- Otherwise, it reads the record’s first 32-bit word.
- A zero first word causes return.
- A nonzero first word permits invocation through the global indirect-call target.

The record’s first word is a gate, not the demonstrated call target. No explicit arguments or result use are shown for the indirect invocation. Its complete calling convention is **documented_only**.

## Side effects

Direct state changes include:

- Consuming the pending byte before processing.
- Clearing the context resource after existing-resource processing.
- Publishing a replacement result, including zero.
- Writing region metadata and registration slots.
- Updating the accumulated extent and registration count.

Dependency calls and the optional indirect invocation can have additional, unspecified effects. No dependency is assumed to preserve globals, object fields, or resource ownership.

Safe normal execution requires accessible memory for every reached access. In particular:

- The pending-indicator address must permit its required byte accesses.
- The returned context must permit the reached field accesses.
- The reloaded associated-object address must remain usable across subsequent calls.
- Region storage must permit all reached four-byte accesses, including P − 4, P + U, and P + 8.
- A nonzero callback-record address must permit a four-byte read.
- The global indirect target must be callable when its gates pass.

There are no visible locks, retries, rollback operations, explicit memory-release operations, or conversions of invalid memory access into an error result.

Stored and passed addresses do not by themselves prove ownership transfer, lifetime extension, or eventual release.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-da86f56f2dd4 | Called once in zero mode or twice in nonzero mode after consuming pending work |
| F-65576a99eee9 | Called after the initial call sequence |
| F-fdd8589eca2d | Supplies the context address for extended processing |
| F-1375398b8c0d | Receives argument 1; its result is tested for zero |
| F-88503fb8bb83 | Receives the nonzero preceding result and a freshly read existing-resource value |
| F-4e23d415fadf | Receives the nonzero preceding result and a second, fresh read of the extent global; its result is published in the context |
| F-c0fc6d033caa | Called after reloading the associated-object address and before reading registration-related fields |
| F-a817d8d58c10 | Invoked on either integrity failure; represented as nonreturning |
| F-cc6c9b4a0f68 | Receives the associated-object address after accumulator update and before the count is reread and incremented |

The first retained extent L is not established as F-4e23d415fadf’s second argument. It supplies subsequent region arithmetic and the endpoint check.

All unspecified dependency behavior is **documented_only**, including memory mutation, failure conventions beyond visible zero tests, ownership, and preservation of external state.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests.

1. **No pending work.** With a readable zero pending byte, either mode returns without dependency calls or direct writes.

2. **Short path.** With a nonzero pending byte and normally returning dependencies, zero mode clears the byte, calls F-da86f56f2dd4 once, calls F-65576a99eee9, and returns without obtaining a context.

3. **Extended prefix.** Any nonzero mode clears a nonzero pending byte, calls F-da86f56f2dd4 twice, calls F-65576a99eee9, and obtains a context.

4. **Missing associated object.** If the associated-object field is zero, the status field is not read for selection. If the resource is also zero, no call to F-1375398b8c0d follows.

5. **Fresh existing-resource argument.** Suppose the existing resource is initially nonzero, and F-1375398b8c0d returns nonzero after changing it. F-88503fb8bb83 receives the changed resource value. After that call returns, the field is explicitly zeroed.

6. **Zero prerequisite result on replacement path.** If F-1375398b8c0d returns zero, the first extent read still occurs. There is no second extent read for F-4e23d415fadf, no call to that dependency, and no direct resource-field write on this path.

7. **Different extent observations.** Suppose the first extent read yields 0x100 and the second yields 0x200. F-4e23d415fadf receives 0x200. If it returns B = 0x1000 and registration proceeds, the retained L is still 0x100, giving P = 0x1008 and U = 0xF0. This case is conditional on distinct values actually being observed; their cause is not specified.

8. **Zero replacement result.** If F-4e23d415fadf returns zero, zero is first stored in the context resource. The function then returns without F-c0fc6d033caa, region registration, or final indirect invocation.

9. **Ordinary registration.** With B = 0x1000, L = 0x100, stable count zero, noninterfering dependencies, and valid writable storage, P = 0x1008 and U = 0xF0. The end marker is at 0x10F8; the checked upper endpoint is 0x10FC. The first table entry receives P and U. Successful checks precede accumulator growth by 0xF0, F-cc6c9b4a0f68, and the post-call count increment.

10. **Capacity boundary.** A count of 19 permits registration; 20 skips it. An unsigned value such as 0xFFFFFFFF also skips it. Both outcomes can reach the final gates.

11. **Post-call count mutation.** If registration succeeds and F-cc6c9b4a0f68 leaves the count at 7, the subsequent increment stores 8, irrespective of the earlier slot index.

12. **Exact final gate.** A gate byte of 2 suppresses invocation. A byte of 1 permits it only when both the record address and its first word are nonzero.

13. **Undersized extent.** An insufficient L can yield a wrapped U. Acceptance must not require an orderly integrity failure: potentially invalid memory accesses occur before the checks.

## Uncertainties

- **documented_only:** Dependency semantics, ownership, allocation and release responsibilities, and behavior beyond the observed zero-result branches.
- **documented_only:** Whether the two extent reads observe equal values in a particular environment. Equality requires an external stability guarantee.
- **documented_only:** Concrete global locations, complete layouts, byte order, memory mapping, alignment enforcement, and concurrency guarantees.
- **documented_only:** General pointer-order semantics outside the represented address model, particularly around wraparound or unrelated storage.
- **documented_only:** Effects of invalid memory access, invalid indirect targets, and unexpected dependency control transfer.
- **documented_only:** The intended purpose of the unreachable alternate-region selection.
- **documented_only:** The indirect invocation’s complete calling convention and implicit inputs.
- **Inferred intent:** Region writes resemble allocator metadata initialization, and the indirect invocation resembles notification. Neither interpretation is required by the established behavior.

## Evidence

This contract is based solely on the supplied static representation: its branch conditions, access widths and offsets, argument flow, arithmetic, and ordering of reads, writes, and calls.

In particular, the representation shows two separate extent-global reads on the nonzero replacement-prerequisite path. The first precedes the prerequisite-result test and supplies retained L; the second supplies F-4e23d415fadf’s extent argument. No equality guarantee is established.

No implementation access, runtime execution, or tests were performed. Acceptance examples are conditional static deductions, not observed results.
