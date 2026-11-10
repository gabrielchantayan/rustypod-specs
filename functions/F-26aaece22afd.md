<!-- Trusted provenance: F-26aaece22afd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-26aaece22afd

## Purpose

F-26aaece22afd processes fixed-stride records belonging to a mutable object. Each entered iteration either:

- Temporarily adjusts an object field, calls a fallback dependency, applies a compensating subtraction, and passes an opaque value through a dependency chain; or
- Prepares two four-word tuples, configures an embedded context, and submits the current record once or twice.

The function has no explicit return value. Its observable behavior consists of reads, direct writes, dependency calls, and effects produced by those dependencies.

The tuples and repeated submissions suggest positioning or rendering work. That is **inferred intent**, not an established semantic contract. The supplied evidence does not define tuple coordinates, output media, or dependency internals.

## Interface

### Arguments and result

| Input | Established role |
|---|---|
| Object | Pointer to mutable storage containing fields, an embedded context, and records |
| Argument A | Opaque 32-bit value passed to the indirect object operation and fallback dependency |
| Argument B | Opaque 32-bit value passed to the fallback dependency |

The function returns no explicit result or status.

The supplied representation uses four-byte words, 32-bit integer operations, and pointers represented in that environment. It does not establish an interface for wider pointers.

### Directly accessed object layout

Offsets are byte offsets from the object address.

| Offset | Use |
|---|---|
| `0x000` | Dispatch-table pointer |
| `0x048` | Flags; bit `0x08` controls configuration-input selection |
| `0x080` | Address stored into reusable local pointer storage before submissions |
| `0x084` | Field adjusted before and after the fallback dependency |
| `0x0A4` | Embedded context supplied to dependencies |
| `0x23C` | Adjustment amount used on the fallback path |
| `0x244` | Signed 32-bit record count, also used in tuple calculations |
| `0x254` | First record |
| `0x464` | Value subtracted when positioning the second tuple |

Each record occupies a stride of `0x6C` bytes. For zero-based index \(j\), the record address is:

\[
\text{object}+0x254+0x6Cj
\]

The following record-relative locations are used:

| Offset | Use |
|---|---|
| `0x044` | Address passed to F-19fde5f06ccc |
| `0x048` | Word supplied to the first tuple transformation and reread for an optional submission |
| `0x050` | Word read when preparing an optional submission |
| `0x058` | Word saved before the iteration’s selected path |
| `0x05C` | Word supplied to a transformation and reread after the first submission |

The indirect operation is obtained from dispatch-table byte offset `0x130`. It receives the object and Argument A.

These accesses do not establish complete object or record types, allocation boundaries, or ownership.

### Preconditions and errors

There are no explicit null checks, capacity checks, dispatch-target checks, allocation checks, or error-return branches.

Defined operation therefore requires accessible storage for every reached address, a callable dispatch target, and dependencies compatible with the supplied arguments. Local tuple buffers must be usable as four consecutive 32-bit words.

A positive count is not checked against available record storage. A nonpositive count prevents iteration only when observed by the loop condition; it does not suppress initialization or necessarily terminate an iteration already entered.

## Behavior

### Initialization

The following calls occur before the first record-count test:

1. F-88dd67d704e5 receives the object and its embedded context at `0x0A4`.
2. F-412e4635fdb0 receives the object and writable storage for a four-word base tuple.
3. The function calculates a retained comparison value from that tuple.
4. The indirect object operation receives the object and Argument A.
5. If, and only if, the indirect result equals integer `1`, F-42382d1a1b3b receives the object and `1`.

Let the base tuple’s words, in storage order, be \(B_0,B_1,B_2,B_3\). The retained value is:

\[
E=B_3-B_1
\]

The function does not recompute \(E\) during iteration. An indirect result of zero, a negative value, or a positive value other than one does not trigger F-42382d1a1b3b.

### Iteration and branch selection

