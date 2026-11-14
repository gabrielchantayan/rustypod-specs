<!-- Trusted provenance: F-fbc6bccb5b94; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fbc6bccb5b94

## Purpose

F-fbc6bccb5b94 processes two ordered collections associated with a mutable target. It assigns metadata markers, examines existing entries, applies optional cumulative limits, issues addition or removal-related requests, and finalizes reconciliation.

“Candidate,” “existing entry,” “addition,” “removal,” and “reconciliation” are inferred roles. Established behavior consists of memory accesses, fixed-width arithmetic, branch decisions, and dependency calls. A returning dependency call does not establish that a requested membership change succeeded.

The 32-bit return value indicates whether an addition-request path or removal-triggered rebuild path completed. It is neither a change count nor a general success status. Returning zero can coexist with metadata changes, dependency calls, and completed finalization.

## Interface

### Argument and result

The sole argument is a 32-bit target address.

The returned 32-bit integer is:

- **0:** neither an addition request nor a removal-triggered rebuild set the change indicator. Both collection-acquisition early exits return zero.
- **1:** an addition request returned, or a removal-related event caused the rebuild dependency to return.

There is no distinct error result. Invalid-memory access and abnormal dependency termination have no established recovery behavior.

### Observed layouts

Offsets are bytes. Addresses and word fields are 32 bits; byte fields are eight bits.

| Target offset | Observed use |
|---|---|
| +0x08 | Auxiliary-object pointer; mode 2 accesses auxiliary bytes +0x8e and +0x8f. |
| +0x0c | Parent/context pointer. |
| +0x48 | Embedded address passed to changed-path notification. |
| +0x1ac | Status byte; completed processing clears bit 0x02. |
| +0x1b1 | Configuration-enable byte, reread during later processing. |
| +0x1b2 | Controls predicate-argument retrieval and required-field configuration. |
| +0x1b3 | Enables exclusion by metadata flag 0x04. |
| +0x1b8 | Predicate argument word. |
| +0x1bc | Limit quantity word. |
| +0x1c0 | Limit selector byte. |
| +0x1c4 | Selection-mode word. |
| +0x1c8 | Direction adjustment and required-field control byte. |

The parent/context contains a selection-object pointer at +0x3c. The selection object contains nullable pointers at +0x414 and +0x418; the first pointed-to byte determines temporary clearing.

A collection contains an unsigned 32-bit count at +0x0c and four-byte entry addresses beginning at +0x10. Null entries are skipped.

Each nonnull entry contains a metadata pointer at +0x20.

| Metadata offset | Observed use |
|---|---|
| +0x14 | Exclusion flag byte. |
| +0x20 | Writable reconciliation marker word. |
| +0x24 | Membership-record chain head. |
| +0x5c | Word passed to the accounting dependency. |
| +0x70 | Unsigned 32-bit quantity contribution. |

Membership records contain an owner address at +0x00 and a successor address at +0x28. Mode-specific eligibility fields are listed below.

### Preconditions and ownership

All accessed addresses must remain usable for their respective reads and writes. Collections must support the indices actually visited. Traversed record chains must terminate for the function to terminate.

No bounds checks, cycle detection, address validation, or arithmetic-overflow rejection are visible. Collection ownership, alias safety, and entry or record lifetime are not established. Calls described as cleanup are not proven to deallocate anything.

Temporary-storage addresses are passed to dependencies. The caller does not establish that those dependencies initialize the storage, write the subsequently consumed bytes, or restrict writes to the apparent local objects.

## Behavior

### Entry state

A global word is captured before an entry helper is called. Every returning path passes this saved word to an exit helper, rather than rereading the global.

After the entry helper returns, the target’s parent/context pointer is captured. F-04cf29f6f1ed supplies a separate value retained for unchanged-path finalization.

The change indicator, removal-event indicator, threshold-crossed latch, and accounting totals begin at zero.

A different global word is read later for changed-path notification. Dependencies may alter either global; neither meaning nor preservation is established.

### Initial configuration

If the initial read of target +0x1b1 is zero:

- Mode is fixed to 0x36.
- Direction is forward.
- No threshold is active.
- Selection decoding and required-field configuration are skipped.

Otherwise, +0x1c0 selects a threshold:

| Selector | Compared total | Threshold |
|---|---|---|
| 0 | Unsigned 32-bit accounting total | Low 32 bits of quantity × 60. |
| 1 | Unsigned 64-bit quantity total | Quantity × 2²⁰, subject to separate reads below. |
| 2 | Unsigned 32-bit entry count | Quantity. |
| 3 | Unsigned 32-bit accounting total | Low 32 bits of quantity × 3600. |
| 4 | Unsigned 64-bit quantity total | Quantity × 2³⁰, subject to separate reads below. |
| Other | None | No active threshold. |

