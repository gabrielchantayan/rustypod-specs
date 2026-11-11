<!-- Trusted provenance: F-dc5eae7aff5d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc5eae7aff5d

## Purpose

F-dc5eae7aff5d evaluates a request against selector-indexed shared storage, conditionally invokes an operation, emits a response through a dependency, and performs status-dependent follow-up work.

Three values have separate roles:

- **Response status:** a 32-bit word initialized to 4, changed to 7 on admission, and potentially changed to 0 or 17 after a supported operation.
- **Operation result:** a signed 32-bit result from a supported operation, or 11 when mode 2 finds no handle.
- **Function return:** a signed 32-bit candidate initialized to 62, changed to 87 on admission, and potentially replaced by an operation, query, or nonzero-status follow-up.

The response status does not directly determine the return value. A response prepared with status 0 can be followed by a query error that becomes the function return. Status 17 can accompany an operation result of zero.

This contract describes established static behavior. “Request,” “record,” “response,” and “context” are descriptive roles, not established protocol definitions.

## Interface

### Arguments and return

| Item | Representation | Direct use |
|---|---|---|
| Context token | Opaque 32-bit value | Forwarded to dependencies; not directly dereferenced. |
| Request | Pointer to storage accessed as 32-bit words and a 16-bit field | Read directly and passed to dependencies. |
| Return | Signed 32-bit value | Selected from constants and dependency results. |

Direct request accesses are:

| Byte offset | Width | Capture and use |
|---|---:|---|
| 0 | 32 bits | Captured before selector resolution; reused for either response call. |
| 8 | 32 bits | Captured only after admission; supplied to the selected operation. |
| 12 | Unsigned 16 bits | Captured before selector resolution; used for admission and operation arguments. |

Mode 1 receives the full captured 16-bit value. Mode 2 receives its low eight bits.

Dependencies receiving the request pointer may access additional storage. Their ownership and mutation behavior are not established.

### Shared-storage addressing

F-5f5f14cedda3 supplies an unsigned 32-bit selector.

A shared record-table-base global is read **twice separately**, immediately after selector resolution:

1. The first read captures a **handle base**, retained for subsequent mode-2 and query handle reads.
2. The second read supplies the base used to calculate a **selected record address**, at a 36-byte selector stride. That address is retained for admission fields, operation-mode checks, and eventual mode restoration.

No consistency guarantee makes these reads equivalent. Even though they access the same global, the handle base and selected record address need not refer to the same table instance.

Relevant offsets within the 36-byte-stride layout are:

| Offset | Width | Consumer |
|---|---:|---|
| 0 | 32 bits | Handle read using the first captured base plus the selector stride. |
| 4 | 8 bits | Admission state, captured through the selected record address. |
| 24 | Unsigned 16 bits | Upper admission bound through the selected record address. |
| 28 | 8 bits | Mode checks, later mode capture, and conditional restoration through the selected record address. |
| 33 | Unsigned 8 bits | Lower admission bound through the selected record address. |

A separate shared base supports a 28-byte selector stride. Its indexed addresses are supplied to paired dependencies. That base is read when each such address is calculated; the mode-2 and later query calculations are not one retained base snapshot.

Other accesses use freshly read shared bases: a byte at offset 64 for the status override, an unsigned 32-bit word at offset 52 from another base, and a 32-bit mask obtained through a shared pointer.

These are partial layouts. Byte order, allocation sizes, valid selector limits, and complete object representations are **documented_only**.

### Preconditions and arithmetic

No direct null check, request-length check, selector-range check, or checked-address calculation is established. Normal execution requires accessible request fields and valid indexed storage for every path taken.

The represented selector arithmetic and addresses are 32-bit. A wider intermediate, overflow rejection, or safe fallback is not established. Mathematical indexing without wrap requires the stride products and address additions to fit the applicable address range; behavior outside valid addressed storage is **documented_only**.

## Behavior

### Initial reads and admission

The function initializes response status to 4, captures request offsets 0 and 12, and establishes return candidate 62 before resolving the selector.

Changes made by F-5f5f14cedda3 to those request fields do not replace the captures. Request offset 8 has not yet been read.

After selector resolution, the function performs the two record-table-base reads described above, in order: first the retained handle base, then an independent base read for calculating the retained selected record address. It then captures the admission-state byte at selected-record offset 4.

