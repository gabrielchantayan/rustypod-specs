<!-- Trusted provenance: F-ce6efdb4c19a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ce6efdb4c19a

## Purpose

F-ce6efdb4c19a collects a bounded sequence of byte values from a dependency-driven input source, terminates the collected sequence with a zero byte, and submits it to an ordered series of comparisons. The first comparison returning zero selects a fixed 32-bit result. If none returns zero, the function returns zero.

The established behavior is a bounded collection-and-classification operation. The selected results resemble character values used for markup or entity decoding, but that interpretation is **inferred intent**, not an established vocabulary or input format. The comparison objects’ contents are not supplied, so this contract does not assign names or spellings to accepted sequences.

This is a static, source-facing contract. No runtime tests have been performed.

## Interface

### Inputs

The function has four arguments:

| Argument | Established representation | Observable use |
|---|---|---|
| First | 32-bit address-bearing value | Identifies a structure or storage region containing a readable 32-bit field at byte offset 4. |
| Second | 32-bit value | No direct use in the supplied function. |
| Third | 32-bit value | Copied into the first local word before dependency calls. That word subsequently provides collection storage. |
| Fourth | 32-bit value | Copied into a second local word before dependency calls. |

No structure definition, allocation size, alignment guarantee, or validity check is provided for the first argument.

On every input-related dependency invocation, the function obtains the dependency argument by reading the 32-bit field at offset 4 from the first argument. That field is not cached across calls. A dependency that changes it can therefore affect which value is passed to a later invocation.

### Output

The function returns one 32-bit value from this set:

- `0x0000003C`
- `0x0000003E`
- `0x00000026`
- `0x00000027`
- `0x00000022`
- `0x000000A0`
- `0x0000000A`
- `0x00000000`

The first seven results correspond, in order, to seven comparisons returning zero. Zero is the fallback when every comparison returns a nonzero result. It is not a separately reported input error.

### Local collection layout

The collection area is byte-addressed despite being represented through word-sized local storage.

Its operational layout is:

| Byte offset from collection start | Use |
|---|---|
| 0–3 | Up to four collected bytes |
| Number of collected bytes | Mandatory zero-byte terminator |

Consequently, collecting four bytes requires a fifth writable byte at offset 4. The supplied representation also includes a second initialized local word, but it does not establish a portable source-language object layout or independently prove adjacency of those local words. The behavioral storage requirement is nevertheless clear: four payload bytes plus one terminator byte.

The function advances the collection position by **one byte**, not by one 32-bit word.

### Preconditions

Normal execution requires:

- A valid 32-bit read at offset 4 from the first argument whenever that field is reread.
- Valid operation of F-3b19cbd01e9c and F-2979ae55a86a for the values supplied to them.
- Writable local storage covering the collection and terminator writes.
- Valid comparison objects and a comparison dependency that accepts the pointers passed to it.

There are no explicit checks enforcing these preconditions. Violations have no defined error result in the supplied evidence.

## Behavior

### Initialization and first dependency calls

Before invoking a dependency, the function copies the third and fourth arguments into local 32-bit storage.

It then performs the following ordered actions:

1. Reads the 32-bit field at offset 4 from the first argument.
2. Calls F-3b19cbd01e9c with that field value.
3. Ignores that dependency’s return value.
4. Rereads the same field.
5. Calls F-2979ae55a86a with the newly read value.
6. Interprets the returned value as a signed 32-bit integer for the collection conditions.

The first call to F-2979ae55a86a occurs even if its result immediately prevents collection.

The meaning of F-3b19cbd01e9c is unknown. It must not be described as consuming a delimiter, initializing a reader, or advancing a cursor without additional evidence.

### Collection conditions

A returned value is stored only when all three conditions hold:

- It is not `0x0000003B`.
- Its **signed 32-bit value** is less than 128.
- Fewer than four bytes have already been stored.

The value `0x3B` is therefore a distinguished stopping value. Its conventional character interpretation is a semicolon, but the source establishes a numeric condition rather than a textual protocol.

The signed comparison is significant. Negative values satisfy “less than 128.” There is no lower-bound check and no explicit end-of-input check.

For each accepted value:

- The function writes its low eight bits to the current collection byte.
- It calls F-2979ae55a86a again, after rereading the field at offset 4.
- It advances the collection position by one byte.
- It reevaluates the collection conditions using the new returned value.

