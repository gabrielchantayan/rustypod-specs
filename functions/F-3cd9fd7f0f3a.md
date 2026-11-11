<!-- Trusted provenance: F-3cd9fd7f0f3a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3cd9fd7f0f3a

## Purpose

F-3cd9fd7f0f3a coordinates workspace allocation, mode selection, input metadata queries, block reads, processing, finalization, and cleanup. Read requests are capped at 65,536 bytes. Finalization calls specify a 20-byte length; the transformation and result format are not established.

Two descriptive mode names are used:

- **Context mode:** processing receives workspace state and a buffer directly; finalization receives fields captured from an auxiliary context.
- **Handle mode:** initialization obtains a processing handle through an embedded caller-object field; subsequent processing receives that handle.

These names describe call structure, not verified domain semantics. Successful finalization triggers a positioning call whose apparent restoration purpose remains inferred.

## Interface

### Arguments and return

The interface accepts three 32-bit arguments:

| Argument | Established use |
|---|---|
| Caller-object address | Supplies a 32-bit input handle at offset `0x40000` and an embedded field address at offset `0x400F0` |
| Handle-mode result argument | Forwarded unchanged to handle-mode finalization and residual cleanup |
| Context-mode result argument | Forwarded unchanged to context-mode finalization |

No argument validation, object-size check, alignment check, or result-buffer capacity check is visible. The result arguments’ direction, representation, and ownership are unknown.

Normal returns carry a 32-bit status. Zero denotes success along the visible status checks; nonzero dependency statuses generally terminate the corresponding operation. Minimum-size rejection produces `-39`. An explicitly nonreturning cleanup dependency prevents a normal return on paths reaching it.

### Workspace and auxiliary layouts

The function reads a global 32-bit sizing value twice, at distinct points:

1. It caches the first read for alignment-related processing and a counting loop.
2. It rereads the global for the allocation request, adding `0x40` modulo \(2^{32}\).

Equality of these values requires an explicit stability assumption. Neither value is independently validated.

The allocation result is adjusted to a 32-byte boundary, with a conditional high-bit transformation described below. Relative to the resulting working base:

| Offset | Established use |
|---|---|
| `0x00000` | Input-buffer destination |
| `0x10000` | Start of a `0x54`-byte processing-state region |
| `0x1004C` | Stored 32-bit address of a 20-byte stack buffer |
| `0x10050` | 32-bit block-flags field |

Valid execution requires accessible storage through offset `0x10053`, including validity of any transformed address. The allocation request alone does not establish this.

The auxiliary context is accessed at offsets `0x54`, `0x5C`, and `0x60`, each as a 32-bit field.

Two adjacent 32-bit output slots hold queried size information, interpreted as low and high words. A separate adjacent pair holds saved-position information. A mutable 32-bit count slot is passed by address to the read dependency. Those addresses escape to dependencies; subsequent preservation of their contents is not established.

## Behavior

### Workspace preparation

The function caches the first sizing-global read, initializes allocation bookkeeping and the processing-handle slot to zero, retains its arguments, and initially selects a minimum size of `0xA0`.

F-638beded7699 receives the separately reread sizing value plus `0x40`, modulo \(2^{32}\), and the constant `3`.

For a nonzero returned address:

1. The address plus `0x1F`, modulo \(2^{32}\), is rounded down by clearing its lowest five bits.
2. Bit `0x08000000` of the address-plus-`0x1F` intermediate is tested. This is bit 27, not the sign bit.
3. If set, F-20810a394e6a receives the rounded address and the **cached first sizing value**. The working address then has bit `0x80000000` set.
4. A counting loop runs against the cached first sizing value.
5. Allocation bookkeeping is marked active.

The loop contains no explicit memory operation or dependency call in the supplied representation. It establishes no synchronization or minimum-delay guarantee.

If allocation returns zero, the working base remains zero, but execution continues into workspace operations. No safe allocation-failure return is established.

