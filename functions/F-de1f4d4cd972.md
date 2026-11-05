<!-- Trusted provenance: F-de1f4d4cd972; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de1f4d4cd972

## Purpose

F-de1f4d4cd972 conditionally expands a compact byte record into two adjacent sequences of 32-bit numeric values. It publishes the sequences’ addresses in a supplied context.

The record begins with a one-byte header containing two four-bit counts. Each following two-byte payload item encodes an unsigned, most-significant-byte-first value. The first count determines the boundary between the two output sequences; their sum determines how many items are decoded.

An existing nonzero base field bypasses record validation and initialization.

The result is a 64-bit composite, not an input cursor. Its low half is the final contents of a local 32-bit status location. Its high half is the original input-limit representation unless the allocation dependency is reached, in which case it is zero.

The status that permits initialization and the status eventually returned are distinct observations. The dependency receives the status location’s address. If later publication or decoding writes overlap that location, the returned status can differ from the zero value that permitted initialization.

## Interface

### Arguments and address model

The function has three observed arguments:

| Argument | Role |
|---|---|
| Input start | Address of the candidate record’s header. |
| Input limit | Boundary used for record-availability checks. |
| Context | Address of storage containing the fields below. |

The supplied representation uses 32-bit pointer values and 32-bit context slots. The input limit functions as an exclusive upper boundary under ordinary, ordered, nonwrapping buffer assumptions.

Relevant context fields are:

| Byte offset | Width | Observed use |
|---|---:|---|
| `0x00` | 32 bits | Value passed as the first argument to F-6dd003795db7. |
| `0x40` | 32 bits | Destination for the second-sequence pointer. |
| `0x4C` | 32 bits | Existing-initialization gate and destination for the base pointer. |

No interpretation is established for other context storage or for the field at offset `0x00`.

### Record layout

For a stable header:

- Bits 0–3 specify the first sequence’s count.
- Bits 4–7 specify the second sequence’s count.
- Each count is between 0 and 15.
- The total is between 0 and 30.
- The required record length is one header byte plus twice the total: 1 through 61 bytes.
- Payload items for the first sequence precede those for the second.

Each payload item consists of an upper byte followed by a lower byte. Its numeric value is 256 times the upper byte plus the lower byte, ranging from 0 through 65,535.

Each output element occupies four bytes and contains that numeric value without sign extension. This specifies the value, not the destination memory’s byte order.

### Result

The result’s two halves are:

| Path | Low 32 bits | High 32 bits |
|---|---|---|
| Existing base is nonzero | Zero | Original input limit |
| Either availability check fails | Final status after the diagnostic call | Original input limit |
| Allocation dependency is reached | Final status after any publication and decoding | Zero |

Before a malformed-input diagnostic, the function sets status to `8`. If the diagnostic returns normally and does not change that location, the low result half is `8`.

After F-6dd003795db7 returns, the function checks status once to decide whether to publish and decode. It reads status again for the result after that branch finishes. Consequently:

- Zero status at the branch decision permits initialization.
- A nonzero returned status does not prove initialization was skipped.
- A zero-status dependency result produces a zero composite result only if status remains zero until the final read.
- The high half never reports the end of the consumed record.

Other nonzero status patterns are not interpreted by this function.

### Preconditions and validity limits

The initial context gate must be readable. If initialization proceeds, the additional context accesses must also be valid.

Ordinary buffer interpretation requires valid, ordered input addresses and address additions that do not wrap or leave the supported address model. The function does not independently establish those conditions.

Successful decoding requires the returned destination to support the calculated number of consecutive four-byte stores. The function does not independently check capacity, alignment, non-nullness, lifetime, ownership, or separation from input, context, or status.

Output storage disjoint from status is insufficient to guarantee an unchanged final status. Both four-byte context publication destinations must also be disjoint from the status location, and other agents must leave status unchanged.

## Behavior

### Existing-base bypass

The function first initializes its local status to zero, then reads the context field at offset `0x4C`.

If that read yields a nonzero value, the function returns immediately:

- No header or payload is read.
- Neither dependency is called.
- Neither context pointer field is written through the context.
- The result has low half zero and high half equal to the original input limit.

The gate checks only nonzeroness. It does not validate the existing allocation, the second-sequence pointer, or the input range.

The zero initialization precedes the gate read. Statements about unchanged external storage therefore require that storage not overlap the local status location.

### Header availability and count capture

When the gate is zero, the function first checks whether the input start advanced by one byte is at or below the input limit.

Failure takes the malformed-input path without a header read.

On success, the first count is obtained from the header’s low nibble. A subsequent header access supplies the high-nibble contribution to the total count. The representation does not establish a single atomic header snapshot.

With stable input, the two counts describe the same header. With mutation between accesses, the first count and high-nibble contribution can come from different byte values. Each contribution still lies between 0 and 15, and their sum still lies between 0 and 30.

