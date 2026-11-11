<!-- Trusted provenance: F-6b47dda9a4e5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6b47dda9a4e5

## Purpose

F-6b47dda9a4e5 dispatches a byte-oriented record according to its first payload byte and its exact payload length. It first obtains a classification through a dependency, then reads the record descriptor and checks entry conditions. Ordinary completion returns either the 32-bit value 4 or a command dependency’s result.

Selector 4 additionally performs conditional global-state operations and indirect calls before its main command call, followed by another dependency call after that command returns. Selectors 0 and 15 invoke the same dependency using different input encodings.

These are call-site behaviors established by the supplied static representation. Command meanings, classification meanings, and the application-level interpretation of return value 4 are not established.

## Interface

### Arguments and descriptor layout

| Input | Representation | Use |
|---|---|---|
| Context | Opaque 32-bit value | Supplied to the selected main command dependency. |
| Descriptor | Address in the represented 32-bit address model | Supplied to F-5f5f14cedda3, then used for direct field reads. |

The directly accessed descriptor fields are:

| Byte offset | Width | Interpretation |
|---|---:|---|
| +8 | 32-bit address | Payload base address. |
| +12 | Unsigned 16-bit integer | Payload length in bytes. |

The payload’s byte at offset 0 is the selector. Other payload offsets in this contract are relative to the payload address retained after the classification call.

The descriptor must be usable by F-5f5f14cedda3 and accessible for the direct field reads. No descriptor-null check is established before that call. The subsequent payload-null and minimum-length checks do not prove that payload memory is accessible or that the stated length describes its actual allocation.

### Result and fatal conditions

The ordinary result is a 32-bit value:

- 4 for unsupported selectors, selectors 5 and 12, and exact-length mismatches.
- The selected main dependency’s result for a matching implemented command, provided all required subsequent operations return normally.

After classification and descriptor-field reads, F-a817d8d58c10 is invoked if:

- The observed payload address is null;
- The observed unsigned length is less than 2; or
- The classification, interpreted as signed 32-bit, is greater than 2.

F-a817d8d58c10 is represented as non-returning. These paths do not ordinarily return 4.

The classification test is signed. Bit patterns representing 3 through 2,147,483,647 fail it; bit patterns representing negative signed values do not fail it merely because their unsigned interpretation exceeds 2.

Entry checks concern the descriptor values read **after** F-5f5f14cedda3 returns. Initially null or short fields do not alone establish a fatal outcome, because that dependency may change them.

## Behavior

### Entry observations and retained state

The established entry order is:

1. Call F-5f5f14cedda3 with the descriptor and retain its 32-bit classification result.
2. Read and retain a global pointer used later for selector-4 flag-byte-0 accesses.
3. Read the descriptor’s unsigned 16-bit length.
4. Read the descriptor’s payload address.
5. Apply the fatal conditions.
6. Read the selector from the retained payload address.

The classification call therefore occurs even for records whose subsequently observed fields fail validation. Its mutations can affect both field reads and the global pointer captured afterward.

The length and payload address are retained for dispatch. Later descriptor changes do not cause these fields to be reloaded. Payload contents are not generally snapshotted: command bytes are read when their branch reaches the corresponding use.

### Command selection and layouts

Every listed length is the required **total** payload length, including the selector. Matching only a prefix is insufficient.

“Decode at +1” means supplying that payload address to the named dependency. It does not independently establish the dependency’s byte order, complete read footprint, or return-value domain.

