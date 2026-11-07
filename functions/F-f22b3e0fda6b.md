<!-- Trusted provenance: F-f22b3e0fda6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f22b3e0fda6b

## Purpose

F-f22b3e0fda6b initializes selected fields of a caller-supplied memory region, records two 32-bit input values, derives a one-byte flag from their nonzero status, and copies four 32-bit values from an externally selected source.

The established behavior is a sequence of memory writes, not a whole-region clear. The destination must provide at least 920 bytes, but 16 bytes within that extent are not directly written. Existing contents in those gaps survive unless affected by behavior outside this function.

**Inferred intent:** the destination is an object containing scalar fields, arrays, flags, and a four-word default value. The source does not establish the object’s domain, field meanings, or whether this operation is intended only for fresh storage.

## Interface

### Inputs

| Input | Contract |
|---|---|
| Destination | Base address of a writable memory region extending through byte offset `0x397`, inclusive. |
| First value | A 32-bit integer stored at destination offset `0x0C4`. |
| Second value | A 32-bit integer stored at destination offset `0x0C8`. |

All destination offsets in this document are byte offsets from the supplied base address.

The two integer inputs are stored as complete 32-bit values. Their signs do not affect control flow. Each is also compared with zero when calculating the derived flag.

### Output

There is no explicit return value. The result is the modified destination region.

There is no returned status, error indicator, allocated object, or replacement destination pointer.

### Required storage and access conditions

For the described behavior to complete normally:

- The destination must be non-null and support every listed write.
- Its accessible extent must be at least `0x398` bytes, or 920 bytes.
- The destination must satisfy the execution environment’s requirements for 32-bit memory accesses.
- An external pointer-valued location must be readable at each specified observation.
- The final pointer obtained from that location must identify four readable 32-bit words, spanning 16 bytes.
- Address calculations and accesses must be valid in the execution environment.

The source uses a 32-bit address interpretation for one explicitly byte-addressed store. Consequently, portability to an environment with wider addresses is not established merely by supplying a sufficiently large allocation.

The function does not check these preconditions.

### Widths and representation

The contract distinguishes:

- **Word writes:** exactly 32 bits each.
- **Byte writes:** exactly one byte each.
- **Integer comparisons:** zero versus nonzero over the complete 32-bit input value.
- **Derived flag:** a Boolean result stored through a one-byte field.

The flag is false when either input is zero and true otherwise. The supplied representation expresses a Boolean store; its exact true-byte encoding is not independently established by machine instructions. A consumer requiring a particular nonzero encoding needs that additional evidence.

The byte order used for word storage is the execution environment’s byte order. No conversion is shown.

## Behavior

### Destination layout

The following table describes the final values after normal completion, subject to the source-read and aliasing qualifications below.

| Destination offsets, inclusive | Size | Result |
|---|---:|---|
| `0x000–0x00F` | 16 bytes | Four zero words |
| `0x010–0x0AF` | 160 bytes | Twenty consecutive pairs of zero words |
| `0x0B0–0x0B3` | 4 bytes | Zero word |
| `0x0B4` | 1 byte | Zero |
| `0x0B5` | 1 byte | Zero |
| `0x0B6–0x0BF` | 10 bytes | Not directly written |
| `0x0C0` | 1 byte | Zero |
| `0x0C1–0x0C3` | 3 bytes | Not directly written |
| `0x0C4–0x0C7` | 4 bytes | First input |
| `0x0C8–0x0CB` | 4 bytes | Second input |
| `0x0CC` | 1 byte | Both-inputs-nonzero Boolean |
| `0x0CD–0x0CF` | 3 bytes | Not directly written |
| `0x0D0–0x0DF` | 16 bytes | Four words obtained from the external source |
| `0x0E0–0x21F` | 320 bytes | Eighty zero words |
| `0x220–0x223` | 4 bytes | Zero word |
| `0x224–0x30B` | 232 bytes | Fifty-eight zero words |
| `0x30C–0x30F` | 4 bytes | Zero word |
| `0x310–0x38F` | 128 bytes | Thirty-two zero words |
| `0x390–0x397` | 8 bytes | Two zero words |

This covers a 920-byte extent with 904 bytes directly written and 16 bytes left untouched. Those 904 bytes comprise 225 word writes and four byte writes.

