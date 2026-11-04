<!-- Trusted provenance: F-dc1e9c59eefe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-dc1e9c59eefe

## Purpose

F-dc1e9c59eefe returns a 32-bit classification derived from a lookup result, several conditional dependency responses, and, on one path, a byte in the object identified by the lookup result.

The lookup uses both a four-byte field from the input record and a value read from a source-referenced global datum. A zero lookup result or a failed qualification check produces the default classification, 30. Otherwise, further decisions select one of 23, 24, 25, 27, 28, or 30.

Every path that passes qualification invokes F-a1a61d2e50f9 with the lookup result and the numeric argument 1 after classification. Paths that fail lookup or qualification do not invoke it.

These are established numerical and call-order behaviors. “Lookup,” “qualification,” and “classification” are descriptive labels, not established domain meanings. In particular, the final invocation is not established to release, retain, finalize, or otherwise manage the object.

## Interface

### Input record

The function accepts one 32-bit argument used as an address. The visible body reads a four-byte field beginning at byte offset `0x38` from that address.

| Offset | Width | Use |
|---|---:|---|
| `0x38` | 4 bytes | First argument to F-682adfd5a561 |

No other input-record field is visibly accessed. The field’s semantic type and meaning are unknown.

A zero input address is not checked before the read. Safe execution requires that the reached address calculation and four-byte access be valid.

### Global lookup input

The second argument to F-682adfd5a561 is the value read from a source-referenced global datum. This read is an established part of evaluating the lookup arguments, not merely a possible implementation detail.

Both the input-field value and the global-datum value are evaluated before F-682adfd5a561 is invoked. Their relative evaluation order is not established by the supplied source.

The global datum is not an additional public argument to F-dc1e9c59eefe. Its concrete value, declared width, representation, mutability, and semantic role are unknown. Calling it a discriminator would describe possible intent, not establish its type or purpose.

### Lookup result and object access

F-682adfd5a561 supplies a result used as a 32-bit value:

- Zero stops processing and produces 30.
- A nonzero value is passed unchanged to the reached object-related dependencies.
- On one conditional path, that same value supplies the base address for a one-byte read.

The only directly established object field is:

| Offset | Width | Use |
|---|---:|---|
| `0x71c` | 1 byte | Zero/nonzero decision between 24 and 23 |

No broader object layout is established. A nonzero result does not by itself prove that this later access is valid.

### Return value

The function returns a four-byte value. Its ordinary-completion results are:

| Value | Selection condition |
|---:|---|
| 30 (`0x1e`) | Lookup returns zero; qualification returns zero; or the unsigned classification response is at least 4 |
| 28 (`0x1c`) | Qualification succeeds and F-69ee34fe0c1f returns nonzero |
| 27 (`0x1b`) | Qualification succeeds, F-69ee34fe0c1f returns zero, and F-0ea2853b9ea1 produces 3 |
| 25 (`0x19`) | The unsigned response is below 3 and F-71a5f3c665ee returns zero |
| 24 (`0x18`) | The unsigned response is below 3, F-71a5f3c665ee returns nonzero, and the object byte is zero |
| 23 (`0x17`) | The same preceding conditions as for 24, but the object byte is nonzero |

The last three rows also require successful lookup, successful qualification, and a zero response from F-69ee34fe0c1f.

There is no visible output parameter or separate error result. In particular, 30 is not exclusively a failure indication: a qualification-success path can also return it.

### Preconditions

For normal execution of the visible behavior:

- The input must permit the four-byte read at offset `0x38`.
- The source-referenced global datum must be readable when evaluated.
- Each reached dependency must accept the arguments supplied.
- If the object-byte path is reached, the original lookup result must still identify readable storage at offset `0x71c` after the preceding dependency calls.
- Address calculations and accesses must be valid in the execution environment.

The body does not visibly validate these conditions or recover from their violation.

## Behavior

### Initial evaluation and lookup

The default result is 30.

The function evaluates two lookup arguments: the four-byte input-record field and the source-referenced global datum. Both evaluations precede the lookup invocation, but no relative ordering between them is established.

F-682adfd5a561 is then invoked with the field value as its first argument and the global-datum value as its second argument. Neither value is visibly reread for later processing.

If the lookup result is zero, the function returns 30. No qualification call, classification call, object-byte read, or final invocation occurs. The input-field and global-datum reads have nevertheless occurred.

### Qualification

For a nonzero lookup result, F-d35171ffbceb receives that result.

A zero response causes the function to return 30 without further dependency calls or an object-byte read. A nonzero response permits the remaining classification work and the final invocation.

