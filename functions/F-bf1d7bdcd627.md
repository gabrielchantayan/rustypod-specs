<!-- Trusted provenance: F-bf1d7bdcd627; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bf1d7bdcd627

## Purpose

F-bf1d7bdcd627 initializes shared transfer state, obtains data through one of two producer interfaces, submits fixed 1,024-byte blocks, and polls a consumer for completion. It prepares a nominal 5,120-byte staging region, a separate 4,096-byte region, and two separately requested 1,024-byte transfer buffers.

“Producer,” “consumer,” “staging,” “allocation,” and “release” describe inferred roles. The established contract is the visible argument forwarding, memory accesses, arithmetic, calls, and control flow; dependency implementations are unavailable.

The selected 32-bit result is zero for consumer status `1` and one for explicitly detected failures. Actual returnability remains unresolved: both descriptor-release calls are represented as nonreturning. Later cleanup and the final return are conditional on execution continuing past those calls.

## Interface

### Arguments and result

The interface has nine 32-bit arguments.

| Argument position | Established use |
|---|---|
| 1 | Exactly `1` selects the direct producer; all other values select the object-based producer. |
| 2 | Forwarded to source-object construction. |
| 3–4 | Forwarded to the direct producer. |
| 5 | Published into shared control state; its working copy subsequently serves as the staging-block index. |
| 6–9 | Published into four consecutive shared control words. |

No application-level types or ownership obligations are established for arguments 2–9.

The nominal result is selected as follows:

- `0`: consumer low-word status equals `1`.
- `1`: a producer returns a nonzero low-word status, construction returns null, the constructed object contains the rejection sentinel, or the consumer returns a nonzero low-word status other than `1`.

Producer and consumer results are represented as 64-bit values. Only their low 32 bits determine failure, continuation, or completion.

### Descriptor layout

Two 16-byte descriptors use the following fields:

| Byte offset | Width | Observed role |
|---|---:|---|
| `0` | 32 bits | Original backing value |
| `4` | 32 bits | Derived address, potentially high-bit tagged |
| `8` | 32 bits | Capacity |
| `12` | 8 bits | Initialization flag |

Bytes `13–15` are not explicitly initialized.

The first descriptor has capacity `4096` and requests `4160` backing bytes. The second has capacity `5120` and requests `5184` backing bytes.

### Shared control layout

A global pointer supplies shared control storage. Its observed field layout is:

| Byte offset | Width | Value explicitly published during initialization |
|---|---:|---|
| `0` | 32 bits | Zero through a fresh global-pointer read |
| `4`, `28` | 32 bits each | Zero through the earlier captured pointer |
| `8` | 32 bits | First descriptor’s derived address plus `512` |
| `12` | 32 bits | First descriptor’s derived address |
| `16` | 32 bits | That address plus `256` |
| `20` | 32 bits | That address plus `2560`, rounded down to a 1,024-byte boundary |
| `24` | 32 bits | Offset-20 value plus `1024` |
| `32–48` | Five 32-bit words | Arguments 5–9 |
| `52`, `56` | 32 bits each | Results of separate 1,024-byte allocation requests |

Except for the zero store at offset `0`, initialization writes use the earlier captured control pointer. The offset-0 store therefore need not address the same region as the remaining stores.

A separate global state pointer is captured after external initialization. Its accessed fields are:

| Byte offset | Width | Observed role |
|---|---:|---|
| `76` | 32 bits | Submission-mode selector |
| `100`, `104` | 32 bits each | Transfer destinations |
| `132` | 32 bits | Position subtracted from the producer position |
| `140` | 32 bits | Producer position |

### Preconditions and arithmetic limits

Addresses, sizes, indices, and position fields are represented by 32-bit words. Address additions, index scaling, alignment calculations, and position updates have no explicit overflow checks. Their observed bit-level calculations use 32-bit results; this does not establish that wrapped addresses identify valid storage.

Descriptor-allocation results are dereferenced without null checks. Null backing or transfer-buffer results do not select an explicit failure path. Safe execution therefore requires accessible storage and compatible dependency behavior; allocation-failure recovery is not established.

There are no explicit staging-capacity checks before block submission.

## Behavior

### Descriptor preparation

The 4,096-byte descriptor is prepared first, followed by the 5,120-byte descriptor.

