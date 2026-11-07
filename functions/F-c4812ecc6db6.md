<!-- Trusted provenance: F-c4812ecc6db6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c4812ecc6db6

## Purpose

F-c4812ecc6db6 selects a named layout request from receiver state and dependency results, prepares that request, and submits it through one of two dispatch routes.

**Four request names are observable:**

- `GotoDefaultLayout`
- `GotoDefaultVideoLayout`
- `GotoStatusBarVideoLayout`
- `GotoDefaultSubtitlesLayout`

The two video requests use a receiver virtual operation for dispatch. The default and subtitles requests use F-97dbacdc5cbc. Every normally completed route subsequently calls F-8a5aadac202f on the local request-storage address supplied to preparation.

The request names suggest user-interface layout transitions. That is inferred intent. The visible body establishes request selection and submission, not an actual layout change, its timing, or its success.

## Interface

### Arguments and result

The visible interface accepts four arguments:

| Argument | Established use |
|---|---|
| Receiver | Address of an object whose fields and virtual-operation table are accessed |
| Second argument | 32-bit payload with no visible use |
| Third argument | 32-bit payload copied into local auxiliary storage |
| Fourth argument | 32-bit payload copied into separate local request storage |

The third-argument copy is passed by address to F-9b9c1c7483f4 only for default and subtitles requests. The fourth-argument copy is passed by address to F-9b9c1c7483f4 on every route and to F-8a5aadac202f after dispatch.

These addresses refer to invocation-local copies, not the caller’s argument storage. Dependencies receiving those addresses may modify the copies according to their unknown contracts.

There is no explicit return value and no visible success or failure result.

### Widths and layout

The supplied representation uses 32-bit words and represents addresses through 32-bit values. The following offsets are measured in bytes from the receiver address:

| Offset | Access | Role |
|---|---|---|
| `0x00` | Operation-table address | Resolves receiver virtual operations |
| `0xB0` | 32-bit word | Argument to F-b7dc2cfb1085 |
| `0xC8` | 32-bit word | Optional-operation gate and eventual argument to F-0683c3467ba0 |
| `0xE0` | 32-bit word narrowed to eight bits | Selects video versus non-video processing |
| `0xE1` | One byte | Initial gate for specialized processing |

The access at `0xE0` is a word read followed by narrowing. Selection depends only on the resulting low eight numerical bits, not on whether the complete word is nonzero. A nonzero value in the word’s upper 24 bits cannot by itself select video processing.

The word access at `0xE0` overlaps the addressed byte at `0xE1`. Their numerical relationship depends on byte order. On a little-endian target, the word’s low eight bits occupy byte `0xE0`, not byte `0xE1`. Target byte order is not independently established here.

The virtual-operation offsets are byte offsets within operation tables:

- Receiver operation at `0x11C`: video-request dispatch.
- Receiver operation at `0x13C`: eligibility query for an optional call.
- Operation at `0x13C` on the object returned by F-b7af80e829ec: specialized-processing gate.

The latter two operations occupy the same table offset on potentially different objects. Their argument expressions and result tests differ; they must not be treated as interchangeable.

### Preconditions

Normal execution requires every reached memory access and call target to be valid:

- The receiver supports the initial byte read at `0xE1`.
- Any subsequently reached receiver fields are readable at their stated widths.
- Every reached operation-table pointer and operation entry is usable.
- F-b7af80e829ec returns an object whose operation table can be accessed. There is no visible null check.
- Dependencies accept the values and local-storage addresses provided.

The body provides no independent validation, error reporting, or recovery boundary. These are path-dependent requirements: an early default selection does not require the later specialized fields or virtual operations to be accessed.

## Behavior

### Entry and initial gate

The function first copies the third argument into local auxiliary storage, then copies the fourth argument into local request storage. Both copies are 32 bits.

It next reads the receiver byte at `0xE1`.

If that byte is zero, the selected request is `GotoDefaultLayout`. This route does not call F-b7af80e829ec, read the video-selection word at `0xE0`, or execute the optional-operation sequence.

If the byte is nonzero:

1. F-b7af80e829ec is called without visible explicit arguments.
2. The returned object’s operation-table address is read.
3. The operation at table offset `0x13C` is invoked without visible explicit arguments.
4. Its result is compared with exactly `1`.

Only a result of exactly `1` permits specialized processing. Zero, `2`, negative values, and all other represented results select `GotoDefaultLayout`.

The absence of an explicit argument in this virtual call does not establish its complete calling convention or the absence of an implicit receiver.

### Video versus non-video selection

After the specialized gate succeeds, the function reads the receiver word at `0xE0` and narrows it to eight bits:

- Nonzero narrowed value: video processing.
- Zero narrowed value: non-video processing.

This read occurs after the returned-object virtual call. It is not an entry-state snapshot. If an earlier dependency changes the receiver, the subsequent access observes the state available when that access occurs.

### Optional-operation sequence

