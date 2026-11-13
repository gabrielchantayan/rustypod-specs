<!-- Trusted provenance: F-83acbc8ae719; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-83acbc8ae719

## Purpose

F-83acbc8ae719 dispatches commands against a mutable object. It can answer scalar queries, construct a four-word shared result, attempt iterative acquisition into embedded object storage, or invoke a fallback construction path.

The return value reports whether the request was handled. A separate caller-provided 32-bit output slot carries the result. Depending on the selected path, that result is zero, a scalar word, or an address.

This contract describes behavior established by the supplied static representation. Resource meanings, dependency implementations, ownership protocols, and synchronization guarantees are not established.

## Interface

### Arguments and return

The represented interface has four arguments:

1. **Object address:** points to object state, slot records, an indirect-dispatch pointer, and embedded result storage.
2. **Selector:** a 32-bit token compared against externally stored values. This function does not invoke the selector.
3. **Command:** a signed 32-bit dispatch value.
4. **Output-slot address:** points to writable storage for one 32-bit result.

Normal returns are:

- **1:** a handled path wrote the output slot.
- **0:** no handled result was established.

A zero return does not imply that the output was preserved. The processing-command path clears it before checking eligibility.

No explicit null-pointer checks protect the object, output slot, resource addresses, or dispatch-table accesses. Valid execution requires every reached read, write, and indirect call to have suitable storage or a valid target at the time it occurs. Dependencies may impose further requirements.

The represented address and word width is 32 bits. Byte fields are 8 bits; the descriptor discriminator is 16 bits.

### External values

The following names distinguish uses without asserting distinct numeric values:

- **Processing command:** externally stored command checked before numeric dispatch.
- **Control selector:** selects scalar control queries.
- **Descriptor selector:** selects shared-result construction.
- **Acquisition selector:** selects acquisition processing and fallback construction.
- **Completion selector:** selects the completion query and is passed to the completion callback.
- **Resource-table base:** externally stored address of 32-bit slot-resource entries.
- **Shared-result base:** externally stored address of at least four writable words.
- **Candidate source, candidate argument, and fallback resources:** external values used by the corresponding paths.

Their values, equality relationships, and stability across calls are unknown.

### Object layout

Offsets are in bytes from the object address.

| Offset | Observed use |
|---|---|
| 0 | Address of an indirect-dispatch table |
| 24 | Embedded state passed to entry and exit dependencies |
| 32, low byte | Mode |
| 232, low byte | Acquisition-enabled state |
| 234 | One-byte processing eligibility flag |
| 236, low byte | Current slot index |
| 248 | Embedded result storage |
| 252 | Word tested after acquisition conversion |

Slot records have a 24-byte stride. For slot index \(s\):

| Offset | Observed use |
|---|---|
| \(36 + 24s\) | Count-like 32-bit word |
| \(44 + 24s\) | Signed range-start word |
| \(48 + 24s\) | Signed range-end word |
| \(52 + 24s\) | Cursor word |
| \(56 + 24s\), low byte | Completion flag |

The initial processing check accepts indices 0–3. Later current-index reads are not independently range-checked.

The representation uses signed-byte conversions for several fields, including the current index, and an explicitly unsigned byte for the acquisition-selection enabled test. Exact machine-level extension semantics remain **documented_only** because the supplied representation reports unsettled type propagation.

## Behavior

### Dispatch and scalar queries

Processing-command equality has precedence over all numeric handlers.

Otherwise, signed command comparison against the processing-command value partitions numeric dispatch:

- The greater-than-or-equal partition recognizes `0x7f81` and `0x7f82`.
- The lower partition recognizes `0x7e80` and `0x7e81`.

These handlers are therefore conditional on the external processing-command value. A numeric command equal to the processing command instead enters processing.

When its handler is reachable:

- **`0x7e81`, control selector:** write zero and return 1.
- **`0x7f81`, control selector:** write zero if acquisition-enabled is nonzero, otherwise one; return 1.
- **`0x7f82`, completion selector:** require a current index in 0–3, then write zero or one according to whether that slot’s completion byte is zero or nonzero; return 1. An invalid index proceeds to fallback without a query-result write.
- **`0x7e80`, control selector:** a valid completed slot produces one. Otherwise, acquisition-disabled state produces one. Otherwise, a valid slot produces its count word, and an invalid slot produces zero. Write that result and return 1.