For selector 1, the high threshold word is formed from one quantity read shifted right by 12; the low word uses another read shifted left by 20. Selector 4 similarly uses right shift 2 and left shift 30. The product interpretations require both reads to agree. Selectors 0 and 3 retain only the low 32 multiplication bits.

F-4bee99d148b1 receives the then-current selection word and an address of temporary storage. After it returns, the caller rereads +0x1c4 as the retained mode and reads a local byte as the initial direction value.

The caller does **not** establish that F-4bee99d148b1 writes that byte, initializes the temporary storage, or confines its writes to that storage. “Selection decoding” is an inferred role, not a guaranteed dependency effect. If the subsequently read byte lacks an established value, the resulting direction cannot be determined from this caller alone.

If +0x1c8 is nonzero, direction becomes 1 for an initial byte of zero and 0 for every nonzero byte. Otherwise, the byte is retained. Retained zero selects forward traversal; any nonzero value selects reverse traversal.

The required-field rule is latched only when +0x1c8 and +0x1b2 are both zero and the retained mode is one of 0x0b, 0x16, 0x1a, 0x1b, 0x1d, 0x2b, or 0x2c. Later predicate configuration does not recompute this rule.

### Candidate acquisition

Mode 2 calls F-81e0dd8ea0d5 with the captured parent’s selection-object pointer, zero, and auxiliary bytes +0x8e and +0x8f. The auxiliary pointer is accessed separately for these byte arguments; a single stable auxiliary object is not independently guaranteed.

Other modes use F-ec96abf5fded. Before that call, selection fields +0x414 and +0x418 are examined in that order:

- A null pointer is left alone.
- A pointer to a zero first byte remains installed and is not saved for restoration.
- A pointer to a nonzero first byte is saved, and its field is cleared.

F-ec96abf5fded then receives the selection-object pointer and retained mode.

Saved pointers are restored in +0x414, then +0x418 order. Before each restoration, the current field is read. If nonzero, F-d4e440af2e49 is called with an argument not established by the supplied representation. The saved pointer is then written back.

The captured parent’s selection-object pointer is repeatedly reread, including after dependency calls. These accesses therefore need not address one unchanged selection object.

A zero candidate result returns zero through the exit helper. Ordinary-mode restoration occurs before this decision. No preparation, marking, reconciliation finalization, or status clearing follows.

### Preparation and existing collection

For nonzero candidates, F-2bec6f6726b6 receives four ordered calls with the target and these explicit word pairs:

1. 0x20000000 and 0.
2. 0x40000000 and 0.
3. 0 and 0x00400000.
4. 0 and 0x00800000.

Each call also receives a secondary machine-result word from preceding operations. Ordinary-mode restoration calls can affect the first such value. Its meaning and stability are unknown.

F-ec96abf5fded then receives the target and retained mode to acquire the existing collection.

If this result is zero, F-d4e440af2e49 receives the candidate collection, and the function returns zero through the exit helper. Marking, finalization, and status clearing do not occur.

### Candidate eligibility marking

The caller rereads +0x1b2. If nonzero, it reads +0x1b8. When both enable and argument are nonzero, F-bc06e21a2603 receives the address of a 24-byte local workspace. Otherwise, predicate evaluation is disabled.

There is no established caller-side initialization of this workspace before that call. The call does not itself prove initialization of any byte, complete initialization, or a bounded write footprint. Its initialization role is inferred. The workspace address is subsequently passed to predicate calls.

The exclusion-control byte +0x1b3 is captured after the workspace-related call, when present.

Candidates are visited forward, with count reread at each loop condition. For each nonnull entry, its metadata pointer is captured before predicate evaluation. Eligibility checks proceed in order:

1. If enabled, F-9e5616121879 receives the entry, predicate argument, and workspace address. A zero return rejects the entry.
2. If exclusion is enabled, metadata flag 0x04 rejects it.
3. If the required-field rule is latched, a zero value in the following field rejects it:

| Mode | Required metadata field |
|---|---|
| 0x0b | Word +0x6c |
| 0x16 | Word +0x58 |
| 0x1a | Word +0x90 |
| 0x1b | Word +0x98 |
| 0x1d | Byte +0x7e |
| 0x2b | Word +0xa0 |
| 0x2c | Word +0xa8 |

Rejected entries receive marker 0; others receive marker 1. This write uses the metadata pointer captured before the predicate call, even if that call changes the entry’s metadata pointer.

### Existing-entry marking

The existing collection is visited forward, with count reread at every loop condition.

For each nonnull entry:

- Marker zero causes F-d624eceacbef to be called and records a removal-related event. The argument is not established here.
- Any nonzero marker is overwritten with 2.

Marker 2 establishes only that this pass encountered nonzero metadata. An existing entry absent from candidate marking may already have had a nonzero marker. Interpreting 2 as “eligible existing candidate” requires additional, unchecked coverage, shared-metadata, initialization, and mutation invariants.

### Accounting

Accounting captures metadata before calling F-d7be9cfb4000 with metadata +0x5c and 1000. The returned 32-bit contribution is added to the accounting total. Metadata +0x70 then contributes to the two-word quantity total, and the entry count increments.

The accounting total and entry count wrap modulo 2³². Quantity accumulation uses a carry into a 32-bit high word. With consistent quantity reads, this implements modulo-2⁶⁴ accumulation of unsigned 32-bit contributions.

The quantity field is read once for carry determination and again for low-word addition. If those reads differ, ordinary addition of one stable contribution is not guaranteed.

A threshold is crossed only when the updated unsigned total is **strictly greater** than the threshold. Equality is accepted. The latch, once set, is never cleared, including after wraparound. Later enable-byte changes do not recalculate threshold selection or values.

### Mode-2 existing-entry prepass

If retained mode is 2 and a fresh +0x1b1 read is nonzero, the existing collection receives another forward pass. Its count is captured once.

For each nonnull entry with a nonzero marker:

- Before crossing, accounting occurs.
- The crossing entry is passed to F-d624eceacbef.
- After crossing, later qualifying entries are passed without accounting.
- The captured metadata marker is cleared after the applicable calls, including for entries that remain within limits.

Each removal call records a removal-related event. A crossing in this prepass skips the **entire** main candidate pass, even if a dependency subsequently clears the enable byte.

### Main candidate pass

If the prepass leaves the latch clear, the candidate count is captured once. Entries are visited forward or reverse using retained direction. Null entries and entries whose initially inspected marker is zero are skipped.

For each qualifying entry, +0x1b1 is reread:

- **Zero:** no accounting occurs. A fresh marker read equal to 1 causes F-3779d5070ab5 to receive the entry, target, zero, and zero. A returning call sets the change indicator.
- **Nonzero, latch clear:** accounting and threshold checks occur. If still within limits, a fresh marker read equal to 1 causes the same addition request.
- **Nonzero, latch already set or newly crossed:** no addition is requested. A fresh marker read equal to 2 causes membership-chain traversal.

Fresh marker reads use the captured metadata pointer. Accounting dependencies may therefore change the ensuing decision.

For a traversed chain, each record whose owner equals the target is passed to F-d624eceacbef, recording a removal-related event. Successor addresses are read after removal calls return. Safe continuation depends on undocumented record lifetime and mutation behavior.

A main-pass crossing suppresses later accounting and additions only while the per-entry enable read is nonzero. If a dependency clears that byte, a later marker-1 entry can still take the zero-enable addition branch despite the persistent latch.

### Finalization and return

If any removal-related event was recorded, F-1a4bd6c2e396 receives the target. After it returns, the change indicator becomes 1.

F-69e6eb5ce134 receives the parent/context pointer captured near entry.

- With indicator zero, F-fd61d89535ad receives the target’s currently read parent pointer and the value previously obtained from F-04cf29f6f1ed.
- With indicator one, the caller directly zeroes the first word of the earlier temporary storage. F-ebd0ee53699f receives target +0x48, the then-read notification global, and the target. F-df1f4510064d follows with the target.

The status byte is read and rewritten with bit 0x02 cleared and other bits preserved. F-d4e440af2e49 receives candidates, then the existing collection.

The return indicator is captured before the exit helper receives the entry-saved global word.

## Side effects

Established externally addressed writes include temporary selection-pointer clearing/restoration, candidate marker replacement, conversion of existing nonzero markers to 2, mode-2 marker clearing, and completed-path status-bit clearing.

The changed path also directly zeroes one local temporary word. Passing temporary-storage or workspace addresses is **not** evidence that dependencies write or initialize them. Any dependency-mediated workspace writes, their extent, and their initialization sufficiency remain unknown.

Dependencies may modify globals, configuration, collections, metadata, records, or other storage. No preservation or bounded-write guarantee is established.

There is no visible rollback. Markers are not generally restored. Collection cleanup calls occur in candidate-then-existing order even if the collection addresses alias; alias safety is undocumented.

## Dependencies

