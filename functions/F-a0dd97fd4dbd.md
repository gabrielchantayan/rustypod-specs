<!-- Trusted provenance: F-a0dd97fd4dbd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a0dd97fd4dbd

## Purpose

F-a0dd97fd4dbd conditionally copies a counted byte sequence into storage obtained from F-6dd003795db7, replaces certain byte values with `0x3F`, and appends a zero byte.

The established behavior is a counted copy, not a scan for a string terminator. Embedded zero bytes do not stop processing. Whether copying occurs is determined solely by a private status cell after the dependency returns.

The character transformation requires particular care: under the ordinary integer-promotion semantics of the supplied source, bytes `0x00` through `0x7F` are preserved, and bytes `0x80` through `0xFF` become `0x3F`. The expression does not establish that control bytes below `0x20` are rejected.

The apparent intent is to produce a zero-terminated, restricted-byte representation. The destination’s ownership, persistence, and exposure to the caller are not established.

## Interface

The function has four positional inputs and no explicit return value.

| Input | Established use |
|---|---|
| First | An address used to access a descriptor containing a byte count and a source pointer. |
| Second | A 32-bit value forwarded unchanged as the first argument to F-6dd003795db7. Its meaning is unknown. |
| Third | A 32-bit value that the function does not use. |
| Fourth | An integer copied into a private status cell before that cell’s address is passed to F-6dd003795db7. |

### Descriptor layout

The first input identifies storage with these relevant fields:

| Offset from descriptor address | Field | Interpretation |
|---|---|---|
| `+0x08` | Unsigned 16-bit quantity | Number of source bytes to process. |
| `+0x10` | Byte pointer | Address from which source bytes are read. |

No use of other descriptor fields is shown. The evidence does not establish a complete descriptor size, a packing convention, or the validity of any unrelated fields.

The source-pointer field is read before the count field. Both values are captured before the dependency call. Later changes to the descriptor fields do not change these captured values.

### Widths and numerical bounds

The supplied representation uses an unsigned 16-bit count, an unsigned integer loop index and working count, 32-bit opaque arguments, and integer-valued address arithmetic. A 32-bit execution model is consistent with that representation, but the target ABI and pointer representation are not independently documented.

The source count ranges from 0 through 65,535. After widening, one is added to obtain the size argument passed to the dependency. That argument therefore ranges from 1 through 65,536; it does not wrap at the 16-bit boundary.

Each source and destination element is one byte. The appended terminator is also one byte.

### Preconditions for defined memory access

The function performs no explicit pointer, bounds, alignment, or allocation validation.

At entry, the two descriptor fields must be readable in the applicable execution environment. If the post-call status permits copying:

- For a positive count, the captured source address must permit all sequential byte reads.
- The dependency’s returned address must permit writing the counted bytes and one additional terminator byte.
- Address calculations must identify valid storage without invalid pointer or address wraparound.
- The dependency must not have invalidated storage needed for the subsequent accesses.

For a zero count, no source byte is read. A writable destination byte is nevertheless required if copying is enabled.

## Behavior

### Capturing inputs and invoking the dependency

The function first reads the source pointer from descriptor offset `+0x10`, then reads the unsigned 16-bit count from offset `+0x08`.

It initializes a private integer status cell from the fourth input and calls F-6dd003795db7 once. The six arguments, in order, are:

1. The second input, unchanged.
2. Integer constant `1`.
3. Integer constant `0`.
4. The captured count plus one.
5. Integer constant `0`.
6. The address of the private status cell.

The dependency’s return value is treated as the destination address.

The third input has no effect on these arguments or on subsequent local behavior. The function does not conditionally skip the dependency call based on the initial status, the count, or the source-pointer value.

### Status gate

After the dependency returns, the function checks the current value of the private status cell.

- If the value is exactly zero, the function performs the copy and writes a terminator.
- If the value is nonzero, the function performs no source-byte reads and no destination-byte writes of its own.

There is no separate validation of the returned destination address. A zero status enables destination access even if the returned address is unusable.

The fourth input supplies only the initial status value. If the dependency leaves that value unchanged, an initial zero enables copying and an initial nonzero value suppresses copying. If the dependency changes it, the post-call value controls the decision.

The status cell is private to this invocation. Its final value is not explicitly returned to the caller.

### Byte transformation

When the status is zero, processing proceeds from source offset zero through source offset count minus one, in ascending order.

For each offset, the function reads the source byte, determines its replacement if necessary, and writes the resulting byte to the corresponding destination offset before processing the next source byte.

Under the ordinary integer promotions of the supplied expression:

| Source byte | Destination byte |
|---|---|
| `0x00`–`0x7F` | Unchanged |
| `0x80`–`0xFF` | `0x3F` |

The comparison subtracts `0x20` from the promoted byte and asks whether the result exceeds `0x5F`. For values below `0x20`, the result is negative rather than a wrapped byte value, so those values are preserved. At `0x7F`, the result equals `0x5F`, which does not satisfy the strict comparison. At `0x80`, it exceeds that boundary.

Consequently, this behavior is not equivalent to replacing every byte outside printable ASCII. It preserves zero bytes, other low control values, and `0x7F`.

### Termination and zero-length input

After processing exactly the captured count of bytes, the function writes `0x00` at destination offset equal to that count.

An embedded source zero is copied as data. Processing continues through later counted bytes, and the final terminator is still appended.

For a zero count and zero post-call status:

- The dependency receives a size argument of one.
- No source byte is read.
- A single zero byte is written at the returned destination address.

For any nonzero post-call status, the function does not append a terminator.

### Aliasing and mutation during processing

The copy is forward and incremental. The function does not first snapshot the entire source sequence.

