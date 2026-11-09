<!-- Trusted provenance: F-b21fbef03aec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b21fbef03aec

## Purpose

F-b21fbef03aec conditionally obtains records through an indirect input interface and places their fields into storage obtained from a dependency. It supports two input layouts and uses a common 24-byte output-record stride.

The 64-bit result combines a completion indicator in the low 32 bits with the contents of a working location in the high 32 bits. It is not an ordinary Boolean. The working location initially contains the input-object address, is cleared when processing becomes eligible, and subsequently serves as the destination for compact-format second-field reads.

Positioning, reading, and allocation are inferred dependency roles. Their domain semantics, ownership rules, and effects beyond the visible interactions are not established.

## Interface

The function accepts seven arguments. Addresses and ordinary scalar arguments in the supplied representation are 32 bits.

| Argument | Established use |
|---|---|
| A1 | Unused by the visible body. |
| A2 | Address of an input-interface object. Its first 32-bit word supplies a dispatch-table address. |
| A3 | Unsigned low word of a two-word positioning value. |
| A4 | Unsigned high word of that value; its sign bit also controls eligibility. |
| A5 | Unused by the visible body. |
| A6 | Unused by the visible body. |
| A7 | Nullable address of a descriptor containing at least two consecutive 32-bit words. |

The descriptor layout is:

| Byte offset | Use |
|---|---|
| 0–3 | Record count, interpreted as signed 32-bit when tested. |
| 4–7 | Storage address overwritten with the result of F-fab3799b3190 after an allocation attempt. |

A null descriptor disables processing. It does not request processing without output.

For an eligible invocation, the input object, applicable dispatch entries, and descriptor must support the accesses described below. Record processing additionally requires sufficient writable storage for every destination actually used. There is no visible address, capacity, or alignment validation.

### Result representation

| Result bits | Meaning |
|---|---|
| 31–0 | Zero for completion; one for failure. |
| 63–32 | Current 32-bit contents of the return-working location when the result is constructed. |

The working location begins with A2. It is cleared before the first dependency call on an eligible path.

Consequently:

- Eligibility rejection returns A2 in the high half and one in the low half.
- After eligibility, the high half remains zero unless the working location changes.
- Compact second-field reads can change the high half on both success and failure.
- With preserved temporaries and at least one completed compact record, successful processing returns the last second-field value in the high half.

The function returns neither a completed-record count nor a consumed-byte count.

## Behavior

### Eligibility and initial state

Processing is eligible only when both conditions hold:

1. A4, interpreted as signed 32-bit, is nonnegative.
2. A7 is nonnull.

Thus the accepted A4 range is zero through 2,147,483,647.

Rejection makes no visible dependency calls or descriptor writes. The return has low half one and high half A2.

After acceptance, the function clears the return-working location and initializes two other 32-bit temporaries from A3 and A4. The first later receives compact first fields. The second receives the header and remains live until format selection.

### Indirect dependency resolution

Every indirect invocation resolves its callable through the input object’s current dispatch table:

- Byte offset 0x14 identifies the role F-dispatch-14.
- Byte offset 0x10 identifies the role F-dispatch-10.

The object’s dispatch-table word and the applicable table entry are obtained again for subsequent calls. An implementation reproducing this behavior cannot assume that an earlier table or callable remains applicable after a dependency returns.

The initial call also obtains the entry at offset 0x14 as an explicit argument. These accesses do not establish an atomic lookup or any synchronization protocol.

### Initial positioning operation

The first dependency invocation is F-dispatch-14, with these arguments:

1. A2.
2. The entry address obtained from dispatch offset 0x14.
3. The low 32 bits of A3 plus eight.
4. A4 plus the carry from that addition, using 32-bit arithmetic.

The carry is one exactly when A3 exceeds 0xfffffff7. The last two arguments therefore represent addition of eight to the supplied two-word value.

The second argument is an entry address. Its meaning as an argument is unresolved; it is not established as a positioning-mode constant.

