<!-- Trusted provenance: F-7fd29baf5f2f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7fd29baf5f2f

## Purpose

F-7fd29baf5f2f performs a bounded sequence of processing attempts on a caller-supplied control object, conditionally performs recovery-related operations, and returns a bitwise-filtered status.

The established behavior distinguishes the status value `0x11` from all other processing results. That value can trigger further attempts or a final recovery path. A nonzero flag in an associated context can override the eventual status to `7`.

The meanings of these status values, the context flag, and the recovery operations are not established. Terms such as “processing,” “recovery,” and “release” below describe observable roles, not guarantees about the underlying subsystem.

This contract describes the supplied static evidence. No runtime tests have been performed.

## Interface

### Inputs and return value

The interface exposes four incoming argument slots:

| Input | Established use |
|---|---|
| Control-object address | A nullable address of a writable object containing 32-bit fields and at least one independently accessed byte. |
| Second argument | No direct use is visible. |
| Third argument | No direct use is visible. |
| Fourth argument | Forwarded to F-a061acf353ed if the final context-flag handling path invokes that dependency. |

The return value is an unsigned 32-bit quantity.

A null control-object address returns `0x15` immediately. This result is not masked and does not require any dependency call or object access.

For a nonnull control-object address, the function reads the associated-context address from the first 32-bit word before making any dependency call. That address is retained for subsequent context accesses. The function does not visibly reload it from the control object.

The supplied representation uses 32-bit address-sized and integer-sized accesses. The contract therefore describes a 32-bit layout; it does not establish a portable layout for a different address width.

### Directly accessed layout

Offsets are byte offsets from the relevant object’s base address.

| Object | Offset | Width | Observed role |
|---|---:|---:|---|
| Control object | `0x00` | 32 bits | Associated-context address, captured once on entry. |
| Control object | `0x74` | 32 bits | Receives `7` on one terminal recovery branch. |
| Control object | `0xF4` | 32 bits | Value supplied to F-94cb53877c15; later replaced on a recovery branch. |
| Control object | `0xFF` | 8 bits | Cleared after each successful retry-eligibility decision and retry action. |
| Control object | `0x148` | 32 bits | Nonzero enables examination of a context recovery field. |
| Associated context | `0x18` | 32 bits | Final return mask. |
| Associated context | `0x1E` | 8 bits | Nonzero triggers terminal notification and status override. |
| Associated context | `0xC8` | 32 bits | Additional nonzero condition for the recovery sequence. |

The byte at control offset `0xFF` is separate from the 32-bit field beginning at `0xF4`. Clearing that byte is not evidence of clearing the field at `0xF4`.

For a nonnull control object, its first word must be readable. Further readability and writability requirements depend on the path taken. In particular, the retry-status path may dereference the captured context without first checking it for null.

### Ownership and lifetime

The function does not visibly allocate either object, transfer the control-object address, or replace its first word.

It passes the field at control offset `0xF4` to F-94cb53877c15 and may replace that field afterward. The evidence does not establish whether this field represents an owned resource, a borrowed reference, a handle, or another value. No ownership or lifetime guarantee is inferred from the call.

Both objects must remain valid for every direct access actually reached. Dependency behavior may impose additional requirements that are not available here.

## Behavior

### Initial processing and retry limit

After capturing the associated-context address, the function begins with a retry count of zero and invokes F-9a93975eefed with the control-object address.

A result other than `0x11` immediately selects terminal status handling. It does not invoke retry eligibility or the recovery sequence described below.

A result equal to `0x11` selects one of two paths:

- If five retry actions have already completed, the function selects terminal recovery handling without calling F-cefc2c84681f again.
- Otherwise, it calls F-cefc2c84681f with the control-object address. A zero result selects terminal recovery handling. Any nonzero result permits another retry action.

A permitted retry action consists of these ordered effects:

1. Call F-6ed206137014 with the control-object address.
2. After that call returns, write an 8-bit zero at control offset `0xFF`.
3. Advance the retry count.
4. Invoke F-9a93975eefed again.

Thus, assuming the dependencies return normally, the function performs at most five calls to F-6ed206137014 and at most six calls to F-9a93975eefed in one invocation.

F-cefc2c84681f is called only after a processing result of `0x11` and only while fewer than five retry actions have completed. Its result is interpreted solely as zero versus nonzero.

The byte clear at offset `0xFF` occurs after F-6ed206137014, not before it. A write performed by that dependency to the same byte can therefore be overwritten by the explicit clear.

