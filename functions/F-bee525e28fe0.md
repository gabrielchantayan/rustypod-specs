<!-- Trusted provenance: F-bee525e28fe0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bee525e28fe0

## Purpose

F-bee525e28fe0 obtains an associated object, conditionally processes a representation supplied by that object, and submits an accepted result together with its second argument.

The ordinary return is a packed 64-bit value:

- The low 32 bits indicate whether final submission was reached and returned: zero for no submission, one for submission.
- The high 32 bits contain the then-current value of a temporary 32-bit field. That field initially contains the second argument. One processing branch clears it and exposes its address to a dependency.

The visible sequence resembles representation conversion using shared processing state. That interpretation is inferred intent, not an established domain contract. In particular, neither successful submission nor a nonzero high return word establishes downstream success.

This specification describes static evidence. No runtime tests have been performed.

## Interface

### Arguments and representation

The supplied representation uses 32-bit pointer values and 32-bit integer fields.

| Argument | Visible role |
|---|---|
| First | Object pointer used to obtain an associated object through dispatch-table byte offset `0x164`. |
| Second | Object pointer receiving the initial dispatch call; initial high-return value; first argument to final submission. |
| Third | 32-bit value copied into temporary storage whose incoming contents are not subsequently consumed. |
| Fourth | Opaque 32-bit value copied into temporary storage whose incoming contents are not subsequently consumed. |

Before decomposition, the storage initially receiving the third and fourth arguments is overwritten with zero. Their incoming values do not determine a visible branch or dependency argument.

An object’s first 32-bit word supplies its dispatch-table address. Dispatch offsets in this contract are byte offsets, not entry numbers. Required objects, tables, and selected entries must be accessible when used. There are no local null checks for the first or second argument.

### Result

On an ordinary return, bits `31:0` are exactly zero or one. Bits `63:32` are the current contents of the high-return field, preserving its 32-bit bit pattern.

Outside the copied-input branch, the function does not expose that field’s address and does not explicitly change it from the second argument. In the copied-input branch, it:

1. Clears the field.
2. Passes its address to F-63011374610b.
3. Eventually constructs the return from its then-current contents.

The field remains live across subsequent calls. Without a dependency contract excluding retained access, its final contents need not equal either zero or the value present when F-63011374610b returned.

A zero candidate result invokes a dependency marked non-returning; that path has no ordinary return.

### Shared state and global inputs

Distinct global slots supply:

- A shared-base address.
- An initialization word.
- An initialization context value captured after guard acquisition is attempted.
- Another initialization argument.
- A processing address loaded separately at each relevant use.

None is established as immutable.

For ordinary execution without intervening changes to the shared-base global, the shared layout is:

| Offset | Width | Visible role |
|---|---:|---|
| `0x08` | 32-bit flag read | Bit zero gates the initialization attempt; its address is passed to guard operations. |
| `0x18` | 32 bits | Receives the initialization word; later supplies a dispatch-table address. |
| `0x1C` | 32 bits | Cleared during initialization. |
| `0x20` | 32 bits | Lower-address cached-token word. |
| `0x24` | 32 bits | Upper-address cached-token word. |

After the initial dispatch, the function captures a shared base for later explicit field accesses. It separately derives the initialization-slot address from the shared-base global. The flag test and guard-acquisition argument also reference that global. No dependency call intervenes between these initial reads, but the evidence does not establish atomic snapshot semantics under concurrent modification.

The retained base anchors the later cache accesses and guard-release address. The retained initialization-slot address anchors its write and the later processing-table lookup. A subsequent dependency changing the shared-base global does not automatically redirect those retained addresses.

## Behavior

### Initial dispatch and guarded initialization

The first dependency invocation is through the second argument’s dispatch-table offset `0x0C`, with that argument as receiver. Its result is ignored. Shared-base acquisition follows this call, so changes it makes to globals can affect subsequent reads.

The function tests bit zero of the 32-bit guard field:

- If set, it skips both guard acquisition and the initialization-context global load associated with that attempt.
- If clear, it invokes F-b255b26284e6 with the guard-field address.

