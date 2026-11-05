<!-- Trusted provenance: F-1a28d27dcc4e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1a28d27dcc4e

## Purpose

F-1a28d27dcc4e obtains a constructed 32-bit value, looks up a writable record associated with an enclosing object, initializes selected record fields, and submits that record to another dependency. A particular submission outcome triggers an additional state write and a further operation. A final dependency is called before every ordinary return from the record-present path.

This is a behavioral contract for the supplied static representation, not a claim about runtime observations. The business meaning of the enclosing object, record, constructed value, and status codes is unknown.

## Interface

### Inputs

The function has two explicit inputs:

- **Object address:** a value represented as a 32-bit integer and used to form addresses within an enclosing object.
- **Construction argument:** an uninterpreted 32-bit value passed to F-438c1b0292b6.

The object address is used to derive four addresses:

| Offset from object address | Established use |
|---|---|
| 0x3C | First argument to record lookup and submission; stored into the returned record; also passed to later dependencies |
| 0x58 | First argument to the conditional follow-on operation |
| 0x74 | Second argument to record lookup |
| 0xD8 | Second argument to record submission |

These are address calculations, not direct reads of object contents by F-1a28d27dcc4e. The dependencies may access the corresponding storage, but their access patterns are not supplied.

### Output

The function returns a 32-bit result on an ordinary return:

- **20**, equivalently 0x14, if record lookup returns a null pointer.
- The **low 32 bits** of the submission dependency’s 64-bit result, unless the conditional follow-on operation is invoked.
- The follow-on operation’s **32-bit result**, if that operation is invoked.

A zero result from the construction dependency instead invokes F-a817d8d58c10, which is marked non-returning in the supplied representation. No ordinary result is established for that path.

The meaning and signedness of nonzero result codes are not established beyond their displayed 32-bit representation. In particular, 20 must not be assigned a named error meaning without further evidence.

### Preconditions and memory requirements

For an execution to complete normally, all invoked dependencies must accept their supplied arguments and return where required.

If lookup returns a non-null record, the direct accesses in this function require writable storage at these record-relative locations:

| Offset | Width | Direct use |
|---|---:|---|
| 0 | 1 byte | Written with 1 |
| 1 | 1 byte | Written with 0; conditionally read later |
| 4 | 4 bytes | Written with the object-derived address at offset 0x3C |
| 8 | 1 byte | Conditionally written with 1 |
| 12 | 4 bytes | Written with the constructed value |

The record must therefore support accesses through offset 15. This does not establish its full size or layout.

The displayed address model is 32-bit. The input must permit the required address calculations and dependency accesses to be meaningful. The supplied representation does not establish whether overflowing signed address arithmetic has defined target behavior; portable signed arithmetic assumptions and modular wrapping are not interchangeable here.

## Behavior

### Construction precedes lookup

F-8157a180d901 is invoked first with the numeric argument 12. Its 32-bit result is then supplied as the first argument to F-438c1b0292b6. The second argument to that dependency is the function’s construction argument.

There is no direct null or failure check on the result of F-8157a180d901 before it is passed onward.

The result from F-438c1b0292b6 is tested against zero:

- If zero, F-a817d8d58c10 is invoked. The supplied representation identifies this call as non-returning.
- If nonzero, execution proceeds to record lookup.

Thus, successful construction occurs before the function learns whether a record is available. A later lookup failure does not cause any explicit disposal call in this function.

### Record lookup and the null path

F-a37cf1fc8c44 is called with the object-derived addresses at offsets 0x3C and 0x74, in that argument order.

If it returns null, F-1a28d27dcc4e returns 20 immediately. On this path:

- No direct record field access occurs.
- No submission occurs.
- Neither the conditional follow-on operation nor the final dependency is called.
- No explicit local release of the previously constructed value occurs.

These statements do not establish the side effects or ownership behavior of the dependencies already invoked.

### Record-present initialization

If lookup returns a non-null record, the unresolved dependency identified below as U-01 is invoked before any of the function’s direct record writes. Its result is not used.

