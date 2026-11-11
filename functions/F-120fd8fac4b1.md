<!-- Trusted provenance: F-120fd8fac4b1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-120fd8fac4b1

## Purpose

F-120fd8fac4b1 conditionally appends an entry to a state-associated table, obtains entry-related values through dependencies, and invokes state processing. Rejected operations and selected failures reach context-associated reporting.

The operation is not transactional. A descriptor mismatch retains the appended entry without assigning its name or completing processing. Other failures initiate rollback against the current last entry, which need not be the originally appended entry. Failure to obtain table storage sets a state byte and returns without reporting.

These are structural behaviors. The application-level meanings of the entries, descriptors, configuration bytes, and status values are not established.

## Interface

The function has three explicit inputs and no explicit return value.

| Input | Established representation and use |
|---|---|
| Context | A 32-bit address. Its 32-bit field at offset `0x18` supplies the state address. Reporting paths conditionally write a nonzero status at offset `0x34`. |
| Auxiliary value | An opaque 32-bit value saved locally; no subsequent semantic use is visible. |
| Argument-pair address | Addresses two consecutive 32-bit words, each read and passed separately to F-95b79f0a129e. |

The state address is captured before either conversion call. Later state accesses use that captured address, not a fresh read of the context’s state field.

Each null conversion result is replaced with the address of the same fixed object. Its contents, encoding, and ownership are unknown.

### State layout

| Offset | Width | Observed use |
|---|---:|---|
| `0x04` | 32 bits | Signed entry count |
| `0x08` | 32 bits | Table address |
| `0x10` | 32 bits | Initialization flags |
| `0x1c` | 8 bits | Nonzero enables the append attempt |
| `0x1e` | 8 bits | Set to one on selected failures |
| `0x1f` | 8 bits | First configuration argument |
| `0x20` | 8 bits | Second configuration argument |
| `0x6c` | 32 bits | Signed threshold component |
| `0x14c` | — | Offset identifying the embedded table |

Entries have a stride of `0x18` bytes.

| Entry offset | Width | Observed use |
|---|---:|---|
| `0x00` | 32 bits | Name-acquisition result and duplicate-comparison candidate |
| `0x04` | 32 bits | Handle-output location supplied to initialization |
| `0x09` | 8 bits | Written with `3` after name acquisition |
| `0x14` | 32 bits | Descriptor result |

Descriptor offsets `0x58` and `0x59` are accessed as individual bytes. Compatibility comparison uses unsigned values from offset `0x59`.

### Preconditions and arithmetic

All conditionally accessed addresses must support the indicated reads and writes. There are no explicit null checks for the context, argument-pair address, or captured state address.

Handle-derived storage must be valid whenever configuration is reached, including after descriptor lookup returns null. A zero initialization status does not independently establish that validity.

The representation uses 32-bit words and addresses. Count comparisons are signed. Threshold addition, count adjustment, size multiplication, and address formation have no visible overflow checks. Overflow interpretation, alignment requirements, and byte order remain documented_only. A negative count is not explicitly rejected.

No dependency is assumed to preserve state, globals, referenced storage, or ownership merely because preservation would make the operation conventional.

## Behavior

### Conversion and eligibility

The function initializes an optional-message slot and operation status to zero.

It reads and converts the first argument word through F-95b79f0a129e, then reads and converts the second word through the same dependency. Both calls complete before null-result substitutions.

Both conversions therefore occur even when eligibility will fail. The argument pair is not read atomically: the first conversion may affect storage read for the second.

Eligibility requires the signed count to be less than the signed threshold component plus two. Only when that comparison succeeds is the enable byte at state offset `0x1c` tested.

Failure of either condition reaches common reporting with status zero. Neither rejection directly writes the failure byte at `0x1e`.

### Duplicate scan

Eligible operations scan entries in ascending index order from zero. Each iteration reads the entry’s first 32-bit field. A nonzero field is compared with the second converted-or-fallback address through F-e6db2171781a. A zero comparison result immediately reaches common reporting without appending.

The loop condition rereads the state count. Entry access uses the state table address read for that iteration. Comparison calls may therefore affect subsequent bounds or table selection.

