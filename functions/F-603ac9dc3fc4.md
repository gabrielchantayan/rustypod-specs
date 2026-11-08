<!-- Trusted provenance: F-603ac9dc3fc4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-603ac9dc3fc4

## Purpose

F-603ac9dc3fc4 performs ordered, selector-driven dispatch. It recognizes one of eight candidate labels, conditionally invokes the corresponding handler with an object reference and a 32-bit value, and returns a 32-bit recognition result.

Recognition and handler execution are distinct:

- A recognized candidate produces result **1**, including when sentinel equality suppresses its handler.
- Failure to recognize every candidate produces result **0**.
- A normally returning handler does not determine the wrapper’s result.

The labels suggest menu, highlighting, selection, playback, and contextual-menu actions. Those meanings are inferred intent. The supplied evidence establishes dispatch behavior, not the application behavior of the handlers.

Two global sentinel locations participate in suppression. Their read timing differs by candidate: the first and third candidates use an additional direct read, while the second and fourth-through-eighth candidates use previously captured values. These distinctions are part of the source-level contract.

## Interface

### Inputs

| Input | Representation established by the supplied representation | Use |
|---|---|---|
| Object reference | Pointer to storage whose first word supplies a dispatch-table address on indirect paths | Forwarded unchanged to a selected handler; dereferenced locally only for an unsuppressed indirect dispatch |
| Selector | Opaque 32-bit value | Forwarded unchanged to the comparison dependency with each candidate label |
| Value | 32-bit integer | Compared for equality with a sentinel value and forwarded unchanged to an invoked handler |

The selector’s representation is not established beyond its four-byte width. In particular, the evidence does not establish that it is a string pointer, an interned identifier, or a numeric action code.

The value undergoes no visible arithmetic, narrowing, masking, or conversion. Suppression uses equality or inequality between 32-bit operands; signed ordering is irrelevant to those comparisons.

### Result

The function returns a four-byte result:

- **1:** a candidate was recognized and its branch completed normally.
- **0:** all eight comparison calls returned nonzero.

These are normal-completion results. No return value is established for execution interrupted by an invalid memory access, a dependency that does not return, or another abnormal transfer of control.

### Required state and layout

Two distinct readable global 32-bit locations are designated **sentinel A** and **sentinel B**. Their values, initialization, intended meanings, and writers are unknown.

An unsuppressed indirect dispatch requires:

1. A readable first word at the object reference.
2. A usable dispatch-table address obtained from that word.
3. A readable callable entry at the selected table offset.
4. A target callable with the object reference and original 32-bit value.

The selected byte offsets are **0x150**, **0x154**, and **0x128**. They are offsets from the dispatch-table address, not entry indices or object-member offsets.

The representation is consistent with a 32-bit address model. It does not establish a complete ABI, a portable native-pointer layout, or callable-entry representation on other architectures. No value-derived index or offset arithmetic occurs.

### Ownership and validity

No allocation, release, reference-count adjustment, or explicit ownership transfer is visible in the wrapper.

That observation does not constrain dependency ownership behavior. Comparison and handler dependencies may have effects not supplied here, including effects on globals, object validity, or retained references. The wrapper does not visibly restore state or perform cleanup after a handler.

Accessible-state requirements are path-dependent. A suppressed branch performs no local dispatch-table lookup, but this does not establish null-object support: dependency requirements remain unknown. The object and relevant storage must remain valid through every operation actually performed.

## Behavior

### Ordered recognition

The comparison dependency is called with the original selector and one candidate label. A zero result recognizes that candidate; any nonzero result advances to the next candidate.

The order is:

1. `HandleMenuOnKeyboard`
2. `EnableSearchListHilite`
3. `HandleSelect`
4. `HandleHilite`
5. `HandlePlayPause`
6. `HandleAddToOTG`
7. `HandleRemotePlayPauseForVideo`
8. `HandleShowContextualMenu`

The first zero result ends recognition. Later labels are not compared, even if the recognized branch suppresses its handler.

If candidate number \(n\) is recognized, exactly \(n\) comparison calls occur on a normally progressing invocation. If none is recognized, all eight occur.

This establishes a zero/nonzero dispatch convention, not textual comparison semantics. Case sensitivity, encoding, termination requirements, malformed-input handling, and whether multiple labels could compare equal to one selector are unknown. If multiple comparisons would return zero, only the earliest reached one controls dispatch.

### Sentinel reads and their ordering

The following describes accesses expressed by the supplied source representation. It does **not** establish measured machine-level load counts, atomicity, or synchronization.

**Sentinel A:**

- After the first comparison returns, sentinel A is read and its value is captured, regardless of the comparison result.
- If the first candidate matched, sentinel A is read again for the suppression comparison. The earlier capture is not used for that branch.
- If the first candidate did not match but the second did, suppression uses the capture taken before the second comparison. There is no additional A read in the second branch.