F-c67cb20988a1 receives the state-region address and length `0x54`. The function then stores the address of its 20-byte stack buffer at working-base offset `0x1004C`. Initialization is an inferred purpose of the dependency; its exact writes are unknown.

### Mode selection

F-398e0e9a318a receives a temporary-object address and a fixed string reference. The reference’s label suggests an iPod/iTunes database-related name, but the underlying string bytes are absent. Exact spelling, separators, punctuation, and any leading slash are unresolved.

After that call, the function reads another global 32-bit value and writes it through the returned address. F-bd3e46af741a then receives the temporary object and zero.

A zero query result selects handle mode.

For a nonzero query result:

- F-8157a180d901 receives `100`.
- F-b20180f748aa receives that result, the temporary object, and zero.
- Its returned auxiliary-context address is retained.
- The field at auxiliary-context offset `0x60` is read without a visible null check.

A zero field selects context mode. The function records constant `1` and captures the fields at offsets `0x54` and `0x5C`, in that order, for finalization.

A nonzero field selects handle mode, while retaining the auxiliary-context address.

Handle mode changes the minimum size to `0x6C`. F-eddcf3edc3d7 receives the address of the processing-handle slot and caller-object address plus `0x400F0`. A nonzero status branches directly to final temporary-object and workspace-bookkeeping cleanup, bypassing residual-handle and auxiliary-context cleanup—even if either resource was populated.

### Metadata, minimum size, and positioning

After successful mode setup, the function uses the caller-object field at `0x40000` as the input-handle source.

Calls occur in this order:

1. F-cdf60417d657 receives the current input handle and the saved-position output address.
2. On zero status, F-3cbde1b148f3 receives the current input handle and the size-output address.

Either nonzero status skips processing and enters ordinary cleanup.

The minimum-size check reads the size outputs. It rejects only when the high word is zero and the unsigned low word is below the selected threshold:

- Context mode: below 160 bytes.
- Handle mode: below 108 bytes.

Any nonzero high word bypasses this rejection. Rejection sets status to `-39`.

For an accepted size, F-f7e2f4cd400d receives the current input handle, the size high word sampled for that call, zero, and zero. Its positioning-origin semantics are unknown.

**After this positioning call**, the function loads the size-output slots into its remaining-count pair. Thus the size used for the minimum check, the high word passed to positioning, and the remaining count are not guaranteed to describe the same value. The positioning dependency may affect previously escaped memory. Processing proceeds only if its status is zero.

### Block loop and sampling order

The loop continues while either remaining-count word is nonzero. The remaining pair is distinct from the live size-output slots.

Before each read, the function writes block flags:

- `0` if the remaining high word is nonzero or the unsigned low word exceeds `0x10000`.
- `2` otherwise.

Using the same condition, it writes the mutable count slot with either `0x10000` or the remaining low word. F-74745fd09174 receives the current input handle, the count-slot address, and the working base.

A nonzero read status immediately enters cleanup. That iteration performs no first-block range operations, block processing, successful-processing finalization, or restoration attempt.

After a successful read, the first-block condition compares the remaining pair against the **live size-output slots**, rereading those slots after the read. The high-word comparison precedes the low-word comparison, which is needed only if the high words match.

This is not a comparison against an immutable original-size snapshot. If dependencies preserve the size-output slots, it acts as an original-size comparison; otherwise first-block treatment can be suppressed, repeated, or reached later.

When the comparison matches:

- Handle mode invokes F-c67cb20988a1 on these ranges, in order:
  - Working-base offset `0x18`, length `8`.
  - Offset `0x32`, length `0x14`.
  - Offset `0x58`, length `0x14`.
- Both modes read the current flags and OR in bit `1`.

The read precedes these range operations. Their exact memory effects are unknown. They are not limited by the count reported by a short read.

Processing then follows the selected mode:

- **Context mode:** F-d15c1be26a89 receives the state address, working base, and the count slot’s value sampled at this invocation. Its return value is ignored.
- **Handle mode:** F-93e2c171219a receives the processing handle. A nonzero return enters cleanup. No buffer or count argument is explicitly supplied.

