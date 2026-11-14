<!-- Trusted provenance: F-f73a3eca298a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f73a3eca298a

## Purpose

F-f73a3eca298a performs an ordered series of direct memory writes, gathers additional values through several address-bearing inputs, and transfers control to an unresolved indirect destination.

The supplied static representation establishes memory-access behavior, not a recognizable application-level purpose. It does not establish that the affected addresses represent ordinary objects, hardware registers, a table, or a serialized format. Repeated writes and low absolute addresses therefore remain observable parts of the described behavior; they must not be dismissed as redundant initialization.

This contract distinguishes:

- **Established static evidence:** operations and ordering represented in the supplied material.
- **Inferred interpretation:** a 32-bit machine-state model used to describe address and value calculations.
- **documented_only:** behavior for which the supplied material does not establish a reliable executable contract.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Width | Established use |
|---|---:|---|
| Primary input | 32 bits | Written repeatedly as an unchanged word; also used as an address base for a word read at offset `0x74`. |
| Secondary input | 32 bits | Written once to an ambient-base-relative destination before that destination receives repeated replacement writes. |

The primary input has a signed integer presentation in the supplied material, but its uses include address formation and raw word storage. No numeric range, sign restriction, or object type is established.

### Ambient inputs

Three additional entry-state values are required even though they are absent from the displayed parameter list. This contract calls them **base A**, **base B**, and **base C**. These are descriptive interface labels, not recovered source identifiers.

- **Base A** is represented as an unsigned 32-bit quantity. It supplies address bases, a doubling operand, a shift-count contribution, and an eight-bit value.
- **Base B** supplies address bases and its low eight bits.
- **Base C** supplies address bases and is itself stored into memory.

A caller supplying only the two explicit inputs has not supplied enough information to reproduce the behavior. The mechanism by which the ambient values arrive is **documented_only**.

### Output and completion

No explicit return value is established.

The final represented action invokes an unresolved indirect destination with four values. The representation then shows completion, but the original transfer mechanism, destination, returnability, and calling convention are not established. Normal return to the caller must not be promised.

### Memory requirements and ownership

For a fault-free execution, every dynamically reached read and write must be valid for its access width. This includes addresses obtained from earlier memory reads, doubled address-like values, and low absolute addresses.

No allocation, release, ownership transfer, bounds check, null check, or access validation is visible before the indirect transfer. Address-bearing values are not evidence of ownership. The indirect destination’s memory and lifetime behavior is unknown.

## Behavior

### Widths and address model

The visible accesses use three widths:

- Byte: eight bits.
- Halfword: sixteen bits.
- Word: thirty-two bits.

Offsets are byte offsets. Narrow writes affect only the addressed byte or halfword, subject to the target’s actual memory semantics. Wider writes may overlap and replace earlier narrow writes.

For address descriptions below, doubling base A means its 32-bit unsigned doubling. The derived repeated-word destination is that result plus `0x54`. A 32-bit modular address model is consistent with the representation, but target address wrapping, alignment handling, and fault behavior are **documented_only**.

The low byte of a value means its least significant eight bits. Byte interpretation as signed or unsigned does not change those stored bits.

### Initial repeated-write region

The function first writes the complete secondary input to the word at base B plus `0x34`.

It then performs a fixed repeated pattern over four destinations:

| Destination | Access width | Written value |
|---|---:|---|
| Twice base A, plus `0x54` | Word | Primary input |
| Base B plus `0x19` | Byte | Low byte of base A |
| Base A plus `0x11` | Byte | Low byte of base B |
| Base B plus `0x34` | Word | Twice base A |

Within each repetition, the order is the table order. The pattern does not advance an index or vary its values. No intervening memory reads or calls are represented within this repeated region.

The initial secondary-input write is therefore temporary when the destinations are ordinary, non-overlapping writable memory and execution reaches the replacement writes. It is still an observable write. A device, access monitor, fault, or overlapping destination can distinguish it from an implementation that writes only the eventual value.

The exact repetition count is not asserted by this contract. Preservation of multiplicity, rather than replacement by a single final-state update, remains a requirement for trace-sensitive use; numerical trace certification is **documented_only** here.