For each descriptor, F-8157a180d901 receives size `16`. The function writes the original backing field to zero, the capacity to its fixed value, the derived-address field to zero, and the initialization byte to zero, in that order.

F-638beded7699 then receives the backing request size and selector `3`. Its result is stored in the original backing field.

For a nonzero backing result:

- Adding `31` and clearing the lowest five bits produces the initial derived address.
- That address is stored at descriptor offset `4`.
- If bit `27` of the 32-bit addition result is set, F-20810a394e6a receives the derived address and the descriptor’s current capacity.
- After that call returns, bit `31` is set in the descriptor’s then-current offset-4 field. The call is not assumed to preserve that field.

A counter-only loop follows, governed by the capacity field, and the initialization byte is set to `1`. The loop contains no backing-memory stores and establishes no zero-fill guarantee.

For a zero backing result, this branch is skipped. The function’s own derived-address and initialization-byte values remain zero, and processing continues.

### Shared-state initialization and pointer identity

The initialization access sequence distinguishes two reads of the global control pointer:

1. Capture the global control pointer.
2. Read and retain the second descriptor’s derived address as the staging address.
3. Read the global control pointer again and store zero at offset `0` through this freshly obtained pointer.
4. Store zero at offsets `4`, `8`, and every subsequent four-byte offset through `56`, in increasing order, through the pointer captured in step 1.

These are fourteen zero stores through the earlier pointer, following one zero store through the fresh pointer. Equality of the two pointer values is not guaranteed by the body.

The function next reads the first descriptor’s derived address. Through the earlier captured pointer, it publishes layout values in offset order `12`, `16`, `8`, `20`, `24`. Offset `20` is calculated by adding `2560` and clearing the lowest ten bits; offset `24` receives that result plus `1024`.

Two F-638beded7699 calls each receive size `1024` and selector `0`. Each returned value is stored through the earlier captured pointer immediately after its call: first at offset `52`, then at offset `56`. Arguments 5–9 are then written at offsets `32`, `36`, `40`, `44`, and `48`.

F-unmapped-01 receives `262144`, `0`, and `1`. Only after it returns is the separate global state pointer captured.

Initialization is not transactional. Dependencies may observe partially initialized fields. Later changes to the global control pointer do not retarget stores through an already captured pointer. Concurrency and pointer stability are unresolved; the body supplies no synchronization guarantee.

### Producer selection and construction

Transfer processing begins with zero pending blocks, no source object, zero producer output words, and no consumer terminal status.

A producer refill is attempted only when the pending count is not positive and the low eight bits of the end-state word are zero.

For argument 1 equal to `1`, F-88718fabb791 receives, in order:

- The captured staging address and capacity `5120`.
- References to progress, count, and end-state words.
- Arguments 3 and 4.
- A reference to an auxiliary output word.

For other modes, a source object is constructed lazily on the first refill. F-8157a180d901 receives two supplied words, `84` and `0`. The second word comes from the upper half of an internal 64-bit result accumulator. It is zero on this first construction path because the accumulator starts at zero and no producer or consumer call precedes construction. Its dependency-level meaning is unknown.

The returned storage value is passed to F-3fc3239fa339 with argument 2 and fixed words `1`, `0`, `1024`, `1`, `0`.

Null construction selects failure. For a nonnull object, its 32-bit field at offset `24` is read; the value `0xffffffff`, interpreted as signed `-1`, also selects failure. A rejected nonnull object remains retained for conditional cleanup. No separate release of the initial storage request is visible when construction returns null.

An accepted object is reused. F-1713355bc5e0 receives the staging address, capacity `5120`, references to count, end-state, and auxiliary words, and the object.

Producer output words persist between calls unless a dependency changes them. A nonzero producer low result word selects failure immediately, bypassing the consumer call for that iteration.

### Pending blocks and staging offsets

After a successful producer call, a nonzero count resets the working staging-block index to zero and replaces the pending count:

- End-state low byte zero: unsigned count divided by `1024`, discarding the remainder.
- End-state low byte nonzero: that quotient plus one.

The terminal rule is not ordinary ceiling division: an exact nonzero multiple of `1024` still gains an additional block.

A zero count leaves the index and pending count unchanged. In particular, zero terminal output does not create an empty final block.

