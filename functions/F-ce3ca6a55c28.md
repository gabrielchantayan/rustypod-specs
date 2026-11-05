<!-- Trusted provenance: F-ce3ca6a55c28; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ce3ca6a55c28

## Purpose

F-ce3ca6a55c28 conditionally performs preparation, decides whether to refresh a resource selection, stores a new 32-bit observation, and invokes a main operation. A nonzero main-operation result triggers a final selection call with the constant 1. The function returns the original main-operation result.

The preparation test and subsequent state processing do not necessarily address the same storage. The function captures a shared base value at entry, independently rereads that shared source for the preparation test, and conditionally rereads it again for a guard argument. Later direct state accesses use the entry-captured base.

“Preparation,” “resource,” “selection,” “refresh,” and “observation” describe local roles. They do not establish application semantics. In particular, observations are not established to represent time, and operation results are not established to mean success or failure.

## Interface

### Inputs and output

| Item | Established representation | Use |
|---|---|---|
| First input | Opaque 32-bit value | Forwarded unchanged as the second argument to F-aa672b15efb7. |
| Second input | 32-bit integer | Compared with the selection field and written there when unequal. |
| Return value | 32-bit integer | Result of F-aa672b15efb7, preserved across any final call. |

The function does not dereference or otherwise interpret the first input. It imposes no local range restriction on the second input. Selection equality and storage use the complete 32-bit value.

A main-operation result of zero suppresses the final call. Any nonzero result, including a negative 32-bit integer, enables it.

### Shared sources and state layout

One shared source supplies a base value at three distinct evaluation points:

| Base role | Read condition | Use |
|---|---|---|
| Captured base | At entry, before the preparation test | Retained for subsequent state accesses and the preparation-completion address. |
| Test base | After the entry capture | Supplies the address of the preparation-test word. |
| Guard base | Only when the tested bit is clear | Supplies the address passed to the guard dependency. |

No supplied contract guarantees that these reads yield the same value.

The relevant layout uses byte offsets:

| Location | Width or form | Local use |
|---|---|---|
| Test base +8 | 32-bit word | Read to test bit 0. |
| Guard base +8 | Address | Passed to F-b255b26284e6. |
| Captured base +8 | Address | Passed to F-d08d3ca17b81. |
| Captured base +12 | 32-bit field | Selection comparison, conditional write, and arguments to preparation and refresh calls. |
| Captured base +16 | 32-bit field | Stored observation read and subsequent observation write. |

The function does not directly read through the guard-base address merely because it passes that address to a dependency. Likewise, passing captured base +8 does not establish a direct local write there.

Other shared sources provide:

- A resource argument used at several call sites.
- A preparation configuration argument.
- Two additional preparation arguments, supplied in their established order.

These values are evaluated for their respective calls, not captured together at entry. Their application-level types and ownership properties are unknown; the resource argument is not established to be a pointer.

### Preconditions

Normal execution requires valid storage for every evaluated direct access:

- The shared base source must be readable at each read.
- Test base +8 must permit a 32-bit read.
- Captured base +12 must permit the required 32-bit reads and conditional write.
- Captured base +16 must permit the required 32-bit read, when applicable, and write.
- Other shared argument sources must be readable when evaluated.
- Each dependency must accept the values and addresses supplied to it.

The guard and completion addresses must satisfy their respective dependency contracts. They need not designate the same allocation.

There are no local null checks, bounds checks, address-validity checks, or recovery paths for invalid storage. Required alignment, address width, and address-arithmetic validity are not independently established.

## Behavior

### Preparation decision

The function first captures the shared base value. It then rereads the base source and reads the 32-bit word at test base +8.

Only bit 0 controls preparation:

- If bit 0 is set, all preparation calls are skipped. The guard-base read does not occur.
- If bit 0 is clear, the function rereads the shared base source and calls F-b255b26284e6 with guard base +8.

Consequently, the guard argument need not identify either the tested word or captured base +8.

A zero guard result skips the remaining preparation calls. A nonzero result enables them. The function does not repeat the bit test after the guard returns.

### Preparation calls

After a nonzero guard result, the following calls occur in order:

1. F-1058afc38c04 receives the current resource argument, the 32-bit selection at captured base +12, the constant 0, and the preparation configuration argument.
2. F-804adc29bbfb receives the opaque 32-bit result of F-1058afc38c04 and the two additional shared preparation arguments in their established order.
3. F-d08d3ca17b81 receives captured base +8.

The selection read for the first call occurs after the guard returns. A guard-side mutation can therefore affect that argument.

The result of F-1058afc38c04 is forwarded without a local validity check or zero check. A zero result is still forwarded. Results of the next two preparation calls are not inspected.

Replacing the shared base source during preparation does not redirect the captured-base accesses. In particular, the completion address remains captured base +8. If a dependency invalidates captured storage, the function has no local recovery.

