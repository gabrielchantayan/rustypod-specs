<!-- Trusted provenance: F-2536c4e7dfbd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2536c4e7dfbd

## Purpose

F-2536c4e7dfbd processes an input object against a destination-associated resource, conditionally sets two destination flags, and updates a destination-associated scheduling record. When that record is not already marked active, the function may insert it into a global singly linked list and then conditionally invoke an indirect dispatch target.

The resource-processing and list-manipulation effects are established by the supplied static evidence. Describing the record as a timer, the list as a scheduling queue, or the final indirect call as a scheduler wake-up is inferred intent, not an established dependency contract.

The function has no explicit result value. Its observable results are dependency calls, memory reads and writes, possible list insertion, and possible indirect dispatch.

## Interface

### Inputs

The function receives two 32-bit address-like arguments:

- **Destination:** an object whose relevant fields include a resource reference, two one-byte flags, and a scheduling-record reference.
- **Input:** an object passed to two dependencies and subsequently read for a four-byte field.

The supplied representation uses 32-bit addresses and four-byte words. This contract describes that representation; it does not establish portability to a wider-pointer environment.

### Destination layout

Offsets are byte offsets from the destination address.

| Offset | Width | Established use |
|---|---:|---|
| `0xb0` | 4 bytes | Resource reference passed to F-985e3cec822f. |
| `0xbd` | 1 byte | Set to `1` when F-985e3cec822f returns nonzero. |
| `0xc1` | 1 byte | Set to `1` immediately before the write at `0xbd`, under the same condition. |
| `0xc4` | 4 bytes | Scheduling-record address, loaded independently at three points. |

A zero result from F-985e3cec822f does not cause either flag to be cleared.

### Input layout

| Offset | Width | Established use |
|---|---:|---|
| `0x0c` | 4 bytes | Captured after F-8bb3020950cd returns and passed to F-985e3cec822f. |

Other input requirements are dependency-defined and unknown.

### Scheduling-record layout

The record address ultimately used for direct accesses is captured from destination offset `0xc4` after two preceding scheduling-related calls.

| Offset | Width | Established use |
|---|---:|---|
| `0x00` | 4 bytes | Next-record link written during insertion. |
| `0x04` | 4 bytes | Interval-like value; zero prevents insertion. |
| `0x08` | 4 bytes | Computed deadline-like value and address passed to F-0d70d6f9010c. |
| `0x18` | 4 bytes | Receives a global value during nonzero-interval insertion. |
| `0x1c` | 4 bytes | Active/list-membership marker; tested for nonzero and later set to `1`. |
| `0x20` | 4 bytes | Receives a global callback-like value before the active-marker test. |

The names “interval,” “deadline,” and “active” describe apparent roles. Their broader meanings are not established.

### Preconditions and output

For normal completion, every address actually dereferenced must designate accessible memory of the required width. Dependency-specific preconditions must also hold. The function does not validate destination, input, scheduling-record, list-node, or global-pointer addresses before using them.

The function returns without an explicit value. No recoverable error result is defined.

## Behavior

### Input processing and destination flags

The function first calls F-8bb3020950cd with the input address. It then captures the four-byte value at input offset `0x0c`.

Next it calls F-c0a513a969e0 with the input address. After that call returns, it reads the destination resource reference at offset `0xb0` and calls F-985e3cec822f with:

1. The resource reference read from the destination.
2. The previously captured input field.
3. The result from F-c0a513a969e0.

This ordering matters if dependencies modify shared memory. The input field is captured before F-c0a513a969e0, while the resource reference is read afterward.

If F-985e3cec822f returns any nonzero value, the function writes the one-byte value `1` to destination offset `0xc1`, then writes the one-byte value `1` to offset `0xbd`. No other bits or bytes in those fields are explicitly updated. If the result is zero, neither flag is directly written.

Scheduling-related processing follows regardless of whether that result was zero or nonzero.

### Scheduling-record preparation

The function performs three separate reads of destination offset `0xc4`:

1. The first address is passed to F-936a8728139d.
2. After that call, the field is read again and passed to F-6904f7699031 together with the integer value `500`.
3. After that call, the field is read a third time and retained as the record address for subsequent direct accesses.

Consequently, these calls and later accesses need not target the same record if a dependency changes destination offset `0xc4`. An equivalent implementation must not collapse these reads into one cached reference without an additional stability guarantee.

After capturing the third address, the function calls F-936a8728139d again. No explicit argument is recoverable for this occurrence. It must not be silently equated with the earlier call that explicitly receives a record address.