Thus, accepting a byte always causes another input-related dependency call before the next condition check.

### Stopping and read-ahead

Collection stops when:

- The latest returned value equals `0x3B`;
- The latest returned value is a signed value of at least 128; or
- Four bytes have already been stored.

The value that stops collection is not written into the collection area.

However, that value has already been returned by F-2979ae55a86a. The function does not request that it be restored or pushed back. Whether obtaining it consumes source state is a dependency property, not established here.

The four-byte limit also causes read-ahead. After storing the fourth byte, the function still calls F-2979ae55a86a once more. The resulting fifth value is not stored, regardless of its value.

If normal execution stores \(n\) bytes, F-2979ae55a86a has been called exactly \(n+1\) times, where \(0 \leq n \leq 4\). These counts describe calls, not guaranteed source consumption.

### Termination

After collection stops, the function writes a zero byte immediately after the collected bytes.

This write occurs even if no bytes were collected. In that case, byte offset 0 is set to zero.

If four bytes were collected, the terminator is written at byte offset 4. No fifth payload byte is stored.

A returned value whose low eight bits are zero can also create a zero byte *inside* the collected payload. The collection loop does not stop merely because a stored byte is zero. Whether such a byte truncates the effective input seen by a comparison depends on the comparison dependency’s semantics.

### Ordered comparisons and result selection

After writing the terminator, the function invokes an unidentified comparison dependency. An opaque function ID for this dependency was not supplied.

Each invocation receives:

- A pointer to the start of the local collection area.
- A pointer to one of seven fixed comparison objects.

The objects are tried in this order:

| Comparison position | Result if that comparison returns zero |
|---|---:|
| First | `0x0000003C` |
| Second | `0x0000003E` |
| Third | `0x00000026` |
| Fourth | `0x00000027` |
| Fifth | `0x00000022` |
| Sixth | `0x000000A0` |
| Seventh | `0x0000000A` |

A zero comparison result immediately selects the corresponding result. Later comparisons are not invoked. Any nonzero comparison result proceeds to the next comparison, without distinguishing positive from negative values.

If all seven results are nonzero, the function returns `0x00000000`.

This establishes **zero-result precedence**, not necessarily textual equality precedence. The comparison dependency’s actual semantics and the fixed objects’ contents are unknown.

The same local buffer pointer is passed at each comparison step. The function does not reconstruct or restore the buffer between calls. If an earlier comparison modifies the buffer or shared state, later comparisons may observe those changes.

## Side effects

### Directly established effects

The function directly performs:

- Local copies of the third and fourth arguments.
- Repeated 32-bit reads from offset 4 of the first argument.
- Between zero and four one-byte payload writes into local storage.
- One one-byte terminator write.
- One call to F-3b19cbd01e9c.
- Between one and five calls to F-2979ae55a86a.
- Between one and seven comparison calls, provided execution reaches that phase and each called dependency returns normally.

The supplied body contains no explicit heap allocation, deallocation, external-memory write, global assignment, or pushback operation. This observation does not constrain dependency side effects.

### Dependency-mediated effects

No supplied contract establishes whether any dependency:

- Advances input state;
- Modifies the field at offset 4;
- Changes global state;
- Performs I/O;
- Allocates or releases memory;
- Retains pointers;
- Fails to return normally.

Those effects must remain unknown. In particular, ignoring F-3b19cbd01e9c’s return value does not make that call side-effect-free.

### Ownership and lifetime

The function does not explicitly transfer ownership of caller-provided memory or release it.

The comparison dependency receives a pointer to local storage whose lifetime ends when this function returns. The supplied evidence does not establish a retention prohibition for that dependency. Any use of the pointer after return would therefore require an external lifetime contract not supplied here.

The function returns a scalar value, not the collection buffer or a pointer into it.

## Dependencies

| Dependency | Established role | Unknown contract |
|---|---|---|
| F-3b19cbd01e9c | Called once before collection, with a freshly read 32-bit field value; return ignored. | Purpose, state changes, ownership, failure behavior. |
| F-2979ae55a86a | Supplies values tested as signed 32-bit integers; called initially and after every stored byte. | Input consumption, end-of-input encoding, valid return range, state changes, failure behavior. |
| Comparison dependency; opaque ID unavailable | Called against up to seven fixed objects; zero selects the associated result. | Equality semantics, read bounds, mutation, ownership, retention, failure behavior. |

