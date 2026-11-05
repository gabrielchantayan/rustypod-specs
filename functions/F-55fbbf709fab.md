<!-- Trusted provenance: F-55fbbf709fab; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-55fbbf709fab

## Purpose

F-55fbbf709fab repeatedly examines a selected fixed-stride record within an indefinitely repeated sequence of processing phases. An iteration may invoke dependencies, clear a record byte, and conditionally invoke another dependency. Every completed iteration calls a polling dependency. A zero low-order polling result ends the current phase, not the function.

The function has no normal return path. If all dependencies return and memory accesses remain valid, execution continues indefinitely.

The record examined initially need not be the record subsequently modified. The function reads the selection once to decide whether to process a record, then reads it again before clearing a byte. Dependency effects or other external changes can make those reads select different records.

“Selection,” “activity,” “control,” and “phase” describe observable roles. They do not establish application-level meanings such as jobs, locks, events, or resource ownership.

## Interface

### Explicit arguments and return

The function takes no explicit arguments and produces no normal return value.

Dependency non-return, invalid memory access, or another external interruption can prevent further execution. The body does not convert such events into a returned error.

### Implicit inputs

The supplied representation uses the following externally stored inputs:

| Input | Representation | Access timing |
|---|---|---|
| Selection-storage pointer | Pointer to a signed 32-bit integer | Captured once at entry |
| Record-storage base | 32-bit address-bearing value | Captured once at entry, after the selection-storage pointer |
| Auxiliary argument | 32-bit value | Read only in the optional-call branch, after F-6ac70711bbb5 returns |

The captured pointer and base are reused across all phases. Updating the external locations from which they were obtained does not itself replace the captured values.

The selection integer is not captured once for the invocation. Its contents are read during each iteration and read again when record processing reaches the clearing operation.

### Record layout

The record stride is 20 bytes.

| Offset from record start | Access width | Established use |
|---|---:|---|
| 14 | 8 bits | Activity byte: zero skips record processing; any nonzero value enables it |
| 15 | 8 bits | Optional-call control byte: zero skips two calls; any nonzero value enables them |
| 16 | 32 bits | Post-clear control field: zero skips F-ext-03; any nonzero value enables it |

The body does not directly access offsets 0 through 13.

The initial selection check recognizes exactly signed 32-bit −1 as a sentinel. No other negative value or excessive positive value is rejected by an explicit bounds check.

### Preconditions for valid execution

The following are external requirements rather than validations performed by this function:

- The entry-time input locations must be readable.
- The captured selection-storage pointer must remain valid for every selection read.
- Each selection actually used for addressing must yield valid storage for the particular field accessed.
- The activity-byte destination must be writable whenever the processing branch reaches its store.
- The auxiliary input location must be readable when the optional-call branch accesses it.
- Dependencies must support the argument and return conventions used at their call sites.

Record count, allocation extent, alignment requirements, and permitted index range are not supplied. In particular, the sentinel check does not establish that other selection values are safe.

## Behavior

### Entry capture and phase entry

At function entry, the selection-storage pointer is obtained first and the record-storage base second. These captures precede the repeated phases.

Each phase starts with these calls, in order:

1. F-ext-01 with integer argument 8.
2. F-ext-02 with integer argument 4.

After both return, the processing loop executes at least once. Neither call is conditional on the selection or record contents.

The external identifiers F-ext-01 through F-ext-04 are document-local opaque dependency IDs. Their meanings and resolved identities are not supplied. The call positions and numeric arguments do not establish locking, scheduling, allocation, or initialization behavior.

### Initial selection and activity test

Each processing iteration first reads the signed 32-bit selection through the captured pointer.

If that value is −1, the function skips record processing without reading a record field. It still proceeds to polling.

Otherwise, the function computes a record address from the captured base and 20 times that selection, then reads the activity byte at offset 14.

- If the byte is zero, record processing is skipped and polling follows.
- If the byte is nonzero, record processing continues.

Any nonzero byte value qualifies. There is no requirement that the value equal 1.

The branch depends on the activity value read at this point. The function does not subsequently validate that this byte remains nonzero.

### Optional dependency calls

After observing a nonzero activity byte, the function reads offset 15 of the same initially addressed record.

A zero control byte skips the optional calls. A nonzero control byte causes this sequence:

1. Call F-6ac70711bbb5 without explicit arguments.
2. Retain its 32-bit result.
3. Read the auxiliary 32-bit input.
4. Call F-2926dbb9b751 with the retained result as its first argument and the auxiliary value as its second.

The result of F-2926dbb9b751 is not used by this body.

There is no failure-value check on the result of F-6ac70711bbb5. Zero, all-one bits, and other 32-bit results are passed onward without a distinct branch.

