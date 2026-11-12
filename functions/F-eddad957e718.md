<!-- Trusted provenance: F-eddad957e718; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eddad957e718

## Purpose

F-eddad957e718 rebuilds receiver state and shared category flags while processing an indexed collection. It also processes objects through temporary reconciliation state and optionally traverses another source to dispatch operations according to the category flags.

“Selection,” “category,” and “reconciliation” describe inferred roles. The established behavior is the ordering of accesses, calls, conditional branches, and updates described below.

The routine has no explicit return value. Some conditions invoke F-a817d8d58c10, represented as non-returning. Earlier effects are not rolled back, and normal cleanup is not visibly performed on those paths.

## Interface

### Inputs and representation

The routine receives:

- A mutable receiver pointer, used without an initial null check.
- A 32-bit optional traversal-source value. Zero skips the additional traversal.
- A 32-bit notification control. Any nonzero value enables a final receiver virtual call.

Pointers and ordinary words have a 32-bit representation. Byte writes affect only the addressed byte. Indexed lookup arguments are narrowed to signed 16 bits and then passed as signed integer values. Thus an iteration index of 32768 is passed as −32768, not as 32768.

### Receiver layout

Offsets are byte offsets.

| Offset | Observed role |
|---|---|
| +0x00 | Virtual-dispatch-table pointer. |
| +0x18 | Outer indexed-collection handle. |
| +0x1c | Current-record pointer, explicitly cleared during initialization. |
| +0x20 | Descriptor destination; its first word is subsequently used for resolution. |
| +0x30 | Byte flag, explicitly cleared during initialization. |
| +0x34 | 32-bit counter, explicitly cleared during initialization. |
| +0x38 | Collection-like storage used in reconciliation. |
| +0x48 | Word read during initial range-related processing. |
| +0x54 | Collection-like storage used for nested entries. |
| +0x70 | Collection-like storage used for outer records. |
| +0x8c | Start pointer of a four-byte-element sequence. |
| +0x90 | Logical end pointer of that sequence. |
| +0x94 | Capacity-end pointer of that sequence. |

Clearing +0x1c does not clear the descriptor at +0x20. Clearing +0x30 does not explicitly clear neighboring bytes.

### Referenced layouts and access requirements

An outer indexed lookup returns a wrapper whose word at +0x08 supplies the record pointer.

| Record offset | Observed role |
|---|---|
| +0x30 | Address passed to F-1515a8d1ace0. |
| +0x60 | Signed 32-bit category index. |
| +0x64 | Byte controlling receiver selection-state updates. |
| +0x68 | Handle used for descriptor production and nested processing. |
| +0x94 | Byte enabling nested processing and required for category 1. |

A nested entry supplies a first word and an address at +0x04. First-word values zero and 0x80 are sentinels.

Reconciliation nodes expose parent, left, and right links at +0x04, +0x08, and +0x0c, with a payload word at +0x10.

An intermediate object in the optional traversal must contain the word 1 at +0x04. Nested optional targets expose a category at +0x4c and a virtual-dispatch-table pointer at +0x00.

Normal execution requires readable or writable storage for every access actually reached, valid virtual dispatch targets, and valid structures for the traversals performed. No general pointer, category-range, or structure validation is visible.

## Behavior

### Initial reconciliation setup

Before clearing derived receiver fields, the routine performs these operations in order:

1. F-9addaaed381d receives temporary reconciliation storage, receiver +0x38, and zero.
2. F-a8c343f922a8 receives receiver +0x38; its result is stored in a temporary word.
3. Receiver +0x48 is read into another temporary word and copied into separate output storage.
4. F-9a305cc01ef7 receives that output storage’s address, receiver +0x38, and the addresses of the two preceding temporary words.
5. F-063eea01efb9 receives receiver +0x54, then receiver +0x70.

These calls expose writable temporary storage. Their collection semantics, write extents, retention, and ownership effects are not established.

Receiver +0x1c is then cleared. F-d0b90560f1fd receives the currently read receiver +0x18 handle and supplies the outer iteration bound. Only after that call is receiver byte +0x30 cleared.

### Sequence reset and category-flag clearing

The sequence end and start pointers are captured. If they differ, a represented element-copy loop performs no iterations: its initial source cursor already equals its comparison endpoint.

An empty traversal then advances a temporary pointer from the captured start in four-byte steps until it equals the receiver’s end pointer. The endpoint is read from the receiver during the comparisons. Normal continuation requires that endpoint to be reachable by this traversal; reaching the later subtraction is not established for malformed endpoints.

