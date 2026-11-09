<!-- Trusted provenance: F-ed044fef08a3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed044fef08a3

## Purpose

F-ed044fef08a3 initializes an embedded record in a caller-supplied context and repeatedly processes the record designated by a mutable cursor. Each iteration classifies four signed endpoints, selects a processing path, and checks whether the cursor has moved below the initial record’s address.

Established behavior includes:

- Testing whether one closed endpoint range is contained within another.
- Handling failed containment before testing for equal outer endpoints.
- Selecting direction 1 or 2 from the ordering of the outer endpoints.
- Conditionally invoking mode-transition dependencies.
- Returning 1 immediately when a checked dependency returns nonzero.
- On completion, writing two values cached from the last examined record into persistent context fields and returning 0.

A geometric or ordered-range-processing purpose is inferred intent only. The endpoint units, meaning of the opaque words, and application-level purpose of the modes are not established.

## Interface

### Arguments and result

The function receives seven arguments. The supplied representation uses a 32-bit address model and four-byte record words.

| Argument | Established use |
|---|---|
| A1 | Address of the mutable context. |
| A2 | Four-byte opaque word stored at context offset 0xCC. |
| A3 | Four-byte word stored at 0xD0 and subsequently interpreted as a signed 32-bit endpoint. |
| A4 | Four-byte opaque word stored at 0xC4. |
| A5 | Four-byte word stored at 0xC8 and subsequently interpreted as a signed 32-bit endpoint. |
| A6 | Four-byte opaque word stored at 0xBC; potentially copied to persistent offset 0x48 on completion. |
| A7 | Four-byte word stored at 0xC0, interpreted as a signed 32-bit endpoint, and potentially copied to persistent offset 0x4C on completion. |

The four-byte result is:

- **0:** the cursor satisfies the completion comparison.
- **1:** a checked dependency returned nonzero.

A dependency’s actual nonzero result is not propagated. No ordinary return behavior is established for invalid memory accesses, nonreturning dependencies, or an execution that continues indefinitely.

### Context layout

Offsets are relative to A1.

| Offset | Width | Role |
|---|---:|---|
| 0x38 | 4 bytes | Mutable record cursor. |
| 0x48 | 4 bytes | Persistent opaque word; read during initialization and written on completion. |
| 0x4C | 4 bytes | Persistent endpoint word; read during initialization and written on completion. |
| 0x50 | 4 bytes | Argument value read when preparing direction-2 processing. |
| 0x54 | 4 bytes | Argument value read when preparing direction-2 processing. |
| 0x68 | 1 byte | Current mode, compared with zero and the selected direction. |
| 0xBC–0xDB | 32 bytes | Initial embedded record. |

The initial record contains:

| Record-relative offset | Initial value |
|---|---|
| +0x00 | A6 |
| +0x04 | A7 |
| +0x08 | A4 |
| +0x0C | A5 |
| +0x10 | A2 |
| +0x14 | A3 |
| +0x18 | Word read from context offset 0x48 |
| +0x1C | Word read from context offset 0x4C |

Although the record spans 32 bytes, the function’s explicit cursor adjustments are **24 bytes**. A replacement must not substitute a 32-byte stride.

### Preconditions and ownership

The context must support the reads and writes on the path taken. Initialization accesses fields through offset 0xDB. Each subsequent iteration requires readable words at offsets +0x00, +0x04, +0x0C, +0x14, and +0x1C relative to the cursor selected for that iteration.

No null check, storage-capacity check, alignment repair, or memory-fault recovery is established. The available storage around the embedded record is unknown.

No direct allocation, release, or ownership transfer is visible. Passing the context to a dependency does not establish whether that dependency retains it, releases associated storage, or preserves its validity. Continued execution requires the later accesses to remain valid; the supplied body does not enforce that requirement.

## Behavior

### Initialization and access order

Before the first iteration, the function performs the following ordered accesses:

1. Stores A1 + 0xBC into the cursor at offset 0x38.
2. Reads the word at 0x48 and stores it at 0xD4.
3. Reads the word at 0x4C and stores it at 0xD8.
4. Stores A2 at 0xCC.
5. Stores A3 at 0xD0.
6. Stores A4 at 0xC4.
7. Stores A5 at 0xC8.
8. Stores A7 at 0xC0.
9. Stores A6 at 0xBC.

Initialization is not transactional. These writes occur before any processing dependency can cause a failure return.

The address A1 + 0xBC is retained as the invocation’s completion boundary. Changing the cursor does not change that boundary. There is no completion check before the first record is examined.

### Record observation

At the start of each iteration, the function reads the cursor at context offset 0x38. It then reads four signed 32-bit endpoints from that record in this order:

1. +0x1C.
2. +0x04.
3. +0x14.
4. +0x0C.

For this contract, the **outer endpoints** are the values at +0x04 and +0x1C. The **inner endpoints** are the values at +0x0C and +0x14. These labels describe comparison roles, not established domain semantics.

The function selects the lower and upper values of each pair using signed comparisons. It then reads and caches the opaque word at +0x00 before selecting the processing branch.

The endpoint at +0x04 is also retained for possible completion writes. Both completion values therefore come from the current iteration’s observations, before any dependency call in that iteration.

Endpoint classification uses comparisons and selections, not subtraction or range-width arithmetic. Signed values from −2,147,483,648 through 2,147,483,647 participate in that ordering without an endpoint-difference calculation.

### Containment and branch precedence

Containment fails if either:

- The lower inner endpoint is less than the lower outer endpoint.
- The upper inner endpoint is greater than the upper outer endpoint.

Otherwise, the entire closed inner range is contained in the closed outer range. Equality at either boundary is accepted. Mere overlap is insufficient.

Reversing either pair’s order does not change containment. Reversing unequal outer endpoints does change direction selection.

The containment decision precedes the outer-endpoint equality decision. Equal outer endpoints therefore do not bypass failed containment.

### Failed containment

When containment fails:

1. F-7d08bc9687d6 is called, with no explicit arguments shown.
2. After it returns, the cursor at context offset 0x38 is read again.
3. The function adds 0x18 bytes to that reread value and stores the result back at 0x38.
4. Execution proceeds to the completion check.

The dependency’s return value is ignored. Its cursor changes are not ignored: the increment uses the cursor left after the call, not necessarily the record address captured at iteration entry.

This branch does not perform the function’s mode checks or direction-dependent processing.

### Contained, equal outer endpoints

If containment holds and the outer endpoints are equal, the function stores the iteration-entry record address minus 0x18 bytes into the cursor. It then performs the completion check without calling a dependency on this path.

Containment within an equal-endpoint outer range requires both inner endpoints to equal that same value. If either differs, failed-containment processing takes precedence.

This decrement uses the record address captured at iteration entry. It is distinct from the failed-containment branch’s post-call cursor reread.

### Direction and mode transition

For contained ranges with unequal outer endpoints:

- Direction is **2** if the endpoint at +0x04 is less than the endpoint at +0x1C.
- Direction is **1** if the endpoint at +0x04 is greater than the endpoint at +0x1C.

The function reads the mode byte at context offset 0x68.

If the first mode read equals the selected direction, both transition dependencies are skipped.

If it differs, the function reads the mode byte a second time:

- If the second read is nonzero, F-ecde43d0a2b1 is called with the context address. A nonzero result immediately returns 1.
- If the second read is zero, that dependency is skipped.

Unless this sequence has returned failure, F-dbf79ec942b4 is then called with the context address and selected direction. Its nonzero result immediately returns 1.

Thus, a second read of zero skips only F-ecde43d0a2b1, not F-dbf79ec942b4. Once the first read has established a mismatch, the second read is tested for nonzero, not reconsidered for equality with the direction.

The function itself does not write the mode byte. It does not validate the mode again after transition dependencies return.