A common conditional sequence appears on two routes: default-video processing and specialized non-video processing.

First, the receiver word at `0xC8` is read and compared with zero.

- If zero, neither the receiver eligibility operation nor F-0683c3467ba0 is called.
- If nonzero, the receiver operation at table offset `0x13C` is resolved and invoked with the receiver as its visible argument.

The receiver operation’s result uses a zero-versus-nonzero test:

- Zero skips F-0683c3467ba0.
- Any nonzero result causes another read of the receiver word at `0xC8`, followed by F-0683c3467ba0 with that newly read value.

The second read is required behavior. The value passed to F-0683c3467ba0 need not equal the value that initially passed the nonzero test.

There is no second nonzero check. If the receiver operation changes the field to zero and returns nonzero, F-0683c3467ba0 receives zero.

### Video processing

The video branch reads the receiver word at `0xB0` and passes it to F-b7dc2cfb1085.

If F-b7dc2cfb1085 returns zero:

1. Execute the optional-operation sequence.
2. Select `GotoDefaultVideoLayout`.

If F-b7dc2cfb1085 returns nonzero:

1. Skip the optional-operation sequence entirely.
2. Select `GotoStatusBarVideoLayout`.

Both video requests then use the same submission sequence:

1. Call F-9b9c1c7483f4 with the local request-storage address and the selected name.
2. Resolve the receiver operation at table offset `0x11C`.
3. Invoke that operation with the receiver and the 32-bit result returned by F-9b9c1c7483f4.
4. Call F-8a5aadac202f with the local request-storage address.
5. Return normally.

The receiver dispatch operation is resolved after preparation returns. Its target is not captured before F-9b9c1c7483f4.

Video preparation does not visibly receive the auxiliary-storage address. Neither video route calls F-97dbacdc5cbc.

### Non-video processing

The specialized non-video branch performs these actions in order:

1. Execute the optional-operation sequence.
2. Call F-7e4f952e879e without visible explicit arguments.
3. Call F-a664bdd89f4f without visible explicit arguments.
4. Compare the latter result with exactly `1`.

Exactly `1` selects `GotoDefaultSubtitlesLayout`. Every other result selects `GotoDefaultLayout`.

The optional-operation decision and F-7e4f952e879e occur before the subtitles decision. Selecting the default request does not undo them.

### Default and subtitles submission

Every selection of `GotoDefaultLayout` or `GotoDefaultSubtitlesLayout` uses this sequence:

1. Call F-9b9c1c7483f4 with the local request-storage address, selected name, and local auxiliary-storage address.
2. Call F-97dbacdc5cbc with the receiver and the 32-bit preparation result.
3. Call F-8a5aadac202f with the local request-storage address.
4. Return normally.

Thus, `GotoDefaultLayout` has multiple selection paths but one visible submission route. It may be selected immediately, after the specialized gate fails, or after non-video processing has already produced effects.

The visible argument list for F-9b9c1c7483f4 differs between video and default/subtitles routes. This distinction is part of the contract; it must not be normalized into an assumed fixed-arity interface.

## Side effects

The function directly initializes two local 32-bit storage locations. It makes no visible direct write to receiver fields or global storage.

Dependencies and virtual operations may change receiver state, globals, resources, or the local storage passed to them. Their implementations are unavailable, so preservation must not be assumed.

Observable ordering constraints include:

- Local-copy initialization precedes the initial receiver-byte read.
- The initial receiver-byte read precedes every dependency call.
- The video-selection word is read only after the specialized gate returns exactly `1`.
- The `0xB0` read precedes F-b7dc2cfb1085.
- The first optional `0xC8` read precedes the receiver eligibility operation.
- When required, the second `0xC8` read follows that operation and supplies F-0683c3467ba0.
- Non-video processing calls F-7e4f952e879e before F-a664bdd89f4f.
- Preparation precedes dispatch.
- F-8a5aadac202f follows dispatch.

On each normally completed invocation, the visible body performs exactly one preparation call, one dispatch call, and one final F-8a5aadac202f call. These counts concern direct calls made by this body, not additional activity inside dependencies.

F-8a5aadac202f receives the same local address used as the first preparation argument. The contents at that address may have changed during preparation or subsequent dependency activity. Its final position suggests cleanup-like intent but does not establish destruction, deallocation, or ownership release.

No ownership transfer is established for the receiver, either local payload, the values read at `0xB0` or `0xC8`, or the preparation result. The passed local addresses are valid during this invocation; no safe post-return retention contract is established.

No visible payload arithmetic increments, scales, or combines values. Relevant numerical behavior consists of fixed-offset accesses, zero tests, exact comparisons with `1`, and narrowing from 32 bits to eight bits.

No explicit failure branch checks preparation or dispatch success. If a dependency fails to return normally, later dispatch or final-call behavior is not established by this body.

## Dependencies

