<!-- Trusted provenance: F-bf9c97ca59d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bf9c97ca59d1

## Purpose

F-bf9c97ca59d1 performs an ordered sequence of memory reads, mixed-width writes, address calculations, chained-record mutations, and a terminal indirect transfer represented as a call.

No application-level purpose is established. Repeated offsets suggest linked records, but do not establish a list, tree, allocator, serializer, or other domain operation.

This is a conditional, source-facing contract. The supplied representation exposes implicit entry state and uncertain control-flow recovery; it does not establish a safely callable standalone interface.

## Interface

### Explicit inputs

The represented interface contains four 32-bit inputs:

| Input | Established use |
|---|---|
| First word | No visible use. |
| Second word | No visible use. |
| Third word | Reduced by `0x70`; the result becomes an address-like operand and is stored in a reached record. |
| Fourth word | Written in full by the first visible store; subsequently used in address arithmetic and truncated to a byte. |

The third and fourth inputs are represented as signed 32-bit quantities. Their use as addresses does not establish valid ranges or make signed overflow portable.

Unused inputs remain part of the represented interface; their omission from an actual calling convention is not justified.

### Implicit inputs and preconditions

Execution also requires:

- An entry-record address and a separate address for the first store.
- An additional 32-bit entry value.
- An unsigned 32-bit arithmetic operand and a byte-sized arithmetic operand.
- An unresolved executable target, used both numerically and for the terminal transfer.
- External storage and every memory location reached through computed addresses.

These values are not initialized within the represented body.

Successful completion requires all reached reads, writes, arithmetic operations, and the terminal transfer to be supported by the execution environment. There are no visible checks establishing those conditions. In particular, byte-derived addresses can designate low memory, and some destinations incorporate an executable-associated address.

### Outputs, widths, and layout

There is no explicit return value. Observable outputs are memory operations and the terminal invocation with three arguments.

Access widths are 8, 16, and 32 bits. Address-like values occupy 32 bits in the representation. All offsets below are byte offsets, including `0x64`, which is decimal 100.

No complete structure layout, allocation size, byte order, alignment rule, or address-validity model is established. Fields overlap: a byte at `0x11` can modify storage belonging to a word at `0x10`.

Low-byte and low-halfword conversions retain the least significant 8 or 16 bits. This specifies the value supplied to a store, not its byte-order-dependent memory image.

## Behavior

### Ordering and captured values

A captured value means the result of a particular completed read. Subsequent uses of that capture do not imply another read of its source.

Conversely, repeated source reads remain distinct, including reads of the same external address. External storage is not established to be immutable, independent of reached records, or unaffected by earlier writes.

The sequence below describes established ordering between separate operations. Relative evaluation order of multiple reads within one expression is not established merely by their textual placement.

### Entry and initial linkage

The first visible operation writes the complete fourth input at the separate entry address plus `0x54`. This precedes any visible read of the entry record.

The entry record then supplies, in order:

1. An unsigned byte at `0x05`, retained for address calculations.
2. A word at `0x50`, copied to `0x20`.
3. A linked-record address at `0x74`.
4. A nested-record address read from the linked record at `0x64`.
5. A fresh byte read at the entry record’s `0x05`, copied to its `0x11`.
6. The entry record’s word at offset zero.

The two reads of the entry byte are distinct. The copied byte is not necessarily the earlier captured byte if aliasing or external activity changes the location.

The operation stores low bytes of record addresses into the linked record’s `0x11` and the entry record’s `0x09`. It saves the nested-record address at the entry record’s `0x54`.

A fresh unsigned byte from the linked record’s `0x05` becomes a numeric address base. The linked record receives the captured entry word at `0x54` and the adjusted third input at `0x34`. The nested record receives low bytes at `0x0c`, `0x15`, and `0x09`.

The byte-derived base receives:

- The nested-record address as a word at `0x64`.
- The additional implicit entry value as a word at `0x54`.
- The low byte of the nested-record address at `0x11`.

No pointer-table lookup is shown between reading that byte and using it as an address.

A word captured from the nested record’s `0x60` supplies a byte written at the sum of the nested-record address and captured entry word. Its low halfword is stored at the retained entry byte plus `0x45`. Between these operations, a byte containing the low entry word is stored at the fourth input plus the retained entry byte.

A destination word is loaded from the adjusted third input plus the nested-record address. The reached destination receives word `0x45` at `0x45`. The earlier halfword destination then receives the fourth input’s low byte.

### External-read identities and early indexed writes

The following external addresses identify storage locations, not immutable constants or callable dependencies. All listed reads are 32-bit reads; subsequent truncation does not reduce their read width.

