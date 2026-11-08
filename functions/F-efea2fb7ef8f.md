<!-- Trusted provenance: F-efea2fb7ef8f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-efea2fb7ef8f

## Purpose

F-efea2fb7ef8f performs a stateful, byte-by-byte transformation of a buffer obtained from an opaque object. It modifies that buffer in place and returns no value.

A zero mode selects an additive transformation. Any nonzero mode selects a subtractive transformation. Both modes combine two lookup tables, a position selector, a shifting portion of object-associated flags, and feedback from the preceding byte. The initial feedback byte is fixed at hexadecimal 23.

The two modes are consistent with complementary encoding and decoding operations when applied with matching metadata and stable lookup tables. That interpretation is inferred intent; the established behavior is the arithmetic and state progression specified below.

A significant boundary is the eight-bit processed-byte counter: a reported length of 256 or more cannot be reached by that counter. Such a length does not produce an ordinary finite traversal of the reported buffer.

## Interface

### Inputs

The function receives two machine-word arguments:

- **Object reference:** represented as a 32-bit integer in the supplied interface. Zero is rejected. Nonzero values are passed unchanged to four dependencies. The object’s layout and reference-validity rules are not established.
- **Mode:** represented as a 32-bit integer. Exactly zero selects addition; every other bit pattern selects subtraction. Its magnitude and sign otherwise have no effect.

For a nonzero object reference, dependencies supply:

- A buffer address.
- A 32-bit unsigned flags value.
- A 32-bit unsigned length.
- A 32-bit starting-position value, presented as signed by the supplied interface.

The buffer is accessed in units of unsigned eight-bit bytes. The transformation does not use multibyte buffer fields, terminators, or a stride greater than one byte.

### Output

There is no explicit return value. Successful completion is observable through changes to the buffer and any side effects of the dependencies.

There is no returned byte count, status code, newly allocated buffer, or output length.

### Required environment

For defined, finite completion:

- The object reference must be accepted by all four metadata dependencies.
- The returned buffer address must be nonzero, even when the reported length is zero.
- The reported length must be at most 255.
- Every processed buffer byte must be readable and writable.
- Both lookup-table bases and their selected byte entries must be readable whenever used.
- Relevant state must remain valid throughout execution.

The wrapper performs no explicit capacity, table-length, address-range, or metadata-consistency checks.

## Behavior

### Entry and metadata acquisition

A zero object reference causes an immediate call to F-a817d8d58c10. None of the four metadata dependencies is called on that path.

For a nonzero object reference, the function calls these dependencies in the following order:

1. F-656bfc1f57ae, obtaining the buffer address.
2. F-da62c29c744d, obtaining flags.
3. F-4e7604290e11, obtaining length.
4. F-594a8bb0474f, obtaining the starting position.

Each receives the same original object reference.

The returned buffer address is checked only after all four calls have completed. Consequently, a null buffer result does not prevent the remaining three dependency calls.

If the buffer address is zero, F-a817d8d58c10 is called. The supplied evidence identifies this dependency as nonreturning. It does not establish whether that means process termination, an exception-like transfer, or another mechanism.

The four results are obtained sequentially, not as an atomic snapshot. No assumption is made that one dependency preserves global state or leaves the object unchanged for the next dependency.

### Flag-derived selectors

Two independent three-bit quantities are assembled from the flags:

| Flags mask | Contribution to initial position offset | Contribution to initial shift |
|---|---:|---:|
| Hexadecimal 0080 | 4 | 0 |
| Hexadecimal 2000 | 2 | 0 |
| Hexadecimal 0400 | 1 | 0 |
| Hexadecimal 1000 | 0 | 4 |
| Hexadecimal 0008 | 0 | 2 |
| Hexadecimal 8000 | 0 | 1 |

Contributions within each column are combined as distinct bits. Each result is therefore between zero and seven.

The initial position state is the low eight bits of the sum of the returned starting position and the initial position offset. The addition has 32-bit unsigned semantics before truncation; a negative signed starting-position result does not cause a signed-overflow requirement here.

A separate shifting state begins as the low 16 bits of the flags, logically shifted right by the initial shift. Its associated shift counter begins at that same initial shift.

