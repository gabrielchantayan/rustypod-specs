<!-- Trusted provenance: F-42b31ced2afb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-42b31ced2afb

## Purpose

F-42b31ced2afb traverses an opaque sequence using a descriptor and key, evaluates nonzero candidates against a caller-provided unsigned threshold, and conditionally constructs a result from the **previous candidate**.

The selected candidate depends on how traversal ends:

- If lookup returns zero, the preceding nonzero candidate is selected, if one exists.
- If successful evaluation is followed by a working value greater than the threshold, the candidate preceding the just-evaluated candidate is selected.
- If evaluation returns a nonzero status, the function returns zero immediately, without constructing a result from any earlier candidate.

A selected nonzero candidate causes a storage request with size 28, followed by construction and optional postprocessing.

These are established control-flow properties. “Sequence,” “evaluation,” “storage request,” and “construction” describe apparent roles; the dependencies’ underlying data models and resource semantics are not established.

## Interface

The function takes four arguments and returns a 32-bit value.

| Input | Representation | Established use |
|---|---|---|
| Descriptor | Address carried by a 32-bit input | Provides a 32-bit field at byte offset 4 and, conditionally, an 8-bit field at byte offset 8. |
| Key | Opaque 32-bit value | Passed unchanged to every lookup and to construction. |
| Postprocessing selector | Signed 32-bit integer | Zero skips postprocessing; every nonzero value enables it. |
| Threshold | Unsigned 32-bit integer | Initializes a writable working value and supplies the reference for unsigned comparisons. |

The returned bit pattern is either zero from an early-return path or the 32-bit construction result. Construction itself may return zero. The interface provides no separate indication distinguishing these cases.

Neither candidates nor the return value are established to be pointers. Their nonzero values are treated as opaque 32-bit quantities.

### Descriptor layout and access requirements

| Byte offset | Width | Access condition |
|---|---:|---|
| 4 | 32 bits | Read for the initial gate, then reread before every lookup call. |
| 8 | 8 bits | Read only after a nonzero predecessor has been selected and the storage-request dependency has returned. |

No direct access to offset 0 is shown. The full descriptor layout, byte order, alignment requirements, and allocation provenance are unknown.

The descriptor must support the accesses actually reached during execution. In particular:

- Its offset-4 field must be readable at entry.
- If the initial gate succeeds, that field must remain readable for each subsequent lookup preparation.
- If construction is reached, its offset-8 byte must be readable after the storage request returns.

There is no explicit descriptor-address validation. A zero or otherwise invalid descriptor address is not a supported null-input case merely because a zero *field value* causes an early return.

Validity requirements imposed by dependencies on handles, keys, candidates, and requested storage remain unknown.

## Behavior

### Initial gate

The first operation relevant to the descriptor is a 32-bit read at offset 4.

If that value is zero, the function returns zero. No dependency is called, and the offset-8 byte is not read.

If it is nonzero, the function initializes:

- The current candidate to zero.
- A private unsigned 32-bit working value to the threshold.

The threshold remains the comparison reference; updates to the working value do not replace it.

### Lookup and evaluation cycle

At the beginning of each iteration, the current candidate is retained as the previous candidate.

The function then rereads the descriptor’s offset-4 field and calls F-f29429647db8 with, in argument order:

1. The freshly read 32-bit descriptor field.
2. The previous candidate.
3. The unchanged key.

The returned 32-bit value becomes the current candidate.

If that result is zero, traversal ends immediately. The evaluator is not called, and no threshold comparison is performed for that iteration.

For a nonzero current candidate, F-3bcbe089f356 is called with:

1. The current candidate.
2. The address of the writable working value.

A nonzero 32-bit evaluation status causes an immediate zero return. This path does not select an earlier candidate, request storage, construct a result, or invoke postprocessing. Any working-value modification associated with that failed evaluation does not change this precedence.

If the evaluation status is zero, the function compares the working value with the original threshold using an **unsigned 32-bit less-than-or-equal comparison**:

- Working value less than or equal to threshold: begin another iteration.
- Working value greater than threshold: stop traversal.

Equality therefore continues traversal.

### Working-value updates and address escape

The function directly initializes the working value once. It performs no subsequent direct arithmetic updates, increments, decrements, or resets.

The evaluator receives the working value’s address. The evidence does not establish that modifications through this address occur exclusively within evaluator calls. For example, an evaluator could publish or retain the address, allowing a later lookup dependency to modify the working value during the same invocation.

Consequently:

- A successful evaluation status does not establish that the evaluator wrote the working value.
- A value observed after evaluation may include changes made by an earlier dependency through an escaped address.
- The working value persists across iterations; it is not reinitialized before each lookup or evaluation.
- Lookup returning zero bypasses the comparison, even if that lookup modified the working value through an escaped address.

