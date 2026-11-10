<!-- Trusted provenance: F-92a1afcfc013; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-92a1afcfc013

## Purpose

F-92a1afcfc013 allocates and prepares a 44-byte temporary record, optionally invokes provider-selected callbacks, conditionally invokes a fallback processor, and publishes record fields into a caller-supplied state object.

A 32-bit status controls processing. Zero selects publication. Status 2 selects a byte-level eligibility check for fallback. Other nonzero statuses bypass publication.

The status storage is exposed by address to the allocator. Consequently, a dependency retaining that address may subsequently change the status while this invocation remains active. Ignoring a dependency’s return value does not establish that the status survives its call unchanged. In particular, the returned status need not describe whether publication already occurred.

“State,” “request,” “record,” and “provider” identify observed roles, not established application-level types.

## Interface

### Arguments and result

| Item | Representation | Role |
|---|---|---|
| State | 32-bit address-bearing value | Supplies initial record fields, an optional provider, and publication destinations. |
| Request | 32-bit address-bearing value | Supplies preparation data, allocation context, callback arguments, and fallback input. |
| Result | 32-bit status | Read from the live status storage at the applicable return point. |

No explicit null, alignment, extent, or address-range validation is visible. Every access and indirect call taken by the selected path requires valid storage or a valid target. Invalid-memory behavior and abnormal dependency termination are not specified.

### State layout

Offsets are byte offsets from the state address.

| Offset | Width | Observed use |
|---|---:|---|
| +0x08 | 32 bits | Read, combined with flag 0x40, and written when publication finds a nonzero record marker. |
| +0x34 | 32 bits | Receives the signed right-shift conversion of record +0x04. |
| +0x38 | 32 bits | Receives the signed right-shift conversion of record +0x08. |
| +0x3c | 32 bits | Receives the unsigned biased conversion of record +0x0c. |
| +0x40 | 32 bits | Receives the unsigned biased conversion of record +0x10. |
| +0x46 | 16 bits | Receives the biased extraction from record +0x14. |
| +0x48 | 16 bits | Receives the biased extraction from record +0x18. |
| +0x84 | Address only | Its address is written into the callback context; not directly dereferenced here. |
| +0x1c8, +0x1cc, +0x1d0, +0x1d4 | Four 32-bit fields | Supply initial record values and receive corresponding values during publication. |
| +0x1e4 | 32-bit address-bearing field | Supplies the optional provider address. |
| +0x1e8 | 32-bit address-bearing field | Receives the record address when publication finds a nonzero marker. |

The provider contains a 32-bit callback-table pointer at +0x18.

### Request layout

| Offset | Width | Observed use |
|---|---:|---|
| +0x04 | 32 bits | Preparation argument; later read as an unsigned size for fallback eligibility. |
| +0x1c | 32 bits | Captured allocation context; separately reread for callback initialization. |
| +0x20 | 32-bit address-bearing field | Callback initialization argument; separately reread as the fallback buffer address. |
| +0x24 | 32 bits | Callback initialization argument. |

These fields are not captured as one snapshot. Dependencies can run between their reads, and later values need not equal earlier ones.

### Record and callback-context layouts

The record allocation requests exactly 0x2c bytes.

| Record offset | Width | Direct handling |
|---|---:|---|
| +0x00 | 32 bits | Neither initialized nor read directly. |
| +0x04 through +0x10 | Four 32-bit fields | Initialized from the four state fields beginning at +0x1c8. |
| +0x14 | 32 bits | Initialized from a separate reread of state +0x1d4. |
| +0x18 | 32 bits | Initialized from a separate reread of state +0x1cc. |
| +0x1c through +0x27 | 12 bytes | Neither initialized nor read directly. |
| +0x28 | 32 bits | Read during publication as a retention marker; not directly initialized. |

The callback receives the address of an eight-byte local scratch region. Three adjacent 32-bit fields follow it:

| Offset from scratch address | Content established after initialization succeeds |
|---|---|
| +0x08 | Record address |
| +0x0c | Value read from an opaque 32-bit global |
| +0x10 | Address of state +0x84 |

This function does not visibly initialize the scratch bytes before calling initialization. It establishes the three following fields only after initialization returns zero. Their assignment order is specified below; it is not ascending offset order.

