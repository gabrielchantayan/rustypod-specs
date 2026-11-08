<!-- Trusted provenance: F-e25db53ba6a5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e25db53ba6a5

## Purpose

F-e25db53ba6a5 prepares an object-like value from an input record, performs ordered recipient dispatch, and returns the first nonzero recipient result.

Dispatch has two stages:

1. An optional iterator-based stage invokes recipients until one returns nonzero or iteration ends.
2. If the first stage produces no nonzero result, a traversal-based stage invokes eligible recipients. An object query determines whether additional eligibility checks apply.

Before normal return, a nonnull prepared object receives a final indirect invocation.

“Object,” “recipient,” “iterator,” “query,” and “finalization” describe observed roles, not established application types or ownership rules. In particular, the initial dependency call is **not established to allocate storage**: its integer argument and the forwarding of its result are the only demonstrated behavior at this call site.

## Interface

### Input and output

The function accepts one 32-bit value used as an input-record address.

| Input-relative byte offset | Established use |
|---|---|
| 12 | The address formed by adding 12 to the input address is passed to F-4d7c82c9282a. This function does not directly read the contents at that address. |
| 20 | A 32-bit value is read and passed to F-4d7c82c9282a. |

The offset-12 argument is an address within the input record, not a pointer loaded from that location. The extent and interpretation of data accessed there by the dependency are unknown.

The returned value is a 32-bit integer:

- Zero if no invoked recipient callback returns nonzero.
- Otherwise, the unchanged result of the first recipient callback that returns nonzero.

Negative callback results count as nonzero and stop dispatch. The result is not normalized to a Boolean. Preparation, iterator, eligibility, query, advancement, and finalization results do not directly replace the retained recipient result.

### Widths and layouts

The supplied representation uses 32-bit addresses and 32-bit pointer-sized fields. Dispatch-table offsets and record offsets below are byte displacements.

| Referenced structure | Byte offset | Observed use |
|---|---:|---|
| Context returned by F-3fdd14ecd250 | 40 | Second-stage initial recipient pointer |
| Context returned by F-3fdd14ecd250 | 44 | First-stage collection value |
| Recipient | 0 | Dispatch-table pointer |
| Second-stage recipient | 12 | Associated pointer; also tested before advancement |
| Associated structure | 12 | 32-bit eligibility condition |
| Prepared object | 0 | Dispatch-table pointer |

The local iterator-state area occupies 20 bytes. A separate iterator output area contains two 32-bit pointer-sized slots; only its first slot is visibly consumed.

Two indirect operations produce represented 64-bit results. Their lower and upper 32-bit halves have distinct observed uses; the representation does not establish their higher-level types.

Address calculations are represented at 32-bit width. No wider arithmetic, bounds checking, overflow checking, or valid wraparound-address behavior is established.

### Preconditions for the described normal behavior

Every executed memory access must address suitable storage. Every executed indirect call requires a suitable receiver, readable dispatch table, and callable slot with a compatible interface.

In particular:

- The input’s offset-20 field must be readable.
- Context results must support the field reads that follow them.
- A nonzero iterator-advance result must provide a usable first output pointer.
- Enabled second-stage filtering requires the associated pointer to support a condition read before any later null check.
- Dependencies must return normally for the stated cleanup and return sequence to complete.

These are validity requirements, not checks performed by this function.

## Behavior

### Initial dependency calls and retained object

The first operation is a call to F-483a717208e6 with the integer argument 20.

Its result is then forwarded to F-4d7c82c9282a together with:

1. The integer constant 11.
2. The input address displaced by 12 bytes.
3. The 32-bit value read at input offset 20.

The first dependency call precedes the input-field read used by the second call. The supplied representation does not establish a meaningful relative evaluation order between forming the offset-12 address and reading the offset-20 value.

The result of F-4d7c82c9282a is retained as the prepared-object pointer. The recipient-result state is initialized to zero after that call.

There is no visible validation of F-483a717208e6’s result before it is forwarded. A zero result is forwarded as supplied. This does not establish that zero denotes failure, that the result identifies storage, or that the argument 20 is a byte count.

A null prepared-object pointer does not terminate dispatch. First-stage callbacks can receive null, while second-stage querying and finalization are conditional on nonnullness.

