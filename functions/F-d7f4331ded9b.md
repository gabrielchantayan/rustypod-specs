<!-- Trusted provenance: F-d7f4331ded9b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d7f4331ded9b

## Purpose

F-d7f4331ded9b processes an indexed request through preparation, payload classification, conditional acceptance, completion reporting, and subsequent state transitions.

This is not a pure validator. Unsupported lengths and rejected requests still cause dependency calls, notifications, descriptor handoff, and state changes. An accepted request can report completion result 0 and subsequently return 7.

“Channel,” “selector,” “mask,” “mode,” and “peer” describe observable roles. Their application meanings are inferred.

## Interface

### Inputs and descriptor

The inputs are a 32-bit opaque context value and a descriptor pointer.

| Descriptor byte offset | Width | Observed use |
|---|---:|---|
| 0 | 32 bits | Captured early for completion reporting |
| 8 | 32 bits | Payload address, read only for length 12 |
| 12 | 16 bits | Compared with 12 |

F-5f5f14cedda3 receives the descriptor and supplies the channel index before these fields are consumed. No local pointer, index, or capacity validation is visible.

For length 12, F-c25a2fcc75b5 reads payload locations in this order:

1. Offset 8: initial selector, narrowed to its low 16 bits.
2. Offset 0: 32-bit request mask.
3. Offset 4: 32-bit flags.

Flags supply mode from bits 0–1, option from bits 2–3, and peer from bits 4–7. Option remains encoded as 0, 4, 8, or 12.

Acceptance rejects reserved bits 8–31 in the **working flags**. The qualified-peer, zero-request-mask path replaces working flags with zero first; reserved bits in the original payload therefore do not invariably cause rejection. Mode, option, and peer are decoded before this replacement and are not recomputed.

Payload-reader byte order, physical access width, and alignment requirements are **documented_only**.

### Results and storage obligations

| Return | Condition on normal continuation |
|---:|---|
| 4 | Length differs from 12 |
| 2 | Length 12 fails acceptance |
| 0 | Acceptance succeeds without the late override |
| 7 | Acceptance succeeds in mode 2 and F-b816af9a7d2b returns nonzero |

Explicit non-returning paths also exist. Invalid storage has no visible local recovery.

The descriptor is passed to F-09b5d9827724, then F-256e8a77aa59 with argument 0. No subsequent direct descriptor read is visible. Release, retention, and ownership transfer are not established.

Dependencies also receive temporary-word addresses and a temporary 20-byte record:

| Record offset | Width | Meaning |
|---|---:|---|
| 0 | 8 bits | Reason |
| 1–3 | 3 bytes | Not visibly initialized |
| 4 | 32 bits | Status |
| 8 | 32 bits | Channel |
| 12 | 32 bits | Saved working selector |
| 16 | 32 bits | Saved working mask |

These temporaries have invocation-local storage. Safe retention by dependencies is not established.

## Behavior

### Initial preparation

The function derives the channel, calls F-a2715f79cdcc, captures the current default global, and captures descriptor word 0.

Separate context acquisitions then precede the saved-mask query F-62d1a359629f, saved-state query F-c1e400b68436, and association query F-40c28640bfd1, in that order.

It increments the channel’s 32-bit attempt-counter storage without an explicit overflow check. It initializes the temporary record with reason 0, status 16, channel, selector 0, and mask 0.

F-704f99fd5e22 is called for operations 0x36, 0x37, 0x38, 0x14, and 0x17, in order. Each receives a newly written temporary copy of the **early captured default**. Operation 0x36 targets index 1; the others target the selected channel.

A fresh acquisition precedes F-ebd9fed2fee8 for the channel.

If the captured association is nonzero, the function separately:

- Queries that associated index’s mask through F-62d1a359629f.
- Calls F-ebd9fed2fee8 and F-334068ea5e0c for it.
- Writes byte 0xff at offset 4 of its 36-byte entry.
- Calls F-2250ce450b3d with 0xffffffff.