After that call returns, the function reads a global callback-like value and writes its four-byte representation to record offset `0x20`. It then reads the four-byte marker at record offset `0x1c`.

If the marker is nonzero, the function returns immediately. This path still includes all preceding calls and the callback-like field write. It does not explicitly perform list insertion, call F-5bcae2eff9a7, or evaluate the final dispatch condition.

### Inactive-record processing

When the marker is zero, the function:

1. Reads a global coordination value and passes it to F-338d16f9ce72.
2. Calls F-add44d4e9aaa with the retained record address.
3. Reads the four-byte interval-like field at record offset `0x04`.

The interval is therefore sampled after both dependencies have returned. Neither dependency is known to preserve that field, the list, or other globals.

If the interval is zero, the function skips deadline computation and insertion but continues to the common post-processing path.

### Deadline computation

For a nonzero interval, the function invokes a dependency with no explicit arguments and uses its returned value in a deadline-like calculation. This dependency has no supplied opaque function ID.

The stored deadline is the low 32 bits of:

**returned value + interval × 1000**

Both the destination field and the arithmetic intermediates shown are 32-bit. There is no explicit range check, saturation, or overflow report. In the represented 32-bit machine model, multiplication and addition retain their low 32 bits; mathematical results outside that range cannot be treated as preserved values.

The interval’s nonzero test is not a positivity test. A bit pattern interpreted as a negative signed 32-bit value also enters this path. Whether such an interval is valid is unknown.

After the return from this dependency, the function captures the global list-head storage address. It then writes the computed deadline at record offset `0x08`, reads a separate global value, and writes that value at offset `0x18`.

The time-like interpretation of the dependency and the units implied by multiplication by `1000` are inferred intent only.

### List traversal and insertion

The function reads the list head through the captured head-storage address and traverses a singly linked list.

For each non-null candidate node, it calls F-0d70d6f9010c with two addresses, in this order:

1. Candidate node address plus `0x08`.
2. New record address plus `0x08`.

A zero result means traversal continues. The candidate’s next link is read only after that comparison returns. A nonzero result stops traversal at that candidate.

Thus, comparison calls may affect subsequent traversal if they modify candidate links. The exact ordering relation is unknown: the evidence does not establish whether a nonzero result means “earlier,” “later,” “equal,” or another predicate.

Insertion occurs before the first candidate for which the comparison returns nonzero, or at the end if all comparisons return zero. An initially empty list causes insertion without any comparison call.

The insertion writes occur in this order:

1. Update either the head storage or the previous node’s next link to point to the new record.
2. Write the four-byte value `1` to the new record’s marker at offset `0x1c`.
3. Write the stopping candidate address, possibly null, to the new record’s next link at offset `0x00`.

This order publishes the new record into the list before its marker and next link are updated. Whether concurrent readers can observe that intermediate state depends on synchronization behavior not established here.

The traversal has no explicit cycle detection, iteration bound, or duplicate-record check.

### Common post-processing and dispatch

After the inactive-record path, including the zero-interval case, the function reads a global post-processing value and passes it to F-5bcae2eff9a7.

After that call returns, it independently reloads the global coordination pointer used in the final condition. It reads a four-byte pointer through that address. If the resulting pointer is null, it returns.

If the pointer is non-null, the function reads its first four-byte word. If that word is zero, it returns. If it is nonzero, the function invokes a global indirect target with no explicit arguments visible in the supplied representation.

The indirect transfer was not fully resolved. Its exact calling convention, arguments, return behavior, and effects are documented_only unknowns.

## Side effects

### Explicit memory writes

Subject to the branches above, the function directly writes:

- Two one-byte destination flags, in the order `0xc1`, then `0xbd`.
- The retained record’s four-byte callback-like field.
- For a nonzero interval on the inactive path:
  - The four-byte deadline-like field.
  - The four-byte global-derived field at offset `0x18`.
  - Either global head storage or a predecessor node’s next link.
  - The four-byte active marker.
  - The record’s four-byte next link.

It does not explicitly clear a flag, reset an active marker to zero, allocate storage, or free storage.

### Ownership and lifetime

No ownership transfer or release is established by the visible operations. Dependencies may have additional ownership effects; those must not be presumed absent.

After successful insertion, the list contains the retained record address. Safe subsequent use therefore requires that record storage to remain valid for the list’s actual lifetime. The owner responsible for that lifetime is unknown.

### Synchronization and errors

F-338d16f9ce72 and F-5bcae2eff9a7 occur in positions consistent with coordination around list work, but no lock, unlock, wake-up, or exclusion semantics are proven.