Selector mismatches and unrecognized commands proceed to fallback.

### Processing entry and eligibility

For the processing command, the output slot is first cleared to zero. F-338d16f9ce72 is then called with the embedded address at offset 24.

Eligibility requires, in order:

1. Mode equal to 2 or 3.
2. Nonzero eligibility byte at offset 234.
3. Current slot index in 0–3.
4. Nonzero count word for that slot.

The represented mode expression can read the mode again when its first comparison does not find 2.

Eligibility failure calls F-1333331b2da1 with the same embedded address, then proceeds to fallback. Eligible paths also call this exit dependency after their selector-specific work. The calls suggest synchronization entry and exit, but locking behavior is not established.

The checked slot index is retained for acquisition’s slot-record accesses. Descriptor processing and some acquisition arguments instead reread the current index.

### Descriptor processing

An eligible descriptor-selector match performs the following observable sequence.

First, F-0260b20d4c56 receives the object address. One 32-bit word is read from its returned address. F-1e7091dc86a0 receives a temporary handle location and that word. The function then reads the handle location and records whether it is nonzero.

**No initialization of that handle location is visible before the call.** Defined execution requires the subsequent handle read to be valid and initialized; the supplied evidence does not establish that F-1e7091dc86a0 satisfies this requirement.

A separate temporary word is explicitly initialized to zero and passed to F-01b37b84d143. Immediately after that call, the function captures the resource-table base from its external location. **This capture occurs even when the recorded handle test was false.**

Construction proceeds only when the earlier recorded test was nonzero:

1. The resource-table base is read again from its external location. Using this fresh base and a freshly read current index, one table entry is passed to F-9d06d181244d together with a temporary-region address.
2. After that call, two 32-bit words and a 16-bit discriminator are read from the temporary region. No prior initialization of those fields is visible. Their initialization is a requirement for defined execution, not a proven dependency guarantee.
3. The pair is retained in its observed order, except that discriminator `0x005a` or `0x010e` reverses it.
4. Four other temporary words are explicitly zeroed: two primary callback-result words and two auxiliary words.
5. F-47c987e17054 receives the earlier resource word and a cursor selected using another current-index read.
6. The handle location is reread. If nonzero, its referenced word supplies an interface address; otherwise the interface address becomes zero.
7. F-beb6c6f57d68 receives a resource-table entry selected using **the earlier captured table base**, not the fresh base used for F-9d06d181244d. Its index is freshly read.
8. An indirect callback selected at byte offset 36 of the interface’s dispatch table receives the interface address, the preceding two dependency results, and addresses of the two primary temporary result words.

There is no validity check before dereferencing the interface dispatch table. The earlier handle test does not establish that the later handle, referenced interface, or callback target remains valid.

The two table-base observations are behaviorally distinct. An intervening dependency can change the externally stored base: F-9d06d181244d can then receive an entry from the new table while F-beb6c6f57d68 receives an entry from the previously captured table.

After the callback, the function reads a candidate word from offset 4 of an externally referenced block:

- A set low bit retains the entire candidate, without masking.
- A clear low bit causes F-b255b26284e6 to be called with an external argument.
- A zero return from that dependency makes the candidate zero.
- A nonzero return causes F-d08d3ca17b81 to supply the candidate.

These calls occur before the primary callback-result words determine the result form. If the first primary word is nonzero, the second primary word replaces the candidate.

The function captures the shared-result base and performs four ordered word writes:

- **Both primary words nonzero:** write zero, zero, the second primary word, and the first primary word. Then call F-cca11044abb5 with the externally read shared-result base and the first auxiliary-word address.
- **Otherwise:** write the two auxiliary words, followed by the selected resource-derived pair.

The first shared-result write uses the external base directly; the subsequent three use the captured base. No dependency call intervenes between those writes. The captured base is written to the caller’s output slot, and handled state becomes one.