A negative count admits no scan iterations under the signed condition, but remains available to subsequent size and entry-address calculations.

### Storage acquisition and append

The current table address is compared with the captured state address plus `0x14c`.

For the embedded-table case, F-391e3d30e634 receives `0x48`. A zero result sets state byte `0x1e` to one and returns immediately. With a nonzero result, F-U01 receives:

- The returned storage address.
- The table address read from state after the storage call.
- `0x30`.

F-U01 is an unmapped dependency identified only within this document. Its arguments do not establish copying, preservation, initialization, or ownership transfer. Its return is unused.

For the other-table case, F-0ee6114cab5e receives the table address and the size expression formed from the current count plus one, multiplied by `0x18`. A zero result likewise sets byte `0x1e` to one and returns immediately.

Neither storage-failure path subsequently performs a direct table replacement, count increment, reporting call, or context-status write. Earlier dependency effects are not undone.

After a nonzero storage result, the function:

1. Installs that result as the state table address.
2. Captures an entry address using that result and the current count times `0x18`.
3. Increments the state count.
4. Calls F-U02 with the captured entry address and `0x18`.

F-U02 is a distinct unmapped dependency. No zero-fill, initialization, bounded-write, or ownership behavior is established.

The captured entry address remains in use after dependency calls. If a dependency replaces the state table, subsequent writes through the captured entry address do not automatically follow that replacement. Continued validity of the captured storage is a conditional requirement.

### Initialization and descriptor handling

F-94e50f72111d receives, in order:

1. The captured state address.
2. The first converted-or-fallback address.
3. Zero.
4. Decimal `2000`.
5. The state flags at `0x10`, with bit `0x100` set in the argument.
6. The address of the captured entry’s handle field at `0x04`.

The flag expression does not directly modify the state flags. The dependency’s return becomes the operation status.

A nonzero status skips descriptor lookup and configuration but still proceeds to name acquisition.

For zero status, F-96819d3b47ed receives the state and the handle read from the captured entry. Its result is stored at entry offset `0x14`. A null result also sets operation status to `7`.

For a nonnull descriptor:

- A zero byte at descriptor offset `0x58` bypasses compatibility comparison.
- A nonzero byte causes comparison of its unsigned byte at `0x59` with the corresponding byte of the descriptor referenced by the first entry in the current state table.

The first entry’s descriptor has no local null check.

A mismatch reaches common reporting with status zero. The appended count and descriptor assignment remain. Configuration, name acquisition, the marker write, processing, and rollback are skipped.

Without a mismatch, configuration proceeds even if lookup returned null and set status `7`. The function reads the pointer at handle offset `0x04`, then the first 32-bit word at that pointer. That word is captured as the configuration target.

F-36f4d8c04634 receives the target and state byte `0x1f`. F-5c4fede89f8c then receives the same captured target and state byte `0x20`. The second byte is read after the first call, so it can reflect changes caused by that call.

### Name acquisition and processing

Except on the mismatch branch, F-8ad0042eaec1 receives the state and second converted-or-fallback address. Its result is stored at captured-entry offset `0x00` without a local failure check. Entry byte `0x09` is then written with `3`.

If operation status is zero, the following calls occur in order:

1. F-b3e71981d63f with the state.
2. F-811e6eb087fc with the state and the address of the optional-message slot.
3. F-a1cc018ab664 with the state.

F-811e6eb087fc supplies the new operation status and may write the message slot. F-a1cc018ab664 runs regardless of that returned status.

A zero status then returns immediately. There is no direct context-status clear, failure-byte clear, reporting call, or message-disposal call on that path.

Exposure of the message-slot address does not establish that F-811e6eb087fc stops using it when its call returns.

### Rollback after nonzero status

A nonzero status reaching rollback causes the current state count minus one to be captured as the last-entry index. This need not be the original append index.

The handle at that index in the current state table is tested. If nonzero:

1. F-7446dea3e07b is called with no explicit arguments shown.
2. The state table address is reread for a zero write to the handle field at the captured index.
3. The descriptor field at that index is also written with zero through the state-referenced table.

