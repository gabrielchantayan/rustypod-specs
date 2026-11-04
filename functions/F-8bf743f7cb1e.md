<!-- Trusted provenance: F-8bf743f7cb1e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8bf743f7cb1e

## Purpose

F-8bf743f7cb1e selects an index through a reverse search of a globally referenced collection, or invokes a creation-and-insertion path when the search finds no qualifying element. It then writes the selected index and two caller-supplied values into three consecutive 32-bit fields of a destination object.

The search treats any nonzero result from F-efdae002d289 as qualifying. When multiple elements qualify, the first encountered in descending index order wins.

The apparent intent is to find or create a collection entry associated with a supplied key, then record an association in another object. That interpretation is inferred. The established behavior is the sequence of dependency calls, memory accesses, branching, and destination writes described below; the dependencies’ semantic contracts are not supplied.

## Interface

**Return:** No explicit return value.

**Inputs:**

| Input | Representation | Visible use |
|---|---|---|
| Destination | Address represented by a 32-bit machine value | Base address for three final writes |
| Search/construction value | Opaque 32-bit value | Passed to F-efdae002d289 during search and to F-b9acea4b06ad during creation |
| Associated value | Opaque 32-bit value | Stored at destination offset 188 bytes |
| Original pointer | Pointer-sized, represented as 32 bits | Used to initialize a local pointer slot and later stored unchanged at destination offset 192 bytes |

The original pointer is not directly dereferenced by this function. Its pointee type, validity requirements beyond those imposed by unknown callees, and ownership are not established.

**Destination layout:**

| Byte offset | Width | Value written on normal completion |
|---|---:|---|
| 184, or 0xB8 | 32 bits | Selected index, interpreted as a signed integer |
| 188, or 0xBC | 32 bits | Associated value |
| 192, or 0xC0 | 32 bits | Original pointer |

No destination size, alignment, or validity check is visible. Normal completion requires these locations to be writable and addressable under the target memory model.

**Externally referenced state:**

- A global reference to a collection-like object.
- A 32-bit signed count read from byte offset 4 of that object.
- A first-word dispatch-table reference in the initially captured collection object.
- A separate global 32-bit value copied into the first word of a newly produced object.

These are descriptive roles, not claims about an undocumented collection or object ABI.

## Behavior

### Initial state capture

The function first captures the current global collection reference. It then obtains the initial count by reading the second 32-bit word through the global collection reference.

The captured reference and the initial count serve different purposes:

- The captured reference is retained for later dispatch-table lookup if creation is necessary.
- The initial count supplies both the search starting point and the index recorded on the creation path.

A local pointer slot is initialized with the original pointer input. The destination fields are not written during this initialization.

The initial count is a snapshot. It is not visibly refreshed after dependency calls, even if those calls change the collection or its count.

### Reverse search

The search starts from the initial count and decrements the working index before checking whether it is negative.

For an ordinary nonnegative count \(N\):

- If \(N=0\), the function enters the creation path without calling either search dependency.
- If \(N>0\), candidate indices are considered in the order \(N-1, N-2, \ldots, 0\).
- Once the working index becomes negative, the function enters the creation path.

For each nonnegative candidate index:

1. F-6495483bd2c4 receives the collection reference obtained from the global reference at that call site and the current index.
2. Its opaque 32-bit result is passed to F-efdae002d289 together with the search/construction value.
3. A zero result from F-efdae002d289 continues the search.
4. Any nonzero result ends the search successfully.

A successful search selects the current index. The function does not invoke the creation dependencies or the indirect insertion dependency on that path.

The function does not visibly retain one collection reference for all search calls. Consequently, the initial count and the objects searched need not remain associated if a dependency changes the global reference. The supplied evidence contains no synchronization, revalidation, or recovery for that condition.

### Creation and insertion path

When the decrement-and-negative check ends the search, the function performs these operations in order:

1. It calls F-8157a180d901 with the integer value 8.
2. It passes that dependency’s opaque 32-bit result and the search/construction value to F-b9acea4b06ad.
3. It replaces the local pointer slot with the pointer returned by F-b9acea4b06ad.
4. It reads the separate global 32-bit value and writes it into the first 32-bit word of the returned object.
5. It invokes an indirect dependency, designated here as **F-indirect-001**, with:
   - the collection reference obtained from the global reference for this invocation; and
   - the address of the local pointer slot.
6. After that invocation returns, it selects the original count snapshot as the index to record.

The value 8 is an established argument. Its interpretation as an allocation size is inferred, not supplied by a dependency contract.

Likewise, the final indirect call resembles collection insertion, but successful insertion, insertion position, count adjustment, and ownership transfer are not independently established.

F-indirect-001 is resolved through a dispatch-table reference read from the first word of the **initially captured collection object**. The selected entry is at byte offset 28, or 0x1C, in that table.

The dispatch lookup and receiver need not refer to the same object: the former uses the captured reference, while the latter uses the global reference available for the call. No check reconciles them if dependencies have changed the global reference.

F-indirect-001 receives the address of the local pointer slot, not merely its pointer value. It can therefore potentially inspect or replace that value. Any replacement is not subsequently copied into the destination by the visible function.

### Final destination writes

After either a search match or a normally returning creation path, the function writes:

1. The selected index at destination offset 184.
2. The associated value at destination offset 188.
3. The **original pointer input** at destination offset 192.

On the creation path, the first write occurs only after F-indirect-001 returns.

The pointer stored in the destination is not the result of F-b9acea4b06ad and is not a value substituted into the local pointer slot by F-indirect-001.

These three writes are ordered operations, not an atomic record update. No rollback is visible if execution fails between them.

## Side effects

**Direct memory writes:**

- Initialization and possible replacement of a local pointer slot.
- On the creation path, one 32-bit write to the first word of the object returned by F-b9acea4b06ad.
- On normal completion, three 32-bit destination writes in the order specified above.

**Indirect effects:**

All called dependencies may have additional effects that are not described by the supplied body. They must not be assumed to preserve globals, preserve collection contents, avoid reentrancy, or leave the destination untouched.

**Ownership and lifetime:**

- No explicit release operation appears.
- The original pointer is stored, not copied by dereferencing its pointee.
- Passing a local slot’s address to F-indirect-001 does not establish whether the callee borrows it only during the call or retains it.
- The lifetime and ownership of the object returned by F-b9acea4b06ad are unknown.
- The apparent insertion operation does not, by itself, prove ownership transfer.

**Failure handling:**

No explicit checks cover null references, invalid indices, allocation failure, an invalid returned object, an invalid dispatch table, or an invalid destination. No error status or cleanup path is visible.

If a dependency faults, aborts, or otherwise fails to return normally, the subsequent operations are not reached through the visible control flow. Earlier dependency effects or memory writes are not visibly undone.

**Aliasing:**

No disjointness checks are present. If the destination overlaps collection state, the created object, or another dependency-accessible region, the final writes may modify that region. Any such behavior depends on the actual layout and dependency effects; non-aliasing is not established by the interface.

## Dependencies

| Opaque ID | Established call contract at this site | Unknowns |
|---|---|---|
| F-6495483bd2c4 | Receives a global collection reference and a nonnegative signed index; returns an opaque 32-bit value used by F-efdae002d289 | Retrieval semantics, bounds handling, side effects, returned-value lifetime |
| F-efdae002d289 | Receives the preceding result and the search/construction value; zero continues, nonzero selects the current index | Equality or ordering semantics, valid inputs, side effects |
| F-8157a180d901 | Receives 8; its 32-bit result is passed to F-b9acea4b06ad | Allocation semantics, failure representation, ownership |
| F-b9acea4b06ad | Receives the preceding result and the search/construction value; its result is used as a writable pointer | Construction semantics, resulting layout, failure behavior, ownership |
| F-indirect-001 | Locally assigned label for the unresolved indirect target; dispatch entry at byte offset 28; receives the global collection reference and a pointer to a local pointer slot | Actual identity, insertion behavior, return convention, ownership and retention rules |