The four copied words are not subsequently cleared by this function.

### Ordered initialization

The source establishes the following operation order. These are logical source-level accesses, not a guarantee about compiler-generated instructions, atomicity, or device-memory transactions.

1. **Record inputs and early flags.**  
   The first input is written at `0x0C4`, followed by the second input at `0x0C8`. The byte at `0x0C0` is cleared. The Boolean at `0x0CC` is then written according to whether both inputs are nonzero. The byte at `0x0B5` is cleared next.

2. **Clear the leading scalar area.**  
   Four word writes clear offsets `0x000`, `0x004`, `0x008`, and `0x00C`, in that order. The byte at `0x0B4` is then cleared.

3. **Clear twenty pairs and observe the external pointer.**  
   The function processes twenty consecutive two-word pairs beginning at `0x010`. Within each pair, it clears the lower-addressed word before the next word. Pairs are processed in increasing address order.

   After each pair, the function reads the external pointer-valued location. There are twenty such source-level observations. Only the pointer obtained after the twentieth pair is retained for the later four-word copy. Earlier observed pointers are not dereferenced by this function.

4. **Clear the word immediately after the pair area.**  
   The word at `0x0B0` is set to zero. This occurs after the final pointer observation and before any of the four copied source words are read.

5. **Read and copy the four-word source value.**  
   Using the retained pointer, the function reads source words in this order:

   - The word at source offset `0x04`.
   - The word at source offset `0x08`.
   - The word at source offset `0x0C`.
   - The word at source offset `0x00`.

   The last of these reads supplies the first destination word. The destination writes then occur in increasing address order: `0x0D0`, `0x0D4`, `0x0D8`, and `0x0DC`.

   Each destination receives the correspondingly positioned source word: source offset zero maps to `0x0D0`, source offset four maps to `0x0D4`, and so on. All four source values have been obtained before the first destination word in this copy is overwritten.

6. **Clear the remaining word regions.**  
   The function clears eighty consecutive words starting at `0x0E0`, then the word at `0x220`. It next clears fifty-eight consecutive words starting at `0x224`, then the word at `0x30C`. It then clears thirty-two consecutive words starting at `0x310`, followed by separate word writes at `0x390` and `0x394`.

   Each multiword region is traversed in increasing address order.

7. **Return without a result value.**

### Integer and boundary behavior

The input values do not control iteration counts, destination offsets, or copied length. Zero affects only the derived Boolean flag; it does not cause early return.

The loops execute fixed counts of 20 pairs, 80 words, 58 words, and 32 words. Their counters remain small enough that the shown arithmetic does not approach a 32-bit overflow boundary.

The highest word write starts at `0x394` and ends at `0x397`. No destination access beyond that boundary is shown.

### Aliasing and observation timing

The source does not establish that the destination, the external pointer location, and the selected four-word source occupy disjoint storage.

This matters in two distinct ways:

- If earlier destination writes overlap the external pointer location, later pointer observations may obtain a changed value. The relevant source pointer remains the value observed after the twentieth pair, not necessarily its value at function entry.
- If destination writes performed before the copy overlap the selected source words, the copy observes the resulting contents, not an entry-time snapshot.

Within the four-word copy itself, all four values are read before its destination writes begin. Thus overlapping source and destination ranges do not cause an earlier copy write to corrupt a later source read.

There is no concurrency or synchronization contract. Concurrent mutation of the pointer location or source contents cannot be treated as a coherent snapshot based on this evidence alone.

## Side effects

The function directly modifies only the destination bytes listed as written in the layout table. It also reads the external pointer location and the selected source words.

No dependency calls, allocation, release, reference-count adjustment, logging, or explicit synchronization are shown.

The destination remains caller-supplied storage. No ownership transfer is expressed. Likewise, the external source is read without any shown acquisition or release operation.

If destination fields previously contained pointers or resource handles, their values may be overwritten without cleanup. Whether such contents are valid before this call is unknown; the function itself supplies no resource-lifetime protection.

There is no validation or rollback. An invalid access may occur after earlier destination writes have completed. The resulting fault behavior, termination behavior, and partial-state visibility are properties of the execution environment, not an error protocol established here.

