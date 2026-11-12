<!-- Trusted provenance: F-eace0518d18e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eace0518d18e

## Purpose

F-eace0518d18e processes an ordered collection referenced by a mutable context. Its visible operations include collection initialization, two extension mechanisms, tail examination and recovery, event publication, optional checks, and final processing.

The contract below describes explicit branches, accesses, calls, and return construction in the supplied static body. Collection terminology describes observed usage; application-level meanings of entries, event codes, and accounting fields remain unknown.

Execution is not transactional. Failure may leave collection changes, context writes, or acquired resources in place. Three direct-return paths bypass shared cleanup.

## Interface

### Arguments and result

The visible interface accepts four arguments:

| Argument | Width | Observable role |
|---|---:|---|
| Context address | 32 bits | Addresses mutable fields described below. |
| Auxiliary word | 32 bits | Supplies the upper return word until explicitly cleared. |
| Unused word | 32 bits | No explicit use in the supplied body. |
| Initial candidate | 32 bits | Initializes a retained candidate slot whose address can be passed to the producer callback. |

The ordinary result is a 64-bit bit pattern composed of:

- Upper 32 bits: the retained auxiliary word.
- Lower 32 bits: the retained processing result.

The processing result starts at zero. Producer results, selected predicate results, event callback results, optional processing, preparation, and final processing can replace it at the points specified below. Most predicates do not replace it.

Two returns use fixed values instead:

- Missing root: `0x00000000FFFFFFFF`.
- Failed checked insertion during the second extension phase: `0x0000000000000000`.

A negative second-phase producer result returns immediately through ordinary word composition, preserving its 32-bit representation in the lower word. The complete result has no established Boolean or signed interpretation.

### Context layout

Offsets are byte offsets from the context address. Listed accesses use 32-bit fields.

| Offset | Observable role |
|---|---|
| `0x08` | Required nonzero root value. |
| `0x0C` | Nonzero enables auxiliary-state acquisition; later reread to permit first-phase production. |
| `0x10` | Signed-positive value enables intermediate processing. |
| `0x14` | Signed-positive value enables a later entry check. |
| `0x24` | Optional final callback address. |
| `0x28` | Event callback address. |
| `0x2C` | Producer callback address. |
| `0x30` | Predicate callback address. |
| `0x34` | Preparation callback address. |
| `0x48` | Signed traversal limit, captured once before extension. |
| `0x50` | Mutable accounting or boundary value. |
| `0x54` | Collection handle. |
| `0x58` | Published position. |
| `0x5C` | Published event code. |
| `0x60` | Published entry value. |

Callback interfaces visible at call sites are:

- **Event:** receives zero and the context; returns a 32-bit result.
- **Producer:** receives the address of the retained 32-bit candidate slot, the context, and the current entry; returns a signed 32-bit status. Receiving the slot address does not establish that the callback writes it.
- **Predicate:** receives the context and two entry values; its result is tested for zero.
- **Preparation:** receives the context and returns a 32-bit result.
- **Optional final callback:** receives no explicit arguments and supplies the final processing result.

The earlier event callback address is captured after the root check and before initialization. Earlier event notifications use this captured address. The optional later check reloads the event callback before its count and lookup operations.

### Preconditions and arithmetic

Reached context fields, callback addresses, and dependency arguments must be valid for their uses. No explicit pointer validation is visible.

Counts, indices, positions, entry values, and accounting arithmetic use visible 32-bit quantities. Traversal comparisons and positivity tests are signed. Arithmetic includes count minus one, position minus two, increments, decrements, and entry-address addition by `0x10`; no overflow guards are visible.

The body performs last-entry lookups without an explicit nonempty check. It does not establish bounds, overflow semantics at extremes, or dependency behavior for invalid indices.

## Behavior

### Root validation and initialization

The first context access reads `0x08`.

If it is zero, F-4fc89ec41a6d receives `(11, 127, 105)`, followed by the fixed missing-root return. Event callback capture and ordinary cleanup do not occur.

Otherwise, the event callback is captured and `0x54` is inspected. A nonzero handle skips initialization and initially preserves the supplied auxiliary word.

For a zero handle:

1. F-c141a8d8555e is called without explicit arguments, and its result is immediately stored at `0x54`.
2. If nonzero, that result and the current root value are passed to F-754616bec4de.
3. If this call succeeds with a nonzero result, the retained auxiliary word is cleared. F-bbc5977ee4f2 receives the then-current root value plus `0x10`, followed by `1` and `3`. Afterwards, `0x50` is written as one.