### Direction-dependent processing and global read

After any required transition succeeds, or immediately when the mode already matches:

- **Direction 1:** F-d2e511bc1037 is called with no explicit arguments shown.
- **Direction 2:** F-ed9f0758543c is called with five arguments:
  1. The context address.
  2. Integer value 3.
  3. An opaque value read from a source global.
  4. The four-byte word read from context offset 0x50.
  5. The four-byte word read from context offset 0x54.

The global is **read when preparing the direction-2 call**. It is not an established immutable datum or a value captured during initialization.

The global read and both context-word reads occur after any mode-transition dependencies for that iteration have returned successfully. Their order relative to one another is unspecified by the supplied representation.

Unknown dependencies may modify that global. Consequently:

- A transition dependency may affect the third argument supplied to the ensuing direction-2 call.
- Earlier dependency calls may affect the value observed in later iterations.
- Equal argument values across calls or iterations are not guaranteed.
- The read must not be hoisted ahead of transition dependencies on an assumed-constant basis.

The global’s domain meaning and declaration width are not established by the supplied function body.

A nonzero result from either direction-processing dependency immediately returns 1. A zero result leads to the completion check.

### Completion and continuation

After every non-failing branch, the function reads the cursor at context offset 0x38 and compares it with the retained initial-record address.

If the cursor compares below that boundary, the function:

1. Writes the cached record word from +0x00 to context offset 0x48.
2. Writes the cached endpoint from +0x04 to context offset 0x4C.
3. Returns 0.

The write order is 0x48 followed by 0x4C. Dependency modifications to those record words do not replace the cached values used here.

If the cursor does not compare below the boundary, processing repeats with a fresh cursor read and fresh record observations.

On the unequal-endpoint processing paths, this function performs no explicit cursor adjustment. Progress on those paths depends on dependency effects or other changes not established by the visible body. Neither termination nor an iteration bound is guaranteed.

The exact machine-level address ordering and wrap behavior behind the displayed pointer operations remain unresolved. In particular, endpoint signedness must not be used to infer cursor-comparison signedness.

## Side effects

Direct writes comprise initialization, branch-specific cursor updates, and the two successful-completion writes. No direct endpoint-normalization writes are shown.

A return of 1:

- Preserves initialization writes rather than rolling them back.
- Does not undo dependency side effects.
- Bypasses the function’s successful-completion writes to 0x48 and 0x4C.
- Does not imply those persistent fields are unchanged, because dependencies may modify them.

Direction-2 processing additionally reads an opaque global while preparing its arguments. No direct write to that global is shown. Dependency effects on it remain unknown.

No locking, atomicity, concurrency, or reentrancy guarantee is established. The described read order is not a promise that arbitrary concurrent mutation is supported.

## Dependencies

| Opaque ID | Established invocation and result handling |
|---|---|
| F-7d08bc9687d6 | Failed containment. No explicit arguments shown. Result ignored; cursor reread afterward. |
| F-ecde43d0a2b1 | Required transition when the second mode read is nonzero. Receives context; nonzero causes return 1. |
| F-dbf79ec942b4 | Required transition after any preceding transition call succeeds. Receives context and direction; nonzero causes return 1. |
| F-d2e511bc1037 | Direction-1 processing. No explicit arguments shown; nonzero causes return 1. |
| F-ed9f0758543c | Direction-2 processing. Receives context, 3, a freshly read opaque global value, and words from 0x50 and 0x54; nonzero causes return 1. |

“No explicit arguments shown” does not establish absence of implicit inputs or side effects. None of these dependencies is established to preserve globals, context contents, cursor state, storage lifetime, or ownership.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless specified otherwise, memory accesses are valid, dependencies return normally, and no unmentioned external mutation occurs.

1. **Immediate degenerate completion.**  
   A7, the entry word at 0x4C, A3, and A5 all equal 9. Containment succeeds, and the equal-endpoint branch moves the cursor 24 bytes below the initial record. In a nonwrapping address region with ordinary address ordering, no dependency is called; A6 is written to 0x48, 9 is written to 0x4C, and the result is 0.