| Opaque ID | Established caller interaction |
|---|---|
| F-04cf29f6f1ed | Return value retained for unchanged finalization. |
| F-4bee99d148b1 | Receives selection word and temporary-storage address; a local byte is read afterward. |
| F-81e0dd8ea0d5 | Return used for mode-2 candidates. |
| F-ec96abf5fded | Return used for ordinary candidates or existing collection. |
| F-2bec6f6726b6 | Receives four ordered preparatory requests. |
| F-bc06e21a2603 | Receives workspace address before enabled predicate processing. |
| F-9e5616121879 | Receives entry, predicate argument, and workspace address; zero rejects. |
| F-d7be9cfb4000 | Return contributes to accounting. |
| F-3779d5070ab5 | Receives addition requests. |
| F-d624eceacbef | Receives removal-related calls; argument visibility varies by call site. |
| F-1a4bd6c2e396 | Receives removal-triggered rebuild request. |
| F-69e6eb5ce134 | Receives captured parent/context. |
| F-fd61d89535ad | Receives unchanged-path finalization arguments. |
| F-ebd0ee53699f | Receives changed-path notification arguments. |
| F-df1f4510064d | Receives target after notification. |
| F-d4e440af2e49 | Receives collection cleanup calls; also called during pointer restoration. |

Opaque identities for the entry and exit helpers are unavailable. No dependency’s full implementation, write footprint, ownership behavior, or preservation guarantees are established.

## Acceptance examples

These are conditional static consequences, not executed tests. Unless stated otherwise, dependencies return normally, relevant storage remains valid and stable, and consumed temporary bytes have established values.

1. **Saved global:** The entry global contains A; the entry helper changes it to B; candidate acquisition returns zero. The exit helper receives A, and the result is zero.

2. **Late notification:** An addition request returns, and parent finalization changes the notification global to C. Notification receives C.

3. **Missing existing collection:** Candidates succeed but existing acquisition returns zero. All four preparation calls and candidate cleanup occur; status clearing does not.

4. **Direction without a proven write:** F-4bee99d148b1 returns, and the subsequently read direction byte has established value 2. With adjustment disabled, traversal is reverse; with adjustment enabled, it is forward. Neither outcome requires asserting that the dependency wrote the byte.

5. **Predicate storage:** Predicate enable and argument are nonzero. F-bc06e21a2603 receives the workspace address before predicates. This establishes call order, not workspace initialization. Given a subsequent predicate return of zero, the candidate receives marker 0.

6. **Required field:** Mode 0x1b with the required-field rule latched rejects a zero +0x98 word. A nonzero +0x1b2 during rule selection disables this rule even if the later predicate argument is zero.

7. **Inclusive count limit:** With enable remaining nonzero, count limit 2, and three distinct eligible marker-1 candidates, only the first two in traversal order receive additions.

8. **Post-crossing enable change:** A marker-2 entry crosses a main-pass limit. Its matching-record removal call clears +0x1b1. A later marker-1 entry can receive an addition despite the latch.

9. **Prepass distinction:** A mode-2 prepass crossing skips the entire candidate pass, even if a removal dependency clears +0x1b1.

10. **Uncovered metadata:** An existing entry absent from candidate marking starts with marker 7. Existing-entry marking writes 2 without proving eligibility.

11. **Wrapped threshold:** Selector 0 and stable quantity 0x80000000 produce threshold zero. An accounting total updated from zero by contribution 1 crosses it.

12. **Completed unchanged path:** With no additions or removal events, unchanged finalization runs, status bit 0x02 is cleared, both collections receive cleanup calls, and the result is zero.

## Uncertainties

- **documented_only:** Global meanings, valid domains, and ownership.
- **documented_only:** Full dependency signatures, secondary results, omitted arguments, and entry/exit helper semantics.
- **documented_only:** Whether either temporary-storage dependency initializes any consumed byte; temporary-layout interpretation, read/write footprints, and workspace initialization sufficiency.
- **documented_only:** Allocation ownership, cleanup effects, alias safety, record lifetime, and failure behavior.
- **documented_only:** Permitted mutations of configuration, counts, arrays, parent pointers, markers, and chains.
- **documented_only:** Coverage and initial-marker invariants needed for stronger marker meanings.
- **Inferred intent:** Selection decoding, predicate-workspace initialization, membership reconciliation, duration-like or size-like accounting, rebuilding, and notification.

Invalid inputs, malformed chains, and nonreturning dependencies have no established recovery contract. Return 1 does not establish transactional success.

## Evidence

The supplied static representation establishes the caller’s access widths, offsets, explicit writes, arithmetic, branch conditions, argument passing, call order, saved entry-global dataflow, late notification-global read, and return-indicator assignment.

It establishes temporary-storage address passing followed by local-byte consumption or subsequent workspace use. It does not establish dependency initialization, complete output production, or confined write footprints.

No runtime tests have been performed. Acceptance examples are conditional consequences of static evidence, not observed executions.