A zero creation result or zero initialization result takes the diagnostic failure route: clear the retained auxiliary word, report `(11, 127, 65)` through F-4fc89ec41a6d, then perform ordinary failure cleanup. The handle store is not rolled back.

### Auxiliary state and first extension phase

If `0x0C` is nonzero, F-80b3be498d4e obtains auxiliary state. A zero result takes the diagnostic failure route. Otherwise, or if acquisition was skipped, execution obtains:

- A collection count through F-10b760faca36.
- An entry through F-6bbddce1b930 at that count minus one.
- The traversal limit from `0x48`.

The initial count also becomes the traversal position. The lookup rereads the context’s collection handle after the count call.

While position is signed-less-than-or-equal-to the captured limit:

- F-cf23980ec4fa receives the current entry.
- The predicate is loaded from `0x30` and invoked once, with the current entry in both entry-argument positions.
- A nonzero predicate result ends this phase.
- Otherwise, `0x0C` is reread; zero ends this phase.
- F-5c131907de4a receives the context, retained auxiliary state, and current entry. Its return value replaces the candidate slot. Zero ends this phase.
- For a nonzero candidate, F-754616bec4de receives the current collection handle as its **sole visible argument**. Zero takes the diagnostic failure route.
- On nonzero return, the auxiliary return word is cleared. F-bbc5977ee4f2 receives candidate plus `0x10`, `1`, and `3`; then F-51a9703516d3 receives auxiliary state and candidate.
- `0x50` is read and incremented. The candidate becomes current, and position increments.

The one-visible-argument call must not be treated as a proven two-argument insertion.

### Tail examination and recovery

After the first phase, the function refreshes the collection count and retrieves the entry at that refreshed count minus one. It then calls F-6261bc5c6581 without explicit arguments.

After that call, it loads and invokes the predicate **once**, passing the refreshed last-entry value as both entry arguments.

A zero predicate result enters the second extension phase directly.

For a nonzero result, another collection count is obtained:

- **Count other than one:** F-2eeed9557344 receives the current collection handle, and its result is retained as a detached entry. The function decrements `0x50`, looks up the entry at the existing traversal position minus two, decrements that position, and enters the second phase. This lookup uses the traversal position, not the newly obtained count.
- **Count exactly one:** the producer receives the candidate-slot address, context, and current entry. Its signed status replaces the processing result. Whether it modifies the candidate slot is unknown.

For the single-entry case, a positive producer status causes F-ed04800b7b7b to receive the current entry and the slot’s resulting value. A positive status does not establish a newly written candidate.

If that comparison returns zero:

1. F-fa5325efe259 receives the old current entry.
2. The slot’s then-current value becomes the current entry.
3. F-c2674fa3e73d receives the current collection handle, the earlier refreshed count minus one, and the slot’s value at the replacement call.
4. `0x50` is written as zero.
5. Execution enters the second phase.

Otherwise, event `18` is published in this order: event code at `0x5C`, current entry at `0x60`, earlier refreshed count minus one at `0x58`. If the producer status is exactly one, F-fa5325efe259 then receives the candidate slot’s current value. Other statuses do not cause that explicit candidate cleanup.

The captured event callback is invoked next. Zero takes ordinary failure cleanup; nonzero enters the second phase. A negative producer status in this recovery branch therefore does **not** itself cause immediate return.

### Second extension phase

This phase retains the traversal position and captured limit. While position is signed-less-than-or-equal-to the limit:

1. F-cf23980ec4fa receives the current entry.
2. The current predicate is invoked once with that entry in both entry positions. Nonzero ends the loop.
3. The producer receives the candidate-slot address, context, and current entry. Its signed status replaces the processing result.
4. The slot’s resulting value is retained separately before insertion.

The producer may leave the slot unchanged. Subsequent operations use its resulting value even then; positive status alone neither proves a write nor guarantees a nonzero candidate.

Status handling is:

- **Negative:** immediately return the composed result, bypassing ordinary and shared cleanup.
- **Zero:** end the loop.
- **Positive:** call F-754616bec4de with the current collection handle and candidate-slot value.

On successful insertion, the separately retained candidate value becomes current, and position increments. No explicit `0x50` increment occurs.