A zero return permits continuation. Any other return causes failure without a header read. Effects already produced by the dependency are not undone.

### Header and count acquisition

After successful positioning, a fresh F-dispatch-10 lookup requests four bytes into the header temporary, with A2 as the object argument and one as the final scalar argument.

Only a return equal to four permits continuation. Otherwise the function fails without requesting the count.

After an exact header return, the function masks the current header temporary with 0xff000000. This masking occurs once.

A fresh lookup then requests four bytes directly into the descriptor’s count word, again with final argument one. A return unequal to four causes failure. The count word may already have changed; the storage-address word has not yet been assigned by this function.

The header temporary remains live after its address has been exposed. A dependency with access to a retained pointer or other relevant shared state may change it after masking. Such changes are not masked again.

Following an exact count return, the current count is tested as signed 32-bit:

- Zero or a negative value completes processing without allocation or record reads.
- A positive value proceeds to allocation.

On the nonpositive path, this function does not assign the descriptor’s storage-address word.

### Allocation and format selection

For a positive count, F-fab3799b3190 receives a 32-bit count-times-24 value. No overflow check is visible.

Its 32-bit result is immediately stored into the descriptor’s storage-address word:

- Zero causes failure.
- Nonzero permits continuation.

An allocation failure therefore overwrites any prior storage address with zero. No prior address is saved for restoration or explicit release.

After successful allocation, the current **full 32-bit header temporary** is compared with 0x01000000:

- Exact equality selects wide input.
- Every other value selects compact input.

If the masked temporary is preserved, selection depends only on whether the originally read header’s high byte was 0x01. If a dependency changes the temporary after masking, its later full-word value controls selection. A newly introduced low-order bit can therefore change the selected format.

Selection occurs once. Subsequent header changes do not switch loops.

### Output layout and arithmetic

Each record has a 24-byte stride:

| Relative bytes | Wide input | Compact input, with preserved temporaries |
|---|---|---|
| 0–7 | Direct eight-byte read | First 32-bit field zero-extended to 64 bits |
| 8–15 | Direct eight-byte read | Second 32-bit field sign-extended to 64 bits |
| 16–19 | Direct four-byte read | Direct four-byte read |
| 20–23 | No explicit write | No explicit write |

The function does not initialize the trailing four bytes.

Allocation-size multiplication, record-offset multiplication, and destination-address additions use the visible 32-bit representation. There are no visible overflow or bounds checks. For example, count 178,956,971 has mathematical byte size 4,294,967,304, whose low 32 bits are eight. A nonzero allocation result for that request does not establish sufficient capacity for the record accesses.

These width observations do not establish the original language’s rules for overflowing signed expressions.

### Wide-input processing

The signed record index begins at zero. Before each iteration, the function compares it with the descriptor’s current signed count.

For each admitted record, three fresh indirect lookups request, in order:

1. Eight bytes into record-relative bytes 0–7.
2. Eight bytes into record-relative bytes 8–15.
3. Four bytes into record-relative bytes 16–19.

Every return must equal its requested size. The first mismatch causes immediate failure and skips the remaining reads for that record.

The descriptor’s storage-address word is read again when forming each destination. A dependency can therefore redirect later fields of the same record to another base address.

A completed wide record involves 20 requested input bytes. The body performs no explicit conversion of the two eight-byte fields.

### Compact-input processing

The signed index and current-count comparison operate as on the wide path.

For each admitted record, fresh indirect lookups request:

1. Four bytes into the first-field temporary.
2. Four bytes into the return-working location.
3. Four bytes directly into record-relative bytes 16–19.

Every return must equal four. A mismatch causes immediate failure.

Only after all three reads return exactly four does the function explicitly store, in order:

1. The current first-field temporary into bytes 0–3.
2. Zero into bytes 4–7.
3. The current return-working value into bytes 8–11.
4. An arithmetic sign-extension word derived from the return-working location into bytes 12–15.