### Terminal recovery handling

This stage is reached only after a processing result of `0x11`, either because eligibility returned zero or because five retry actions have already completed.

The function examines the 32-bit field at control offset `0x148`.

- If the field is zero, it does not read the context field at `0xC8` at this point.
- If the field is nonzero, it reads the 32-bit field at offset `0xC8` in the captured context.

The control field at `0x148` is then read again as part of deciding whether to enter the recovery sequence. The sequence requires both a nonzero second observation of that control field and a nonzero context-field value obtained above. If the first control-field observation was zero, the context-condition value used for this decision is zero.

There is no intervening dependency call between these two control-field reads. The evidence nevertheless shows separate accesses; it does not establish an atomic snapshot or synchronization rule.

When both conditions hold, the function performs the following ordered operations:

1. Call F-95b79f0a129e and retain its 32-bit result as an opaque token.
2. Read the current 32-bit field at control offset `0xF4` and pass it to F-94cb53877c15.
3. After that dependency returns, read the byte at captured-context offset `0x1E`.
4. Select one of two branches:
   - If the byte is zero, call F-8ad0042eaec1 with the captured-context address and the retained token. After the call returns, store its 32-bit result at control offset `0xF4`.
   - If the byte is nonzero, first write the 32-bit value `7` at control offset `0x74`, then write a 32-bit zero at control offset `0xF4`.

The field at `0xF4` supplied to F-94cb53877c15 is read after F-95b79f0a129e returns. It is not a snapshot taken before that call.

The context flag deciding which branch to take is likewise read after F-94cb53877c15 returns. Neither earlier flag values nor assumptions about dependency side effects replace that observation.

Entering this sequence does not itself replace the retained processing status of `0x11`. Its visible status override, if any, occurs during terminal status handling.

### Terminal status handling

This stage handles both an ordinary processing result and the retained `0x11` result after terminal recovery handling.

If the captured context address is zero, the return value is the processing status bitwise-ANDed with `0x000000FF`. This keeps only its low eight bits.

If the captured context address is nonzero, the function reads its byte at offset `0x1E`.

When that byte is nonzero:

1. Invoke F-a061acf353ed using the captured context, the values `7` and `0`, the fourth incoming argument, and additional opaque incoming machine context visible in the supplied call representation.
2. After the call returns, select `7` as the status to be returned.
3. Write an 8-bit zero at captured-context offset `0x1E`.

The function then reads the 32-bit mask at captured-context offset `0x18` and returns the selected status bitwise-ANDed with that mask.

When the flag byte is zero, no notification call or explicit flag clear occurs in this stage. The original processing status is masked by the context field at `0x18`.

The mask is read after any notification call and explicit flag clear. Therefore, the returned value can reflect mask changes made by that dependency. The function does not visibly preserve an earlier mask value.

The notification decision is made by a fresh flag read. For example, the recovery initializer may change the flag before terminal status handling reaches it.

## Side effects

The function’s direct writes are limited to the following conditional effects:

- An 8-bit zero at control offset `0xFF` after each completed retry action.
- A 32-bit replacement value at control offset `0xF4` after F-8ad0042eaec1 returns.
- Alternatively, a 32-bit `7` at control offset `0x74`, followed by a 32-bit zero at `0xF4`.
- An 8-bit zero at captured-context offset `0x1E` after F-a061acf353ed returns.

Dependency calls may have additional effects. Those effects are not specified here.

There is no visible rollback. In particular, F-94cb53877c15 is called before the function decides whether to obtain and store a replacement field value or explicitly zero the field. If a dependency does not return normally, later stores and terminal masking are not established to occur.

The captured context address remains the one used throughout this invocation even if a dependency changes the control object’s first word. This does not imply that the context’s contents remain unchanged.

No synchronization, locking, exception conversion, or validation of dependency results is visible. All 32-bit return values from the replacement dependency are stored as received; no explicit success check occurs.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-9a93975eefed | Receives the control-object address. Its unsigned 32-bit result determines whether the function retries or terminates processing. Only exact equality with `0x11` selects retry handling. |
| F-cefc2c84681f | Receives the control-object address. Zero ends retry eligibility; nonzero permits one retry action, subject to the five-action limit. |
| F-6ed206137014 | Receives the control-object address during a permitted retry. Its return value is not used. |
| F-95b79f0a129e | Called without explicit arguments in the supplied representation. Produces the opaque 32-bit token used by F-8ad0042eaec1. |
| F-94cb53877c15 | Receives the current 32-bit value at control offset `0xF4`. Its return value is not used. Resource release is an inferred role, not an established ownership contract. |
| F-8ad0042eaec1 | Receives the captured-context address and opaque token. Its 32-bit result replaces the control field at `0xF4`. |
| F-a061acf353ed | Invoked when the terminal context-flag read is nonzero. Receives the context, fixed values `7` and `0`, the fourth incoming argument, and opaque incoming machine context. Its return value is not used. |

