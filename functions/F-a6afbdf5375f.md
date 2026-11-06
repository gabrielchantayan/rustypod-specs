<!-- Trusted provenance: F-a6afbdf5375f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a6afbdf5375f

## Purpose

F-a6afbdf5375f selects between an indirect object operation for absent or initially empty input and a byte-producing operation for initially nonempty input.

On the byte-producing path, it obtains a destination pointer through another indirect object operation, then encodes successive nonzero unsigned 16-bit input units into one, two, or three bytes. It appends one zero byte when it encounters a zero input unit.

For input units outside the surrogate range, the byte patterns agree with UTF-8 encodings of the corresponding Basic Multilingual Plane values. The routine does not combine surrogate pairs, reject isolated surrogates, or produce four-byte encodings. Describing it as a fully validating UTF-16-to-UTF-8 converter would therefore exceed the evidence.

The meaning of the two object operations, the result produced by F-c04a2c7dc225, and ownership of the returned destination are not established.

## Interface

**Inputs**

- **Receiver:** an object pointer whose first field is used to obtain an indirect-dispatch table.
- **Input:** either a null pointer or a pointer to a sequence of unsigned 16-bit units, terminated by a zero unit.

**Output**

There is no explicit result value. Observable results consist of indirect calls and, when a non-null destination is obtained, writes through that destination.

**Relevant layout**

The supplied representation establishes these dispatch accesses:

| Receiver-relative structure | Use |
|---|---|
| First field of the receiver | Pointer to the dispatch table |
| Dispatch-table byte offset 8 | Operation called with the receiver, the result of F-c04a2c7dc225, and a zero argument |
| Dispatch-table byte offset 12 | Operation called with the receiver alone |

The representation uses four-byte table entries and a 32-bit result from F-c04a2c7dc225. These are properties of the supplied static representation, not a portable application binary interface established for other targets.

The operation at offset 8 returns a pointer interpreted as byte-addressable writable storage. The meaning of any result from the operation at offset 12 is not used by this routine.

**Caller-side preconditions for defined, completed execution**

- The receiver and the selected dispatch entry must be valid when accessed.
- A non-null input pointer must permit the initial 16-bit read.
- If encoding proceeds, each subsequently reached input unit must be readable until a zero unit is encountered.
- A non-null destination must support every byte write performed, including the final zero byte.
- Pointer advancement and accesses must remain valid for the relevant storage.
- Interactions with called operations must leave sufficient valid state for any subsequent accesses.

These requirements are not checked by the routine. In particular, the interface supplies no explicit input bound or destination capacity.

## Behavior

### Initial selection

The input pointer is checked for nullness.

If it is null, the routine invokes the receiver operation at dispatch-table offset 12 with the receiver as its sole explicit argument, then returns. It does not read an input unit, call F-c04a2c7dc225, or perform its own destination writes on this path.

If the input pointer is non-null, the routine reads its first unsigned 16-bit unit.

- If that unit is zero, it invokes the same offset-12 operation and returns.
- If that unit is nonzero, it continues to destination acquisition.

The initial read tests only whether the input begins with zero. It does not validate the remaining sequence, determine its length, or establish that the sequence remains unchanged.

The offset-12 operation must not be described as “clear,” “reset,” “release,” or any other semantic action without additional evidence. Only its invocation is established.

### Destination acquisition

For initially nonempty input, the routine calls F-c04a2c7dc225 with the original input pointer.

It then obtains the receiver’s dispatch table and invokes the operation at byte offset 8 with these arguments, in order:

1. The receiver.
2. The 32-bit result returned by F-c04a2c7dc225.
3. Zero.

The helper result is forwarded without a visible adjustment. Its interpretation as a byte count, capacity, character count, or other quantity is unknown. In particular, the routine does not visibly add space for a terminator.

The dispatch lookup for this operation occurs after F-c04a2c7dc225 returns. The contract must not assume that the helper preserves the receiver, its dispatch state, input contents, or unrelated global state.

If the offset-8 operation returns a null pointer, the routine returns immediately. It performs no encoding-loop reads and no direct destination writes on that path. It does not invoke the offset-12 operation as a fallback and does not report an explicit error value.

A non-null result becomes the starting destination address.

