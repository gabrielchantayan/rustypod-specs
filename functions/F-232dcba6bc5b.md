<!-- Trusted provenance: F-232dcba6bc5b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-232dcba6bc5b

## Purpose

F-232dcba6bc5b reads a five-byte header followed by a counted sequence of packed records and appends six decoded values per record to a state-managed destination array.

Established behavior includes:

- Checking the header and declared payload against an exclusive input boundary.
- Selecting field widths from five header flag bits.
- Requesting destination capacity before checking payload completeness.
- Computing the append address with wrapping 32-bit arithmetic.
- Writing six unsigned values into each 36-byte destination slot.
- Updating the recorded element count after successful payload processing.
- Returning a 64-bit composite containing a status and a separately retained address.

The operation is not transactional. Capacity-related state writes can remain visible when subsequent payload validation fails.

The meanings of the decoded fields and the surrounding format are not established.

## Interface

### Inputs and preconditions

The routine receives three inputs:

| Input | Representation and local use |
|---|---|
| Input start | A 32-bit address supplied through a signed 32-bit parameter representation |
| Input boundary | A byte-addressed pointer used for upper-bound comparisons |
| State | A pointer to a structure accessed through 32-bit words |

The boundary is exclusive. An endpoint equal to the boundary passes the corresponding size check.

There are no explicit checks for null pointers, state-object size, destination accessibility, alignment, or consistency between the recorded count, capacity, and allocation. Safe conventional decoding requires valid accessible storage, adequate destination space, and input and destination regions that do not overlap.

The size comparisons do not independently establish that all addresses are valid. In particular, the routine does not reject overflow in the append-address arithmetic.

### Header layout

Offsets are relative to the input start.

| Byte offset | Meaning |
|---|---|
| 0–2 | Not directly read by this routine |
| 3 | Width-selection flags |
| 4 | Unsigned record count |
| 5 onward | Packed records |

The record count ranges from 0 through 255. Only flag bits 0–4 influence this routine’s decoding; bits 5–7 have no direct effect.

### State layout

Each state entry is a 32-bit word.

| Word index | Byte offset | Local role |
|---|---:|---|
| 0 | 0 | Opaque value supplied to F-6dd003795db7 |
| 23 | 92 | Recorded element count |
| 24 | 96 | Recorded capacity, in elements |
| 25 | 100 | Destination base address |

Other state words are not directly accessed.

The routine does not independently enforce a count-at-most-capacity invariant. These fields describe how the routine uses the state, not guarantees about its validity.

### Destination layout

Each destination slot occupies 36 bytes, comprising nine 32-bit words.

| Slot word | Byte offset | Effect |
|---|---:|---|
| 0 | 0 | Receives field A |
| 1 | 4 | Receives field B |
| 2 | 8 | Receives field C |
| 3–4 | 12–16 | Not directly written |
| 5 | 20 | Receives field D |
| 6 | 24 | Receives field E |
| 7 | 28 | Receives field F |
| 8 | 32 | Not directly written |

The six values are stored as unsigned 32-bit quantities. The routine does not clear slots or initialize their three unwritten words. Destination-word byte order is not independently established here; packed-input byte order is specified below.

### Return value

The result is a 64-bit composite:

| Bits | Content |
|---|---|
| 0–31 | 32-bit status |
| 32–63 | Retained 32-bit address |

The status is initially zero. Either structural-size failure assigns status 8. A capacity dependency may supply another nonzero status.

The retained address is:

- The supplied input boundary when no capacity request occurs.
- The destination address read before the capacity call when a capacity request occurs.

The second case applies even if the capacity call fails. The upper half is therefore neither consistently an input-consumption position nor consistently the current destination address.

## Behavior

### Header check and retained input values

The first check compares the address five bytes after the input start with the boundary.

If that endpoint exceeds the boundary, the routine:

1. Assigns status 8.
2. Calls F-4457a9a44a68 with a fixed diagnostic argument.
3. Returns if that dependency returns normally.

On this path, the routine directly reads neither header bytes nor state words. The retained return address remains the supplied boundary.

When the header fits, the routine reads the unsigned count at offset 4 and then the flags at offset 3. Both values are retained; neither is reread after a dependency call. The payload starts at offset 5.

These comparisons do not constitute a separate address-overflow or mapped-memory validation mechanism.

### Capacity decision

The routine adds the retained record count to state word 23 using unsigned 32-bit arithmetic. The sum wraps modulo \(2^{32}\).

It compares that wrapped sum with state word 24 using an unsigned comparison. Growth is requested only when the sum exceeds the recorded capacity.

The requested capacity is formed by adding three to the sum, again modulo \(2^{32}\), and clearing the lowest two bits. For nonoverflowing values this rounds upward to a multiple of four. Near the 32-bit limit, the addition can wrap and produce a smaller request, including zero.

Consequences include:

- An overflowing count sum can suppress a capacity request.
- Capacity rounding can wrap even when the preceding count sum did not.
- A zero-record header can still cause growth if the existing count exceeds capacity.