The seven fixed comparison objects are data dependencies, not identified functions. Their bytes, lengths, encoding, and termination are unavailable.

No extra opaque function identifier is invented for the unidentified dependency.

## Acceptance examples

These examples are conditional static consequences, not executed tests. Unless otherwise stated, dependencies return normally and do not modify the relevant collection bytes or field unexpectedly.

### Immediate stopping value

If the first F-2979ae55a86a result is `0x3B`:

- No payload byte is stored.
- The first collection byte is set to zero.
- F-2979ae55a86a is called once.
- Comparison begins against the resulting zero-terminated collection.
- The return value depends on the comparison outcomes; it is not automatically zero.

### Signed upper-bound rejection

If the first result is 128:

- No payload byte is stored.
- No second F-2979ae55a86a call occurs.
- Comparison proceeds after writing the terminator at offset 0.

The same collection outcome applies to any initial signed 32-bit value greater than 128.

### Negative returned value

If successive results are −1 and `0x3B`:

- −1 passes the signed upper-bound condition.
- Byte `0xFF` is stored at offset 0.
- The stopping value is not stored.
- A zero terminator is written at offset 1.
- F-2979ae55a86a is called twice.

This remains true even if −1 conventionally represents end-of-input elsewhere; no such special treatment appears here.

### Embedded zero byte

If successive results are 0, 65, and `0x3B`:

- The stored payload is byte values 0 and 65.
- The explicit terminator is written at offset 2.
- Three F-2979ae55a86a calls occur.

If the comparison dependency uses conventional zero-terminated string semantics, it would see an empty string. That interpretation is conditional, not established.

### Four-byte capacity and extra retrieval

If successive results are 65, 66, 67, 68, and 69:

- Offsets 0 through 3 receive the first four values.
- The fifth result is obtained but not stored.
- Offset 4 receives the zero terminator.
- No sixth F-2979ae55a86a call occurs.

This behavior does not require a stopping value or an out-of-range value to appear.

### Comparison precedence

If the first comparison is nonzero and the second is zero:

- The result is `0x0000003E`.
- Comparisons three through seven are not invoked.

If multiple comparison objects could produce zero, the earliest zero-producing comparison wins, subject to any state changes caused by earlier calls.

### Changed field between calls

If F-3b19cbd01e9c changes the field at offset 4 from one value to another:

- The first F-2979ae55a86a call receives the new field value.

Likewise, a field change during one collection call affects the next call’s argument because the field is reread.

## Uncertainties

The following are **documented_only** limitations:

- Accepted textual spellings cannot be determined without the seven comparison objects.
- The comparison dependency cannot be confirmed to implement string equality or to stop reading at a zero byte.
- The physical relationship between the two initialized local words is not independently established by the supplied representation.
- The purpose of the initial dependency call is unknown.
- Input consumption, cursor positioning, end-of-input handling, and recovery behavior are unknown.
- No dependency can be assumed to preserve globals, preserve caller-visible memory, or avoid memory ownership operations.
- There is no supplied contract for invalid addresses, dependency failure, reentrancy, or concurrent modification.
- The third and fourth arguments’ initial values are not explicitly used in result selection. Under conventional read-only, zero-terminated comparison semantics, they would not affect the collected string because each visible payload byte and its terminator are overwritten. Without that dependency contract, complete result independence is not established.

The apparent character-decoding purpose is inferred only. It does not justify assuming particular entity names, requiring a final delimiter, or rejecting negative input results.

## Evidence

Established static evidence supports:

- Four 32-bit arguments and a 32-bit scalar return.
- Local initialization from the third and fourth arguments.
- A fresh offset-4 field read before each input-related dependency call.
- Signed comparison against 128 and a distinct check against `0x3B`.
- One-byte stores and one-byte position increments.
- A maximum of four payload bytes followed by a zero-byte write.
- One additional retrieval after every stored byte, including the fourth.
- Seven ordered zero-result checks with the exact result mapping stated above.
- A zero fallback after all seven comparisons fail to return zero.

The supplied evidence does not include dependency implementations, comparison-object contents, caller contracts, or execution traces. No runtime behavior, memory-safety property, or accepted spelling has been experimentally verified.
