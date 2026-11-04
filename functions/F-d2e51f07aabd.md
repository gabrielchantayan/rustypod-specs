<!-- Trusted provenance: F-d2e51f07aabd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2e51f07aabd

## Purpose

F-d2e51f07aabd searches a collection for a record whose first 32-bit word matches a supplied key. On a match, it returns that record’s second 32-bit word. If no match is found, it attempts to allocate a new two-word record and obtain an associated value through F-a028838d7a4b. A nonzero associated value triggers an insertion-like indirect call.

The established behavior is a lookup followed, on a miss, by a construction attempt. Treating the collection as a cache, the key as an object identifier, or the returned word as an owned object pointer is **inferred intent**, not an established contract.

A zero result is not a unique error indicator: an existing matching record may contain zero, the record allocation may fail, or F-a028838d7a4b may return zero.

## Interface

### Inputs

The function accepts four arguments:

| Position | Established representation | Observed use |
|---|---|---|
| 1 | 32-bit integer key | Compared with the first word of each inspected record; stored in a newly allocated record; passed to F-a028838d7a4b. |
| 2 | Opaque 32-bit word | Not read or forwarded by the visible function body. |
| 3 | Opaque 32-bit word | Not read or forwarded by the visible function body. |
| 4 | Pointer-shaped argument | Initially copied into a local pointer slot, then overwritten on the miss path. Its incoming value is never dereferenced or forwarded. |

The second, third, and fourth arguments do not affect the visible control flow or memory accesses. In particular, the fourth argument is not an established output parameter. The address passed to the insertion-like dependency is the address of a local slot, not the incoming fourth argument.

### Output

The return value is a 32-bit integer word obtained from one of these sources:

- The second word of the first matching record encountered during traversal.
- An explicit zero when allocation of the two-word record fails.
- The second word of the newly allocated record when F-a028838d7a4b returns zero.
- The second word of the record designated by the local pointer slot after the insertion-like call returns.

The last case is significant: the insertion-like dependency receives access to that pointer slot and may replace its contents. The function does not necessarily return the value originally stored in the newly allocated record.

### Relevant layouts

The supplied representation uses four-byte words and pointer slots.

**Collection object**

- Byte offset `+0x00`: pointer to an indirect-dispatch table.
- Byte offset `+0x04`: signed 32-bit traversal count.

**Dispatch table**

- Byte offset `+0x40`: callable entry used to retrieve an indexed element.
- Byte offset `+0x1c`: callable entry used on the insertion-like path.

These are byte offsets into the dispatch table, not fields directly invoked within the collection object.

**Indexed element**

- Byte offset `+0x00`: pointer to a record.

**Record**

- Byte offset `+0x00`: 32-bit key.
- Byte offset `+0x04`: 32-bit associated value.

The function allocates eight bytes for a new record. It separately requests 36 bytes for an object or storage region passed to F-a028838d7a4b. The layout and eventual use of those 36 bytes are unknown.

## Behavior

### Collection acquisition and traversal

The function first calls F-db2e4665a74c without explicit arguments. It treats the returned value as a collection pointer and reads the signed 32-bit count at byte offset `+0x04`.

There is no visible null check on the collection pointer before that read.

The count is captured once for traversal. Subsequent changes to the collection’s count do not change this invocation’s loop bound. Traversal starts at index zero and advances by one while the index is less than the captured count using a signed comparison.

Consequences of that comparison include:

- A count of zero produces no indexed-element calls.
- A negative count also produces no indexed-element calls.
- A positive count permits inspection of indices zero through one less than the captured count, unless a match is found earlier.
- The visible increment does not overflow for a positive signed 32-bit bound: after the largest permitted index, the increment reaches the bound and traversal stops.

For each visited index, the function resolves and invokes the dispatch-table entry at byte offset `+0x40`, passing the collection pointer and the index.

It then follows two levels of returned data:

1. Read the first word of the returned indexed element as a record pointer.
2. Read the first word of that record as the candidate key.

