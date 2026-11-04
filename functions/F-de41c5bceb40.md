<!-- Trusted provenance: F-de41c5bceb40; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de41c5bceb40

## Purpose

F-de41c5bceb40 selectively initializes a destination region using a discriminator from one input region and two four-byte fields from another. It writes a fixed one-byte value, conditionally initializes up to four additional bytes, copies each four-byte source field twice, and clears four four-byte destination fields.

This is partial initialization. Some destination bytes are never written, and the conditional initialization has a different write footprint for each recognized discriminator. The function does not establish that the destination becomes a completely initialized object.

The semantic meanings of the discriminator, copied fields, and constants are unknown. “Discriminator” describes its role in selecting a branch, not an established application-level type.

## Interface

The function accepts three address-bearing arguments and has no defined return value.

| Argument label | Role |
|---|---|
| **A** | Base address of the region containing two four-byte source fields. |
| **B** | Base address of the region containing the one-byte discriminator. |
| **D** | Base address of the destination region. |

These labels identify access roles only. They do not establish object types, allocation boundaries, ownership, or whether the regions are distinct.

### Access widths and layouts

All offsets below are hexadecimal byte offsets. One-, two-, and four-byte accesses correspond to 8-, 16-, and 32-bit quantities in the supplied representation. The representation does not independently establish the machine address width or complete calling convention.

The source accesses are:

| Region | Offset | Width | Access behavior |
|---|---:|---:|---|
| A | `0x28` | 4 bytes | Read twice, at separate points. |
| A | `0x2C` | 4 bytes | Read twice, at separate points. |
| B | `0x08` | 1 byte | Read once for copying, then again for branch selection. |

The destination accesses are:

| Offset or range | Width | Content written |
|---|---:|---|
| `0x00` | 1 byte | First discriminator read. |
| `0x01` | 1 byte | Numeric value `3`. |
| `0x04–0x07` | 4 bytes | Zero. |
| `0x08–0x0B` | 4 bytes | Zero. |
| `0x0C–0x0F` | 4 bytes | First read of A at `0x28`. |
| `0x10–0x13` | 4 bytes | First read of A at `0x2C`. |
| `0x14–0x17` | 4 bytes | Second read of A at `0x28`. |
| `0x18–0x1B` | 4 bytes | Second read of A at `0x2C`. |
| `0x24–0x27` | 4 bytes | Zero. |
| `0x28–0x2B` | 4 bytes | Zero. |
| Starting at `0x30` | Branch-dependent | No write, three byte writes, one two-byte write, or one four-byte write. |

For ordinary contiguous regions, an extent of `0x30` bytes from A covers all source accesses through A, and an extent of `0x09` bytes from B covers both discriminator reads. A destination extent of `0x34` bytes covers all possible destination accesses. These are coverage bounds, not evidence of allocation sizes or accesses to every intervening byte.

### Preconditions and ownership

Every access taken must be valid for its address, width, and execution environment. Any required alignment must also be satisfied. The function contains no explicit null, bounds, alignment, or address-overflow checks.

No allocation, release, or explicit ownership-transfer operation is visible. Ownership is otherwise unknown: the addresses may identify borrowed, shared, globally owned, or other storage. Merely supplying an address does not establish that the caller owns it.

The function does not establish a lifetime extension or a general ownership policy. Storage validity for the duration of the accesses is a precondition, not an ownership conclusion.

## Behavior

### Initial writes and discriminator selection

The first destination effect is a one-byte write of numeric value `3` at D offset `0x01`.

Next, the function reads one byte from B offset `0x08` and writes that byte at D offset `0x00`. It then reads B offset `0x08` again to select the conditional initialization. Branch selection therefore uses a separate source read, not an established reuse of the byte copied into D.

The second discriminator read selects the following behavior:

