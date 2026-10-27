<!-- Trusted provenance: F-86ba47dd8d6f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-86ba47dd8d6f

## Purpose

F-86ba47dd8d6f transfers bytes from a zero-terminated input into a destination whose first byte serves as a running length field. Transferred bytes begin immediately after that field. The operation stops when the next input byte is zero or, for a nonzero input byte, when the length field equals the supplied limit.

The destination is **not** a conventional zero-terminated output string. The input terminator is not copied, and no output terminator is appended.

The limit is compared against the current one-byte length field, not against an independent total number of bytes transferred. Consequently, a limit outside the field’s representable range does not impose a transfer bound. The length field wraps while the input and output positions continue advancing.

The apparent intent is a length-prefixed byte-sequence copy with a caller-selected limit. That intent is inferred; the contract below describes the observable mechanism, including its wraparound and aliasing behavior.

## Interface

### Inputs

The function receives three arguments:

1. **Input address.** Identifies the first byte to examine. A zero byte ends the transfer.
2. **Destination address.** Identifies a writable one-byte length field. Payload storage begins one byte after this address.
3. **Limit.** An unsigned integer compared for equality with the current destination length field.

The represented width model is an 8-bit byte and a 32-bit unsigned limit. Under this model:

- Input and payload values range from 0 through 255.
- The destination length field ranges from 0 through 255.
- The limit ranges from 0 through 4,294,967,295.
- A stored length increment is reduced modulo 256.
- The length field is widened without sign extension for comparison with the limit.

The supplied material does not include the underlying type definitions or an ABI declaration. These numeric widths therefore describe the recovered representation’s intended model, rather than an independently verified ABI.

### Output

There is no explicit return value. Results are conveyed through destination memory:

- The first destination byte holds the final running count modulo 256.
- Subsequent bytes contain the payload written during this invocation.
- Bytes beyond the last payload write are not explicitly changed by this function.

A final count of zero does not necessarily mean that no bytes were copied. It can also represent 256, 512, or another positive multiple of 256 transfers.

### Preconditions and caller obligations

For defined, successful execution:

- The destination header must be writable before any input access.
- Every input position actually examined must be readable.
- Every payload position actually written must be writable.
- Address advancement must remain valid for the objects and execution environment involved.
- Available destination capacity must accommodate the actual number of transfers, not merely the final header value.

No null-address, capacity, overlap, or termination validation is present.

A zero limit does not remove the requirement for a readable first input byte. The function examines that byte before deciding whether the limit prevents transfer.

For limits greater than 255, finite destination capacity cannot be justified from the limit alone. Successful bounded execution then requires some other established constraint, such as a reachable input terminator and sufficient output storage.

## Behavior

### Initialization and storage layout

The first observable memory operation is a zero write to the destination header. The previous header value is discarded unconditionally.

The first potential payload byte occupies the address immediately following the header. Successive payload bytes occupy successive addresses. No padding, multi-byte length, encoding conversion, or alignment-dependent payload layout is visible.

The destination position is advanced before each input examination. On an iteration that stops without copying, this position advancement does not itself produce a payload memory access.

### Input examination and stopping order

Each iteration reads the current input byte before deciding whether to transfer it.

If that byte is zero:

- Execution stops.
- The header is not read for the limit comparison on that iteration.
- No payload byte is written.
- The input position is not advanced.

If the input byte is nonzero:

- The destination header is read.
- Its unsigned value is compared with the entire supplied limit.
- Equality stops execution without copying the examined byte.
- Inequality permits one transfer.

The order matters at a limit boundary: reaching the limit does **not** prevent the next input byte from being read. It prevents that byte from being copied only after the byte has been fetched and found nonzero.

For example, with a limit of three and three successful transfers, a fourth input-byte read still occurs. If that fourth byte is inaccessible, reaching the limit does not make execution safe.

### Successful transfer

A successful transfer has these observable memory effects, in order:

1. The previously examined input byte is written to the next destination payload position.
2. The input position advances by one byte.
3. The destination header is read again for the increment.
4. The incremented value is written back to the header.

The header read used for the increment is distinct from the earlier read used for the limit comparison. The visible operation is not an atomic payload-and-count update.

Under ordinary noninterfering execution, the header after each successful transfer equals the total number of transfers modulo 256.

No successfully transferred byte is zero: a zero input byte ends processing before its payload write.

### Limit interpretation and count wraparound

Equality, rather than an ordering comparison, governs stopping.

Under the represented width model:

- **Limit zero:** No payload is transferred, because initialization makes the header zero. The first input byte is nevertheless read.
- **Limits one through 255:** In noninterfering execution, no more than that many payload bytes are transferred. A zero input byte may stop the operation earlier.
- **Limits 256 through 4,294,967,295:** The header can never equal the limit. The limit therefore cannot stop processing.

For the last category, after 255 transfers the header is 255; after the next transfer it becomes zero. The payload destination does not restart, and the input does not rewind. Both continue progressing through memory.

The limit is not first narrowed to one byte. In particular, 256 does not behave like zero.

### Termination and output suffix

The terminating zero is consumed only as an examined value; it is not copied. No replacement terminator is written.

Likewise, when the limit stops processing, the first excluded nonzero byte is read but not transferred. The destination position reserved for that iteration receives no write.

Any existing byte immediately after the copied payload remains unchanged unless aliasing with another explicit write causes a change.

### Overlapping input and destination