### Selection comparison

After preparation completes or is skipped, the function compares captured base +12 with the second input.

If the values differ:

- It writes the second input to captured base +12.
- Refresh is required.
- It does not obtain an observation for the difference decision.

If the values are equal:

- It calls F-380d68353221 without arguments.
- After that call returns, it reads the stored observation at captured base +16.
- It decides whether refresh is required using the returned observation and that stored value.

The selection comparison is not repeated after the observation call. If that dependency modifies the selection field, execution still follows the already-entered equality path.

### Observation-dependent refresh decision

On the equality path, the represented decision uses the unsigned 32-bit difference between the returned observation and the subsequently read stored observation. Subtraction is interpreted modulo \(2^{32}\):

\[
d=(\text{returned observation}-\text{stored observation})\bmod 2^{32}.
\]

Refresh is suppressed when \(d<1001\). Therefore:

- Differences from 0 through 1000 suppress refresh.
- Differences from 1001 through 4,294,967,295 require refresh.

The conditional result of F-380d68353221 therefore affects control flow through this arithmetic comparison. It is not merely recorded or ignored.

This is not a signed ordering comparison. A returned observation smaller than the stored value may produce a large unsigned difference. Conversely, crossing the 32-bit boundary may produce a small difference.

This contract describes the represented 32-bit behavior. It does not assert that a transcription using overflowing signed arithmetic would be portable in another implementation language.

The stored-observation read follows the dependency call. If that dependency changes captured base +16 before returning, the comparison uses the changed value.

### Refresh calls

When refresh is required, the function calls:

1. F-849b643bed3c with the resource argument evaluated for that call.
2. F-cdc135a18dea with the resource argument evaluated for the second call and a fresh read of captured base +12.

Neither result is inspected.

The selection passed to F-cdc135a18dea need not equal the second input or the value used in the earlier comparison. F-849b643bed3c may change captured base +12 before the fresh read. It may also change the shared resource source, affecting the next call’s resource argument.

The function does not restore the selection after refresh. If the difference decision suppresses refresh, neither call occurs at this stage.

### Observation write and main operation

After refresh completes or is skipped, the function calls F-380d68353221 and writes its 32-bit result to captured base +16.

This is a fresh call. An observation obtained earlier for the difference decision is not reused for the write.

The function then calls F-aa672b15efb7 with:

- The resource argument evaluated at that call site.
- The first input, unchanged.
- The constant 0.

The observation write precedes this operation.

If the operation returns zero, the function returns zero without a follow-up call.

If the operation returns any nonzero 32-bit value, the function calls F-cdc135a18dea with the resource argument evaluated at that later call site and the constant 1. If that call returns normally, the function returns the original operation result unchanged.

The function does not directly write 1 into the selection field. Any such change would be a dependency effect.

## Side effects

### Direct writes

The identified direct writes are:

- A conditional 32-bit write of the second input to captured base +12 after an unequal selection comparison.
- A 32-bit write of a fresh observation to captured base +16 before the main operation.

There is no direct write to a +8 word and no direct replacement of the shared base source.

These writes do not establish final state at return. Later dependencies may modify the same storage.

### Observable ordering

For normal execution, the established order is:

1. Capture the shared base.
2. Reread the shared base and test bit 0 at test base +8.
3. If that bit is clear, reread the shared base for the guard argument and call the guard.
4. If the guard returns nonzero, perform the remaining three preparation calls.
5. Compare the captured selection with the second input.
6. Either write the changed selection or obtain a comparison observation and then read the stored observation.
7. Perform refresh when required.
8. Obtain a fresh observation and write it to captured base +16.
9. Invoke the main operation.
10. If its result is nonzero, perform the constant-1 follow-up.
11. Return the main-operation result.

The supplied representation does not establish a left-to-right order among independent argument evaluations within a single call. The inter-call sequence and the explicitly described accesses remain established.

### Ownership, return-dependent branches, and errors

No explicit allocation, deallocation, ownership transfer, or release occurs locally. Forwarding the preparation result once does not establish whether it denotes memory or whether either dependency consumes, retains, or releases it.

Two dependency results receive direct zero/nonzero tests:

- The guard result controls whether the remaining preparation calls occur.
- The main-operation result controls whether the final constant-1 call occurs.

A third result controls a branch indirectly: the conditional F-380d68353221 result participates in the unsigned 32-bit difference comparison that determines whether refresh occurs.

The later F-380d68353221 result is stored rather than used for a local branch. The F-1058afc38c04 result is forwarded without a local branch. Other dependency results are not inspected.

There is no local rollback, retry, error translation, or exception handler. If a dependency does not return normally, subsequent calls, writes, and the final return are not guaranteed.

### Concurrency

The guard pattern may suggest synchronization, but its semantics are unknown. The function alone establishes no thread-safety guarantee, atomic access requirement, memory ordering, or protection of the selection and observation fields.

