<!-- Trusted provenance: F-a9dd153e2ab9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a9dd153e2ab9

## Purpose

F-a9dd153e2ab9 conditionally processes an object’s associated entity, issues two notification-like calls, and performs a final embedded-state completion operation. It returns a 32-bit result determined by prerequisite checks, dispatched operations, and completion outcomes.

The established sequence is:

1. Check an embedded prerequisite.
2. Invoke an object-dispatched gate.
3. Prepare an embedded region.
4. If an associated entity is present, clear a byte, perform conditional entity operations, and issue two notifications.
5. Check and update completion-related fields, possibly invoking a final dependency.

The field relationships suggest synchronization ownership and recursive-release behavior. Those meanings are inferred intent, not established API semantics. The contract below specifies accesses, calls, arithmetic, writes, and return propagation without requiring those interpretations.

## Interface

### Input and result

The function receives one object pointer and returns a 32-bit integer.

The supplied representation uses 32-bit object words and a 32-bit address model. All object offsets below are byte offsets. Dispatch-table offsets are identified separately.

There is no explicit output parameter. Observable outputs are:

- The returned 32-bit value.
- Direct writes to the object.
- Effects of reached dependencies and dispatched operations.

No explicit null-input check precedes the first dependency call. Valid execution requires every reached object region, pointer, dispatch table, and call target to support the corresponding access. Exact allocation sizes, alignment requirements, calling conventions, and lifetime rules are not established.

### Object layout used by this function

| Offset | Width or role | Observed use |
|---|---|---|
| `0x000` | 32-bit dispatch-table pointer | Selects the object gate at table offset `0x1A4`. |
| `0x014` | Embedded dispatch-bearing region | Its first 32-bit word selects an operation at table offset `0x14`; the region address is passed to that operation. |
| `0x054` | 32-bit associated-entity pointer | Initially checked for zero, then reread for later entity operations. |
| `0x060` | Embedded region, extent unknown | Passed to F-b7abcf60c990. |
| `0x93E` | 8-bit field | Cleared upon entering the associated-entity branch. |
| `0x944` | 32-bit field | Conditionally read; only its low eight bits affect a call decision. |
| `0x964` | Embedded region, extent unknown | Passed to F-936a8728139d. |
| `0xA4C` | Embedded completion-related region | Passed to preparation and conditionally to F-621dae46a146; its first 32-bit word participates in a comparison. |
| `0xA50` | 32-bit field | Compared with F-196812b4b122’s result and conditionally cleared. |
| `0xA5A` | Unsigned 16-bit field | Snapshotted before F-196812b4b122; bits 4 and 5 determine the completion mode. |
| `0xA5E` | Signed 16-bit field | Conditionally decremented or cleared. |
| `0xA60` | Embedded region, extent unknown | Passed to F-259d2fe281b0 after completion fields are cleared. |

For each entity-dispatched operation, the function reads the dispatch-table pointer from offset `+4` of the then-current associated entity. The selected table offsets are `0xF0` and `0x118`.

### Global inputs

Two global storage locations participate:

- A 32-bit notification token. Within the associated-entity branch, it is evaluated twice: first to capture a value for the final indirect notification, then again to supply F-069a1d444a2d’s argument.
- A comparison value used against the first 32-bit word at `0xA4C`.

The function contains no direct write to either global. Their meanings and mutability are unknown. Dependencies are not assumed to preserve them.

## Behavior

### 1. Prerequisite and object gate

The function first calls F-b7abcf60c990 with the address at `0x060`.

If that call returns zero, the function immediately returns zero. No object gate, preparation, entity processing, or completion handling is reached.

Otherwise, the function reads the input object’s dispatch-table pointer and invokes the operation at table offset `0x1A4`, passing the input object.

- A nonzero result is returned unchanged.
- A zero result permits processing to continue.

Effects of calls already reached are not rolled back on either early-return path.

### 2. Embedded-region preparation

