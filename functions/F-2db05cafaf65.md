<!-- Trusted provenance: F-2db05cafaf65; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2db05cafaf65

## Purpose

F-2db05cafaf65 updates a stateful object through an accumulation phase and a transition phase. A dynamically dispatched dependency can select an alternate path that bypasses both phases.

In accumulation, the function adds an increment to a selected record and evaluates thresholds. In transition, it advances an offset and, when a boundary is reached, updates the selected record index and offset using an arithmetic dependency.

“Progress,” “phase,” “notification,” and “span” are descriptive labels, not established application-level meanings. A cycling or scrolling purpose is plausible but remains inferred.

The function also exposes a temporary four-word argument block to a dependency. Its return value contains the first two words of that block **as read during return assembly**, not necessarily their incoming values or their values immediately after preprocessing.

## Interface

### Inputs and temporary argument block

The represented interface accepts four 32-bit values:

1. An object address.
2. A second input word.
3. A third input word.
4. A fourth input word.

The last three inputs are represented as signed 32-bit integers. The object address also participates in arithmetic through its copy in the argument block.

Before calling dependencies, the function places these inputs, in order, into a contiguous four-word temporary block. F-412e4635fdb0 receives the original object address and the block’s address.

Subsequent direct object accesses use the original object address. Changing the block’s first word does not redirect those accesses.

The block remains relevant throughout execution:

- Its first and third words supply a horizontal difference.
- Its second and fourth words supply a vertical difference.
- Its first and second words supply the return value.

The block’s address escapes to F-412e4635fdb0. The supplied representation does not establish that this dependency discards the address, or that later dependencies cannot access it indirectly. Consequently, block contents must be interpreted at their respective read points.

### Return value

Both normal exits return a 64-bit bit pattern containing:

| Result portion | Source |
|---|---|
| Low 32 bits | First temporary-block word at return assembly |
| High 32 bits | Second temporary-block word at return assembly |

The last dependency call on the selected path occurs before these return reads. A dependency that can reach the escaped block may therefore affect the return value even if preprocessing initially left the block unchanged.

No separate success indicator is established. The bit composition does not establish a higher-level return type or portable calling convention.

### Directly accessed object layout

All offsets are byte offsets from the original object address.

| Offset | Width | Use |
|---|---:|---|
| `0x000` | 32 bits | Dynamic dispatch-table address |
| `0x0F8` | 32 bits | Additional argument to the dynamic dependency |
| `0x234` | 32 bits | Progress increment |
| `0x244` | 32 bits | Mode/count |
| `0x464` | 32 bits | Transition offset |
| `0x494` | 8 bits | Notification flag |
| `0x495` | 8 bits | Phase flag |
| `0x498` | 32 bits | Selected record index |

Records have a represented stride of `0x6C` bytes. For selected index \(k\):

| Relative address | Width | Use |
|---|---:|---|
| `0x2A8 + 0x6C × k` | 32 bits | Secondary threshold |
| `0x2AC + 0x6C × k` | 32 bits | Progress threshold |
| `0x2B0 + 0x6C × k` | 32 bits | Progress accumulator |

These expressions establish accessed addresses, not a complete allocation layout or a valid index range. An unchecked index can address storage outside the intended records or overlap other fields.

Writes to `0x494` and `0x495` affect individual bytes, not an entire surrounding word.

### Preconditions and errors

Normal completion requires readable and writable storage for the addresses actually used, a callable dispatch entry, and dependencies that return normally.

No explicit validation or recoverable error path is visible for null addresses, invalid dispatch targets, invalid indices, insufficient storage, overflow, or invalid arithmetic-helper inputs. The mode/count field is not an index-bounds check.

## Behavior

### Arithmetic and read-time values

The representation uses 32-bit arithmetic. For bit-level descriptions below, additions, subtractions, and products retain their low 32 bits; ordering comparisons are signed. Portable language-level behavior for signed overflow is not established.