No such address publication or later modification is established to occur. It remains a permitted unknown rather than a required behavior. Safe retention beyond the working state’s lifetime is not established.

### Candidate selection

After ordinary traversal termination, the function checks the **previous candidate**, not the current candidate.

There are two termination cases:

| Termination reason | Current candidate | Candidate considered for construction |
|---|---|---|
| Lookup returns zero | Zero | The preceding candidate, which is the last nonzero lookup result if one exists. |
| Successful evaluation followed by working value above threshold | The just-evaluated nonzero candidate | Its predecessor. |

If the previous candidate is zero, the function returns zero without requesting storage.

This means a first nonzero candidate that immediately crosses the threshold is not constructed: its predecessor is still zero. Likewise, an immediate zero lookup result produces no selected candidate.

A selected predecessor is not reevaluated before construction. Selection does not establish that its associated memory, if any, remains unchanged or valid; dependency behavior is unknown.

### Storage request and construction

For a nonzero selected predecessor, the function calls F-8157a180d901 with the size value **28**.

After that call returns, the function reads the descriptor’s 8-bit field at offset 8. It then calls F-5f70729e3d77 with:

1. The storage-request result.
2. The selected predecessor.
3. The unchanged key.
4. The descriptor byte read after the storage request.

There is no intervening zero check on the storage-request result. If that result is zero and the dependency returns normally, construction is still invoked with zero as its first argument.

The descriptor byte is not an entry-time snapshot. If the storage-request dependency changes it through an alias or shared state, the subsequent read observes the byte present at that later access, subject to otherwise-valid memory access and synchronization.

The field itself is exactly 8 bits wide. Its meaning and the construction dependency’s formal parameter interpretation are not established.

### Optional postprocessing and return

The 32-bit construction result is retained for return.

If the postprocessing selector is nonzero, F-e03ee9fb75f9 is called with:

1. The construction result.
2. Zero.

This call is made even if construction returned zero. Any ordinary return value from postprocessing is ignored.

After postprocessing returns normally, the function returns the retained construction result. If the selector is zero, it returns that result without postprocessing.

Returning the same bit pattern does not establish that an object identified by that pattern remains live, unchanged, or caller-owned.

## Side effects

### Direct reads and writes

The function directly performs:

- One initial 32-bit read of the descriptor’s offset-4 field.
- Another 32-bit read of that field before each lookup.
- Initialization of private candidate state and the unsigned working value.
- Exposure of the working value’s address to evaluation.
- One conditional 8-bit read of the descriptor’s offset-8 field.

It makes no explicit write to the descriptor or through a candidate or construction result. There is no direct release operation.

### Observable ordering

On a path reaching construction, the established sequence is:

1. Initial descriptor-field read and gate.
2. Repeated descriptor-field reads and lookup calls.
3. Evaluation after each nonzero lookup result.
4. After successful evaluation, an unsigned comparison.
5. Selection of a nonzero predecessor.
6. Storage request with size 28.
7. Descriptor-byte read.
8. Construction.
9. Optional postprocessing.
10. Return of the retained construction result.

Lookup exhaustion skips evaluation and comparison for that iteration. Evaluation failure skips all later selection and construction activity.

The initial descriptor field is not cached as the lookup handle. If it changes after the gate, subsequent lookups receive the newly read value. In particular, a later field value of zero is still passed to lookup; the initial zero-field gate is not reapplied as a separate guard.

### Dependency-mediated mutation and ownership

No dependency is established to preserve globals, descriptor contents, candidate objects, or other shared state.

The working-value address creates an additional possible mutation channel during the invocation. Later dependencies may have access to it if it has escaped; the source does not restrict all such writes to evaluation calls.

The size-28 request suggests allocation intent, but the following remain unknown:

- Whether storage is actually allocated or initialized.
- Whether construction consumes, borrows, replaces, or releases the supplied storage.
- Whether construction failure releases anything.
- Whether postprocessing mutates, retains, transfers, or releases the result.
- Whether candidates or descriptor-associated resources require release elsewhere.

The function does not independently establish an ownership transfer to its caller.

## Dependencies

| Opaque ID | Contract visible at this call site | Unestablished behavior |
|---|---|---|
| F-f29429647db8 | Receives a freshly read handle, predecessor, and key; returns a 32-bit candidate. Zero ends traversal. | Sequence order, progress, validity, mutation, ownership, and possible access to escaped working state. |
| F-3bcbe089f356 | Receives a nonzero candidate and writable working-value address. Nonzero status causes immediate zero return. | Status meanings, update rules, address publication or retention, and other side effects. |
| F-8157a180d901 | Receives size 28; its result is passed to construction without a zero check. | Allocation semantics, initialization, failure convention, and ownership. |
| F-5f70729e3d77 | Receives storage result, selected predecessor, key, and descriptor byte; returns the final 32-bit result. | Object representation, validation, failure behavior, and resource consumption. |
| F-e03ee9fb75f9 | Conditionally receives construction result and zero. Its ordinary return value is ignored. | Operation meaning, lifetime effects, mutation, and failure signaling. |