### First-stage guard and initialization

F-3fdd14ecd250 is called, and the returned context’s 32-bit field at offset 44 is read as a guard.

If this read is zero:

- There is no second read of that field on this path.
- No iterator initialization, advancement, or finalization occurs.
- Processing proceeds to second-stage setup.

If the guard is nonzero, the same context’s offset-44 field is read again. That second value is passed to F-538044b67754 with the local 20-byte iterator-state area and the 32-bit bit pattern `0xFFFFFFFE`.

Only the first read is tested. The second read is not independently required to be nonzero.

There is no visible dependency call between these two reads, and both use the same context address. Nevertheless, the function performs two reads rather than retaining the guard value. Equality requires a stability assumption that is not established here. No concurrent-access or atomicity contract is supplied.

The meaning of `0xFFFFFFFE` is unknown; its bit pattern is established without assigning iterator-specific semantics.

### First-stage iteration

After initialization, F-6e4651b532dc receives the iterator-state area and the two-slot output area.

- A zero result ends iteration.
- A nonzero result causes the first output slot to be treated as the current recipient pointer.

For each supplied recipient, the function reads its dispatch table and invokes the slot at byte offset 52, passing the recipient and the retained prepared-object pointer.

A zero callback result causes another iterator-advance call. A nonzero result is retained and ends the first stage.

F-944b595e25e9 receives the iterator-state area once on either visible normal exit from an initialized iteration:

- Iterator advancement returns zero.
- A recipient callback returns nonzero.

A nonzero first-stage result suppresses the second context lookup and all second-stage processing. Iterator finalization still occurs before prepared-object finalization.

There is no recipient null check before the first-stage dispatch-table access. The second iterator output slot is not visibly read.

### Second-stage setup

This stage is reached only if the first stage was skipped or exhausted without a nonzero callback result.

F-3fdd14ecd250 is called again. Its returned context’s 32-bit field at offset 40 is read and retained as the initial recipient pointer. This is a fresh context lookup; equality with the earlier context result is not required.

The initial recipient pointer is captured before any prepared-object query.

If the prepared object is null, no query occurs and filtering is disabled. The upper half of the initial represented traversal state is assigned `0x00000A00`.

If the prepared object is nonnull, its dispatch-table slot at byte offset 20 is invoked. The represented result is 64 bits:

- A zero lower 32-bit half disables filtering.
- A nonzero lower 32-bit half enables filtering.
- The upper 32-bit half is incorporated into traversal state but has no independently demonstrated effect on eligibility or the returned result.

The query occurs even when the captured initial recipient pointer is null. If the query changes the context’s offset-40 field, that change does not trigger a reread; traversal begins from the already captured pointer.

### Second-stage eligibility and callbacks

At each traversal entry, a null current recipient ends traversal. A retained nonzero recipient result also ends it, although the visible callback path already stops immediately upon obtaining such a result.

The filtering mode established during setup remains fixed throughout this stage.

**Filtering disabled:** the current recipient’s dispatch-table slot at byte offset 48 is invoked with the recipient and prepared-object pointer. No associated-structure condition read or eligibility call to F-35b8bd5fa9e0 is performed.

**Filtering enabled:** the function first reads the current recipient’s associated pointer at byte offset 12, then reads a 32-bit condition at offset 12 within that associated structure.

There is no null check of the associated pointer before this condition read.

- A zero condition permits the slot-48 callback without an eligibility dependency call.
- A nonzero condition causes F-35b8bd5fa9e0 to be called with the prepared-object pointer.
- A nonzero eligibility result permits the callback.
- A zero eligibility result skips the callback and leaves the recipient result at zero.

Eligibility is evaluated separately for each recipient whose condition requires it. Its result is not cached across recipients.

An invoked slot-48 callback supplies the retained 32-bit recipient result.

### Second-stage advancement and stopping

A nonzero slot-48 callback result stops traversal immediately. The later associated-pointer read and advancement call do not occur for that recipient.

If the callback returns zero, or filtering skips the callback, the recipient’s offset-12 pointer is read:

- Zero ends traversal.
- Nonzero permits invocation of the recipient’s dispatch-table slot at byte offset 28.