| External location | Read occurrences and significant uses |
|---|---|
| `0x087fc56c` | Read once into a retained value, then read again to address a halfword store of `0x53` relative to the adjusted third input. The first capture, not the second read, supplies a later halfword sum with the unresolved target’s low halfword. |
| `0x087fc57c` | Read first as a retained address-like value, again to capture a low byte, and again for the destination of a doubled-address byte store. These three reads precede that store. |
| `0x087fc598` | Captured between two consecutive byte stores at twice the retained first `0x087fc57c` value. |
| `0x087fc5b0` | Read to form a doubled-address word destination. |
| `0x087fc664` | Read into a retained address, then read separately for another destination relative to the first comparison-adjusted doubled value. |
| `0x087fc5c8` | Captured before the first byte store at twice the retained `0x087fc664` value; its low byte is used in a later store there. |
| `0x087fc5cc` | Captured between the first two byte stores at that doubled address. |
| `0x087fc5e4` | Read into a retained address-like value, then read again to capture a low halfword. |
| `0x087fc6a0` | Read into a retained address. Later, two further reads supply the destination and low-byte value of one doubled-address byte store; their relative evaluation order is unresolved. |
| `0x087fc670` | Its low byte is captured after a computed word read and before that doubled-address byte store. |
| `0x087fc6dc` | Captured after the first arithmetic-right-shift region. |
| `0x087fc6cc` | Read to address a byte store of `0x59` at offset `0x59`. |

After the second read of `0x087fc56c`, the earlier loaded destination receives word `0x53` at offset zero. A halfword containing the first `0x087fc56c` capture plus the target’s low halfword is written relative to the nested-record address and the implicit unsigned operand.

The three reads of `0x087fc57c` have different roles. The third read determines the first doubled-address destination, while the second supplies its byte value. Later stores use twice the first capture. Thus these destinations need not coincide.

Other operations using the first capture include:

- A byte store relative to the adjusted third input.
- A halfword store at the sum of that capture and a sign-extended byte read from twice the earlier loaded destination.
- A word read at offset `0x52`.
- Successive byte stores at its doubled address: first the low byte captured from the nested record’s `0x60`, then the second `0x087fc57c` capture’s low byte.
- A later halfword store at the same doubled address containing the target’s low halfword.

The `0x087fc598` capture occurs between those two byte stores and remains available for later halfword and byte stores.

The word read at the retained address plus `0x52` supplies stores at its own `0x52` and at twice a fresh `0x087fc5b0` read. A halfword constant `0x52` is also written at a comparison-adjusted address.

The retained `0x087fc664` address receives multiple doubled-address byte writes: the target’s low byte, its own low byte, and later the low byte captured from `0x087fc5c8`. The intervening `0x087fc5cc` capture supplies a byte destination at `+0x2c`, a signed-byte source at its doubled address, and a comparison against `0x4e`.

### Arithmetic and the first right shift

One intermediate address doubles the adjusted third input and adds the unsigned comparison bit for the unresolved target being no greater than that adjusted input. Another doubles this intermediate and adds whether the captured `0x087fc5cc` word exceeds `0x4e`.

The retained `0x087fc5e4` value participates in a computed word address. That word is captured before a halfword overwrite at the same address. Its bits are interpreted as signed 32-bit data and arithmetically shifted right by its own low-byte count.

The shifted result is:

- Added to the original captured word to form a destination.
- Written there as a 32-bit value.
- Used in further byte and halfword destinations.

Counts are between zero and 255. Behavior for counts at least 32 is **documented_only**; no masking, saturation, or portable result is established.

The `0x087fc6dc` capture supplies a later word destination and halfword destination. The associated address of F-32c0bc2d3e1d then participates numerically in byte destinations. No invocation of that dependency is shown.

### Carry-sensitive construction and transition to records

Further external reads occur in this order:

| External location | Capture and later use |
|---|---|
| `0x087fc710` | First captured for doubling and carry calculations; read again for the destination of a halfword store at `+0x54`. That store uses the low halfword from the second `0x087fc5e4` read. |
| `0x087fc6f8` | Captured for a later halfword value. |
| `0x087fc6e8` | Captured for a subsequent halfword sum. |
| `0x087fc6e4` | Read for an address increased by a carry bit; read again for the unsigned threshold comparison; conditionally read a third time for the addition-carry test. |
| `0x087fc6f8` | Read again for a comparison with the implicit unsigned operand. This comparison does not use the earlier capture. |
| `0x087fc73c` | Captured after a byte store of `0x45` relative to the first `0x087fc6e4` result. |
| `0x087fc688` | Its low halfword is captured immediately afterward. |

