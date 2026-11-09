<!-- Trusted provenance: F-b8f47c5be3a1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b8f47c5be3a1

## Purpose

F-b8f47c5be3a1 coordinates a cached mode byte, three object callbacks, and a conditional byte update through a shared registry.

On a normally continuing execution, it:

1. Queries an initial mode and conditionally updates the object’s cache.
2. Performs additional calls governed by that mode and a second object byte.
3. Issues three ordered object callbacks.
4. Queries a status independently and derives a zero-or-one target.
5. Traverses and validates a registry selection.
6. Conditionally writes a payload byte and issues two notifications.

The return value distinguishes the outcomes of the final selection check. It does not report whether either byte changed, whether earlier calls succeeded, or whether notifications were delivered successfully.

“Mode,” “registry,” “payload,” and “notification” are descriptive labels. Application-level intent, such as visibility or interface synchronization, is not established.

## Interface

### Input and accessed layouts

The function accepts one pointer to an object. The supplied representation uses 32-bit pointers and words, with separately accessed one-byte fields.

| Object location | Established access |
|---|---|
| `+0x00` | Pointer to a virtual dispatch table. |
| `+0xBC` | One-byte field, read only when the initial mode is exactly one. |
| `+0xC6` | One-byte cached mode, read and conditionally written. |
| Dispatch-table `+0x58` | Entry used for three object callbacks. |

The service objects returned by F-b7af80e829ec have a dispatch-table pointer at their start. Entries at table offsets `0x108` and `0x120` are invoked.

Registry and payload accesses use these layouts:

| Location | Established access |
|---|---|
| Registry base `+0x10` | Header pointer. |
| Registry base `+0x19` | Address supplied as comparison context. |
| Header `+0x04` | Initial traversal-node pointer. |
| Traversal node `+0x08` | Branch pointer used after a zero comparison result. |
| Traversal node `+0x0C` | Branch pointer used after a nonzero comparison result. |
| Traversal node `+0x10` | Address supplied as a comparison operand. |
| Selected node `+0x14` | Payload pointer. |
| Payload `+0x64` | One-byte field compared with and conditionally set to the target. |

These are partial access layouts, not complete type definitions.

The caller must supply storage and dispatch entries valid for every executed access. Dependencies must return usable objects whenever subsequent operations dereference their results. No visible null checks, bounds checks, or topology validation establish those conditions.

### Output

The normal return is a 32-bit word:

- **One:** the final F-f063d8ade9f7 call returns zero, and execution reaches the end of the selected-entry branch.
- **Zero:** that final call returns nonzero.

A return of one does not require a payload change. A return of zero does not undo earlier effects.

### Opaque global words

Besides the registry-pointer global, the function reads several opaque 32-bit global words:

- Construction descriptor.
- Common callback word.
- First, second, and third callback selectors.
- Registry key.
- First and second notification selectors.

These are global words, **not established immutable constants**. Their contents, encodings, and meanings are unknown. Their sampling boundaries are part of the behavior below; equal values across separate reads must not be assumed.

## Behavior

### 1. Initial mode and cache update

The function acquires a service through F-b7af80e829ec and invokes its dispatch entry at `0x108`. The result is treated as an unsigned 32-bit initial mode.

It reads the object byte at `+0xC6`, zero-extends it, and compares it with the entire mode word.

When they differ:

1. The mode’s low eight bits are written to object `+0xC6`.
2. F-b7af80e829ec is called again.
3. Dispatch entry `0x120` on that returned service is invoked.
4. If its 32-bit result is nonzero, the following sequence executes:
   - F-483a717208e6 receives the word value eight.
   - **After that call returns**, the construction-descriptor global is read.
   - F-17c2f3b940d8 receives the preceding result and that sampled descriptor.
   - F-8e50f265e4fa receives the input object, the result from F-17c2f3b940d8, and zero.

The descriptor is therefore not a snapshot taken before F-483a717208e6. Changes to that global during F-483a717208e6 can affect the argument subsequently supplied.

The cache write precedes the additional service acquisition and conditional calls. No rollback is visible.

This is a full-width comparison followed by a truncated write. For example, mode `0x00000101` differs from cached byte `0x01`, but the write still stores `0x01`.

The result from F-8e50f265e4fa does not control any subsequent branch in the supplied function.

### 2. Initial-mode-one action

If the original mode is exactly one, the function reads object byte `+0xBC`. If that byte is zero, it calls F-a886da7a3894 with the input object.

This read occurs after the entire cache-mismatch sequence, including any construction-related calls. It observes the field at that later point, not a value saved before those calls.

For every initial mode other than one, the function neither reads this field nor calls F-a886da7a3894 through this condition.

### 3. Three object callbacks and global sampling

The function next performs three calls through the input object’s dispatch entry at `0x58`.

Before the first call, it reads and saves the common-callback global word. The first callback nevertheless uses a **separate read** of that global, rather than the saved word:

| Call | Explicit arguments after the object |
|---|---|
| First | Separately read common-callback word; freshly read first selector. |
| Second | Saved common-callback word; freshly read second selector. |
| Third | Saved common-callback word; freshly read third selector. |

