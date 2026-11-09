<!-- Trusted provenance: F-c51ae8bb81bd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c51ae8bb81bd

## Purpose

F-c51ae8bb81bd conditionally copies selected fields from a mutable object into an indexed record area within that object. On the transfer path, it clears a gate byte, increments a 32-bit index, and calls a post-transfer dependency.

The visible normal-return results are:

- **0:** completion of either the no-transfer path or the transfer path.
- **3:** the gate was nonzero and the checked index equaled exactly 4.

“Gate,” “index,” and “record” describe the observed access pattern. A pending-state flag, four-record collection, and synchronization protocol are plausible interpretations, not established application contracts.

The transfer comprises **18 field transfers through one retained destination**, followed by a gate-clear write and **five byte transfers whose destinations each use a fresh index read**. It is neither a whole-record initialization nor a coherent snapshot of the source fields.

## Interface

### Input and storage requirements

The sole explicit input is a 32-bit address-like value identifying the object base. All object offsets below are byte offsets.

No explicit length, capacity, allocator, or error-output argument is present. The visible body performs no null, alignment, accessible-memory, or general index-range check.

Defined completion requires that every reached memory access be valid, that the address calculations be valid under the applicable execution model, and that dependencies permit subsequent operations to proceed. Dependency-specific storage and lifetime requirements are unknown.

### Result

The result is a 32-bit value containing 0 or 3 on the visible normal-return paths. Dependency results are not visibly used to choose this value.

A result of 0 does not establish that a record was written. A result of 3 identifies the exact index-equals-4 branch; it is not a general out-of-range status.

A fault or a dependency that does not return normally may prevent any result. No visible mechanism translates such events into a status.

### Relevant object layout

| Object-relative location | Width or extent | Observed use |
|---|---:|---|
| `0x40` | 1 byte | Gate read after the entry dependency; cleared on the transfer path |
| `0x78` | Unknown extent | Address passed to the entry and exit dependencies |
| `0x98` | 4 bytes | Index comparison, destination calculations, and increment |
| `0x9C + 0x48 × index` | Record origin | Origin used to describe the selected record |
| Source offsets listed below | 1 or 4 bytes | Individually read transfer inputs |

The record stride is `0x48`, or 72 bytes. With a stable index, direct record writes occupy selected locations from record-relative offset `0x00` through `0x45`; they do not cover every byte in that range or the entire stride.

The highest explicitly accessed source field begins at object offset `0x1C4` and has width four bytes, extending through `0x1C7`. This establishes a reached access requirement on the transfer path, not the complete object size.

## Behavior

### Entry and branch selection

The first visible operation calls **F-338d16f9ce72** with the object address plus `0x78`.

After that call returns, the function reads the gate byte at `0x40`. Any nonzero byte selects the index check; zero selects the no-transfer path. The branch therefore depends on the gate after the entry dependency, not necessarily its value at invocation.

For a zero gate:

- The body does not read the index.
- No field transfer, gate clear, or index increment occurs.
- The post-transfer dependency is skipped.
- **F-1333331b2da1** is called with the object address plus `0x78`.
- The function returns 0 if that call returns normally.

For a nonzero gate, the body reads the 32-bit index at `0x98` and compares it with exactly 4.

If that read yields 4, the function calls **F-1333331b2da1** with the address at `0x78`, then returns 3. This branch performs no direct record write, gate clear, index increment, or post-transfer call.

The comparison is equality, not an upper-bound check. Values greater than 4 and negative values do not satisfy it.

### Retained-destination transfers

When the checked index is not 4, the body reads the index again to calculate a destination. The comparison read and this calculation read are distinct. They need not yield the same value if intervening external mutation is possible.

For describing the writes, the selected record origin is:

**object base + `0x9C` + 72 × the index read for this calculation**

This destination is retained for **exactly 18 transfers**:

| Order | Source offset from object base | Width | Destination offset from selected record origin |
|---:|---:|---:|---:|
| 1 | `0x0A` | 1 byte | `0x00` |
| 2 | `0x0B` | 1 byte | `0x01` |
| 3 | `0x0C` | 4 bytes | `0x04` |
| 4 | `0x10` | 4 bytes | `0x08` |
| 5 | `0x18` | 4 bytes | `0x24` |
| 6 | `0x14` | 4 bytes | `0x28` |
| 7 | `0x1C` | 4 bytes | `0x0C` |
| 8 | `0x20` | 4 bytes | `0x10` |
| 9 | `0x2C` | 4 bytes | `0x18` |
| 10 | `0x30` | 4 bytes | `0x14` |
| 11 | `0x24` | 4 bytes | `0x1C` |
| 12 | `0x28` | 4 bytes | `0x20` |
| 13 | `0x34` | 1 byte | `0x2C` |
| 14 | `0x46` | 1 byte | `0x2D` |
| 15 | `0x3C` | 4 bytes | `0x30` |
| 16 | `0x1BE` | 1 byte | `0x34` |
| 17 | `0x1C0` | 4 bytes | `0x38` |
| 18 | `0x1C4` | 4 bytes | `0x3C` |

