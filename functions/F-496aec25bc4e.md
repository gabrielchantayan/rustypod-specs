<!-- Trusted provenance: F-496aec25bc4e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-496aec25bc4e

## Purpose

F-496aec25bc4e performs a fixed sequence of memory mutations, reads additional addresses and values from that memory, and finally invokes an unresolved indirect dependency with four arguments.

No higher-level purpose is established. The evidence supports neither a conventional object initializer nor a safe data-copy operation: several inputs serve as both addresses and integer values, some addresses are formed by doubling an input, and the operation accesses fixed low memory addresses.

This contract describes the behavior represented by the supplied static evidence. It does not establish that the operation is executable on a particular platform, that its apparent entry interface is complete, or that the final dependency returns.

## Interface

### Explicit inputs

| Contract name | Represented width | Uses |
|---|---:|---|
| Payload | 32 bits | Stored as a full word; also used as an address base for a read at offset 0x74. |
| Destination base | 32 bits | Address base for an initial write at offset 0x54; also stored into another memory location. |

The represented interface has no result value.

### Additional entry state

Three additional values are consumed without a visible assignment within the operation:

| Contract name | Represented width | Uses |
|---|---:|---|
| Primary base | Unsigned 32 bits | Address base, doubled address component, full-word stored value, byte source, and shift-count source. |
| Secondary base | 32 bits | Address base and byte source. |
| Context base | 32 bits | Address base and full-word stored value. |

These are required entry-state dependencies, not proven explicit parameters. Their calling convention and initialization requirements are **documented_only**.

The following terms describe values derived from entry state:

- **Primary byte:** the low eight bits of Primary base.
- **Secondary byte:** the low eight bits of Secondary base.
- **Doubled primary:** twice Primary base, reduced to 32 bits.
- **Word:** four bytes.
- **Halfword:** two bytes.
- **Byte:** eight bits.

Names in this contract describe roles only. They do not assert object types or ownership.

### Memory layout and preconditions

The operation accesses:

- Fields relative to Payload, Destination base, Primary base, Secondary base, and Context base.
- An address formed from Doubled primary plus 0x54.
- Further addresses obtained from word-sized memory reads.
- A two-byte location at twice a subsequently loaded value.
- A four-byte location at twice another subsequently loaded value.
- Fixed addresses 0x21, 0x3A, 0x67, and 0xA9.

Successful completion requires every actually executed access to be valid with the required width and access permission. The evidence contains no null checks, bounds checks, alignment checks, or range checks.

No disjointness guarantee is established. All addresses may overlap unless an external caller contract rules that out. In particular, a word write can replace bytes that were written separately earlier, and a write can change a later pointer or value read.

Byte order, alignment handling, and the treatment of signed address overflow are **documented_only**. They must not be inferred from the field offsets alone.

## Behavior

### Initial repeated writes

Before the later reads and pointer-dependent mutations, the operation performs a repeated write sequence involving four destinations:

| Destination | Width | Stored value |
|---|---:|---|
| Destination base + 0x54, initially; Doubled primary + 0x54, subsequently | 4 bytes | Payload |
| Secondary base + 0x19 | 1 byte | Primary byte |
| Primary base + 0x11 | 1 byte | Secondary byte |
| Secondary base + 0x34 | 4 bytes | Destination base initially; Doubled primary subsequently |

Within each occurrence, these four writes occur in the table’s order. After the initial occurrence, the evidence contains a fixed, straight-line repetition of the subsequent form. There is no represented condition, variable iteration count, or early exit in this region.

The repetition must not be treated as a single combined update when stores themselves are observable. Even when the final ordinary-memory contents would be identical, repeated stores can matter for overlapping addresses, memory-mapped devices, faults, or externally observed write traces. The exact repetition count is not asserted by this contract.

### Initial captures and context updates

After the repeated region, the operation captures a word from fixed address 0xA9 and then a word from Payload + 0x74.

It next writes one byte at fixed address 0x3A. The resulting byte is the low eight bits of twice Primary byte plus 0x30. This specifies the stored bit pattern without assuming whether a plain character value is signed.

The operation then captures:

1. A word from Context base + 0x14.
2. A word from Primary base + 0x04.

These captures precede the following mutations, in order:

- Context base + 0x0D receives Primary byte.
- Primary base + 0x08 receives byte 0x6C.
- Primary base + 0x0D receives Secondary byte.
- Primary base + 0x09 receives Secondary byte.
- Context base + 0x54 receives the full Context base value.
- Fixed address 0x67 receives the low byte of the earlier word captured from 0xA9.