The function forms and retains the embedded address at `0xA4C`, then passes it to a preparation dependency whose opaque ID is not supplied.

No preparation return value is consumed.

Preparation occurs before the associated-entity pointer is tested. Consequently, an absent associated entity does not bypass preparation or the final completion phase.

The preparation dependency’s effects on memory, globals, ownership, or lifetime remain **documented_only**.

### 3. Associated-entity processing

After preparation returns, the function reads the pointer at `0x054`.

If it is zero, all operations in this subsection are skipped. Otherwise, the following sequence applies:

1. Clear the single byte at `0x93E`.
2. Call F-936a8728139d with the address at `0x964`.
3. Reread the associated-entity pointer, read its dispatch-table pointer from entity offset `+4`, and invoke the operation at table offset `0xF0`.
4. If that operation returns nonzero, reread the associated-entity pointer and pass it to F-dfbce8415e0b.
5. Reread the associated-entity pointer, read its dispatch-table pointer from entity offset `+4`, and invoke the operation at table offset `0x118`.
6. If the `0x118` operation returns nonzero, read the 32-bit field at `0x944` and retain its low eight bits for the condition.
7. If the `0x118` result and those low eight bits are both nonzero, reread the associated-entity pointer and call F-97d68c2fa8e6 with that pointer and a second argument of zero.
8. Evaluate the global notification token and retain this first 32-bit value for the final indirect operation.
9. Evaluate the same global again and call F-069a1d444a2d with the input object and this second value.
10. After F-069a1d444a2d returns, read the dispatch-table pointer at `0x014`. Invoke its operation at table offset `0x14`, passing the address at `0x014`, the retained first token value, and zero.

A zero result from the `0xF0` operation suppresses only F-dfbce8415e0b. It does not suppress the `0x118` operation or either notification.

The read at `0x944` occurs only when the `0x118` operation returns nonzero. Higher bits of the 32-bit field do not affect the condition. A low byte of `0x80` is nonzero regardless of whether its narrowed character interpretation is signed.

#### Entity-pointer stability

The initial nonzero pointer check does not establish a stable receiver for the entire branch. Later operations use fresh reads of `0x054`.

A dependency may therefore redirect later operations to a different associated entity. If it clears the pointer, subsequent dispatch-table accesses are not protected by another explicit null check. Successful execution requires the pointers used by those later accesses to remain valid.

The two entity-dispatched calls show no explicit arguments in the supplied representation. An implicit receiver or other ABI-level inputs cannot be established from this body alone.

#### Notification-token evaluation order

The token accesses are two distinct evaluations, not one shared load:

- The first supplies the value retained for the final indirect notification.
- The second supplies F-069a1d444a2d’s token argument.

No dependency call intervenes between these evaluations. That fact alone does not establish that the two values are equal. Equality requires an explicit stability assumption about the global during this interval.

Mutation of the global during F-069a1d444a2d cannot replace the first value already retained for the final operation. In contrast, the dispatch-table pointer at `0x014` is read after that dependency returns and can reflect changes made by it.

### 4. Retained embedded-address check

After the optional associated-entity branch, the function tests the previously formed address at `0xA4C`.

- If it is zero, return `26`.
- Otherwise, continue.

For an ordinary valid object allocation, this embedded address is nonzero. The explicit zero-address branch does not constitute an input-validity check: preparation and possibly other operations have already occurred.

Reachability of this branch under the original language’s pointer rules remains **documented_only**.

### 5. Conditional special handler

The function compares the first 32-bit word at `0xA4C` with the global comparison value.

If they differ, it skips F-621dae46a146.

If they match, it calls F-621dae46a146 with the retained embedded-region address and zero:

- A nonzero result is returned immediately.
- A zero result allows completion handling to continue.

A nonzero return from this dependency prevents the subsequent mode snapshot, owner-like comparison, depth handling, and final dependency call. Earlier effects are not undone.

### 6. Mode snapshot and owner-like comparison