The index starts at zero and advances by one after each completed iteration. Every loop test reads the signed count at object offset `0x244` again. There is no entry-time count snapshot.

For each entered iteration:

1. The current record address is formed.
2. F-19fde5f06ccc receives the address at record offset `0x044`.
3. Its signed 32-bit result \(R\) is compared with \(E\).
4. If \(R\le E\), the count is read again for path selection.
5. The record word at `0x058` is saved before either selected path executes.

The path is:

| Condition | Selected path |
|---|---|
| \(R>E\) | Record processing |
| \(R\le E\), and the selection-time count is greater than one | Record processing |
| \(R\le E\), and the selection-time count is at most one | Fallback |

The subtraction-and-overflow conditions in the supplied representation implement these signed comparisons; they are not unsigned comparisons.

The count admitting the iteration and the count selecting its path can differ. In particular, F-19fde5f06ccc executes between those reads. If it changes the count to zero or a negative value, the current iteration still proceeds according to the selection above.

### Fallback path

The fallback path performs these actions in order:

1. Reads the adjustment field at `0x23C` and the field at `0x084`, then stores their sum at `0x084`.
2. Calls F-f686e667bc1f with the object, Argument A, and Argument B.
3. Reads the then-current fields at `0x084` and `0x23C`, then stores their difference at `0x084`.
4. Calls F-a66a9be680df with no explicit arguments.
5. Passes its result and `0x0C` to F-c5a232e4455b.
6. Passes that result, the fallback-chain global operand, zero, and zero to F-afb630a7107c.
7. Calls F-8e50f265e4fa with the object, the resulting opaque value, and zero.

The final subtraction is not a restoration from a saved original field value. Changes made by F-f686e667bc1f to either participating field affect the result.

This path does not prepare the per-record tuples or perform the context submissions described below. It does not return early: normal loop progression follows. Dependency-driven count changes can permit further iterations.

### First tuple

On the record-processing path, a local tuple \(S\) receives all four words of the base tuple.

F-d156b0b83eb3 then receives:

1. \(S\), the current record word at `0x048`, and zero.
2. The same \(S\), the current record word at `0x05C`, and zero.

The `0x05C` read occurs after the first transformation returns. It is not an iteration-entry snapshot.

The transformation’s coordinate system and exact mutation rules are unknown. The caller supplies writable tuple storage; it does not establish that any particular component remains unchanged.

### Second tuple and conditional adjustment

A separate local tuple \(D\) receives all four words of the base tuple.

F-d156b0b83eb3 receives \(D\), zero, and:

\[
j(B_2-B_0)-\text{object}[0x464]
\]

After that call, the function uses the resulting tuple to calculate:

\[
W=D_2-D_0
\]

\[
T=W(\text{count}-1)
\]

The count here is a read made after the transformation.

An additional F-d156b0b83eb3 call occurs exactly when both conditions hold:

\[
-D_0\ne T
\qquad\text{and}\qquad
D_0\le -T
\]

For that call, the arguments are \(D\), zero, and:

\[
\text{newly read count}\times(D_2-D_0)
\]

The count is read again rather than reused from the calculation of \(T\). The width expression is also evaluated again from the tuple.

Equality in the first condition excludes the additional transformation. This exclusion remains significant even when the second comparison would hold.

### Configuration-input read and selection

After the second-tuple work, including the conditional transformation if taken, the function performs the following ordered operations:

1. **Unconditionally loads the configuration-pointer global.**
2. Reads the flags word at object offset `0x048`.
3. Selects the configuration input using bit `0x08`.

If bit `0x08` is set, the selected input is the pointer value already loaded from the global.

If bit `0x08` is clear, the function calls F-1ef14c326cd5 with writable local pointer storage and the current record. It then replaces the selected input with the **address of that local pointer storage**, not simply the pointer value written into it.