The horizontal span is the third block word minus the first block word, with the latter treated as a 32-bit integer. The vertical span is the fourth block word minus the second.

These are expressions evaluated when needed, not values guaranteed to remain constant across dependency calls. In particular, transition completion can evaluate the horizontal span both before and after the first arithmetic-helper call.

Some accumulation decisions use a subtraction’s sign together with its signed-overflow indication. Their effect is a signed comparison of the operands, not merely a sign test on a wrapped difference.

The representation establishes ordering between calls and separate updates. It does not establish an order between every operand read within a single expression.

### Initial calls and dynamic gate

Execution begins with:

1. F-412e4635fdb0, receiving the original object and the temporary argument block.
2. F-7a1c296974df, receiving the original object.
3. An unidentified dynamic dependency selected through the object’s dispatch table at table byte offset `0x130`.

The dynamic dependency receives the original object and the word at object offset `0x0F8`. The dispatch information and argument are obtained after the two preceding calls; their entry-time values are not guaranteed to survive those calls.

If the dynamic return equals **exactly 1**, the function calls F-42382d1a1b3b with the object and integer 1, then assembles its return value. This path performs no direct phase update and does not call F-92bea7c30bd6.

Every other dynamic return proceeds to phase selection. Effects of all dependencies remain possible on either path.

### Accumulation phase

A zero byte at `0x495`, read after the dynamic gate, selects accumulation.

The function reads the selected index, adds the progress increment to that record’s accumulator, and stores the updated accumulator. It then reads the record’s progress threshold and tests the mode/count.

The increment is stored before any threshold decision, including decisions that produce no further update.

#### Mode/count equals 1

The updated accumulator is compared with the progress threshold using signed ordering.

If it is below the threshold, this branch makes no further direct write.

If it meets or exceeds the threshold, the following ordered effects occur:

1. Reset the selected accumulator to zero.
2. Write 1 to notification byte `0x494`.
3. Call F-a66a9be680df without explicit arguments.
4. Pass its result and `0x0000000C` to F-c5a232e4455b.
5. Pass that result and `0x31800000`, 0, and 0 to F-afb630a7107c.
6. Pass the object, that result, and 0 to F-8e50f265e4fa.

The constant `0x31800000` is established as a bit pattern; its semantic type is unknown.

This branch does not directly change phase byte `0x495`. Dependencies may change it or any other reachable state.

#### Mode/count differs from 1

The branch triggers transition entry when either signed comparison succeeds:

- Updated accumulator is at least the progress threshold.
- Vertical span is at least the secondary threshold.

If the progress comparison fails, the secondary comparison determines whether entry is triggered. If progress alone establishes the trigger, the secondary threshold is still examined to decide whether to reset the accumulator.

When triggered:

- The accumulator is reset to zero if the vertical span meets or exceeds the secondary threshold.
- The phase byte is then written as 1.

Thus a progress-only trigger preserves the updated accumulator. A secondary-threshold trigger resets it, regardless of whether progress also met its threshold.

When neither comparison succeeds, the updated accumulator remains stored and neither flag is directly changed.

### Transition phase

Any nonzero phase byte selects transition.

The function adds 2 to the transition offset and stores that result. It then calculates a boundary from the current selected index plus 1, multiplied by the horizontal span.

Completion occurs when the updated offset equals the boundary or exceeds it in signed ordering. If completion does not occur, the offset increment is this phase’s only direct update.

On completion, the observable sequence is:

1. Reset the currently selected record’s accumulator to zero.
2. Read the selected index again, increment it, and store it.
3. Read and retain the mode/count.
4. If that retained count is less than 1, replace the selected index with zero.
5. Otherwise, call F-870c4fcf7295 with the incremented index and retained count, then replace the selected index with the helper’s secondary result.
6. Read the selected index from the object and calculate its product with the horizontal span as evaluated at this stage. Store the product as the transition offset.
7. If the retained count is less than 1, overwrite the offset with zero.
8. Otherwise, evaluate whether the horizontal span is zero. If so, overwrite the offset with zero.
9. If neither bypass applies, call F-870c4fcf7295 with the stored product value and the low 32 bits of retained count multiplied by the horizontal span. Replace the offset with this call’s secondary result.
10. Clear phase byte `0x495` to zero.
11. Set notification byte `0x494` to 1.