After traversal completion, the current end field is reduced by the captured end-minus-start byte difference masked with 0xfffffffc. For an ordinary aligned, valid sequence, this resets the logical end to the start. There is no visible element destruction or storage release. If captured start and end were equal, this block is skipped.

The routine next captures the shared category-table base, clears receiver +0x34, and visits exactly 34 pointer slots in ascending order, numbered 0 through 33. For each nonnull entry, it clears bit 0x02 in the byte at entry +0x04, preserving the other bits.

Later accesses use this captured table base but reload individual slot values. Dependencies are not assumed to preserve the table entries or their flags.

### Outer-record processing

The outer index begins at zero and is compared as signed against the saved bound. A nonpositive bound skips all outer records. The bound is not refreshed.

Each iteration performs the following ordered work:

- F-0ff4e60503e7 receives the receiver’s currently read +0x18 handle and the signed-16-bit-narrowed index. The returned wrapper’s word at +0x08 supplies the record pointer.
- F-1515a8d1ace0 receives a temporary word’s address, receiver +0x70, and record +0x30.
- F-ccabec065f5e receives the record’s +0x68 handle and temporary descriptor storage.
- Record +0x94 determines whether nested processing occurs.
- Record +0x64 determines the selection-state branch.
- The record’s category is used for ordinary category marking.
- A fresh category read determines whether category-1 processing occurs.
- The outer index increments.

The first argument of the outer F-1515a8d1ace0 call is exposed writable storage. It is not established to be an omitted argument, an ownership transfer, or merely a return-value convention.

### Nested-entry processing

If record +0x94 is nonzero, F-73d4385c28ec receives its +0x68 handle and returns a temporary nested handle.

F-d0b90560f1fd is then called with no explicit argument shown at this site. Its result supplies an unsigned 32-bit bound. The actual calling interface remains unresolved.

Nested indices start at zero and advance under an unsigned comparison. F-6e0389afef8f receives the nested handle and each index narrowed to signed 16 bits.

An entry whose first word is zero or 0x80 causes no further per-entry work. For any other first word:

- F-283eda65a14c is called without an explicit argument shown. Zero invokes failure.
- F-682adfd5a561 receives that result and selector 5. Zero invokes failure.
- The selector-5 result is stored in a temporary object word.
- F-a56aefc95716 receives the temporary reconciliation collection and that word’s address.
- F-2b924c211df1 receives temporary result storage, receiver +0x38, and the same object-word address.
- Result words and a result byte are copied locally.
- F-1515a8d1ace0 receives a distinct temporary buffer, receiver +0x54, and nested-entry +0x04.
- F-c7ba4c901230 receives the then-current object-word contents and the receiver.
- Only if that call returns zero, F-8f9bd59e3ec5 receives freshly evaluated object-word contents and the receiver.

The object word need not retain the selector-5 result. Either address-taking reconciliation call may replace it. Because its address has escaped, subsequent dependencies also cannot be assumed to preserve it. In particular, the conditional fallback need not receive the same object value as the preceding predicate-like call.

After normal nested-loop exhaustion, F-5d38327d98c6 receives the nested handle’s address. Failure paths do not visibly reach that call.

### Selection state and index sequence

Record byte +0x64 is read after nested processing.

If it is zero, receiver byte +0x30 is set to 1. This branch performs neither the sequence operation nor the counter increment below.

If it is nonzero:

- When the temporary descriptor’s first word is neither zero nor 0x80, receiver +0x1c is set to the record pointer. F-ccabec065f5e then receives the current record +0x68 handle and receiver +0x20 as destination.
- The current sequence end is read, and the outer index is copied into temporary addressable storage.
- If end equals capacity end, F-8281dcced3b1 receives receiver +0x8c, that end pointer, and the temporary index word’s address.
- Otherwise, receiver +0x90 is advanced by four bytes before the element store. If the old end is nonnull, the index is stored there. If it is null, the store is skipped despite the end update.
- Receiver +0x34 is read and incremented.

The capacity-equality branch establishes delegation only. It does not establish insertion, allocation, or sequence-pointer updates by F-8281dcced3b1.

A sentinel descriptor suppresses replacement of the current-record pointer, not sequence handling or the counter increment. An exact cumulative count additionally requires dependencies not to modify the counter.

### Category marking and category 1

The signed category at record +0x60 selects a pointer from the captured category table using four-byte slot spacing. There is no visible bounds check. If the selected pointer is nonnull, bit 0x02 is set in its byte at +0x04.

The category is read again. If it equals 1, record +0x94 must be nonzero and receiver +0x1c must equal this record pointer. Either failed requirement invokes failure after preceding record effects.

