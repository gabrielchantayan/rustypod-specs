<!-- Trusted provenance: F-b79dfc7a36d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b79dfc7a36d1

## Purpose

F-b79dfc7a36d1 initializes an input-associated object and runs a synchronous record-processing loop. Each iteration obtains a record, optionally dispatches it through a sequence of callbacks, optionally invokes a completion dependency, and checks whether the current record key matches a global termination value.

The function returns only after an iteration reaches that final comparison and the values match. A termination record is not skipped: it passes through the same dispatch and conditional-completion stages as any other record.

The structure suggests an event or message-processing loop. That interpretation is inferred intent, not an established description of the record format, callback registry, or completion operation.

## Interface

### Input and return

The function accepts one integer-valued argument used as the base address of a mutable object.

No return value is exposed. Dependency results are used only where described below; the function does not return a success indicator, error code, callback result, or final record.

The supplied representation uses 32-bit integer and four-byte scalar accesses. Address calculations add byte offsets to the input or to addresses returned by dependencies. Valid execution requires these calculations to identify usable memory. The evidence does not define behavior for invalid addresses, address wraparound, inaccessible storage, or misalignment on the target platform.

### Object and context layout

Only the following locations have established roles:

| Storage | Offset | Access and significance |
|---|---:|---|
| Input object | 0x54 | Its address is passed to the initialization dependency. This function does not directly read or write that location. |
| Input object | 0x88 | Receives one four-byte value read from the first context returned by the context-provider dependency. |
| First returned context | 0x1c | Supplies the four-byte value copied into the input object. |
| Second returned context | 0x1c | Supplies a separate four-byte value captured for repeated record acquisition. |
| Second returned context | 0x20 | Supplies a 32-bit dispatch-context value, reread after every record acquisition. Zero disables dispatch for that iteration. |

The first and second returned contexts need not be identical. Likewise, their values at offset 0x1c need not match. The function does not compare them.

The complete object sizes and field meanings are unknown. The offsets establish only the accesses listed above.

### Temporary storage

The function provides dependencies with invocation-local storage:

- A record buffer containing eight 32-bit integer slots, totaling 32 bytes.
- A 32-bit completion-condition slot.
- A callback-pointer output slot.
- A four-byte callback-argument output slot.
- A separate four-byte auxiliary output area.

The first record slot is the dispatch key and later the termination-comparison value. No interpretation of the other seven slots is established locally.

The function does not explicitly initialize these temporary areas before their first use. In particular, the record-acquisition dependency must establish the values subsequently consumed as the record key and completion condition. A successful dispatch lookup must establish a callable callback pointer and its argument.

## Behavior

### Initialization and context capture

The function first invokes F-861a66c91b48 with four arguments, in this order:

1. The input object’s base address.
2. The address at byte offset 0x54 within that object.
3. Zero.
4. Zero.

Its result is not used. Its internal behavior and any effects on the object or global state are not established here.

After that call returns, the function invokes F-3fdd14ecd250. It reads four bytes at offset 0x1c from the returned address and writes those bytes to offset 0x88 of the input object.

It then invokes F-3fdd14ecd250 a second time. This second returned address is retained for the processing loop. The function reads the four-byte value at offset 0x1c from this second context and captures it for record acquisition.

The ordering is significant:

- The initialization call occurs before either context-provider call.
- The object-field write occurs after the first context-provider call and before the second.
- The acquisition value comes from the second returned context, not from the object field just written.
- The acquisition value is captured once; this function does not refresh it from the context on later iterations.

Mutations performed by the initialization dependency or either context-provider call can therefore affect subsequent reads. There is no established guarantee that the two context-provider calls observe the same state.

### Record acquisition

The loop is entered unconditionally. There is no pre-acquisition termination check.

At the start of every iteration, F-d8deec1d50bf receives:

1. The captured four-byte acquisition value.
2. The address of the 32-byte record buffer.
3. The address of the completion-condition slot.

Its direct return value is not used. The function proceeds using the supplied storage.

The same record buffer and completion-condition slot are reused across iterations without an explicit local reset. The function does not require or verify that all eight record slots are overwritten. Any later consumer that depends on additional record contents relies on the acquisition dependency’s contract.

After acquisition returns, the function reads the 32-bit value at offset 0x20 from the retained second context. This read occurs on every iteration, after acquisition. The address of that context remains the one captured during setup.