F-01b37b84d143 is subsequently called on the handle location, including when construction was skipped. Normal exit processing follows. Its role must not be assumed to be release or destruction.

### Acquisition processing

This eligible path requires an acquisition-selector match and a nonzero unsigned acquisition-enabled byte.

F-0260b20d4c56 receives the object address, and one word is read from its returned address. A signed 32-bit failure counter starts at zero. A signed 32-bit iteration position starts at the checked slot’s range-start word.

An iteration executes only while the position is no greater than that slot’s currently read range end and no accepted result has been recorded.

Each iteration calls F-fbe427d8956f with:

- The resource word.
- A temporary acquisition-result address.
- The checked slot’s current cursor.
- A table entry obtained using a fresh resource-table base and fresh current index.
- A temporary flag-word address.

The call returns a status. Neither the flag word nor the acquisition-result region has visible initialization before the first call. The flag is subsequently read; the result region can be passed to conversion. Defined execution therefore requires appropriate initialization before those uses. Whether this dependency provides it is unknown. The flag is not reset at the start of every iteration.

If the flag’s low byte is nonzero and status is zero, F-8c586dbad7fd receives the temporary result address and embedded storage at offset 248. The function then reads offset 252:

- **Zero:** explicitly clear the **entire 32-bit temporary flag location** to zero, then proceed directly to the cursor-boundary comparison.
- **Nonzero:** reread the flag’s low byte. If nonzero, reset the failure counter, set handled state to one, and write the embedded-storage address to output. If zero, follow ordinary status-zero handling.

The full-word flag clear is observable storage modification. Its address was supplied to F-fbe427d8956f; retained access cannot be excluded.

Status handling is:

- **0 or 8:** compare the checked slot’s cursor with its range start.
- **1:** increment the failure counter and compare it against the checked slot’s count minus one. If the counter is at least that threshold, set that slot’s completion byte to one, invoke the object callback at dispatch-table offset 88 with the completion selector, and leave the loop. Otherwise call an unmapped dependency with argument `0x32`.
- **Other values:** no additional status-specific action is visible.

If the cursor-boundary comparison finds equality, clear the acquisition-enabled byte, call F-9d39dcd3197c with the object and a freshly read current index, and leave the loop.

Otherwise, call F-7d6c4bec2818 with the checked slot-record address, then increment the iteration position. This also happens after accepting a result; handled state prevents another iteration.

After normal exit processing, an accepted result returns 1. Range exhaustion, completion notification, and boundary termination do not independently establish handled state.

### Fallback

Fallback returns zero unless the saved command is `0x7f80` and the saved selector equals the acquisition selector read for this check.

For a match:

1. Call F-9d0ecc5f6adf without explicit arguments.
2. Pass its result and two external fallback resources to F-5710a8628cba.
3. Pass that result, embedded storage at offset 248, and two zero arguments to F-4225f441f48b.
4. Write the embedded-storage address to output and return 1.

No constructor success or error result is checked.

## Side effects

Direct writes include output-slot assignments, descriptor temporary initialization, the four shared-result writes, the full-word acquisition flag clear, completion-byte assignment to one, and acquisition-enabled assignment to zero.

Dependencies receive temporary addresses, object addresses, and shared-storage addresses. Their additional writes, retained pointers, allocation, release, and global mutations are unknown. Explicit initialization establishes a value at the write, not preservation across later calls.

The embedded result belongs spatially to the supplied object. The shared result is externally addressed storage. Neither returned address establishes allocation, ownership transfer, or a caller release obligation.

No non-aliasing check exists. If output overlaps object or shared state, output writes can affect later accesses; ordinary independent-storage examples below exclude such overlap.

No rollback or exception cleanup is visible. A dependency that does not return normally can bypass the exit dependency.

## Dependencies

Roles below describe observed use, not implementation guarantees.

