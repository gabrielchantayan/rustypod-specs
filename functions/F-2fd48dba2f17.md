<!-- Trusted provenance: F-2fd48dba2f17; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2fd48dba2f17

## Purpose

F-2fd48dba2f17 performs a fixed sequence of mixed-width memory updates using two explicit inputs and three additional incoming machine-state values, then transfers control through an unresolved indirect target.

The supplied static representation establishes memory-access relationships and their represented order. It does not establish a meaningful application-level purpose. In particular, interpreting this operation as initialization, linking objects, updating a device, or dispatching an event would be **inferred intent**, not established behavior.

This contract describes the visible access trace rather than replacing it with a final-state summary. Repeated writes, overlapping locations, and reads after writes can affect behavior. Their effects cannot safely be collapsed into independent field assignments.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Use |
|---|---:|---|
| Primary input | 32 bits | Written unchanged to a computed destination and also used as an address base for a four-byte read at offset 0x74. |
| Secondary input | 32 bits | Used by the first four-byte write. Later writes target the same address with a different value. |

The primary input is represented as a signed integer, but its object type and pointer validity are unknown. The secondary input is an uninterpreted four-byte value.

### Additional incoming state

Three incoming machine-state values are used without visible initialization or explicit parameter binding. This contract calls them:

- **Index/base input:** a represented unsigned 32-bit value used for address calculation, byte extraction, multiplication by two, and a shift count.
- **First base input:** a represented 32-bit value used principally as a memory base, and also reduced to its low byte.
- **Second base input:** a represented 32-bit value used as a memory base and stored in four-byte destinations.

These are descriptive interface labels, not claims that the values name valid objects. The actual calling convention supplying them is **documented_only**.

### Output and completion

No meaningful return value is established. The visible body ends with an indirect invocation and then a return in the supplied representation.

The unresolved target receives four represented 32-bit arguments, in this order:

1. 0x72.
2. A value read through a dynamically obtained base at offset 0x64.
3. The value saved earlier from the second base input at offset 0x14.
4. A later value read through that same dynamically obtained base at offset 0x74.

Whether the actual control transfer behaves as an ordinary returning call, a tail transfer, or another form of dispatch is **documented_only**.

### Preconditions and layout

There are no visible null checks, bounds checks, alignment checks, or input-validation branches.

For execution to reach the final transfer, every accessed address must support its represented access width and access direction. This includes addresses derived from values loaded during execution, not merely the three incoming bases.

The representation uses:

- One-byte writes.
- One two-byte write.
- Four-byte reads and writes.
- Absolute locations at 0x21, 0x3A, 0x67, and 0xA9.

No enclosing structure size, allocation boundary, alignment guarantee, or non-aliasing guarantee is established. Numeric offsets describe accesses, not a recovered public data structure.

## Behavior

### Width and arithmetic rules

Four-byte values are retained as 32-bit quantities. Byte stores retain the low eight bits of the supplied value. Consequently, byte results do not preserve a value’s signedness or higher bits.

Doubling the unsigned index/base input has a 32-bit unsigned result. Address expressions using that unsigned value are consistent with modulo-2³² arithmetic in the represented model. Actual address-space behavior is not independently established.

Other expressions use signed-looking values as addresses or multiply them by two. Signed overflow behavior and the mapping from those expressions to machine addresses are **documented_only**. This contract does not silently assign modular semantics to every signed expression.

The shift described below uses the low eight bits of the index/base input as its count. Neither a count restriction to 0–31 nor hardware masking of larger counts is established. The behavior of a right shift of a negative signed-looking value is also unresolved.

### Repeated-write prefix

The operation begins by writing the secondary input as four bytes at the first base input plus 0x34.

A fixed, repeated write pattern follows. Its destinations and ordering are:

| Relative order within the pattern | Destination | Width | Value |
|---|---|---:|---|
| First | Twice the index/base input, plus 0x54 | 4 bytes | Primary input |
| Second | First base input plus 0x19 | 1 byte | Low byte of the index/base input |
| Third | Index/base input plus 0x11 | 1 byte | Low byte of the first base input |
| Fourth | First base input plus 0x34 | 4 bytes | Twice the index/base input |

The supplied representation contains many consecutive occurrences of this same pattern, without a represented intervening read, branch, or call. It is not an input-controlled loop. The exact occurrence count is **documented_only** in this contract; it is not asserted as a verified numeric constant.

All represented occurrences precede the remaining behavior. A single occurrence is not an equivalent replacement where write multiplicity is observable.