The function does not visibly validate either pointer.

A candidate matches when its 32-bit key equals the first argument. This is exact equality; there is no range comparison, normalization, or alternate matching rule.

On the first match, traversal stops. The function reads and returns the matching record’s second word. It does not inspect later records, allocate either block, or invoke the insertion-like dependency.

The dispatch-table lookup is performed for each indexed call. The body does not establish that a dependency preserves the dispatch pointer, collection contents, or any global state.

### Miss and record allocation

If traversal completes without a match, the function calls F-8157a180d901 with an allocation size of eight bytes.

If this call returns zero, the function immediately returns zero. It performs no visible allocation of the 36-byte region and does not call F-a028838d7a4b or the insertion-like dependency.

For a nonzero allocation result, the function treats the returned address as a writable two-word record. It writes the supplied key into the first word.

No initialization of the second word occurs before the next dependency calls.

### Associated-value construction

After writing the key, the function calls F-8157a180d901 again, this time requesting 36 bytes. It passes that call’s result and the supplied key to F-a028838d7a4b.

There is no intervening check for a zero result from the 36-byte allocation. Consequently, F-a028838d7a4b is called even when that allocation returns zero.

The 32-bit result of F-a028838d7a4b is written into the record’s second word. The same result controls whether the insertion-like call occurs:

- Zero: skip insertion.
- Nonzero: invoke the insertion-like dependency.

The test is zero versus nonzero, not positive versus negative. A result with its high bit set still takes the insertion-like path.

### Zero associated value

When F-a028838d7a4b returns zero, the function returns the second word of the newly allocated record, which it has just set to zero.

There is no visible insertion call on this path. There is also no visible release of the eight-byte record or the 36-byte allocation, if the latter succeeded. This establishes the absence of explicit cleanup in this function, not the complete ownership behavior of its dependencies.

### Nonzero associated value and insertion-like call

For a nonzero associated value, the function resolves the collection’s dispatch-table entry at byte offset `+0x1c`. It passes:

- The collection pointer.
- The address of the local slot containing the newly allocated record pointer.

The indirect call’s own return value is not used.

After that call returns, the function reloads the record pointer from the local slot and returns the second word of the designated record. It does not recheck the key or require the pointer to remain equal to the original allocation.

Therefore:

- If the dependency preserves the pointer and record contents, the returned word equals the earlier result of F-a028838d7a4b.
- If the dependency changes the second word, the changed word is returned.
- If the dependency replaces the pointer, the replacement record supplies the return value.
- If the dependency leaves a null or otherwise invalid pointer, the subsequent record read is invalid; no protective branch is visible.

Successful insertion, deduplication, ownership transfer, and collection-count changes are not established by the call site alone.

## Side effects

The visible function body can perform these writes:

1. Write the key to the first word of a newly allocated eight-byte record.
2. Write F-a028838d7a4b’s result to its second word.
3. Expose the local record-pointer slot for modification by the insertion-like dependency.

No caller-provided memory is explicitly written through the fourth argument.

The observable dependency order on a miss is:

1. Acquire the collection.
2. Read its count and perform any indexed lookups.
3. Request eight bytes.
4. If successful, write the key.
5. Request 36 bytes.
6. Call F-a028838d7a4b.
7. Write its result into the record.
8. If that result is nonzero, perform the insertion-like call.
9. Read the selected record’s second word for return.

Calls may themselves read or mutate memory, including globals. No preservation guarantee is supplied for any dependency. The list above specifies the ordering visible in this function, not all memory activity of the complete call chain.

No explicit deallocation, rollback, synchronization, logging, or error-reporting call appears. Nothing in the supplied evidence establishes thread safety or safe concurrent mutation of the collection.

## Dependencies

