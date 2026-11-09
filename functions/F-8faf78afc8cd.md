<!-- Trusted provenance: F-8faf78afc8cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8faf78afc8cd

## Purpose

F-8faf78afc8cd obtains an object address, initializes selected fields, performs five ordered status-bearing operations, and conditionally publishes the address through an optional output location. An option can also publish the address into parent storage before the checked operations begin.

The function returns a 32-bit status:

- **−50:** the parent address is zero.
- **−108:** the acquisition dependency returns zero.
- **Zero:** all five checked operations return zero and the subsequent completion calls return control.
- **Any other checked status:** initialization stops at the first nonzero checked result, invokes the failure-path dependency, and returns that result unchanged.

**Established static evidence:** the supplied representation establishes the direct accesses, dependency arguments, call ordering, status decisions, and output writes described below.

**Inferred intent:** “parent,” “object,” “initialization,” “completion,” and “cleanup” describe apparent roles. They do not establish concrete types, successful registration, ownership transfer, or resource release.

**Unknown:** dependency implementations, global values, object invariants, and the domain meanings of statuses and fields.

## Interface

The interface has five inputs and a 32-bit integer result.

| Input | Width and interpretation | Visible use |
|---|---|---|
| Parent address | 32-bit address-valued quantity | Rejected if zero; otherwise used for direct accesses and derived dependency arguments. |
| Associated value | Opaque 32-bit value | Stored at object offset `0x198`. |
| Options | Unsigned 32-bit bit field | Five bits select byte updates; one also selects an early parent-field write. |
| Completion argument | Opaque 32-bit value | Passed to F-7237a44426b8 on the successful path. |
| Output address | Optional address of a writable 32-bit location | Cleared at entry; written again on paths reaching common completion. |

Exactly zero denotes success for each checked dependency. Positive and negative nonzero results both select failure. No broader status interpretation is established.

### Memory preconditions

All scalar accesses below are four bytes unless explicitly identified as one-byte accesses. Offsets are byte offsets.

For execution to remain valid:

- A nonzero output address must be writable whenever an output store is reached.
- The parent must support the direct accesses and dependency operations actually reached.
- A nonzero acquired address must support the direct object accesses and dependency operations actually reached.
- Storage must remain valid for each access, including accesses after dependencies return.
- Dependencies’ additional preconditions must hold; these are unavailable.

The highest directly accessed object byte is `0x623`, belonging to the four-byte field at `0x620`. This establishes a direct-access extent, not a complete allocation size or a guarantee that dependencies stay within it.

The function does not visibly check alignment, storage extent, overlapping objects, output aliasing, or address arithmetic overflow. Invalid-memory behavior is not converted into a documented status.

Derived addresses use fixed offsets. No variable-size multiplication or allocation-size computation is visible. Safe address wrapping is not established.

## Behavior

### Entry and acquisition

If an output address is supplied, the function first writes four-byte zero there. This precedes the parent check and can therefore affect subsequently accessed storage when the output aliases it.

If the parent address is zero, the function returns −50 without calling a dependency.

Otherwise, it reads an opaque global acquisition argument and calls F-858f5b8d5984 with that value. A zero result produces −108 immediately. On that branch:

- No direct object initialization occurs.
- The failure-path dependency is not called.
- There is no second output write.

A nonzero result is retained as the acquired object address. Acquisition is not proven to allocate fresh storage.

### Initial direct accesses

After acquisition succeeds, the following operations occur in order. Global values are read at their stated positions, not captured together at entry.

| Order | Location or operation | Behavior |
|---|---|---|
| 1 | Object `0x008` | Read four bytes from parent `0x008`, then store that value here. |
| 2 | Object `0x004` | Read an opaque global value and store it here. |
| 3 | Object `0x198` | Store the associated-value input. |
| 4 | Object `0x02C` | Call F-2c43e072c88d with no explicit arguments, then store its 32-bit result here. |
| 5 | Object `0x190` | Read another opaque global value and store it here. |
| 6 | Global read | Read another opaque four-byte global value for use at object `0x0B8`. |
| 7 | Object `0x0C8` | Store 1. |
| 8 | Object `0x0B8` | Store the value read in step 6. |
| 9 | Object `0x0BC` | Store zero. |
| 10 | Object byte `0x1AC` | Read the byte and write its value ANDed with `0x8F`. |
| 11 | Object byte `0x618` | Store 1. |
| 12 | Object `0x61C` | Store zero. |
| 13 | Object `0x620` | Store zero. |

The update at `0x1AC` clears masks `0x10`, `0x20`, and `0x40`, preserving the remaining bits from that particular read.

F-4570bc8c62ec is then called with the interior address at object offset `0x19C`. Its result does not control subsequent execution.

This is selective initialization, not a whole-object clear. Unwritten bytes retain their existing contents unless changed indirectly by overlapping accesses or dependencies.