Each applicable operation has its own acquisition. If the associated mask was nonzero, operation 0x3b is emitted through F-704f99fd5e22 for the selected channel, with value 0xffffffff, followed by F-da3be2b0546d for the associated index and captured mask.

F-15643c36365d supplies an object for a virtual operation at offset 0x48. A nonzero result causes another F-15643c36365d call and a virtual operation at offset 0x44 with argument 0. The objects need not be identical.

### Length and restriction-read timing

A length other than 12 skips payload parsing and enters the nonaccepted common path.

For length 12, the function:

1. Reads the payload address.
2. Reads and narrows the selector.
3. Unconditionally snapshots the channel-1 restriction global.
4. Reads the request mask.
5. Reads flags and decodes mode, option, and peer.
6. For a channel other than 1, replaces the snapshot with a read of the other restriction global.

Thus channel 1 retains a restriction snapshot taken before the mask and flags readers. Other channels use a replacement read after both readers; the initial channel-1 read still occurs.

An additional acceptance restriction later rereads the channel-1 global independently. Dependencies are not assumed to preserve either global.

### Peer classification and working-input replacement

Peer 0 is initially permitted, except that channel 4 requires selector in inclusive range 0x200–0x2ff.

A nonzero peer is disqualifying if greater than 2 or equal to the selected channel. A different peer of 1 or 2 is directly permitted for mode 2 with selector in that range. Otherwise, qualification requires selector 0 and mode 0.

On this special path, separate acquisitions precede F-d5cded701dac and F-40c28640bfd1. Qualification succeeds when the peer association equals the selected channel and the queried peer state equals 6. Another acquisition then precedes F-8f3cd6356d31 to capture the peer mask.

For a zero working request mask:

- Acquire context for F-b37968c9f644.
- Pass its result and five zeros to F-cd7e1905512f.
- A zero result invokes non-returning F-a817d8d58c10.
- Otherwise pass the returned object to F-256e8a77aa59 with argument 1.
- Freshly acquire context for F-ebd9fed2fee8 on the peer, then call F-2c4fc5b4a966 with action 0.
- Replace the working mask with 1 and working flags with 0.

The decoded mode remains 0. The decoded option need not be zero and remains unchanged.

For a nonzero working request mask, replace the restriction snapshot with the current configuration capability mask at offset 0x28, excluding captured peer-mask bits. Freshly acquire context for F-1bc5bcdc9cca, replace the selector with its result, and mark a transfer. No additional 16-bit narrowing of this replacement selector is visible.

If special qualification fails, a peer association of 0 or the selected channel disables a later fallback notification. It does not remove the peer disqualification.

### Length-12 preparation

F-352a899fb784 receives the input context, the saved F-a2715f79cdcc result, and 1. Separate acquisitions precede F-39405f5827ef and F-4944ce526800; the latter receives channel followed by two zeros.

Before synchronization entry, an exit address is computed using the channel and a separate table with 28-byte entries. After entry, the first 32-bit word of the channel’s 36-byte state entry is read. Nonzero invokes F-ae524095c0b0 followed by non-returning F-feee9156a257. Otherwise, synchronization exit receives the previously computed address. No acquisition-success check is visible.

The channel’s four-byte tracking record is then handled:

- Byte 0 equal to 1: increment byte 3, storing an 8-bit result.
- Otherwise, selector 0x83 or 0x86: write bytes 0–3 as 1, 0, 0, 0, in order.
- Otherwise: leave the record unchanged.

The working selector and mask are saved in the temporary completion record before later reductions. A fresh acquisition precedes F-67b16ca5ec28 with value 0x13.

F-87c09191bebe evaluates the mask. Separately acquired context precedes F-b6b18c9159c8, which evaluates the selector and returns a capability mask. A nonzero capability result establishes availability as 1; otherwise it remains 0.

### Acceptance conditions

Accumulated disqualifications are:

- Peer-classification failure.
- Selector 0x46 or 0x1da.
- Mode 3.
- Any reserved bit 8–31 in the **current working flags**.
- Clear bit 0 in the working mask.
- For mode 0, intersection with the working restriction snapshot.
- For nonzero modes, selector 0.

Mode 0 calls F-ca1c4e2f14bb with argument 1 before testing its restriction intersection.

Unless mode is 2, a further group applies when the captured peer mask is zero. It rejects an intersection with a fresh channel-1 restriction read, selector in 0x200–0x2ff, or option 8 or 12.

Acceptance additionally requires:

- Nonzero bitwise conjunction of the F-87c09191bebe result, complement of the accumulated disqualification value, and availability.
- Every working requested bit supported by the intersection of the current configuration capability mask and the F-b6b18c9159c8 result.

Availability is only 0 or 1. Consequently, the evaluation gate requires bit 0 of the evaluation result, not merely a nonzero result.

### Accepted state initialization

Acceptance selects result 0. If the saved state equals 0x13, requested bits are removed from the saved channel mask, except that bit 0x10 is preserved. F-da3be2b0546d receives the resulting saved mask regardless of that condition.

Mode 0 with peer 0 calls F-646d64bbe501.

After a fresh acquisition, the channel’s 36-byte entry address is captured. F-9eed47a4e560 receives the selector and addresses of entry bytes 0x1c and 0x1d.

**Immediately after that call returns**, the function unconditionally reads the global base value used for the fivefold calculation. This read precedes both checking the dependency result and examining byte 0x1c. A nonzero result then invokes non-returning F-a817d8d58c10.

Normal continuation initializes the captured entry:

| Byte 0x1c | Word 0x10 | Word 0x14 | Byte 0x1b | Halfword 0x18 |
|---:|---|---|---:|---:|
| 1 | 1000 | A separate global value | 4 | 0x46 |
| 2 | Fresh read of the base-value global | Five times the earlier unconditional snapshot | 2 | 0xff |
| Other | 0 | 0 | 0 | 0 |

Writes occur in column order. For byte 0x1c equal to 2, the two base-value reads are distinct and need not agree. The multiplication is stored in 32 bits without an explicit overflow check.

The captured entry address remains the target even if F-9eed47a4e560 changes the global entry-table base.

Separate acquisitions precede selector storage through F-da2fe84eecde, working-flags storage through F-56ee0cebf586, and F-4944ce526800. The last call’s final argument is 1 exactly when the previously decoded option equals 12.

The mask is reduced by the current global eligibility mask. Separate acquisitions precede F-1cd96d01ddae and F-9ef7726bf517 with that reduced mask.

A global object-slot address is captured. If its contents are nonzero, the object’s virtual operation at offset 0x1c runs, then zero is written through the captured slot address. Ownership consequences are unknown.

For each set bit 0 through 12 of the reduced mask, in ascending order, a fresh acquisition precedes F-9266831caef7. Its returned object receives a virtual operation at offset 0x0c with arguments 4 and channel. Higher bits receive no such callback.

Channel 1 calls F-2708b7c79728 with 1 and a temporary word containing 0x10, then invokes D-04 with 0x23.

Transfers acquire context for F-dfe6e6b7ec7c with peer and channel. A zero result causes separately acquired operations to reread the peer selector and store it for the selected channel.

### Rejection and unsupported length

Rejected length-12 requests select result 2, freshly acquire context for F-ebd9fed2fee8 on the channel, and restore the saved mask through F-da3be2b0546d.

Reason selection uses the current working mask:

| Priority condition | Reason | Subsequent mask |
|---|---:|---|
| Intersects configuration exclusion mask at offset 0x2c | 1 | Remove excluded bits |
| Otherwise, evaluation result is zero | 5 | Unchanged |
| Otherwise, no eligible bits | 4 | 0 |
| Eligible bits exist, but none are currently capable | 3 | 0 |
| Eligible and currently capable bits exist | 2 | Their intersection |