On failed insertion, F-fa5325efe259 receives the candidate slot’s value **at cleanup time**, followed by diagnostic `(11, 127, 65)`. The function returns fixed numeric zero without ordinary or shared cleanup. The cleanup argument is not necessarily identical to the separately retained value if an intervening dependency can change the slot.

### Post-extension event selection

The function calls F-cf23980ec4fa on the current entry, then invokes the predicate once with that entry in both entry positions.

If nonzero, this event-selection block is skipped. That predicate result does not replace the processing result.

If zero:

- With no nonzero detached entry, select event `2` when signed `0x50` is less than position; otherwise select event `20`.
- With a nonzero detached entry, invoke the predicate once with current and detached entries. This comparison **does** replace the processing result. A zero result selects event `2` or `20` using the same boundary rule.

For events `2` and `20`, write the event code to `0x5C`, then current entry to `0x60`.

If the current-versus-detached comparison is nonzero, instead:

1. Pass the current collection handle and detached entry to F-754616bec4de; ignore its result.
2. Write event `19` to `0x5C`, then detached entry to `0x60`.
3. Increment position.
4. Clear the retained detached-entry reference.
5. Write the incremented position to `0x50`.

For all three events, write position minus one to `0x58`, then invoke the captured event callback. Its result replaces the processing result. Zero takes ordinary failure cleanup.

### Optional checks and final processing

If signed `0x10` is positive, F-27727134fe01 receives the context and replaces the processing result. Afterwards, zero processing result takes ordinary failure cleanup, whether or not this optional call occurred.

With nonzero processing result and signed-positive `0x14`:

1. Reload the event callback.
2. Obtain the current count and last entry, rereading the collection handle for lookup.
3. Call F-0a74fc9b8503 with that entry, the then-current `0x14` value, and zero.
4. Set the processing result to one.

Check result one publishes nothing. Otherwise, write count minus one to `0x58`, the retrieved entry to `0x60`, and event `28` for check result two or event `27` for every other result to `0x5C`. Invoke the reloaded event callback; its result becomes the processing result. Zero takes ordinary failure cleanup.

If processing remains nonzero, call F-16d2f95f7f7b with zero and the current collection handle. Then invoke the preparation callback loaded from `0x34`.

Zero preparation result takes ordinary failure cleanup, causing another F-16d2f95f7f7b call. Nonzero preparation result selects final processing:

- A zero callback field at `0x24` selects F-8d0ea5a63663 with the context.
- Otherwise, the callback at `0x24` is invoked without explicit arguments.

The selected result becomes the processing result, followed by shared cleanup. A zero final result does not itself cause ordinary failure cleanup.

## Side effects

Ordinary failure cleanup first sets the processing result to zero, then invokes F-16d2f95f7f7b with zero and the current collection handle.

Shared cleanup subsequently:

1. Calls F-9aef742d6dcd if retained auxiliary state is nonzero.
2. Calls F-fa5325efe259 if the retained detached entry is nonzero.
3. Returns the composed result.

Missing-root return, negative second-phase production, and failed second-phase insertion bypass shared cleanup.

Explicit writes affect collection handle, accounting value, event fields, and the candidate slot through first-phase production. Collection mutation also occurs through dependencies. There is no general rollback or unconditional candidate cleanup.

Event `19` clears the local detached reference even when the ignored insertion result indicates failure. This prevents its later explicit shared cleanup but does not prove successful ownership transfer.

Dependencies may affect context, globals, or previously exposed storage. Repeated reads are not an immutable snapshot. In particular, the callback may retain or modify the candidate slot’s address; preservation across later calls is not established. Cleanup-shaped calls do not independently prove deallocation, reference-count behavior, or ownership.

## Dependencies

These are observed call roles, not complete dependency contracts.