The largest unsigned 32-bit count produces quotient `4,194,303`, or `4,194,304` pending blocks after the terminal increment. Pending blocks are decremented only when nonzero. Consequently, the reachable pending count remains within positive signed 32-bit range, and the signed refill test does not conflict with the nonzero submission test.

This bound does not protect the staging region. Large reported counts can cause submissions beyond its nominal 5,120-byte capacity.

### Submission and backpressure

Each submission attempt reloads the global control pointer. The separate state pointer remains the one captured after initialization.

The selected staging address is the captured staging base plus the working block index multiplied by `1024`, without a bounds check.

When state offset `76` equals `1`, the function:

- Writes zero to state offset `140`.
- Reads control offset `52` and writes it to state offset `100`.
- Reads control offset `56` and writes it to state offset `104`.
- Calls F-unmapped-02 with the first transfer destination, selected staging address, and size `1024`.

In other submission modes, the 32-bit difference between state offsets `140` and `132` is interpreted as unsigned:

- A difference greater than `1023` defers submission.
- Otherwise, the producer position is read again. Its bit `10` selects the destination: clear selects state offset `100`; set selects offset `104`.

Thus the position used for destination selection is a separate read from the position used in the difference check. Concurrent changes are not ruled out.

A permitted submission calls F-unmapped-02 for exactly `1024` bytes. No returned copy status is checked.

After this call returns, the pending count decreases by one, the staging-block index increases by one, and state offset `140` is read again and increased by `1024`. The update uses the post-call field value, not an assumed preserved pre-call position.

A deferred submission does not change those counters or the position through this function. Terminal blocks can extend beyond the reported byte count; no padding or tail clearing is established.

### Consumer polling

After a submitted block, deferred block, or iteration without a block, F-c696a887f1a7 receives a freshly loaded global control pointer.

Its low 32-bit result determines the next action:

| Status | Effect |
|---|---|
| `0` | Continue processing |
| `1` | Select nominal success |
| Any other nonzero value | Select nominal failure |

A nonzero end-state low byte suppresses future producer calls once pending blocks are exhausted; it does not terminate consumer polling.

There is no visible timeout or iteration limit. Consumer status zero can therefore sustain indefinite polling under backpressure or after production ends.

## Side effects

The function requests descriptor storage, backing storage, transfer buffers, and optionally source-object storage. It writes descriptor metadata, shared control fields, and selected state fields. Dependency effects beyond these observed boundaries remain unknown.

All represented processing exits invoke F-fa161ddc0e09 before cleanup.

### Descriptor and object cleanup

The first descriptor is handled first. If its initialization byte is nonzero, its capacity governs a counter-only loop when nonzero. The initialization byte is then cleared. A nonzero original backing value is passed to F-8349c252c168 with selector `3`. The descriptor is passed to F-feee9156a257.

F-feee9156a257 is represented as nonreturning. Taken literally, the first descriptor’s release prevents subsequent cleanup and the nominal return on the ordinary nonnull-descriptor path.

If execution continues, a retained nonnull source object receives F-unmapped-03. This indirect call is obtained from byte offset `4` of the dispatch table referenced by the object’s first word. Object destruction, storage release, and ownership effects are not established.

After this optional dispatch, the global control pointer is captured for later buffer cleanup. The second descriptor then receives the same flag handling, optional backing release, and represented-nonreturning descriptor-release call.

### Transfer-buffer cleanup

If execution continues past second-descriptor cleanup:

1. A fresh global control-pointer read supplies offset `52` as the first F-8349c252c168 argument, with selector `0`.
2. After that call returns, offset `52` is cleared through the pointer captured before second-descriptor cleanup.
3. Offset `56` through that earlier captured pointer supplies the second release argument, again with selector `0`.
4. After the second call returns, offset `56` is cleared through that same earlier pointer.

Neither transfer-buffer release argument receives a visible null check. The first release can read one control region and clear another. Other published address fields are not cleared and can remain stale.

Names such as “release” do not establish actual deallocation or ownership transfer.

## Dependencies