After F-b255b26284e6 returns, the initialization-context global is read **even if the dependency returned zero**. A zero result skips initialization after that load.

For a nonzero result, the sequence is:

1. Read the initialization-word global and write that 32-bit value to the retained initialization slot at offset `0x18`.
2. Clear the 32-bit field at the captured base plus `0x1C`.
3. Read the other initialization-argument global.
4. Invoke F-804adc29bbfb with the retained initialization-slot address, that newly read argument, and the previously captured initialization-context value.
5. Invoke F-d08d3ca17b81 with the captured base plus `0x08`.

These operands are global loads, not fixed constants. F-b255b26284e6 can change all of them before their respective reads. The context argument is captured before the initialization-word read and before the clear at `0x1C`; it is not reloaded for F-804adc29bbfb.

Guard synchronization guarantees, flag updates performed by dependencies, and behavior on abnormal exit are not established.

### Initial cached-word check

Using the captured base, the function reads offset `0x24` first. It reads offset `0x20` only if the first observed word is zero.

If both observed words are zero, it explicitly writes zero to `0x20`, then zero to `0x24`. A nonzero observed word prevents these writes. This is not an unconditional cache clear.

### Associated-object lookup

The first argument’s dispatch entry at `0x164` supplies an associated-object pointer.

- A null result selects the unavailable-object path without an eligibility call.
- A nonnull result receives a dispatch call at `0xEC`.
- A zero eligibility result also selects the unavailable-object path.

On that path, the order is specifically:

1. Clear the captured base’s word at `0x20`.
2. Load and retain the current processing-address global.
3. Clear the captured base’s word at `0x24`.
4. Invoke F-438680ffebfe with the captured processing address and a fixed opaque code-address argument.
5. Return low word zero and high word equal to the second argument.

The processing-address load occurs between the two clears, not after both. No representation getter or final submission is invoked on this path.

### Representation acquisition and decomposition

For an eligible object, dispatch offset `0x0C` returns a 64-bit representation token.

The function initializes two separate 32-bit temporary fields to zero, then invokes F-79fa759bfe87 with:

- Token bits `31:0`.
- Token bits `63:32`.
- The address of the source-like field.
- The address of the count-like field.

“Source-like” and “count-like” describe subsequent uses, not verified types. Only the count-like field is tested.

If its value at the test is zero, the function returns low word zero and high word equal to the second argument. It does not perform the subsequent token write, processing dispatch, or result retrieval.

Any nonzero bit pattern passes the test, including one with bit 31 set. There is no positivity test, bounds check, or visible count arithmetic.

Both field addresses have now been exposed. Later calls can potentially mutate them through retained access. Each subsequent use therefore means the value read for that particular operation, not an immutable decomposition result.

### Caching and processing dispatch

After a nonzero count observation, the function writes the entire 64-bit token into storage beginning at the captured base plus `0x20`.

It then obtains a dispatch table from the retained initialization slot and calls the entry at table offset `0x0C`. The receiver argument comes from a fresh load of the processing-address global.

The table source and receiver source are distinct. They are not established as one object.

Next, the associated object receives a dispatch call at `0xF4`:

- Zero selects the copied-input branch.
- Any nonzero result selects the alternate branch.

The token is already cached. Later failure or rejection does not trigger an explicit rollback.

The processing dispatch and mode dispatch occur after decomposition and before branch arguments are read. If retained output access is permitted, either call may change the source-like or count-like field. The earlier nonzero test is not repeated.

### Copied-input branch

The sequence is:

1. Invoke F-fab3799b3190 with the current count-like value and retain its 32-bit result.
2. Invoke F-unmapped-01 with that retained result and the source-like and count-like values read for this call.
3. Clear the high-return field.
4. Freshly load the processing-address global and invoke F-63011374610b with that address, the retained intermediate result, the current count-like value, and the high-return field’s address.
5. Invoke F-58ba61416cc9 with the retained intermediate result.

F-unmapped-01 is a documentary opaque ID for the unmapped three-argument dependency.