### Options and early publication

After F-4570bc8c62ec returns, supported option bits are processed in the following order.

| Option mask | Direct byte operation | Additional operation |
|---|---|---|
| `0x00000001` | OR `0x10` into object byte `0x18C`. | None. |
| `0x00000002` | OR `0x40` into object byte `0x18C`. | Immediately store the acquired address at parent `0x03C`. |
| `0x00000004` | OR `0x20` into object byte `0x18C`. | None. |
| `0x00000010` | OR `0x20` into object byte `0x18D`. | None. |
| `0x00000020` | OR `0x10` into object byte `0x18D`. | None. |

Each enabled option performs its own byte read-modify-write. Disabled options perform no corresponding access and do not clear existing flag bits. Other option bits, including `0x00000008`, have no direct effect. The full options value is not forwarded to a dependency.

For option `0x02`, publication into parent `0x03C` occurs before checked initialization. There is **no explicit parent-targeted rollback** on failure. This does not guarantee that the published address remains there: dependencies or later direct stores through overlapping addresses can replace it.

Following option processing, the function writes four-byte zero to object offsets `0x210`, `0x218`, and `0x21C`, in that order. No direct initialization of `0x214` is shown. These stores can themselves overwrite the early-publication location if parent and acquired-object storage overlap suitably.

### Checked initialization

The following operations execute in order, stopping at the first nonzero result.

| Step | Dependency | Explicit arguments |
|---|---|---|
| 1 | F-8c5b7e6f4bf3 | Acquired object address. |
| 2 | F-1935c5244ef1 | Object address plus `0x048`. |
| 3 | F-54060b1135e1 | Object address plus `0x07C`; an opaque global value read for this call; acquired object address. |
| 4 | F-6caf95f9c95b | Parent address plus `0x054`; object address plus `0x07C`. |
| 5 | F-6caf95f9c95b | Object address plus `0x048`; parent address plus `0x088`. |

An earlier failure suppresses every later checked call. The global argument for step 3 is read only if execution reaches that step.

The two calls to F-6caf95f9c95b have distinct argument pairs. Their semantic relationship is unknown.

### Successful completion

If all five checked results are zero:

1. Store the original parent address at object offset `0x00C`.
2. Call F-7237a44426b8 with the acquired object address and completion argument.
3. Read the four-byte value at object offset `0x00C` after that call.
4. Call F-fd61d89535ad with the value just read and the integer 1.
5. If an output address was supplied, store the retained acquired address there.
6. Return zero.

The value passed to F-fd61d89535ad is not guaranteed to equal the original parent address. F-7237a44426b8 may modify the field that is reread.

Neither completion call contributes a checked status. A zero return establishes that the five checked results were zero and subsequent calls returned control; it does not establish that those calls achieved their intended effects or preserved object lifetime.

### Checked failure

If any checked operation returns nonzero:

1. Call F-7d6ed32294de with the acquired object address and the integer 1.
2. Replace the retained candidate for output with zero.
3. If an output address was supplied, write zero there.
4. Return the failing 32-bit status unchanged.

There is no explicit reversal of earlier object initialization, early publication, or previous dependency effects. Dependencies may perform reversal, but that behavior is unknown. Later direct writes can also overwrite earlier effects through aliasing without constituting an explicit rollback operation.

## Side effects

### Output writes and aliasing

The function directly writes a supplied output location:

- Once on the null-parent path.
- Once on the zero-acquisition-result path.
- Twice on paths reaching successful completion or checked failure.

These counts describe the function’s own stores, excluding dependency effects.

No non-aliasing condition is enforced. Consequently:

- The entry-time zero can modify parent fields, object storage later returned by acquisition, or other subsequently read storage.
- The final output store can overwrite a parent or object field.
- Parent-targeted stores can overwrite object fields when their storage overlaps.
- Object-targeted stores can overwrite parent fields under the same condition.
- Partial overlaps can affect only some bytes; their interpretation can depend on byte order.

In particular, if the output address equals parent `0x03C`, option `0x02` first publishes the acquired address there, but checked failure later writes zero to that same location. This happens even if no dependency modifies the location.

Likewise, later object stores can overwrite parent `0x03C` before any checked failure. Retention of early publication requires the absence of both dependency modifications and overlapping subsequent direct writes.

### Ownership and lifetime

The visible function obtains an address from F-858f5b8d5984, supplies it to F-7d6ed32294de after checked failure, and omits that failure-path call on success.

These facts do not establish:

- Fresh allocation.
- Ownership transfer.
- Reference counting.
- Deallocation.
- Retention or validity after any dependency returns.

Early publication can therefore expose an address before checked initialization succeeds. If no dependency or overlapping later direct write changes parent `0x03C`, that address remains there after failure. Whether it then designates live storage is unknown.

No locking, atomic access, or concurrency guarantee is visible.

## Dependencies

Only opaque function IDs identify dependencies.