Admission reads proceed as follows:

- The unsigned lower bound at offset 33 is compared against the captured request value.
- If the request value is below that bound, admission fails without reading the upper bound.
- Otherwise, the upper bound at offset 24 is read for an equality comparison.
- The upper bound is read again for a separate less-than-or-equal comparison.
- If the second comparison succeeds, admission depends on the captured state being 5.
- If the second comparison fails, the result of the earlier equality comparison determines admission.

With stable bound storage, admission therefore requires the captured state to equal 5 and the request value to lie within both inclusive bounds. Stable lower bounds greater than upper bounds admit nothing.

With changing upper-bound storage, the simpler range rule is insufficient. An equality on the first upper-bound read followed by a failed second comparison can admit a request even when the captured state is not 5. No atomic snapshot or concurrency guarantee is established.

### Admitted preparation and mode selection

Admission captures request offset 8, changes response status to 7, and establishes return candidate 87.

F-704f99fd5e22 then receives the context token, original selector, addresses of two 32-bit local words initialized respectively to all-one bits and 23, and a final zero argument.

Mode reads occur after this call and use the retained selected record address from the **second** initial base read. They do not use the retained handle base and do not refresh the table-base global.

Mode 1 is checked first. If that comparison fails, the mode byte is read again for the mode-2 comparison. These checks are not an immutable mode snapshot. The admission-state byte is not recaptured.

### Mode 1

For a mode-1 selection, the established call order is:

1. F-unmapped-01 supplies a context value.
2. F-1bc5bcdc9cca receives that value advanced by four bytes and the selector.
3. F-bd3e4559b3e7 receives a local auxiliary object.
4. F-c35ba8ef24f5 receives that object and an address for a saved 32-bit value.
5. F-3ad2b53f374e receives the object and 12.
6. A fresh F-unmapped-01 call supplies another context value.
7. F-36736f048548 receives that value, the selector, the result of F-1bc5bcdc9cca, the captured request payload, and the full captured 16-bit request value.
8. F-3ad2b53f374e receives the object and saved value.
9. F-3e5f2a992ae5 receives the object.

F-36736f048548 supplies the operation result and return candidate. The subsequent auxiliary calls do not explicitly replace that candidate.

Temporary state installation and restoration are inferred intent. The auxiliary object’s representation, ownership, resource effects, and abnormal-exit requirements remain **documented_only**.

### Mode 2

For a mode-2 selection, the function calculates the original selector’s address in the 28-byte-stride area and calls F-unmapped-02. No explicit argument for this occurrence is established by the supplied representation.

Only afterward does it read the 32-bit handle at the original selector’s 36-byte stride from the **first captured handle base**:

- A zero handle establishes operation result and return candidate 11.
- A nonzero handle is supplied to F-163acd2d3f97 with the captured payload and the low eight bits of the captured request value. Its signed 32-bit result becomes the operation result and return candidate.

F-unmapped-03 then receives the calculated 28-byte-stride address.

Thus mode selection may come from one table instance while the handle comes from another. Replacing the table-base global after the initial captures does not redirect either retained address calculation, although dependencies may still mutate or invalidate the addressed storage.

The paired calls suggest synchronization; exclusion and handle lifetime are not established.

### Operation status and unsupported modes

If neither mode check selects an operation, response status remains 7 and return candidate remains 87.

After either supported operation, a zero operation result changes response status to 0; a nonzero result leaves it at 7.

Following a zero result, status becomes 17 only when these ordered conditions all succeed:

- The original selector equals 1.
- A byte at offset 64 from a freshly read record-table base equals 2.
- An unsigned 32-bit word at offset 52 from another freshly read shared base exceeds 15.

These accesses are distinct from both initial base captures. The override does not change the operation result or return candidate.

### Response and request finalization

Every admitted request calls F-dece790ddc92 with five explicitly represented arguments: request pointer, captured offset-0 value, zero, zero, and 25.

Immediately before that call, adjacent outgoing storage contains the address of the 32-bit response status followed by the 32-bit value 1.

Every rejected request instead calls F-09b5d9827724 with request pointer, captured offset-0 value, zero, zero, and 2. Its adjacent outgoing storage contains, in order:

- The response status’s low eight bits represented in a 32-bit slot.
- The 32-bit value 24.
- The 32-bit value zero.
- The 32-bit value 255.