Consequently, a first-candidate match expresses two A reads. Every other normally progressing path expresses one.

**Sentinel B:**

- B is not read on a first- or second-candidate match.
- If the first two comparisons are nonzero, the third comparison occurs. Immediately after it returns, sentinel B is read and captured, whether the third comparison matched or not.
- If the third candidate matched, sentinel B is read again for the suppression comparison. The earlier B capture is not used for that branch.
- If the third comparison was nonzero, candidates four through eight use that captured B value. Later comparison calls do not refresh it.

Consequently, a third-candidate match expresses two B reads. Paths reaching the third comparison but not matching it express one.

| Outcome | A reads expressed | B reads expressed | Suppression source |
|---|---:|---:|---|
| First candidate recognized | 2 | 0 | Second A read |
| Second candidate recognized | 1 | 0 | Captured A |
| Third candidate recognized | 1 | 2 | Second B read |
| Fourth through eighth candidate recognized | 1 | 1 | Captured B |
| No candidate recognized | 1 | 1 | No suppression comparison |

No dependency call is expressed between the paired A reads on the first branch or between the paired B reads on the third branch. Nevertheless, the evidence provides no stability or synchronization guarantee permitting those reads to be treated as necessarily equal.

Comparison side effects also matter. The first comparison precedes the A capture; the third precedes the B capture. Changes during those calls can therefore affect the respective captures. A change during the second comparison does not refresh captured A. A change during comparisons four through eight does not refresh captured B.

### Handler selection and suppression

| Recognized label | Suppress when the input value equals | Unsuppressed handler |
|---|---|---|
| `HandleMenuOnKeyboard` | A’s additional direct read in this branch | Indirect entry at 0x150 |
| `EnableSearchListHilite` | A captured after comparison one | Indirect entry at 0x154 |
| `HandleSelect` | B’s additional direct read in this branch | F-5ff4771492d5 |
| `HandleHilite` | B captured after comparison three | F-e7680695fe4c |
| `HandlePlayPause` | B captured after comparison three | F-8ecf431a0d7f |
| `HandleAddToOTG` | B captured after comparison three | F-cb5d084192f5 |
| `HandleRemotePlayPauseForVideo` | B captured after comparison three | Indirect entry at 0x128 |
| `HandleShowContextualMenu` | B captured after comparison three | F-5e39cda56b97 |

For every recognized branch:

- Equality suppresses the handler and produces result 1.
- Inequality invokes the selected handler with the original object reference followed by the original value.
- If the handler returns normally, the wrapper returns 1 regardless of any handler result.
- The wrapper invokes no second handler during that invocation.

If all comparisons return nonzero, the wrapper returns 0 without invoking a handler. The A and B captures still occur at their stated points.

### Indirect dispatch

For each indirect branch, suppression is decided before the object’s dispatch-table address and selected callable entry are read.

An unsuppressed indirect branch reads the object’s first word, reads the selected entry from the resulting table, and invokes that entry. No table address or target is visibly cached before selector evaluation.

Therefore, if a comparison changes the object’s table field and execution remains valid, the later lookup uses the field as subsequently read. No object-null check, table-bounds check, callable validation, or fallback target is expressed.

## Side effects

### Directly established effects

On normally progressing paths, the wrapper:

- Calls the comparison dependency between one and eight times.
- Reads sentinel A once or twice, according to the selected branch.
- Reads sentinel B zero, once, or twice, according to the selected branch.
- Reads the object’s table field and one callable entry only for an unsuppressed indirect dispatch.
- Invokes at most one handler.
- Returns 0 or 1.

These read counts describe the supplied source representation, not a runtime measurement or a guarantee about generated memory transactions.

No explicit outward write to the object, sentinel locations, or selector storage is visible. Capturing sentinel values changes only local state within the wrapper.

### Dependency-mediated effects

Comparison calls are observable dependency calls, not established pure lookups. They may affect global state or other accessible storage. A behavioral replacement must not assume that eliminating or reordering them preserves behavior.

Handler effects are also unknown. The wrapper does not establish that handlers preserve the object, preserve globals, retain no references, release no memory, or avoid reentrancy.

The wrapper’s absence of explicit outward writes does not make the invocation side-effect-free.

### Errors and abnormal completion

No explicit error result or recovery branch is provided for:

- Invalid selector representation.
- Inaccessible sentinel storage.
- Invalid object or dispatch-table storage.
- An unusable indirect target.
- A dependency that fails to return normally.

Result 0 means only that all comparisons returned nonzero. It is not an established general error code.

If a dependency prevents normal continuation, subsequent reads, handler execution, and a final recognition result are not guaranteed.

## Dependencies

### Comparison dependency

**Identity: documented_only; no opaque function ID supplied.**

It receives the original selector and the current candidate label. Only the wrapper’s interpretation of its result is established: zero selects, nonzero continues.

Its detailed input contract, result width, comparison semantics, ownership behavior, global effects, and abnormal-completion behavior remain unknown.