The count supplied in steps 1, 2, and 4 is not guaranteed identical: preceding dependencies may modify the address-exposed count field. Likewise, the source supplied in step 2 need not equal its value immediately after decomposition. A later zero count does not cause a new local rejection.

No local check rejects a zero intermediate result or validates the source-like value. The return values of F-unmapped-01, F-63011374610b, and F-58ba61416cc9 are not used.

Allocation, transfer, and release are plausible interpretations of this sequence, but ownership and memory-management behavior remain unknown. F-58ba61416cc9 is an explicit later call, not proof of deallocation.

### Alternate branch

The function invokes F-unmapped-02 with the source-like and count-like values read for that invocation.

- If its result is zero, it skips the two remaining preparation dependencies.
- Otherwise, it invokes F-9f8c9e1a456f with newly read source-like and count-like values.
- It then invokes F-5f2661b4a1e8 with a freshly loaded processing address, the source-like value read after F-9f8c9e1a456f, and that dependency’s retained result.

F-unmapped-02 is a documentary opaque ID for the unmapped two-argument predicate dependency.

The predicate’s zero result does not directly return failure. Both outcomes continue to result retrieval. The function does not explicitly clear or expose the high-return field in this branch.

### Retrieval, rejection, and submission

After either branch, the function freshly loads the processing-address global and passes that value to F-44362605ab79.

Consequently, the processing dispatch, F-63011374610b or F-5f2661b4a1e8, and F-44362605ab79 can receive different processing addresses. Unknown calls intervene between their loads; address stability is not a precondition established by this function.

If retrieval returns zero, F-a817d8d58c10 is invoked with no visible arguments. It is marked non-returning.

For a nonzero candidate:

1. F-1cb585febd84 receives the candidate.
2. Its 32-bit result is compared with `0xFFFFFFFF`, the signed `−1` bit pattern.
3. Exact equality rejects the candidate and returns low word zero.
4. Every other result, including zero, permits F-27684eb6b19e to receive the original second argument and the retained candidate.
5. If submission returns, the function returns low word one.

The submission result is ignored. In the copied-input branch, all these later calls occur while the exposed high-return field remains live; return packing uses its contents after those calls, not a snapshot taken immediately after F-63011374610b.

## Side effects

Directly visible effects include object dispatches, guarded initialization writes, conditional cached-word clears, an eight-byte token write, dependency calls, and optional final submission.

Three temporary-field addresses are exposed: source-like and count-like fields to F-79fa759bfe87, and the high-return field to F-63011374610b. This function supplies no ownership transfer or validity beyond their invocation lifetime. Whether dependencies retain or use these addresses during later calls is **documented_only**.

No input-object retain, release, or ownership transfer is established. Dependency calls may affect globals, shared fields, or reachable storage; this contract does not assume preservation.

The token store’s atomicity and physical subword ordering are not established. The guard pattern does not establish thread safety or reentrancy of later shared processing. Ordering described here concerns the supplied static representation, not additional hardware memory-order guarantees.

## Dependencies

| Opaque ID | Established local role |
|---|---|
| F-b255b26284e6 | Gates initialization following a clear flag-bit observation. |
| F-804adc29bbfb | Receives initialization storage and two globally sourced operands. |
| F-d08d3ca17b81 | Receives the captured-base guard address after initialization. |
| F-438680ffebfe | Receives the captured processing address after the unavailable-path clears. |
| F-79fa759bfe87 | Receives split token words and two writable temporary addresses. |
| F-fab3799b3190 | Produces the copied-input intermediate value from a count observation. |
| F-unmapped-01 | Receives the intermediate value and current source/count observations. |
| F-63011374610b | Receives copied-input arguments and the writable high-return address. |
| F-58ba61416cc9 | Receives the retained intermediate value afterward. |
| F-unmapped-02 | Gates alternate preparation, not final retrieval. |
| F-9f8c9e1a456f | Produces an alternate-preparation argument. |
| F-5f2661b4a1e8 | Receives a fresh processing-address load, current source, and preparation result. |
| F-44362605ab79 | Retrieves the candidate using another processing-address load. |
| F-a817d8d58c10 | Non-returning dependency for a zero candidate. |
| F-1cb585febd84 | Produces the exact rejection-sentinel comparison value. |
| F-27684eb6b19e | Receives the original second argument and accepted candidate. |