| Dependency | Established call-site contract | Not established |
|---|---|---|
| F-db2e4665a74c | Called without explicit arguments; result is used as a collection pointer. | Nullability, collection lifetime, ownership, global side effects. |
| F-8157a180d901 | Called with sizes eight and 36; results are treated as addresses, with zero checked only for the eight-byte request. | Allocation policy, alignment guarantee, initialization, failure mechanism beyond returned zero, release responsibility. |
| F-a028838d7a4b | Receives the 36-byte allocation result and the key; returns a 32-bit word stored in the record. | Whether it constructs an object, accepts null storage, retains or releases storage, changes globals, or returns an owned resource. |
| Unresolved indirect dependency at dispatch offset `+0x40` | Receives the collection and index; result is dereferenced to obtain a record pointer. | Bounds behavior, element lifetime, mutation, ownership, failure representation. |
| Unresolved indirect dependency at dispatch offset `+0x1c` | Receives the collection and address of a mutable local record-pointer slot. | Whether it inserts, replaces, rejects, frees, transfers ownership, or modifies the collection. |

No opaque function IDs were supplied for the two indirect targets. Their identities remain unresolved rather than being equated with a named dependency.

## Acceptance examples

These examples are conditional acceptance criteria derived from static evidence. They are not executed tests.

1. **First matching record wins.**  
   Given a captured count of three, valid lookup results, and matching keys at indices zero and two, the function returns the second word from index zero’s record. Indices one and two are not requested, and neither allocation call occurs.

2. **An existing zero value is returned unchanged.**  
   Given a matching record whose second word is zero, the result is zero without a construction attempt. This result alone does not distinguish a hit from an allocation or construction failure.

3. **Negative count skips traversal.**  
   Given a count of negative one and an eight-byte allocation that returns zero, the function makes no indexed lookup calls and returns zero after that allocation attempt.

4. **Record-allocation failure stops the miss path.**  
   Given no matching record and a zero result for the eight-byte request, the function returns zero without requesting 36 bytes.

5. **Second allocation failure is forwarded.**  
   Given a successful eight-byte allocation and a zero result for the 36-byte request, F-a028838d7a4b still receives zero as its first argument and the original key as its second. Behavior then depends on whether that dependency returns normally and what it returns.

6. **Zero construction result suppresses insertion.**  
   Given a normal return of zero from F-a028838d7a4b, the record’s second word becomes zero, the insertion-like call is omitted, and the function returns zero. No explicit cleanup occurs in the visible body.

7. **Insertion can determine the final result.**  
   Given a nonzero construction result and an insertion-like dependency that replaces the local pointer with another valid two-word record, the function returns the replacement record’s second word, not necessarily the construction result.

8. **Unused incoming arguments do not direct memory access.**  
   Holding the key and dependency behavior fixed, changing arguments two through four does not alter this body’s visible reads, writes, or dependency arguments. In particular, a null fourth argument is not dereferenced.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the key and associated value.
- Whether the collection is a cache, registry, or another structure.
- The actual identities and full contracts of the indirect targets.
- The ownership and release obligations for either allocated block.
- Whether the 36-byte region is consumed, retained, replaced, or released by F-a028838d7a4b.
- Whether the insertion-like dependency retains or frees the record.
- Whether dependencies mutate globals, collection contents, dispatch entries, or records.
- Behavior under concurrent mutation, reentrancy, invalid pointers, inaccessible memory, or nonreturning dependencies.
- Byte order, signed interpretation of the associated value, and any pointer meaning attached to it.

For defined traversal and return behavior, the relevant collection, dispatch entries, indexed elements, and records must remain valid whenever accessed. A successful allocation must provide enough writable storage for the accesses described above. These are access-validity preconditions, not evidence that the function enforces them.

## Evidence

The contract is based solely on the supplied static representation of F-d2e51f07aabd and the supplied opaque dependency mapping.

Established evidence includes the four-argument interface, four-byte accesses, dispatch offsets, signed traversal comparison, first-match return, allocation sizes, absence of a check on the second allocation, zero/nonzero construction branch, and post-call reload of the mutable record pointer.

The lookup-and-construction interpretation is an inference from those operations. Dependency internals, ownership, synchronization, and broader application meaning are not available.

No runtime tests have been performed.