The exclusion mask is read for the test and again for removal. The saved completion-record mask is not rewritten by these reductions.

Both rejected and unsupported-length paths call F-ca1c4e2f14bb with 1, then F-646d64bbe501 with an additional visible argument 1. Their working selector becomes the early captured default.

### Shared completion and progress

All normal paths call F-09b5d9827724 with descriptor, captured descriptor word, two zeros, 2, current result, and 0x13. F-256e8a77aa59 then receives descriptor and 0.

The configuration base is captured. Synchronization entry uses the value at configuration offset 0x5c. The low eight bits of its 32-bit word at offset 0x34 are interpreted as signed 8-bit status. F-88707f7d96dd runs, then that captured status is copied into the temporary record.

Status 100 writes 1 at captured configuration base plus 0x20 plus channel, then calls F-43ce968102ca with 0xffff, operation 0x38, mask 0, and delay 2000.

For length 12 only:

- Status 100 copies reason, status, channel, saved selector, and saved mask, in that order, into a channel-indexed 20-byte record.
- Other statuses call F-c6c589453fa8 with two zeros unless the peer path disabled this fallback.

Accepted mode 2 with nonzero peer freshly acquires context for F-5fda5988215d.

For accepted requests whose progress byte is below 4, the progress-table base is captured before any F-4b57201d73f0 call. Candidate progress is:

- 2 when the reduced mask has no bits outside bits 0 and 2, and selector is zero.
- Otherwise 4 if F-4b57201d73f0 returns zero, or 3 if nonzero.

After candidate selection, including the optional call, the function reads the **current default global**. This first progress-time snapshot occurs before testing whether an update is allowed.

Through the captured progress base, it rereads the current byte. Update occurs if current progress is no greater than the candidate, or candidate is 2.

Inside the update block, it reads the default global **again** into a temporary notification value, then sets a separate flag for captured status 100 and writes candidate progress.

Candidate 2 calls F-43ce968102ca with the **first progress-time default snapshot**, operation 0x37, and the reduced mask. Delay is 800, or 1 when the current unsigned attempt count exceeds 3. The second default read still occurs, although this notification does not use it.

Candidates 3 and 4 call F-704f99fd5e22 with addresses of the second default snapshot, operation 0x37, and the working restriction snapshot. For candidate 4, after that callback:

1. Reread progress through the captured base.
2. Set another flag if it remains 4.
3. Set temporary reason 14 and status 100.
4. Call F-c6c589453fa8 with two zeros, record address, and size 20.

Neither progress-time default snapshot is necessarily the early preparation default.

On every normal path, synchronization exit rereads offset 0x5c through the captured configuration base; it does not reuse the entry argument.

After exit:

- Accepted mode 2 calls F-b816af9a7d2b; nonzero changes the final return to 7.
- Other accepted nontransfers call F-2c4fc5b4a966 with 2 for mode 1, otherwise 0xffffffff.
- Accepted transfers freshly locate the current channel entry, clear bytes 0x1c and 0x1d, and invoke actions 0, 3, and 4 in order. **After those actions**, they read the current default global and emit operation 0x3b through F-43ce968102ca with mask 0 and delay 500.
- Accepted modes other than 2 call F-d2eae028ceec.
- Nonaccepted paths force selector zero if availability was never established, freshly acquire context to store the selector, then call F-a6e52da0a17c with channel and 0.

## Side effects

Observable effects include counters, tracking records, channel entries, completion records, configuration bytes, progress bytes, flags, object-slot clearing, virtual operations, and dependency calls.

No rollback is visible. Non-returning failures can follow earlier notifications, resets, or peer changes.

Captured values and addresses are distinct from later reads. Important examples are restriction snapshots, the first synchronization exit address, accepted-entry address, progress-table base, shared configuration base, and the multiple default-value snapshots.

## Dependencies

