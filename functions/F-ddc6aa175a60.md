<!-- Trusted provenance: F-ddc6aa175a60; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ddc6aa175a60

## Purpose

F-ddc6aa175a60 reads one packed 16-bit value through a cursor stored at a caller-supplied address, advances that cursor by two bytes, and writes four consecutive output bytes.

The packed value contains three 5-bit fields and one single-bit field. Each 5-bit field is expanded to eight bits by repeating its upper three bits in the lower three output positions. The single-bit field becomes either 0 or 255.

The field arrangement is consistent with conversion from a packed RGB-plus-alpha representation to four-byte RGBA. This is **inferred intent**, not an established format identity. The established contract is the numeric transformation, cursor modification, and memory-access ordering described below.

The cursor update occurs **before** the packed input is loaded. This distinction affects overlapping storage and possible partial effects if a later memory access fails.

## Interface

The function accepts three arguments and has no explicit return value.

| Argument | Established role | Representation and access requirements |
|---|---|---|
| Output address | Identifies the first of four output byte positions | Must support writes at byte offsets 0 through 3 |
| Opaque argument | Not used by the supplied body | Represented as a 32-bit quantity |
| Cursor-slot address | Identifies storage containing the current source address | The slot is represented as 32 bits and must support a read followed by a write |

These are **caller-supplied addresses**. Supplying an address does not establish who owns the referenced storage. Ownership of the source, output, and cursor storage is unknown.

The cursor slot initially contains the source address. The function writes an address two bytes later into the same slot. The input is then read from the original source address, not from the advanced address.

The supplied representation uses a 32-bit slot to hold an address. This establishes the width visible in the supplied interface; it does not establish portability to a wider-pointer environment or the validity of every possible 32-bit address value.

### Packed input layout

Numbering the loaded integer’s bits from 0, the least significant bit, through 15, the most significant bit:

| Packed bits | Field width | Destination |
|---|---:|---|
| 15–11 | 5 bits | Output byte 0 |
| 10–6 | 5 bits | Output byte 1 |
| 5–1 | 5 bits | Output byte 2 |
| 0 | 1 bit | Output byte 3 |

Every bit contributes to one output field.

The source is accessed as one unsigned 16-bit value. The supplied body does not establish target byte order. Consequently, this table describes the loaded integer, not a separately established ordering of the two source bytes.

### Output layout

| Byte offset | Result |
|---|---|
| 0 | Expanded value of packed bits 15–11 |
| 1 | Expanded value of packed bits 10–6 |
| 2 | Expanded value of packed bits 5–1 |
| 3 | 0 when packed bit 0 is clear; 255 when it is set |

There is no returned status, byte count, or updated address. The cursor change is communicated through the cursor slot, and the conversion result through the four byte writes.

## Behavior

### Observable access order

For an ordinary successful execution, the supplied body expresses the following order:

1. Read the original source address from the cursor slot.
2. Write the address two bytes beyond it into the cursor slot.
3. Read the unsigned 16-bit value at the original source address.
4. Write output byte 0.
5. Write output byte 1.
6. Write output byte 2.
7. Write output byte 3.
8. Return without an explicit result.

The input is loaded once. Its value is retained for all four output calculations; the body does not reread the source between output writes. It also does not reread the cursor slot.

This is the ordering expressed by the supplied body. It is not a guarantee of atomicity, hardware transaction boundaries, inter-thread visibility, or memory-mapped-device behavior.

### Expansion of the three 5-bit fields

For each 5-bit field, let \(q\) be its unsigned value, where \(0 \leq q \leq 31\). Its output value is:

\[
E(q)=8q+\left\lfloor\frac{q}{4}\right\rfloor
\]

The original five bits occupy output bits 7–3. Their upper three bits are repeated into output bits 2–0.

The two parts occupy disjoint bit positions, so their bitwise combination has the same numeric value as their sum. The result lies between 0 and 255 inclusive.

| Field value | Expanded byte |
|---:|---:|
| 0 | 0 |
| 1 | 8 |
| 3 | 24 |
| 4 | 33 |
| 15 | 123 |
| 16 | 132 |
| 30 | 247 |
| 31 | 255 |