The first common-word read and the first call’s separate read are distinct observations. There is no intervening explicit dependency call between them, but their equality is not an unconditional property of the supplied evidence.

The second selector is sampled after the first callback returns. The third selector is sampled after the second callback returns. Changes made by earlier callbacks can therefore affect later selectors, while callbacks two and three continue to receive the saved common word.

The object’s dispatch table and its `0x58` entry are obtained again for each invocation. A callback is not assumed to preserve the dispatch table.

All three callback results are ignored. Subject to normal continuation, these calls occur regardless of whether the cache changed or the mode-one action ran. The source does not establish a total ordering among individual argument and dispatch-entry reads within a single call expression.

### 4. Independent status query and ordered registry initialization

After the third callback, the function again acquires a service through F-b7af80e829ec and invokes entry `0x120`.

This is independent of the conditional query in the cache-update phase. Neither service identity nor query result is assumed stable across acquisitions.

The following initialization order is established:

1. Capture the current registry-pointer global as a base retained for later comparison-context and header accesses.
2. Compute the target from the final unsigned 32-bit query result.
3. Read the registry-key global into the first word of an addressable search record.
4. Independently reread the registry-pointer global, read that registry’s header at `+0x10`, and read the initial traversal pointer from that header at `+0x04`.
5. Independently reread the registry-pointer global again and read that registry’s header at `+0x10` as the initial candidate.
6. Store the target in the search record’s second word.

The target calculation uses unsigned 32-bit arithmetic: one minus the query result, modulo \(2^{32}\), followed by replacement with zero when the result exceeds one. Its final value is therefore:

- One for query result zero.
- Zero for every nonzero query result.

The initialized search record has two 32-bit words:

| Record offset | Initial content |
|---|---|
| `+0x00` | Sampled registry key. |
| `+0x04` | Target, zero or one. |

The retained registry base, initial traversal node, and initial candidate arise from **three separate registry-global reads**. They need not refer to the same registry. Treating them as one coherent snapshot requires a stability assumption absent from the supplied evidence.

After initialization, comparisons and later header reads use the retained base; they do not reacquire the registry global.

### 5. Registry traversal

While the traversal pointer is nonzero, F-bcefa5957e76 receives:

- The retained registry base plus `0x19`.
- The current traversal node plus `0x10`.
- The address of the two-word search record.

Its zero/nonzero result determines the next access:

- **Zero:** read the current node’s pointer at `+0x08`; use that newly read pointer as both the next traversal node and the candidate.
- **Nonzero:** read the current node’s pointer at `+0x0C`; use it as the next traversal node, leaving the candidate unchanged.

The candidate on the zero branch becomes the **descended-to pointer**, not the node just compared. If that branch pointer is zero, the candidate also becomes zero.

Each branch pointer is read after the comparison returns. The comparison may have changed node storage or the search record; preservation is not supplied as a dependency contract.

There is no visible iteration limit, cycle detection, or recovery from malformed links.

### 6. Candidate validation and final selection check

When traversal ends, the function reads the header at retained-base `+0x10`. It places the candidate and that header in separate addressable one-word storage and passes their addresses to F-f063d8ade9f7.

If this check returns zero:

1. F-539f08165386 receives the candidate word then present in its addressable storage.
2. F-bcefa5957e76 receives the retained comparison context, the search-record address, and the result from F-539f08165386.
3. If this comparison returns zero, the candidate storage supplies the selected value.

If either the first check or the validation comparison returns nonzero, a fresh read of retained-base `+0x10` supplies the selected value instead.

The selected value is copied into another addressable word. The function reads retained-base `+0x10` again into separate addressable storage and calls F-f063d8ade9f7 with both addresses.

- A nonzero result returns zero.
- A zero result proceeds to payload access.

Both checks can expose writable local words to their dependency. The payload branch uses the selected word present **after** the final check, not an assumed preserved copy from before it.

An iterator-versus-sentinel interpretation is plausible but unproven. The contract depends on the observed zero/nonzero results, not assumed pointer equality or ordinary container semantics.

### 7. Payload update and notifications

In the selected-entry branch, the function reads the payload pointer at selected-node `+0x14`, then reads the byte at payload `+0x64`. It compares that byte, zero-extended, against the independently retained target.

If equal, it returns one without an explicit payload write or notification-receiver acquisition.

If different:

1. The payload pointer at selected-node `+0x14` is loaded again.
2. The target’s low eight bits are written at that payload’s `+0x64`.
3. F-9d0ecc5f6adf acquires a receiver.
4. The receiver’s dispatch entry at `0x58` is invoked with the receiver, the first notification-selector global word, and the search record’s first word.
5. After that call returns, the receiver’s dispatch entry at `0x58` is obtained again and invoked with the receiver, the second notification-selector global word, and the search record’s first word.
6. The function returns one.

The selector globals are sampled for their respective calls. In particular, the second selector is read after the first notification returns. The search record’s first word is also read separately for each notification; it is not guaranteed to remain the original registry key.

The target is not reloaded from the search record’s second word. Mutation of that record does not by itself redefine the target used for the payload comparison and write.