The retained count, rather than a fresh object-field read, governs both helper decisions and the later count-times-span argument.

The first helper’s secondary result is written to the index field after the call. The later product uses an index read from the object, not merely an assumed mathematical normalization.

The first helper may affect the escaped argument block. Therefore, a zero span at the boundary calculation does **not** alone establish that the second helper is skipped. The zero-span decision uses the block contents when that decision is reached.

Intermediate index and offset writes are observable even when subsequently overwritten. The helper resembles a division or normalization operation, but its secondary result is not established unconditionally as a remainder.

### Common completion

Every normal phase-processing path calls F-92bea7c30bd6 after its direct updates and any branch-specific dependencies.

Return assembly follows this call. The object fields immediately after direct updates are therefore not guaranteed to be their values at function return. F-92bea7c30bd6 may alter the object or the escaped block.

There is no visible rollback.

## Side effects

Direct writes target the addressed record accumulators, selected index, transition offset, and two flag bytes described above. With invalid or overlapping record addresses, those writes need not remain within the intended logical fields.

Dependency effects are unrestricted by the supplied evidence. No dependency is established as pure, object-preserving, block-preserving, or global-state-preserving.

Important ordering guarantees include:

- Accumulator increment precedes accumulation decisions.
- Single-mode reset and notification precede the four-call chain.
- Transition completion resets the old record before incrementing the selected index.
- Arithmetic-helper calls precede the final direct phase and notification writes.
- F-92bea7c30bd6 follows phase processing.
- Return-word reads follow the final dependency on either exit path.

No allocation, release, reference-count update, or ownership transfer is directly established. Values passed through the call chain may be scalars, handles, or addresses; their ownership is unknown.

The argument block is temporary storage exposed by address. Retention during this invocation is not excluded. Permission to use that address after the invocation is not established.

No locking, atomicity, thread-safety, or reentrancy guarantee is visible.

## Dependencies

| Dependency | Established role |
|---|---|
| F-412e4635fdb0 | Receives the original object and mutable four-word block |
| F-7a1c296974df | Receives the object before dynamic dispatch |
| Unidentified dynamic dependency, table offset `0x130` | Receives the object and its `0x0F8` word; return exactly 1 selects the alternate path |
| F-42382d1a1b3b | Receives the object and 1 on the alternate path |
| F-a66a9be680df | Starts the single-mode threshold call chain |
| F-c5a232e4455b | Receives the preceding result and `0x0000000C` |
| F-afb630a7107c | Receives the preceding result, `0x31800000`, 0, and 0 |
| F-8e50f265e4fa | Receives the object, preceding result, and 0 |
| F-870c4fcf7295 | Produces secondary results used to overwrite index and offset |
| F-92bea7c30bd6 | Runs after every normal phase-processing path |

No opaque identity is supplied for the dynamic target. Its dispatch location identifies its role without establishing its implementation.

## Acceptance examples

These are conditional specifications, not executed tests.

Unless an example explicitly permits a dependency effect, assume all dependencies return normally and preserve the object and all four argument-block words. Also assume valid, nonoverlapping storage for the fields and records used, no concurrent mutation, and arithmetic values small enough to avoid overflow. Phase examples assume the dynamic return differs from 1.

These preservation conditions apply through F-92bea7c30bd6 or F-42382d1a1b3b, as applicable. Without them, the specified direct writes and call order still apply, but corresponding final-state claims need not hold.

1. **Exact alternate-path selection.**  
   A dynamic return of 1 causes F-42382d1a1b3b to receive the object and 1. No direct phase processing or F-92bea7c30bd6 call follows. A dynamic return of 2 proceeds to phase processing instead.