The supplied dependency mapping also includes F-15a13295fe3c. The supplied body does not establish a call to that dependency; a control-flow destination is not treated as an invocation.

## Acceptance examples

These are conditional static acceptance cases, not reported executions. Unless a case says otherwise, assume valid accessible objects, normal dependency returns, no asynchronous mutation, and no dependency mutations beyond those stated.

1. **Null control object.**  
   Input is a null control-object address. Expected return: `0x15`. No dependency calls or object reads occur.

2. **Ordinary result without a context.**  
   The captured context is null and the first processing result is `0x12345678`. Expected return: `0x78`. Retry eligibility is not called.

3. **Ordinary result with a context mask.**  
   The processing result is `0x12345678`, the context flag is zero, and the mask is `0x0000FF00`. Expected return: `0x00005600`.

4. **Terminal flag override.**  
   The processing result is `0x80`, the context flag is nonzero, and the mask read after notification is `0xFFFFFFFF`. F-a061acf353ed is called, the flag is subsequently cleared, and the return is `7`, not `0x80`.

5. **Override remains subject to masking.**  
   The terminal flag is nonzero and the final mask is `0x00000002`. Expected return: `2`, because the selected status `7` is masked. A zero mask would produce zero.

6. **One retry followed by completion.**  
   Processing returns `0x11`, eligibility returns nonzero, and the next processing call returns `4`. Expected order: processing, eligibility, retry action, byte clear at `0xFF`, processing, terminal handling. There is one retry action.

7. **Retry cap.**  
   Processing always returns `0x11` and each eligibility call returns nonzero. There are six processing calls, five eligibility calls, and five retry-action calls. The sixth processing result proceeds directly to terminal recovery handling.

8. **Recovery disabled.**  
   Processing returns `0x11`, eligibility returns zero, and control offset `0x148` is zero. The recovery dependencies are not invoked, and context offset `0xC8` is not read by that recovery gate. Terminal handling still applies.

9. **Recovery with replacement.**  
   Both recovery conditions are nonzero. After F-94cb53877c15 returns, the context flag is zero. F-8ad0042eaec1 receives the captured context and previously obtained token; its result is stored at `0xF4`. The later terminal flag read independently determines whether status `0x11` is overridden.

10. **Recovery with a set flag.**  
    Both recovery conditions are nonzero, and the flag remains nonzero after F-94cb53877c15 returns. The function writes `7` at `0x74`, then zero at `0xF4`. If the flag remains set until terminal handling, notification occurs, the flag is cleared, and the return is `7` masked by the final context mask.

## Uncertainties

The following are **documented_only**:

- Domain meanings of `0x11`, `0x15`, `7`, the mask, and the accessed fields.
- Complete object sizes, alignment requirements, and layouts beyond the accessed locations.
- Ownership and lifetime semantics of the field at `0xF4`.
- Dependency effects on memory, globals, resources, and control flow.
- The semantic meaning and complete calling convention of the opaque incoming machine context forwarded to F-a061acf353ed.
- Whether concurrent, interrupt-driven, or externally triggered mutation is supported.
- Failure behavior for inaccessible addresses or dependencies that do not return normally.

A null captured context is supported by terminal status handling, but not unconditionally by the entire function. If processing returns `0x11` and the first recovery-enable read is nonzero, recovery handling attempts to read context offset `0xC8` without a visible null check. No safe result is specified for that situation.

## Evidence

**Established static evidence:** the supplied body supports the widths, field offsets, branch conditions, bounded retry counts, dependency-call order, direct write order, retained context address, and return-mask behavior described above.

**Inferred intent:** the repeated `0x11` path appears to represent retryable processing; the terminal dependency sequence appears recovery-related; the byte at context offset `0x1E` appears to request exceptional terminal handling. These interpretations are not required to implement the established observable behavior.

**Unknown:** dependency internals, broader subsystem contracts, concurrency rules, resource ownership, and runtime failure behavior are unavailable. No runtime tests have been performed, and the acceptance examples are conditional requirements derived from static evidence rather than execution results.
