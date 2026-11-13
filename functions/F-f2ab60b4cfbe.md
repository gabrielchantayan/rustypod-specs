<!-- Trusted provenance: F-f2ab60b4cfbe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2ab60b4cfbe

## Purpose

F-f2ab60b4cfbe dispatches a message according to a signed 16-bit selector. It checks structural conditions, conditionally initializes global state, performs selector-specific work, and ordinarily invokes common completion.

The established contract covers represented field accesses, widths, comparisons, dependency arguments, call ordering, and stores. Terms such as “message,” “response,” “selection,” and “size” describe observable roles, not verified protocol semantics.

Dependency implementations are unavailable. Calls that resemble encoding, allocation, copying, or cleanup do not establish those behaviors, memory extents, or ownership rules.

## Interface

### Inputs and message layout

The inputs are an opaque 32-bit context value and a message address represented in a 32-bit address space.

| Byte offset | Represented value | Use |
|---|---|---|
| 0 | 32-bit word | Identifier captured before classification |
| 8 | 32-bit address | Payload address captured after classification |
| 12 | Unsigned 16-bit integer | Payload length captured after classification |
| 16 | Word narrowed to signed 16 bits | Dispatch selector, subsequently reread for completion |

The selector expression establishes narrowing and signed interpretation, not independently the physical load width. Unlisted message fields have no established contract here.

The identifier is read before the first dependency call and before any message-null check. Null entry is therefore not a supported input. Later null-message branches do not remove this precondition.

A null payload is rejected when the captured length is nonzero. A nonnull payload is not otherwise validated. Readability, writable storage, alignment, and dependency-specific bounds remain caller or dependency preconditions.

### Return and status

The represented return type is signed 32-bit. Its provisional value starts at 62. A separate 32-bit operation status starts at 4.

Operation status is not generally the return value. On ordinary nonnull-message completion, F-256e8a77aa59 supplies the final return value, replacing any provisional result.

Several dependencies are annotated as non-returning. The supplied representation establishes no normal continuation after those calls; their actual termination mechanism is unknown.

## Behavior

### Entry and initialization

The identifier is captured first. F-5f5f14cedda3 then receives the message address and produces the initial signed 32-bit classification.

After classification, the function captures a global-object address, then captures the selector, length, and payload address in that order. Classification may therefore change the later message observations without changing the saved identifier.

F-a817d8d58c10 is invoked before initialization or dispatch if:

- Classification is greater than 2 under signed comparison.
- The captured identifier is zero.
- The payload address is null and its captured length is nonzero.

There is no lower-bound classification check.

The function reads initialization-object byte 1. Only when that byte is zero does it read byte 2. Observations of zero and 6 respectively trigger a store of 1 to byte 1, followed by U-01. After U-01 returns, the function reads the 32-bit field at offset 8 through the previously captured global-object address and passes that value, together with U-01’s result, to F-8becc2a0337c.

The initialization store precedes both calls; no rollback is visible. The offset-8 field is not snapshotted before U-01. In particular, U-01 may alter that field or global state.

F-1a836ac93559 subsequently receives the captured identifier and 3. Its 32-bit result is the initial size-related value used by operation 6.

### Explicit responses and common completion

Every explicit F-dece790ddc92 response supplies the message, captured identifier, constants 0 and 3, a response selector, a payload address, and a length. Its return value is ignored. The call alone establishes neither message mutation nor copying, retention, or consumption of payload storage.

Ordinary completion first checks whether the message address is nonnull. For a nonnull message, it rereads and narrows the selector for comparison with the captured dispatch selector.

**If the selectors match, two 32-bit stores occur before the fallback call:**

1. The cell also used as operation 6’s output handle and operation 15’s first query output is assigned 255, equivalently 0x000000ff.
2. The cell used as operation 15’s second query output is assigned zero.

F-09b5d9827724 then receives, in order, the message, captured identifier, constants 0, 3, and 0, current status, and captured selector. Both stores are skipped when the selectors differ. They are not conditional on whether either cell was used by the selected operation.

These stores are potentially observable: earlier calls may have received either cell’s address, and their retention behavior is unknown. They are not merely descriptions of unused temporary values.

After either selector-comparison outcome, F-256e8a77aa59 receives the message and 0; its result becomes the function result. There is no additional selector check between the two stores and the fallback call.