No dependency implementation is supplied. Descriptions of subsequent actions assume the invoked dependencies return normally.

## Acceptance examples

These are conditional acceptance cases derived from static control flow, not executed tests. A and B denote distinct nonzero 32-bit candidates.

### Zero initial handle

Given a readable offset-4 field containing zero:

- Return zero.
- Invoke no dependency.
- Do not read the offset-8 byte.

### Immediate exhaustion

Given a nonzero initial handle and a first lookup result of zero:

- Do not evaluate a candidate.
- Return zero because the predecessor is zero.
- Do not request storage.

### First candidate crosses the threshold

Given first candidate A, evaluation status zero, and a working value above the threshold at comparison:

- Evaluate A.
- Stop traversal.
- Return zero without construction because A has no nonzero predecessor.

### Exhaustion after one eligible candidate

Given A, successful evaluation leaving the working value at or below threshold, and then a zero lookup result:

- Select A.
- Request size 28.
- Read the descriptor byte after the request returns.
- Construct using A.
- Return the construction result after any enabled postprocessing.

### Later threshold crossing

Given A followed by B, with successful evaluation of A leaving the working value at or below threshold and successful evaluation of B followed by a value above threshold:

- Select A, not B.
- Perform no further lookup before construction.

### Modification through an escaped address

Given threshold 10:

- Evaluation of A returns zero, leaves the working value at 6, and publishes its address.
- The next lookup modifies that value to 12 through the published address and returns B.
- Evaluation of B returns zero without changing the working value.

The comparison observes 12, traversal stops, and A is selected. The contract does not require the latest evaluator call to be the writer responsible for the threshold crossing.

If that second lookup instead returns zero, evaluation and comparison are skipped and A is selected through lookup exhaustion.

### Evaluation failure after progress

Given successful progress through A followed by a nonzero evaluation status for B:

- Return zero immediately.
- Do not construct A as a fallback.
- Do not request storage or invoke postprocessing.

### Equality and unsigned limits

If the working value equals the threshold after successful evaluation, continue traversal.

With threshold 4, a working value of 4,294,967,295 stops traversal; it is not interpreted as negative.

With threshold 4,294,967,295, every unsigned 32-bit working value satisfies the continuation comparison. Termination must come from another path, or execution may continue indefinitely.

### Zero construction-related results

If the storage request returns zero, still invoke construction with zero as its first argument.

If construction returns zero and the selector is nonzero, invoke postprocessing with zero and zero. If it returns normally, the function returns zero regardless of postprocessing’s ordinary return value.

### Descriptor changes between calls

If an earlier dependency changes the offset-4 field to zero, the next lookup receives zero as its handle.

If the storage-request dependency changes the offset-8 byte while leaving it readable, construction receives the subsequently read byte, not its earlier value.

## Uncertainties

The following are **documented_only**:

- Semantic types of the descriptor, key, candidates, and result.
- Complete descriptor layout, byte order, and alignment requirements.
- Meaning of the working value and threshold.
- Whether evaluation writes the working value on any particular call.
- Whether its address escapes and which later dependencies can access it.
- Whether traversal is finite, ordered, acyclic, or stable under mutation.
- Whether dependencies preserve shared state or referenced objects.
- Storage initialization, failure semantics, and ownership.
- Postprocessing’s meaning and lifetime effects.
- Exceptions, nonlocal exits, process termination, or other dependency behavior preventing ordinary return.
- Synchronization and concurrent-access requirements.

No finite iteration bound is established. Repeated nonzero candidates, zero evaluation statuses, and comparison values at or below threshold can sustain traversal indefinitely. Neither monotonic working-value updates nor candidate progress is enforced.

## Evidence

The supplied function body establishes the field-access offsets and widths, zero gates, call ordering and arguments, unsigned comparison, predecessor-selection rule, size-28 request, conditional postprocessing, and retained return-value flow.

It also establishes that the working value is initialized directly and its address is passed to evaluation. It does **not** establish exclusive evaluator control over subsequent modifications. Address escape and later dependency-mediated writes are unresolved possibilities, not observed events.

Search, allocation, and construction terminology expresses inferred intent rather than proven dependency semantics. No runtime tests have been performed. All acceptance examples are conditional deductions from the supplied static evidence.
