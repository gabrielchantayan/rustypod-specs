<!-- Trusted provenance: F-f8ab014ad7b0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f8ab014ad7b0

## Purpose

F-f8ab014ad7b0 consumes a sequence of byte-addressed items, dispatches each item according to a classification table, drains outstanding entries, and performs unconditional finalization. A qualifying item whose first byte is `0x0E` additionally enables a reporting and notification phase after finalization.

The established behavior is the ordering of these operations, the predicates selecting them, the supplied arguments, and the direct memory accesses. Interpreting the items as tokens, outstanding entries as unmatched tags, or the final phase as diagnostic presentation is inferred intent, not a prerequisite for the contract.

This function has no explicit result value. Its observable results are calls, state changes made by dependencies, and one conditional direct byte write.

## Interface

### Inputs

The function receives four arguments:

1. **State address.** A pointer to a structure represented through 32-bit words. Relevant fields and subobjects are described below.
2. **Second argument.** A 32-bit value with no observable use in the supplied body.
3. **Third argument.** A 32-bit value with no observable use in the supplied body.
4. **Initial item address.** A byte pointer copied into temporary pointer storage on entry. That initial value is never subsequently read by the visible body: before the temporary storage is passed to a callback, it is overwritten with the current item address.

Consequently, the visible behavior does not depend on the values of the second, third, or fourth arguments. This does not establish a broader ABI guarantee about callers or surrounding code.

### Widths and state layout

The supplied representation uses 32-bit integers and pointer-bearing words. Offsets below are byte offsets. These are requirements of this represented interface, not a portable layout for another architecture.

| State offset | Visible role |
|---|---|
| `+0x00` | A word interpreted as an owner-object address. |
| `+0x04` | Start of an embedded subobject; its first word is interpreted as a dispatch-table address. |
| `+0x08` | A signed 32-bit outstanding-entry count. |
| `+0x1C` | A word supplied to the item-fetch dependency on every iteration. |

The embedded subobject begins at state `+0x04`; therefore the outstanding-entry count also lies within the address range of that subobject. Treating these as independent objects with unrelated storage would not preserve the visible layout.

The owner object is accessed at:

| Owner offset | Visible role |
|---|---|
| `+0x24` | Address passed to an accessor during conditional reporting. |
| `+0x38` | One-byte field set to `1` during conditional reporting. |
| `+0x3C` | Pointer to an object used for a final indirect call. |
| `+0x54` | Address passed to a notification dependency. |

### External data

The function also uses:

- A classification-table base captured once at entry.
- A diagnostic-context global, evaluated at its individual uses.
- A global pointer whose addressed byte gates supplemental reporting.
- Several message or text references whose exact contents are not supplied.

Each classification-table record has a stride of 16 bytes. For an item’s unsigned first byte, the function reads record byte `+0x0C` during consumption and record byte `+0x0D` during draining. Covering all possible byte values requires readable storage through table offset `0xFFD`.

### Preconditions and completion

All directly dereferenced addresses must be valid at the time of access. In particular:

- Every non-null fetched item must expose its first byte.
- A drain operation must return an address exposing a first byte whenever the count was positive at the preceding check.
- Dispatch objects, tables, and selected call slots must be valid.
- Owner fields and external data must remain valid when accessed, including after intervening calls.

No null checks, bounds checks, count repair, or memory-fault recovery are visible for these requirements.

Normal completion additionally depends on the item-fetch operation eventually returning null, the drain loop eventually observing a nonpositive count, and every invoked dependency returning normally.

## Behavior

### 1. Establish invocation-local state

The function captures the classification-table base before its first dependency call. It initializes an invocation-local reporting latch to false and initializes temporary pointer storage from the fourth argument.

The captured table base is reused throughout both loops. A dependency changing the global table-base value would not redirect this invocation to a new table. Changes to memory behind the captured address could still affect later classification reads.

### 2. Consume fetched items

The function repeatedly calls F-28d252870c28 with the current word at state `+0x1C`. It reloads this field for each call; it does not preserve the first supplied value across iterations.

A null result ends consumption. For each non-null result:

1. Read the item’s first byte as an unsigned value from `0` through `255`.
2. Select its 16-byte table record.
3. Read the classification byte at record `+0x0C`.
4. Select exactly one action, in the following precedence order.

**Classification byte equals zero.**

The function places the fetched item address into temporary pointer storage. It then obtains the embedded subobject’s dispatch table and calls the slot at table offset `+0x1C`, supplying:

- The embedded subobject address, state `+0x04`.
- The address of the temporary pointer storage.

The callback may change the stored pointer. The visible body does not consume that changed value afterward. On a later occurrence of this branch, the storage is overwritten with that later item address before another callback.

**Classification byte equals one.**

The function calls F-d7d31e5c4880 with the state address.

**Classification byte is neither zero nor one, and the item’s first byte equals `0x0E`.**

The function sets the invocation-local reporting latch to true. It does not call F-593b94f57438 for this item.

**All remaining cases.**