### Captured values and initial publication

After the repeated region, the function captures a word from absolute address `0xA9`, followed by a word from the primary input plus `0x74`.

It next writes a byte to absolute address `0x3A`. That byte is the low eight bits of twice the low byte of base A plus `0x30`. This is arithmetic on bits, not an established decimal-digit conversion; no restriction to digit-like inputs is visible.

The next captured words come from base C plus `0x14`, followed by base A plus `0x04`. These captures occur before subsequent stores that might overlap their source locations.

The following publication sequence has these ordering properties:

- The low byte of base A is written at base C plus `0x0D`.
- Byte `0x6C` is written at base A plus `0x08`.
- The low byte of base B is written at base A plus `0x0D`, then at base A plus `0x09`.
- Base C is written as a word at base C plus `0x54`.
- The low byte of the word captured from absolute address `0xA9` is written to absolute address `0x67`.

The word at base C plus `0x44` is read only after these stores. Consequently, if any preceding store overlaps that word, the read can observe the modified bytes.

After that read, the byte at absolute address `0x67` is copied to absolute address `0x21`. This is a memory-sourced copy at that point, not a guarantee that the original captured byte is reused independently of intervening aliasing.

### Replacement writes and pointer selection

The word obtained from base C plus `0x44` supplies several different-width results:

- Its low byte is written at base A plus `0x10`.
- Its complete word is written at base B plus `0x54`.
- A shifted form supplies a byte written at base C plus `0x01`.

The shift count is the low eight bits of base A. The shifted operand has a signed 32-bit presentation. Arithmetic versus logical right-shift behavior, and behavior for counts of 32 or greater, are **documented_only**. The source presentation is insufficient to impose a portable result for those cases.

The word at base B plus `0x54` is then replaced by base A.

Next, a word is read from base A plus `0x40`. Only after that capture:

- The previously captured word from base A plus `0x04` is written at base B plus `0x74`.
- The newly captured word from base A plus `0x40` is written at base A plus `0x10`.
- Base C is written at base B plus `0x44`.

The word write at base A plus `0x10` overlaps the earlier byte writes at offsets `0x10` and `0x11`. Final-state descriptions must account for that overlap.

The byte at absolute address `0x67` is then copied to base A plus `0x09`. The low byte of base A is written at base C plus `0x09`, and the initially captured word from absolute address `0xA9` is written at base A plus `0x44`.

A word read from base A plus `0x54` now selects a further address base, called the **selected base**. The byte at absolute address `0x67` is copied to base A plus `0x09` again. These two copies are separate memory reads with intervening operations; aliasing may make their values differ.

### Selected-base updates and final dependent accesses

The low byte of the earlier capture from the primary input plus `0x74` is written at the selected base plus `0x0D`.

The saved word originally read from base C plus `0x14` is written at base C plus `0x54`. This replaces the earlier base-C-valued write there, unless aliasing has introduced additional effects.

A word is then captured from base A plus `0x14`. After that capture:

- The same low byte from the primary-input-relative capture is written at base C plus `0x11`.
- The low byte of the selected base is written at base A plus `0x09`.
- The selected base is written as a word at base A plus `0x44`.

The function next reads a word from base C plus `0x04`; call this the **link value**. It writes the sixteen-bit value `0x0065` to twice the link value, then reads a word from the link value plus `0x64`; call that the **transfer value**. The halfword write precedes this read and can affect it if the addresses overlap.

The low byte of the saved base-A-plus-`0x14` capture is written at the selected base plus `0x11`. The complete word value `100`, equivalently `0x64`, is written at the selected base plus `0x54`.

A word is captured from the link value plus `0x74`. A separate word read from base C plus `0x64` supplies the **final base**.

The remaining direct writes occur in this order:

1. The low byte of the link value at base C plus `0x09`.
2. The transfer value as a word at the final base plus `0x54`.
3. The final base as a word at twice the transfer value.

The unresolved indirect destination then receives these four arguments, in order:

| Position | Value |
|---|---|
| First | `0x72` |
| Second | Transfer value |
| Third | Saved word originally read from base C plus `0x14` |
| Fourth | Saved word read from the link value plus `0x74` |

The last argument is a captured value, not a required reread after the final writes.

## Side effects

The established direct side effects are memory writes. Some destinations are base-relative; others are low absolute or doubled addresses. No independent global-preservation guarantee exists.

Aliasing is unrestricted by the visible interface. In particular:

- The primary input can address storage that earlier writes modify before its offset-`0x74` read.
- Bases A, B, and C can designate overlapping regions.
- Absolute locations can overlap base-relative accesses.
- Loaded address-like values can point back into previously accessed storage.
- A captured word remains the captured value, while a later memory read can observe intervening modifications.

The described order is the order in the supplied static representation. It does not establish atomicity, memory barriers, inter-thread visibility rules, or device-bus transaction boundaries.

No error code or recovery path is visible. If an access fails, earlier writes may already have occurred. Rollback and continued execution are not established.

## Dependencies

No identified direct callee is present in the supplied evidence.

One unresolved indirect dependency follows the final direct write. No opaque function ID for that destination was supplied, so none is invented here.

Its destination-selection mechanism, preconditions, return behavior, global effects, register preservation, exceptions, and memory ownership behavior are all **documented_only**. The four outgoing values are established; they do not establish the dependency’s semantic parameter types.

## Acceptance examples

These are conditional static acceptance examples, not executed tests.

### Non-overlapping ordinary memory

Assume valid ordinary memory, non-overlapping relevant regions, supported alignment, and execution reaching the indirect transfer.

Before that transfer:

- Base B plus `0x34` contains twice base A, not the secondary input.
- The repeated-word destination contains the primary input.
- Base A plus `0x10` contains the captured word from base A plus `0x40`.
- Base C plus `0x54` contains the saved word from base C plus `0x14`.
- The selected base plus `0x54` contains word value `100`.

These are pre-transfer assertions only. The dependency may change memory.

### Byte arithmetic wraps

If the low byte of base A is `0xF0`, the byte written to absolute address `0x3A` is `0x10`: twice `0xF0` plus `0x30`, reduced to eight bits.

Acceptance does not require a printable character.

### Word and byte overlap

Assume no external aliasing. The byte write at base A plus `0x10` and the earlier byte write at base A plus `0x11` do not independently survive the later word write starting at base A plus `0x10`.

Acceptance requires the later word’s byte representation across offsets `0x10` through `0x13`. The exact byte order depends on target endianness.

### Captured value survives later mutation

If the final writes alter the storage at the link value plus `0x74`, the fourth outgoing argument nevertheless remains the word captured before those writes.

An implementation that rereads that location at transfer time is not equivalent under this condition.

### Fault after partial progress

If the word read from the primary input plus `0x74` faults, the initial repeated-write region has already been reached before that read. The contract does not require those writes to be undone or an error value to be returned.

## Uncertainties

The following remain **documented_only**:

- Application-level purpose and object identities.
- Ambient-input delivery and actual calling convention.
- Exact repeated-write count for numerical trace certification.
- Target endianness, alignment requirements, and address-space mapping.
- Signed shift semantics and excessive shift-count behavior.
- Machine-level consequences of signed arithmetic overflow in address expressions.
- Whether accesses target ordinary memory, shared memory, or devices.
- Whether the indirect transfer returns, and all effects after entering it.
- Whether the supplied representation faithfully separates executable operations from incorrectly interpreted data.

These limits prevent a claim of complete executable equivalence. They do not justify deleting established accesses or assuming non-aliasing.

## Evidence

This contract is based solely on the supplied static function representation.

Established evidence includes the explicit word and narrow access widths, repeated write pattern, literal values, offsets, capture-before-write relationships, dependent address reads, and final four outgoing values.

The 32-bit machine-state interpretation is an inference from the represented scalar widths and unsigned arithmetic. No original instruction stream, target ABI, memory map, dependency implementation, or runtime observation was supplied.

No runtime tests, access traces, fault experiments, or indirect-destination execution have been performed.