Each row describes an individual source read and corresponding destination write, in the listed order. The supplied representation establishes no semantic conversion of the four-byte values. Their application meaning—integer, floating-point, address, or otherwise—is unknown.

The destination is not recalculated between these 18 transfers. A change to index storage during this group does not, by itself, redirect the remaining writes in the group.

Source fields are read as their transfers are reached, rather than captured together beforehand. If destination storage overlaps source storage, an earlier write can affect a later read. A whole-object snapshot or overlap-safe bulk-copy contract would therefore be incorrect.

### Gate clear and fresh-index transfers

After the eighteenth retained-destination transfer, the body writes zero to the gate byte at `0x40`.

It then performs five additional one-byte transfers in this order:

| Order | Source offset | Destination offset from object base | Record-relative offset if the index is unchanged |
|---:|---:|---|---:|
| 1 | `0x47` | `0xDE + 72 × freshly read index` | `0x42` |
| 2 | `0x4A` | `0xE1 + 72 × freshly read index` | `0x45` |
| 3 | `0x41` | `0xDD + 72 × freshly read index` | `0x41` |
| 4 | `0x48` | `0xDF + 72 × freshly read index` | `0x43` |
| 5 | `0x49` | `0xE0 + 72 × freshly read index` | `0x44` |

Every row reads index storage at `0x98` again for its own destination calculation. In particular, the transfer from `0x47` is **not** part of the retained-destination group and occurs **after** the gate clear.

There is no dependency call between these transfers. Nevertheless, without a valid synchronization or aliasing contract, their index reads cannot be assumed to return identical values.

The representation does not establish the finer ordering between the source-byte read and destination-index read within one transfer. Both supply that transfer before its destination write.

### Index advancement and completion

After all five later transfers, the body reads the current 32-bit index, adds one, and writes the result back to `0x98`.

This increments the value read at that point. It is not necessarily the value used for the equality check or the retained destination.

The body next calls **F-ae23f27a31f9** with the original object base. Before this dependency is entered, the body has performed all 23 field transfers, the gate clear, and the index update.

After that call returns, the body calls **F-1333331b2da1** with the object address plus `0x78`, then returns 0.

These facts describe completed direct operations before the calls. They do not guarantee the final gate, index, or record contents: either subsequent dependency may change storage or affect object lifetime.

### Widths and arithmetic boundaries

The supplied representation uses a 32-bit address-like argument and 32-bit integer index operations. Destination calculations multiply the index by 72, then combine it with the object base and a fixed offset.

For stable ordinary indices:

| Index | Selected record origin |
|---:|---:|
| 0 | `0x9C` |
| 1 | `0xE4` |
| 2 | `0x12C` |
| 3 | `0x174` |

For index 3, the highest explicitly written record byte is at object offset `0x1B9`.

There is no visible protection against negative indices, multiplication overflow, address overflow, or index-increment overflow. The representation does not establish guaranteed modular wrapping or a checked-arithmetic failure mode. Exact behavior outside valid arithmetic and storage conditions remains **documented_only**.

## Side effects

The directly visible side effects are:

- An entry dependency call on every invocation.
- On the transfer path, 18 retained-destination field writes followed by the gate clear and five fresh-index byte writes.
- A 32-bit index update after those transfers.
- A post-transfer dependency call only on the transfer path.
- An exit dependency call on every visible normal-return path.

With a stable index and no aliasing into the record, the direct body leaves record-relative bytes `0x02–0x03`, `0x2E–0x2F`, `0x35–0x37`, `0x40`, and `0x46–0x47` unwritten. Dependencies may modify these or other bytes.

No allocation, deallocation, ownership transfer, or resource release is explicit in the body. This absence does not establish that dependencies refrain from such operations.

The entry and exit calls resemble a synchronization pair, but their pairing does not prove exclusion, barriers, reentrancy safety, or lifetime protection. No atomic record-publication guarantee is established.

