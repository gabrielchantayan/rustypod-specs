<!-- Trusted provenance: F-18ed0aca2e0e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-18ed0aca2e0e

## Purpose

F-18ed0aca2e0e conditionally reads two adjacent bytes through a cursor stored in a memory-resident object. It returns a 32-bit unsigned result and writes the selected cursor value back to the object.

Consumption is controlled by a strict unsigned comparison between the cursor-plus-one candidate and a stored boundary. If the comparison succeeds, the function selects a cursor advanced by two and reads the two bytes preceding that selected cursor. If the comparison fails, it reads neither byte, returns zero, and still writes the unchanged cursor.

The inferred purpose is a bounded two-byte reader with the first byte contributing the low eight result bits. This interpretation has two qualifications:

- The actual eligibility condition is an unsigned address comparison, not a validated buffer-length calculation.
- The second byte passes through a plain character interpretation. If that character type is signed, high-bit second bytes can produce nonzero bits above bit 15 of the result.

An initial null check can invoke F-540af0155702. That branch directly reads a diagnostic global datum and supplies its **value** as the third argument. The dependency’s behavior, including whether it returns, is unknown.

## Interface

### Input and object layout

The function accepts one argument represented as a signed 32-bit integer used as an object address. No typed object interface or allocation contract is supplied.

The directly accessed object fields are:

| Offset from object address | Access width | Interpretation |
|---|---:|---|
| `0x20` | 32 bits | Cursor address, read and written through a signed integer representation |
| `0x24` | 32 bits | Boundary, read as an unsigned integer |

The boundary participates in a 32-bit unsigned comparison. The cursor’s signed representation does not make that comparison signed.

The function also directly reads a diagnostic global datum, but only on the failing-validation branch. The supplied representation does not establish that global datum’s access width, type, or contents.

No other object fields are directly accessed. The object’s total size, alignment requirements, allocation origin, and broader invariants are not established.

### Output

Normal completion returns a 32-bit unsigned integer:

- Ineligible consumption returns zero.
- Eligible consumption returns the combination of the two byte contributions described below.

There is no separate success indicator. A zero result can mean either that consumption was ineligible or that eligible consumption read two zero bytes.

### Access requirements

Ordinary execution without invoking the diagnostic dependency requires a nonzero object argument and a nonzero cursor in the initial cursor-field read.

Successful normal completion additionally requires:

- Readable object fields at each actual field access.
- Writable cursor-field storage for the final 32-bit store, including on the nonconsuming path.
- Readable bytes at both selected byte addresses when consumption is eligible.
- Address formation and integer arithmetic valid under the actual execution model.
- On a failing-validation path, an accessible diagnostic global datum and whatever prerequisites F-540af0155702 requires.

These are requirements for execution, not validations performed by the function. Nonzero addresses do not establish memory validity, allocation membership, or ownership.

## Behavior

### Initial validation

If the object argument is nonzero, the function first reads the 32-bit cursor field at offset `0x20` for validation.

If the object argument is zero, this initial cursor-field read is skipped.

Validation fails when either:

- The object argument is zero; or
- The cursor obtained by the initial validation read is zero.

A failing validation invokes F-540af0155702 with three arguments:

1. An assertion-related diagnostic reference.
2. The integer 364.
3. The value read from a diagnostic global datum.

The first and third arguments have distinct roles. The first is a diagnostic reference. The third is the global datum’s value, not a reference to that global datum.

The direct global read occurs on the failing-validation branch before the dependency executes. It is absent from the ordinary non-asserting path. The supplied representation does not establish the global read’s width or an evaluation order among the three argument expressions.

There is no direct cursor write before this dependency call. There is also no direct error return associated with validation failure.

If F-540af0155702 returns, execution continues without repeating the initial validation. In particular:

- A zero object argument is still used for subsequent field-address formation.
- A previously zero cursor is not rejected again.
- Changes made by the dependency can affect the subsequent field reads.

The dependency must not be treated as a proven terminating assertion or a proven state-repair operation.

### Operational field reads

After validation and any returning dependency call, the function reads:

1. The cursor field at offset `0x20`.
2. The boundary field at offset `0x24`.

This cursor read is separate from the initial validation read. Its value controls the subsequent operation.

On the ordinary non-asserting path, the function therefore reads the cursor field twice. On a zero-object validation path, the initial read is skipped, but the operational field accesses are still attempted if the dependency returns.

The operational cursor and boundary values are retained for the remaining computation. The repeated eligibility decisions do not introduce additional boundary-field reads.

### Eligibility and selected cursor