### Conditional callback dispatch

If the newly read dispatch-context value is zero, the function performs no dispatch lookup and invokes no callback for that iteration.

If it is nonzero, the function invokes F-379469ac0462 with:

1. The dispatch-context value.
2. The record’s current first 32-bit slot.
3. The address of the callback-pointer output slot.
4. The address of the four-byte callback-argument output slot.
5. The address of the four-byte auxiliary output area.

A zero result ends dispatch immediately. Callback outputs are not consumed on that path.

A nonzero result causes an indirect callback invocation. The callback receives the address of the entire record buffer and the four-byte callback argument produced by the lookup.

The callback’s result controls continuation:

- A nonzero result ends dispatch for this iteration.
- A zero result causes a call to F-50fc78800344.

F-50fc78800344 receives the same kinds of arguments as the initial lookup: the captured dispatch-context value, the record’s current first slot, and the three output-storage addresses. Its zero result ends dispatch. Its nonzero result causes another callback invocation using the latest callback pointer and argument.

This process continues until either a lookup returns zero or a callback returns nonzero.

Several distinctions are externally relevant:

- The dispatch-context value is captured once per iteration. It is not reread between callbacks.
- The record key is read again for each lookup. A callback that changes the first record slot can change the key supplied to the continuation lookup.
- Callbacks receive writable record storage. No local copy protects the original record.
- The function passes the same auxiliary output area to the initial and continuation lookup dependencies, but does not itself inspect its contents.
- No local iteration limit or cycle detection bounds callback dispatch.

Interpreting nonzero callback results as “handled” or zero results as “not handled” is plausible intent. The established behavior is only that those results stop or continue dispatch, respectively.

### Conditional completion and termination

After dispatch finishes, or immediately after dispatch is skipped, the function reads the completion-condition slot.

If it is nonzero, F-41d0ae4663e8 is called with no explicit arguments. Its return value is not used.

If the slot is zero, that dependency is not called.

Only afterward does the function compare the record’s current first slot with the current value of a global termination scalar. Equality ends the loop and returns from the function. Inequality begins another acquisition using the same captured acquisition value.

The termination decision is therefore based on the record contents at the end of the iteration, not necessarily the contents originally produced by acquisition. It also occurs after the conditional completion call; the function does not snapshot the global termination value during setup.

A callback result cannot directly return from F-b79dfc7a36d1. It can only stop the callback sequence. The outer loop still reaches conditional completion and the termination comparison.

## Side effects

### Direct effects

The established direct object write is the four-byte assignment at input-object offset 0x88 during setup.

The function also writes and reuses its invocation-local storage through dependency calls. It does not explicitly allocate or free heap memory, transfer an object to a caller, or return a pointer.

### Dependency-mediated effects

All invoked dependencies and callbacks may have effects beyond the visible arguments. Their effects on global state, object storage, registries, memory ownership, or execution flow are not established by this function.

In particular:

- Initialization may modify the supplied object or storage reachable from it.
- Acquisition may modify state that influences the subsequent dispatch-context read.
- Callbacks may modify the record, including its dispatch key.
- The completion dependency may modify state before the termination comparison.
- Context storage or global state may change during a dependency call.

No assumption that a dependency preserves globals or keeps a returned context alive is justified by the local evidence.

### Ownership and lifetime

The function uses context addresses and four-byte values without locally releasing them. That does not establish whether they are borrowed, reference-counted, owned elsewhere, or released indirectly.

The conditional completion call cannot be characterized as freeing a record or releasing a resource from this evidence alone: it receives no explicit record pointer, and its implementation is unavailable.

Addresses of invocation-local storage are passed to dependencies and callbacks. Their validity as local storage ends when this invocation returns. Whether callees retain those addresses is unknown; safe delayed use is not established.

### Failure and nontermination

The function contains no explicit validation or recovery for:

- An invalid input-object address.
- An unusable context-provider result.
- A nonzero lookup result paired with an invalid callback pointer.
- An acquisition that fails to establish required output values.
- A dependency or callback that does not return.

There is no explicit error-reporting channel. Zero lookup results and zero callback results are dispatch-control outcomes, not established errors.