## Behavior

### Allocation and preparation

The first request-field read captures request +0x1c as the allocation context. The function initializes its live 32-bit status storage to 2 and calls F-3ce007b692cc with that context, the size 0x2c, and the status address.

The allocator’s returned address becomes the record address if the resulting status is zero. A nonzero status causes immediate return, without a direct release or request-finalization call. If the allocator leaves the status unchanged, this return is 2.

The status address has escaped at this point. The source does not establish whether F-3ce007b692cc retains or communicates it.

After allocation status zero, the function reads request +0x04 and calls F-d036abe1e536 with the request address and that value. Its return value is assigned to the live status storage, replacing any earlier value there.

If the assigned preparation status is nonzero, the function calls F-bf7242c98d06 with the originally captured allocation context and the record address. It then reads and returns the live status. It does not call F-9787e15c337d on this path.

The release operation’s return value is ignored. However, release may change the live status through an address retained from allocation. Therefore, returning the original preparation error is conditional on no such intervening modification. Even if release changes the status to zero, this path returns directly; it does not resume record initialization or publication.

### Record initialization

After preparation returns zero, direct record initialization has the following observable order:

1. Read state +0x1cc, then +0x1d0, then +0x1d4, retaining their 32-bit values.
2. Read state +0x1c8 and write that value to record +0x04.
3. Write the three retained values to record +0x08, +0x0c, and +0x10, respectively.
4. Reread state +0x1d4 and write it to record +0x14.
5. Reread state +0x1cc and write it to record +0x18.
6. Read state +0x1e4 as the provider address.

The duplicate source fields are genuinely reread. Equality between their earlier and later values is not an independent guarantee, particularly if storage aliases.

No zero-initialization of the record is established.

### Optional callbacks and context construction

A zero provider address skips callbacks. Otherwise, the function reads the table pointer at provider +0x18. A zero table pointer also skips callbacks. The no-callback path selects processing status zero, not 2; absence of callbacks does not itself trigger fallback.

For a nonzero table, the entries are 32-bit callable addresses:

| Table offset | Role |
|---|---|
| +0x00 | Initialization |
| +0x04 | Callback finalization |
| +0x08 | Processing |

Initialization receives the scratch address and separately read values from request +0x1c, +0x20, and +0x24. The relative evaluation order of those three request-field reads is not established by the supplied representation.

The initialization return is assigned to the live status. If nonzero, processing and callback finalization are skipped, and that status proceeds toward fallback selection or common finalization.

If initialization returns zero, context construction occurs in this order:

1. Read an opaque 32-bit global and write its value to context +0x0c.
2. Write the record address to context +0x08.
3. Write the address of state +0x84 to context +0x10.

The global is read after initialization succeeds. Its value is not established to be an embedded constant or an entry-time snapshot. Allocation, preparation, initialization, or other earlier dependency effects may change the value observed by this read.

Next, the function rereads the table pointer from the already selected provider’s +0x18 field and invokes its +0x08 entry with the scratch address. The processing return is assigned to the live status.

It then rereads provider +0x18 again and invokes that table’s +0x04 entry with the same scratch address. The finalization return value is ignored. Only after this call does the function read the live status for subsequent processing.

Thus, the effective post-callback status is not unconditionally the processing callback’s return. Callback finalization may change the live status through the escaped address. Such a change can select fallback, select publication, or suppress either.

The provider address remains the one selected before initialization. Changes to state +0x1e4 do not select a replacement provider within this sequence. Changes to provider +0x18 can select different processing and finalization tables. Those later table-pointer reads have no renewed null checks.

### Conditional fallback

Only effective status 2 enters the fallback gate.

The function first reads request +0x20 as the buffer address, then reads request +0x04 as an unsigned 32-bit size.

- A size below 7 rejects fallback without reading buffer bytes.
- Otherwise, byte 0 is read.
- Byte 1 is read only if byte 0 equals zero.
- The required prefix is byte 0 equal to 0 and byte 1 equal to 1.
- Only after that prefix matches are bytes 2 through 5 used to assemble an unsigned little-endian 32-bit quantity.
- That quantity must equal a fresh read of request +0x04.