| Selector | Required length | Behavior when length matches |
|---:|---:|---|
| 0 | 5 | Decode at +1 using F-c25a2fcc75b5; call F-fe1b1452b557 with context and the decoded value. |
| 1 | 5 | Decode at +1 using F-c25a2fcc75b5; call F-e25e25f723f0 with context and the decoded value. |
| 2 | 3 | Decode at +1 using F-64e2dfdd4a5d; call F-aa3cba10a94f with context and the decoded value. |
| 3 | 2 | Call F-9fc356bbe31f with context and byte +1. |
| 4 | 4 | Perform the global-state behavior below; call F-a6d44bb3a0a1 with context, bytes +1 through +3, and classification. |
| 5 | No command-specific requirement | Return 4. |
| 6 | 6 | Decode at +1 using F-c25a2fcc75b5; call F-3867c434fcca with context, decoded value, byte +5, and classification. |
| 7 | 3 | Call F-0b77559fa811 with context, bytes +1 and +2, and classification. |
| 8 | 3 | Call F-adf871004f14 with context, bytes +1 and +2, and classification. |
| 9 | 7 | Decode at +1 using F-64e2dfdd4a5d; zero-extend byte +6 into 32-bit scratch storage; call F-bb85056f54e1 with the visibly represented arguments context, decoded value, and bytes +3 through +5. |
| 10 | 5 | Store classification in 32-bit scratch storage; call F-55290c3145ab with the visibly represented arguments context and bytes +1 through +4. |
| 11 | 3 | Call F-adb56aabcc86 with context, bytes +1 and +2, and classification. |
| 12 | No command-specific requirement | Return 4. |
| 13 | 3 | Call F-d9032d50340b with context, bytes +1 and +2, and classification. |
| 14 | 3 | Call F-280d7b6ddabc with context, bytes +1 and +2, and classification. |
| 15 | 3 | Decode at +1 using F-64e2dfdd4a5d; transform the result as described below; call F-fe1b1452b557 with context and the transformed value. |
| 16 | 5 | Store classification in 32-bit scratch storage; call F-6ae62d4262ef with the visibly represented arguments context and bytes +1 through +4. |

All other selector values return 4 after successful entry validation. An implemented selector with a mismatching length returns 4 without its decoder, main command dependency, or selector-4 preparation.

For selectors 6 and 9, the decoder returns before the later payload-byte reads. Decoder mutations can therefore affect those bytes. Relative ordering among multiple byte reads appearing as arguments to one call is not independently established.

The scratch stores for selectors 9, 10, and 16 are explicit, but their relationship to outgoing arguments is unresolved. They must not be omitted from the evidence or promoted into definite additional parameters without calling-convention evidence.

### Selector 15 arithmetic

After F-64e2dfdd4a5d returns, its result is used as a signed 32-bit value in multiplication by 1,000. The product is combined with a global mask using bitwise AND, and the resulting 32-bit value is passed to F-fe1b1452b557.

For a decoded signed value whose product is representable in signed 32-bit arithmetic, the supported transformation is that product’s 32-bit bit pattern ANDed with the observed mask.

The supplied material does not establish the decoder’s range or behavior when the signed product is not representable. Wraparound, trapping, saturation, and other overflow policies are **documented_only**, not selected by this contract. The mask’s application-level purpose is also unknown.

The mask is evaluated after decoding; it must not be treated as an entry-time snapshot. Selectors 0 and 15 share a main dependency but have different lengths and transformations.

### Selector 4: flag addressing and global operands

Three kinds of state must be distinguished:

- **Retained flag pointer:** captured at entry, after classification. Every direct flag-byte-0 read or write uses this retained pointer.
- **Fresh flag pointer evaluation:** the initial flag-byte-1 test reevaluates the global pointer and reads byte +1 through that newly observed pointer.
- **Opaque global operands:** values used in comparison preparation and receiver operations. Some are retained; others are read again at later call sites.

The two flag accesses must not be modeled as necessarily addressing the same object. If the global pointer differs between its entry capture and its selector-4 reevaluation, flag byte 1 is tested through the fresh pointer while flag byte 0 is accessed through the retained one.

### Selector 4: preparation and conditional work

This behavior applies only when the retained length is 4.

**Initial flag handling.** The function reads flag byte 1 through the fresh global-pointer evaluation. If that byte is zero, it calls F-cdd33263ee89, obtains an indirect target through the returned object’s dispatch table at byte offset 184, and invokes it. After that invocation returns, it writes 1 to flag byte 0 through the retained pointer. If flag byte 1 is nonzero, this call sequence and write are skipped.

**First comparison.** F-7a117b899bf6 receives the first eight-byte temporary receiver region. After this call returns, an opaque comparison-source global is read and retained. F-04f46bd4cf22 receives comparison storage and a separately evaluated read of that same global.