Lookup failure and qualification failure therefore have the same return value but different call traces.

### First classification decision

After qualification succeeds, F-69ee34fe0c1f receives the original lookup result.

A nonzero response selects 28. Its magnitude is otherwise irrelevant. This path skips F-0ea2853b9ea1, F-71a5f3c665ee, and the object-byte read, then proceeds to the final invocation.

A zero response continues to the unsigned classification decision.

### Unsigned classification decision

F-0ea2853b9ea1 receives the original lookup result. Its response is used as a 32-bit unsigned value.

The decision partitions the complete unsigned range:

| Response | Effect |
|---|---|
| 0, 1, or 2 | Continue to the secondary decision |
| Exactly 3 | Select 27 |
| 4 through `0xffffffff` | Retain 30 |

This is not a signed comparison. For example, `0xffffffff` belongs to the range that retains 30; it does not qualify as below 3.

For a response of 3 or greater, F-71a5f3c665ee is not invoked and the object byte is not read. The final invocation still occurs because qualification has succeeded.

No classification arithmetic is performed on the response. Only its unsigned ordering relative to 3 and its equality to 3 matter.

### Secondary decision and object byte

For an unsigned response of 0, 1, or 2, the function selects 25 and invokes F-71a5f3c665ee with the original lookup result.

If that dependency returns zero, 25 remains selected. The object byte is not read.

If it returns nonzero, the function reads one byte at offset `0x71c` from the original lookup result:

- Zero selects 24.
- Any nonzero byte selects 23.

Byte signedness does not affect this zero/nonzero decision. Both `0x01` and `0xff`, for example, select 23.

The read occurs after F-71a5f3c665ee returns. The relevant value is therefore the value observed at that later access, not a presumed entry-time value. Any preceding dependency may have affected the storage; the supplied body does not establish preservation by those dependencies.

### Final invocation and return

Every qualification-success path invokes F-a1a61d2e50f9 with the original lookup result and the numeric argument 1.

This invocation occurs after the path’s classification decisions, including the object-byte read when reached. Its return value is not visibly consumed. There is no later visible input-field read, global-datum read, object-byte read, or classification decision.

If the invocation returns normally, the function returns the classification already selected. Its response does not provide a visible mechanism for replacing that classification.

This ordering does not establish that F-a1a61d2e50f9 preserves global state, keeps the object alive, releases it, or returns normally.

## Side effects

### Direct reads and writes

The visible body performs:

1. One four-byte read from the input record at offset `0x38`.
2. One evaluation that reads the source-referenced global datum for the lookup’s second argument.
3. At most one one-byte read from the lookup result at offset `0x71c`.

The first two reads occur before the lookup call, with their relative order unspecified by the supplied source. The width of the global read is not established.

The object-byte read requires all of the following:

- Nonzero lookup result.
- Nonzero qualification response.
- Zero response from F-69ee34fe0c1f.
- Unsigned response below 3 from F-0ea2853b9ea1.
- Nonzero response from F-71a5f3c665ee.

There are no explicit stores to the input record, object, or global storage in the visible body. Internal result selection is not an externally visible memory write.

### Observable ordering

Subject to the stated conditional branches, the established order is:

1. Evaluate the input-field value and global-datum value, without an established order between those two evaluations.
2. Invoke F-682adfd5a561.
3. Invoke F-d35171ffbceb if lookup is nonzero.
4. Invoke F-69ee34fe0c1f if qualification is nonzero.
5. Invoke F-0ea2853b9ea1 if the preceding response is zero.
6. Invoke F-71a5f3c665ee if the unsigned response is below 3.
7. Read the object byte if the secondary response is nonzero.
8. Invoke F-a1a61d2e50f9 on every qualification-success path.
9. Return if execution completes normally.

### Ownership and indirect effects

No allocation, ownership transfer, release, retention, reference-count adjustment, or synchronization is explicitly established by this body.

Dependencies may read or modify memory, change globals, invalidate storage, fail, or not return. Their implementations and contracts are not supplied. The absence of direct stores here does not make the overall operation read-only.

All reached object-related calls receive the same lookup value. There is no visible lookup refresh or replacement object after a dependency executes.

## Dependencies