The size used for the comparison is not necessarily the value used for the minimum-size check. The relative evaluation order within the assembled-size comparison is not fully established.

A failed gate leaves status 2 in place before common finalization. The gate requires a size of at least 7 but imposes no direct value requirement on byte 6.

A matching gate invokes F-7f73efe33c6b with the state, request, and record addresses. Its return is assigned to the live status. This assignment replaces any status modification made through the retained address during the fallback call.

### Publication

Publication is entered only when the status tested after callback and optional fallback processing is zero. Record reads occur after those dependencies, so their record modifications can affect the published values.

The first publication group:

1. Reads record +0x08, +0x0c, and +0x10 into retained values.
2. Reads record +0x04 and writes it to state +0x1c8.
3. Writes the retained values to state +0x1cc, +0x1d0, and +0x1d4.

Derived-field writes follow, in this order, using fresh record reads:

| Destination | Conversion |
|---|---|
| State +0x34 | Interpret record +0x04 as signed 32-bit and shift right by 16 with sign extension. |
| State +0x38 | Apply the same conversion to record +0x08. |
| State +0x3c | Add 0xffff to record +0x0c in unsigned 32-bit arithmetic, then logically shift right by 16. |
| State +0x40 | Apply the same unsigned conversion to record +0x10. |
| State +0x46 | Add 0x8000 to the signed 32-bit value at record +0x14, convert the result to unsigned 32-bit, logically shift right by 16, and store 16 bits. |
| State +0x48 | Apply the corresponding biased extraction to record +0x18. |

The 0xffff additions wrap modulo 2³²; they do not saturate. The signed 0x8000 additions require the overflow qualification under **Uncertainties**.

The function then reads the 32-bit marker at record +0x28. If nonzero, it reads state +0x08, sets bit 0x40 while preserving its other bits, writes that field back, and then writes the record address to state +0x1e8.

A zero marker causes neither of those two writes. Existing flags and an existing stored record address are not directly cleared.

### Common finalization and return

Every ordinary completion path after successful preparation calls F-9787e15c337d with the request address. This includes callback errors, rejected fallback gates, fallback errors, and completed publication.

Its return value is ignored. The function subsequently reads and returns the live status storage.

If this finalizer changes that storage through the escaped pointer, the returned status changes without reconsidering earlier branches. Publication may therefore occur before a nonzero return, or be skipped before a zero return. No rollback or delayed publication is visible.

## Side effects

Direct writes affect the allocated record, callback-local context, live status storage, and—conditionally—the listed state fields. The opaque global supplying context +0x0c is directly read, not directly written.

Dependencies receive addresses and may have additional effects. No dependency is assumed to preserve the global, request, state, record, provider tables, or escaped status storage.

The only explicit record-release call occurs after preparation failure. No direct release is visible on later error or success paths.

Storing the record address when its marker is nonzero suggests retention, but does not establish ownership transfer. Allocation ownership, dependency-side release behavior, and the record’s lifetime after request finalization remain unknown. A zero marker does not prove that the record is freed. An existing state-held record is not directly released here.

Nonzero return does not imply rollback. Zero return does not independently prove publication, because release or common finalization may change the status after the relevant control-flow decision.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-3ce007b692cc | Receives allocation context, size 44, and the live status address; returns an address-bearing value. |
| F-d036abe1e536 | Receives the request and its current +0x04 value; its return is assigned as preparation status. |
| F-bf7242c98d06 | Receives the original allocation context and record address after preparation failure; its return is ignored. |
| F-7f73efe33c6b | Receives state, request, and record after status 2 passes the input gate; its return replaces status. |
| F-9787e15c337d | Receives the request on ordinary completion after successful preparation; its return is ignored, but live status is read afterward. |

Three indirect dependencies occupy callback-table offsets +0x00, +0x04, and +0x08. Their concrete opaque function IDs are unavailable. Their call interfaces and sequencing are established above; their internal behavior is not.

The opaque global is a data dependency, not an established constant or function dependency.

## Acceptance examples

These are conditional static expectations, not runtime observations. They assume valid memory, normal dependency returns, and no asynchronous mutation. Status-preservation assumptions are explicit where relevant.

1. **Allocator leaves status unchanged.** If F-3ce007b692cc leaves the initial status at 2, the function returns 2 immediately. Preparation and common finalization are not called.

