<!-- Trusted provenance: F-5a9d10def27f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5a9d10def27f

## Purpose

F-5a9d10def27f transfers a bounded number of bytes from a one-byte-length-prefixed source into a destination and appends a zero byte. It returns the number of payload bytes transferred, excluding the appended terminator.

For a stable, readable source header, the transfer count is the smaller of:

- The unsigned 8-bit value at the beginning of the source.
- The caller’s unsigned 32-bit limit.

The operation is byte-oriented. It does not search for an existing zero byte, interpret text encoding, or stop when a payload byte is zero. The appended terminator does not imply that the copied payload contains no earlier zero bytes.

The apparent intent is conversion from a length-prefixed representation to a bounded, zero-terminated representation. That intent is inferred; the established behavior is the ordered byte transfer and final zero write described below.

## Interface

### Inputs

The function takes three inputs:

| Input | Representation | Meaning |
|---|---|---|
| Source | Pointer to 8-bit bytes | Addresses the length header, followed immediately by payload bytes. |
| Destination | Signed 32-bit integer used as an address | Identifies the address where the first copied byte is written. Its representation is an address carrier, not a destination length. |
| Limit | Unsigned 32-bit integer | Maximum number of payload bytes to transfer. It does not include space for the appended zero. |

The source layout is:

| Offset from source | Contents |
|---|---|
| 0 | Unsigned 8-bit payload-length header |
| 1 onward | Payload bytes |

There is no separate source-size input and no destination-capacity input.

### Output

The return value is an unsigned 32-bit count of copied payload bytes. Under a stable source header, it is between 0 and 255 inclusive.

The appended zero is not included in the return value. Consequently, a return value of 255 corresponds to 256 destination byte writes: 255 payload writes and one terminator write.

### Preconditions for normal completion

For a stable header value and a resulting transfer count of \(n\):

- The source header must be readable, including when the limit is zero.
- The next \(n\) source bytes must be readable when their respective reads occur.
- The destination must provide \(n+1\) writable bytes at the addresses used by the operation.
- Address calculations must resolve to accessible memory on the execution platform.
- If ordinary copying of the original payload is required, destination writes must not change source bytes before those bytes are read.

The function does not establish these conditions itself. In particular, the limit is a payload bound, not a total destination-capacity bound.

If the destination allocation has capacity \(c\), a caller seeking to avoid writing beyond that allocation needs room for the terminator as well as the payload. A zero-capacity destination is insufficient even when no payload bytes are copied.

## Behavior

### Transfer-count selection

The source header is interpreted as an unsigned byte, so values from 128 through 255 are positive lengths rather than negative values.

The supplied representation first obtains a header value and then uses a header access in the comparison with the unsigned limit. If the limit is smaller, the selected count becomes the limit. Otherwise, the previously obtained header value remains the selected count.

When the header remains unchanged during this selection, the count is exactly the minimum of the header and the limit. That stable-header case defines the ordinary contract.

The header is consulted before any destination write. Subsequent destination writes do not cause the transfer count to be recalculated.

### Payload transfer

Payload bytes are processed in increasing source-offset order, beginning at source offset 1. Each byte is read and then written to the corresponding increasing destination offset, beginning at destination offset 0.

The visible sequence for each selected payload position is:

1. Read that position’s source payload byte.
2. Write that byte to the corresponding destination position.
3. Advance to the next position.

The evidence shows a forward, byte-at-a-time transfer rather than an initial snapshot of the entire payload. This distinction matters when source and destination overlap.

Exactly \(n\) payload bytes are processed on a normally completing call. There is no content-dependent early exit. A zero-valued payload byte is copied like any other byte, and later selected bytes are still read and written.

### Terminator and return

After the last payload write, the function writes an 8-bit zero at destination offset \(n\). It then returns \(n\).

When \(n=0\):

- No payload byte is read.
- A zero is still written at destination offset 0.
- The return value is zero.

No destination location beyond offset \(n\) is explicitly written by this function’s visible body. Such locations retain their previous contents unless some separate mechanism changes them.

### Overlapping storage

There is no overlap check and no temporary payload buffer.

If a destination write changes a source byte that has not yet been read, the later read uses the changed value. Thus, overlap can produce repeated or otherwise altered payload bytes instead of a copy of the original payload.

A destination beginning at the source header is a useful special case: the payload is shifted toward lower addresses, and overwriting the header does not change the already selected count. This case still requires all relevant reads and writes to be valid.

The final terminator write may also change source storage when the regions overlap. It occurs after all selected payload reads.

### Widths and address arithmetic

The header is 8 bits. The limit, selected count, loop progression, and returned count are represented as unsigned 32-bit quantities.

With a stable header, the selected count cannot exceed 255. Therefore:

- Payload destination offsets range from 0 through at most 254.
- Payload source offsets range from 1 through at most 255.
- The terminator destination offset is at most 255.
- The number of required destination bytes is at most 256.
- The loop count does not approach unsigned 32-bit overflow.

The destination address carrier is signed 32-bit, while destination offsets are unsigned 32-bit. The supplied representation therefore exposes 32-bit unsigned address addition for destination accesses. It contains no overflow or address-wrap check.