No visible rollback restores earlier writes if a later access or dependency fails. The exit dependency is reached on the shown normal paths; its execution after an abnormal interruption is not established.

## Dependencies

| Opaque ID | Argument | Invocation condition and order |
|---|---|---|
| **F-338d16f9ce72** | Object address plus `0x78` | First visible operation, before the gate read |
| **F-ae23f27a31f9** | Original object base | Transfer path only, after all field transfers, gate clearing, and index advancement |
| **F-1333331b2da1** | Object address plus `0x78` | Before every visible normal return; after the post-transfer call when that call occurs |

Dependency return values are not visibly consumed.

No dependency bodies are supplied. Their memory effects, global-state effects, synchronization properties, ownership behavior, and abnormal-control-flow behavior are unknown. In particular, this contract does not assume that a dependency preserves globals, retains the object, or refrains from releasing memory.

Normal completion of later accesses is conditional on those accesses remaining valid after earlier dependency calls.

## Acceptance examples

These examples are conditional consequences of static evidence, not executed tests. Unless an example says otherwise, dependencies return normally, required storage and arithmetic are valid, and no dependency or external writer changes the discussed state.

### Zero gate, arbitrary index contents

If the gate is zero after the entry dependency returns:

- The exit dependency is called next.
- The body does not inspect the index or write a record.
- The post-transfer dependency is skipped.
- The result is 0.

An index containing 4 does not change this branch.

### Nonzero gate and index exactly 4

If the gate is nonzero and the checked index is 4:

- No direct record transfer occurs.
- The body leaves the gate and index unchanged.
- The post-transfer dependency is skipped.
- The exit dependency is called.
- The result is 3.

The unchanged-state statements concern direct body operations, not unknown dependency effects.

### Stable index 2

If the gate is nonzero and the index remains 2 through the transfers:

- The retained record origin is `0x12C`.
- The four-byte value at `0x18` is transferred to `0x150`.
- The four-byte value at `0x14` is transferred to `0x154`, after the preceding transfer.
- The eighteenth retained-destination transfer writes four bytes from `0x1C4` to `0x168`.
- The gate is then cleared.
- The byte from `0x47` is subsequently written to `0x16E`, using a fresh index read.
- After all five later transfers, the index is updated to 3.
- The post-transfer dependency precedes the exit dependency.
- The result is 0.

This scenario distinguishes the 18-transfer retained group from the five later transfers.

### Reaching the equality boundary

A transfer starting with stable index 3 writes the fourth nominal record and advances the index to 4.

If the gate is externally made nonzero before the next invocation, and entry leaves those values unchanged, that invocation returns 3 without a transfer. If the gate instead remains zero, it returns 0 without reading the index.

### Index 5

A nonzero gate and checked index 5 do not select the return-3 branch. With a stable index, the retained record origin would be `0x204`, and the body attempts the transfer path.

This is evidence of an absent general upper-bound check, not evidence that the object provides storage for that record. Completion remains conditional on valid storage, arithmetic, and dependency behavior.

### Entry changes the gate

If the gate is initially nonzero but the entry dependency changes it to zero, the no-transfer path is selected. Conversely, entry changing zero to nonzero enables the index check.

The pre-entry gate is not independently consulted.

## Uncertainties

The following remain **documented_only**:

- The actual object type, full allocation extent, and field meanings.
- The allowed index domain and whether callers guarantee a range of 0 through 4.
- Whether the entry and exit dependencies implement synchronization.
- Whether concurrent or reentrant mutation is permitted.
- All dependency effects on memory, globals, ownership, resources, and object lifetime.
- Exact overflow, alignment, inaccessible-memory, and abnormal-return behavior.
- Endianness and semantic interpretation of four-byte fields.
- The post-transfer dependency’s effect on the newly written state.
- Machine-level ordering or atomicity beyond what the supplied representation establishes, including read ordering within individual later transfers.

An intended four-record capacity is a reasonable interpretation of the stride and equality check, not an established safe bound or a substitute for missing caller and dependency contracts.

## Evidence

This contract is based solely on static inspection of the supplied function representation.

Established evidence includes the dependency-call order, byte and four-byte access widths, exact comparison with 4, separate index reads, 72-byte stride, 18 retained-destination transfers, intervening gate clear, five fresh-index transfers, subsequent index increment, and normal-return values 0 and 3.

Application meanings and synchronization intent are identified as interpretations rather than facts. No dependency bodies, surrounding type definitions, caller contracts, or runtime observations were supplied.

No runtime tests have been performed.