Thus, the configuration-pointer global is read on both branches. On the clear-bit branch, that global read precedes both the flag read and F-1ef14c326cd5, even though its loaded value is not ultimately passed to the configuration dependency.

The function next calls:

1. F-ccb5d9751852 with the embedded context and selected input.
2. F-33ca0546dba1 with the embedded context and `0x23`.

There is no evidence that the configuration global must remain unchanged after its load, nor that either dependency preserves global state.

### First submission

After configuration, the reusable local pointer storage is assigned the address of object offset `0x080`.

F-b774eb6cb163 then receives five explicit arguments:

1. Embedded context.
2. Current record.
3. Address of \(S\).
4. Address of \(D\).
5. `0xFF`.

The local-pointer assignment is visible, but the supplied call representation does not identify it as an additional submission argument. It must not be promoted into a sixth argument.

### Optional second submission

After the first submission returns, the function reads record offset `0x05C` and calculates:

\[
H=E-(Q-\text{record}[0x05C])
\]

Here \(Q\) is the record word saved from `0x058` before path selection completed.

A second submission requires:

- \(H>0\); and
- A further read of record offset `0x05C` yielding a nonzero value.

The second read is conditional on \(H>0\). The calculation read and eligibility read are distinct; no stable-record assumption is established.

When eligible, preparation proceeds in this order:

1. \(S_1\) receives the current record word at `0x048`.
2. The reusable local pointer storage receives the address of object offset `0x080`.
3. \(S_3\) receives the current record word at `0x050`.
4. \(D_1\) receives \(B_3-H\).
5. \(D_3\) receives \(B_3\).

F-b774eb6cb163 is then called with the same five argument roles and `0xFF`.

Other tuple components are not reset. Their values include any modifications made through the tuple pointers by earlier calls. Configuration selection and the `0x23` call are not repeated.

### Integer interpretation

The visible scalar arithmetic is 32-bit. No widening, saturation, or explicit overflow rejection is shown. Signed ordering is used for the loop, path selection, conditional tuple adjustment, and \(H>0\).

The formulas above describe operand relationships and evaluation stages, not an unlimited-precision arithmetic guarantee. Arithmetic extremes, including overflowing products and negation of the minimum signed value, require a target arithmetic interpretation not independently established here. The acceptance examples avoid overflow.

## Side effects

The only direct writes to object fields are the fallback-path writes at `0x084`. Other direct writes initialize or modify local tuple and pointer storage.

The record-processing path includes an unconditional configuration-pointer global read before the flag read, including when the clear-bit path subsequently discards the loaded pointer. The fallback chain separately uses another global operand.

Dependencies receive addresses into the object, records, embedded context, and local storage. Their effects are not restricted by the visible caller. In particular, the contract does not assume preservation of counts, flags, record contents, globals, or writable tuples.

No allocation, release, ownership transfer, or retained-pointer lifetime policy can be established from the dependency names or argument flow. Local addresses are available during the call; their safe retention beyond the invocation is not established.

There is no visible recovery mechanism if a dependency fails, does not return, or accesses invalid storage. The compensating subtraction is reached only if the fallback dependency returns normally.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-88dd67d704e5 | Initial call with object and embedded context |
| F-412e4635fdb0 | Receives writable base-tuple storage |
| F-42382d1a1b3b | Receives object and `1` only after indirect result `1` |
| F-19fde5f06ccc | Receives record-relative address `0x044`; returns comparison value |
| F-f686e667bc1f | Receives adjusted object and both opaque arguments |
| F-a66a9be680df | Starts the fallback opaque-value chain |
| F-c5a232e4455b | Receives preceding result and `0x0C` |
| F-afb630a7107c | Receives preceding result, global operand, and two zeros |
| F-8e50f265e4fa | Receives object, final chained value, and zero |
| F-d156b0b83eb3 | Receives writable four-word tuple and two scalar inputs |
| F-1ef14c326cd5 | Receives local pointer storage and record on the clear-bit path |
| F-ccb5d9751852 | Receives embedded context and selected pointer input |
| F-33ca0546dba1 | Receives embedded context and `0x23` |
| F-b774eb6cb163 | Receives context, record, two tuple addresses, and `0xFF` |