The first receiver’s dispatch-table operation at byte offset 12 is then invoked with the receiver and output storage. F-927372e9397c derives another temporary from that output and a different global operand. F-07039e49888b compares the derived temporary with the prepared comparison storage.

If this comparison returns zero, the remaining conditional preparation is skipped. No flag-byte-0 write is added by this comparison path, although the earlier initial-flag handling may already have written it.

**Nonzero first comparison.** F-04f46bd4cf22 prepares the comparison storage again, this time using the retained comparison-source value. The first receiver’s offset-12 operation runs again to refresh its output. Only afterward is flag byte 0 read through the retained pointer.

A nonzero flag byte 0 selects the additional-operation branch without another F-07039e49888b call. If flag byte 0 is zero, F-927372e9397c derives a separate temporary using the refreshed output and a third global operand. A second F-07039e49888b call selects the additional-operation branch exactly when its result is nonzero.

**Additional-operation branch.** F-7a117b899bf6 receives a second eight-byte receiver region. F-cdd33263ee89 is called, and the returned object’s offset-184 operation is invoked. The second receiver’s offset-12 operation then receives the receiver and a **fresh read of the global operand previously used in the first derivation**. This is not reuse of the earlier operand value: intervening dependency calls may have changed that global. F-ec22bd8215d3 subsequently receives the second receiver.

Whenever the first comparison was nonzero, flag byte 0 is written to 1 through the retained pointer after the optional additional-operation branch, whether that branch was selected or skipped.

### Selector 4: main call and completion

After preparation, F-a6d44bb3a0a1 receives context, payload bytes +1 through +3, and the retained classification. These payload bytes are read from the retained payload address after the preparation calls; their initial contents are not guaranteed to survive those calls.

The main command result is retained while F-ec22bd8215d3 receives the first receiver. If this final call returns normally, the retained command result is returned. Its own result is ignored, but its side effects are not.

No later operation is guaranteed if a preceding dependency fails to return normally.

## Side effects

Directly established effects include descriptor and payload reads, global reads, temporary-storage writes, conditional writes of 1 to flag byte 0, external calls, and indirect calls.

No direct descriptor or payload writes are shown. No direct write to flag byte 1 or to the global flag-pointer slot is shown. These are not preservation guarantees: dependencies may modify reachable memory or globals.

No allocation, ownership transfer, or payload release is explicitly established. F-ec22bd8215d3’s position suggests receiver cleanup, but destruction, reference release, deallocation, and ownership semantics remain inferred intent rather than proven behavior.

Selector-4 temporary regions are represented as eight-byte local storage regions. Dependency retention of their addresses is unknown; safe retention beyond their lifetime is not established.

There is no represented rollback or recovery. Flag writes and other dependency effects may precede a later failure. Invalid memory accesses have no shown handling. Concurrency, atomicity, synchronization, and reentrancy guarantees are not established.

## Dependencies

| Opaque ID | Established call-site role |
|---|---|
| F-5f5f14cedda3 | Produces classification before direct descriptor reads. |
| F-a817d8d58c10 | Represented non-returning response to failed entry checks. |
| F-c25a2fcc75b5 | Decoder role for selectors 0, 1, and 6. |
| F-64e2dfdd4a5d | Decoder role for selectors 2, 9, and 15. |
| F-fe1b1452b557 | Main dependency for selectors 0 and 15. |
| F-e25e25f723f0 | Main dependency for selector 1. |
| F-aa3cba10a94f | Main dependency for selector 2. |
| F-9fc356bbe31f | Main dependency for selector 3. |
| F-a6d44bb3a0a1 | Main dependency for selector 4. |
| F-3867c434fcca | Main dependency for selector 6. |
| F-0b77559fa811 | Main dependency for selector 7. |
| F-adf871004f14 | Main dependency for selector 8. |
| F-bb85056f54e1 | Main dependency for selector 9; complete argument list unresolved. |
| F-55290c3145ab | Main dependency for selector 10; complete argument list unresolved. |
| F-adb56aabcc86 | Main dependency for selector 11. |
| F-d9032d50340b | Main dependency for selector 13. |
| F-280d7b6ddabc | Main dependency for selector 14. |
| F-6ae62d4262ef | Main dependency for selector 16; complete argument list unresolved. |
| F-cdd33263ee89 | Supplies an object used for an offset-184 indirect invocation. |
| F-7a117b899bf6 | Receives storage used as a receiver afterward. |
| F-04f46bd4cf22 | Prepares comparison storage from an opaque value. |
| F-927372e9397c | Derives comparison input from receiver output and a global operand. |
| F-07039e49888b | Supplies zero/nonzero branch decisions. |
| F-ec22bd8215d3 | Receives receiver storage after use. |