| Dependency | Established interaction |
|---|---|
| F-b7af80e829ec | Returns the object used for the specialized-processing gate |
| Returned-object operation at `0x13C` | Allows specialized processing only when its result equals `1`; no explicit arguments are shown |
| F-b7dc2cfb1085 | Receives the receiver’s `0xB0` value; zero selects default video, nonzero selects status-bar video |
| Receiver operation at `0x13C` | Receives the receiver and gates F-0683c3467ba0 using a zero-versus-nonzero result |
| F-0683c3467ba0 | Receives the reread `0xC8` value when the optional gate succeeds |
| F-7e4f952e879e | Runs on the specialized non-video route before the subtitles query |
| F-a664bdd89f4f | Selects subtitles only when its result equals `1` |
| F-9b9c1c7483f4 | Receives request storage and the selected name; additionally receives auxiliary storage on default/subtitles routes; returns a 32-bit dispatch value |
| Receiver operation at `0x11C` | Receives the receiver and preparation result on video routes |
| F-97dbacdc5cbc | Receives the receiver and preparation result on default/subtitles routes |
| F-8a5aadac202f | Receives local request storage after dispatch |

Return values other than those explicitly used for decisions or forwarding do not influence the visible control flow. Their broader meaning remains unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Each assumes valid reached storage, valid call targets, and normally returning dependencies.

1. **Initial gate disabled.**  
   With receiver byte `0xE1` equal to zero, select `GotoDefaultLayout`. Prepare with both local-storage addresses, dispatch through F-97dbacdc5cbc, and call F-8a5aadac202f. Do not call F-b7af80e829ec or inspect `0xE0`.

2. **Specialized gate returns a different nonzero value.**  
   With the initial byte nonzero and the returned-object operation yielding `2`, select `GotoDefaultLayout`. Treating this result as a generic true value would be incorrect.

3. **Status-bar video.**  
   With both specialized gates satisfied, a nonzero narrowed `0xE0` value, and a nonzero F-b7dc2cfb1085 result, select `GotoStatusBarVideoLayout`. Skip the optional sequence and dispatch through receiver operation `0x11C`.

4. **Default video without an optional value.**  
   With video selected, F-b7dc2cfb1085 returning zero, and the subsequent `0xC8` read yielding zero, select `GotoDefaultVideoLayout`. Do not invoke receiver operation `0x13C` for the optional sequence or F-0683c3467ba0.

5. **Optional eligibility denied.**  
   If the first `0xC8` read is nonzero but the receiver eligibility operation returns zero, omit F-0683c3467ba0. Continue with the request selection prescribed by the enclosing branch.

6. **Optional value changes during eligibility.**  
   If the first `0xC8` read is nonzero and the receiver operation changes that field to zero while returning nonzero, F-0683c3467ba0 receives zero. Reusing the earlier value would violate the contract.

7. **Subtitles selected.**  
   With specialized non-video processing selected and F-a664bdd89f4f returning `1`, select `GotoDefaultSubtitlesLayout`. The optional sequence and F-7e4f952e879e have already occurred. Prepare with both local-storage addresses and dispatch through F-97dbacdc5cbc.

8. **Subtitles rejected after earlier effects.**  
   Under the same branch conditions, a result of `0` or `2` from F-a664bdd89f4f selects `GotoDefaultLayout`. Earlier non-video calls remain part of the observable sequence.

9. **Upper bits do not select video.**  
   If the specialized gates pass and the word read at `0xE0` has nonzero upper bits but zero low eight bits, select non-video processing. Testing the full word for nonzero would be incorrect.

10. **Preparation changes the video dispatch target.**  
    If F-9b9c1c7483f4 changes the receiver’s operation table to another valid table, video dispatch uses the subsequently resolved entry at `0x11C`, not an entry saved before preparation.

## Uncertainties

The following remain **documented_only**:

- Exact source-level types, character signedness, target byte order, alignment requirements, and complete calling conventions.
- Whether the differing preparation argument lists represent a variable-argument interface or incomplete signature information.
- The complete receiver convention for the returned-object virtual operation.
- Field meanings beyond the reads, comparisons, and forwarding described here.
- Whether request preparation constructs, formats, allocates, or references an object.
- Whether dispatch changes a layout, queues work, or performs another action.
- Allocation, ownership, reference-counting, retention, and release behavior.
- Dependency effects on receiver state, global state, or other reachable storage.
- Behavior on invalid addresses, dependency failures, or abnormal control transfer.
- Whether any finalization occurs after a dependency fails to return normally.

These unknowns do not weaken the established branch conditions, argument expressions, access widths, or ordering requirements.

## Evidence

The supplied static body establishes four request-name selections, the receiver access offsets, local 32-bit copies, exact-one versus nonzero comparisons, the repeated `0xC8` read, the two preparation argument lists, the dispatch split, and the final common call.

Interpreting the requests as layout transitions and the final call as cleanup is inferred intent only. Dependency implementations and externally visible layout outcomes are not established.

No runtime execution, smoke checks, or tests have been performed.