The base address is obtained after the reads for the first pair of stores and obtained again for the second pair. Aliasing that lets earlier stores change the descriptor can therefore redirect the second pair.

The exposed temporaries are not immutable snapshots. The second read can change the first temporary through a retained pointer. The third read can change either temporary before the explicit stores. Later iterations can also change the return-working contents before return construction.

Under non-overlapping storage and preserved-temporary assumptions, the first field is zero-extended and the second field is sign-extended. Without those assumptions, the actual values and destinations follow the ordered accesses; the body does not establish an immutable field snapshot across potentially aliasing stores.

If the third read fails, bytes 16–19 may already have changed, but the explicit stores to bytes 0–15 do not occur for that record. Earlier read failures can likewise leave changed temporary contents.

A completed compact record involves 12 requested input bytes.

### Mutable descriptor, completion, and early failure

The record count is not snapshotted for loop termination. Every loop condition reads its current signed value. The storage address is likewise reread during processing rather than fixed to the allocation result.

Dependencies may increase, decrease, or negate the count, or redirect later output destinations. Successful allocation does not guarantee that any record is attempted: the allocation dependency could leave the count nonpositive before the first loop condition.

For valid non-overlapping storage, with the initially read positive count preserved throughout processing, **successful allocation and exact-size returns from every required record read** result in exactly that initial count of completed records.

Preserving descriptor fields alone is insufficient. Any record-read mismatch terminates processing with failure, potentially before the first record completes. Completion of an individual record also does not imply completion of the collection: a subsequent record can fail.

After the selected loop terminates normally, the function returns low half zero. There is no independent final check that the number of completed records equals the count originally read.

## Side effects

Visible effects include:

- The initial positioning operation.
- Repeated dispatch-table and entry accesses.
- Exposure of temporary addresses to dependencies.
- Direct writes into the descriptor’s count word.
- Replacement of the descriptor’s storage-address word after allocation.
- Partial or complete record writes.
- Exposure of the return-working contents through the result’s high half.

Failure is not transactional. The function does not visibly restore input position, descriptor contents, temporary values, or partial output.

There is no explicit release on success or failure. This does not establish ownership transfer or prove that dependencies do not allocate, retain, release, or otherwise manage memory internally.

Dependency effects are not limited by the visible body. Preservation of globals, dispatch entries, descriptor fields, exposed temporaries, or unrelated memory requires an external contract.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-fab3799b3190 | Receives a 32-bit count-times-24 value. Its result is stored as a 32-bit address; zero causes failure. | Allocation mechanism, capacity guarantees, alignment, initialization, ownership, release, and other effects. |
| F-dispatch-14 | Resolved through offset 0x14. Receives the object, an entry address, and adjusted low and high positioning words. Zero permits continuation. | Actual callable identity, positioning semantics, entry-address argument meaning, and other effects. |
| F-dispatch-10 | Resolved through offset 0x10 for each invocation. Receives object, destination, requested byte count, and one. Its return must equal the requested size. | Actual callable identity, byte order, partial-transfer behavior, cursor effects, pointer retention, and other effects. |

The dispatch IDs identify indirect roles, not verified fixed callees. Different invocations can resolve to different functions.

## Acceptance examples

These are conditional static expectations, not executed tests.

Unless an example expressly permits mutation, assume valid non-overlapping objects, sufficient storage, stable dispatch state, a preserved count after its designated read, and dependencies that modify only the current designated destination. Assume previously exposed temporaries remain unchanged except when subsequently used as designated destinations.

1. **Eligibility rejection.** A4 is 0x80000000. No dependency is invoked and the descriptor is unchanged. The return has high half A2 and low half one. A null descriptor independently produces the same rejection behavior.

2. **Position carry.** A3 is 0xfffffffc and A4 is two. The positioning arguments contain low word four and high word three.

3. **Position failure.** F-dispatch-14 returns nonzero. No header read follows. With the cleared working location preserved, the return is 0x0000000000000001.