### Encoding

After both calls return, the routine begins reading input again from the original starting address. Consequently, the first unit read for encoding is a new read, not a reuse of the initial nonempty-test value.

Each iteration reads one unsigned 16-bit unit. A zero unit ends the loop. A nonzero unit is encoded as follows, where \(u\) is its unsigned value:

| Input range | Bytes emitted, in address order |
|---|---|
| 1 through 127 | \(u\) |
| 128 through 2047 | \(192 + \lfloor u/64 \rfloor\), then \(128 + (u \bmod 64)\) |
| 2048 through 65535 | \(224 + \lfloor u/4096 \rfloor\), then \(128 + (\lfloor u/64 \rfloor \bmod 64)\), then \(128 + (u \bmod 64)\) |

All emitted values fit in eight bits:

- One-byte output ranges from 1 through 127.
- Two-byte leading bytes range from 194 through 223.
- Three-byte leading bytes range from 224 through 239.
- Continuation bytes range from 128 through 191.

For each nonzero unit, the routine writes all bytes for that unit before advancing to and reading the next input unit. Within a multibyte encoding, it writes the leading byte first, followed by continuation bytes in increasing destination-address order.

After a unit is emitted, the input position advances by one 16-bit unit and the destination position advances by that unit’s encoded length.

There is no separate character-validity check. Every nonzero 16-bit value follows one of the three ranges above.

### Termination and output extent

When the encoding loop reads a zero unit, it writes one zero byte at the current destination position and returns.

For a stable sequence of \(n\) nonzero units followed by zero, let \(n_1\), \(n_2\), and \(n_3\) be the numbers of units in the one-, two-, and three-byte ranges. The routine writes

\[
L = n_1 + 2n_2 + 3n_3
\]

payload bytes and one terminator byte. Thus the required writable extent is \(L+1\) bytes.

This is a mathematical description of the writes, not a capacity calculation visibly performed by this routine. No local bound check, overflow check, truncation, or destination-growth request is established.

No destination bytes beyond the final zero byte are written directly by the routine.

### Encoding boundaries and special values

- A zero unit is a terminator, not an embedded character to encode.
- Values 127 and 128 lie on the one-/two-byte boundary.
- Values 2047 and 2048 lie on the two-/three-byte boundary.
- Surrogate-range units, 55296 through 57343, are each encoded independently as three bytes.
- A surrogate pair therefore produces six bytes, not a four-byte supplementary-character encoding.
- The value 65535 is accepted by the same three-byte path.
- A leading byte-order-mark value is encoded like any other unit in its range; it is not specially skipped.
- No byte-order conversion is visible. Reads interpret memory as unsigned 16-bit units in the execution environment’s representation.

## Side effects

**Direct effects**

The routine reads receiver dispatch state, reads input units, invokes the selected dependencies, and conditionally writes encoded bytes and a trailing zero through the returned destination.

It does not explicitly assign to the receiver’s fields or to the input sequence. This does not establish that either remains unchanged:

- Called operations may modify accessible state.
- Destination storage may alias the receiver, input, or other memory.
- Destination writes can therefore affect later accesses if storage overlaps.

The routine does not copy the complete input before writing. With overlapping input and destination storage, a write may change a unit that has not yet been read, including a later terminator. The resulting behavior cannot generally be derived from the input’s entry-time contents alone.

**Ownership and lifetime**

No direct allocation or deallocation operation is visible. The offset-8 operation may acquire, expose, replace, or otherwise manage storage, but none of those interpretations is established.

The routine does not explicitly release the returned destination and does not return it to its caller. Neither fact establishes who owns that storage or how long it remains valid.

Likewise, passing the input pointer to F-c04a2c7dc225 does not establish whether the helper retains it, modifies it, or affects its lifetime. Continued reads require valid input storage afterward.

**Errors**

A null destination return causes a silent early return from this routine. It does not establish failure atomicity: the preceding calls may already have changed state.

Invalid pointers, insufficient destination space, missing termination, and invalid dispatch state have no visible recovery path. The evidence does not establish a specific exception, signal, error code, or diagnostic for such conditions.

## Dependencies

### F-c04a2c7dc225

Established interface at this call site:

- Receives the original non-null input pointer.
- Is called only after the first input unit was observed to be nonzero.
- Returns a 32-bit value.
- That value becomes the second argument of the offset-8 receiver operation.

Its computation, units, termination rules, side effects, memory ownership, and failure conventions are **documented_only** unknowns. A sizing role is plausible intent, not established behavior.

### Receiver operation at dispatch-table offset 8

Established interface:

- Receives the receiver, the helper result, and zero.
- Returns a pointer interpreted as writable byte storage.
- A null result prevents all direct encoding writes.

No opaque function ID is supplied for the dynamic target. Its identity and storage-management semantics remain **documented_only**.

### Receiver operation at dispatch-table offset 12

Established interface:

- Receives the receiver.
- Is selected when the input pointer is null or its initially read first unit is zero.
- Is followed by return from F-a6afbdf5375f.

No opaque function ID is supplied for this dynamic target. Its semantic effect remains **documented_only**.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Encoding examples assume stable readable input, dependencies that permit continued execution, and a non-null, sufficiently large, non-overlapping destination. Byte values below are hexadecimal.

| Case | Required observation |
|---|---|
| Null input pointer | Invoke offset 12 once with the receiver; do not call F-c04a2c7dc225 or offset 8; perform no direct encoding writes. |
| First input unit is zero | Read that unit, invoke offset 12, then return without destination acquisition. |
| Initially nonempty input; offset 8 returns null | Call F-c04a2c7dc225, then offset 8 with its result and zero; return without loop reads or direct destination writes. |
| Input units 0041, 0000 | Write `41 00`. |
| Input units 007F, 0080, 07FF, 0800, 0000 | Write `7F C2 80 DF BF E0 A0 80 00`. |
| Input units FFFF, 0000 | Write `EF BF BF 00`. |
| Input units D800, 0000 | Write `ED A0 80 00`; do not substitute a replacement character. |
| Input units D83D, DE00, 0000 | Write `ED A0 BD ED B8 80 00`; do not combine the pair into four bytes. |
| Input units FEFF, 0041, 0000 | Write `EF BB BF 41 00`; do not strip the leading value. |
| Input units 0041, 0000, 0042 | Write `41 00`; do not encode the unit after the terminator. |

Additional ordering cases:

- If a dependency changes the first input unit from nonzero to zero before encoding begins, and returns a valid destination, the routine writes only the final zero byte. It does not switch to the offset-12 path.
- If the helper returns a particular 32-bit value, the offset-8 operation receives that value unchanged; the routine does not visibly increment it for termination.
- For input 0800 followed by zero, the direct writes occur in this order: `E0`, `A0`, `80`, then `00`, at successive destination addresses.
- Under the stated storage assumptions, a sentinel immediately beyond the required destination extent remains untouched by this routine’s direct writes. Dependency effects are excluded from that assertion.

## Uncertainties

**Established static behavior**

The branch conditions, dependency-call order, dispatch offsets, input-unit width, byte formulas, null-destination return, and final terminator write are directly supported by the supplied representation.

**Inferred intent**

The structure suggests conversion of a zero-terminated 16-bit sequence into byte storage associated with the receiver. It also suggests that F-c04a2c7dc225 may determine a quantity needed by the destination operation. Neither interpretation establishes storage ownership, allocation policy, valid-Unicode handling, or sizing units.

**Documented-only unknowns**

- The receiver’s complete layout and dynamic type.
- The semantic identities of its two indirect operations.
- The helper’s exact result and side effects.
- Whether destination capacity includes termination space.
- Whether callees preserve input, receiver state, globals, or memory lifetimes.
- Whether overlapping storage is permitted by the surrounding interface.
- Behavior when pointer arithmetic or required memory accesses are invalid.
- External synchronization and concurrent mutation rules.

These unknowns must not be replaced with assumed guarantees.

## Evidence

This contract is based solely on the supplied static representation of F-a6afbdf5375f and the supplied opaque dependency mapping.

The representation supports the two dispatch paths, forwarding of the helper’s 32-bit result, repeated unsigned 16-bit reads, ordered one-/two-/three-byte writes, and trailing zero-byte write.

No runtime tests have been performed. The acceptance examples are derived from the visible branches and arithmetic; they are not observed execution results.
