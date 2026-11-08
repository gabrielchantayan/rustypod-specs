<!-- Trusted provenance: F-de83fe44e03c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de83fe44e03c

## Purpose

F-de83fe44e03c prepares a buffered 64-byte block for final processing, invokes F-29cbb14d23e7 once or twice, and exports five 32-bit words as 20 output bytes. It then invokes an unresolved indirect dependency with three fixed arguments.

The padding and length layout are consistent with finalization of a block-based digest. That purpose is **inferred intent**, not an established algorithm identification. The supplied evidence does not establish which algorithm is implemented, where its complete state resides, or whether the exported words are associated with the supplied context.

This contract distinguishes behavior directly established by the supplied body from behavior conditional on unknown dependencies.

## Interface

### Inputs

The function receives two 32-bit values used as memory addresses:

- **Context address:** identifies storage containing a 64-byte block buffer and two adjacent 32-bit fields.
- **Output address:** identifies the destination for 20 byte stores.

No explicit length, capacity, ownership flag, or error-result argument is present.

The directly accessed context layout is:

| Byte offset | Size | Directly observed role |
|---|---:|---|
| 0–63 | 64 bytes | Buffered block, padding area, and encoded length |
| 64–67 | 4 bytes | Low count word |
| 68–71 | 4 bytes | High count word |

“Low count word” and “high count word” describe how the function constructs the length trailer. The supplied body does not establish how these fields were accumulated.

The known direct accesses require the context storage through offset 71 to be readable, and the 64-byte block area to be writable. F-29cbb14d23e7 may impose additional context-storage requirements that cannot be determined here.

The output must support 20 successive byte stores, subject to any aliasing effects described below.

### Return and errors

The function has no declared return value. No status is returned for padding, processing, or export.

There are no visible checks for null addresses, address validity, output capacity, initialization, or a finalized context. There is no locally implemented recovery path.

Normal completion requires both dependencies to return normally. Faults, exceptions, nonlocal exits, or termination inside dependencies are not handled by the supplied body.

### Address and arithmetic preconditions

The source representation uses 32-bit address-bearing arguments. A successful call requires all effective addresses used by the direct accesses and dependencies to designate valid storage under the execution environment.

The block index is reduced to six bits, so its value is always 0–63. The local index arithmetic used for padding and output does not overflow a 32-bit quantity. This does not establish that address addition is valid for arbitrary input addresses.

## Behavior

### 1. Select the padding position

The function first reads the 32-bit word at context offset 64. It uses only the lowest six bits to select a position in the 64-byte block.

Let **r** denote that selected position. Thus:

- r is between 0 and 63 inclusive.
- The initial count read determines the padding position for the remainder of the function.
- Later changes to the count fields do not cause the padding position to be recalculated.

The function writes byte value 128, hexadecimal 80, at block offset r. Bytes preceding r are not directly rewritten by this padding stage.

### 2. Choose one-block or two-block processing

The branch boundary is r = 56.

**When r is 0–55:**

- The function writes zero bytes from offset r + 1 through offset 55, inclusive.
- If r = 55, that range is empty.
- No dependency is invoked before length-trailer construction.

**When r is 56–63:**

- The function writes zero bytes from offset r + 1 through offset 63, inclusive.
- If r = 63, that range is empty.
- It calls F-29cbb14d23e7, passing the context address as both arguments.
- If that call returns, the function writes zero bytes to offsets 0–55, inclusive.

Each described zero-filled range is written in increasing offset order.

Consequently, F-29cbb14d23e7 is invoked once when r is 0–55 and twice when r is 56–63, assuming execution reaches all calls and each call returns.

The preliminary call in the two-block case occurs before the count fields are read for the trailer. Its effects therefore matter: the supplied body does not prove that it preserves those fields, other context storage, or global state.

### 3. Construct the eight-byte length trailer

After padding, the next block index is always 56. The function writes eight bytes at offsets 56–63 in increasing order.

The bytes are derived from the 32-bit fields at offsets 68 and 64. If **H** and **L** denote stable values of the high and low fields throughout trailer construction, the trailer is the big-endian encoding of:

**8 × (H × 2³² + L), reduced modulo 2⁶⁴.**

Equivalently, the eight stored bytes are:

| Block offset | Numeric byte value |
|---|---|
| 56 | Bits 21–28 of H |
| 57 | Bits 13–20 of H |
| 58 | Bits 5–12 of H |
| 59 | Low eight bits of H shifted left three, combined with the top three bits of L |
| 60 | Bits 21–28 of L |
| 61 | Bits 13–20 of L |
| 62 | Bits 5–12 of L |
| 63 | Low eight bits of L shifted left three |

All stored results are reduced to eight bits. Overflow beyond the 64-bit encoded result is discarded. In particular, the top three bits of H do not appear in the trailer.

This describes the visible fixed-width byte results; it does not require an implementation to reproduce signed-language shift expressions.

#### Count-read ordering

The function does not explicitly capture a single 64-bit count snapshot for trailer construction.

It separately reads:

- The high field for each of the first three trailer bytes.
- Both fields for the fourth trailer byte.
- The low field for each of the final four trailer bytes.

Thus trailer construction contains four high-field reads and five low-field reads, in addition to the initial low-field read used to select r.

For each trailer byte, its required reads precede its store. The supplied expression for the fourth byte does not establish the relative evaluation order of its high-field and low-field reads.

The single 64-bit length interpretation is therefore conditional on stable field values. If the fields change between reads, the function may construct a trailer that does not represent any single observed count value.

### 4. Process the final block

After writing all eight trailer bytes, the function calls F-29cbb14d23e7 with the context address as both arguments.

This call is unconditional once trailer construction completes. It is the only processing call in the one-block case and the second processing call in the two-block case.

Both arguments alias exactly. The supplied body does not expose the dependency’s return type, internal state layout, memory ownership, validation, or mutation behavior. No returned status is inspected here.

### 5. Export 20 bytes from a global source

After the final processing call returns, the function reads a pointer from a global pointer slot. That pointer, rather than an address visibly derived from the context, supplies the exported data.

The function consumes five successive 32-bit words from this source. For each word, it:

- Reads the complete 32-bit word.
- Writes its least-significant byte to the next output position.
- Writes the remaining three bytes in order of increasing significance.
- Advances to the next source word.

The output is therefore five little-endian 32-bit word encodings, concatenated into 20 bytes.

The global pointer is read once for this export phase. The five source words are read separately, one per output group, rather than all being captured before output begins.

There is no established connection between these source words and the context beyond whatever F-29cbb14d23e7 or other program state may arrange.

### 6. Invoke the indirect dependency

After all 20 output stores, the function invokes an unresolved indirect dependency through a global function-pointer slot.

Its arguments, in order, are:

1. The integer value 1.
2. The integer value 0.
3. The integer value 0.

The output export precedes this invocation. The body does not inspect a result from the indirect call. If the call returns normally, F-de83fe44e03c returns.

The meanings of these arguments and the indirect call’s purpose are unknown.

## Side effects

### Direct writes and state transitions

The body directly modifies:

- The selected padding byte within the context block.
- The applicable zero-filled block ranges.
- All eight trailer bytes.
- Twenty bytes starting at the output address.

It does not directly increment, clear, or otherwise assign either count field.

That observation is not a guarantee that the count fields remain unchanged: a dependency or aliased output storage may modify them.

The function leaves no visible local finalized flag and does not directly erase the complete context. It does not visibly restore the pre-call block contents. Any context bytes observed after return are also subject to dependency mutations.

Calling this function twice on the same context is not established to be safe or idempotent.

### Aliasing and ordering

There is no explicit overlap check.

The ordering of export reads and writes is significant when output storage overlaps the five-word source:

- Each word is fully read before its own four output bytes are stored.
- Those four stores may change memory that a later source-word read will consume.
- The function does not provide snapshot semantics for all 20 source bytes.

Output may also overlap the context or the global pointer slots. The supplied body does not prohibit this, but ordinary non-overlapping behavior cannot be assumed in those cases.

For example, output stores that overwrite the global source-pointer slot do not, by themselves, cause that pointer to be reloaded during export. Conversely, output stores that overwrite the indirect-call target slot occur before that target is used and can affect the eventual call.

### Ownership

No direct allocation or deallocation appears in this body.

That does not establish that the function as a whole allocates nothing, releases nothing, or preserves ownership: neither dependency’s ownership behavior is supplied. Passing the context address twice to F-29cbb14d23e7 is not evidence of ownership transfer or retention.

## Dependencies