The handle test and subsequent clears need not address the same table. Each clearing write is expressed through the state table field rather than the original captured entry address.

If the tested handle is zero, both clears are skipped, even if the descriptor field is nonzero.

F-aacdb1821b51 then receives the state and zero. Afterward, the count is written with the previously captured last-entry index, overriding any count change made by that call.

The failure byte at `0x1e` is set to one for status `7`, or when the represented 32-bit status-minus-`0xc00` comparison equals decimal `10`. In ordinary non-overflowing arithmetic, the latter status is `0xc0a`. Other statuses do not directly set that byte here.

All rollback paths reach common reporting.

### Common reporting

Every common-reporting path calls F-ea27adaa2931 with decimal `128` and the address of local byte storage. This occurs before selecting between local storage and the optional message, even if a supplied message will be used.

The optional-message slot is tested after that call:

- **Zero slot:** The final byte of the represented 128-byte local region is written with zero. F-d8267ca9780b receives the saved context, the local-region address, and `0xffffffff`. No F-94cb53877c15 call follows on this branch.
- **Nonzero slot:** The slot is read to supply the message argument to F-d8267ca9780b, together with the saved context and `0xffffffff`. After reporting returns, the slot is read again and that newly read value is passed to F-94cb53877c15. There is no second null check.

The reporting and disposal-positioned calls are not guaranteed to receive the same message address. The slot’s address was previously exposed to F-811e6eb087fc; retained access and later mutation are not excluded. In particular, F-d8267ca9780b cannot be assumed to preserve the slot. F-94cb53877c15 may receive a different address or zero.

Branch selection is not repeated after reporting. Conversely, if the zero-slot branch was selected, a later slot mutation does not cause a disposal-positioned call.

After reporting and any F-94cb53877c15 call, a nonzero operation status is written as 32 bits at context offset `0x34`. Zero status leaves that field directly unchanged.

## Side effects

Direct writes include table replacement, count increment, rollback count assignment, descriptor and name assignments, the byte-`3` marker, selected handle and descriptor clears, selected failure-byte assignments, local report termination, and conditional context-status assignment.

Rollback does not directly restore the former table allocation or clear the removed entry’s name field or marker.

Ownership is only partially observable:

- A nonzero storage result is installed as the table address; allocation lifetime and former-table treatment are not independently established.
- F-U01 and F-U02 have unknown memory and ownership effects.
- Name acquisition may return allocated, borrowed, shared, or otherwise managed storage.
- F-7446dea3e07b is not explicitly passed the tested handle; its cleanup scope is unknown.
- The nonzero-message reporting branch passes a **post-report reread** of the message slot to F-94cb53877c15. This does not establish disposal of the address used for reporting, or even a nonzero argument.
- Calling F-94cb53877c15 does not by itself prove deallocation.
- The zero-slot reporting branch does not pass its local report region to F-94cb53877c15.

Dependencies may have additional effects not visible here.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-95b79f0a129e | Converts each argument word |
| F-e6db2171781a | Compares an existing nonzero name value with the proposed name |
| F-391e3d30e634 | Receives storage request `0x48` |
| F-0ee6114cab5e | Receives table address and calculated size |
| F-U01 | Receives storage result, subsequently read table address, and `0x30` |
| F-U02 | Receives captured entry address and `0x18` |
| F-94e50f72111d | Receives initialization arguments and handle-output address; returns status |
| F-96819d3b47ed | Supplies the descriptor result |
| F-36f4d8c04634 | Receives captured target and first configuration byte |
| F-5c4fede89f8c | Receives captured target and second configuration byte |
| F-8ad0042eaec1 | Supplies the stored name value |
| F-b3e71981d63f | First processing call |
| F-811e6eb087fc | Returns processing status and receives the message-slot address |
| F-a1cc018ab664 | Final processing call |
| F-7446dea3e07b | Called after a nonzero rollback-handle test |
| F-aacdb1821b51 | Receives state and zero before rollback count assignment |
| F-ea27adaa2931 | Receives `128` and local report storage |
| F-d8267ca9780b | Receives context, selected report address, and `0xffffffff` |
| F-94cb53877c15 | Receives the message-slot value reread after nonzero-branch reporting |