When both hold, F-283eda65a14c receives the first word at receiver +0x20. Its result is passed directly to F-682adfd5a561 with selector 6, without a separate zero check on the resolution result. A zero selector-6 result invokes failure.

F-5710a8628cba receives the selected result and an opaque shared selector value:

- A zero result causes bit 0x02 to be set in all nonnull entries among the 34 table slots.
- A nonzero result is treated as a sequence of 32-bit category indices terminated by zero. Every nonzero index selects a table entry whose byte at +0x04 receives bit 0x02.

The list branch checks neither category bounds nor selected-entry nullness. Zero terminates the list and therefore cannot designate category zero.

### Leftover reconciliation traversal: live end marker

After outer processing, F-a8c343f922a8 receives the temporary reconciliation collection and supplies the initial traversal position.

**The termination boundary is not captured once.** Before every termination comparison, the temporary collection’s current end-marker word is read and copied into a shared temporary output word. The traversal position is then compared with the end-marker word. That same shared temporary word is subsequently used as the optional outer iterator’s output storage.

Consequently:

- The marker is read before the first termination decision, including an immediately empty traversal.
- Following a callback and successor calculation, the marker is read again before deciding whether to process another node.
- The copied value is not used as a stable boundary for the whole traversal.
- At the comparison that ends traversal, the shared output word has just been assigned the current marker value. Later dependencies may change it before or during optional iteration.

The temporary collection’s address has already escaped to dependencies. F-3e5212464798 may therefore indirectly change its marker even though the callback’s explicit arguments do not include that collection. The next termination comparison uses the marker’s current contents, not its value before the callback.

For a position not equal to the current marker, F-3e5212464798 receives the node’s payload at +0x10, the receiver, and zero.

Successor links are read after the callback. A right subtree, when present, is followed leftward; otherwise parent links are followed using the represented structural successor logic. Only after obtaining the next comparison position does the routine perform the next marker copy and termination check. There is no visible iterator repair, boundary snapshot, or restart if callbacks alter links or the marker.

This establishes structural traversal behavior, not an application-level sort order.

### Optional additional traversal

For a nonzero traversal-source input, F-583dc5fa4d66 transforms the source and F-8ca21d3a8cca initializes an outer iterator.

Each nonzero result from F-b38a3aba0e18 supplies an intermediate value through the shared output word previously used for marker copies. F-583dc5fa4d66 resolves that value. The resulting object’s word at +0x04 must equal 1; otherwise failure is invoked.

A separate output word is initialized to zero. The intermediate object’s virtual slot +0x3c receives the object, zero, and that word’s address. F-583dc5fa4d66 transforms the resulting output, and F-8ca21d3a8cca initializes a nested iterator.

Each successful nested advance writes an object pointer through another output location. The current pointer’s category at +0x4c selects a category-table entry. A null entry suppresses all flag-dependent virtual calls.

For a nonnull entry, the routine captures whether bit 0x02 is set:

- If clear, virtual slot +0xd8 is invoked with no explicit argument shown, then +0xc0 receives the then-current output pointer and zero.
- If set, +0xd8 is skipped and +0xc0 receives the then-current output pointer and 1.
- After +0xc0, the output pointer is reloaded to obtain slot +0xa0. That slot receives the freshly evaluated output pointer and the captured activity value: zero for the clear-bit branch, 1 for the set-bit branch.

The iterator output address has escaped. Intervening calls may replace the stored pointer, so these dispatches need not target one stable object. The activity decision is reused rather than recomputed.

F-d0d3f100effc receives the nested iterator after exhaustion and the outer iterator after its exhaustion.

### Normal completion

If notification control is nonzero, receiver virtual slot +0x58 receives the receiver and an opaque shared notification value.

F-feb008c54f7c then receives the temporary reconciliation collection. Notification therefore precedes this final collection operation.

## Side effects

Direct writes affect receiver fields, sequence storage on the spare-capacity branch, and category-entry flag bytes. Multiple dependencies receive addresses of temporary words, buffers, handles, iterator storage, or collection storage.

The end-marker copy also writes the shared temporary word later exposed as optional-iterator output storage. These are successive uses of the same storage, not independent snapshots.

No direct allocation or deallocation primitive is visible. Dependency names and call placement do not establish ownership transfer, address retention, reference counting, destruction, or release. Unknown callees may modify shared state or previously exposed storage.

Failures can leave partially updated receiver state, flags, collections, and dependency effects. There is no visible rollback.

## Dependencies

These opaque IDs identify observed calls, not independently established implementations.