The function reads the unsigned 16-bit field at `0xA5A` and retains that value. It then calls F-196812b4b122.

After that dependency returns, the function reads the 32-bit field at `0xA50` and compares it with the returned value.

If they differ:

- Return `5`.
- Do not directly modify `0xA50` or `0xA5E` in this completion phase.
- Do not call F-259d2fe281b0.

This does not constrain writes performed by earlier dependencies.

If the values match, the retained mode selects the next path. Only bits 4 and 5 matter. The special decrementing mode is selected when those two bits have the value binary `10`, equivalently when the retained value masked by `0x30` equals `0x20`. All other bits are irrelevant to this decision.

Changes made to `0xA5A` by F-196812b4b122 do not change the retained mode. The comparison field at `0xA50`, and the depth field if needed, are read after that call.

### 7. Completion paths

#### Mode other than binary `10`

With a matching owner-like comparison and any other mode:

1. Clear the 16-bit field at `0xA5E`.
2. Clear the 32-bit field at `0xA50`.
3. Call F-259d2fe281b0 with the address at `0xA60`.
4. Return that dependency’s 32-bit result.

This path does not require reading the prior depth.

#### Mode equal to binary `10`

With a matching owner-like comparison and the decrementing mode:

1. Read the signed 16-bit field at `0xA5E`.
2. Subtract one and store the narrowed result back into that 16-bit field.
3. If that result is nonzero, return zero without directly clearing `0xA50` and without calling F-259d2fe281b0.
4. If the result is zero, clear `0xA5E` again, clear `0xA50`, call F-259d2fe281b0 with the address at `0xA60`, and return its result.

The decrement is neither saturating nor guarded by a positivity check. A depth of zero becomes `−1` and follows the nonzero-result path.

For an initial depth of `−32768`, subtraction produces `−32769` before narrowing to 16 bits. That subtraction fits in the represented 32-bit arithmetic, but the narrowed signed result is outside the ordinary signed 16-bit range before conversion. Exact original-language narrowing semantics remain unresolved. Under ordinary 16-bit wraparound, the stored value is `32767` and the function returns zero without final completion.

## Side effects

Direct writes in the supplied body are limited to:

- Clearing `0x93E` when the associated-entity branch is entered.
- Decrementing or clearing `0xA5E` on applicable completion paths.
- Clearing `0xA50` immediately before F-259d2fe281b0.

On final completion, the clear at `0xA5E` precedes the clear at `0xA50`, which precedes the dependency call. The decrement-to-zero path includes a decrement store followed by another zero store to `0xA5E`.

If F-259d2fe281b0 returns nonzero, these writes are not rolled back. Nor does this function restore earlier values after any other early return.

There is no explicit allocation, deallocation, ownership transfer, exception handler, or lifetime restoration in this body. This does not establish that dependencies preserve allocations, object lifetime, globals, or ownership.

No atomic accesses, ordering barriers, or thread-safety guarantees are established. The inferred synchronization purpose does not make concurrent mutation safe.

## Dependencies

| Dependency | Established call-site behavior |
|---|---|
| F-b7abcf60c990 | Receives the address at `0x060`; zero causes immediate zero return. |
| F-936a8728139d | Receives the address at `0x964` after the byte clear; no result is consumed. |
| F-dfbce8415e0b | Receives a freshly read associated-entity pointer when the `0xF0` operation returns nonzero. |
| F-97d68c2fa8e6 | Receives a freshly read associated-entity pointer and zero when the `0x118` result and conditional low-byte value are nonzero. |
| F-069a1d444a2d | Receives the input object and the second evaluation of the global notification token. The first evaluation has already been retained for the subsequent indirect operation. |
| F-621dae46a146 | Receives the region at `0xA4C` and zero after a matching comparison; a nonzero result is propagated. |
| F-196812b4b122 | Called after the mode snapshot; its result is compared with the subsequently read 32-bit field at `0xA50`. |
| F-259d2fe281b0 | Receives the address at `0xA60` after the depth and owner-like fields are cleared; its result is propagated. |