If these destinations are disjoint ordinary memory, the initial secondary-input write is superseded by the doubled-index write. If they overlap, byte-level effects depend on destination overlap and target byte order. If the destinations are device-backed or otherwise externally observed, even repeated stores of identical values may matter.

### Initial reads and byte publication

After the repeated prefix, the operation obtains:

- Four bytes from absolute address 0xA9.
- Four bytes from the primary input plus 0x74.

Those reads occur in that order. The earlier prefix may have modified either source if addresses alias.

Next, it writes one byte at absolute address 0x3A. The stored byte is the low-eight-bit result of doubling the low byte of the index/base input and adding 0x30.

It then reads, in order:

- Four bytes from the second base input plus 0x14.
- Four bytes from the index/base input plus 0x04.

Both results are retained for later writes or final arguments. They are not automatically refreshed when their source addresses change.

The subsequent stores establish this relative order:

- A byte at the second base input plus 0x0D receives the low byte of the index/base input.
- A byte at the index/base input plus 0x08 receives 0x6C.
- Bytes at the index/base input plus 0x0D and then plus 0x09 receive the low byte of the first base input.
- Four bytes at the second base input plus 0x54 receive the second base input itself.
- A byte at absolute address 0x67 receives the low byte of the previously saved four-byte value from 0xA9.

The self-address store at offset 0x54 does not establish object ownership or a durable self-link; that location is written again later.

### Intermediate reads, overwrites, and publication

A four-byte read from the second base input plus 0x44 occurs after the preceding stores. Its result is used for several subsequent writes.

The byte at absolute address 0x67 is then read and copied to absolute address 0x21. This is a read of the byte location, not necessarily a fresh byte extraction from the original four-byte snapshot.

The value obtained from the second base input plus 0x44 is published as follows:

- Its low byte is written at the index/base input plus 0x10.
- Its full four-byte value is written at the first base input plus 0x54.
- A shifted result is reduced to a byte and written at the second base input plus 0x01.

The shift count is the low eight bits of the index/base input. Acceptance of this step is conditional on the platform supplying defined semantics for the particular value and count.

The first base input plus 0x54 is then overwritten with the full index/base input.

Next, four bytes are read from the index/base input plus 0x40. Following that read:

- The earlier snapshot from the index/base input plus 0x04 is written at the first base input plus 0x74.
- The newly read four-byte value is written at the index/base input plus 0x10.
- The second base input is written at the first base input plus 0x44.

The four-byte write at offset 0x10 covers offsets 0x10 through 0x13. It therefore overwrites the earlier single-byte store at offset 0x10 and can overwrite the prefix’s byte at offset 0x11.

The byte at absolute address 0x67 is read again and stored at the index/base input plus 0x09. A byte at the second base input plus 0x09 then receives the low byte of the index/base input. The saved four-byte snapshot from absolute address 0xA9 is written at the index/base input plus 0x44.

### Dynamically addressed updates

A four-byte read from the index/base input plus 0x54 supplies a **first derived base**.

After that read, the byte at absolute address 0x67 is read once more and written at the index/base input plus 0x09. The low byte of the earlier snapshot from the primary input plus 0x74 is then written at the first derived base plus 0x0D.

The saved value originally read from the second base input plus 0x14 is written as four bytes at the second base input plus 0x54. This overwrites the earlier self-address store, subject to aliasing effects.

Four bytes are then read from the index/base input plus 0x14. After this read:

- The second base input plus 0x11 receives the low byte of the saved primary-relative snapshot.
- The index/base input plus 0x09 receives the low byte of the first derived base.
- The index/base input plus 0x44 receives the full first derived base.

A four-byte read from the second base input plus 0x04 supplies a **second derived base**. A two-byte value of 0x0065 is written at twice this second derived base.

Only after that two-byte write does the operation read four bytes from the second derived base plus 0x64. Call this the **link value**. The two-byte store may affect that read if their byte ranges overlap.

The low byte of the saved index-relative snapshot from offset 0x14 is written at the first derived base plus 0x11. Four bytes containing 100, equivalently 0x00000064, are then written at the first derived base plus 0x54.

The operation next reads:

1. Four bytes from the second derived base plus 0x74, retained for the fourth outgoing argument.
2. Four bytes from the second base input plus 0x64, yielding a **third derived base**.

Before the final transfer, the remaining writes occur in this order:

- The low byte of the second derived base is stored at the second base input plus 0x09.
- The link value is stored as four bytes at the third derived base plus 0x54.
- The third derived base is stored as four bytes at twice the link value.