The separate shared-base reads cannot be treated as a single stable-storage guarantee without an external contract.

## Dependencies

| Opaque function ID | Established local contract |
|---|---|
| F-b255b26284e6 | Receives guard base +8 only when the tested bit is clear. Zero skips further preparation; nonzero enables it. |
| F-1058afc38c04 | Receives the resource, captured selection, 0, and configuration. Produces an opaque 32-bit value that is forwarded. |
| F-804adc29bbfb | Receives that forwarded value and two additional shared preparation arguments. Its result is not inspected. |
| F-d08d3ca17b81 | Receives captured base +8 after the preceding preparation calls. Its result is not inspected. |
| F-380d68353221 | Takes no arguments. Supplies a 32-bit observation conditionally for the refresh decision and separately for the later write. |
| F-849b643bed3c | Receives the resource at the start of refresh. Its result is not inspected. |
| F-cdc135a18dea | Receives the resource and freshly read selection during refresh, or the resource and 1 after a nonzero operation result. Its result is not inspected. |
| F-aa672b15efb7 | Receives the resource, first input, and 0. Its 32-bit result controls the follow-up and is returned. |

No dependency is assumed to preserve globals, preserve captured storage, or follow a particular ownership convention.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Unless stated otherwise, dependencies return normally, the shared base remains stable, and dependencies do not modify relevant shared values.

### Preparation-bit behavior

If the tested word has bit 0 set, none of the four preparation dependencies is called, and there is no guard-base read. Selection processing continues.

A word with only bit 1 set does not bypass the guard call.

### Separate base values

Suppose the entry capture yields region A, the test-base read yields region B, and the guard-base read yields region C.

If B +8 has bit 0 clear:

- The guard receives C +8.
- After a nonzero guard result, the preparation selection is read from A +12.
- The completion call receives A +8.
- Later direct selection and observation accesses use A.

This example requires an environment in which those reads and accesses are valid. It does not establish how such changes are synchronized.

### Zero guard result

With the tested bit clear and a zero guard result, the other three preparation calls do not occur. The selection comparison, eventual observation write, and main operation still occur on normal execution.

### Changed selection

With stored selection 4 and second input 9, the function writes 9 and performs both refresh calls. It skips the comparison-only observation call.

Absent intervening mutation, the refresh selection call receives 9. A fresh observation is then obtained and written before the main operation.

### Observation result controls the threshold decision

With matching selections and stored observation 5000:

- A comparison observation of 6000 yields difference 1000 and suppresses refresh.
- A comparison observation of 6001 yields difference 1001 and requires refresh.

Thus changing only this dependency result can change whether the refresh calls occur. Both paths later obtain another observation for storage.

### Unsigned wraparound

With matching selections, stored observation 4,294,967,280, and comparison observation 20, the difference is 36. Refresh is suppressed.

With stored observation 100 and comparison observation 99, the difference is 4,294,967,295. Refresh occurs.

### Mutations affect later reads

If F-849b643bed3c changes the captured selection to 17, the next refresh call receives 17.

If the comparison observation call returns 8000 and changes the stored observation to 7500 before returning, the difference is 500, so refresh is suppressed.

If preparation replaces the shared base source, subsequent direct accesses still use the entry-captured base, provided that storage remains valid.

### Operation-result preservation

If the main operation returns −7, the function performs the constant-1 follow-up and then returns −7, provided the follow-up returns normally. There is no direct local write of 1 to the selection field.

If the main operation returns zero, no follow-up occurs.

## Uncertainties

The following remain **documented_only**:

- Stability of the shared base source across its separate reads.
- Whether the tested, guard-addressed, and completion-addressed locations coincide in actual use.
- Meaning, units, and monotonicity of observations.
- Application semantics of selection values, including constant 1.
- Whether preparation implements initialization, synchronization, registration, or another protocol.
- Dependency side effects, ownership rules, error conventions, and abnormal control flow.
- Required alignment, address representation, and validity of address arithmetic.
- Concurrency guarantees and memory ordering.
- Whether dependencies may validly replace shared sources or invalidate captured storage.
- Final state after dependencies with unknown effects.

Guarded preparation and difference-based refresh suppression are inferred roles, not established external API semantics.

## Evidence

Established behavior comes from the supplied static representation: separate base-source evaluations, 32-bit field accesses, byte offsets, bit testing, selection equality, conditional storage, the unsigned difference threshold of 1001, dependency arguments, call ordering, and preservation of the main-operation result.

The return-dependent decisions include both direct zero/nonzero tests and the observation-dependent arithmetic comparison. Dependency identities alone establish no ownership, synchronization, preservation, or error-handling contract.

No runtime tests, dependency executions, or dynamic observations have been performed. Acceptance examples express conditional behavior derived from static evidence.
