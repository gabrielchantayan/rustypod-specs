<!-- Trusted provenance: F-f3f729ad60be; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f3f729ad60be

## Purpose

F-f3f729ad60be transfers seven consecutive 32-bit words from a source region into a destination region beginning 20 bytes after a supplied destination base address.

The transfer covers 28 destination bytes, at offsets 20 through 47 inclusive relative to that base. The function does not interpret the words, validate their values, return a status, or invoke another function.

The important behavioral distinction is that this is **not established as a snapshot copy of all seven words**. The second, third, and fourth source words are read before the first destination write. The fifth, sixth, and seventh words are each read later, immediately before their respective destination writes. Consequently, overlap can change which values reach the final three destination positions.

This contract describes the supplied static representation. No runtime tests have been performed.

## Interface

### Inputs

The function has two inputs:

| Input | Represented form | Required use |
|---|---|---|
| Destination base | Integer address value | Locates a writable region beginning at byte offset 20 from that address. |
| Source address | Pointer to consecutive 32-bit words | Locates seven words whose values are read during the transfer. |

For the descriptions below:

- **B** denotes the destination base address.
- **S** denotes the source address.
- **D** denotes the first destination word address, equal to B plus 20 bytes.

These are contract notation, not identifiers from the implementation.

The seven source word positions begin at offsets 0, 4, 8, 12, 16, 20, and 24 from S. The corresponding destination word positions begin at offsets 20, 24, 28, 32, 36, 40, and 44 from B.

### Width and layout

Every transferred word is exactly four bytes, or 32 bits. There is no sign extension, truncation, numeric conversion, comparison, or arithmetic on a transferred word.

The layout is:

| Word position | Source byte offset | Destination byte offset from B |
|---|---:|---:|
| First | 0 | 20 |
| Second | 4 | 24 |
| Third | 8 | 28 |
| Fourth | 12 | 32 |
| Fifth | 16 | 36 |
| Sixth | 20 | 40 |
| Seventh | 24 | 44 |

The operation accesses 28 source bytes and writes 28 destination bytes. It does **not** require the first 20 bytes at B to be read or initialized for its own operation. Those bytes have no direct write targets in the supplied representation.

### Output

The function has no return value. Its result is the sequence of writes to destination memory.

There is no success indicator, error code, count, or returned pointer.

### Preconditions

For ordinary successful execution:

- Each source word must be readable when its scheduled read occurs.
- Each destination word must be writable when its scheduled write occurs.
- Address calculations must identify the intended storage on the execution platform.
- Accesses must satisfy any applicable platform alignment and memory-access requirements.
- If stable, reproducible results are required, external agents must not modify the accessed storage during execution.

Source and destination non-overlap is sufficient for straightforward copying, but is not checked or explicitly enforced. Overlap must be evaluated using the access order below.

The exact address width, supported unaligned-access behavior, and consequences of invalid addresses are not established by the supplied representation.

## Behavior

### Established access sequence

The supplied representation establishes the following logical memory-access order:

| Step | Access |
|---:|---|
| 1 | Read the second source word, at S + 4, retaining its value. |
| 2 | Read the third source word, at S + 8, retaining its value. |
| 3 | Read the fourth source word, at S + 12, retaining its value. |
| 4 | Read the first source word, at S. |
| 5 | Write that first word to D. |
| 6 | Write the retained second word to D + 4. |
| 7 | Write the retained third word to D + 8. |
| 8 | Write the retained fourth word to D + 12. |
| 9 | Read the fifth source word, at S + 16. |
| 10 | Write that value to D + 16. |
| 11 | Read the sixth source word, at S + 20. |
| 12 | Write that value to D + 20. |
| 13 | Read the seventh source word, at S + 24. |
| 14 | Write that value to D + 24. |

Thus, four source values have been obtained before the first destination write. Of those four, the second through fourth are acquired before the first.

There are seven logical source reads and seven logical destination writes. No destination read is explicitly present, except where a source read reaches the same storage through overlap.

This order describes the supplied representation’s behavior. It is not evidence of particular hardware instructions, bus transactions, atomicity, or memory barriers.

### Non-overlapping regions

When the source and destination regions do not overlap, and source memory remains stable, each destination word receives the corresponding source word.

All 32 bits of every word are transferred without interpretation. Zero values, high-bit-set values, and arbitrary bit patterns follow the same path. There are no value-dependent branches.

The operation does not append a terminator or initialize any bytes beyond the seven destination words.

### Overlapping regions

The result under overlap depends on which source words earlier destination writes replace.

The retained second, third, and fourth source values are not changed by subsequent writes to their original memory locations. The first source value is also obtained before any destination write.

The final three source words have no such early snapshot. Each is read only after all preceding destination writes. If those writes have overwritten a later source word, the later read obtains the overwritten value under ordinary sequential memory semantics.

Accordingly:

- This function must not be specified as providing a general overlap-safe move.
- It must not be described as capturing the entire source region before writing.
- Source storage can change when it aliases the destination.
- A caller requiring preservation of the original seven-word sequence must establish a suitable layout rather than rely on an unstated overlap guarantee.

