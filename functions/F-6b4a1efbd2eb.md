<!-- Trusted provenance: F-6b4a1efbd2eb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6b4a1efbd2eb

## Purpose

The supplied static evidence describes a state-mutating operation that reads a 32-bit value, derives several addresses from it and from incoming execution state, performs ordered writes of mixed widths, and invokes an unresolved indirect target.

No application-level purpose is established. The evidence does not justify describing this operation as formatting, serialization, initialization, list manipulation, or device control. Several constants resemble characters, but their presence does not establish text semantics.

This contract distinguishes:

- **Established static evidence:** access widths, address relationships, value dependencies, and ordering explicitly represented in the supplied material.
- **Inferred intent:** none adopted as a behavioral requirement.
- **documented_only:** behavior requiring unavailable architecture, calling-convention, memory-map, or indirect-target information.

The contract describes the visible execution prefix. It does not guarantee successful completion, a normal return, or preservation of state across the unresolved invocation.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First address input | 32 bits | Supplies an address for a one-byte write at offset `0x09`. |
| Second address input | 32 bits | Supplies an address for the initial four-byte read at offset `0x74`. |
| Third address input | 32 bits | Supplies initial write addresses and, after subtraction of `0x96`, the basis for further address calculations. |
| Payload input | 32 bits | Written as four bytes at offset `0x14` from the original third address. |

These are address-bearing or raw bit-pattern inputs, not established references to particular object types. No object size, ownership convention, or nullability guarantee is supplied.

### Additional incoming state

The visible behavior also depends on:

- An incoming one-byte execution-state value.
- An incoming 32-bit execution-state value used both as stored data and as an address-calculation operand.
- A four-byte global value supplying another address base.
- Fixed-address storage and memory reached through derived addresses.
- An unresolved indirect control-transfer target.

The two incoming execution-state values are not established as ordinary explicit arguments. Their origin and validity are **documented_only**.

### Contract notation

The following labels describe relationships, not implementation identifiers:

- **Seed:** the initial four-byte value read from the second address plus `0x74`.
- **Double-seed:** twice the seed.
- **Quad-seed:** four times the seed.
- **Adjusted third:** the original third address minus `0x96`.
- **Scaled-third:** four times the adjusted third.
- **Octuple-third:** eight times the adjusted third.
- **Global base:** the four-byte global value loaded after the initial scaled-third writes.
- **Incoming word** and **incoming byte:** the additional execution-state values described above.

Address calculations are represented with 32-bit operands. Low-byte conversions retain eight bits; two-byte and four-byte accesses have widths of 16 and 32 bits respectively.

Modulo-\(2^{32}\) arithmetic is a useful conditional machine-level interpretation, but the supplied material does not independently establish overflow semantics for every signed expression. Examples below avoid arithmetic overflow unless explicitly discussing it.

### Outputs and preconditions

No explicit result value is established. Outputs are memory mutations and effects of the indirect invocation.

For execution to reach that invocation, every preceding access must be permitted by the actual execution environment, with acceptable alignment and access width. There are no visible checks for these conditions.

Validating only the three original addresses is insufficient. The operation also accesses seed-derived addresses, execution-state-derived addresses, fixed low addresses, and addresses obtained from later memory reads.

## Behavior

### Initial read and overlapping writes

The seed is read before any visible write. Its saved value continues to drive later calculations even if subsequent writes overlap its original storage.

The first mutations, in order, are:

| Destination | Width | Written value |
|---|---:|---|
| Original third address + `0x15` | 1 byte | Incoming byte |
| Original third address + `0x54` | 4 bytes | Incoming word |
| First address + `0x09` | 1 byte | Low byte of double-seed |
| Original third address + `0x14` | 4 bytes | Payload input |

The final write necessarily covers the earlier byte at offset `0x15`. For ordinary writable memory, after that four-byte write, the byte at `0x15` is determined by the payload’s byte representation, not by the incoming byte. Byte order is not established.

The incoming-byte write remains part of the access history. It cannot be discarded when intermediate writes are observable.

### Constant writes and scaled-third activity

Before the global base is loaded, the operation performs these accesses in the following order:

| Destination or source | Access |
|---|---|
| Quad-seed | Write one byte containing `0x3A` |
| Eight times the seed | Write two bytes containing numeric value `0x003A` |
| Quad-seed + `0x20` | Write four bytes containing `0x0000003A` |
| Scaled-third + `0x20` | Write four bytes containing `0x00000020` |
| Scaled-third + `0x20` | Write four bytes containing `0x0000002F` |
| Scaled-third + `0x04` | Read one unsigned byte |
| Scaled-third + `0x20` | Write the byte just read, zero-extended to 32 bits |
| Scaled-third + `0x20` | Write four bytes containing `0x00000020` |