The first count and total are retained across the dependency call. They are not recalculated from the header afterward.

Count arithmetic is bounded: twice the total is at most 60, four times the first count is at most 60, and all destination elements occupy at most 120 bytes. These bounds prevent overflow in the observed count calculations, but do not prove that adding the resulting offsets to addresses is valid.

### Complete-record availability and rejection

The second check compares the input start advanced by one header byte and twice the total count against the input limit.

Equality is accepted. A larger supplied range is also accepted; trailing bytes need not be absent.

On either check’s failure, the function:

1. Sets local status to `8`.
2. Calls F-4457a9a44a68 with a fixed diagnostic-string argument.
3. Performs no allocation request or payload decoding.
4. Returns using the original input limit as the high half, provided the diagnostic returns normally.

There is no further direct status assignment on this path. The diagnostic’s broader effects are unknown, so preservation of status across that call is conditional rather than guaranteed.

### Allocation request

After both checks succeed, the function selects zero for the return value’s high half and calls F-6dd003795db7 with six arguments in this order:

1. The 32-bit value read from context offset `0x00`.
2. The constant `4`.
3. Zero.
4. The captured total count.
5. Zero.
6. The address of the local 32-bit status location, initially zero.

The returned value is used as a pointer to four-byte elements.

The argument pattern and subsequent stores suggest a request for the total count of four-byte elements. That is inferred intent, not an established allocator contract.

The call occurs even when the total is zero.

After the dependency returns, the function reads status:

- Nonzero status skips direct publication and decoding.
- Zero status enters publication and decoding.

No second status check interrupts initialization once that branch is entered.

On the nonzero branch, the function performs no direct context-pointer publication or payload store and makes no explicit release call. This does not imply that the dependency left context, input, globals, or allocation state unchanged.

### Pointer publication

On the zero-status branch, publication occurs in this order:

1. Store the returned pointer at context offset `0x4C`.
2. Store that pointer advanced by four times the captured first count at context offset `0x40`.

Both publication stores precede the first decoded-element store.

For ordinary, suitable storage, the second-sequence pointer is:

- Equal to the base when the first count is zero.
- At the first element of the second sequence when that sequence is nonempty.
- Immediately after the populated output when the second count is zero.

The two stores are not an atomic publication transaction. No rollback or synchronization is established.

### Payload decoding

Decoding starts one byte after the original input start and processes exactly the captured total count.

For each item, the function reads the next two source bytes, forms their unsigned big-endian numeric value, and stores that value into the next four-byte destination element. Source positions advance by two bytes; destination positions advance by four bytes.

The representation does not establish which of the two source bytes is physically read first. It does establish sequential item processing, with each item’s destination store preceding processing of the next item.

There is no repeated bounds check and no new header-derived count after the dependency call. The input bytes are read after that call and after pointer publication. Dependency changes, publication aliases, or earlier output stores can therefore affect bytes subsequently decoded.

For a total of zero, both context fields receive the returned pointer and no payload byte or destination element is accessed. A null return is not separately rejected when status is zero.

### Final status observation

The low result half comes from a final status read after the selected branch’s work.

Because the dependency receives the status address, its returned storage is not established to be separate from that location. A decoded-element store can replace status after the successful branch decision.

The context publication destinations are likewise not established to be separate from status. Either publication store can change it before decoding begins, without causing another branch decision.

Thus, disjoint output storage alone cannot justify an unconditional zero-result assertion. A zero result on the allocation path additionally requires that publication and other intervening effects leave status zero.

## Side effects

### Established direct accesses

The principal ordering is:

1. Initialize local status.
2. Read the existing-base gate.
3. Check header availability before header reads.
4. Capture counts before checking complete-record availability.
5. Check complete availability before the allocation dependency.
6. Read the context’s first field for that dependency call.
7. Inspect post-call status before publication.
8. Publish the base, then the second-sequence pointer.
9. Decode payload items into consecutive destination elements.
10. Read final status for the result.

On rejection, status is set to `8` before the diagnostic call.

Successful initialization issues two four-byte context stores and one four-byte output store per captured item. These logical destinations need not be distinct physical locations. Later stores may overwrite earlier publications, input bytes, or status if storage overlaps.

### Ownership, overlap, and partial state

No explicit deallocation, rollback, overlap detection, or protective input copy appears.

The base is published before the output is populated. If execution is interrupted after publication, the context may expose incompletely initialized output.

The retained pointer’s ownership and lifetime are not established. Publishing a pointer does not prove that it refers to owned storage or that the storage outlives the call. In particular, a pointer aliasing the local status location gains no valid post-return lifetime merely through publication.

The function supplies no established atomicity, thread-safety, reentrancy, or cross-thread visibility guarantees.

## Dependencies

### F-6dd003795db7

Established interaction:

- Called once after both availability checks succeed, including for a zero total.
- Receives the six arguments described above.
- Receives an address at which status may be written.
- Returns the pointer used for publication and decoding.
- Its post-call status controls entry into initialization.