This is bit replication. It is not simply multiplication by eight, nor a general promise of floating-point scaling or a particular rounding policy.

### Expansion of the single-bit field

Output byte 3 is exactly:

- 0 if packed bit 0 is clear.
- 255 if packed bit 0 is set.

This value does not modify the other three outputs. In particular, a zero final byte does not clear the first three bytes. No multiplication, compositing, or premultiplication is present.

### Widths and narrowing

The loaded input is an unsigned 16-bit quantity. Its conversion to the 32-bit unsigned arithmetic representation preserves its value without sign extension.

The first output uses the upper source byte. Discarding its lower three bits selects packed bits 15–11; shifting that upper byte right by five supplies the replicated three bits.

For the second and third outputs, masking isolates the relevant field before shifting and narrowing to a byte. The narrowed high portion occupies output bits 7–3, and the replicated portion occupies bits 2–0. These operations preserve the expansion described above and do not produce signed-channel values.

The final-byte calculation produces only 0 or 255. No output value exceeds the range of an unsigned byte.

The cursor advance is one 16-bit element, or two bytes in the supplied representation. It is unrelated to the four-byte output length. No source-address boundary check is present. The body does not establish a valid wraparound policy; normal execution requires a validly representable advanced address under the target’s address rules.

### Overlapping storage

The body performs no overlap checks. The consequences below are conditional on the target permitting the accesses and on ordinary memory semantics.

**Source and output overlap, with a separate cursor slot:** The packed value is loaded before any output write. All outputs therefore depend on that loaded value, even if later writes overwrite its source bytes.

**Source and cursor-slot overlap:** The cursor-slot write precedes the source load. The load can therefore observe bytes changed by the cursor advance. The conversion is not necessarily based on the bytes present before entry.

**Output and cursor-slot overlap:** Output writes occur after the cursor update and can overwrite some or all of the advanced address. The final cursor-slot contents need not equal the advanced address.

**All three regions overlap:** Both kinds of interference may occur: the cursor update can affect the input, and the output writes can subsequently affect the cursor slot.

These observations describe ordering consequences, not a blanket guarantee that overlapping typed accesses are valid. Alignment, address representation, and target-language access rules remain relevant.

## Side effects

For valid, non-overlapping ordinary storage, successful execution explicitly modifies:

- The cursor slot, replacing its original source address with one two bytes later.
- Four consecutive output bytes, in increasing offset order.

There is no separate explicit write to the source location. Overlap can nevertheless cause a cursor-slot or output write to modify storage that is also part of the source.

The supplied body contains no explicit:

- Allocation or deallocation.
- External function call.
- Direct global-storage access.
- Logging or synchronization operation.
- Address retention beyond the cursor-slot update.

All explicit memory accesses are through the supplied addresses or the source address read from the cursor slot. Those addresses may designate global storage; the absence of direct global references does not exclude that possibility.

**Ownership is unknown.** The body establishes neither ownership of the supplied regions nor an ownership-transfer contract. Its lack of allocation and deallocation is an observation about its operations, not proof that the caller owns the storage.

### Preconditions and errors

Normal completion requires:

- A readable and writable cursor slot with the required address representation.
- A readable 16-bit source at the address obtained from that slot.
- Four writable output byte positions.
- Alignment and permissions sufficient for the target’s accesses.
- Valid arithmetic for the two-byte cursor advance.
- No concurrent interference that invalidates these operations.

The body contains no null, bounds, alignment, or address-overflow checks. It supplies no explicit error result or recovery path.

If the source load fails after the cursor-slot write, the cursor update may already have taken effect. If an output write fails, the cursor update and earlier output writes may already have taken effect. No rollback mechanism is expressed.

The exact consequences of an invalid access—including whether it faults, raises an exception, or has other behavior—are not established. The access order must not be treated as a portable guarantee of recoverable partial execution.

## Dependencies

No callable dependencies appear in the supplied body. There is no additional opaque function ID whose behavior must be assumed to explain this function.

Execution depends on environmental properties:

- Address representation in the 32-bit cursor slot.
- Native interpretation of the 16-bit source load.
- Alignment and access-permission rules.
- Pointer arithmetic and address validity.
- Invalid-access and concurrent-access behavior.