The indirect invocation follows these writes.

## Side effects

The visible body modifies both input-derived memory and absolute-addressed storage. Its writes are neither transactional nor accompanied by visible rollback.

A later invalid access can therefore occur after earlier changes have already taken effect. The representation provides no error code, exception translation, recovery branch, or cleanup path.

Overlapping destinations must be interpreted in access order. In particular:

- Stores to offset 0x09 of the index/base input have several different sources over time.
- Stores to offset 0x44 of the index/base input replace a saved absolute-location value with the first derived base.
- Stores to offset 0x54 of the first base input replace one four-byte value with another.
- A four-byte store can cover locations previously written as individual bytes.

The visible body contains no allocation, deallocation, reference-count adjustment, or explicit ownership transfer. Storing a numeric value that may be an address does not prove that the destination acquires ownership.

The final indirect target may have additional side effects. Its preservation of memory, globals, incoming state, and ownership is unknown. No final-memory guarantee here extends unconditionally across that invocation.

## Dependencies

No named direct callee is established.

The final operation depends on an unresolved indirect target whose opaque function ID is unavailable. This document does not assign an invented identity or infer its semantics from the four arguments.

Other execution dependencies are:

- The calling convention that supplies the three additional incoming values.
- The memory map, including low absolute addresses.
- Target byte order and alignment rules.
- Signed arithmetic and shift semantics.
- The validity of dynamically obtained addresses.
- The observability of repeated memory accesses.

No dependency is assumed to preserve globals or to acquire, release, or retain memory.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Disjoint ordinary memory

Assume all accessed regions are valid, disjoint except where explicitly shared by identical addresses, and unaffected by external agents.

Immediately before the final indirect invocation:

- The first base input plus 0x34 contains the doubled unsigned index/base input.
- The first base input plus 0x54 contains the full index/base input.
- The second base input plus 0x54 contains the value previously read from that base’s offset 0x14.
- The first derived base plus 0x54 contains 100.

These observations are invalid if other listed writes alias those regions.

### Byte truncation

If the low byte of the index/base input is 0xF0, the byte written to absolute address 0x3A is 0x10: doubling and adding 0x30 retains only the low eight bits.

This example does not establish the validity of addresses computed from the full index/base input.

### Saved snapshot versus later contents

If the four-byte value read from the primary input plus 0x74 has low byte 0xAB, the later byte writes using that snapshot supply 0xAB even if the original source location changes meanwhile.

Conversely, the copies sourced from absolute address 0x67 use reads at their respective positions in the trace. Aliasing or external changes can distinguish those reads from the earlier saved absolute-location snapshot.

### Access failure

If the primary-relative read at offset 0x74 is invalid, successful return is not required by this contract. The repeated-write prefix has already been reached before that access. No rollback is established.

The precise fault mechanism remains platform-dependent.

### Large shift count

If the low byte used as the shift count is 0x20, no portable shifted-byte result is asserted. A platform-specific masking rule cannot be assumed without additional evidence.

### Final invocation boundary

If execution reaches the indirect target, its four arguments must match the values and order specified in the interface. If that target subsequently modifies any listed destination, this does not contradict the pre-invocation write trace.

## Uncertainties

The following are **documented_only**:

- Actual entry calling convention and function boundary.
- Application-level purpose and field meanings.
- Exact numeric multiplicity of the repeated prefix.
- Whether the represented accesses correspond one-for-one to observable machine transactions.
- Byte order, alignment requirements, and validity of low absolute addresses.
- Signed-overflow behavior and out-of-range shift behavior.
- The identity, return behavior, ownership rules, and side effects of the indirect target.
- Whether any storage is device-backed, shared concurrently, or externally mutable.
- Whether a returning invocation preserves any pre-invocation memory value.

These unknowns restrict the contract; they are not permission to assume conventional heap objects, harmless repetition, or a pure final callee.

## Evidence

**Established static evidence:** the supplied representation contains mixed-width stores, four-byte reads, a repeated prefix, address-dependent later accesses, absolute-addressed byte publication, and a final unresolved four-argument indirect invocation.

**Inferred intent:** none is required for this contract. Terms such as “snapshot,” “derived base,” and “link value” distinguish data roles without assigning application semantics.

**Evidence limits:** the supplied representation is the sole behavioral evidence. Its comments and recovered typing do not independently prove runtime properties. No implementation inspection beyond that supplied material, runtime execution, fault experiment, or acceptance test has been performed.