2. **Return words are read at the end.**  
   Permit preprocessing to set the first two block words to `0x11223344` and `0xAABBCCDD`. If all subsequent calls preserve those words, the return is `0xAABBCCDD11223344`. Alternatively, permit the final dependency to change the second word to `0x55667788` through the escaped block address while preserving the first: the return is then `0x5566778811223344`.

3. **Single-mode equality triggers.**  
   With phase zero, count 1, accumulator 8, increment 2, and progress threshold 10, the accumulator is first written as 10 and then zero. Notification is written as 1. The four-call chain precedes F-92bea7c30bd6.

4. **Progress-only triggering preserves accumulation.**  
   With phase zero, count 2, updated accumulator 10, progress threshold 10, vertical span 4, and secondary threshold 5, phase becomes 1 and the accumulator remains 10.

5. **Secondary threshold triggers independently.**  
   With phase zero, count 2, updated accumulator 3, progress threshold 10, vertical span 5, and secondary threshold 5, the accumulator is reset and phase becomes 1.

6. **An incomplete transition still advances its offset.**  
   With phase nonzero, index 0, horizontal span 10, and offset 6, the offset becomes 8. The boundary is 10, so no completion reset or index update occurs.

7. **Completion consumes helper results.**  
   With phase nonzero, index 0, span 10, offset 8, and count 3, the offset first becomes 10 and completion occurs. The old record is reset. The first helper receives 1 and 3. If its secondary result is 1, index is written as 1 and the offset product as 10. The second helper receives 10 and 30. Its secondary result is written to the offset, then phase is cleared and notification set. Under the stated preservation assumptions, these last values survive common completion.

8. **Nonpositive counts bypass both helper calls.**  
   On completion with retained count 0 or negative, the function directly writes index and offset to zero without either arithmetic-helper call, then clears phase and sets notification. These remain the return-time values only under the stated dependency-preservation assumptions.

9. **Preserved zero span skips only the second helper.**  
   With phase nonzero, index 0, offset −2, count 3, and horizontal span zero, the incremented offset and boundary are both zero, so completion occurs. The first helper still runs. If it preserves the block, the later span remains zero: offset is overwritten with zero and the second helper is skipped.

10. **An escaped-block change can defeat that zero-span bypass.**  
    Use example 9’s entry state, but permit the first helper to change the block so the horizontal span becomes 10, while returning secondary result 1. The subsequent product is 10, the zero-span guard fails, and the second helper receives 10 and 30. Zero span at the earlier boundary check does not require a bypass later.

## Uncertainties

The following remain **documented_only**:

- Application meanings, units, valid ranges, and complete object layout.
- Dependency internals, ownership contracts, and external effects.
- Whether any dependency retains or later accesses the argument-block address.
- The exact arithmetic contract of F-870c4fcf7295, including negative operands and exceptional inputs.
- Machine-level fault behavior and signed-overflow semantics beyond the represented operations.
- Dependency failure, non-return, exception, blocking, and reentrancy behavior.
- Concurrent observation or mutation guarantees.

A positive retained count and a nonzero span do not ensure a nonzero second helper argument: their 32-bit product can wrap to zero. No explicit guard for that product is visible.

Apparent cycling intent does not establish bounds safety, successful normalization, monotonic progress, or eventual completion.

## Evidence

Established static evidence consists of the supplied representation’s field widths and offsets, argument-block exposure, calls, branch conditions, arithmetic expressions, intermediate stores, and return composition. Dependency identifiers follow the supplied opaque mapping.

The distinction between direct writes and return-time state follows from calls occurring after those writes. The possibility of later block mutation follows from its address escaping without a supplied preservation contract; it is a permitted dependency effect, not an observed mutation.

Application-level labels and cycling intent are inferences. Dependency behavior and broader invariants remain unresolved.

No runtime tests have been performed. Acceptance examples describe conditional expected behavior, not observed results.