Thus the final ordinary-memory value at scaled-third plus `0x20` is `0x00000020`, assuming no later aliasing mutation. The intermediate stores and intervening byte read are nevertheless observable operations.

The global base is then loaded once. A four-byte value `0x00000020` is written at absolute address `0x60`.

The loaded global base is retained for subsequent address calculations; no later reload of its supplying global is visible.

### Repeated publication writes

Next, quad-seed is written as four bytes at twice the incoming word plus `0x54`. This is followed by:

- A byte write of the low byte of scaled-third at global base plus `0x19`.
- A byte write of the low byte of global base at scaled-third plus `0x11`.
- A four-byte write of twice the incoming word at global base plus `0x34`.

The visible body then contains a long, fixed repetition of a related four-write group:

| Order within each group | Destination | Width and value |
|---:|---|---|
| 1 | Octuple-third + `0x54` | Four-byte quad-seed |
| 2 | Global base + `0x19` | Low byte of scaled-third |
| 3 | Scaled-third + `0x11` | Low byte of global base |
| 4 | Global base + `0x34` | Four-byte octuple-third |

There is no visible condition or evolving index within this repeated region. The destination addresses and written values remain unchanged across its copies.

The exact repetition count is **documented_only** here; it has not been independently enumerated. A side-effect-sensitive implementation cannot treat this as permission to replace all copies with one group. For ordinary memory without interference, the repeated groups converge on the same stored values. For devices or other access-sensitive storage, multiplicity may be externally significant.

### Late snapshots and byte publication

After the repeated region, the operation takes several snapshots and performs further mutations. Their relative order matters because the addresses may alias.

First, it reads a four-byte snapshot from absolute address `0xA9`, then a four-byte snapshot from quad-seed plus `0x74`.

It writes one byte at absolute address `0x3A`, retaining the low eight bits of eight times the adjusted third plus `0x30`.

It then reads:

- Four bytes from double-seed plus `0x14`, retained for both a later memory write and an indirect-call argument.
- Four bytes from scaled-third plus `0x04`.

Those reads occur after the earlier writes and therefore observe any preceding aliasing effects. The four-byte read at scaled-third plus `0x04` is distinct from the earlier one-byte read at that address.

The next publication sequence writes:

1. The low byte of scaled-third at double-seed plus `0x0D`.
2. `0x6C` at scaled-third plus `0x08`.
3. The low byte of global base at scaled-third plus `0x0D`.
4. The low byte of global base at scaled-third plus `0x09`.
5. Double-seed as four bytes at double-seed plus `0x54`.
6. The low byte of the saved `0xA9` snapshot at absolute address `0x67`.

A four-byte value is then read from double-seed plus `0x44`. After that read, the byte currently at absolute address `0x67` is copied to absolute address `0x21`.

The value read from double-seed plus `0x44` supplies:

- A low-byte write at scaled-third plus `0x10`.
- A four-byte write at global base plus `0x54`.
- A shifted low-byte write at double-seed plus `0x01`.

The shift is represented as a signed right shift, with a count taken from the low eight bits of scaled-third. The behavior for negative operands and counts outside `0` through `31` is **documented_only**; no saturation or count-masking rule is established.

Immediately after these writes, global base plus `0x54` is overwritten with the four-byte scaled-third value.

### Cross-linked writes and final address reads

The remaining memory activity has several overwrite and snapshot dependencies:

- Four bytes are read from scaled-third plus `0x40`.
- The earlier four-byte snapshot from scaled-third plus `0x04` is written at global base plus `0x74`.
- The snapshot from scaled-third plus `0x40` is written at scaled-third plus `0x10`, covering the earlier one-byte write there.
- Double-seed is written as four bytes at global base plus `0x44`.
- The current byte at absolute address `0x67` is written at scaled-third plus `0x09`.
- The low byte of scaled-third is written at double-seed plus `0x09`.
- The saved `0xA9` snapshot is written as four bytes at scaled-third plus `0x44`.

A four-byte value is then read from scaled-third plus `0x54`; call this the **late destination**. The current byte at absolute address `0x67` is again written at scaled-third plus `0x09`. This is a second publication, not merely a final-state description.

The low byte of the earlier snapshot from quad-seed plus `0x74` is written at late destination plus `0x0D`. The saved value from double-seed plus `0x14` is written at double-seed plus `0x54`.

Next, four bytes are read from scaled-third plus `0x14`. The low byte of the quad-seed snapshot is written at double-seed plus `0x11`; the low byte of late destination is written at scaled-third plus `0x09`; and late destination is written as four bytes at scaled-third plus `0x44`.

The final visible address-discovery phase is:

| Event | Required relationship |
|---|---|
| Read an address-bearing word | Four bytes from double-seed + `0x04`; designate the result **secondary base** |
| Write a halfword | Numeric `0x0065` at twice secondary base |
| Read an address-bearing word | Four bytes from secondary base + `0x64`; designate the result **tertiary value** |
| Publish a byte | Low byte of the saved scaled-third + `0x14` value at late destination + `0x11` |
| Publish a word | Four-byte decimal `100` at late destination + `0x54` |
| Capture the last argument snapshot | Four bytes from secondary base + `0x74` |
| Read another address-bearing word | Four bytes from double-seed + `0x64`; designate the result **final destination** |
| Publish a byte | Low byte of secondary base at double-seed + `0x09` |
| Publish a word | Tertiary value at final destination + `0x54` |
| Publish a word | Final destination at twice tertiary value |

The decimal constant `100` and offset `0x64` have the same numeric value but different roles: one is stored data, the other an address displacement.

### Indirect invocation

After the final visible write, the unresolved target is invoked with four represented arguments, in this order:

1. `0x72`.
2. Tertiary value.
3. The saved four-byte value from double-seed plus `0x14`.
4. The saved four-byte value from secondary base plus `0x74`.

The visible continuation contains a return without an explicit result. Whether control actually returns from the target is unknown.

## Side effects

The established footprint includes explicit-input-relative memory, arithmetic-derived addresses, later pointer-like values, global-base-relative storage, and absolute addresses `0x21`, `0x3A`, `0x60`, `0x67`, and `0xA9`.

No allocation, release, ownership transfer, synchronization primitive, rollback, or explicit error reporting is visible before the indirect invocation. This does not establish what the target does.

Aliasing is not prohibited. Therefore:

- Earlier writes may affect later reads, including address-bearing reads.
- Saved snapshots remain tied to their read times.
- A location’s final value cannot generally be derived from its first write.
- Post-invocation memory cannot be inferred solely from the visible prefix.

No failure branch or recoverable error value is shown. Faults, traps, alignment failures, and device-specific failures are **documented_only**. A failure after some writes may leave partial state changes; no restoration is visible.

## Dependencies

No resolved external function dependency is identified by an opaque function ID in the supplied evidence.

The indirect target is a dependency with unavailable identity and contract. No substitute ID is invented. Its argument interpretation, return convention, memory effects, global preservation, ownership behavior, and termination behavior are **documented_only**.

Environmental dependencies include address width, byte order, alignment rules, signed arithmetic and shift semantics, low-address mappings, and whether repeated accesses have device-visible effects.

## Acceptance examples

These are conditional static acceptance criteria, not results of executed tests.

### Payload overwrite

Given accessible ordinary memory and no interfering aliases, the incoming byte is first written at original third plus `0x15`. The following payload write at original third plus `0x14` replaces that byte. Acceptance requires preserving the write order, not retaining the incoming byte as the final value.

### Byte truncation

Given a seed of `0x00000081`, double-seed is `0x00000102`. The byte written at first address plus `0x09` is `0x02`. A range rejection, saturation to `0xFF`, or multi-byte write is not supported.

### Read between constant writes

Given that the byte read at scaled-third plus `0x04` is `0xAB`, the successive four-byte values written at scaled-third plus `0x20` are `0x20`, `0x2F`, `0xAB`, and `0x20`. The `0xAB` is zero-extended, not sign-extended.

### Snapshot versus later memory

If a later aliasing write changes double-seed plus `0x14`, the indirect invocation still receives the value captured by the earlier read. Acceptance must distinguish the saved argument from memory contents at invocation time.

### Zero seed

A zero seed produces writes at address zero and other low addresses. No visible early exit handles this case. Whether execution faults or succeeds depends on the memory environment.

### Invocation effects

If the indirect target modifies global storage or releases memory, the visible prefix establishes no preservation guarantee that would prohibit those effects. Acceptance of post-call state requires the target’s separate contract.

## Uncertainties

The following remain **documented_only**:

- The operation’s application-level purpose.
- The provenance and calling-convention role of incoming execution state.
- Exact multiplicity of the fixed repeated write group.
- Byte order, alignment requirements, and memory-map validity.
- Signed-overflow interpretation and oversized shift behavior.
- Whether accesses target ordinary memory, devices, or overlapping global storage.
- The indirect target’s identity, effects, and return behavior.
- Runtime success, failure modes, timing, and concurrency behavior.

These limitations do not justify adding validation, suppressing repeated writes, assigning ownership, or inventing a conventional data structure.

## Evidence

This contract is based exclusively on the supplied static representation of F-6b4a1efbd2eb. It supports the access widths, constants, address relationships, snapshot dependencies, and represented ordering described above.

No implementation access, independent disassembly, runtime execution, or tests were performed. Conditional examples are derived expectations, not observed outcomes. No snapshot hash or independent provenance claim is supplied.