After processing returns successfully, the function rereads the count slot for remaining-count subtraction and borrow determination. This is a later sampling point than the context-processing argument. The slot’s address previously escaped to the read dependency, so preservation across later calls cannot be assumed.

For a stable post-processing count, subtraction operates as a two-word unsigned quantity: subtract from the low word modulo \(2^{32}\), and decrement the high word if the previous low word was smaller than the count. No intervening explicit dependency call appears between borrow calculation and low-word subtraction.

Consequences, with preservation assumptions stated:

- A zero **post-processing subtraction count** makes no progress. Repeated zero values can cause indefinite looping.
- A read reporting zero does not alone prove nonprogress if a later dependency changes the count slot.
- A post-processing count exceeding the remaining quantity can underflow the pair; no check rejects it.
- Context processing can receive one count while subtraction uses a different later value.
- The function chooses final-block bit `2` before the read and does not recompute it from returned or post-processing counts.
- Unknown dependencies may modify the flags themselves; the later OR preserves the then-current bits rather than reconstructing them.

### Finalization and positioning afterward

When the remaining pair is zero:

- Context mode calls F-b902a8e87b05 with constant `1`, the captured `0x54` and `0x5C` fields, the 20-byte stack-buffer address, length `0x14`, the third argument, and constant `0x2E`.
- Handle mode calls F-a70effbdb56d with the processing handle, the second argument, and length `0x14`. It then clears the handle slot regardless of finalization status.

Both finalizers are represented as returning 64 bits. The low 32 bits become status.

Only a zero low word triggers F-f7e2f4cd400d with:

1. The current input handle reread from the caller object.
2. The finalizer result’s high 32 bits.
3. The two saved-position output words, sampled for this call.

The positioning status becomes the pending return status.

Restoring position is inferred intent, not established positioning semantics. The saved-position slots also escaped to a dependency, so their continued equality to their initially returned values requires preservation.

No restoration attempt is visible after metadata failure, minimum rejection, initial-positioning failure, read failure, processing failure, or nonzero finalization status.

## Side effects

Direct writes include allocation bookkeeping, the temporary object’s returned target, the stack-buffer pointer in workspace state, block flags, request counts, and processing-handle clearing. Dependencies receive addresses allowing additional writes whose extent is unknown.

Input reads and positioning calls may change input state. The input-handle field is reread at successive call sites; a dependency’s modification of that field may redirect later operations.

Ordinary cleanup is ordered as follows:

1. If the processing-handle slot is nonzero, call F-a70effbdb56d with that handle and the second argument. No explicit length argument appears for this invocation. Its result does not replace the pending status.
2. If an auxiliary context is retained, call F-cc6f52873d53 with its address plus `0x54`; call F-a19c2808de87 with the returned address minus `0x54`; then invoke explicitly nonreturning F-feee9156a257.
3. If final cleanup is reached, invoke the unmapped temporary-object cleanup dependency, then F-ffa7163332aa with the allocation-bookkeeping address.
4. Return the pending status.

The handle-initializer failure path enters step 3 directly. A path reaching step 2 does not proceed to step 3 under the supplied nonreturning annotation.

Resource release and ownership transfer are not established. Clearing the processing-handle slot prevents another cleanup call here; it does not prove release. The workspace’s stack-buffer pointer is valid only during this invocation unless a dependency independently arranges another lifetime; no such arrangement is visible.

## Dependencies

These IDs identify call sites, not verified implementations.