No opaque ID is supplied for the indirect dispatch target. Only its dispatch offset, arguments, and exact-result comparison are established.

## Acceptance examples

These are conditional deductions from static evidence, not execution reports. Unless stated otherwise, dependencies return normally and leave unrelated inputs unchanged.

1. **No entered iterations.** If the first loop-condition count is zero or negative, initialization and indirect dispatch still occur. No per-record call occurs.

2. **Exact indirect result.** Result `1` triggers F-42382d1a1b3b with `1`. Result `2` does not.

3. **Comparison boundary.** With \(R=E\), a selection-time count of `1` chooses fallback; a count of `2` chooses record processing.

4. **Count changed during comparison dependency.** If a positive count admits an iteration and F-19fde5f06ccc changes it to zero, \(R\le E\) selects fallback. The current iteration is not cancelled.

5. **Non-restoring subtraction.** If `0x084` is initially `100` and `0x23C` is `7`, the pre-call write produces `107`. If the fallback dependency changes only `0x23C` to `9`, the subsequent direct write produces `98`.

6. **Adjustment equality exclusion.** If the post-transformation tuple has \(D_0=-20\) and \(T=20\), no additional transformation occurs because \(-D_0=T\).

7. **Set configuration bit.** After second-tuple preparation, the configuration-pointer global is loaded before the flags. With bit `0x08` set, F-1ef14c326cd5 is skipped and F-ccb5d9751852 receives the previously loaded pointer.

8. **Clear configuration bit.** The same global load still occurs before the flags and before F-1ef14c326cd5. F-ccb5d9751852 instead receives the address of local pointer storage. A global change made by F-1ef14c326cd5 does not introduce a second configuration-global load in this function.

9. **Second-submission boundary.** If \(H=0\), there is no second submission and no eligibility reread of `0x05C`. If \(H>0\) but that reread is zero, there is also no second submission.

10. **Mutation after first submission.** A first-submission change to record offset `0x05C` affects the subsequent \(H\) calculation. It does not replace the previously saved value from `0x058`.

## Uncertainties

- **documented_only — Tuple semantics:** Coordinate meanings, units, transformation rules, clipping, and preserved components are unknown.
- **documented_only — Submission semantics:** Output appearance, meanings of `0x23` and `0xFF`, and effects beyond visible call order are unknown.
- **documented_only — Globals:** Global contents, pointee layouts, ownership, volatility, and mutation rules are not supplied. The configuration-global read’s position is established independently of those unknowns.
- **documented_only — Ownership and lifetime:** Opaque returned values and passed pointers have no supplied ownership or retention contract.
- **documented_only — Arithmetic extremes:** No portable overflow policy is established beyond the visible 32-bit operations and signed comparisons.
- **documented_only — Failure behavior:** Dependency exceptions, nonlocal exits, invalid-pointer behavior, and external error reporting are unspecified.
- **documented_only — Concurrency:** No synchronization or atomicity contract is visible. Separate reads are documented without assuming concurrent mutation is safe.
- **Inferred intent:** Rendering or repeated-layout behavior is plausible but does not define dependency semantics.

## Evidence

Established behavior is derived from the supplied static function representation and opaque dependency mapping: argument flow, widths, object and record offsets, local-buffer writes, conditional branches, global accesses, and call order.

In particular, the configuration-pointer global load is unconditional and precedes the flag read; only the subsequent selection of the loaded pointer versus local-storage address is conditional.

No dependency implementations, complete type definitions, runtime traces, or external behavioral guarantees were supplied. No runtime tests have been performed. Acceptance examples are conditional static deductions, not observed outcomes.