Dynamic dispatch targets have no supplied opaque identities. Calls sharing offset `0x0C` on different object roles must not be assumed identical.

## Acceptance examples

These are conditional criteria, not executed tests. Dependencies return normally unless otherwise stated.

1. **Guard declines initialization.** With the tested flag bit clear, F-b255b26284e6 returns zero. The initialization-context global is nevertheless read afterward. The initialization-slot write, `0x1C` clear, F-804adc29bbfb, and F-d08d3ca17b81 are skipped.

2. **Guard changes initialization operands.** If the guard dependency changes the operand globals before returning nonzero, the later reads use those changed contents. F-804adc29bbfb receives the context captured immediately after guard return, not a required immutable constant.

3. **Unavailable object.** A null lookup skips eligibility. A zero eligibility result skips representation acquisition. Either case clears `0x20`, captures the processing address, clears `0x24`, and calls F-438680ffebfe with that captured address. The return has low word zero and the second argument above it.

4. **Copied-input replacement output.** Suppose processing reaches the copied-input branch, F-63011374610b writes a chosen high-word value, and no subsequent retained-pointer mutation changes it. With a nonzero candidate and non-sentinel validation, submission occurs and the return contains that chosen high word above one.

5. **Copied-input untouched output.** Under the same successful retrieval conditions, if F-63011374610b leaves its output untouched **and no later call changes it through retained access**, the high return word is zero. Leaving it untouched during that call alone is insufficient.

6. **Count changes after allocation-like call.** If the initial count test observes a nonzero value, F-fab3799b3190 receives the count read at its invocation. If it then changes the exposed count field, the transfer dependency receives the changed count. There is no requirement that the two counts match and no repeated nonzero test.

7. **Processing address changes.** If the processing dispatch receives address A and an intervening call changes the processing-address global to B, a later processing call loads B. If another call changes it to C before retrieval, F-44362605ab79 receives C.

8. **Alternate predicate declines.** A zero F-unmapped-02 result skips both preparation calls but not retrieval. A subsequent acceptable nonzero candidate still permits submission and low word one.

9. **Rejection or non-returning path.** A nonzero candidate followed by validation result `0xFFFFFFFF` skips submission and returns low word zero. A zero candidate instead invokes F-a817d8d58c10 and has no specified ordinary return.

10. **High-bit count.** A count observation of `0x80000000` passes the initial nonzero test. Safe downstream interpretation is not established.

## Uncertainties

The following remain **documented_only**:

- Domain meanings and validity requirements of representations, addresses, counts, and candidates.
- Full contracts of dispatched and opaque dependencies.
- Whether output pointers are retained and which later calls can use them.
- Allocation failure, ownership, retention, and release semantics.
- Mutability and aliasing relationships among global values and shared objects.
- Synchronization, concurrent interference, reentrancy, and atomicity.
- Exception, nonlocal-exit, and cleanup behavior.
- The fixed code-address argument’s meaning.
- The mechanism by which F-a817d8d58c10 does not return.

No particular interargument machine evaluation order is inferred where multiple values occur within one call expression.

## Evidence

The supplied static representation establishes the explicit branches, widths, dispatch offsets, address exposures, global-read locations, shared writes, dependency argument roles, and packed-return construction described above.

It supports separate processing-address loads and post-guard initialization-operand loads. It does not support treating those operands as immutable or address-exposed temporary fields as stable across unknown calls.

No count multiplication, range calculation, or overflow check is visible. Token splitting and return packing operate on two 32-bit halves of 64-bit values.

No runtime execution, dependency implementation inspection, or independent dynamic validation has been performed. The acceptance examples are conditional behavioral obligations derived from static evidence.