The function calls F-593b94f57438 with the state address, item address, and a 32-bit zero.

The classification tests take precedence over the `0x0E` test. An item beginning with `0x0E` does not enable reporting when its classification byte is zero or one. Once enabled, the latch is never cleared by the visible body.

The function itself does not advance an item cursor or alter the input-source field. Progress relies on dependency behavior or external state.

### 3. Drain outstanding entries

After item fetching first returns null, the function repeatedly reads the signed 32-bit value at state `+0x08`.

- A value greater than zero enters another drain iteration.
- Zero or a negative value ends draining.

Each drain iteration calls F-6f6709bb269a with the embedded subobject address. Its returned item address is immediately dereferenced without a null check.

The item’s unsigned first byte selects the same captured table, but this phase reads record byte `+0x0D`:

- If that byte is exactly one, call F-593b94f57438 with the state address, returned item address, and zero.
- Otherwise, call F-bd120056260a with the state address, returned item address, a fixed message reference, and one.

The visible body does not decrement the outstanding-entry count. It rechecks the count after the selected dependency returns. Termination therefore depends on changes made outside this body.

This phase does not inspect record byte `+0x0C`, does not use the consumption callback, and does not enable reporting for an item beginning with `0x0E`.

### 4. Finalize unconditionally

Once draining ends, call F-47a3c8569768 with the state address.

This call occurs whether or not any items were fetched, whether or not draining ran, and whether or not reporting was enabled.

If the reporting latch is false when this call returns, the function returns immediately. The latch is local, but all shared fields used by subsequent operations remain subject to changes made by the finalization dependency.

### 5. Perform conditional reporting

When the reporting latch is true, the function first obtains an address corresponding to owner `+0x24` and saves that address for subsequent accessor calls. It then writes the byte value `1` at owner `+0x38`.

This direct flag write occurs after F-47a3c8569768 and before the diagnostic query. No visible operation resets the byte.

Next, call F-6d70e45e29c9 with the diagnostic-context global.

**If the query result is zero:**

1. Call F-44362605ab79 with the diagnostic-context global as evaluated at this later use.
2. Call F-826f25835f54 with the saved owner-relative address.
3. Call F-151ab4bfe553 with the accessor result first and the F-44362605ab79 result second.

The query call is not known to preserve the diagnostic-context global. Accordingly, the two diagnostic calls are not guaranteed to receive the same value.

**If the query result is nonzero:**

1. Call F-826f25835f54 with the saved owner-relative address.
2. Call F-438680ffebfe with the accessor result and a fixed status-message reference.

The visible condition is only zero versus nonzero. A richer interpretation of the query result is not established.

### 6. Optionally emit supplemental text

After the preceding diagnostic branch completes, the function reads the byte addressed by a separate global pointer.

If that byte is nonzero:

1. Call F-826f25835f54 with the saved owner-relative address.
2. Call F-6c487c12e753 with that result and a fixed separator-like text reference.
3. Call F-826f25835f54 again with the same saved address.
4. Call F-6c487c12e753 with that second result and a second externally stored text reference.

The accessor runs separately for the two operations. Its results need not be equal, and the first output call is not assumed to preserve state used by the second.

The byte condition is evaluated after earlier reporting calls, not at function entry. Those calls may therefore affect whether supplemental reporting occurs.

### 7. Notify and invoke the final indirect operation

Still within the reporting-enabled path:

1. Reload the owner address from state `+0x00`.
2. Call F-d82d61c5d5f8 with owner `+0x54`, the integer value `2`, and zero.
3. Reload the owner address again.
4. Read its pointer field at `+0x3C`.
5. Read the pointed object’s first pointer-bearing word as a dispatch-table address.
6. Invoke the slot at dispatch-table offset `+0x30`.

No explicit arguments are represented for the final indirect call. Its actual calling convention and hidden argument requirements are unknown.

The function then returns if that indirect operation returns normally.

## Side effects

### Direct writes

The only established direct write to caller-reachable object storage is the conditional one-byte store of `1` at owner `+0x38`.

The function also writes invocation-local storage: the reporting latch and temporary item pointer. Passing the temporary pointer’s address to a callback allows that callback to modify it during the call. Retaining and later dereferencing that address would require a lifetime guarantee not supplied here.

### Dependency-mediated changes

All dependencies and indirect callbacks may have side effects not shown by this body. In particular, this contract does not assume they preserve:

- State fields, including the owner pointer, input-source word, or outstanding-entry count.
- Diagnostic globals.
- Classification-table contents.
- Objects returned by earlier calls.

The saved reporting address remains the address computed before the diagnostic query. Later notification operations instead use newly loaded owner addresses. If dependencies replace the owner pointer, reporting and notification can consequently target different objects.

### Ownership and errors

No explicit allocation, deallocation, reference-count operation, or transfer of ownership appears in the supplied body. Passing or receiving a pointer does not establish whether a dependency borrows, retains, consumes, or releases it.