The staging is established. Whether these slots are additional arguments, descriptors, or encoded fields is **documented_only**.

After either response dependency returns normally, F-256e8a77aa59 receives the request pointer and zero. No later direct request-field read occurs.

The function then tests only the low eight bits of response status. The admitted response exposes its address; no preservation guarantee prevents a dependency from changing it before this test.

### Zero-low-byte follow-up

When the status low byte is zero, F-2c4fc5b4a966 first receives the context token, original selector, and 6.

A fresh F-unmapped-01 result, advanced by four bytes, is supplied with the original selector to F-bc2b33235f5b. Bits 4–7 of its unsigned result select a candidate between 0 and 15.

Additional query work occurs only if this candidate is 1 or 2 and differs from the original selector:

- A local 32-bit query-output word is initialized to zero.
- A new indexed address is calculated from the then-current 28-byte-stride base and original selector.
- F-unmapped-02 receives that address.
- The handle is read using the **first initial handle-base capture**, not the selected record address or a fresh record-table-base read.
- A nonzero handle is passed with the output address to F-045ce4bae945. Its signed 32-bit result replaces the function return candidate.
- Only a zero query result causes another F-unmapped-01 call followed by masking of the query-output word using the 32-bit value obtained through the then-current shared mask pointer.
- F-unmapped-03 receives the same indexed address used for query entry.
- The query-output word is tested for nonzero.

A zero handle skips the query and preserves the prior return candidate.

A query error does not itself prohibit the nonzero-output branch. The relevant output is its value after F-unmapped-03, not necessarily its value at query return. Retention of the exposed output address is not excluded. On query success, the accessor preceding the mask may likewise affect the value being masked.

If the post-exit output test is nonzero, the function obtains a fresh context and calls F-13370c09884a with that context, original selector, and the output value read for this argument. It then obtains another context and calls F-5fda5988215d with that context, original selector, and candidate selector.

The first accessor intervenes between the nonzero test and the output argument read. That argument need not remain nonzero. Once the branch is entered, no second output test guards F-5fda5988215d.

After the optional query work, the zero-low-byte path always obtains another context and calls F-d5cded701dac with it and the original selector. If its result equals 6, the function captures the mode byte from the retained selected record address derived from the **second initial base read**. Otherwise, the restoration value is zero.

F-ca1c4e2f14bb receives the context token, selector, and zero. A nonzero captured mode is then written back to offset 28 of that same retained record address. A captured zero causes no write. These final calls do not explicitly replace the return candidate.

### Nonzero-low-byte follow-up

When the status low byte is nonzero, F-b816af9a7d2b runs only if the originally captured admission state equals 5. It receives the context token and original selector, and its result replaces the function return.

Otherwise, the current return candidate is retained. This decision uses the captured admission state, not a later read and not an assumption that admission necessarily implied state 5.

## Side effects

Direct effects include request and shared-storage reads, initialization of local words, exposure of writable local addresses, and conditional restoration of one shared mode byte.

No direct allocation, release, rollback, or exception handler is established.

In particular:

- Passing the request to F-256e8a77aa59 does not establish release or ownership transfer.
- Auxiliary-object calls do not establish construction or destruction semantics.
- Paired calls do not establish locking.
- Receiving a local address does not establish that a dependency stops using it upon return.
- Repeated context lookups and shared-base reads need not return identical values.
- Capturing an address preserves the address calculation, not the contents, lifetime, or validity of its storage.