The function may fail to return if record acquisition blocks, a dependency fails to return, callback continuation never ends, or no completed iteration matches the termination value.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-861a66c91b48 | Called once before context acquisition, with the input base, the address at offset 0x54, and two zeros. Result unused. |
| F-3fdd14ecd250 | Called twice during setup. Each result is used as a context address. |
| F-d8deec1d50bf | Called once per outer-loop iteration with the captured acquisition value and addresses of the record and completion-condition storage. Result unused. |
| F-379469ac0462 | Initial dispatch lookup for a nonzero dispatch context. Nonzero authorizes consuming callback outputs; zero ends dispatch. |
| F-50fc78800344 | Continuation lookup after a callback returns zero. Uses the current record key and the iteration’s captured dispatch context. |
| F-41d0ae4663e8 | Called once per iteration whose completion condition is nonzero, after dispatch and before termination comparison. Result unused. |

Indirect callbacks are additional dependencies without fixed opaque identities in the supplied evidence. Each receives the record-buffer address and one four-byte argument, and returns a value tested against zero.

## Acceptance examples

These examples are conditional requirements derived from the visible control flow. They are not runtime test results.

### Separate context snapshots

Suppose the first context-provider call returns a valid context whose offset-0x1c value is A, and the second returns a valid context whose corresponding value is B.

The input object’s offset-0x88 field must receive A. Record acquisition must receive B on every iteration, even when A and B differ.

### Termination with dispatch disabled

Suppose acquisition produces a record whose first slot equals the termination value, sets the completion condition to zero, and leaves the second context’s dispatch field at zero. Assume relevant state remains unchanged afterward.

The function must perform no lookup or callback, skip the completion dependency, and return after that iteration’s termination comparison.

### Termination still permits dispatch

Suppose acquisition produces a termination-key record and the dispatch field is nonzero.

The function must perform the initial lookup before deciding to return. If that lookup supplies a callback, the callback must be invoked according to the normal dispatch rules. A nonzero completion condition must also cause the completion dependency to run before termination is checked.

### Callback stops dispatch, not the outer loop

Suppose the initial lookup returns nonzero, its callback returns nonzero, and the final record key differs from the termination value.

No continuation lookup occurs. Conditional completion still occurs, and the function starts another acquisition.

### Callback mutation affects continuation

Suppose the initial callback receives a record with key A, changes its first slot to B, and returns zero.

The continuation lookup must receive B, not the original key A. It must still receive the dispatch-context value captured earlier in that iteration.

### Callback mutation affects termination

Suppose acquisition produces a nontermination key, but a callback changes the first slot to the termination value. If that value remains in place through the final comparison and the global termination value remains equal, the function returns after conditional completion.

Conversely, changing an acquired termination key to a nonmatching value prevents return on that iteration.

### Dispatch context changes between iterations

Suppose acquisition changes the retained context’s offset-0x20 field from zero to a nonzero value.

The immediately following dispatch decision must observe the new value. A change to that field during a callback does not replace the already captured dispatch context for later lookups in the same iteration.

## Uncertainties

The following are **documented_only** unknowns:

- The semantic identities of the input object, contexts, records, and termination value.
- Exact target ABI details, including callback calling convention and pointer representation beyond the supplied representation.
- The meaning and valid range of the acquisition value and callback argument.
- Which record slots acquisition writes and which callbacks consume.
- Whether the auxiliary four-byte area is output-only or carries continuation state between lookup calls.
- Memory ownership, retention, and release rules of every dependency.
- Whether the completion dependency releases resources, acknowledges work, changes scheduling state, or performs another operation.
- Dependency-specific failures, blocking behavior, exceptions, or nonlocal exits.
- Synchronization requirements and whether concurrent mutation is supported.
- Any invariant relating the object field at offset 0x88 to the acquisition value captured from the second context.

These unknowns do not alter the established local ordering, but they prevent stronger guarantees about safety, resource management, or progress.

## Evidence

The supplied static representation establishes the call order, byte offsets, four-byte data accesses, eight-slot record buffer, conditional branches, callback-continuation rule, and post-completion termination comparison.

Descriptions such as “record,” “dispatch context,” and “completion” are behavioral labels. Event-loop intent, callback handling semantics, and resource-release intent are not independently established.

No runtime tests have been performed. The acceptance examples are conditional consequences of the static evidence, not observed executions.