The write precedes receiver acquisition and both notifications. Notification results are ignored. Equality of the payload pointers used for comparison and writing requires stability across those separate loads.

## Side effects

The function explicitly performs at most two data-byte writes on a normal invocation:

- A conditional cache write at input-object `+0xC6`.
- A conditional payload write at payload `+0x64`.

It also initializes and copies local search and selection storage. Addresses of some of that storage are supplied to dependencies.

Dependencies and indirect calls may have additional effects. No supplied contract establishes preservation of global words, dispatch tables, object fields, registry structure, or addressable arguments.

There is no explicit release of values returned by F-483a717208e6 or F-17c2f3b940d8. This establishes neither a leak nor ownership transfer. Allocation, borrowing, retention, and release obligations remain unknown.

No visible exception handler, error translation, rollback, or general failure-status mechanism is present.

## Dependencies

| Dependency | Established call-site role |
|---|---|
| F-b7af80e829ec | Acquires services; called twice on a normally completing path, or three times when the initial cache comparison differs. |
| Service entry `0x108` | Supplies the initial 32-bit mode. |
| Service entry `0x120` | Supplies the optional construction condition and, independently, the final status. |
| F-483a717208e6 | Receives eight; result feeds F-17c2f3b940d8. |
| F-17c2f3b940d8 | Receives the preceding result and the descriptor sampled afterward. |
| F-8e50f265e4fa | Receives the object, preceding result, and zero. |
| F-a886da7a3894 | Receives the object under the mode-one, byte-zero condition. |
| Object entry `0x58` | Receives three ordered callbacks with the sampling behavior specified above. |
| F-bcefa5957e76 | Controls traversal and candidate validation through zero/nonzero results. |
| F-f063d8ade9f7 | Checks addressable selection/header words. |
| F-539f08165386 | Supplies the candidate-associated operand for validation. |
| F-9d0ecc5f6adf | Acquires the notification receiver after a payload write. |
| Receiver entry `0x58` | Receives two ordered notifications. |

Indirect implementations have no supplied opaque identities. Dispatch locations identify call sites, not fixed callees.

## Acceptance examples

These are conditional static acceptance cases, not runtime results. They assume valid executed accesses and returning dependencies.

1. **Matching cache:** Initial mode seven and cached byte seven skip the cache write and conditional construction query. All three callbacks and the independent final query still occur.

2. **Full-width mismatch:** Initial mode `0x00000101` and cached byte one take the mismatch path and write one. A zero conditional status skips the construction sequence.

3. **Descriptor changes during acquisition:** If F-483a717208e6 changes the descriptor global from one opaque word to another, F-17c2f3b940d8 receives the later word.

4. **Mode-one field changes:** If an intervening construction dependency changes object `+0xBC` from zero to five, the later mode-one read observes five and skips F-a886da7a3894.

5. **Saved versus fresh callback arguments:** Suppose the saved common word is A, the first callback’s separate read yields B, and that callback changes the common global to C and the second selector to D. The first call receives B; the second receives A and D; the third also receives A.

6. **Distinct registry samples:** If the first registry-global read yields registry A, the second yields B, and the third yields C, comparison context and later header reads use A, initial traversal comes through B’s header, and initial candidate is C’s header. Collapsing these into one registry is not equivalent without a stability condition.

7. **Target normalization:** Final status zero produces target one. Status one, two, or `0xFFFFFFFF` produces target zero.

8. **Zero descended-to candidate:** A zero traversal comparison followed by a zero `+0x08` pointer sets both traversal and candidate to zero. Subsequent checks receive that candidate; safe rejection is not guaranteed by this function.

9. **Matching selected payload:** If the final check returns zero and the accessed payload byte equals the target, the function returns one without acquiring a receiver.

10. **Changed notification inputs:** If the first notification changes the second selector, the second notification samples the changed selector. If the search record’s first word changes before the second call, that call samples its later value.

11. **Final rejection:** A nonzero final check returns zero without payload processing. Earlier cache writes and callbacks remain observable.

## Uncertainties

The following are **documented_only**:

- Domain meanings and contents of opaque global words.
- Exact calling conventions and implicit receiver handling for service queries.
- Whether eight denotes an allocation size.
- Ownership, lifetimes, retention, and release obligations.
- Dependency failure, exception, and non-return behavior.
- Comparator ordering and selection-check semantics.
- Whether dependencies mutate or retain exposed storage.
- Synchronization, concurrency, reentrancy, and global stability guarantees.
- Registry invariants and the intended meaning of the unusual candidate update.
- Behavior following invalid pointers, arithmetic-derived invalid addresses, malformed topology, or cycles.
- Fine-grained evaluation order within individual call expressions where the representation does not specify it.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. The prior draft and feedback do not independently establish behavior.

Established evidence includes access widths and offsets, full-width comparisons, byte truncation, unsigned target arithmetic, separate global-read sites, ordering across calls, candidate assignments, exposed local storage, conditional writes, and normal return values.

Semantic labels and possible container interpretations are organizational or inferred intent, not dependency guarantees.

No implementation access, dependency-body inspection, runtime execution, or runtime tests have been performed.