The carry bit reflects unsigned 32-bit doubling of the first `0x087fc710` capture followed by addition of a comparison bit. A separate predicate tests whether a fresh `0x087fc6e4` value exceeds `0xffffffa6`; only if that test is false does the represented short-circuit expression perform the further read used to test addition of `0x59` and the carry bit.

Consequently, these calculations cannot generally be reduced to arithmetic on one immutable `0x087fc6e4` value.

The resulting addresses receive halfwords, including two destinations separated by the implicit unsigned operand. A computed word is read to form a byte destination at `+0x45`; after an intervening halfword store to its source location, that word is read again and used for a word store of `0x45` at `+0x45`.

The `0x087fc73c` capture supplies a destination at `+0x45` and a record link at `+0x74`. Its halfword destination receives a sum involving the `0x087fc6e8` capture, the low halfword from `0x087fc688`, and whether the unresolved target is below numeric address `0x46`. A subsequent byte write there uses the earlier `0x087fc598` low byte, `0x53`, and the fresh `0x087fc6f8` comparison bit.

Explicit unsigned operations and carry tests have 32-bit significance. Signed-overflow behavior in other additions is not established.

### Central capture, second right shift, and record mutation

For this region, “anchor” denotes the record captured through the `0x087fc73c` value’s `0x74`. “Companion” denotes the record read from the comparison-adjusted base’s `0x14`. These are descriptive roles, not disjoint-object guarantees.

The anchor initially receives an address-derived word at `0x54` and the companion address at `0x44`. Another reached record receives its own address at `0x54`.

A chain through the anchor’s `0x64` and then `0x14` supplies an unsigned 32-bit value. Its `+0x09` receives two consecutive byte stores containing low bytes of two different reached addresses.

**After those two stores, and before traversing the companion’s `0x14`, a 32-bit read occurs at external address `0x000000db`.** Its captured value is retained across the following mutations. It is not reloaded for each use.

The companion’s `0x14` supplies an address whose `+0x20` is retained as an address-derived value. Further linkage writes occur, followed by two separate word reads of the anchor’s `0x44`: the first captures a word for later copying; the second supplies a destination-record address.

A second, distinct right shift then occurs:

- Its source is a signed byte read at twice the previously captured unsigned chain value.
- That byte is promoted to signed 32-bit data.
- The shift count is the unsigned chain value’s low byte.
- The arithmetic-right-shift result is truncated to 8 bits.

This captured byte is stored first at the second `0x44` read’s destination plus `0x08`. A word store then copies the first `0x44` capture into the anchor’s `0x74`. Only afterward is the captured shifted byte stored at the anchor’s `0x11`.

Counts at least 32 are **documented_only** here as well. This operation is not the earlier self-counted word shift.

The companion’s `0x54` next receives the retained address-derived value, immediately followed by the complete `0x000000db` capture. Later, the anchor’s `0x11` receives that capture’s low byte, replacing the shifted byte, and the anchor’s `0x54` receives the complete capture, replacing its earlier address-derived value.

### Remaining chained mutations

Subsequent work traverses and rewrites words at offsets including `0x04`, `0x14`, `0x44`, `0x64`, and `0x74`. Each read uses memory as it exists at that point.

A byte-addressed region reached through the anchor’s `0x14` receives:

| Offset | Ordered stores |
|---|---|
| `0x10` | Words `0x73`, `0x6e`, then a separately captured word from its `0x50`. |
| `0x14` | Byte `0x73`. |
| `0x20` | Word `0x6e`. |
| `0x1c` | Bytes `0x79`, then `0x68`. |
| `0x54` | A captured link value, later the region’s own address. |

Additional writes copy low address bytes, replace link-like fields, and assign containing addresses to `0x54` or `0x44`. Reads of the anchor’s `0x64`, `0x74`, and other fields recur with intervening writes; they are not interchangeable snapshots.

Later chains receive decimal 100, `0x74`, `0x66`, and `0x73` at mixed widths. Notable distinctions include:

- Byte `0x74` followed by word `0x74` at a reached `0x10`.
- Word `0x66` followed by byte `0x66` at another reached `0x10`.
- Consecutive identical byte stores at a reached `0x0d`.
- Explicit word read/write-back operations at `0x54` and `0x64`.
- A word store `0x20` below a captured address, containing that address minus `0x74`.

These remain observable accesses even where their final values appear redundant. No allocation boundary or string interpretation is established.

### Terminal preparation and invocation

External address `0x087fc9c0` is read once and retained. A subsequent record `0x74` read supplies a captured address and its low halfword.

Halfword stores use that low halfword at a sum of record addresses, at the retained external value plus the captured address, and again at the first sum. Another halfword store, between these operations, writes the external capture’s low halfword at twice a reached record address.