This is not evidence that wrapped addresses are usable. Successful execution still depends on the platform’s address mapping and memory-access rules. A negative destination integer is not explicitly rejected, but neither is it established to identify valid writable memory.

## Side effects

On normal completion, the function explicitly performs:

- Header reads used to select the transfer count.
- One source payload-byte read for each selected position.
- One destination payload-byte write for each selected position.
- One final destination zero-byte write.

The function does not explicitly modify the source through the source pointer. Nevertheless, source storage can be changed through destination writes if the two regions alias.

There is no visible allocation, deallocation, ownership transfer, external call, global-variable access, logging, or synchronization operation. The source and destination addresses are used directly; the body does not visibly retain them for later use.

The operation is not atomic and does not provide rollback. If a memory access fails partway through execution, earlier destination writes may already have occurred. The supplied evidence does not define an exception type, signal, error return, or recovery action for such a failure.

A returned zero is an ordinary successful count for an empty or fully limited transfer. It is not a distinct error indicator.

## Dependencies

No dependency invocation appears in the supplied body of F-5a9d10def27f. No additional opaque function IDs are needed to describe its visible behavior.

Execution nevertheless depends on platform facilities that are not specified by this body:

- Readable source memory and writable destination memory.
- Interpretation of the 32-bit destination address.
- Memory-fault behavior.
- Rules governing concurrent access or access to special memory.

No assumption about an unknown callee preserving globals or owning memory is necessary because the supplied body shows no callees.

## Acceptance examples

These examples are conditional static expectations, not executed test results. Unless stated otherwise, they assume stable ordinary memory, valid address calculations, sufficient accessible storage, and non-overlapping source and destination.

### Full transfer

Given a header of 3, payload bytes `41 42 43`, and a limit of 3 or greater:

- Destination offsets 0 through 3 become `41 42 43 00`.
- The return value is 3.
- Destination bytes after offset 3 are not explicitly changed.

### Limit truncation

Given a header of 5, payload bytes `10 20 30 40 50`, and a limit of 2:

- Only payload bytes `10 20` are read and copied.
- Destination offsets 0 through 2 become `10 20 00`.
- The return value is 2.
- The remaining three advertised payload bytes need not be accessed by this call.

### Zero limit

Given a readable header of 200 and a limit of zero:

- No payload read occurs.
- Destination offset 0 is written with zero.
- The return value is zero.

This case still requires a readable header and a writable destination byte.

### Empty source

Given a header of zero and any limit:

- No payload read occurs.
- Destination offset 0 becomes zero.
- The return value is zero.

### Embedded zero byte

Given a header of 3, payload bytes `41 00 42`, and a limit of at least 3:

- Destination offsets 0 through 3 become `41 00 42 00`.
- The return value is 3.

A consumer interpreting the result as a conventional zero-terminated string may observe only the initial `41`, but this function still transfers all three selected payload bytes.

### Maximum header value

Given a header of 255 and a limit of 255 or greater:

- Exactly 255 payload bytes are copied.
- Destination offset 255 receives the appended zero.
- The return value is 255.
- At least 256 writable destination bytes are required.

Increasing the limit above 255 does not increase the transfer count for a stable header.

### Destination begins at the header

Given source storage beginning `03 41 42 43` and a destination equal to the source address, with a limit of at least 3:

- The selected count is 3 before copying begins.
- The first four storage bytes become `41 42 43 00`.
- The return value is 3.

The overwritten header does not shorten or extend this call.

### Forward-overlap propagation

Given source storage beginning `03 41 42 43`, sufficient additional writable storage, a destination two bytes after the source address, and a limit of at least 3:

- The first payload read obtains `41`.
- Its destination write replaces the next unread payload byte.
- Subsequent reads consequently obtain propagated `41` values.
- Destination offsets 0 through 3 become `41 41 41 00`.
- The return value remains 3.

This distinguishes the visible forward transfer from snapshot-based copying.

## Uncertainties

The following remain **documented_only**:

- The original declared interface and calling convention are not independently established. The contract uses the widths and address usage visible in the supplied representation.
- The exact physical number of header loads is not established. The representation contains an initial header access and another header access in the comparison; machine-level load reuse or optimization is not shown.
- No coherent concurrent-update contract is established. If the header changes between the represented accesses, the selected count need not equal the minimum of the limit and any single stable header value.
- Device-memory semantics, volatile-access guarantees, and inter-thread ordering are not established.
- Invalid-memory behavior is platform-dependent and unspecified here. Normal return is not promised after an inaccessible read or write.
- There is no evidence that the source actually contains as many payload bytes as its header advertises.
- Text encoding, intended consumer interpretation, and application-level ownership rules are unknown.
- Arithmetic wrap is exposed by the destination-address expression, but valid access across an address-space boundary is not established.

## Evidence

Established static evidence consists solely of the supplied body of F-5a9d10def27f: unsigned-byte header handling, unsigned limit comparison, an increasing forward byte-transfer loop, an unconditional final zero write, and return of the selected count.

The length-prefixed-to-zero-terminated conversion purpose is inferred from those operations. Platform behavior, concurrency guarantees, and application-level intent are not independently verified.

No runtime tests have been performed. The acceptance examples are deductions from the supplied static evidence under their stated preconditions.