There is no explicit success/failure return, exception handling, rollback, or cleanup-on-error branch. The drain-phase message-producing call does not visibly terminate processing: if it returns, the count is checked again.

Earlier writes and calls are not reversed if a later operation fails or does not return.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-28d252870c28 | Fetch an item from the current state `+0x1C` value; null ends consumption. |
| F-d7d31e5c4880 | Handle consumption classification one using the state address. |
| F-593b94f57438 | Handle selected consumption and drain items, with a final zero argument. |
| F-6f6709bb269a | Obtain an item during draining from the embedded subobject. |
| F-bd120056260a | Handle a drain item with a fixed message reference and final argument one. |
| F-47a3c8569768 | Unconditional post-drain finalization. |
| F-6d70e45e29c9 | Query the diagnostic context; only zero/nonzero is interpreted. |
| F-44362605ab79 | Obtain a value from the diagnostic context on the zero-result path. |
| F-826f25835f54 | Obtain a reporting target or value from the saved owner-relative address. |
| F-151ab4bfe553 | Receive accessor and diagnostic-derived values on the zero-result path. |
| F-438680ffebfe | Receive an accessor result and fixed status message on the nonzero path. |
| F-6c487c12e753 | Receive each optional supplemental text reference. |
| F-d82d61c5d5f8 | Receive the freshly derived notification address, two, and zero. |

Two additional dependencies are indirect: the embedded-subobject callback at slot `+0x1C` and the owner-associated operation at slot `+0x30`. Their identities and complete contracts are unknown.

## Acceptance examples

These examples are conditional static acceptance cases, not executed tests. Assume valid addresses and normally returning dependencies unless stated otherwise.

1. **Empty source, no outstanding entries.**  
   If the first fetch returns null and the signed count is zero, the function calls finalization and returns. It performs no reporting flag write, diagnostic query, notification, or final indirect call.

2. **Classification overrides the marker byte.**  
   If an item begins with `0x0E` and its consumption classification is zero, the embedded-subobject callback receives it through pointer storage. That item alone does not enable reporting. Classification one instead selects F-d7d31e5c4880, also without enabling reporting.

3. **A qualifying marker enables one post-processing phase.**  
   If an item begins with `0x0E` and its classification is neither zero nor one, reporting is enabled. Additional qualifying items do not introduce additional reporting phases. Reporting occurs only after source exhaustion, draining, and finalization.

4. **Ordinary consumption forwarding.**  
   If an item begins with a byte other than `0x0E` and its classification is neither zero nor one, F-593b94f57438 receives that item and a final zero argument.

5. **Drain classification is independent.**  
   If the count is positive and the returned drain item has record byte `+0x0D` equal to one, F-593b94f57438 is selected regardless of record byte `+0x0C`. If the next count check is zero, draining stops.

6. **Negative count skips draining.**  
   If source exhaustion is followed by a negative signed count, no drain fetch occurs. Finalization still occurs.

7. **The count does not make progress.**  
   If every drain iteration returns normally while the count remains positive, the function continues draining. No local decrement, iteration limit, or recovery path ends the loop.

8. **Diagnostic zero branch.**  
   Given an enabled reporting latch and a zero query result, F-44362605ab79 runs before the reporting accessor, followed by F-151ab4bfe553. The fixed-status-message operation is not selected.

9. **Owner replacement during reporting.**  
   If a dependency changes the state’s owner pointer after the reporting address is saved, subsequent accessor calls still receive the saved address. Notification uses the then-current owner; the final indirect dispatch reloads it again after notification.

10. **Supplemental condition changes during reporting.**  
    If earlier reporting calls cause the addressed condition byte to become nonzero, both supplemental text operations occur. An entry-time zero value alone does not suppress them.

## Uncertainties

The following remain **documented_only**:

- Exact item format beyond the first byte.
- Semantic meanings of classification values and message contents.
- The diagnostic query’s business meaning.
- Dependency ownership, mutation, lifetime, and failure contracts.
- Whether the outstanding-entry count is maintained by the drain fetch, its handler, or another mechanism.
- Whether accessor results are stable across calls.
- Whether the final indirect operation returns, and its complete ABI.
- Concurrency, reentrancy, and synchronization requirements.

The visible arithmetic uses unsigned byte indexing, a 16-byte record stride, and fixed byte offsets. Those table offsets do not overflow a 32-bit value. Validity of the resulting addresses, owner-relative address arithmetic, and alignment of pointer-bearing accesses still depends on the supplied memory model and caller-established preconditions.

## Evidence

**Established static evidence:** the supplied body supports the branch precedence, signed count predicate, byte-sized table reads, fixed layouts, call arguments, direct flag write, saved-versus-reloaded addresses, and operation ordering specified above.

**Inferred intent:** token consumption, unmatched-entry draining, diagnostic presentation, separator output, and notification are descriptive interpretations. They do not expand the established contract.

**Unavailable evidence:** dependency implementations, complete data declarations, exact message bytes, full indirect-call signatures, and runtime observations were not supplied.

No runtime tests have been performed.