This is a later read, not a retained copy of the pointer used for filtering. It occurs after any eligibility call and callback. Mutations performed by those calls can therefore change whether advancement occurs.

Slot 28 produces a represented 64-bit result. Its lower 32 bits become the next recipient pointer. Its upper 32 bits remain part of traversal state but do not change filtering mode or independently determine eligibility.

The function does not visibly dereference the associated structure to find the next recipient. The associated-pointer check gates the advancement call; the advancement result supplies the next pointer.

### Finalization and return

After normal dispatch completion, the retained prepared-object pointer is checked.

If nonnull, its dispatch table is read and the slot at byte offset 4 is invoked with that object. The call’s result does not replace the retained recipient result.

If null, this invocation is omitted.

The function returns the retained 32-bit recipient result. No separate release operation is visibly applied to the value originally returned by F-483a717208e6. Whether that value denotes any releasable resource is unknown.

## Side effects

### Observable ordering and reads

Established ordering includes:

1. F-483a717208e6 before F-4d7c82c9282a.
2. Preparation of the retained object before the first context lookup.
3. The first collection-field read before its conditional second read.
4. Iterator initialization before iterator advancement and first-stage callbacks.
5. Iterator finalization before second-stage setup or object finalization.
6. Second-stage initial-pointer capture before the object query.
7. Enabled-filter condition reads before the optional eligibility call and callback.
8. A skipped callback or zero callback result before the later associated-pointer read and possible advancement.
9. Object finalization after recipient processing.

Unknown dependencies may mutate shared state, objects, or dispatch tables. No preservation guarantee is assumed. The two context lookups, two first-stage collection reads, and repeated recipient-field reads remain distinct observations.

The visible body maintains local state and reads referenced fields. Storage writes through dependency arguments or indirect calls are governed by those unknown operations; their precise effects are not established here.

### Ownership and storage

F-483a717208e6 receives the integer 20. The call site alone establishes neither storage allocation nor a requested storage extent.

F-4d7c82c9282a receives that dependency’s result and returns the retained object pointer. The relationship between those two values, including any ownership transfer, aliasing, replacement, or release, is unknown.

Iterator dependencies receive addresses of local storage. Their initialization requirements and whether they retain those addresses are unknown.

No direct release of the input record, contexts, collections, recipients, associated structures, or dispatch tables is visible. Unknown dependencies may retain, mutate, or release memory. The final slot-4 invocation does not establish destruction or deallocation.

### Errors and termination

No explicit error-reporting or recovery path is visible for invalid addresses, unsuitable dependency results, invalid dispatch slots, iterator protocol violations, or null associated pointers encountered during enabled filtering.

There is no cycle detection or iteration limit. Either stage may fail to terminate if dependencies continually supply recipients without a stopping result.

Cleanup is established only for visible normal paths. Faults, exceptions, nonreturning calls, and invalid-memory behavior have no established finalization guarantee.

## Dependencies

| Dependency | Established call-site contract |
|---|---|
| F-483a717208e6 | Receives integer 20. Its result is forwarded without a visible check. Units, result meaning, and effects are unknown. |
| F-4d7c82c9282a | Receives the preceding result, constant 11, input address plus 12, and the input’s 32-bit offset-20 value. Returns the retained object pointer. |
| F-3fdd14ecd250 | Returns a context address used for field reads. Called initially and again if second-stage setup is reached. |
| F-538044b67754 | Receives local iterator storage, the second offset-44 field value, and `0xFFFFFFFE`. |
| F-6e4651b532dc | Receives iterator storage and output storage. Zero ends iteration; nonzero permits consumption of the first output pointer. |
| F-944b595e25e9 | Receives iterator storage on both visible normal exits from initialized iteration. |
| F-35b8bd5fa9e0 | Receives the prepared object when enabled filtering encounters a nonzero condition. Nonzero permits the recipient callback. |

Indirect targets have no supplied opaque IDs and are identified by receiver and dispatch-table byte offset:

| Receiver and slot | Established use |
|---|---|
| First-stage recipient, 52 | Callback supplying a candidate 32-bit return value |
| Prepared object, 20 | Query whose lower 32 bits determine filtering mode |
| Second-stage recipient, 48 | Callback supplying a candidate 32-bit return value |
| Second-stage recipient, 28 | Advancement whose lower 32 bits supply the next recipient |
| Prepared object, 4 | Final invocation before normal return |

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, they assume valid accessed storage and normally returning dependencies.

1. **Initial result forwarding.** F-483a717208e6 receives 20 and returns zero. F-4d7c82c9282a receives zero as its first argument; no intervening rejection occurs. Neither allocation failure nor successful allocation follows from this observation.

2. **Zero collection guard.** The first offset-44 read returns zero. No second read or iterator operation occurs. A fresh context lookup starts second-stage setup.

3. **Different collection reads.** The guard is nonzero, but the second read is zero. Iterator initialization receives zero. Its behavior with that argument remains unknown.

4. **First-stage success.** Callbacks return zero, then negative seven. Iterator finalization occurs, the second stage is skipped, applicable object finalization occurs, and negative seven is returned.

5. **Immediate iterator exhaustion.** The first advancement after initialization returns zero. Iterator finalization still occurs before second-stage setup.

6. **Null prepared object.** No first-stage callback succeeds. The second stage uses disabled filtering, its callbacks receive null, and no object query or object finalization occurs.

7. **Null initial recipient.** The captured second-stage pointer is null and the object is nonnull. The object query still occurs, but no second-stage recipient callback or advancement runs.

8. **Query changes the context.** The query replaces the context’s initial-recipient field. Traversal still starts with the value captured before the query.

9. **Zero filtering condition.** Filtering is enabled and the condition is zero. Slot 48 runs without F-35b8bd5fa9e0.

10. **Eligibility rejection.** The condition is nonzero and F-35b8bd5fa9e0 returns zero. The callback is skipped, but the later pointer check and possible advancement still occur.

11. **Callback changes advancement state.** Slot 48 returns zero after setting the associated pointer to null. The subsequent read observes null and suppresses slot 28.

12. **Nonzero second-stage result.** Slot 48 returns five. No later associated-pointer read or advancement occurs for that recipient. Applicable finalization occurs, and five is returned.

13. **Null association under enabled filtering.** The condition read is attempted through the null associated pointer before any graceful termination check. A normal zero-result outcome is not established.

## Uncertainties

- **documented_only — Initial dependency semantics:** The meaning and units of argument 20, the nature of the returned value, and all allocation or resource-management effects of F-483a717208e6 are unknown.
- **documented_only — Object preparation:** F-4d7c82c9282a’s ownership rules, failure conventions, and relationship to the preceding result are unknown.
- **documented_only — Application meaning:** Record types, recipient classes, and meanings of constants 11 and `0xFFFFFFFE` are unknown.
- **documented_only — Field stability:** Equality of separate collection reads and any concurrency guarantees are not established.
- **documented_only — Iterator protocol:** Output semantics beyond the first consumed pointer, mutation tolerance, and retention of local-storage addresses are unknown.
- **documented_only — Finalization:** Slot 4’s destruction, release, or other effects are unknown.
- **documented_only — Dependency preservation:** Unknown callees are not assumed to preserve globals, objects, dispatch tables, or ownership.
- **documented_only — Wide results:** Higher-level types and meanings of the represented 64-bit results remain unresolved beyond their observed half-width uses.
- **documented_only — Abnormal execution:** Fault, exception, and nonreturning-call behavior is unresolved.
- **Inferred intent:** The structure resembles prioritized dispatch followed by optionally filtered traversal. An allocation-and-construction interpretation of the initial calls is possible, but is not an established requirement.

## Evidence

This contract derives from static inspection of the supplied function representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent evidence of behavior.

Established evidence consists of call arguments, represented widths, field and dispatch-table offsets, distinct reads, branch conditions, result propagation, and normal-path finalization placement. In particular, the first dependency receives integer 20 and its result is forwarded; no supplied dependency implementation establishes allocation semantics.

The supplied representation notes removal of an unreachable block. No behavior is attributed to that omitted block.

No runtime tests or observations have been performed. Acceptance examples describe conditional consequences of the static evidence, not verified executions.