Explicit responses do not inherently suppress fallback. Suppression depends on the selector observed after all preceding operation-specific calls. A represented null-message completion returns the provisional result, but does not make null entry valid.

### Operations 1, 3, 4, and 6

These selectors share setup even when their payload lengths are unsuitable. F-be45cd43f92d receives 0 and supplies a handle stored in a 32-bit slot. F-b7af80e829ec supplies another object, whose D-128 returns the captured current selection. The first object’s D-484 returns a value treated as an unsigned 32-bit count.

Ordinary exit from this group calls F-54033250428c with the first handle slot’s address. Release or ownership transfer is not established.

| Selector | Required length | Operation |
|---|---:|---|
| 1 | 0 | F-b9d9c0b9bfab receives a temporary destination and the selection. A selector-2 response uses that destination and length 4. Status remains 4. |
| 3 | 5 | F-c25a2fcc75b5 receives the payload. Its unsigned result must be below the captured count. A valid result sets status 0; if different from the captured selection, F-3867c434fcca receives context, result, payload byte 4, and initial classification, and supplies the provisional return value. |
| 4 | 0 | F-b9d9c0b9bfab receives a temporary destination and the count. A selector-5 response uses that destination and length 4. Status remains 4. |
| 6 | 4 | Executes the indexed behavior below and ultimately sets status 2. |

For operation 3, byte 4 is directly read only when the index is valid and differs from the selection, after F-c25a2fcc75b5 returns. Invalid indices and length mismatches retain status 4.

The four-byte response lengths for operations 1 and 4 do not establish how many bytes F-b9d9c0b9bfab writes.

For operation 6, F-c25a2fcc75b5 supplies an unsigned index. An index below the count causes D-488 to receive an output-handle cell’s address, the first object, and the index. F-44362605ab79 then receives that cell’s address.

A zero result skips response construction. A nonzero result triggers F-fab3799b3190 with the original size-related value, followed by F-1cb585febd84 with the nonzero result. The first query result plus one is compared unsigned, at 32-bit width, against the original size. If it is no greater, the query is repeated with the same argument, and the current size becomes the second result plus one. Otherwise, the current size remains the original.

U-02 receives F-fab3799b3190’s result, F-44362605ab79’s nonzero result, and the full-width current size. F-5ca71e377766 then receives context, the former result, and the size’s low 16 bits. A selector-7 response uses the same result as payload and the same low-16-bit length, followed by F-58ba61416cc9 with that result.

F-289ec4311c64 subsequently receives the output-handle cell’s address, including when F-44362605ab79 returned zero. This call is skipped for an invalid index. The shared-group exit call still occurs.

Length 4 sets status 2 regardless of index validity or response construction; other lengths retain status 4. On selector-equal completion, the output-handle cell is subsequently overwritten with 255 before fallback.

F-fab3799b3190 always receives the original size, not the revised size. No capacity guarantee is established. The first increment uses unsigned 32-bit arithmetic and can wrap. The second is represented as signed addition before unsigned assignment; signed-overflow behavior is unresolved. No second bounds check follows the repeated query.

### Other delegated operations

Length mismatches skip these operation-specific calls and retain status 4.

| Selector | Required length | Calls and result handling |
|---|---:|---|
| 8 | 4 | F-c25a2fcc75b5 receives the payload. F-2b39d0f1207b receives **context, initial classification, helper result**, in that order. Its result becomes provisional; zero sets status 0, otherwise status remains 4. |
| 12 | 1 | F-a8a62ade417f receives context and message; its result becomes status. |
| 14 | Greater than 1 | F-6b47dda9a4e5 receives context and message; its result becomes status. |
| 17 | 4 | F-c25a2fcc75b5 receives the payload. F-e25e25f723f0 receives context and its result; the latter result becomes status. |
| 22 | 0 | F-02cd09de214f receives context and message; its result becomes status. |
| 30 | 2 | F-d9032d50340b receives context, payload byte 0, payload byte 1, and initial classification; its result becomes status. |

The relative evaluation order of operation 30’s two direct byte reads is not established.

### Operation 10: snapshot and selective clearing

Length 0 selects a 32-bit state word at the state base plus 12 plus four times the initial classification.