No exception handling, rollback, status propagation, or invalid-pointer recovery is visible. A failing or non-returning dependency can prevent later operations. Partial changes made before such a failure are not explicitly reversed.

## Dependencies

| Opaque ID or unresolved dependency | Established interaction |
|---|---|
| F-8bb3020950cd | Called first with the input address; result not consumed. |
| F-c0a513a969e0 | Called with the input address; result becomes the third argument to F-985e3cec822f. |
| F-985e3cec822f | Receives the resource reference, captured input field, and preceding result; nonzero controls both flag writes. |
| F-936a8728139d | Called once with an explicit scheduling-record address and later with no explicit argument recoverable. |
| F-6904f7699031 | Receives a freshly read scheduling-record address and `500`; result not consumed. |
| F-338d16f9ce72 | Receives a global value on the inactive-record path. |
| F-add44d4e9aaa | Receives the retained record before its interval is sampled. |
| F-0d70d6f9010c | Compares or otherwise evaluates two field addresses; zero continues traversal, nonzero stops it. |
| F-5bcae2eff9a7 | Receives a global value after inactive-record processing, including when insertion was skipped. |
| Unmapped deadline-source dependency | Called without explicit arguments; return value participates in 32-bit deadline arithmetic. Opaque ID unavailable. |
| Unresolved indirect target | Called only when the final pointer and its first word are both nonzero. Opaque ID unavailable. |

No dependency is assumed to preserve globals, preserve record contents, allocate memory, or release memory.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests.

1. **Nonzero processing result.** If F-985e3cec822f returns `7`, the function must write byte `1` at destination `0xc1` before writing byte `1` at `0xbd`. Subsequent scheduling processing must still occur.

2. **Zero processing result with existing flags.** If that dependency returns zero and both flags initially contain nonzero values, the function must not explicitly clear or rewrite them. Dependency-originated changes remain possible.

3. **Already-active record.** If the retained record’s marker is nonzero when tested, its callback-like field must already have been updated. The function must then return without entering the inactive-record branch.

4. **Record reference changes between calls.** If the first scheduling dependency changes destination `0xc4` from one valid record address to another, the second scheduling call must receive the newly read address. The later retained address must reflect the third read, not necessarily either earlier address.

5. **Inactive record with zero interval.** If the interval is zero after F-add44d4e9aaa returns, the function must not explicitly compute a deadline or insert the record. It must still call F-5bcae2eff9a7 and evaluate the subsequent dispatch condition.

6. **Insertion into an empty list.** With a nonzero interval and a null head at traversal start, no F-0d70d6f9010c call is required. The head must be written to the record address, followed by marker `1`, followed by a null next link.

7. **Insertion after continuing comparisons.** If comparisons for the first two nodes return zero and the third returns nonzero, the second node’s link must be updated to the record. The record’s next link must ultimately identify the third node.

8. **Arithmetic wrap.** If the deadline-source result is `0xfffffff0` and the interval is `1`, the stored deadline must be `0x000003d8` under the represented 32-bit arithmetic. No overflow error is explicitly produced.

9. **Dispatch suppression.** After F-5bcae2eff9a7 returns, a null pointer loaded through the coordination pointer suppresses dispatch without dereferencing a node. A non-null pointer whose first word is zero also suppresses dispatch.

10. **Dispatch enabled.** If that pointer and its first word are both nonzero, the indirect target must be invoked, subject to all preceding operations returning normally.

## Uncertainties

The following are documented_only:

- Full object definitions, alignment requirements, and dependency-specific accessible-memory requirements.
- Meaning and units of `500`, the interval field, and the deadline-source result.
- Signedness and ordering semantics used by F-0d70d6f9010c, including equality and wraparound handling.
- Why F-936a8728139d appears with different recoverable argument forms.
- Whether F-add44d4e9aaa removes existing links or performs other normalization.
- Whether coordination dependencies establish synchronization or memory-order guarantees.
- Ownership, allocation, release, and reentrancy behavior of every dependency.
- Behavior for malformed lists, self-links, cycles, aliasing, and concurrent mutation.
- Exact semantics and control-flow properties of the unresolved indirect target.

These unknowns do not justify assuming sorted order, safe concurrent publication, stable globals, or automatic memory management.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping. Established observations include access widths and offsets, call order, conditional branches, explicit stores, pointer traversal, and arithmetic structure.

Scheduling terminology, callback terminology, and coordination intent are interpretations, not independently established facts. The unrecovered indirect transfer and unavailable dependency bodies limit stronger conclusions.

No runtime tests, dependency implementations, or external behavioral observations were provided or examined. No runtime verification has been performed.