2. **Preparation error without subsequent status modification.** If allocation succeeds, preparation returns 9, and release leaves live status unchanged, release receives the original allocation context and record address; the result is 9. Common finalization is not called.

3. **Release modifies escaped status.** If preparation returns 9 but release changes live status to 0 through the retained address, the function returns 0 directly. It does not initialize or publish the record.

4. **No provider.** With successful allocation and preparation and a zero provider, callbacks and fallback are skipped and publication is entered. A readable record marker is required despite its lack of direct initialization. The return is zero only if common finalization leaves status zero.

5. **Global changed during initialization.** If successful initialization changes the opaque global to a particular 32-bit value and no further change precedes its read, context +0x0c receives that value. This write precedes the record-address write at +0x08 and the state-relative-address write at +0x10.

6. **Initialization returns 5.** Processing and callback finalization are skipped. Publication is skipped and common finalization runs. The result is 5 if common finalization leaves status unchanged.

7. **Callback finalization changes processing status.** If processing returns 2 but callback finalization changes live status to 0 through the escaped address, fallback is skipped and publication is entered. Conversely, changing a processing result of 0 to 2 selects the fallback gate.

8. **Rejected prefix.** If the effective status remains 2, size is at least 7, and byte 0 is nonzero, fallback is rejected without reading byte 1. Publication is skipped. The return remains 2 only if common finalization preserves status.

9. **Minimum accepted size.** With effective status 2, an unchanged size of 7, and leading bytes 00 01 07 00 00 00, fallback is invoked. Byte 6 has no gate-imposed value. A fallback return of zero selects publication.

10. **Short input.** Size 6 rejects fallback before buffer-byte reads, even if accessible memory contains an otherwise matching prefix and embedded size.

11. **Arithmetic boundaries.** Record +0x0c equal to 0xffffffff produces zero at state +0x3c after unsigned wrapping addition and shift. Record +0x04 equal to 0xffff8000 produces 0xffffffff at state +0x34 through sign-extending shift. Record +0x14 equal to 0x00008000 produces 0x0001 at state +0x46.

12. **Final status differs from publication decision.** If publication occurs and common finalization then changes live status to 7, the function returns 7 without undoing publication. If publication was skipped with status 5 and finalization changes it to 0, the function returns 0 without performing publication.

## Uncertainties

**documented_only — application semantics.** Measurement units, fixed-point meaning, and record purpose are not established. Such interpretations are inferred intent, not requirements.

**documented_only — escaped status address.** Allocation receives the address of status storage that remains live during later calls. Retention or communication of that address is unknown. Later modification through it does not inherently require invalid-memory access. Conversely, the source does not establish that any dependency actually performs such modification.

**documented_only — global meaning and stability.** The callback-context datum comes from an opaque 32-bit global read after successful initialization. Its meaning, initial value, and dependency-side writers are unavailable.

**documented_only — initialization and ownership.** Allocation initialization policy, origin of the record marker, ownership rules, and eventual release are unknown.

**documented_only — callback context.** Contiguous local storage supports a larger-context interpretation, but the dependency-side type, permitted writes, and retention rules are unavailable. This function does not establish persistent lifetime for the local context.

**documented_only — signed arithmetic.** The 0x8000 biases are represented as signed 32-bit additions. Positive inputs above 0x7fff7fff exceed the signed range after addition. Modular target behavior cannot be promoted into a portable signed-overflow guarantee. Likewise, sign-extending right shifts describe the observed signed conversion, not a general guarantee for every language or target.

**documented_only — ordering and exceptional execution.** The staged access order is established, but argument-expression and assembled-size subexpression ordering is incomplete. Faults, nonreturning calls, asynchronous mutation, and concurrency are unspecified.

## Evidence

Established behavior derives solely from the supplied static representation: access offsets and widths, branch conditions, constants, direct-call identities, callback-table offsets, assignments, and return flow.

The global read and its placement before the other context-field writes are explicit. Passing the live status address to allocation, reading status after release and callback finalization, and reading it after common finalization are also explicit.

Retention of the status address, dependency ownership effects, global semantics, and application-level intent remain unresolved. No runtime tests have been performed.