The initial processed-byte counter is zero. The initial feedback byte is hexadecimal 23.

Flags above bit 15 do not affect the wrapper’s transformation after metadata acquisition.

### Lookup layout and per-byte transformation

The function uses two externally supplied table bases:

- **Table A:** selected using a four-bit position index.
- **Table B:** selected using a four-bit index derived from Table A and the shifting state.

Each selected entry is read as one unsigned byte. Each table therefore requires readable entries at offsets zero through 15 for arbitrary supported metadata.

For every iteration whose processed-byte counter is less than the returned length:

1. The current position index is the low four bits of the position state.
2. The position state becomes that index plus one. Thus, although the initial state retains eight bits, table selection uses only four; the effective position sequence repeats every 16 iterations.
3. The byte at the selected Table A position is read.
4. The Table B index is the low four bits of the sum of that byte and the current shifting state.
5. The selected Table B byte is read and combined with the feedback byte using bitwise exclusive-or. This produces an eight-bit transformation quantity.
6. The current buffer byte is read.
7. The processed-byte counter advances modulo 256.
8. The output byte is determined according to mode:
   - **Zero mode:** the output is the low eight bits of the current buffer byte plus the transformation quantity. The output byte becomes the next feedback byte.
   - **Nonzero mode:** the output is the low eight bits of the current buffer byte minus the transformation quantity. The original, pre-write buffer byte becomes the next feedback byte.
9. The output byte is written to the same buffer address.
10. The buffer position advances by one byte.
11. The shift counter and shifting state advance as described below.

Addition and subtraction wrap modulo 256. Negative subtraction results do not generate an error or saturate at zero. Addition does not saturate at 255.

The Table A byte need not itself be limited to four bits: the final Table B index is masked to four bits after addition.

### Shift progression

After each processed byte, the function tests the shift counter’s value from that iteration.

- If that value is at most 11, the counter advances by one and the shifting state is logically shifted right by one.
- If that value is greater than 11, the counter is reset to zero and the shifting state is restored to the original flags’ low 16 bits.

The iteration using shift counter 12 is processed before the reset. The reset is not performed immediately upon reaching 12.

For an initial shift of \(s\), the first segment uses logical flag shifts \(s, s+1, \ldots, 12\). Later segments use shifts zero through 12 repeatedly. The first segment therefore contains \(13-s\) iterations; later segments contain 13.

The reset uses the flags value already returned by F-da62c29c744d. It does not call that dependency again.

### Length and termination

For a reported length from zero through 255, the function performs exactly that many iterations, assuming all accesses and dependencies complete normally.

A zero length still requires all four metadata calls and a nonzero buffer address. It causes no lookup-table entry reads and no buffer-byte reads or writes by this wrapper.

For any reported length from 256 through the maximum 32-bit unsigned value, the loop condition remains true indefinitely: the processed-byte counter ranges only from zero through 255. The buffer address nevertheless continues advancing rather than restarting when that counter wraps.

Accordingly, a length of 256 does **not** mean “process 256 bytes and return.” Execution continues past that point, subject to memory faults, address-space limitations, or other external disruption. No bounds-recovery behavior is established.

## Side effects

The wrapper’s direct data modification is an in-place write of each processed buffer byte. It does not explicitly allocate, resize, release, or replace the buffer.

No ownership transfer is established. In particular, the buffer-returning dependency is not proven to allocate memory, lend storage, transfer ownership, or require release. The wrapper contains no explicit release operation.

The metadata and failure dependencies may have additional side effects. Their contracts are unavailable; this specification does not assume that they preserve globals, object state, or unrelated memory.

Within each iteration, the selected Table A entry is read before the selected Table B entry, and the selected Table B entry is read before the current buffer byte. The buffer byte is read before its replacement is written. Each iteration completes its buffer write before the next iteration’s lookup.

The table bases come from global state. No immutable-table or non-aliasing guarantee is established. If a buffer write changes table contents, or memory determining a later table base, later lookups can differ. Descriptions relying on fixed tables require that stability as a precondition.

The source establishes no concurrency control. Results under concurrent mutation of the object, table state, or buffer are not specified by the available evidence.

The wrapper supplies no rollback. If an access or dependency fails after earlier writes, those writes have already occurred.