F-indirect-001 is a specification-local label, not a resolved identity supplied by the dependency map. No return value from that call is consumed.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence. They are not runtime test results. Unless stated otherwise, they assume valid memory, normally returning dependencies, and no dependency-induced mutation relevant to the scenario.

### Match at the highest searched index

Given an initial count of 4, if F-efdae002d289 returns nonzero for the value obtained at index 3:

- Only index 3 is searched.
- The creation path is not taken.
- The destination receives index 3, the associated value, and the original pointer, in that order.

A negative predicate result qualifies just as a positive result does.

### Match after one rejection

Given an initial count of 4, if the predicate returns zero at index 3 and nonzero at index 2:

- Retrieval and predicate calls occur for indices 3 and 2, in that order.
- Indices 1 and 0 are not examined.
- The recorded index is 2.

If index 0 would also qualify, that does not affect this outcome.

### Empty collection

Given an initial count of zero:

- No retrieval or predicate call occurs.
- F-8157a180d901 receives 8.
- F-b9acea4b06ad receives its result and the search/construction value.
- The returned object’s first word is overwritten with the global 32-bit value.
- F-indirect-001 is invoked.
- On normal return, the recorded index is zero.

### No qualifying element

Given an initial count of 3 and zero predicate results for all candidates:

- The searched indices are 2, 1, and 0.
- Creation follows.
- The recorded index is the original count, 3, regardless of any count value changed by the dependencies.

### Local pointer replacement

Suppose F-b9acea4b06ad returns one pointer and F-indirect-001 replaces the local slot with another pointer. On normal completion, destination offset 192 still receives the original pointer input.

### Global collection replacement

Suppose a search dependency changes the global collection reference after the first rejected candidate:

- A later retrieval uses the collection reference available at its own call site.
- The remaining indices still derive from the original count.
- If creation is reached, dispatch lookup still uses the initially captured object, while the indirect call’s receiver comes from the global reference.

This is a consequence of the visible references, not an assurance that the dependencies support such replacement.

### Negative initial count

For an initial signed count of −1, the first decrement produces −2 and immediately selects creation. If creation returns normally, the recorded index is the original count, −1. No visible normalization converts it to zero.

## Uncertainties

The following remain **documented_only**:

- **Semantic roles:** “Collection,” “key,” “construction,” and “insertion” describe apparent intent. The dependency contracts do not establish those meanings.
- **Signed overflow:** Decrementing the minimum signed 32-bit value is outside the ordinary signed arithmetic range. The supplied representation does not establish whether the executable wraps, traps, or has another relevant behavior. No outcome is specified for that count.
- **Pointer arithmetic:** Normal completion assumes the destination-offset calculations and object accesses resolve to valid target addresses. Address-wrap behavior is not established.
- **Memory representation:** Access widths and offsets are visible; byte order and broader object layouts are not.
- **Dependency mutation:** No preservation of globals, object fields, destination memory, or collection count is assumed.
- **Dispatch evaluation:** The indirect target uses the captured object and the receiver uses the global reference. The relative evaluation order of those call-setup reads is not separately established.
- **Concurrency and reentrancy:** No locking or consistency mechanism is visible.
- **Resource semantics:** Allocation success, object ownership, insertion ownership, and retention of the local slot’s address remain unknown.
- **Failure mechanism:** Exceptions, faults, nonlocal exits, and dependency-specific error conventions are not supplied.

## Evidence

**Established static evidence:** The supplied function body supports the four-input interface, 32-bit accesses, destination offsets, predecrement reverse search, zero/nonzero predicate rule, creation-call sequence, first-word object write, indirect-dispatch offset, use of the original count on creation, and final storage of the original pointer.

**Inferred intent:** The body resembles find-or-create collection handling followed by association-field updates. This interpretation does not replace the explicit observable contract.

**Unknown evidence:** Dependency implementations, a complete ABI description, runtime state, and failure contracts were not supplied. No runtime tests have been performed.