| Second discriminator read | Conditional destination writes |
|---|---|
| `0` | One-byte value `0x10` at `0x30`, then one-byte zero at `0x31`, then one-byte zero at `0x32`. |
| `2` | One two-byte write of numeric value `0x001F` at `0x30`. |
| `3` | One four-byte write of zero at `0x30`. |
| Any other byte value | No conditional destination write. |

For discriminator `0`, the branch does not write D offset `0x33`. Its three stores are individually one byte wide; it is not represented as a four-byte initialization.

For discriminator `2`, D offsets `0x32` and `0x33` are not written. The stored numeric value is decimal 31. Its byte representation is `1F 00` on a little-endian target or `00 1F` on a big-endian target. Target byte order is not independently established.

For discriminator `3`, all four bytes at D offsets `0x30–0x33` become zero.

All other discriminator values, including `1` and `0xFF`, skip this initialization. Signed versus unsigned interpretation of the byte does not alter which byte patterns compare equal to `0`, `2`, and `3`.

### Source-field copies and unconditional clears

After the conditional stage, the source-field copies occur in this order:

| Copy order | Source access | Destination write |
|---:|---|---|
| First | Four bytes at A offset `0x2C` | Four bytes at D offset `0x10` |
| Second | Four bytes at A offset `0x28` | Four bytes at D offset `0x0C` |
| Third | Four bytes at A offset `0x2C`, read again | Four bytes at D offset `0x18` |
| Fourth | Four bytes at A offset `0x28`, read again | Four bytes at D offset `0x14` |

Each source read supplies its corresponding destination write before the next listed copy. There is no established single snapshot of both source fields or reuse of the first pair of read values for the second pair of copies.

After all four copies, four-byte zero writes occur at D offsets `0x28`, `0x24`, `0x04`, and `0x08`, in that order. The function then returns without a result value.

The copied four-byte fields undergo no visible arithmetic transformation. They are not visibly added, scaled, narrowed, widened, or interpreted as application-level numeric quantities.

These ordering statements describe the supplied static representation. They do not independently establish processor scheduling, atomicity, memory barriers, or cross-thread visibility.

### Destination bytes left untouched

The function does not directly write the following destination ranges:

- `0x02–0x03`;
- `0x1C–0x23`;
- `0x2C–0x2F`;
- conditional-field bytes excluded by the selected branch;
- any destination offset outside the listed write footprint.

With ordinary valid storage and no external modification, those bytes retain their previous contents. They are not implicitly initialized merely because surrounding fields are written.

### Overlap and changing inputs

No non-overlap check is visible. If the regions overlap in an execution environment that permits the accesses, earlier destination writes can change later source reads.

The initial write at D offset `0x01` may affect the discriminator before either read. The write at D offset `0x00` precedes the second discriminator read and may also affect that read when the addresses coincide. Conditional writes precede all reads through A and can change source-field contents. Each four-byte destination copy likewise precedes subsequent source reads.

Overlapping cases must therefore be evaluated from the ordered accesses, not from an assumed entry snapshot. Whether a particular overlap is valid under the original language’s typing and aliasing rules remains unknown.

For disjoint regions with stable inputs, the two discriminator reads agree, the two copies of A offset `0x28` agree, and the two copies of A offset `0x2C` agree. Without those conditions, these equalities are not unconditional postconditions.

## Side effects

The visible side effects are the listed memory reads and destination writes. No external call, allocation, release, I/O operation, synchronization operation, or explicit access to named global state appears.

This absence does not imply that global or shared storage is unaffected. Any supplied address may refer to such storage. Memory-mapped or otherwise special regions may also attach additional effects to the represented accesses; their behavior is not established here.

The operation is not represented as transactional. Initial destination writes precede later reads and writes. No rollback, restoration, completion marker, or recoverable error status is visible.

Invalid storage, unsupported alignment, or invalid address formation is not handled by an explicit error path. Such cases must not be specified as successful no-ops or defined partial-initialization outcomes. Failure behavior depends on the language and execution platform.