| Opaque ID | Observed use |
|---|---|
| F-338d16f9ce72 | Entry call on embedded state |
| F-1333331b2da1 | Normal exit call on embedded state |
| F-0260b20d4c56 | Returns an address subsequently read for one word |
| F-1e7091dc86a0 | Receives handle storage subsequently read |
| F-01b37b84d143 | Receives temporary word or handle storage |
| F-9d06d181244d | Receives storage subsequently read as pair and discriminator |
| F-47c987e17054 | Returns a callback argument derived from resource and cursor |
| F-beb6c6f57d68 | Returns a callback argument using the captured table base |
| F-b255b26284e6 | Controls alternate candidate retrieval |
| F-d08d3ca17b81 | Returns alternate candidate |
| F-cca11044abb5 | Receives shared result and auxiliary storage |
| F-fbe427d8956f | Returns acquisition status; receives result and flag storage |
| F-8c586dbad7fd | Receives acquisition temporary and embedded storage |
| F-7d6c4bec2818 | Receives the checked slot record |
| F-9d39dcd3197c | Receives object and freshly read index on boundary exit |
| F-9d0ecc5f6adf | Returns first fallback input |
| F-5710a8628cba | Returns combined fallback input |
| F-4225f441f48b | Receives embedded fallback destination |

The two indirect callbacks and the dependency receiving `0x32` have no supplied opaque IDs. Their behavior is **documented_only**.

## Acceptance examples

These are conditional static cases, not executed tests. They assume valid reached accesses and initialized consumed data.

1. **Handled zero:** reachable `0x7e81` with the control selector writes zero and returns 1.

2. **Invalid index:** reachable `0x7e80`, control selector, enabled acquisition, and an invalid index produces zero with return 1. The corresponding invalid-index `0x7f82` query instead reaches fallback.

3. **Ineligible processing:** a mode other than 2 or 3 causes output clearing, entry call, then exit call. Fallback remains independently conditional.

4. **Handle test false:** if the post-call handle read is valid and zero, the function still initializes the separate temporary, calls F-01b37b84d143 on it, and captures the table base. It skips descriptor construction, then calls F-01b37b84d143 on the handle location.

5. **Different table bases:** suppose the captured table base is A, but a later external read yields B. With construction enabled, F-9d06d181244d uses B; F-beb6c6f57d68 uses A. Their separately read indices need not match.

6. **Pair reversal:** discriminator `0x010e` reverses the pair. If primary callback results remain zero and dependencies do not alter the selected pair, the reversed pair occupies the last two shared-result words.

7. **Conversion rejection:** status zero and a nonzero low flag byte enter conversion. If offset 252 is then zero, the entire temporary flag word becomes zero before cursor comparison. Unequal cursor and range start continue through slot update and position increment.

8. **Accepted acquisition:** if conversion leaves offset 252 nonzero and the reread flag byte nonzero, output becomes the embedded address. Slot update still occurs before normal exit and return 1, assuming the position increment is defined.

9. **Completion without acceptance:** count one and the first status-one result cause counter value one to meet threshold zero. The completion byte is set and callback invoked; the `0x32` call is skipped. This alone does not establish handled state.

10. **Fallback:** matching `0x7f80` and acquisition selector call the three fallback dependencies in order and return the embedded address with return 1, without checking construction success.

## Uncertainties

The following remain **documented_only**:

- External values, equality relationships, and concurrent mutation rules.
- Whether dependencies initialize all subsequently consumed temporary fields.
- Full temporary layouts and dependency access extents.
- Synchronization, ownership, retention, allocation, release, and error behavior.
- Exact signed-byte extension under the reported type uncertainty.
- Application meaning of statuses, resources, and the `0x32` argument.
- Shared-result lifetime and concurrent accessibility.

Failure-counter increment, count-minus-one, and position increment use represented signed 32-bit arithmetic. Overflow behavior is not established. Defined execution requires avoiding overflow unless lower-level evidence supplies different semantics.

The initial slot check does not validate later index reloads. Every later table and object access must remain valid independently.

## Evidence

Established evidence consists of the supplied function representation and dependency mapping: explicit widths and offsets, conditional branches, read/write order, address arguments, and normal call sequence.

“Acquisition,” “completion,” “descriptor,” and “synchronization” are inferred role names. Calls followed by temporary reads establish initialization requirements, not proof that unknown callees meet them. Dependency effects and unresolved representation details remain explicit rather than assumed away.

No runtime tests have been performed.