The function forms a 32-bit unsigned candidate from the operational cursor plus one. Consumption is eligible exactly when this candidate is strictly less than the captured unsigned boundary.

Under ordinary, valid, nonwrapping address arithmetic, this tests whether the address of the second byte lies below the boundary. In that restricted interpretation, the boundary acts as an exclusive upper bound.

The function does not independently establish:

- That the cursor itself is below the boundary.
- That two bytes belong to a readable allocation.
- That the cursor and boundary refer to the same allocation.
- That incrementing the cursor is safe.
- That a subtraction-based remaining-length calculation would permit two bytes.

When eligible, the selected outgoing cursor is the operational cursor plus two. Otherwise, it is the unchanged operational cursor.

The supplied representation performs cursor increments using signed integer expressions before relevant unsigned use. An unsigned comparison does not establish wrapping semantics for those preceding signed operations. Behavior at signed overflow boundaries remains unresolved.

### Byte accesses and result construction

When consumption is eligible, the function reads the bytes in this order:

1. The byte at the selected outgoing cursor minus two.
2. The byte at the selected outgoing cursor minus one.

With valid, nonwrapping arithmetic, these are the operational cursor address and the following address.

The first byte is interpreted as an unsigned byte and converted to a 32-bit unsigned value. It contributes a value from 0 through 255.

The second byte is interpreted as a plain character and then converted to a 32-bit unsigned value. That converted value is shifted left by eight bits and combined with the first contribution using bitwise OR.

Consequences depend on the character interpretation:

- With an unsigned 8-bit character interpretation, the result is the ordinary two-byte little-endian value, from 0 through 65535.
- With a signed 8-bit character interpretation, second bytes below `0x80` produce the same result.
- With a signed 8-bit character interpretation, second bytes at or above `0x80` become negative before conversion to unsigned. The subsequent unsigned shift can leave result bits 16–31 set.

There is no explicit final mask limiting the result to 16 bits.

When consumption is ineligible, neither byte address is read, and the result remains zero.

### Final store and return

After the eligible byte reads, if any, the function writes the selected outgoing cursor to offset `0x20`.

Every path reaching normal completion performs this store:

- Eligible consumption stores the cursor advanced by two.
- Ineligible consumption stores the unchanged operational cursor.

On the consuming path, both byte reads precede the cursor-field store. The final shift and bitwise combination occur after that store.

The function then returns the 32-bit unsigned result. An unchanged-value write is still an observable store and still requires writable storage.

## Side effects

The direct persistent mutation is one 32-bit cursor-field write on each normally completing path. The boundary and consumed bytes are not directly written.

Direct reads depend on the path:

- A nonzero object argument causes the initial validation cursor read.
- Failed validation causes a direct diagnostic-global value read before F-540af0155702 executes.
- Every continuing path performs the operational cursor read followed by the boundary read.
- Eligible consumption performs the two ordered byte reads.

The diagnostic-global read must not be conflated with effects inside F-540af0155702. The global read belongs to this function’s failing-validation branch. Any reads, writes, output, or other effects performed by the dependency are additional and unknown.

The function performs no explicit allocation, deallocation, ownership transfer, or reference-count operation. This does not establish that the diagnostic dependency preserves ownership or releases nothing.

No non-aliasing guarantee is supplied. The byte addresses may overlap object fields or other relevant storage. If they overlap the cursor field, both byte reads precede this function’s final cursor write. No extra field reads are introduced merely because the byte addresses alias object storage.

There is no visible synchronization, atomic-access contract, rollback, or recovery mechanism. The described ordering is the ordering in the supplied representation, not an independently verified machine-level memory-order guarantee.

Invalid accesses and abnormal control flow are not converted into a documented error result. A failure before the final store does not establish completed cursor advancement.

## Dependencies

### F-540af0155702

**Call condition:** The object argument is zero, or the cursor obtained by the initial validation read is zero.

**Arguments:**

| Position | Established role |
|---|---|
| First | Assertion-related diagnostic reference |
| Second | Integer 364 |
| Third | Value obtained by directly reading a diagnostic global datum |

**Associated direct read:** This function reads the diagnostic global on the failing-validation branch before the dependency executes. The ordinary non-asserting path does not perform that read.

**Not established:** The global datum’s access width, type, exact value, or the evaluation order among call arguments.

**Continuation:** If the dependency returns, this function proceeds to the operational cursor and boundary reads without revalidating the object argument or cursor.

**Unknown dependency contract:** Whether it returns, terminates execution, transfers control elsewhere, emits diagnostics, changes object storage or globals, or acquires or releases memory.

The exact diagnostic reference contents and global datum contents are documented_only. No other function calls are visible in the supplied body.