The function then performs these direct writes in the displayed order:

1. The nonzero 32-bit constructed value is stored at record offset 12.
2. The byte at record offset 1 is set to 0.
3. The byte at record offset 0 is set to 1.
4. The 32-bit object-derived address at offset 0x3C is stored at record offset 4.

The function does not directly initialize the byte at offset 8 during this sequence. It also does not directly initialize bytes 2–3 or 9–11.

This is an order of accesses in the supplied representation. It is not an assertion of atomic publication, synchronization, or visibility guarantees between threads.

### Submission and conditional follow-on behavior

F-2db98b0446c5 is called with three arguments, in this order:

1. The object-derived address at offset 0x3C.
2. The object-derived address at offset 0xD8.
3. The record pointer returned by lookup.

Its result is represented as 64 bits. The low 32 bits become the provisional return result.

The next decision depends on that low half:

- If the low half is nonzero, the function does not directly read record offset 1. It skips the conditional write and follow-on operation.
- If the low half is zero, the function reads the byte at record offset 1 after submission has returned.

The follow-on operation occurs only when the low half is zero **and** the byte read from offset 1 is exactly 1.

When both conditions hold:

1. Record offset 8 is written with the byte value 1.
2. U-02 is called with the object-derived addresses at offsets 0x58 and 0x3C, in that order.
3. U-02’s 32-bit result replaces the provisional return result.

No other byte value at offset 1 activates this branch. In particular, values 2 through 255 are not treated as equivalent to 1.

Although the supplied representation extracts the high 32 bits of the submission result, those bits do not determine a visible branch or ordinary return value in this function. For the only branch where the companion comparison matters, that comparison uses the post-submission record byte instead.

### Final call and return

On every normally completing record-present path, U-03 is invoked with the object-derived address at offset 0x3C:

- After submission when the follow-on operation is skipped.
- After U-02 returns when the follow-on operation is taken.

U-03’s result is not used to select the return value. The function then returns the previously selected 32-bit result.

U-03 is not invoked on the null-lookup path or before the non-returning construction-failure call. No finalization guarantee is established if an earlier dependency fails to return normally.

## Side effects

### Direct memory changes

The function directly changes only the listed record locations:

- Offset 12 receives the constructed 32-bit value.
- Offset 1 receives 0 before submission.
- Offset 0 receives 1 before submission.
- Offset 4 receives the object-derived address at offset 0x3C.
- Offset 8 receives 1 only on the conditional follow-on path.

The pre-submission writes are not rolled back when submission returns a nonzero low half. Likewise, the offset-8 write is not rolled back if U-02 returns a nonzero result.

These are direct-write guarantees, not guaranteed final record contents. Dependencies called afterward may modify the record or related storage.

### Reads and dependency effects

The only explicit record-content read is the one-byte read at offset 1, and it occurs only after submission returns a result whose low 32 bits are zero.

The function writes that byte to zero before submission. Therefore, in an execution without intervening modification, the follow-on condition is false. A post-submission value of 1 requires some intervening effect; the supplied body does not identify its source.

No assumption is made that any dependency preserves global state, object contents, record contents, or memory reachable through its arguments.

### Ownership and lifetime

The argument 12 to F-8157a180d901 may suggest a storage request, but allocation semantics are not established.

The function passes that dependency’s result into construction and later stores the constructed result into the lookup record. Neither step alone proves an ownership transfer.

There is no explicit, identifiable release operation for the constructed value in the supplied body. This does not prove either a leak or retention: an unknown dependency may own, consume, retain, or release resources.

Similarly, U-01 and U-03 must not be assumed to form a lock/unlock or acquire/release pair merely because of their positions. Their actual resource and synchronization effects remain undocumented.

## Dependencies

Known dependency identifiers come from the supplied mapping. U-01, U-02, and U-03 are document-local opaque identifiers for unresolved dependencies; they are not published function identities.