The operation then reads a word from Context base + 0x44. Call this the **context field value**.

Fixed address 0x21 receives the current byte at fixed address 0x67. This is not an unconditional guarantee that the two locations retain equal values afterward: later accesses can alias either location.

### Interleaved field replacement

The next phase includes intentionally superseded stores. Their occurrence and order are part of the represented behavior:

1. Primary base + 0x10 receives the low byte of the context field value.
2. Secondary base + 0x54 receives the full context field value.
3. Context base + 0x01 receives a byte obtained by right-shifting the signed context field value. The shift count is the low eight bits of Primary base.
4. Secondary base + 0x54 receives the full Primary base value.

For shift counts from 0 through 31, the represented expression is a signed right shift followed by byte truncation. The treatment of negative values depends on signed-shift semantics. Counts of 32 or greater have no established result in this contract.

Next, the operation reads a word from Primary base + 0x40 and performs these updates in order:

- Secondary base + 0x74 receives the word previously captured from Primary base + 0x04.
- Primary base + 0x10 receives the newly read word from offset 0x40.
- Secondary base + 0x44 receives Context base.
- Primary base + 0x09 receives the current byte at fixed address 0x67.
- Context base + 0x09 receives Primary byte.
- Primary base + 0x44 receives the word previously captured from fixed address 0xA9.

The word write at Primary base + 0x10 covers offsets 0x10 through 0x13. Consequently, it can overwrite both the earlier byte at offset 0x10 and the repeated-region byte at offset 0x11.

### Loaded-target updates

The operation reads a word from Primary base + 0x54 and uses it as an address. Call this the **first loaded target**.

The subsequent ordering is:

- Primary base + 0x09 receives the current byte at fixed address 0x67 again.
- First loaded target + 0x0D receives the low byte of the earlier word captured from Payload + 0x74.
- Context base + 0x54 receives the earlier word captured from Context base + 0x14.
- A word is read from Primary base + 0x14.
- Context base + 0x11 receives the low byte captured from Payload + 0x74.
- Primary base + 0x09 receives the low byte of First loaded target.
- Primary base + 0x44 receives the full First loaded target value.

Restoring Context base + 0x54 uses the captured value, not a fresh read of Context base + 0x14.

The operation then reads a word from Context base + 0x04. Call this the **second loaded target**.

### Final memory-dependent updates

A halfword with numeric value 0x0065 is written at twice Second loaded target. This is a two-byte write, not merely a byte write of 0x65. Its byte order is unspecified.

Only after that write does the operation read a word from Second loaded target + 0x64. Call this the **dispatch value**.

The remaining memory operations occur in this order:

1. First loaded target + 0x11 receives the low byte of the previously captured word from Primary base + 0x14.
2. First loaded target + 0x54 receives the four-byte value 100.
3. A word is captured from Second loaded target + 0x74.
4. A word is read from Context base + 0x64; this becomes the **final loaded target**.
5. Context base + 0x09 receives the low byte of Second loaded target.
6. Final loaded target + 0x54 receives Dispatch value.
7. A word containing Final loaded target is written at twice Dispatch value.

The ordering matters if the halfword or any intervening store overlaps a subsequent read. In particular, the word from Second loaded target + 0x74 is captured after the value 100 is written, not before it.

### Indirect invocation and completion

After the last direct memory write, the operation invokes an unresolved indirect dependency with these arguments, in order:

| Position | Value |
|---|---|
| 1 | Numeric value 0x72 |
| 2 | Dispatch value |
| 3 | Word captured earlier from Context base + 0x14 |
| 4 | Word captured later from Second loaded target + 0x74 |

The first argument is represented as an integer constant; the remaining arguments are word-sized values.

No result from the dependency is used. The represented control flow returns without a value if the dependency returns normally. The target’s identity, target-selection mechanism, calling convention, and behavior are **documented_only**.

## Side effects

The direct side effects are the ordered memory writes described above. The operation does not present a transaction boundary, commit flag, or rollback mechanism.

A fault during execution can therefore leave earlier writes visible. No failure status or recovery behavior is represented.

The following distinctions are required:

- A value captured by a word read remains the value used later, even if the original memory location changes.
- A later explicit read observes memory at that later point; it must not be replaced with an earlier cached value merely because the nominal address is the same.
- “Final field value” claims require aliasing assumptions. Later writes through loaded addresses may revisit earlier destinations.
- The indirect dependency may perform additional reads, writes, allocation, release, or control transfer. None of those effects are bounded by the available evidence.