## Acceptance examples

These are conditional expectations derived from static evidence, not runtime results. Unless stated otherwise, assume stable accessible memory, writable cursor storage, valid nonwrapping arithmetic, and normal completion.

### Ordinary consumption

Given an operational cursor of `0x1000`, a boundary of `0x1002`, and bytes `0x34` and `0x12` at the cursor and following address:

- The candidate `0x1001` is strictly below the boundary.
- The two bytes are read in address order.
- The cursor field is written as `0x1002`.
- The result is `0x00001234`.

The result is independent of plain character signedness because the second byte is below `0x80`.

### Exactly one byte before the boundary

Given a cursor of `0x1000` and boundary of `0x1001`:

- The candidate equals the boundary.
- Consumption is ineligible.
- Neither byte is read.
- The cursor field is written with `0x1000`.
- The result is zero.

Equality is insufficient for consumption.

### Boundary below the cursor

Given a cursor of `0x1000` and boundary of `0x0FFF`, the unsigned comparison fails under the stated arithmetic assumptions. The function reads neither byte, writes the unchanged cursor, and returns zero.

This example does not establish behavior for arithmetic near signed overflow limits.

### Eligible consumption returning zero

Given an eligible comparison and two zero bytes, the function advances the cursor by two and returns zero.

This result is numerically indistinguishable from ineligible consumption. The cursor store and byte-read behavior distinguish the cases.

### High-bit second byte

Given eligible bytes `0x34` and `0x80`:

- Under an unsigned 8-bit plain character interpretation, the result is `0x00008034`.
- Under a signed 8-bit plain character interpretation, the result is `0xFFFF8034`.

Both cases advance the cursor by two. Selecting one expected result requires establishing the character interpretation.

### Non-asserting path

Given a nonzero object argument and a nonzero cursor in the initial validation read:

- F-540af0155702 is not invoked.
- This function does not read the diagnostic global datum.
- The cursor is read again for the operational computation.

The absence of the direct diagnostic-global read does not describe arbitrary external activity or unknown effects outside this function.

### Failed validation and returning dependency

Suppose the object address is valid but the initial cursor value is zero, and the diagnostic global contains some accessible value.

The function reads that global value and supplies it as the third argument to F-540af0155702, alongside the diagnostic reference and 364. It does not pass the global datum’s address as the third argument.

If the dependency returns after changing the object to contain a valid cursor and eligible boundary, the operational reads use those later field values and consumption can proceed. This is conditional behavior, not a promise that the dependency repairs the object.

Changes to the global during dependency execution do not turn the already supplied third argument into a live reference to that global.

### Null object argument

A zero object argument skips the initial cursor-field read, reads the diagnostic global value, and invokes F-540af0155702.

If that dependency returns, the function attempts operational field accesses relative to the zero argument. A clean zero return is not established.

## Uncertainties

- **documented_only — Character interpretation:** Plain character signedness is not independently established. High-bit second-byte results remain conditional.
- **documented_only — Arithmetic limits:** Signed cursor increments and address calculations appear in the representation. Overflow and out-of-range address behavior are not established.
- **documented_only — Diagnostic global:** Its value is directly read on failed validation, but its access width, type, contents, and argument-evaluation ordering are not established.
- **documented_only — Diagnostic dependency:** Termination, output, memory mutation, global preservation, ownership effects, and other behavior of F-540af0155702 are unknown.
- **documented_only — Object schema:** Only the two directly accessed fields and their uses are established.
- **documented_only — Memory and concurrency:** Allocation membership, accessible extents, alias restrictions, and synchronization guarantees are unavailable.
- **Inferred intent:** A bounded little-endian two-byte reader is a useful interpretation under ordinary address arithmetic. It does not override the strict unsigned comparison, signed-arithmetic limitations, or character conversion.

## Evidence

The supplied static body establishes the conditional initial cursor read, failed-validation dependency call, operational field rereads, strict unsigned eligibility comparison, conditional ordered byte reads, unconditional final cursor store on normal completion, and unsigned 32-bit return.

The failing-validation call directly uses the value of a diagnostic global datum as its third argument. This supports a branch-specific global read before dependency execution, distinct from the first argument’s diagnostic reference. It does not establish the global access width or argument-evaluation order.

The field offsets, field access widths, diagnostic integer 364, byte conversion sequence, store placement, and absence of a final 16-bit result mask are supported by the representation.

The reader purpose and exclusive-boundary interpretation are inferred intent under stated arithmetic assumptions. Dependency behavior, global metadata, character signedness, overflow semantics, and wider object invariants remain unresolved.

No runtime tests have been performed.