| Opaque ID | Visible use |
|---|---|
| F-4fc89ec41a6d | Numeric diagnostic triples. |
| F-c141a8d8555e | Collection-handle acquisition. |
| F-754616bec4de | Checked initialization, one-visible-argument operation, checked insertion, unchecked detached-entry insertion. |
| F-bbc5977ee4f2 | Entry/root address plus `0x10`, followed by `1`, `3`. |
| F-80b3be498d4e | Auxiliary-state acquisition. |
| F-10b760faca36 | Collection count. |
| F-6bbddce1b930 | Indexed entry retrieval. |
| F-cf23980ec4fa | Entry operation before selected predicates. |
| F-5c131907de4a | First-phase candidate production. |
| F-51a9703516d3 | Auxiliary state and candidate operation. |
| F-6261bc5c6581 | No-explicit-argument operation before refreshed-tail predicate. |
| F-2eeed9557344 | Detached-tail retrieval. |
| F-ed04800b7b7b | Recovery comparison of current entry and candidate. |
| F-fa5325efe259 | Explicit entry cleanup-shaped operation. |
| F-c2674fa3e73d | Indexed replacement. |
| F-27727134fe01 | Optional context processing. |
| F-0a74fc9b8503 | Optional last-entry check. |
| F-16d2f95f7f7b | Collection operation before preparation and on ordinary failure. |
| F-8d0ea5a63663 | Default final processing. |
| F-9aef742d6dcd | Auxiliary cleanup-shaped operation. |

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Absent root:** With zero at `0x08`, report `(11, 127, 105)` and return `0x00000000FFFFFFFF`. Do not capture the event callback or perform ordinary cleanup.

2. **Creation failure:** With nonzero root and zero collection handle, a zero creation result is stored at `0x54`. Clear the auxiliary return word, report `(11, 127, 65)`, and perform ordinary failure cleanup.

3. **Single refreshed-tail invocation:** After refreshed-tail lookup and F-6261bc5c6581, invoke the predicate once with the retrieved entry as both entry arguments. A nonzero result triggers recovery; it does not trigger a second predicate invocation at this decision point.

4. **Positive status without a write:** Suppose the second phase is reached with candidate slot `0x1234`. If the producer returns positive one without changing the slot, pass `0x1234` to checked insertion. On insertion success, current becomes the retained `0x1234`. Positive status does not require a new candidate value.

5. **Negative second-phase status:** If the producer returns negative two and the retained auxiliary word is `0x12345678`, return `0x12345678FFFFFFFE` without shared cleanup. In single-entry recovery, the same producer status instead reaches event `18` handling.

6. **Insertion failure:** After positive second-phase production, zero insertion result causes candidate-slot cleanup, diagnostic `(11, 127, 65)`, and fixed zero return. Neither auxiliary cleanup nor detached-entry shared cleanup follows.

7. **Boundary event:** With post-extension self-predicate zero and no detached entry, position four selects event `2` for accounting value three and event `20` for accounting value four. Publish position three before notification.

8. **Detached-entry reinsertion:** A nonzero current-versus-detached predicate selects event `19` regardless of insertion return. Increment position, clear the detached reference, and write the new position to `0x50` before notification.

9. **Optional check:** Result one produces no event and sets processing result to one. Result two publishes event `28`; every other result publishes event `27`. Published-event continuation depends on the event callback result.

## Uncertainties

- **documented_only — Domain semantics:** Entry types, event meanings, and accounting semantics are unknown.
- **documented_only — Producer writes:** Passing the candidate-slot address proves writable exposure, not that any particular invocation writes a value. Untouched, stale, zero, or otherwise invalid candidate values are not locally rejected after positive status.
- **documented_only — Dependency signatures:** F-754616bec4de has differing visible argument counts. Its complete calling convention is unresolved.
- **documented_only — Ownership:** Acquisition, insertion, replacement, and cleanup-shaped usage do not establish allocation, ownership transfer, or destruction contracts.
- **documented_only — Arithmetic extremes:** Operand widths and signed tests are visible; portable overflow behavior and valid extreme indices are not established.
- **documented_only — Collection consistency:** Empty collections and counts changing between calls are not rejected locally.
- **documented_only — Mutation and reentrancy:** Dependencies may alter context, globals, or exposed storage. No synchronization or preservation contract is established.
- **Inferred intent:** “Collection,” “tail,” “insertion,” and “cleanup” summarize call usage rather than verified dependency implementations.

## Evidence

The supplied static body establishes the field offsets, visible argument widths, branch conditions, call ordering, explicit stores, cleanup routes, and return composition described here. The opaque mapping supplies dependency IDs only.

The tail decision contains one predicate invocation receiving the refreshed entry twice as arguments. Producer calls expose the candidate-slot address and return signed statuses; the body does not establish that those callbacks write the slot.

No implementation access, dependency inspection, runtime execution, or runtime tests were performed. Acceptance examples are conditional consequences of static evidence, not observed executions.