## Dependencies

| Opaque function ID | Established role in this wrapper | Unknown contract details |
|---|---|---|
| F-656bfc1f57ae | Receives the object reference; supplies the buffer address. Called first for a nonzero reference. | Storage ownership, capacity, lifetime, allocation behavior, and side effects. |
| F-da62c29c744d | Receives the object reference; supplies a 32-bit flags value. Called second. | Meaning of the flags outside their local arithmetic use; side effects. |
| F-4e7604290e11 | Receives the object reference; supplies the unsigned 32-bit loop bound. Called third. | Whether the value is constrained to 255 or represents accessible capacity. |
| F-594a8bb0474f | Receives the object reference; supplies the starting-position value. Called fourth. | Domain meaning, range restrictions, and side effects. |
| F-a817d8d58c10 | Invoked for a zero object reference or zero returned buffer address; identified as nonreturning. | Termination mechanism, diagnostics, cleanup, and effects on external state. |

The two lookup-table bases are additional global data dependencies. Their contents, initialization, ownership, and mutability are not supplied.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

### Empty valid buffer

Given a nonzero object reference, a nonzero returned buffer address, and a reported length of zero:

- All four metadata dependencies are called in the specified order.
- The failure dependency is not called by this wrapper.
- No table entries or buffer bytes are accessed by the transformation.
- The function returns without transforming data.

### Zero object reference

Given a zero object reference:

- F-a817d8d58c10 is invoked.
- None of the four metadata dependencies is invoked.
- No normal return is established.

### Null buffer with otherwise readable metadata

Given a nonzero object reference whose first dependency returns zero:

- The flags, length, and starting-position dependencies are still called.
- F-a817d8d58c10 is then invoked.
- A zero length does not bypass this failure path.

### Additive and subtractive feedback

Assume both tables have zero in every reachable entry, the reported length is three, and storage is valid and non-aliasing.

For zero mode and input bytes 1, 2, 3, the resulting bytes are 36, 38, 41. The initial transformation quantity is 35; each later quantity is the previous output byte.

For any nonzero mode and input bytes 36, 38, 41 under the same metadata and tables, the resulting bytes are 1, 2, 3. Feedback uses the original input bytes, not the newly written subtraction results.

### Byte wraparound

Assume the selected Table B byte is zero on the first iteration.

- In zero mode, a first input byte of 250 becomes 29.
- In nonzero mode, a first input byte of 10 becomes 231.

Both follow modulo-256 arithmetic with initial feedback 35.

### Position wraparound

Given an initial position whose low four bits are 15 and an initial position offset of zero, the first three Table A indices are 15, 0, and 1.

The upper bits of the initial position do not extend the table traversal beyond 16 entries.

### Shift reset boundary

Given an initial shift of seven, the first six iterations use flag shifts seven through 12. The seventh iteration uses shift zero, provided the reported length reaches that iteration.

### Counter-width boundary

Given otherwise valid state:

- Length 255 causes exactly 255 transformations and then normal return.
- Length 256 does not terminate after 256 transformations. The processed-byte counter wraps to zero, while traversal continues at the next buffer address.

## Uncertainties

The following are **documented_only**:

- The object’s representation and the semantic names of its metadata.
- Whether an upstream invariant restricts length to 255.
- The table values and their initialization lifecycle.
- Whether buffers and table-related storage can alias.
- Whether dependencies mutate global state or manage memory.
- The failure dependency’s exact nonreturning behavior.
- The consequences of invalid addresses, concurrent mutation, or continuing traversal beyond valid storage.
- Whether the transformation is intended for obfuscation, compatibility, encoding, or another purpose.

Complementary encoding and decoding is inferred from the opposite arithmetic directions and feedback choices. It is not evidence of cryptographic strength or a guarantee of reversibility under changing metadata or tables.

## Evidence

This contract is based solely on the supplied static function body and dependency-ID mapping.

The visible evidence establishes dependency-call order, flag masks, byte accesses, table indexing, feedback selection, modular arithmetic, shift reset timing, null handling, and the eight-bit counter limitation.

No dependency implementations, table contents, object layout, callers, or runtime observations were provided. No runtime tests have been performed.