| Dependency | Established role |
|---|---|
| F-638beded7699 | Workspace allocation request |
| F-20810a394e6a | Alignment-related operation using cached sizing value |
| F-c67cb20988a1 | State-region and selected buffer-range operations |
| F-398e0e9a318a | Temporary-object construction using a fixed string reference |
| F-bd3e46af741a | Temporary-object query |
| F-8157a180d901 | Produce an argument from `100` |
| F-b20180f748aa | Return an auxiliary-context address |
| F-eddcf3edc3d7 | Initialize the processing-handle slot |
| F-cdf60417d657 | Populate saved-position slots |
| F-3cbde1b148f3 | Populate size slots |
| F-f7e2f4cd400d | Initial and post-finalization positioning calls |
| F-74745fd09174 | Input read with mutable count |
| F-d15c1be26a89 | Context-mode block processing |
| F-93e2c171219a | Handle-mode block processing |
| F-b902a8e87b05 | Context-mode finalization |
| F-a70effbdb56d | Handle-mode finalization or residual cleanup |
| F-cc6f52873d53 | First auxiliary-context cleanup operation |
| F-a19c2808de87 | Second auxiliary-context cleanup operation |
| F-feee9156a257 | Nonreturning operation |
| F-ffa7163332aa | Allocation-bookkeeping cleanup |

The temporary-object cleanup dependency has no supplied opaque ID. Its behavior is documented_only.

## Acceptance examples

These are conditional static scenarios, not executed tests. Unless a scenario explicitly changes escaped slots, assume dependencies preserve them and the relevant flags, memory is accessible, and operations succeed.

1. **Distinct sizing reads.** If the cached first sizing read is `0x20000` and the allocation-time read is `0x30000`, allocation receives `0x30040`. Any alignment-related call and the counting loop use `0x20000`, not `0x30000`.

2. **Minimum boundaries.** Handle-mode size 107 produces `-39`; 108 passes. Context-mode size 159 produces `-39`; 160 passes. With a retained auxiliary context, ordinary cleanup reaches the nonreturning sequence, so these statuses do not imply normal returns.

3. **Single complete block.** A stable handle-mode size of 108 produces a request of 108 with flags `2`. A complete read triggers the three range operations, flags become `3`, and processing precedes finalization.

4. **Block boundary.** Stable size 65,536 produces one request with flags changing from `2` to `3`. Size 65,537 produces requests of 65,536 and 1, with processing-time flags `1` and `2`.

5. **Changed size before snapshot.** If handle-mode size queries report 108 but initial positioning changes the live size slots to 80, the minimum check has already passed. The remaining snapshot is 80, and the first request is 80. No second minimum check occurs.

6. **Changed size after read.** If the remaining snapshot is 108 and the read changes the live size low word to 109, the first-block comparison fails. The range operations and first-block OR are skipped.

7. **Distinct processing and subtraction counts.** With 200 remaining, context processing receives 100 if that is the count slot at invocation. If that dependency changes the slot to 40, subtraction leaves 160, not 100.

8. **Zero-count qualification.** If a successful read leaves zero and all later calls preserve it, the remaining pair does not decrease. If processing changes it to a positive value, subtraction can progress despite the read’s zero count.

9. **Failure routing.** A read error skips block processing and finalization. A nonzero finalization low word suppresses restoration; handle mode has already cleared its handle slot. Handle-initializer failure bypasses residual-handle and retained-context cleanup.

## Uncertainties

The following remain **documented_only**:

- Exact fixed-string bytes and pathname.
- Sizing-global stability, valid sizing bounds, allocation guarantees, address transformation, and safe allocation-failure behavior.
- Exact workspace initialization and first-block range effects.
- Dependency access to escaped size, count, saved-position, and handle slots.
- Result-argument formats, buffer capacities, resource ownership, and release semantics.
- Finalizer ABI details, especially the high result word and shorter cleanup invocation.
- Positioning-origin semantics and actual restoration behavior.
- Guarantees of read progress, count bounds, and workspace capacity.
- External effects of the nonreturning and unmapped cleanup dependencies.

No allocator, file-service, destructor, or transformation convention resolves these gaps by itself.

## Evidence

Established static evidence consists of the supplied function representation and opaque dependency mapping: argument widths, offsets, constants, separate global reads, escaped addresses, sampling order, branches, arithmetic, call order, cleanup routing, and the nonreturning annotation.

Mode names, initialization or normalization purpose, and restoration intent are interpretations. The string label is indicative, not evidence of literal bytes. Dependency implementations and runtime observations are unavailable. No runtime tests have been performed.