If source and destination overlap, earlier destination writes can affect later source reads. For example, when the destination begins one byte after the source, the first output byte overwrites the location that the next iteration will read. The behavior therefore cannot generally be specified as transformation of an immutable entry-time byte sequence.

The captured source address and count survive the dependency call as local values, but the source bytes are read only after that call. If the dependency changes the underlying source contents, the function processes the contents present when each byte is read. It does not restore or validate them.

## Side effects

The function’s own visible memory operations occur in this order:

1. Read the descriptor’s source-pointer field.
2. Read the descriptor’s count field.
3. Invoke F-6dd003795db7.
4. If the private status is zero, perform alternating source-byte reads and destination-byte writes in ascending offset order.
5. If the private status is zero, write the final destination zero byte.

On the enabled path, the function performs exactly the captured count of source-byte reads and count plus one destination-byte writes, excluding any operations performed by the dependency.

There are no direct assignments to the descriptor fields and no explicit global-variable accesses in the supplied body. That does not guarantee that descriptor storage or global state remains unchanged:

- The destination may alias descriptor storage or other caller-visible memory.
- F-6dd003795db7 may modify memory or global state.
- The dependency may invalidate storage on which later operations rely.

No explicit allocation, release, or ownership transfer is performed directly by this function. The dependency may have such behavior, but it is not established here. There is no explicit destination release and no returned destination pointer.

The private status cell is valid during the dependency call. The evidence does not establish that retaining its address beyond the call is supported.

There is no explicit retry, fallback destination, error return, or recovery operation. A nonzero status suppresses the local copy; it does not undo dependency side effects. Invalid memory accesses are not handled by this body.

## Dependencies

### F-6dd003795db7

This is the only explicit dependency.

Its established interface at this call site consists of six arguments: the forwarded second input, constants `1` and `0`, the captured count plus one, another constant `0`, and a writable reference to the private integer status cell.

The function consumes two results from this call:

- The returned integer-valued address, used as the destination when status is zero.
- The post-call contents of the status cell, used to enable or suppress copying.

The size argument and subsequent writes are consistent with requesting destination storage of sufficient capacity. That interpretation is inferred intent, not a proven dependency contract.

Unknown aspects include:

- Whether it allocates, selects, registers, or merely returns storage.
- Whether it guarantees capacity corresponding to the size argument.
- Whether it always writes the status cell.
- What particular nonzero status values mean.
- Whether it can return an invalid address while leaving status zero.
- What memory, global state, or ownership it changes.
- How the produced destination becomes available to other code.

No assumption that this dependency preserves globals or owns or releases any particular storage is justified by the supplied body.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They have not been executed. Unless otherwise stated, assume valid descriptor accesses, valid source and destination storage, no relevant dependency mutation, and no source/destination overlap.

### Boundary-byte transformation

Given a count of eight, source bytes:

`00 1F 20 3F 7E 7F 80 FF`

and a zero post-call status, the destination must contain:

`00 1F 20 3F 7E 7F 3F 3F 00`

The last byte is the appended terminator. The first zero does not stop copying.

### Empty sequence

Given a zero count and zero post-call status, the dependency must receive size argument one. The function must read no source bytes and must write one zero byte at the destination.

A source address unsuitable for dereferencing does not itself prevent this local path, because the zero-count path does not dereference it. The descriptor’s pointer field must still be readable.

### Status unchanged by the dependency

Given an initial fourth input of zero, if the dependency leaves the status cell unchanged, copying must occur.

Given an initial fourth input of any nonzero value, if the dependency leaves the cell unchanged, the function must perform no source-byte reads or destination-byte writes. The dependency must still have been called.

### Status changed by the dependency

Given an initially nonzero status that the dependency changes to zero, copying must occur.

Given an initially zero status that the dependency changes to nonzero, copying and terminator writing must both be suppressed.

### Maximum count

Given a count of 65,535 and zero post-call status, the dependency must receive size argument 65,536. The function must process source offsets 0 through 65,534 and write the terminator at destination offset 65,535.

This criterion requires valid storage and address arithmetic for the entire range.

### Forward overlap

Given three source bytes `41 42 43`, a destination beginning one byte after the source, and zero post-call status, the forward writes cause later reads to observe overwritten bytes. The three destination data bytes become `41 41 41`, followed by a zero terminator.

This example distinguishes the established incremental copy from a snapshot-based or overlap-safe copy.

### Descriptor changes during the dependency

If the dependency changes the descriptor’s count or source-pointer field, those new field values must not replace the values already captured by this function. If it instead changes bytes at the captured source address, subsequent reads observe those changed bytes.

## Uncertainties

The following remain **documented_only**:

- The target ABI, exact pointer width, alignment rules, and address-overflow behavior.
- The descriptor’s complete type, lifetime, and ownership.
- The meanings of the second and third inputs outside this function.
- The semantic contract and side effects of F-6dd003795db7.
- The destination’s ownership, lifetime, capacity guarantee, and retrieval mechanism.
- Behavior when preconditions fail or the dependency does not return normally.
- Whether preserving control bytes and `0x7F` matches the original design intent.

The transformation stated here follows the ordinary promotion semantics of the supplied source expression. No machine-level evidence or execution result was provided to independently validate that representation.

## Evidence

Established static evidence consists of the supplied function body and the provided mapping of its sole explicit dependency to F-6dd003795db7.

That evidence supports the descriptor offsets, read order, dependency argument order, private status gate, counted forward copy, byte comparison, and final zero-byte write.

The interpretation that the dependency provides output storage is inferred from how its return value and size argument are used. It is not established allocation or ownership behavior.

No runtime tests, compilation, dependency inspection, or external validation have been performed. The acceptance examples are specified outcomes under their stated conditions, not observed results.