The final status need not equal the immediate post-call status because subsequent stores may overlap it.

**documented_only:** allocation mechanism, storage extent, alignment, null behavior, zero-count behavior, aliasing exclusions, ownership, release responsibilities, global effects, and abnormal or non-returning behavior.

### F-4457a9a44a68

Established interaction:

- Called on either availability-check failure, after status becomes `8`.
- Receives a fixed diagnostic-string argument.
- Has no observed return-value use.
- Is not called for an existing-base bypass or merely for nonzero allocation status.

**documented_only:** literal diagnostic text, output destination, formatting, shared-state effects, preservation guarantees, and whether it always returns normally.

Neither dependency is assumed to preserve globals or to acquire, transfer, or release ownership.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

Unless explicitly overridden, assume:

- Valid, nonwrapping address arithmetic and stable readable input.
- Normally returning dependencies.
- Suitable output storage disjoint from input, context, and the local status location.
- **Neither four-byte context publication destination overlaps any byte of the local status location.**
- No other status mutation after the dependency’s stated status result.

1. **Existing initialization bypasses unusable input.**  
   A nonzero base gate causes return without header reads or dependency calls, even if the supplied range cannot hold a header. No context-pointer publication occurs. The low result half is zero; the high half is the original input limit.

2. **Missing header.**  
   With a zero gate and input limit equal to input start, the first availability check fails. No header is read. The diagnostic is called after status becomes `8`. If it preserves status, the result has low half `8` and the original input limit as its high half.

3. **Truncated payload.**  
   Header `0x12` captures two first-sequence items and one second-sequence item, requiring seven bytes. A six-byte range fails the complete-record check without allocation or payload decoding.

4. **Exact-length successful decoding.**  
   The seven bytes `0x12, 0x00, 0x01, 0x80, 0x00, 0xFF, 0xFF` produce numeric output values 1, 32,768, and 65,535 when the dependency leaves status zero and returns suitable storage.

   The second-sequence pointer is eight bytes beyond the base. Under the stated assumptions, neither output stores nor either context publication store overlaps status, and no other effect changes it. The composite result is therefore zero.

5. **Trailing bytes.**  
   The same record in a larger range is accepted. With no overlapping storage or dependency mutation, the function’s own decoding accesses only the three payload pairs. The result does not communicate a consumed-input cursor.

6. **Dependency-reported failure.**  
   If F-6dd003795db7 leaves status `5`, the function skips its publication and decoding. With status unchanged thereafter, the result has low half `5` and high half zero. Dependency-originated side effects remain unspecified.

7. **Decoding changes the returned status.**  
   For bytes `0x01, 0x00, 0x05`, override the output/status separation assumption: the dependency leaves status zero and returns the supplied status address as its destination. Assume valid pointer operations and context publication destinations disjoint from status.

   The function publishes the base and the address four bytes beyond it, then stores numeric value 5 into status. It returns low half `5`, high half zero, despite having completed initialization.

8. **Publication may change status independently of output.**  
   Override only the publication/status separation assumption. Suppose a context publication destination overlaps status while output storage remains disjoint. The zero post-dependency status still permits initialization; the overlapping publication may then change the final low half. Disjoint output storage does not establish a zero composite result.

9. **Empty record.**  
   Header `0x00` needs one byte and still causes a count-zero dependency call. With zero status, both context fields receive the returned pointer and there are no payload or element accesses. Under the shared separation assumptions, the result is zero. If the returned pointer is null, the base gate remains zero for a later invocation.

10. **Maximum counts.**  
    Header `0xFF` requires 61 input bytes and represents 30 output elements. With zero status and suitable storage, the second-sequence pointer is 60 bytes beyond the base; 30 four-byte stores populate 120 bytes.

## Uncertainties

The following remain **documented_only**:

- Domain-specific meanings of the sequences and the first dependency argument.
- Dependency storage, ownership, lifetime, cleanup, and preservation contracts.
- Intended permission or exclusion of aliases among output, input, context, and status.
- Behavior for invalid addresses, address wraparound, or unrelated pointer ranges.
- Zero-count allocation and null-return guarantees.
- Diagnostic contents and externally visible reporting behavior.
- Concurrency, reentrancy, synchronization, and abnormal termination.
- Whether the composite return convention is intentional API design.

The existing-base gate suggests initialize-once intent. The allocation arguments suggest four-byte element storage. Neither inference strengthens the established behavior.

## Evidence

The supplied static representation establishes the context access widths and offsets, two availability checks, separate header accesses, bounded count calculations, dependency arguments, status-controlled branch, publication order, unsigned two-byte expansion, and final 64-bit composition.

The distinction between branch status and returned status follows from the separate status observations and the exposed status address. Neither output/status separation nor context-publication/status separation is established by the function itself.

No runtime tests have been performed.