There is no explicit rejection of these conditions.

### Capacity call and state-write order

When growth is required, F-6dd003795db7 receives:

- State word 0 as an opaque argument.
- Element size 36 bytes.
- The recorded element count read for the call.
- The rounded requested capacity.
- The current destination address.
- A writable location containing the initially zero 32-bit status.

The destination address supplied to this call also replaces the retained address used in the result’s upper half.

After the dependency returns, the routine unconditionally stores its 32-bit return value in state word 25. It tests the status only after this write.

If the status is nonzero:

- Processing stops without payload validation or record decoding.
- The routine does not perform its capacity-update write.
- The dependency-supplied status is returned.
- F-4457a9a44a68 is not called by this failure path.

If the status remains zero, the routine writes the requested capacity to state word 24 and continues.

No preservation contract for the dependency is available. Reads performed after the call observe the state then present, rather than necessarily the state used to make the growth decision.

### Record size

Each record has the following packed fields:

| Field | Width with controlling bit clear | Width with controlling bit set | Flag bit |
|---|---:|---:|---:|
| A | 1 byte | 2 bytes | 0 |
| B | 1 byte | 2 bytes | 1 |
| C | 1 byte | 1 byte | None |
| D | 2 bytes | 3 bytes | 2 |
| E | 2 bytes | 3 bytes | 3 |
| F | 1 byte | 2 bytes | 4 |

The packed record size is eight plus the number of set bits among bits 0–4. It ranges from 8 through 13 bytes.

The total declared payload is the retained count multiplied by that size. Its maximum is \(255 \times 13 = 3315\) bytes, so this size multiplication does not overflow 32 bits.

### Append-address arithmetic

Before checking payload completeness, the routine computes the initial destination pointer from state words 23 and 25 as they exist after any capacity call.

Let \(c\) be the unsigned 32-bit count read from word 23 and \(b\) the 32-bit destination base read from word 25. The arithmetic is:

\[
o = (36c) \bmod 2^{32}
\]

\[
a = (b + o) \bmod 2^{32}
\]

The resulting 32-bit value \(a\) is then converted to the destination pointer.

Both operations have established wrapping behavior:

- The count-to-byte-offset multiplication is unsigned 32-bit multiplication.
- The base addition is 32-bit addition.
- Neither operation is widened to preserve the mathematical address.
- Neither overflow causes a reported error by itself.

For example, \(c=\text{0x40000000}\) yields a zero byte offset, because \(36c=9\times2^{32}\). The append pointer is then the destination base, not an address \(36c\) bytes beyond it.

Likewise, base \(\text{0xFFFFFFF0}\) and count 1 produce offset \(\text{0x24}\) and address \(\text{0x00000014}\).

These arithmetic results are established independently of whether the resulting pointer is accessible. Memory faults or other target-specific consequences of using such pointers remain unresolved.

### Payload check

The routine compares the endpoint of the entire declared payload with the input boundary before decoding any record.

If the endpoint exceeds the boundary:

1. Status becomes 8.
2. F-4457a9a44a68 is called.
3. The routine returns if that dependency returns normally.

No record fields have been directly written on this path, and the final count update is skipped. A preceding capacity call and its destination-address and capacity writes are not rolled back.

Equality with the boundary is accepted. Extra input after the declared payload is ignored.

### Decoding and observable write order

All input fields are unsigned. Multibyte fields use most-significant-byte-first order:

- Two bytes represent \(256b_0+b_1\).
- Three bytes represent \(65536b_0+256b_1+b_2\).

The resulting ranges are 0–255, 0–65,535, and 0–16,777,215 for one-, two-, and three-byte fields respectively. There is no sign extension, scaling, or semantic range validation.

Records are processed in input order. For each record, the established field-level sequence is:

1. Read A and write slot word 0.
2. Read B and write slot word 1.
3. Read C and write slot word 2.
4. Read D and write slot word 5.
5. Read E and write slot word 6.
6. Read F and write slot word 7.
7. Advance the destination by one 36-byte slot.

The relative order of individual byte reads within a multibyte field expression is not established.

The routine does not stage the whole payload before writing. Overlap can therefore allow an earlier destination write to change later input bytes or state. Non-overlap is a precondition for the ordinary interpretation of this operation as appending independently decoded records.

### Final count update

After all declared records are processed, the routine rereads state word 23, adds the retained header count using 32-bit arithmetic, and writes the wrapped result back.

This is a fresh state read. It need not equal the count used for the append address if dependency effects or overlapping storage have changed it.

For count zero:

- No packed record bytes are read.
- No slot fields are written.
- Capacity handling and payload validation still occur.
- The final count read-and-write still occurs if payload validation succeeds.

Under valid, nonoverlapping, nonwrapping conditions without unrelated dependency mutations, success appends the declared records and increases the count by that number.

## Side effects

Direct writes comprise:

- State word 25 after a capacity call, regardless of reported status.
- State word 24 after a capacity call whose status is zero.
- Six 32-bit words per decoded slot.
- State word 23 after successful payload processing.