| Dependency | Established call-site role |
|---|---|
| F-858f5b8d5984 | Receives an opaque global argument; returns zero or an address used for initialization. |
| F-2c43e072c88d | Supplies the 32-bit value stored at object `0x02C`. |
| F-4570bc8c62ec | Receives object interior address `0x19C` before option processing. |
| F-8c5b7e6f4bf3 | First checked operation. |
| F-1935c5244ef1 | Second checked operation. |
| F-54060b1135e1 | Third checked operation, including a freshly read global argument. |
| F-6caf95f9c95b | Fourth and fifth checked operations with different address pairs. |
| F-7237a44426b8 | Successful-path call receiving the completion argument. |
| F-fd61d89535ad | Successful-path call receiving the reread object `0x00C` value and 1. |
| F-7d6ed32294de | Failure-path call receiving the acquired address and 1. |

No dependency is assumed to preserve globals, fields, memory validity, or ownership, or to return normally. Global reads occurring after calls observe the values present at those later reads.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. They assume valid reached accesses and normal dependency returns. Any additional preservation assumptions are stated explicitly.

1. **Null parent with output.**  
   The function writes zero to the supplied output, invokes no dependency, and returns −50.

2. **Acquisition returns zero.**  
   With a nonzero parent, F-858f5b8d5984 receives the current acquisition-global value and returns zero. The function returns −108 without object initialization or failure cleanup. The output has no second direct write; it remains zero only if no intervening effect changes it.

3. **First checked operation fails.**  
   Acquisition succeeds and options are zero. Initial direct accesses, F-4570bc8c62ec, and the three post-option zero stores occur. F-8c5b7e6f4bf3 returns a nonzero status. No later checked operation or completion call occurs. F-7d6ed32294de receives the acquired address and 1. A supplied output is then zeroed, and the failing status is returned unchanged.

4. **Final checked operation fails.**  
   The first four checked operations return zero; the fifth returns a nonzero status. Failure cleanup occurs, but neither completion call occurs. The function performs no explicit reversal of earlier checked effects.

5. **All supported option bits.**  
   For options `0x37`, direct flag updates apply masks `0x10`, `0x40`, and `0x20` to byte `0x18C`, then `0x20` and `0x10` to byte `0x18D`. Publication to parent `0x03C` occurs immediately after the `0x40` update. Each byte update preserves other bits from its own read; overlapping stores may separately affect them.

6. **Completion changes the recorded parent.**  
   All checked results are zero. F-7237a44426b8 changes object `0x00C` to another valid value and leaves it readable. F-fd61d89535ad receives that changed value. After it returns, a supplied output receives the acquired address, and the function returns zero.

7. **Early publication retained after failure.**  
   Option `0x02` is enabled and a checked operation fails. If neither dependencies nor overlapping subsequent direct writes modify parent `0x03C`, it retains the acquired address. A separate output location receives zero. The retained address’s lifetime remains unknown.

8. **Output aliases early-publication field.**  
   The output address equals parent `0x03C`. Option `0x02` publishes the acquired address there. A checked operation later fails. Even if dependencies leave that location untouched, the final output store writes zero there. Absence of explicit parent-targeted rollback does not imply retention.

9. **Object initialization overlaps early publication.**  
   Suppose valid storage places the acquired object address `0x1D4` bytes below the parent address. Object `0x210` then coincides with parent `0x03C`. With option `0x02`, publication writes the acquired address, and the subsequent zero store to object `0x210` overwrites it before checked initialization. No dependency modification is required.

## Uncertainties

The following remain **documented_only**:

- Concrete object types, field meanings, flag meanings, and global values.
- Allocation, reuse, ownership, lifetime, and rollback behavior.
- Dependency memory footprints, status meanings, and exceptional or non-returning behavior.
- Required invariants beyond the visible accesses.
- Endianness, alignment requirements, synchronization, and invalid-memory behavior.
- Exact machine-level consequences of overflowing address derivations.
- Whether completion dependencies can internally fail despite a final zero status.
- Whether dependencies preserve storage needed by subsequent direct accesses.
- Whether the supplied representation preserves every machine-level evaluation detail.

Lack of an explicit aliasing check does not establish that every overlapping arrangement satisfies the dependencies’ unknown preconditions. The aliasing examples are conditional on valid execution.

## Evidence

This contract uses the supplied static representation and opaque dependency mapping. Prior prose, comments, and naming implications are not independent behavioral evidence.

**Established:** direct access widths and offsets, masks, ordered stores, dependency arguments and call order, short-circuit status handling, early publication, output writes, and return values. The aliasing consequences follow from those accesses when their addressed storage overlaps.

**Inferred intent:** object acquisition, parent association, initialization, completion, and failure cleanup.

**Unverified:** dependency internals, runtime validity, ownership, concurrency behavior, and domain-level success.

No runtime tests have been performed.