The auxiliary input is read after F-6ac70711bbb5 returns, not at function entry. A change to that input during F-6ac70711bbb5 can therefore affect the next call’s second argument. This establishes read timing, not any restriction on what either dependency may modify.

### Selection reread and activity clearing

After completing or skipping the optional calls, the function reads the selection integer again. It computes a fresh record address from this second observed value and the captured base.

It then writes an 8-bit zero at offset 14 of that recomputed record.

There is no intervening:

- −1 sentinel check;
- comparison with the first selection;
- bounds check;
- activity-byte read for the recomputed record; or
- control-byte read for the recomputed record.

Consequently, a record that was inactive, or was never examined initially, can be the record cleared.

The second selection read occurs even if the optional calls were skipped. In that case, a changed selection would require mutation from another source or an otherwise externally affected read; the body establishes no synchronization contract excluding such changes.

If the second read observes −1, the function still attempts the record-address calculation and activity-byte store. The initial sentinel handling does not protect this later access.

### Post-clear field test

After writing the activity byte, the function reads the 32-bit field at offset 16 using the same recomputed record address.

- A zero field value skips F-ext-03.
- Any nonzero field value causes F-ext-03 to be called without explicit arguments.

The test is against zero, not against a signed positivity threshold. A value with its highest bit set still enables the call if its full 32-bit value is nonzero.

The controlling value is the one obtained by this post-clear read. An earlier nonzero value does not guarantee the call: either optional dependency may change the field before it is read, and external mutation is not ruled out.

There is no selection reread between the clearing store and this field read. A subsequent selection change therefore does not redirect this particular field access.

### Polling, repetition, and phase exit

Every iteration that reaches its end calls F-db50b8e82ff9 with integer argument 0. This includes iterations that skipped record processing because the selection was −1 or the activity byte was zero.

The supplied representation exposes a 64-bit polling result. Only its low 32 bits determine loop continuation:

| Low 32 bits | Consequence |
|---|---|
| Nonzero | Start another processing iteration within the current phase |
| Zero | End the current processing phase |

The high 32 bits do not affect an observable branch, record address, store, or dependency argument in this body.

After the polling low word is zero, the function calls F-ext-04 with integer argument 4. When that call returns, the next phase begins with F-ext-01 and then F-ext-02.

There is no normal exit after phase completion.

## Side effects

### Direct memory accesses

The only explicit store in the body is the 8-bit zero written to offset 14 of the record addressed using the second selection read.

Direct reads comprise:

- The selection-storage pointer and record-storage base at entry, in that order.
- The first selection value in each processing iteration.
- The initially addressed activity byte unless that selection is −1.
- The initially addressed control byte if the activity value read was nonzero.
- The auxiliary input after F-6ac70711bbb5 returns, if the optional sequence is enabled.
- The second selection value if record processing was entered.
- The recomputed record’s 32-bit field after the clearing store.

There are no separate explicit stores to the selection, auxiliary input, control byte, or post-clear field. Storage disjointness is not established, so this does not exclude effects caused by aliasing the activity-byte destination with other storage.

### Dependency effects and ownership

Dependency effects are unknown. No dependency is assumed to preserve the selection integer, record contents, auxiliary input, or other global storage.

The body contains no explicit allocation or release operation. Passing a 32-bit result from F-6ac70711bbb5 to F-2926dbb9b751 does not establish whether that value is an address, handle, status, or ownership-bearing object. Acquisition, retention, transfer, and release obligations remain unknown.

There is no explicit error reporting, timeout, retry limit, rollback, or failure cleanup path. The repeated loops are established control flow, not evidence of a particular recovery policy.

## Dependencies

| Opaque function ID | Invocation condition | Explicit arguments | Result use |
|---|---|---|---|
| F-ext-01 | Start of every phase | 8 | No result affects documented behavior |
| F-ext-02 | After F-ext-01 returns | 4 | No result affects documented behavior |
| F-6ac70711bbb5 | Initially read activity and control bytes are both nonzero | None | Its 32-bit result becomes the next call’s first argument |
| F-2926dbb9b751 | After F-6ac70711bbb5 returns and the auxiliary input is read | Retained result, then auxiliary 32-bit value | Not used |
| F-ext-03 | Post-clear read of offset 16 yields a nonzero value | None | Not used |
| F-db50b8e82ff9 | End of every completed processing iteration | 0 | Low 32 bits control repetition |
| F-ext-04 | Polling low 32 bits are zero | 4 | Not used |

F-ext-01 through F-ext-04 identify four distinct external call targets within this document. Their full calling contracts and resolved provenance IDs are unavailable.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, accessed storage is valid and dependencies return normally. Each example concerns the described iteration; later iterations can observe different state.