The function snapshots the word, passes the snapshot and a temporary destination to F-b9d9c0b9bfab, and requests a selector-11 response of length 4. It then rereads the same selected address and stores the reread value AND the complement of the snapshot.

Consequently, bits newly present in the reread but absent from the snapshot survive. Status remains 4.

Negative classifications are not rejected. No address bounds check is visible; valid storage and valid represented address arithmetic remain preconditions.

### Operation 15: query and conditional non-returning behavior

Length 0 initializes the first 32-bit query-output cell to zero, then the second to zero. F-b7af80e829ec supplies an object before F-d266e8cfbbe8 receives context and both cell addresses.

A nonzero query result invokes F-a817d8d58c10. Otherwise, the second output’s low byte is assigned to response byte 0.

A nonzero assigned byte causes calls to the acquired object’s D-224 and D-248, in that order. Its D-232 supplies another object, whose D-160 is called before F-feee9156a257 receives that object. No normal continuation is established.

A zero assigned byte causes three F-b9d9c0b9bfab calls with 0xffffffff and destination offsets 1, 5, and 9 relative to the response start, in that order. Status becomes 0, and a selector-16 response uses length 13.

The helper calls do not establish twelve initialized trailing bytes, three encoded integers, or preservation of byte 0. Only the earlier direct byte assignment is established.

The first query output’s **returned value is never subsequently read**. Its storage nevertheless has a later visible use: selector-equal common completion overwrites it with 255, then overwrites the second output cell with zero, before fallback. The second output’s low-byte observation and subsequent clearing are distinct events.

### Operation 18: processing and reclassification

For length 7, F-0c335b11b4ca receives context and message, and its result becomes status. F-5f5f14cedda3 then classifies the message again.

A nonzero second classification proceeds to common completion. Zero with a nonnull message causes F-09ba55c206ac to receive the message, followed by F-feee9156a257 without an explicit argument and without established return.

The represented zero-classification/null-message branch returns the provisional result directly; it does not validate null entry.

### Operation 20: object response

For length 0, F-b7af80e829ec supplies an object, whose D-236 supplies a value. F-b9d9c0b9bfab receives a temporary destination and that value. A selector-21 response uses the destination and length 4. Status remains 4; the helper’s write extent is unknown.

### Operations 24 and 31: object-based requests

Both require length 10. F-b7af80e829ec supplies an object, and its D-288 is called with that object. A nonzero result skips further operation-specific work and retains status 4.

For a zero result, payload-relative helper calls occur in this order:

| Selector | Ordered calls |
|---|---|
| 24 | F-c25a2fcc75b5 at offset 0; F-64e2dfdd4a5d at offset 4; F-c25a2fcc75b5 at offset 6 |
| 31 | F-c25a2fcc75b5 at offset 0; F-64e2dfdd4a5d at offsets 4, 6, and 8 |

Offsets do not establish helper read extents. Results occupy 32-bit argument positions; no 16-bit truncation of F-64e2dfdd4a5d’s results is shown.

D-244 receives the original object and first helper result. Its returned object is dereferenced without a visible null check. D-12 supplies a 64-bit result, separated into low and high 32-bit arguments.

Operation 24 passes context, message, those halves, and the remaining two helper results to F-043b2a28b7b3. A zero result with nonnull message causes F-09ba55c206ac(message), followed by F-feee9156a257 without an explicit argument. Other represented post-request paths call F-feee9156a257 with the returned object.

Operation 31 passes the analogous arguments and three remaining helper results to F-b6884dd99886, ignores its result, and calls F-feee9156a257 with the returned object. None of these post-request paths has established normal continuation.

### Operations 26 and 28: byte-length responses

Both require length 0 and initialize a one-byte output cell to zero. Operation 26 calls F-0691460c14f4; operation 28 calls F-6bf63f467c6d. Each receives context and the output address, with its result becoming status.

A nonzero result skips explicit response construction. Zero constructs a selector-27 or selector-29 response respectively. Its length is the output byte, ranging from 0 through 255; its payload address is immediately after that byte.

The indicated payload area is not visibly initialized or bounds-checked by this function.

### Remaining selectors

All selectors lacking operation-specific behavior, including unlisted signed 16-bit values, proceed to common completion with status 4.

## Side effects

Established effects include initialization-byte storage, operation-10 state clearing, temporary-cell assignments, responses, and delegated object operations.