The indirect targets have no supplied opaque IDs. Their identities, complete calling conventions, and effects are **documented_only**.

## Acceptance examples

These are conditional static criteria, not runtime observations. Unless stated otherwise, relevant memory is accessible and dependencies return normally.

1. **Observed null payload.** If the descriptor’s payload address is null when read after F-5f5f14cedda3 returns, the fatal dependency is invoked. An address that was null only before that call is insufficient to require this outcome.

2. **Observed short record.** If the length read after F-5f5f14cedda3 returns is 0 or 1, the fatal dependency is invoked. An initially short length repaired by that dependency does not independently require failure.

3. **Signed classification boundary.** Classification 3 fails entry validation. Classification with bit pattern 0xFFFFFFFF does not fail the classification check by itself; the other entry conditions still apply.

4. **Exact-length mismatch.** Selector 1 with observed length 6 returns 4 without its decoder or main dependency, after successful entry validation.

5. **Default selectors.** Selectors 5 and 12 return 4 for any observed length passing entry validation, without command-specific calls.

6. **Result propagation.** For selector 7 with length 3, the normal 32-bit result of F-0b77559fa811 is returned unchanged.

7. **Representable arithmetic.** For selector 15 with length 3, a decoded value of 2 and an observed mask of 0xFFFFFFFF produce an argument of 2,000 for F-fe1b1452b557.

8. **Distinct flag objects.** If the entry-retained pointer addresses one flag region and the fresh selector-4 pointer evaluation addresses another, the initial byte-1 test uses the latter. Any direct byte-0 write still targets the former.

9. **Zero first comparison.** A zero first selector-4 comparison skips the second preparation and additional-operation branch. The main command still runs, followed by F-ec22bd8215d3 for the first receiver.

10. **Second comparison suppressed.** A nonzero first comparison followed by a nonzero retained-pointer flag-byte-0 read selects additional work without the second comparison call.

11. **Second comparison zero.** A nonzero first comparison, zero flag byte 0 at its later read, and zero second comparison skip additional work but still cause the subsequent flag-byte-0 write of 1.

12. **Fresh operand for second receiver.** If the additional-operation branch runs and intervening calls change the first derivation’s global operand, the second receiver’s offset-12 operation receives its newly read value, not the earlier derivation value.

All flag and global conditions concern values at their specified reads, not merely entry values.

## Uncertainties

The following remain **documented_only**:

- Decoder byte order, exact read footprints, result ranges, and native return widths.
- Application meanings of selectors, classification, global values, and return value 4.
- Full argument lists associated with scratch stores for selectors 9, 10, and 16.
- Selector-15 overflow behavior outside the representable signed-product range.
- Narrow argument extension and complete calling conventions where not explicitly established.
- Indirect target identities and dependency effects.
- Allocation, ownership, destruction, address retention, and lifetime requirements imposed by dependencies.
- Behavior under inaccessible memory, abnormal dependency completion, or concurrent mutation.

No dependency is assumed to preserve globals, descriptor fields, payload contents, or receiver storage merely because preservation would simplify the description.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping. No implementation access, dependency-body inspection, or runtime testing has been performed.

Established static evidence includes descriptor offsets and widths, entry-read ordering, the signed classification check, exact-length dispatch, visible scratch stores, arithmetic, distinct retained and fresh flag-pointer accesses, repeated global-operand evaluation, conditional flag writes, and command-result propagation.

“Decoder,” “comparison,” “receiver initialization,” and “cleanup” describe call-site roles or inferred intent. They do not establish domain semantics, ownership rules, or hidden dependency behavior.