### No selected record

If the first selection read yields −1:

- No record field is accessed during that iteration.
- No record-processing dependency is called.
- Polling occurs with argument 0.
- A nonzero polling low word starts another iteration.
- A zero polling low word leads to F-ext-04 with 4, then a new phase.

A nonzero polling high word does not change these decisions.

### Selected but inactive record

If the first selection is valid and the activity-byte read yields zero:

- The control byte is not read.
- The selection is not reread for clearing.
- No activity-byte store occurs.
- The post-clear field is not read.
- Both optional calls and F-ext-03 are skipped.
- Polling still occurs.

### Active record without optional calls

If both selection reads identify the same valid record, its activity-byte read is nonzero, its control-byte read is zero, and its post-clear field read yields zero:

- F-6ac70711bbb5 and F-2926dbb9b751 are skipped.
- The second selection read precedes the activity-byte store.
- The activity byte is written as zero.
- The offset-16 field is read after that store.
- F-ext-03 is skipped.
- Polling follows.

### Active record with both conditional sequences

Suppose the initially selected record’s activity and control reads are nonzero, both selection reads identify the same valid record, and the offset-16 field is nonzero **when read after clearing**.

Then:

- F-6ac70711bbb5 runs before F-2926dbb9b751.
- Its 32-bit result becomes F-2926dbb9b751’s first argument.
- The auxiliary input is read between those calls.
- Activity clearing follows both calls.
- The offset-16 read follows the clearing store.
- F-ext-03 follows the nonzero field read.
- Polling follows F-ext-03.

An initially nonzero offset-16 field alone is insufficient. If either optional dependency changes it to zero and the later read observes zero, F-ext-03 is skipped despite the stable selection.

### Selection changes during optional calls

Suppose record A supplies the nonzero activity and control values, and the second selection read observes a valid record B distinct from A.

The clearing store and post-clear field read target B. The function does not explicitly clear A merely because A enabled processing. F-ext-03 depends on B’s field value at the later read.

A dependency setting the selection to B is not alone sufficient to establish this outcome. A later dependency or another writer may change it again before the second read.

Likewise, the unchecked-sentinel case applies only if the second selection read actually observes −1. If it does, the function attempts the corresponding address calculation and store without repeating the sentinel check; valid execution is not guaranteed.

### Auxiliary input changes

If F-6ac70711bbb5 changes the auxiliary input and the subsequent auxiliary read observes that changed value, F-2926dbb9b751 receives it as its second argument. The function does not use an entry-time copy of that input.

If F-6ac70711bbb5 returns zero, zero is still passed as the first argument.

### Polling width boundary

A polling result with all low 32 bits set repeats the processing loop, regardless of signed interpretation or high-word contents.

A polling result with a zero low word and a nonzero high word ends the phase and causes F-ext-04 to run.

## Uncertainties

### Established static evidence

The supplied body establishes field widths and offsets, the 20-byte stride, initial sentinel handling, the second selection read, call ordering, the clearing store, the post-clear field test, and indefinite phase repetition.

Signedness of the one-byte control read does not affect its zero-versus-nonzero decision.

### Inferred intent

The descriptive terms used here do not establish application semantics. In particular, “activity clearing” names a zero-byte store; it does not prove acknowledgment, completion, cancellation, or resource release.

### documented_only unknowns

- **Address arithmetic:** Selection and record-address calculations use 32-bit integer quantities in the supplied representation. Exact machine-level overflow and address-conversion behavior are not independently established. Overflowing arithmetic is not a supported indexing guarantee.
- **Storage:** Record count, valid indices, object lifetimes, alignment, and aliasing constraints are unspecified.
- **Concurrency:** Atomicity, synchronization, memory barriers, and race safety are not established. Described access order is not a general hardware memory-order guarantee.
- **Dependency mutation:** No complete mutation set is available for any dependency.
- **Calling conventions:** The representation supplies the result widths used here, but not complete external ABI contracts or semantic return types.
- **Ownership:** No dependency’s allocation, retention, transfer, or release obligations are established.
- **Failures:** Faults, exceptions, nonlocal exits, and dependency-specific failure behavior are unspecified.
- **External identity:** Resolved opaque provenance IDs are unavailable for F-ext-01 through F-ext-04.

## Evidence

This contract is based on static inspection of the supplied function body and opaque dependency mapping. The prior draft and review feedback are not treated as independent behavioral evidence.

No runtime tests, dependency executions, smoke runs, concurrency experiments, or memory-layout validations have been performed. Acceptance examples specify consequences under their stated observations and preconditions; they are not measured results.