| Opaque ID | Observed boundary |
|---|---|
| F-8157a180d901 | Descriptor requests of `16`; initial source-storage request with words `84`, `0` |
| F-638beded7699 | Size-and-selector requests for backing regions and transfer buffers |
| F-20810a394e6a | Derived address and current capacity before high-bit tagging |
| F-3fc3239fa339 | Source-object construction boundary |
| F-88718fabb791 | Direct producer |
| F-1713355bc5e0 | Object-based producer |
| F-c696a887f1a7 | Consumer continuation and completion boundary |
| F-fa161ddc0e09 | Pre-cleanup call |
| F-8349c252c168 | Backing or transfer-buffer release boundary |
| F-feee9156a257 | Descriptor release, represented as nonreturning |
| F-unmapped-01 | External initialization |
| F-unmapped-02 | Fixed-size transfer boundary |
| F-unmapped-03 | Indirect source-object cleanup |

The three unmapped IDs are document-local opaque labels. No dependency is assumed to preserve globals, descriptor fields, state fields, or ownership.

## Acceptance examples

These are conditional static expectations, not executed tests. Storage validity and dependency returns are prerequisites where required.

1. **Stable initialization pointer.** If both initialization reads obtain the same control pointer, offsets `0–56` receive zero stores in increasing four-byte order before layout publication.

2. **Changed initialization pointer.** If the initial capture identifies region A and the fresh read after staging-address capture identifies region B, offset `0` is cleared in B. Offsets `4–56` are cleared in A, and subsequent layout, buffer-result, and argument publications use A. The function does not explicitly clear A at offset `0` or B at offsets `4–56`. This example does not assert that concurrent replacement is safe.

3. **Mode selection.** Argument 1 equal to `1` selects the direct producer without source construction. Every other value selects lazy object construction.

4. **Construction rejection.** The first object-path refill requests storage with words `84`, `0`. Null construction or object offset `24` equal to `0xffffffff` selects failure.

5. **Ordinary output.** Count `2048` with end-state low byte zero establishes two pending blocks, using staging offsets `0` and `1024` when permitted.

6. **Terminal exact multiple.** Count `2048` with terminal end-state establishes three pending blocks. The third transfer remains `1024` bytes; valid padding is not guaranteed.

7. **Short or empty output.** Nonterminal count `1023` creates no block. Terminal count zero also creates no block and suppresses later refills while the end-state byte remains nonzero.

8. **Maximum count.** Terminal count `0xffffffff` establishes `4,194,304` pending blocks, still positive as a signed 32-bit value. It does not establish valid staging storage for those blocks.

9. **Backpressure boundary.** Outside reset mode, unsigned position difference `1023` permits submission; `1024` defers it.

10. **Completion and failure.** Producer failure skips that iteration’s consumer call. Consumer statuses `1` and `2` select nominal results `0` and `1`, subject to cleanup returnability.

11. **Cleanup pointer replacement.** If the pre-second-cleanup capture identifies A but the later fresh read identifies B, the first buffer release reads B at offset `52` and then clears A at offset `52`. The second release reads and clears A at offset `56`, subject to intervening dependency effects.

## Uncertainties

The following remain **documented_only**:

- Actual returnability of F-feee9156a257 and reachability of subsequent cleanup.
- Dependency implementations, concrete application types, and ownership rules.
- Global-pointer stability, concurrency, reentrancy, and synchronization.
- Interpretation and validity of high-bit-tagged addresses.
- Producer-output meanings beyond their observed effects.
- Meaning of upper result words and the second source-storage request word.
- Valid storage for terminal overreads or oversized reported counts.
- Allocation-failure behavior and construction ownership transfer.
- Indirect cleanup semantics and dependency mutation of shared or descriptor fields.
- Any intended timing effect of counter-only loops.

## Evidence

Established static evidence consists of the supplied function body and dependency mapping: field widths and offsets, constants, argument order, memory-access sequence, arithmetic, branches, and represented calls.

The initialization distinction follows from a control-pointer capture before the staging-address read, a separate global-pointer read for the offset-0 store, and use of the earlier capture for the remaining fourteen zero stores. Pointer equality requires an additional stability assumption.

The pending-count bound follows from unsigned 32-bit division by `1024`, an optional increment, and guarded decrements. The zero second source-storage word follows from accumulator initialization and first-construction ordering. Cleanup pointer distinctions follow from separate captured and fresh global-pointer reads.

Transfer roles and resource-management terminology are inferred intent, not dependency guarantees. No implementation access, dependency inspection, or runtime tests have been performed.