The preparation dependency and four indirect dispatch sites have no supplied opaque IDs. Their identities remain **documented_only**. Their observable selection and arguments are described above without inventing dependency identifiers.

No dependency is assumed to preserve memory, globals, ownership, or lifetime beyond what is required for a stated conditional example.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless an example states otherwise, reached dependencies return normally and preserve the values relevant to that example.

1. **Prerequisite failure.** If F-b7abcf60c990 returns zero, the result is zero. No later dependency or dispatch is reached.

2. **Object-gate return.** If the prerequisite succeeds and the `0x1A4` operation returns `−7`, the same 32-bit result is returned. Preparation is skipped.

3. **Absent entity.** If preparation leaves `0x054` zero, this function does not clear `0x93E`, invoke either entity operation, or issue either notification. Completion handling still follows.

4. **Low-byte distinction.** With a nonzero `0x118` result, a field value of `0x00000100` skips F-97d68c2fa8e6; `0x00000180` permits it. With a zero `0x118` result, the field is not read for this condition.

5. **Stable token.** If the notification global remains `11` across both evaluations, F-069a1d444a2d receives `11` and the final indirect operation receives the retained `11`. This equality follows from the stated stability condition, not from a single shared evaluation.

6. **Different token evaluations.** Conditional on the first evaluation yielding `11` and the second yielding `22`, F-069a1d444a2d receives `22`; the final indirect operation receives `11`. The function performs no intervening dependency call that explains such a change, and no claim is made that concurrent mutation is safe.

7. **Mutation during notification.** If the first evaluation yields `11` and F-069a1d444a2d later changes the global to `33`, the final indirect operation still receives `11`. Its dispatch table is nevertheless obtained after that dependency returns.

8. **Special-handler stop.** If the comparison matches and F-621dae46a146 returns `9`, return `9`; do not call F-196812b4b122 or F-259d2fe281b0.

9. **Owner-like mismatch.** If F-196812b4b122’s result differs from the subsequently read `0xA50`, return `5` without a direct completion-phase depth update or owner-like clear.

10. **Depth remains nonzero.** With matching values, decrementing mode, and depth `2`, store depth `1`, leave `0xA50` directly unchanged, skip F-259d2fe281b0, and return zero.

11. **Depth reaches zero.** Under the same conditions with depth `1`, store zero, perform the additional depth clear, clear `0xA50`, and return F-259d2fe281b0’s result.

12. **Zero depth.** In decrementing mode, depth zero becomes `−1`; final completion is skipped and the result is zero.

13. **Other mode.** With matching values and any other mode, clear both completion fields and invoke F-259d2fe281b0 regardless of prior depth.

## Uncertainties

The following remain **documented_only**:

- Semantic types, full layouts, alignment, and lifetimes of all objects and embedded regions.
- Whether preparation acquires a lock or performs another operation.
- Whether the owner-like value is a thread identifier or another token.
- Meanings of results `5`, `26`, and dependency-provided values.
- Dependency mutation, allocation, release, reentrancy, and abnormal-exit behavior.
- Implicit inputs and receiver conventions of the entity-dispatched operations.
- Notification-global stability and memory-access semantics beyond the two represented evaluations.
- Reachability of the retained-null-address branch.
- Original-language out-of-range signed 16-bit narrowing behavior.
- Behavior with invalid pointers, corrupt dispatch tables, unsafe concurrent mutation, or non-returning dependencies.

The synchronization and recursive-release interpretation is inferred intent only.

## Evidence

Established static evidence consists of the supplied function representation and dependency-ID map. It supports the stated widths, offsets, conditional accesses, direct writes, dependency ordering, return propagation, and two distinct notification-token evaluations.

No dependency implementations, runtime traces, independent layout declarations, or platform ABI documentation were supplied. No runtime tests have been performed. Acceptance examples are conditional static expectations, not runtime verification claims.