No allocation or release operation is directly identified. Passing an address-like integer to the dependency does not establish ownership transfer. Preservation of globals, memory contents, or object lifetimes across that invocation is not guaranteed.

No concurrency or atomicity contract is established. Source-level access order does not by itself establish hardware memory ordering.

## Dependencies

The only identified function is **F-496aec25bc4e** itself.

One unresolved indirect invocation is present. No opaque function ID is supplied for its target, so this contract does not invent one or assign it another function’s identity.

Its visible contract is limited to the four arguments and invocation position documented above. The evidence does not establish:

- Whether it returns.
- Whether it interprets any argument as a pointer.
- Whether it retains arguments.
- Whether it mutates or releases reachable memory.
- Whether it changes fixed-address state.
- Whether it reports errors through any external mechanism.

Address resolution, memory permissions, alignment support, and arithmetic execution semantics are environmental prerequisites rather than identified function dependencies.

## Acceptance examples

These are conditional acceptance criteria, not executed tests.

### Non-overlapping ordinary memory

Assume all accessed regions are valid, sufficiently aligned, non-overlapping, and unchanged by external actors. Assume the indirect dependency returns without modifying them.

Under those conditions:

- Destination base + 0x54 contains Payload.
- Doubled primary + 0x54 contains Payload.
- Secondary base + 0x34 contains Doubled primary.
- Secondary base + 0x54 contains Primary base, despite the earlier write of the context field value.
- Context base + 0x54 contains the word originally captured from Context base + 0x14, despite its earlier replacement with Context base.
- First loaded target + 0x54 contains the four-byte value 100.
- The final word write at twice Dispatch value contains Final loaded target.

These expectations are invalid if a later destination overlaps one of the listed locations.

### Byte truncation

If Primary byte is 0xE1, the byte written at fixed address 0x3A is 0xF2: twice 0xE1 plus 0x30, reduced to eight bits.

If the captured word from Payload + 0x74 is 0x123456AB, the byte sent to First loaded target + 0x0D and Context base + 0x11 is 0xAB, regardless of the other three bytes.

### Captured versus current values

If the word captured from Context base + 0x14 is later overwritten through an alias, the captured original word still supplies the later restoration at Context base + 0x54 and the third indirect-call argument.

Conversely, if fixed address 0x67 changes through an intervening overlapping write, a subsequent read from that address uses the changed byte rather than necessarily reusing the original low byte from 0xA9.

### Width-sensitive overlap

If the halfword destination at twice Second loaded target overlaps Second loaded target + 0x64, Dispatch value must reflect the preceding halfword write according to the platform’s byte order. Reading Dispatch value before performing that write would not satisfy the represented ordering.

### Fault after partial progress

If initial writes succeed but the later read from Payload + 0x74 faults, the operation has already performed its repeated-region writes and the read from 0xA9. No rollback or indirect invocation is established for that execution.

### Unsupported shift boundary

If Primary byte is 32 or greater, this contract provides no expected result for the signed shift. A test must not assume count masking, zero fill, or saturation without additional platform evidence.

## Uncertainties

The following remain **documented_only**:

- The complete entry calling convention and provenance of the three additional entry-state values.
- Whether the represented function boundary and indirect transfer accurately describe executable control flow.
- The exact repeated-store multiplicity.
- Target byte order, address width, alignment behavior, and signed address-overflow semantics.
- Signed right-shift behavior for negative operands and behavior for oversized shift counts.
- Whether fixed low addresses are ordinary memory, mapped devices, or invalid.
- Any valid object layouts, object sizes, alias restrictions, or lifetime requirements.
- The unresolved dependency’s identity and side effects.
- Any application-level meaning for the numeric constants or field offsets.

**Inferred intent:** some writes resemble linking records or replacing fields, but the evidence does not support adopting that interpretation as a behavioral requirement.

## Evidence

The supplied static representation establishes the apparent two-input, no-result interface; consumption of three additional entry-state values; explicit byte, halfword, and word accesses; ordered captures and overwrites; pointer-dependent destinations; fixed-address accesses; and the final four-argument indirect invocation.

The supplied material also indicates uncertainty in the recovered indirect control transfer and possible overlap among fixed-address symbols. Accordingly, this contract treats fixed-address storage as potentially overlapping memory, not as independent abstract variables.

No implementation access, runtime execution, or runtime tests were performed. The acceptance examples are derived conditions, not observations. No provenance hash is asserted.