A signed byte is read from another sum of record addresses and sign-extended to 32 bits, then converted to unsigned 32-bit form. This converted value becomes an address base receiving word `0x45` at `+0x45`.

The final captures then occur in order:

1. A 32-bit read at `0x087fc9dc`.
2. A 32-bit read at `0x087fc9c8`.
3. A word read at the current record’s `0x44`.

The converted-byte base receives byte `0x44` at `+0x44`. Twice the `0x087fc9dc` capture receives the complete `0x087fc9c8` capture. A final halfword destination combines a retained record address with the `0x087fc9dc` capture and receives that capture’s low halfword.

The unresolved terminal dependency receives:

| Argument | Value |
|---|---|
| First | Converted terminal signed byte, bitwise ORed with `0x44`. |
| Second | Captured value from `0x087fc9dc`. |
| Third | Captured terminal read of the current record’s `0x44`. |

The third argument is not reloaded after the final stores.

## Side effects

The body reads and writes potentially aliased record storage, external storage, low addresses, and executable-associated addresses.

No allocation, deallocation, ownership transfer, locking, validation, or rollback is visible. None is guaranteed absent from the unresolved terminal dependency.

Unsupported access, invalid alignment, arithmetic behavior outside the established model, or an invalid transfer can prevent completion. No local error result or recovery path is shown. Earlier completed stores are not visibly undone.

Intermediate writes are contract-relevant, including overwritten and identical writes. Final memory contents alone are insufficient to establish equivalence.

## Dependencies

### F-32c0bc2d3e1d

Only its associated address is visibly used. Numerical address use does not establish invocation, readable contents, writable storage, or ownership rights.

### F-indirect-01

This document-local opaque ID denotes the unresolved terminal target supplied by implicit entry state. It also participates numerically in earlier comparisons and truncations.

Its actual identity, calling convention, side effects, ownership behavior, global preservation, return behavior, and whether the transfer is truly a conventional call are **documented_only**.

### External storage

The addresses identified in the read tables are storage dependencies, including locations whose supplied labels suggest executable content. Such labels do not establish calls.

The representation warns of overlapping external storage. Neither independence nor immutability follows from distinct source labels or address roles.

## Acceptance examples

These are conditional static criteria, not executed tests.

- **Entry ordering:** With fourth input `0x12345678`, the first store writes that complete word at the separate entry base plus `0x54`, before the first entry-record read.

- **Fresh versus retained external reads:** If the three reads at `0x087fc57c` yield different supported values, the first is retained for later address calculations, the second supplies the captured byte, and the third determines the first doubled-address byte destination. Replacing all three with one capture changes the contract.

- **Captured external replacement:** If the read at `0x000000db` yields `0x12345678`, the companion’s `0x54` receives that complete value after its address-derived store. The anchor later receives byte `0x78` at `0x11` and word `0x12345678` at `0x54`, without another read at `0x000000db`.

- **Second shift:** If the retained unsigned chain value has low byte one and the signed byte at its doubled address is negative two, the shifted byte is `0xff`. That same captured byte reaches the destination record’s `0x08` and, after the intervening word store, the anchor’s `0x11`. This example does not resolve counts of 32 or more.

- **Partial overwrite:** A captured word `0x1234abcd` supplies halfword `0xabcd` at the retained entry byte plus `0x45`. A later fourth-input low byte `0x78` overwrites one byte at that address. Final halfword interpretation depends on byte order and aliases.

- **Terminal capture:** If the terminal record’s `0x44` changes because of a final store, the third argument still uses the earlier captured word.

- **Terminal sign conversion:** A terminal signed byte of negative one produces first argument `0xffffffff`; zero produces `0x44`. Neither example establishes that the associated address accesses succeed.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, complete function boundary, and actual calling convention.
- Origins and validity of implicit entry state.
- Executable accuracy of the recovered straight-line control flow.
- Byte order, alignment, memory permissions, and address wrapping.
- Signed overflow and large-count shifts, including both distinct shift operations.
- Relative evaluation order of reads within a single expression.
- Aliasing, concurrency, volatile semantics, and device-memory behavior.
- Terminal dependency effects, ownership obligations, and return behavior.

Record manipulation is inferred intent only. It does not justify stronger object invariants.

## Evidence

The supplied static representation is the only evidence. No runtime tests have been performed.

Established evidence includes access widths, constants, offsets, separate read occurrences, capture dependencies, ordering between operations, both right shifts, explicit carry predicates, numerical dependency-address use, and terminal argument construction.

Embedded comments and prior prose are not independent behavioral authority. Unknown platform and dependency behavior remains documented rather than replaced with assumed guarantees.