Overlap at non-word-aligned displacements would additionally depend on supported access alignment and byte layout. The supplied representation does not establish a portable result for such cases.

### Address arithmetic

Destination offsets range from 20 through 44 bytes for word starts. The final four-byte write extends through byte offset 47.

Source word starts range from 0 through 24 bytes. The final read extends through byte offset 27.

The destination input is represented as an integer used to form addresses. The evidence does not establish a portable overflow or wrapping rule for those calculations. A caller must not infer that arithmetic near an address-space boundary safely wraps or that an integer outside the valid address range is accepted.

### Completion and errors

On ordinary completion, all seven destination writes have occurred, and the function returns without a value.

There is no validation, explicit error branch, retry, recovery, or rollback. Invalid memory access or another platform-level interruption may prevent completion. The resulting diagnostic, exception, or process behavior is outside the established evidence.

Earlier writes may already have changed memory before a later access fails. The function provides no transactional all-or-nothing guarantee.

## Side effects

The explicit side effects are seven 32-bit destination writes.

No explicit write targets bytes outside the destination interval from B + 20 through B + 47. If another object or the source region aliases that interval, the same writes can affect that object or source storage.

The function does not explicitly:

- Allocate or release memory.
- Retain either address for later use.
- Change ownership of either region.
- Invoke a callback or dependency.
- Perform input/output.
- Update a separately identified global variable.
- Synchronize with another execution context.

These are statements about visible operations in the supplied representation, not promises that writing an arbitrary caller-selected address cannot affect shared or global storage.

The caller remains responsible for storage lifetime and access permissions. Nothing establishes that the function takes ownership or releases either region.

No atomicity is established for the whole transfer or for an individual four-byte access. Concurrent observers must not assume that all seven words become visible together.

## Dependencies

No called functions appear in the supplied representation. There are therefore no external function dependencies requiring opaque IDs.

F-f3f729ad60be depends on the execution environment for address interpretation, readable and writable memory, access alignment, and handling of invalid accesses. Those platform facilities are not represented as callable dependencies, and their detailed behavior is undocumented here.

## Acceptance examples

The following are conditional acceptance examples derived from static evidence, not executed tests.

Letters denote distinct arbitrary 32-bit values. Unless otherwise stated, memory is valid, accesses satisfy platform requirements, and there are no external modifications.

### Independent source and destination

Initially, the source contains A, B, C, D, E, F, G, and the destination interval is disjoint.

Expected completion state:

- The destination contains A, B, C, D, E, F, G in order.
- The source remains unchanged.
- The function returns no value.

A correct result must preserve all 32 bits of each value, including a set high bit.

### Destination begins exactly at the source

Suppose B + 20 equals S.

Each destination word occupies the same location as its corresponding source word. With stable ordinary memory, each write restores the value already present at that word position. The seven words remain A, B, C, D, E, F, G.

This is a case-specific consequence of the access sequence, not a general overlap-safety guarantee.

### Destination begins one word after the source

Suppose D equals S + 4, and the original source contains A, B, C, D, E, F, G.

The expected destination sequence is:

**A, B, C, D, D, D, D.**

The original second through fourth values were retained before any write. However, the fourth destination write replaces the fifth source word with D. The next read therefore obtains D, and subsequent read/write pairs propagate it into the remaining destination positions.

An implementation that always produces A, B, C, D, E, F, G for this layout would not reproduce the supplied access sequence.

### Destination begins four words after the source

Suppose D equals S + 16, with original source values A, B, C, D, E, F, G.

The expected destination sequence is:

**A, B, C, D, A, B, C.**

The first three destination writes overwrite the fifth, sixth, and seventh source words before their later reads. Those reads consequently obtain A, B, and C.

### Later invalid access

Suppose the first four destination writes complete, but the subsequent fifth source read cannot be performed.

No normal completion or status return is promised. The first four writes are not rolled back. The exact failure mechanism is documented only as platform-dependent.

## Uncertainties

The following remain **documented_only**:

- The semantic type or meaning of the seven words.
- The larger destination object’s layout and purpose.
- Whether callers intentionally permit overlap.
- Exact address and pointer widths.
- Endianness and supported alignment.
- Address-overflow behavior.
- Hardware access granularity, ordering, and atomicity.
- Behavior for memory-mapped devices or concurrently modified storage.
- Platform failure handling for invalid reads or writes.

A plausible intent is to populate a seven-word subregion of a larger object while leaving its leading 20 bytes alone. That is an inference from the destination offsets, not an established semantic purpose.

## Evidence

Established static evidence consists of seven four-byte source reads, seven four-byte destination writes, fixed source and destination offsets, the early retention of the second through fourth source values, and an unconditional return without a value.

The representation contains no calls, branches, value checks, allocation, release, or explicit error handling.

The access-order description and conditional overlap outcomes are derived from those visible operations. Platform-specific execution details and higher-level field meanings are not established.

No runtime tests have been performed.