F-U01 and F-U02 are distinct document-local identifiers for unmapped dependencies.

## Acceptance examples

These are conditional static cases, not executed tests. Unless stated otherwise, arithmetic is non-overflowing, referenced storage remains valid, and dependencies introduce no additional state changes.

1. **Count at limit.** Threshold component `4` and count `6` cause rejection after both conversions. The enable byte need not be read. Common reporting runs with status zero.

2. **Disabled append.** Count below threshold and enable byte zero cause reporting without storage acquisition or a direct failure-byte write.

3. **Duplicate.** An existing nonzero name whose comparison returns zero prevents append. Context status is not directly overwritten.

4. **Embedded storage succeeds.** A nonzero storage result causes F-U01 to receive that result, the subsequently read table address, and `0x30`. No copying outcome is required.

5. **Storage fails.** A zero storage result sets failure byte `0x1e` to one and returns without subsequent direct count adjustment or reporting.

6. **Entry preparation.** Table installation and count increment precede F-U02. Existing entry bytes are not required to become zero.

7. **Successful processing.** With successful initialization, valid accesses, no mismatch, and zero processing status, name assignment and the byte-`3` write occur. Return bypasses reporting and direct context-status assignment.

8. **Descriptor mismatch.** Reporting occurs with zero status after descriptor assignment. The increased count remains; name assignment and rollback do not occur.

9. **Initialization status `7`.** Name acquisition and marker assignment still occur. Processing is skipped; rollback, failure-byte assignment, reporting, and context-status assignment follow.

10. **Null descriptor.** A null lookup result is stored and sets status `7`, but handle-derived configuration is still attempted. Valid handle-derived storage remains required.

11. **Rollback table replacement.** If the captured last index is `2` and F-7446dea3e07b replaces the table after a nonzero handle test, subsequent clears target index `2` through the replacement table. No direct clear of the tested old entry is established.

12. **Zero rollback handle.** A zero tested handle skips both clearing writes, even with a nonzero descriptor.

13. **Message-slot mutation during reporting.** After nonzero processing status and rollback, the report-builder call still occurs. Suppose the message slot is nonzero when selected and F-d8267ca9780b receives address A. If reporting changes the slot through previously retained access to address B, F-94cb53877c15 receives B. If it changes the slot to zero, F-94cb53877c15 receives zero without another null check. Only conditional slot preservation makes the two arguments equal. Context-status assignment follows that call.

## Uncertainties

- **documented_only — Unmapped dependencies:** F-U01 and F-U02 have no established copy, fill, preservation, or ownership semantics.
- **documented_only — Arithmetic:** Overflow behavior and storage-size interpretation are unresolved. Negative counts are not rejected.
- **documented_only — Mutation and validity:** Dependencies may change counts, tables, globals, referenced storage, or the exposed message slot. Captured addresses do not establish continued storage validity.
- **documented_only — Message-slot access:** Retention and later use of the exposed slot address are neither established nor excluded. Reporting-to-disposal argument identity is not guaranteed.
- **documented_only — Representation and reporting:** Alignment, byte order, message encoding, builder write bounds, and the interpretation of `0xffffffff` are unknown.
- **documented_only — Ownership and numeric meanings:** Acquisition and release obligations, and the meanings of status and configuration values, are unresolved.
- **Inferred intent:** The operation resembles named-resource registration with compatibility checking and partial rollback. This interpretation is not required by the established behavior.

## Evidence

Established behavior derives from the supplied static representation: access widths and offsets, explicit arguments, signed conditions, captured values, state rereads, writes, and call order.

The message-slot address is passed to F-811e6eb087fc. On the nonzero-message reporting branch, the slot supplies the reporting argument and is read again afterward for F-94cb53877c15, without another null check. Those separate reads establish no argument-identity guarantee across reporting.

No runtime tests have been performed. Acceptance examples describe conditional consequences of static evidence, not observed executions. Dependency internals and unresolved ownership, mutation, and arithmetic behavior remain documented_only.