Common completion adds two conditional 32-bit writes: 255 to the shared handle/first-output cell, then zero to the second-output cell. Because dependencies previously receive those addresses on some paths, their possible observation cannot be dismissed without a retention contract.

Ownership remains unresolved:

- Handle-address calls do not establish release or transfer.
- F-58ba61416cc9 does not independently establish deallocation.
- Response calls using temporary storage require a consumption or lifetime contract that is unavailable.
- Non-returning F-feee9156a257 calls cannot be treated as ordinary returning cleanup.

Unknown dependencies may change messages, globals, objects, or address-passed storage. Repeated selector reads, state reads, and size queries remain separate observations.

## Dependencies

Mapped direct dependencies are identified only by opaque F identifiers. U-01 and U-02 are document-local opaque IDs for the two unmapped direct dependencies.

D-12, D-128, D-160, D-224, D-232, D-236, D-244, D-248, D-288, D-484, and D-488 identify indirect dependencies by represented table byte offset. Their names imply no domain semantics. Explicit receiver arguments are stated where shown; additional ABI details are unresolved.

F-b9d9c0b9bfab, F-c25a2fcc75b5, and F-64e2dfdd4a5d have no supplied memory-access contracts. Argument spacing and response lengths do not establish byte order, access width, validation, or complete initialization.

## Acceptance examples

These are conditional static scenarios, not executed tests.

1. **Rejection:** Classification 3 invokes F-a817d8d58c10 before initialization and dispatch.

2. **Negative classification:** Classification −1 passes the upper-bound check. Operation 10 selects the word four bytes before the classification-0 word, subject to valid storage and arithmetic.

3. **Already selected:** Operation 3, length 5, with an index below count and equal to selection skips F-3867c434fcca and sets status 0.

4. **Invalid indexed request:** Operation 6, length 4, with index equal to count skips D-488 and F-289ec4311c64, invokes shared-group exit, and sets status 2.

5. **Different size arguments:** With original size 100 and query results 9 then 19, F-fab3799b3190 receives 100, while U-02 receives size 20. Allocation and copying remain unproven.

6. **Selective clearing:** Snapshot 0x00000005 and reread 0x0000000d produce stored value 0x00000008.

7. **Output-cell overwrite:** Successful operation 15 with zero second-output low byte constructs its selector-16 response. If the completion selector still matches, the first output cell is then assigned 255 and the second zero before F-09b5d9827724. The first query result need not be read for this overwrite to be observable.

8. **Selector mismatch:** If any operation reaches completion with a different selector, both completion stores and F-09b5d9827724 are skipped. F-256e8a77aa59 still runs.

9. **Return precedence:** Operation 8 passes context, classification, and decoded result to F-2b39d0f1207b. A zero return sets status 0, but ordinary completion returns F-256e8a77aa59’s result.

10. **Empty byte-length response:** Successful operation 26 with output byte zero requests a selector-27 response of length zero; no nonzero payload extent is established.

## Uncertainties

The following remain **documented_only**:

- Protocol meanings of selectors, statuses, and provisional return 62.
- Helper access extents, encoding, byte order, and initialization guarantees.
- Validity and capacity of temporary response areas.
- Allocation, copying, failure handling, release, and ownership contracts.
- Retention of payload, handle-cell, and query-output addresses, including observation of completion overwrites.
- Dependency preservation of messages, globals, and temporary storage.
- Operation-6 second-query stability and signed-overflow behavior.
- Negative-classification address validity and precise machine arithmetic effects.
- Indirect-call ABI details and unmapped dependency behavior.
- Accuracy of non-return annotations, concurrency, and reentrancy guarantees.

Dispatch and response terminology expresses inferred intent only. Allocation-like, copy-like, encoding-like, and cleanup-like interpretations do not strengthen the established contract.

## Evidence

Established static evidence is the supplied function representation and opaque dependency mapping: represented layouts and widths, signed and unsigned comparisons, branch conditions, argument order, repeated observations, stores, and return precedence.

The common-completion branch explicitly establishes the ordered 32-bit stores of 255 and zero before fallback. Earlier address-passing establishes their potential observability, but not actual retention or observation by a dependency.

Dependency bodies and independent contracts were not supplied. Recovered types, annotations, comments, and prior descriptions are not runtime proof.

No runtime observations or tests have been performed.