| Dependency | Established interaction | Unknown behavior |
|---|---|---|
| F-29cbb14d23e7 | Called with the context address in both argument positions; once or twice according to the padding boundary | Required context extent, state mutations, global mutations, ownership effects, failures, and relationship to the exported words |
| Unresolved indirect callee; opaque function ID unavailable | Invoked after export with arguments 1, 0, 0 | Identity, purpose, return behavior, memory effects, and ownership effects |

Two global data dependencies are also visible:

- A pointer slot supplying the base address of five exported 32-bit words.
- A function-pointer slot supplying the final indirect target.

Neither global dependency is established to be immutable, per-context, thread-local, or protected against concurrent changes.

## Acceptance examples

These are conditional behavioral examples derived from static evidence. They are not runtime test results.

### One-block boundary: remainder 55

Given an initial low count word whose lowest six bits equal 55:

- Offset 55 receives hexadecimal 80.
- No padding zeros are needed before the trailer.
- Offsets 56–63 receive the trailer.
- F-29cbb14d23e7 is called once.

### Two-block boundary: remainder 56

Given an initial low count word whose lowest six bits equal 56:

- Offset 56 receives hexadecimal 80.
- Offsets 57–63 receive zero.
- F-29cbb14d23e7 is called.
- After its return, offsets 0–55 receive zero.
- The count fields are then read to construct the trailer.
- F-29cbb14d23e7 is called again.

If the first call changes the count fields, the trailer reflects the later reads, not necessarily the original count.

### Last block position: remainder 63

Given an initial remainder of 63:

- Offset 63 receives hexadecimal 80.
- There are no following bytes to zero before the first processing call.
- The second block’s first 56 bytes are zeroed before its trailer is written.
- Two processing calls occur.

### Stable zero count

If both count words remain zero during trailer construction, all eight trailer bytes are zero.

With an initial low count of zero, offset 0 first receives hexadecimal 80 and offsets 1–55 receive zero.

### Carry from the low count word

If the fields remain stable with high word zero and low word hexadecimal 20000000, the encoded bit length is hexadecimal 0000000100000000.

The trailer bytes, in order, are:

00, 00, 00, 01, 00, 00, 00, 00.

This illustrates that the top three low-word bits contribute to the high half of the encoded bit length.

### Maximum stable count

If both fields remain hexadecimal FFFFFFFF, the trailer bytes are:

FF, FF, FF, FF, FF, FF, FF, F8.

The multiplication by eight wraps modulo 2⁶⁴; no overflow error is reported.

### Export byte order

If a source word has numeric value hexadecimal 12345678 when read, its corresponding output group is:

78, 56, 34, 12.

This result concerns the numeric word read by the function, not an independently assumed host-memory byte order.

### Preliminary-call mutation

If the two-block path is taken and F-29cbb14d23e7 changes the count fields before returning, acceptance must be based on the fields subsequently read for trailer construction. Requiring the original entry count would contradict the visible read order.

## Uncertainties

The following remain **documented_only**:

- The algorithm name and complete context format.
- Whether the two count fields represent bytes processed, despite the apparent conversion to a bit count.
- Whether F-29cbb14d23e7 preserves the count fields or either global pointer slot.
- Whether the global export source belongs to the supplied context.
- The indirect callee’s identity and purpose.
- Dependency-specific errors, termination behavior, allocation, and release behavior.
- Validity requirements for overlapping storage.
- Thread safety, reentrancy, and behavior under concurrent mutation.
- Whether repeated finalization or subsequent updates are supported.
- Native byte order and alignment requirements for 32-bit memory reads.

These unknowns do not weaken the established padding boundaries, direct store order, dependency-call order, or export byte order. They limit claims about complete algorithmic results and post-call state.

## Evidence

Established static evidence consists of the supplied function body:

- A six-bit mask selects the initial block offset.
- Padding uses hexadecimal 80 followed by zero filling.
- The branch threshold separates remainders 0–55 from 56–63.
- Eight trailer bytes are written at offsets 56–63 using repeated 32-bit count-field reads.
- F-29cbb14d23e7 receives two identical context addresses.
- Five globally sourced words are serialized least-significant byte first.
- A final indirect call receives the values 1, 0, and 0.

Digest finalization, count units, and the meaning of the exported state are inferred rather than independently established.

No runtime tests have been performed.