| Dependency | Observed use |
|---|---|
| F-9addaaed381d, F-a8c343f922a8, F-9a305cc01ef7 | Collection setup and position/range-related operations. |
| F-063eea01efb9 | Operations on receiver +0x54 and +0x70. |
| F-d0b90560f1fd | Signed outer bound and unsigned nested bound; nested interface unresolved. |
| F-0ff4e60503e7, F-6e0389afef8f | Indexed lookup with signed-16-bit-narrowed indices. |
| F-1515a8d1ace0 | Three-argument operations exposing temporary storage. |
| F-ccabec065f5e, F-73d4385c28ec | Descriptor production and nested-handle acquisition. |
| F-283eda65a14c, F-682adfd5a561 | Resolution and selector-5/selector-6 operations. |
| F-a56aefc95716, F-2b924c211df1 | Reconciliation operations sharing mutable object-word storage. |
| F-c7ba4c901230, F-8f9bd59e3ec5 | Predicate-like operation and conditional fallback. |
| F-5d38327d98c6 | Operation on the nested handle’s address. |
| F-8281dcced3b1 | Capacity-equality delegation. |
| F-5710a8628cba | Zero or a zero-terminated category-index sequence. |
| F-3e5212464798 | Leftover-node callback, preceding successor-link reads and marker reload. |
| F-583dc5fa4d66, F-8ca21d3a8cca, F-b38a3aba0e18, F-d0d3f100effc | Optional resolution and iterator lifecycle. |
| F-a817d8d58c10 | Represented non-returning failure. |
| F-feb008c54f7c | Final operation on temporary reconciliation storage. |

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Empty outer collection.** A zero bound skips outer records, but initialization, valid sequence reset, flag clearing, and leftover traversal still occur. An immediately empty leftover traversal still copies its current end marker before testing equality.

2. **Unselected ordinary record.** With category other than 1 and byte +0x64 zero, receiver +0x30 becomes 1. This record causes no sequence operation or counter increment; ordinary category marking still occurs.

3. **Selected sentinel descriptor.** With byte +0x64 nonzero and descriptor first word 0x80, the descriptor does not replace receiver +0x1c. A valid nonnull spare-capacity end receives the index after the end field advances four bytes. At capacity equality, only the specified delegation is established. Either branch proceeds to counter increment if calls return.

4. **Unreachable sequence endpoint.** If four-byte advancement cannot reach the endpoint, completed reset is not established merely because a subtraction appears later.

5. **Mutable nested result.** If F-a56aefc95716 replaces the object word, F-2b924c211df1 receives that same storage address with the replacement present. Later calls use subsequent contents rather than a guaranteed original selector-5 result.

6. **Nested failure.** A non-sentinel entry followed by zero from F-283eda65a14c invokes failure before nested-handle finalization.

7. **Category-1 expansion.** With required receiver state and successful resolution, zero from F-5710a8628cba marks all 34 nonnull entries. A nonzero result marks listed indices until zero, without per-entry null protection.

8. **Changed reconciliation boundary.** Suppose a leftover callback changes the collection’s marker and the structure still permits a valid successor calculation. Before the next termination decision, the new marker is copied into the shared output word and used as the comparison boundary. Equality with the old marker alone does not establish termination.

9. **Optional inactive target.** If output storage remains stable, a clear flag produces +0xd8, +0xc0 with zero, then +0xa0 with zero. If an intervening call replaces the pointer, subsequent dispatches use reloaded storage while retaining the original flag decision.

## Uncertainties

The following remain **documented_only**:

- Application meanings, complete layouts, dependency contracts, and actual calling conventions where explicit arguments are absent.
- Dependency write extents, retained addresses, ownership, lifetime, and global effects.
- Effects of capacity-equality delegation.
- Validity of successor traversal after callback-driven structural or marker changes. Reloading the marker does not itself make mutation safe.
- Behavior for invalid pointers, unreachable endpoints, malformed trees, out-of-range categories, or unterminated category lists.
- Overflow and pointer-arithmetic behavior outside valid representable ranges. No explicit overflow checks are visible; ordinary machine-width expressions do not establish portable behavior for invalid addresses.
- The actual failure mechanism and any cleanup beyond the represented control flow.

## Evidence

Established evidence is limited to the supplied static representation: widths, offsets, explicit arguments, escaped addresses, reloads, branches, and ordering.

In particular, the leftover traversal places an end-marker-to-output-word assignment immediately before each termination comparison. Callback execution and successor-link reads precede the next such assignment. The shared output word is later passed to the optional outer iterator as output storage. These observations establish a live, repeatedly read boundary—not a saved traversal-wide marker.

Semantic labels are inferred intent. Dependency implementations and runtime behavior remain unknown. No runtime tests have been performed.