### Direct handlers

| Opaque function ID | Invocation condition |
|---|---|
| F-5ff4771492d5 | Third candidate recognized; value differs from the additional B read |
| F-e7680695fe4c | Fourth candidate recognized; value differs from captured B |
| F-8ecf431a0d7f | Fifth candidate recognized; value differs from captured B |
| F-cb5d084192f5 | Sixth candidate recognized; value differs from captured B |
| F-5e39cda56b97 | Eighth candidate recognized; value differs from captured B |

Each receives the original object reference and original 32-bit value. Any returned value is ignored.

### Indirect targets and global storage

**Indirect target identities: documented_only.** The three targets are selected through table offsets 0x150, 0x154, and 0x128. Their concrete identities and behavioral contracts are not supplied.

Sentinel A and sentinel B are required readable global storage on the paths described above. Their writers, synchronization, and lifetime guarantees are not supplied.

No dependency is assumed to preserve globals or to have particular memory-ownership behavior.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless a case states otherwise, required memory is accessible and dependencies return normally. Examples involving differing successive sentinel values describe conditional observations; they do not assert that concurrent modification is supported.

### First candidate uses the additional A read

Suppose the first comparison returns zero. The initial A capture yields 7, and the additional A read used for suppression yields 9.

- With value 9, the handler is suppressed and the result is 1.
- With value 7, the handler at offset 0x150 is invoked, provided the indirect call is valid; normal completion returns 1.
- Only one comparison occurs.
- Two A reads are expressed, and B is not read.

Equality with the initial A capture alone does not establish suppression.

### Second candidate retains captured A

Suppose the first comparison is nonzero and A is captured as 7. The second comparison changes A to 9 and returns zero.

- Value 7 suppresses the handler.
- Value 9 invokes the handler at offset 0x154.
- Both normally completing cases return 1.
- A is not reread for suppression, and B is not read.

### Third candidate uses the additional B read

Suppose the first two comparisons are nonzero and the third returns zero. The B capture after that comparison yields 12, while the additional B read yields 13.

- Value 12 invokes F-5ff4771492d5.
- Value 13 suppresses it.
- Both normally completing cases return 1.
- One A read and two B reads are expressed.

A value equal to the initial B capture does not establish suppression when the additional read differs.

### Later candidate retains captured B

Suppose the first three comparisons are nonzero, and B is then captured as 20. The fourth comparison changes B to 21 and returns zero.

- Value 20 suppresses F-e7680695fe4c.
- Value 21 invokes it.
- Both normally completing cases return 1.
- No additional B read is expressed.

### Remote-playback dispatch

Suppose the first six comparisons are nonzero and the seventh is zero. If the value differs from captured B, the wrapper reads the object’s table address, reads the callable entry at offset 0x128, and calls it with the original object and value.

The eighth comparison does not occur. Normal completion returns 1.

### Handler result is not the wrapper result

Suppose the fifth candidate is recognized and not suppressed. If F-8ecf431a0d7f returns normally, the wrapper returns 1 regardless of the dependency’s returned value.

### Unrecognized selector

If all eight comparisons return nonzero:

- All eight occur in order.
- One A read follows comparison one.
- One B read follows comparison three.
- No handler or local dispatch-table lookup occurs.
- The result is 0.

Equal numerical contents in A and B do not merge these distinct, differently timed source-level reads.

## Uncertainties

The following remain **documented_only**:

- Selector representation and detailed comparison semantics.
- Sentinel values, domain meanings, initialization, and writers.
- Whether repeated reads can validly observe different values in the execution environment.
- Synchronization, atomicity, memory ordering, and concurrency guarantees.
- Complete object layout, ABI, pointer width beyond the represented model, and callable-entry representation.
- Comparison dependency identity and effects.
- Concrete indirect target identities.
- Dependency mutation, ownership, retention, release, error, and reentrancy behavior.
- Behavior for invalid storage or abnormal dependency completion.
- Exact generated memory-access behavior.

The sentinel equality conditions are established suppression rules. Interpreting them as “disabled,” “invalid,” or “no selection” would be inferred domain meaning without supporting evidence.

No runtime tests have been performed.

## Evidence

This contract is based on the supplied static function representation and opaque direct-dependency mapping.

Established static evidence comprises:

- Eight ordered comparison calls with zero-result recognition.
- An A capture after comparison one, plus an additional A read when candidate one matches.
- A B capture after comparison three when reached, plus an additional B read when candidate three matches.
- Use of captured A for candidate two and captured B for candidates four through eight.
- Equality-based suppression.
- Five mapped direct handlers and three indirect table offsets.
- Unchanged forwarding of the object reference and 32-bit value.
- Ignored handler results and explicit normal return values 0 and 1.

The supplied material contains no dependency implementations, runtime traces, synchronization contract, or ownership specification. Application intent and behavior beyond those visible operations remain explicitly unresolved.