The function does not directly write outside the destination. Nevertheless, if external state aliases destination storage, its contents can change through those destination writes.

## Dependencies

### Function dependencies

No function calls are present. There are therefore no additional opaque function IDs to enumerate.

### External data dependency

The function depends on one external pointer-valued location. Its symbolic name, address, initialization, and ownership are outside this contract.

The established requirements are:

- It is read once after each of the twenty pair-clearing iterations.
- The final observed pointer supplies the four-word source.
- Four 32-bit source words are read using that retained pointer.
- No fallback source or validity check is shown.
- No release or ownership operation is shown.

**Inferred intent:** this location selects a default four-word value.

**Unknown:** whether the location is immutable, whether the selected value represents an aggregate object, and whether aliasing is forbidden by a surrounding API.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reported runtime results.

### Both inputs are zero

Given valid destination and source storage, with both inputs zero:

- The words at `0x0C4` and `0x0C8` are zero.
- The flag at `0x0CC` is false.
- All fixed clearing and copying operations still occur.
- The untouched gaps retain their prior bytes when no external mutation intervenes.

### Exactly one input is zero

With first input `0x12345678` and second input zero:

- `0x0C4` contains the complete first value.
- `0x0C8` contains zero.
- The derived flag is false.

Exchanging the zero and nonzero inputs also produces a false flag, with the values stored in their respective fields.

### Both inputs are nonzero

With first input `−1` and second input `2`:

- The first field contains the 32-bit representation of `−1`.
- The second field contains `2`.
- The derived flag is true.

Negative values are not rejected or treated as false.

### Distinct source words

Suppose the retained source pointer identifies these stable words in increasing address order:

| Source offset | Value |
|---|---|
| `0x00` | `0x11223344` |
| `0x04` | `0x55667788` |
| `0x08` | `0x99AABBCC` |
| `0x0C` | `0xDDEEFF00` |

After normal completion, those values appear respectively at destination offsets `0x0D0`, `0x0D4`, `0x0D8`, and `0x0DC`. Their placement follows source position, despite the nonascending source-read order.

### Preserved gaps

If the destination initially contains a recognizable byte pattern and the external data is disjoint from it, bytes `0x0B6–0x0BF`, `0x0C1–0x0C3`, and `0x0CD–0x0CF` retain that pattern.

Replacing the behavior with a whole-region clear would fail this criterion.

### Source overlaps an earlier-cleared region

If the retained source points to destination offset `0x000`, the four source words have already been cleared before they are read. The copied block at `0x0D0–0x0DF` consequently becomes zero, regardless of that leading region’s entry contents.

### Source is the copy destination

If the retained source points exactly to destination offset `0x0D0`, the four words there are read before any copy write begins. In the absence of external mutation, those words retain their values through the copy.

### Invalid storage

A null destination or an unreadable final source pointer has no specified successful outcome. Acceptance must not require a graceful error return: no such path is established.

## Uncertainties

The following are **documented_only**, not implementation requirements inferred from naming or layout:

- The semantic meanings of the destination fields and arrays.
- The intended lifecycle stage at which this function is called.
- The actual allocation size beyond the minimum accessed extent.
- The precise byte encoding of a true Boolean.
- Target-specific alignment, fault, and memory-ordering rules.
- Whether the external pointer location is constant, volatile, synchronized, or allowed to alias the destination.
- Ownership and lifetime of the selected source storage.
- Whether overwriting previously initialized fields is supported.
- Whether compiler output preserves each source-level pointer observation as a distinct physical load.

No claim is made that this operation is thread-safe, atomic, idempotent under all aliasing arrangements, or safe for memory-mapped devices.

## Evidence

The established contract comes from static inspection of the supplied function representation:

- Explicit four-byte and one-byte stores establish field widths and destination coverage.
- Fixed loop bounds establish the clearing counts and final accessed offset.
- Input comparisons establish zero/nonzero flag behavior without signed-range validation.
- The external pointer assignment within the pair-clearing loop establishes the source-level observation sequence.
- The four source loads and destination stores establish copy mapping and overlap behavior.
- The absence of calls, validation branches, and status-producing paths limits the resource and error behavior that can be claimed.

Object-purpose interpretations are identified as inferred intent. Environmental and representation gaps remain documented_only. No runtime tests have been performed.