D-01, D-02, D-03, and D-04 are opaque local dependency IDs for context acquisition, synchronization-related entry, synchronization-related exit, and the external operation, respectively. These descriptions do not establish semantics or ABI.

Every occurrence of the following operations has its own immediately preceding D-01 acquisition:

| Acquired argument form | Operations |
|---|---|
| Acquired value directly | F-40c28640bfd1, F-ebd9fed2fee8, F-2250ce450b3d, F-d5cded701dac, F-8f3cd6356d31, F-b6b18c9159c8, F-9eed47a4e560, F-dfe6e6b7ec7c, F-5fda5988215d |
| Acquired address plus 4 | F-62d1a359629f, F-c1e400b68436, F-334068ea5e0c, F-b37968c9f644, F-1bc5bcdc9cca, F-39405f5827ef, F-4944ce526800, F-67b16ca5ec28, F-da2fe84eecde, F-56ee0cebf586, F-1cd96d01ddae, F-9ef7726bf517, F-9266831caef7 |

No acquisition result is established as reusable across operations. Dependency internals, global preservation, ownership, and synchronization guarantees remain **documented_only**.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Unsupported length:** Valid storage and normally returning dependencies yield return 4 without payload-reader calls, but still with preparation and shared completion.

2. **Original reserved flags replaced:** A qualified peer, selector 0, mode 0, and original mask 0 can replace working mask with 1 and working flags with 0. Original reserved bits then do not reject the request. Acceptance still requires every other gate; an originally decoded option 8 or 12 remains relevant.

3. **Restriction timing:** If the mask reader changes the channel-1 restriction global, channel 1 retains its earlier working snapshot. The additional restriction test uses a fresh read. Other channels obtain their replacement snapshot after the flags reader.

4. **Evaluation width:** Evaluation result 2 cannot pass the bit-0 acceptance gate, despite being nonzero.

5. **Entry snapshots:** If F-9eed47a4e560 replaces the entry-table base, initialization still targets the captured entry. The base-value global is read even if that dependency returns nonzero. For entry byte 2, offset 0x14 uses five times that snapshot; offset 0x10 uses a separate later read.

6. **Progress defaults:** A default-global change during F-4b57201d73f0 affects the subsequent first progress snapshot. If update is refused, that first read still occurs but the second does not. Candidate 2 uses the first snapshot and reduced mask; candidates 3 and 4 use the second snapshot.

7. **Captured addresses:** A progress-table replacement during F-4b57201d73f0 does not redirect updates or candidate-4 verification. A change to offset 0x5c of the captured configuration changes the synchronization exit argument.

8. **Transfer notification:** A default-global change during transfer action 4 is reflected in the following operation-0x3b notification.

9. **Late override and progress boundary:** Accepted mode 2 reports completion result 0 before a possible final return 7. Existing progress 3 may become candidate 2; existing progress 4 skips candidate selection and its default reads entirely.

## Uncertainties

- **documented_only:** Business meanings, valid indices, capacities, pointer validity, concurrency, and dependency ownership.
- **documented_only:** Whether successive D-01 acquisitions identify the same object.
- **documented_only:** Definitive ABIs for D-02, F-646d64bbe501, and F-c6c589453fa8, whose visible argument counts differ.
- **documented_only:** The peer-association result’s represented upper word; it has no established independent contractual meaning.
- **documented_only:** Exact overflow semantics for the signed-looking counter increment and fivefold multiplication. Storage widths and absence of explicit checks are established.
- **documented_only:** Endianness, exceptional dependency behavior, and virtual-operation identities.
- **Inferred intent:** D-02 and D-03 delimit protected regions. Their placement does not prove mutual exclusion.

## Evidence

Established static evidence consists of the supplied representation’s field widths, offsets, constants, branches, dependency ordering, direct reads and writes, captured-address lifetimes, and result selection.

The representation includes a type-propagation warning. Ambiguous interfaces and arithmetic semantics remain unresolved rather than normalized.

No implementation access, runtime execution, or tests have been performed.