4. **Nonpositive count.** Header and count reads each return four, and the count is zero or 0xffffffff. No allocation or record read occurs. The prior storage address remains unchanged and the return is zero.

5. **Format selection.** With positive count and successful allocation, preserved header input 0x01abcdef selects wide input. Header input 0x02000000 selects compact input, not an unsupported-format error.

6. **Retained header mutation.** Header input 0x01abcdef is masked to 0x01000000. During the count read, a dependency changes that retained temporary to 0x01000001. Successful allocation followed by no further mutation selects compact input. Replacing it with exactly 0x01000000 instead selects wide input regardless of the original header.

7. **Dispatch replacement.** The positioning dependency replaces the object’s table pointer. The header read uses offset 0x10 in the replacement table. If that read changes the entry, the count read resolves the newly installed entry.

8. **One completed wide record.** Count one, successful allocation, and record-read returns eight, eight, and four produce direct writes to bytes 0–19. Bytes 20–23 receive no explicit write. The return is zero.

9. **Stable count does not guarantee completion.** Count two remains unchanged and allocation succeeds. The first wide-field read returns zero. The function fails before any record completes, performs no second-field or third-field read, and does not attempt the second record. With the working location preserved, the return is one.

10. **Exact-count completion.** Count two remains unchanged, allocation succeeds, and all six required wide record reads return their requested sizes. Exactly two records complete and the return is zero. On the compact path, the corresponding condition is six returns of four.

11. **Compact extension.** First field 0xffffffff and second field 0xfffffffe, followed by a successful third read, produce 0x00000000ffffffff and 0xfffffffffffffffe in the first two output quantities. For one record, the return is 0xfffffffe00000000.

12. **Compact third-read failure.** The second field supplies 0x80000000 and the third read returns two. The return is 0x8000000000000001. The third destination may have changed, but no explicit compact stores to bytes 0–15 occur for that record.

13. **Compact retained-temporary mutation.** Designated reads supply first value five and second value six. During a successful third read, the dependency changes the retained first temporary to nine and the return-working location to 0xffffffff. With no later mutation, output contains 0x0000000000000009 and 0xffffffffffffffff, and the return is 0xffffffff00000000.

14. **Partial second-field operation.** The compact second-field operation changes part of its destination but returns less than four. Failure is certain. The high return half reflects the resulting working contents, not necessarily a complete input field.

15. **Allocation failure.** A positive count causes an allocation attempt returning zero. The storage-address word becomes zero and no record read follows. With the working location preserved, the return is one.

## Uncertainties

The following remain **documented_only**:

- Input byte order and domain meanings of the header and record fields.
- Absolute versus relative positioning semantics.
- Meaning of the positioning operation’s entry-address argument.
- Dependency pointer-retention and memory-preservation guarantees.
- Allocation capacity, alignment, initialization, ownership, and release requirements.
- Partial-read write extent, byte consumption, and cursor advancement.
- Behavior for invalid addresses, insufficient storage, hostile aliasing, or concurrent mutation.
- Original-language consequences of overflowing signed expressions beyond the visible 32-bit representation.
- Any caller-intended interpretation of the return high half.

These unknowns do not remove the established exact-size checks: a mismatching record-read return causes failure even when all descriptor fields remain stable.

## Evidence

Established static evidence includes argument widths, eligibility checks, repeated dispatch resolution, dependency-call order, destination exposure, one-time header masking, later full-word format comparison, exact return-size checks, descriptor accesses, record stride, compact-store order, extension operations, loop conditions, and result packing.

The loop conditions establish rereading of the current count. The record-read checks independently establish early failure. Exactly the initial positive count completes only under the stated successful-allocation, exact-read, valid-storage, and count-preservation conditions.

Positioning, reading, allocation, and field meanings are inferred roles rather than independently verified dependency contracts.

This specification uses only the supplied static representation and opaque dependency mapping. No implementation inspection beyond that representation, dependency inspection, runtime execution, or tests have been performed.