2. **Equality does not override containment failure.**  
   Outer endpoints 9 and 9 with inner endpoints 8 and 9 invoke F-7d08bc9687d6, not the equal-endpoint path.

3. **Overlap does not suffice.**  
   Outer endpoints 0 and 10 with inner endpoints −1 and 5 fail containment despite overlapping.

4. **Boundary equality and reversed inner order.**  
   Outer endpoints at +0x04 and +0x1C of 0 and 10, with inner endpoints 10 and 0, pass containment and select direction 2.

5. **Reversed outer order.**  
   Outer endpoints at +0x04 and +0x1C of 10 and 0, with inner endpoints 2 and 8, pass containment and select direction 1.

6. **Matching mode.**  
   Direction 1 with the first mode read equal to 1 skips both transition dependencies and calls F-d2e511bc1037.

7. **Zero-mode transition.**  
   Direction 2 with both mode reads zero skips F-ecde43d0a2b1 but calls F-dbf79ec942b4 with direction 2. Direction-2 processing follows only if that call returns zero.

8. **Failure short-circuit.**  
   If F-ecde43d0a2b1 returns nonzero during a required transition, the result is 1. Neither F-dbf79ec942b4 nor a direction-processing dependency is subsequently called.

9. **Post-call cursor increment.**  
   If failed-containment processing leaves the cursor at address X, the function stores X plus 24 bytes, subject to the unresolved address-arithmetic semantics. It does not necessarily increment the iteration-entry address.

10. **Cached completion values.**  
    If a successful processing dependency changes the examined record’s first two words and moves the cursor below the boundary, completion still writes the pre-call cached values.

11. **Signed endpoints.**  
    Endpoint bit pattern 0xFFFFFFFF participates as −1, not 4,294,967,295. Containment and direction must follow signed 32-bit ordering.

12. **Transition changes the global.**  
    If a successful transition dependency changes the opaque global from value G to value H, and nothing changes it afterward, the ensuing direction-2 call receives H as its third argument.

13. **Global read is not an entry snapshot.**  
    If two direction-2 iterations reach argument preparation with different global values, each call receives the value read for that call. The first call’s argument must not be reused merely because both calls belong to one invocation.

14. **Unspecified argument-read order.**  
    Acceptance must not require the global read to precede or follow the reads at 0x50 and 0x54. It may require all three to occur after successful transition dependencies.

## Uncertainties

The following remain **documented_only**:

- Domain meanings and units of endpoints, opaque words, and modes.
- The opaque global’s value, declaration width, interpretation, initialization, and mutation rules.
- Ordering among the global read and the two context reads used for direction-2 arguments.
- Dependency internals, implicit inputs, global effects, ownership effects, and storage-lifetime guarantees.
- Capacity and construction rules for cursor-designated records.
- Whether the 24-byte adjustments intentionally traverse overlapping records.
- Exact address comparison, wrapping, and invalid-address behavior.
- Supported alignment, volatile-memory behavior, concurrency, and reentrancy.
- Progress guarantees, iteration limits, and surrounding-system termination conditions.
- Error meanings beyond normalization of checked nonzero results to 1.

Inferred range-processing intent does not justify adding validation, retries, allocation, cleanup, endpoint rewrites, or assumed global constancy.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping.

Established evidence supports the field accesses and widths shown, initialization order, signed endpoint comparisons, branch precedence, mode-read conditions, dependency-call conditions, 24-byte cursor adjustments, failure short-circuiting, and cached completion writes.

The direction-2 call expression establishes a source-global value read, not an immutable constant. Its placement establishes that argument preparation follows successful mode-transition calls; it does not establish the relative order of the three memory reads used as arguments.

Application semantics, dependency behavior, the global’s declaration, and exact machine-level pointer semantics are not established. No implementation access or runtime tests have been performed.