| ID | Established interaction |
|---|---|
| F-8157a180d901 | Called first with 12; its 32-bit result is forwarded to construction without a local failure check |
| F-438c1b0292b6 | Receives the preceding result and the construction argument; a zero result selects the non-returning path |
| F-a817d8d58c10 | Called when construction returns zero; marked non-returning |
| F-a37cf1fc8c44 | Receives object-derived addresses at offsets 0x3C and 0x74; returns the record pointer tested against null |
| U-01 | Called on the record-present path before direct record initialization; no explicit arguments are shown and its result is unused |
| F-2db98b0446c5 | Receives two object-derived addresses and the record pointer; returns the represented 64-bit submission result |
| U-02 | Called only after a zero submission low half and a post-submission record byte of exactly 1; supplies the replacement return result |
| U-03 | Called after normally completing submission processing on the record-present path; receives the object-derived address at offset 0x3C |

For U-01, the absence of displayed arguments does not establish absence of implicit inputs or global-state access.

## Acceptance examples

These are conditional contract examples, not executed tests. Each assumes valid address calculations, valid required storage, and normal return from dependencies except where explicitly stated.

### Construction returns zero

If F-438c1b0292b6 returns zero:

- F-a817d8d58c10 is called.
- Record lookup is not reached under the stated non-returning behavior.
- No ordinary return value is specified.

### Lookup returns null

If construction returns a nonzero value and lookup returns null:

- The function returns 20.
- U-01, submission, U-02, and U-03 are not called.
- No direct record write occurs.
- No explicit local disposal of the constructed value occurs.

### Submission returns a nonzero low half

If lookup succeeds and submission returns a low half of 7:

- The four initialization writes occur before submission.
- The function does not read record offset 1 after submission.
- It does not directly write offset 8 or invoke U-02.
- U-03 is called, and the function returns 7.

This remains true even if submission leaves record offset 1 equal to 1.

### Submission returns zero without an activating record byte

If submission’s low half is zero and the subsequent read of record offset 1 yields 0:

- U-02 is skipped.
- There is no direct write to offset 8.
- U-03 is called.
- The function returns zero.

The same branch outcome applies when the byte is 2 or 255.

### Submission activates the follow-on path

If submission’s low half is zero and the subsequent byte read yields 1:

- Offset 8 is written with 1 before U-02 is called.
- U-02 receives the object-derived addresses at offsets 0x58 and 0x3C.
- If U-02 returns 9, U-03 is then called and the function returns 9.

The direct offset-8 write occurs regardless of whether U-02’s result indicates success according to any external convention.

### Submission’s high half does not activate the branch

If submission’s low half is zero, its high half is 1, and the post-submission byte at offset 1 is 0, U-02 is skipped and the ordinary result is zero after U-03.

Conversely, a high half other than 1 does not prevent U-02 when the low half is zero and the record byte is exactly 1.

## Uncertainties

The following are **documented_only**:

- The enclosing object’s type, full layout, and required initialization.
- The record’s full size, alignment, field meanings, and lifetime.
- Whether 12 denotes an allocation size or another request parameter.
- Whether the constructed value is a pointer, handle, or another 32-bit representation.
- Ownership and release obligations for intermediate and constructed values.
- The business meanings of return codes, including 20.
- The exact contracts of U-01, U-02, and U-03.
- Whether any dependency changes globals, object storage, record storage, or resource ownership.
- The target’s overflow behavior for integer-represented address calculations.
- Concurrency, reentrancy, asynchronous modification, and memory-ordering guarantees.
- Exceptional behavior and cleanup obligations when a dependency does not return normally.
- The concrete representation and broader meaning of the submission dependency’s 64-bit result.

**Inferred intent:** the sequence resembles record preparation followed by submission and a conditionally triggered follow-on operation. That interpretation is not required to establish the access order, branch conditions, or return selection.

## Evidence

Established static evidence consists of the supplied function representation and dependency mapping: explicit call order, argument values, zero checks, byte and 32-bit record accesses, the represented 64-bit submission result, return selection, and the non-returning annotation.

The contract does not infer dependency internals from names or call placement. No runtime tests have been performed. The acceptance examples describe behavior implied by the static representation under their stated conditions; they are not observed execution results.