The routine also calls the diagnostic dependency on either structural-size failure.

There is no direct release operation and no established ownership transfer for the input, state, or destination. The capacity dependency may have additional ownership or allocation effects, but those are not specified by this caller.

Failures are not rollback boundaries. Capacity failure can leave word 25 changed; payload failure following growth can leave both words 24 and 25 changed. Dependency effects may add further changes.

Invalid storage, insufficient actual allocation, or wrapped addresses are not visibly converted into status 8.

## Dependencies

### F-6dd003795db7

**Established:** conditionally receives an opaque state value, element size, existing count, requested capacity, old destination address, and writable status location. Its return value is stored as the destination address before the status is tested.

**Inferred intent:** obtaining sufficient destination storage while retaining existing elements.

**Documented_only:** allocation strategy, copying, release behavior, ownership transfer, failure guarantees, status meanings, preservation of globals and reachable memory, and whether it returns normally. None is assumed.

### F-4457a9a44a68

**Established:** called with a fixed diagnostic argument after status is assigned 8 for a short header or insufficient declared payload. Its return value is unused.

**Inferred intent:** reporting malformed or truncated input.

**Documented_only:** diagnostic text, output destination, global and memory effects, callbacks, and normal-return or termination behavior.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, assume valid nonoverlapping storage, adequate actual allocation, nonwrapping input addresses, and dependencies that return normally without unrelated mutations.

### Minimum-width record

Flags zero, count one, and payload bytes 1 through 8 produce:

| Slot word | Value |
|---|---:|
| 0 | 1 |
| 1 | 2 |
| 2 | 3 |
| 5 | 1,029 |
| 6 | 1,543 |
| 7 | 8 |

Words 3, 4, and 8 remain unchanged. With sufficient recorded capacity, status is zero and the recorded count increases by one.

### Maximum-width record

With controlling bits 0–4 set, count one, and payload bytes 1 through 13, the fields are:

| Field | Value |
|---|---:|
| A | 258 |
| B | 772 |
| C | 5 |
| D | 395,016 |
| E | 592,395 |
| F | 3,085 |

Setting bits 5–7 as well does not change these values or the 13-byte record size.

### Exact endpoint and short input

A boundary exactly at the declared payload endpoint passes. Additional trailing bytes are ignored.

A boundary before the five-byte header endpoint causes status 8 and a diagnostic call without direct header or state reads. The result’s upper half remains the supplied boundary.

### Successful growth followed by truncation

With recorded count 3, capacity 3, and header count 2, the requested capacity is 8.

If F-6dd003795db7 reports success, its returned address and capacity 8 are stored before payload validation. A subsequent short-payload result sets status 8 and calls the diagnostic dependency. No record fields or final count update are directly written.

The result’s upper half contains the destination address supplied to the capacity call, not the replacement address.

### Capacity failure

If growth is required and the dependency sets a nonzero status, its returned address is still written to word 25. No payload check, decoding, capacity-update write, final count update, or structural-error diagnostic follows.

### Wrapped append offset

Suppose the recorded count is \(\text{0x40000000}\), capacity is \(\text{0x40000001}\), and the header declares one minimum-width record.

The capacity comparison does not request growth. The append offset is zero modulo \(2^{32}\), so the record is written at the destination base. After successful decoding, the count becomes \(\text{0x40000001}\).

This case requires writable storage at the base; it does not require an allocation extending the mathematical product \(36c\) bytes beyond that base.

### Wrapped base addition

With base \(\text{0xFFFFFFF0}\) and count 1, the computed append address is \(\text{0x00000014}\). This is an arithmetic acceptance condition only. Successful writes additionally require valid writable storage at the resulting addresses; a managed error for inaccessible storage is not established.

### Empty record set

With a complete header, count zero, and a count not exceeding capacity, no growth occurs and no record bytes are read. Payload validation passes under the stated nonwrapping input assumptions. The final count read-and-write still occurs, and status remains zero.

## Uncertainties

The following remain documented_only:

- Semantic names and units of the six fields.
- Validation and invariants supplied by callers.
- Allocation ownership and lifetime.
- Dependency preservation of globals, input, state, and destination storage.
- Whether callers use both halves of the result.
- The intended interpretation of the retained return address.
- Target-specific consequences of invalid, unaligned, overlapping, or wrapped pointers.
- Initial contents of the three unwritten slot words.
- Behavior when a dependency does not return normally.

The append multiplication and base addition are not uncertain: both wrap modulo \(2^{32}\) before pointer conversion. Uncertainty concerns subsequent pointer use, not those arithmetic results.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping.

Established evidence covers header-read order, unsigned count and flag interpretation, state offsets, capacity comparison and rounding, dependency-call sequencing, 32-bit append-address multiplication and addition, payload validation, field widths and byte order, slot-write order, final count update, and composite return construction.

Capacity management and diagnostic reporting are inferred purposes, not dependency guarantees. Dependency bodies, runtime traces, and implementation access were unavailable. No runtime tests have been performed.