The visible implementation does not reject overlap, stage the entire input, or provide snapshot semantics. Earlier destination writes can affect later input reads.

Relevant consequences include:

- If the input address is the destination header address, initialization replaces the first input byte with zero. The next input examination observes zero, and no payload is transferred.
- If the input starts at the first payload position, each transferred byte is written back to its current location before the input advances. With suitable capacity and a terminator or effective limit, this can operate in place.
- If the first payload position lies ahead of the input position within the same region, a payload write can replace a byte that would otherwise have been read later. The operation can repeat overwritten values or destroy an apparent terminator.
- If the destination header lies later in the input region, initialization can introduce a zero into the future input stream; subsequent header updates can change that byte again before it is examined.

An analysis based on the input contents at call entry is therefore valid only when aliasing cannot change the relevant future reads.

## Side effects

The function directly accesses only the input bytes, destination header, and payload positions described above.

Its explicit writes are:

- One initial zero write to the destination header.
- One payload-byte write per successful transfer.
- One header update per successful transfer.

It does not visibly allocate or release memory, retain either address, perform I/O, or invoke another function. No ownership transfer is expressed. The caller must provide the storage and arrange its lifetime.

There is no explicit error return, failure flag, rollback, or recovery path. If an access faults or the environment otherwise interrupts execution, earlier writes may already be visible.

In particular, interruption between a payload write and the following header update can leave a newly written payload byte that is not reflected in the header. The supplied material does not establish transactional or concurrency guarantees.

## Dependencies

No function dependencies are visible. F-86ba47dd8d6f contains no explicit calls, so there are no additional opaque function IDs to enumerate.

Its environmental dependencies are readable and writable byte-addressed memory, valid address progression, and the integer conversion behavior described under the width model.

The supplied material establishes no allocator, runtime error handler, external state service, or synchronization dependency.

## Acceptance examples

These are conditional contract examples derived from static evidence. They are not runtime test results. Unless otherwise stated, storage is valid, input and output do not overlap, and no external agent modifies memory during execution.

### Empty input

Given an input whose first byte is zero and an arbitrary limit:

- The destination header becomes zero.
- The first input byte is read.
- No payload byte is written.
- No header read for limit comparison is needed.
- Existing payload bytes remain unchanged.

### Zero limit with nonempty input

Given input bytes 65, 66, zero and limit zero:

- The destination header becomes zero.
- Input byte 65 is read.
- The header is read and equals the limit.
- No payload byte is written.
- Input byte 66 is not examined.

### Input ends before the limit

Given input bytes 65, 66, zero and limit five:

- The destination header ends at two.
- Payload positions one and two contain 65 and 66.
- The zero input byte is examined but not copied.
- Payload position three retains its previous value.

### Limit reached before input termination

Given input bytes 65, 66, 67, zero and limit two:

- The destination header ends at two.
- Payload positions one and two contain 65 and 66.
- Input byte 67 is read and rejected because the header equals two.
- The later zero byte is not examined.
- Payload position three receives no write.

### Input readability at the limit boundary

Given limit one, a readable nonzero first input byte, and an inaccessible next input address:

- The first payload write and the corresponding header update occur.
- The next iteration attempts to read the inaccessible input address.
- A normal return is not established merely because the requested limit has been reached.

The environmental outcome of that invalid access is unspecified by the supplied material.

### Highest effective limit

Given limit 255 and at least 256 readable nonzero input bytes:

- Exactly 255 payload bytes are written.
- The header ends at 255.
- The 256th input byte is read but not copied.
- The count does not wrap on this execution path.

### Limit beyond the header range

Given limit 256, exactly 256 nonzero input bytes followed by zero, and sufficient destination storage:

- All 256 nonzero bytes are transferred.
- The header wraps to zero.
- The following input terminator stops execution.
- Payload positions one through 256 contain transferred data.

### Header aliases the input start

Given identical input and destination-header addresses:

- Initialization writes zero to that shared address.
- The first input read observes zero.
- No payload transfer occurs, regardless of the prior contents or limit.

## Uncertainties

### Established static evidence

The supplied representation directly supports:

- Unconditional header initialization before input access.
- Input examination before the limit check.
- Omission of the limit-check header read when input is zero.
- Equality-based stopping for nonzero input.
- Payload write before header increment.
- Separate header reads for comparison and increment.
- Forward progression of input and payload positions.
- Absence of explicit terminator insertion, validation, allocation, and function calls.

### Inferred intent

A one-byte length-prefixed output format and a bounded-copy purpose are plausible interpretations. Neither justifies treating large limits as safe bounds or assuming that downstream consumers interpret the final header correctly after wraparound.

### documented_only unknowns

The supplied material does not establish:

- Independent ABI confirmation of the represented integer widths.
- Actual allocation sizes or whether particular overlapping regions are permitted by the originating language’s object model.
- Behavior after invalid addressing or address-range exhaustion.
- Volatile-memory, device-memory, signal-handler, or concurrent-mutation semantics.
- Whether any caller restricts limits to 255 or less.
- Whether downstream consumers require a terminator or prohibit wrapped lengths.

These gaps are not filled by assuming validation elsewhere.

## Evidence

This contract is based solely on the supplied static representation of F-86ba47dd8d6f. No implementation access, runtime execution, or tests were used.

The behavioral basis is the visible ordering of byte accesses, the conditional header comparison, the one-byte stored count, and the absence of explicit calls or additional output writes. Intent and unverified environmental assumptions are separated from those observations above.