Sequencing assumes normal dependency returns and continued validity of subsequent accesses. Recovery from traps, nonreturn, invalidation, or retained pointers outliving local storage is **documented_only**.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-5f5f14cedda3 | Supplies selector from request. |
| F-704f99fd5e22 | Receives admitted parameters and two writable local words. |
| F-1bc5bcdc9cca | Supplies an argument for mode 1. |
| F-bd3e4559b3e7 | Initial auxiliary-object call. |
| F-c35ba8ef24f5 | Receives auxiliary object and saved-value destination. |
| F-3ad2b53f374e | Receives auxiliary object with 12, then saved value. |
| F-36736f048548 | Supplies mode-1 result. |
| F-3e5f2a992ae5 | Final auxiliary-object call. |
| F-163acd2d3f97 | Supplies mode-2 result for a nonzero handle. |
| F-dece790ddc92 | Admitted response. |
| F-09b5d9827724 | Rejected response. |
| F-256e8a77aa59 | Post-response request call. |
| F-2c4fc5b4a966 | Initial zero-low-byte follow-up. |
| F-bc2b33235f5b | Supplies candidate-selector bits. |
| F-045ce4bae945 | Queries handle through writable 32-bit output. |
| F-13370c09884a | Receives later output value after branch entry. |
| F-5fda5988215d | Receives original and candidate selectors. |
| F-d5cded701dac | Governs mode capture by comparison with 6. |
| F-ca1c4e2f14bb | Final zero-low-byte call before optional restoration. |
| F-b816af9a7d2b | Replaces eligible nonzero-low-byte return. |
| F-unmapped-01 | Repeated context accessor. |
| F-unmapped-02 | Paired entry; represented arguments differ by occurrence. |
| F-unmapped-03 | Paired exit receiving indexed address. |

The unmapped IDs are document-local opaque identifiers.

## Acceptance examples

These are conditional static consequences, not executed tests. Unless specified otherwise, assume valid storage, normal returns, stable field reads, and no dependency-induced status mutation.

1. **Distinct initial bases.** The first base read identifies table A; the second identifies table B. Admission and mode selection use B. Mode-2 and later query handles use A. Conditional mode capture and restoration use B.

2. **Later global replacement.** After both initial captures, the table-base global changes to table C. Retained handle and selected-record accesses remain directed through their captures. The status-override byte check uses the then-current global instead.

3. **Inclusive bounds.** State 5 and bounds 3 through 20 admit values 3 and 20, but reject 2 and 21.

4. **Unstable upper bounds.** A value of 10, lower bound 3, state 4, first upper read 10, and second upper read 9 admits the request. If mode is unsupported and status stays 7, F-b816af9a7d2b is skipped because captured state is 4; return remains 87.

5. **Stable state rejection.** An in-range value with state 4 takes the rejected response and request finalization, skips nonzero-status follow-up, and returns 62.

6. **Bounds rejection with state 5.** An out-of-range value takes the rejected response and finalization, then returns F-b816af9a7d2b’s result.

7. **Mode-2 narrowing and missing handle.** An admitted value of 4660 is passed as 52 to F-163acd2d3f97. If the captured-base handle is instead zero, that operation is skipped, result 11 is established, and paired exit still occurs.

8. **Status override.** A zero operation result, original selector 1, checked byte 2, and checked unsigned word 16 establish status 17. Unchanged status selects nonzero-low-byte follow-up.

9. **Query error with output.** An eligible query error replaces the return candidate and skips masking. If output is nonzero after paired exit, both downstream branch calls occur; nonzero output merely at query return is insufficient.

10. **Output mutation after branch entry.** If the post-exit test sees 9 but the next accessor changes the exposed output to zero, F-13370c09884a receives zero. F-5fda5988215d still follows.

11. **Restoration destination.** If F-d5cded701dac returns 6 and the retained selected record’s mode is 2, that same record receives 2 after F-ca1c4e2f14bb, regardless of later table-base replacement. Captured zero causes no write.

12. **Low-byte decision.** Status 256 selects zero-low-byte follow-up despite the full word being nonzero.

## Uncertainties

The following remain **documented_only**:

- Complete dependency signatures and response-storage interpretation.
- Protocol meanings of statuses, constants, and return codes.
- Complete layouts, byte order, selector limits, and overflow outcomes.
- Consistency between separate reads of the same shared global or field.
- Ownership, lifetime, allocation, retention, reentrancy, and synchronization guarantees.
- Auxiliary-object semantics and exceptional cleanup.
- Dependency mutation of globals, pointed-to storage, and exposed locals.
- Behavior under invalid storage, concurrent mutation, or abnormal termination.

Neither the common origin of the initial bases nor their adjacent reads establishes that they identify the same storage.

## Evidence

Established evidence consists of the supplied static representation: access widths and offsets, separate ordered global reads, retained address consumers, constants, comparisons, strides, call ordering, exposed addresses, and explicit status and return assignments.

Response construction, context access, temporary-state handling, and synchronization-like pairing are inferred intent only. They add no preservation or ownership guarantees.

No runtime tests have been performed. Acceptance examples are conditional consequences of static control and data flow, not observed executions.