## Dependencies

No external function dependency is visible. No additional opaque function ID is needed to describe the established behavior of F-de41c5bceb40.

Execution depends on:

- valid memory for the accesses actually taken;
- platform semantics for one-, two-, and four-byte accesses;
- target byte order for the nonzero two-byte constant;
- applicable address-arithmetic and alignment rules;
- applicable rules for overlapping or externally changing memory.

There is no callee whose memory ownership, release behavior, or preservation of global state needs to be assumed.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of runtime tests.

### Stable discriminator zero

Given valid, disjoint regions, stable source fields, and a discriminator of `0`:

- D offset `0x00` receives `0`, and offset `0x01` receives `3`.
- D offsets `0x30–0x32` receive bytes `10 00 00`.
- D offset `0x33` retains its previous content.
- Each stable four-byte source field appears in both assigned destination locations.
- The four unconditional zero fields are cleared.
- The destination gaps remain unchanged.

A four-byte clear of the conditional field would not preserve this branch’s write footprint.

### Stable discriminator two

Under the same memory conditions with discriminator `2`, the conditional stage writes the 16-bit numeric value 31 at D offset `0x30`. It leaves offsets `0x32` and `0x33` unchanged.

On a little-endian target, the two written bytes are `1F 00`; on a big-endian target, they are `00 1F`. Neither byte sequence can be required without establishing target byte order.

### Stable discriminator three

With stable discriminator `3`, D offset `0x00` receives `3`, and all four bytes at `0x30–0x33` become zero. The four source-field copies and four unconditional clears still occur afterward.

### Unrecognized discriminator

With stable discriminator `0xFF`, D offset `0x00` receives `0xFF`, but the conditional stage writes none of offsets `0x30–0x33`. The unconditional copies and clears still occur. No rejection or error return is established.

### Distinct values on repeated reads

If a valid memory environment supplies discriminator `0` on the first read and `3` on the second, D offset `0x00` receives `0`, while the conditional stage clears four bytes. Always selecting the branch from the previously copied byte would not preserve the represented read behavior.

Likewise, if the two reads at A offset `0x2C` validly return different four-byte values, D offset `0x10` receives the earlier value and offset `0x18` receives the later value. This criterion does not establish that unsynchronized concurrent mutation is valid.

### Destination preservation

Given otherwise valid, disjoint inputs and a destination prefilled with recognizable bytes, ranges `0x02–0x03`, `0x1C–0x23`, and `0x2C–0x2F` remain unchanged. A blanket destination clear would violate this behavior.

### Ownership-neutral invocation

If valid supplied addresses refer to borrowed or globally owned storage, the function’s visible access behavior is unchanged. Successful initialization does not establish that the caller owns those regions, that ownership transferred, or that any participant may release them.

## Uncertainties

The following remain **documented_only**:

- Semantic types and meanings of the discriminator, copied fields, and constants.
- Ownership of every supplied region.
- Original object declarations, allocation boundaries, and aliasing restrictions.
- Complete ABI, address width, and address-arithmetic overflow behavior.
- Target byte order and alignment requirements.
- Atomicity and cross-thread visibility of accesses.
- Whether callers permit overlap or externally changing inputs.
- Behavior for invalid accesses or interrupted execution.
- Initialization requirements for bytes not written here.

Selective record initialization with duplicated field values is plausible inferred intent. It does not establish a complete object schema, ownership policy, or stronger initialization guarantee.

## Evidence

The supplied static representation establishes the three address roles, access offsets and widths, constants, discriminator comparisons, repeated reads, represented access order, and absence of visible calls or an explicit error path.

No allocation, release, or explicit ownership-transfer operation appears. That observation supports only the absence of such visible operations; it does not identify the owner of any region.

This contract separates those static observations from inferred purpose and unresolved platform or language properties. It does not claim a verified ABI, runtime trace, or concurrency guarantee.

No runtime tests have been performed.