| Opaque function ID | Supplied arguments | Response use |
|---|---|---|
| F-682adfd5a561 | Four-byte input field; value read from the source-referenced global datum | Zero stops processing; nonzero is reused as the object value |
| F-d35171ffbceb | Original lookup result | Zero stops processing; nonzero permits classification and final invocation |
| F-69ee34fe0c1f | Original lookup result | Nonzero selects 28; zero enables unsigned classification |
| F-0ea2853b9ea1 | Original lookup result | Response is partitioned as an unsigned 32-bit value below 3, equal to 3, or above 3 |
| F-71a5f3c665ee | Original lookup result | Zero retains 25; nonzero enables the byte read |
| F-a1a61d2e50f9 | Original lookup result; numeric argument 1 | Response is not visibly used |

The table describes only the interactions established at this call site. It does not assign internal semantics or ownership obligations to any dependency.

## Acceptance examples

These are conditional deductions from the supplied body, not runtime observations. Each assumes valid reached accesses and normal return from invoked dependencies.

| Conditions | Expected result | Required observable distinctions |
|---|---:|---|
| Lookup returns zero | 30 | Input-field and global-datum reads precede lookup; no later dependency executes |
| Lookup is nonzero; qualification returns zero | 30 | Qualification executes; classification and final invocation do not |
| Qualification succeeds; F-69ee34fe0c1f returns nonzero | 28 | Skip both later classification dependencies and byte read; invoke F-a1a61d2e50f9 |
| Qualification succeeds; first classification response is zero; unsigned response is 3 | 27 | Skip secondary dependency and byte read; execute final invocation |
| Same prefix; unsigned response is 4 | 30 | Retain default; execute final invocation |
| Same prefix; unsigned response is `0x80000000` or `0xffffffff` | 30 | Neither value enters the unsigned below-3 branch |
| Unsigned response is 0, 1, or 2; secondary response is zero | 25 | No object-byte read; execute final invocation |
| Unsigned response is below 3; secondary response is nonzero; byte is zero | 24 | Read byte after secondary dependency and before final invocation |
| Same preceding conditions; byte is `0xff` | 23 | Nonzero byte selects 23 regardless of signed-byte interpretation |

Additional ordering and state examples:

- If the secondary dependency changes the byte from zero to nonzero before returning nonzero, the subsequent read selects 23, provided the storage remains valid.
- If the secondary dependency returns zero, no object-byte access occurs, regardless of the stored byte.
- Both an input-field-first evaluation and a global-datum-first evaluation are consistent with the supplied argument expression. Requiring one fixed order is not justified by this evidence.
- If the lookup changes the global datum internally, its second argument was already evaluated before that invocation. This function does not visibly read the datum again.
- If the final dependency returns a number different from the selected classification, that response does not visibly replace the result.
- Lookup failure and an unsigned classification response of 4 both produce 30, but only the latter path executes the final invocation.

## Uncertainties

**Inferred intent:** The structure suggests object lookup, qualification, classification, and a common post-classification action. No domain meaning is established for those labels or the six result values.

**documented_only — global datum:** Its read is established. Its concrete value, declared width, representation, mutability, semantic role, and any synchronization requirements remain unresolved. The supplied source does not establish its read order relative to the input-field read.

**documented_only — dependency contracts:** Internal effects, failure behavior, termination, global-state preservation, object-lifetime effects, and ownership rules remain unknown for every dependency.

**documented_only — data representation:** Complete record sizes, byte order, alignment requirements, allocation origins, and lifetime rules are not supplied. Only the stated field widths and offsets are established.

**documented_only — address arithmetic:** The accesses use byte offsets `0x38` and `0x71c` in the supplied 32-bit address model. The source does not provide a portable overflow, wraparound, or invalid-address contract. Valid address formation is a precondition, not an error case mapped to 30.

**documented_only — abnormal execution:** There is no visible recovery for an invalid access, dependency failure, or nonreturning call. Exact fault or exception behavior is not established.

**documented_only — concurrency:** No synchronization is visible. Atomicity, concurrent mutation behavior, and validity under data races cannot be inferred.

## Evidence

This contract derives from static inspection of the supplied function body, using the supplied dependency mapping only to identify opaque callees. Prior prose and reviewer feedback are not independent evidence of behavior.

Established static evidence includes:

- The four-byte input-field read at offset `0x38`.
- The source-referenced global-datum read as the lookup’s second argument.
- Evaluation of both lookup arguments before invocation, without an established relative order.
- Conditional dependency ordering and zero/nonzero decisions.
- The unsigned 32-bit comparison against 3.
- The conditional one-byte object read at offset `0x71c`.
- The six return constants.
- The final invocation on every qualification-success path.

No dependency implementations or runtime traces were supplied. No runtime tests have been performed. Acceptance examples describe consequences of the visible control flow, not verified executions.