These are environmental dependencies, not evidence of hidden helper calls. No unknown callee is assumed to preserve globals, allocate storage, or release memory.

## Acceptance examples

The following are **conditional static acceptance examples**, not observed executions.

Unless otherwise stated, assume valid non-overlapping ordinary storage, a cursor identifying the listed 16-bit value, four writable output bytes, and a valid two-byte cursor advance. After successful completion, the cursor contains the address two bytes beyond its initial source address.

| Loaded 16-bit value | Output bytes at offsets 0, 1, 2, 3 |
|---|---|
| `0x0000` | 0, 0, 0, 0 |
| `0x0001` | 0, 0, 0, 255 |
| `0xF800` | 255, 0, 0, 0 |
| `0x07C0` | 0, 255, 0, 0 |
| `0x003E` | 0, 0, 255, 0 |
| `0xFFFE` | 255, 255, 255, 0 |
| `0xFFFF` | 255, 255, 255, 255 |
| `0x0843` | 8, 8, 8, 255 |
| `0x8421` | 132, 132, 132, 255 |

Additional acceptance conditions:

- **Replication boundary:** A 5-bit field of 3 produces 24; a field of 4 produces 33. This distinguishes replication from a simple three-bit left shift.
- **Independent final byte:** `0xFFFE` produces three maximum-valued bytes even though the final byte is zero.
- **Unused argument:** Changing only the opaque 32-bit argument does not change this body’s accesses or calculations.
- **Successive calls:** If the cursor slot remains separate from output storage and nothing else changes it, two successful calls load successive 16-bit elements and leave the cursor four bytes beyond its initial address.
- **Source/output overlap:** If the target permits the overlap and the cursor slot is separate, all four outputs reflect the single value loaded before output writes begin.
- **Cursor/output overlap:** If such overlap is permitted, acceptance must account for the ordered output writes overwriting the cursor slot; retaining an advanced cursor is not guaranteed.
- **Source-load failure:** In an environment exposing a failure after a completed cursor-slot write, no rollback of that write is provided. The failure mechanism itself remains environment-dependent.

None of these examples establishes storage ownership or a particular byte-stream endianness.

## Uncertainties

The following are **documented_only**:

- **Ownership and lifetime:** The owners of the source, output, and cursor storage are unknown. Broader lifetime obligations are not established beyond the validity needed for the accesses.
- **Byte order:** The relationship between the two source bytes and the loaded integer is unspecified by the supplied body.
- **Format identity:** RGB and alpha terminology is inferred from the field arrangement, not confirmed by an external format contract.
- **Opaque argument purpose:** Its intended API role is unknown; it is unused here.
- **Pointer model:** The representation uses a 32-bit cursor slot, but address-space restrictions and pointer-provenance rules are unavailable.
- **Alignment tolerance:** Support for unaligned cursor-slot or 16-bit accesses is not established.
- **Overlap legality:** Ordering consequences are identifiable, but the legality of particular overlapping accesses is not established.
- **Fault and concurrency semantics:** Atomicity, fault delivery, hardware observation order, and concurrent-access guarantees are unknown.
- **Allocation bounds:** No storage sizes or allocation metadata are supplied.
- **Image interpretation:** Channel names, color space, transfer function, and downstream use are unknown.

These unknowns do not change the numeric mapping once a particular 16-bit value has been successfully loaded. They do limit which executions can be treated as valid and what surrounding guarantees can be claimed.

## Evidence

The supplied static body establishes:

- One cursor-slot read and one cursor-slot write.
- A two-byte cursor advance before the source load.
- One unsigned 16-bit source load.
- Three 5-bit field expansions by bit replication.
- One single-bit expansion to 0 or 255.
- Four output byte writes in increasing offset order.
- No use of the opaque argument.
- No explicit calls, allocation, deallocation, validation, or error result.
- No explicit address retention beyond the cursor update.

The packed-color interpretation is inferred intent. Ownership, platform semantics, and broader caller contracts remain unresolved where identified above.

No runtime tests or execution traces have been performed. The acceptance examples are derived from the supplied static body, not measured results.
