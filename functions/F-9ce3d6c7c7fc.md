<!-- Trusted provenance: F-9ce3d6c7c7fc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ce3d6c7c7fc

## Purpose

F-9ce3d6c7c7fc performs an ordered series of byte, halfword, and word accesses, propagates values between fixed and calculated addresses, and ends with an unresolved indirect transfer represented as a four-argument invocation.

**Established static evidence:** the supplied representation contains repeated stores, absolute-address accesses, reads through two implicit address bases and one explicit input, subsequent accesses through loaded address-like words, and a final indirect invocation. No explicit validation, allocation, release, or error-reporting branch is represented.

**Inferred intent:** no higher-level purpose is established. The access pattern alone does not justify describing the operation as initialization, serialization, device programming, or management of a particular data structure.

This contract describes the represented memory behavior. It does not establish that the recovered interface, operation boundaries, or final control transfer exactly match the original executable.

## Interface

### Explicit inputs

| Input | Width | Represented use |
|---|---:|---|
| Input address | 32 bits, represented as signed | Stored as a word during the opening phase; later used as the base of a word read at offset `0x74`. |
| Input word | 32 bits | Stored during the first occurrence of the opening pattern. Its signedness does not affect that store. |

The input address’s use in address calculations does not establish an object type, allocation size, lifetime, or ownership.

### Implicit inputs

Three incoming machine-state values are consumed without appearing as explicit parameters:

| Contract role | Width and interpretation | Uses |
|---|---|---|
| Primary base | Unsigned 32-bit value | Read and write address base, doubled value, low-byte source, and shift-count source. |
| Secondary base | Signed 32-bit value | **Write address base and low-byte source only.** |
| Tertiary base | Signed 32-bit value | Read and write address base and stored word value. |

These are descriptive roles, not source identifiers or declarations of separate objects. The explicit inputs alone are insufficient to reproduce the represented operation.

**No memory read is addressed through the secondary base as such.** Secondary-base-relative destinations require support for the specified writes, not an independent ability to read those destinations. An address may nevertheless require readability if it aliases a separately specified read.

### Outputs and completion

No explicit result value is represented. Observable outputs are the memory accesses, resulting state, and effects of the final indirect invocation.

A return is shown after that invocation. Normal completion is therefore conditional on the invocation returning normally and on all preceding operations completing. No dependency result is consumed.

The interpretation of the final transfer as a returning call remains `documented_only`; its representation does not establish the actual calling convention or control-flow recovery.

### Widths and address arithmetic

All offsets are byte offsets. Access widths are:

- Byte: 8 bits.
- Halfword: 16 bits.
- Word: 32 bits.

“Low byte” denotes the least significant eight bits, without a character-encoding interpretation.

Arithmetic involving the unsigned primary base uses 32-bit unsigned results. In particular, doubling discards bits above bit 31. Adding an offset to that unsigned value likewise does not establish an unbounded mathematical address.

Address expressions involving signed inputs or signed loaded words have unresolved overflow behavior. The unsigned wraparound rule must not be extended to them.

Byte order, alignment requirements, integer-to-address interpretation, and address-space accessibility are not established.

## Behavior

### Access requirements and aliasing

For normal completion, the environment must support each access when it is reached:

- Word reading at input address + `0x74`.
- The specified word reads through the primary and tertiary bases.
- Byte and word writes through the primary, secondary, and tertiary bases.
- Opening word writes through twice the primary base.
- The absolute-address accesses described below.
- Reads and writes through subsequently loaded address-like values.
- The final indirect transfer and any requirements imposed by its unresolved target.

There is no separate requirement to read secondary-base-relative destinations merely because they receive stores. Permissions must be determined from actual accesses and numerical overlap, not from an assumption that every destination is a readable object.

No represented check validates any address or arithmetic result. No documented status code replaces a failed access.

Aliasing is not prohibited. A write can change the value returned by a later read when their byte ranges overlap. Conversely, a value already captured by a read is not implicitly refreshed when its original storage changes.

The ordering below describes the supplied representation. Hardware ordering, atomicity, compiler transformations, and observability of ordinary versus device memory remain unresolved.

### Opening repeated stores

The opening phase repeats an ordered pattern over fixed calculated locations:

| Order within an occurrence | Destination | Width | Stored value |
|---:|---|---:|---|
| 1 | Secondary base + `0x19` | 8 bits | Primary base’s low byte |
| 2 | Primary base + `0x11` | 8 bits | Secondary base’s low byte |
| 3 | Secondary base + `0x34` | 32 bits | Input word on the first occurrence; twice the primary base thereafter |
| 4 | Twice the primary base + `0x54` | 32 bits | Input address |

The last occurrence stops after the third store. It does not perform a fourth store before the next phase.

The repetition uses fixed addresses, not advancing elements. No data-dependent termination condition is represented. The exact numerical occurrence count is retained as `documented_only`; this contract does not supply an independently established count.

For ordinary memory without relevant overlap or external mutation, later opening stores replace the input word at secondary base + `0x34` with twice the primary base. This final-value equivalence does not authorize deleting the first store or collapsing repeated accesses where access events matter.

### Initial captures and publication

After the opening phase, the operation captures a word from absolute address `0xA9`, then a word from input address + `0x74`.

It next writes one byte at absolute address `0x3A`. That byte is the low eight bits of twice the primary base’s low byte plus `0x30`.

Two more word captures follow, in order:

- Tertiary base + `0x14`.
- Primary base + `0x04`.

These captures precede the following publication writes:

| Order | Destination | Width | Value |
|---:|---|---:|---|
| 1 | Tertiary base + `0x0D` | 8 bits | Primary base’s low byte |
| 2 | Primary base + `0x08` | 8 bits | `0x6C` |
| 3 | Primary base + `0x0D` | 8 bits | Secondary base’s low byte |
| 4 | Primary base + `0x09` | 8 bits | Secondary base’s low byte |
| 5 | Tertiary base + `0x54` | 32 bits | Tertiary base |
| 6 | Absolute address `0x67` | 8 bits | Low byte of the word captured from `0xA9` |

A signed word is then read from tertiary base + `0x44`. This contract calls it the **shift source**.

After that read, the byte at absolute address `0x67` is read and written to absolute address `0x21`. This is a new access to `0x67`, not merely reuse of the captured word’s low byte.

### Shift-dependent update and replacement

The shift source supplies two successive publications: its low byte is written at primary base + `0x10`, and its full word is written at secondary base + `0x54`.

Next, one byte is written at tertiary base + `0x01`. Its value is obtained by right-shifting the signed shift source by the primary base’s low eight bits and retaining the resulting low byte.

The representation establishes the operand and count expression, but does not resolve all shift semantics:

- Counts from 0 through 31 depend on the target’s signed-right-shift convention where the source is negative.
- Counts from 32 through 255 require further target evidence.
- Hardware count masking, trapping, saturation, or another oversized-count rule must not be assumed.

The word at secondary base + `0x54` is then replaced with the primary base.

A word is captured from primary base + `0x40`. Following that capture, these updates occur in order:

1. The earlier capture from primary base + `0x04` is written as a word at secondary base + `0x74`.
2. The capture from primary base + `0x40` is written as a word at primary base + `0x10`.
3. The tertiary base is written as a word at secondary base + `0x44`.
4. The byte at absolute address `0x67` is read again and written at primary base + `0x09`.
5. The primary base’s low byte is written at tertiary base + `0x09`.
6. The word captured from absolute address `0xA9` is written at primary base + `0x44`.

The word replacement beginning at primary base + `0x10` covers offsets `0x10` through `0x13`. It therefore overlaps both the earlier shift-source byte at `0x10` and the opening byte stores at `0x11`. Their earlier access events remain part of the represented behavior.

### Updates through captured addresses

A signed word is read from primary base + `0x54`; call its value the **first derived address**.

The byte at absolute address `0x67` is then read again and written at primary base + `0x09`. Next, the low byte of the earlier input-address-plus-`0x74` capture is written at first derived address + `0x0D`.

The word at tertiary base + `0x54` is replaced with the earlier capture from tertiary base + `0x14`.

Only after those writes is a word captured from primary base + `0x14`. Earlier destinations may affect this capture through aliasing.

Three publications follow:

- At tertiary base + `0x11`, the low byte of the input-address-plus-`0x74` capture.
- At primary base + `0x09`, the first derived address’s low byte.
- At primary base + `0x44`, the full first derived address.

A signed word is then read from tertiary base + `0x04`; call it the **second derived address**.

The operation writes the 16-bit value `0x0065` at the numerical address equal to twice the second derived address. This is a halfword access, not a byte access.

A signed word is then read from second derived address + `0x64`; call it the **linked value**.

The low byte of the capture from primary base + `0x14` is written at first derived address + `0x11`. The word value `100`, equivalently `0x00000064`, is subsequently written at first derived address + `0x54`.

A word is captured from second derived address + `0x74` after those writes.

A signed word is then read from tertiary base + `0x64`; call it the **final destination address**. The final three writes before the indirect invocation are:

| Order | Destination | Width | Value |
|---:|---|---:|---|
| 1 | Tertiary base + `0x09` | 8 bits | Second derived address’s low byte |
| 2 | Final destination address + `0x54` | 32 bits | Linked value |
| 3 | Twice the linked value | 32 bits | Final destination address |

Both the second derived address and linked value are represented as signed 32-bit quantities. Their doubling has unresolved signed-overflow behavior. Neither doubled value is guaranteed to identify valid storage.

### Final indirect invocation

After the preceding accesses, the unresolved target receives four arguments in this order:

1. Numeric value `0x72`.
2. Linked value.
3. Word captured from tertiary base + `0x14`.
4. Word captured from second derived address + `0x74`.

The latter three are 32-bit captured values in the supplied representation. The first is an integer constant; the actual argument-passing convention is unresolved.

The arguments do not request fresh reads of their originating memory. Subsequent changes to that storage do not change the captured argument values.

## Side effects

The represented operation directly:

- Reads a word beginning at absolute address `0xA9`.
- Writes bytes at absolute addresses `0x3A`, `0x67`, and `0x21`.
- Reads the byte at `0x67` on multiple later occasions.
- Reads and writes through the primary and tertiary bases.
- Writes through the secondary base without using it as a memory-read base.
- Reads through the explicit input address.
- Performs accesses through doubled values and loaded address-like words.

No broader object layout is established. Access ranges may overlap, and descriptive roles do not imply independent allocations.

No atomicity, synchronization, transaction, rollback, or exception-recovery guarantee is represented. Failure or interruption may leave earlier writes visible.

No explicit allocation or release occurs before the indirect invocation. This does not establish ownership, retention rights, or release obligations for any region.

The unresolved target may change globals, accessed memory, or other state. Consequently, stores preceding the invocation do not by themselves guarantee final contents after it returns.

## Dependencies

No opaque function ID is supplied for the indirect target. Its identity is **unresolved — `documented_only`**.

The supplied mapping identifies F-9ce3d6c7c7fc itself; it does not establish that the indirect transfer targets the same function. No recursive dependency or additional function identity is inferred.

The target’s calling convention, argument meanings, memory effects, ownership behavior, error behavior, and return guarantees are unknown. In particular, this contract does not assume that it preserves globals or retains or releases referenced storage.

The execution environment must also supply the implicit inputs and support the actual memory accesses. These are environmental requirements, not additional resolved function dependencies.

## Acceptance examples

These are conditional static criteria, not executed tests.

### Secondary destinations need not be independently readable

Assume all specified reads resolve to readable locations, all specified stores resolve to writable locations, and no secondary-base-relative destination overlaps a separately specified read.

Secondary-base-relative locations may be writable without being readable. Such permissions do not violate this contract merely because those locations receive word stores.

If a secondary destination overlaps primary base + `0x40`, that numerical range also needs readability for the separately specified read there. The requirement arises from the read, not from the secondary-base role.

### Opening replacement preserves access history

Assume valid ordinary-memory destinations without relevant overlap, primary base `0x1000`, secondary base `0x2000`, and input word `0xAABBCCDD`.

The opening pattern includes byte `0x00` at `0x2019`, byte `0x00` at `0x1011`, and an initial word `0xAABBCCDD` at `0x2034`. Later opening occurrences write word `0x00002000` at `0x2034`.

Writing only the final word can match a snapshot while failing to reproduce the represented access history.

### Low-byte arithmetic

If the primary base’s low byte is `0xF0`, the byte written at absolute address `0x3A` is `0x10`, the low eight bits of twice `0xF0` plus `0x30`.

No decimal-digit interpretation follows.

### Byte and word overlap

Assume the capture from primary base + `0x40` is `0x11223344`, with no relevant external mutation or additional overlap.

The later word store at primary base + `0x10` replaces the bytes at offsets `0x10` through `0x13`. Which byte occupies each offset depends on target byte order. The earlier byte stores still belong to the represented sequence.

### Captured value survives storage changes

Suppose the read from tertiary base + `0x14` captures word X, and a subsequent aliased write changes that storage.

The later replacement at tertiary base + `0x54` and the third indirect argument use X. They do not reread the modified storage.

### Halfword width and oversized shifts

If doubling the second derived address is supported and the resulting destination permits a halfword store, the operation writes all 16 bits of `0x0065`. A byte-only store of `0x65` is not equivalent.

If the primary base’s low byte is decimal 40, the shift result remains `documented_only`; acceptance must not assume count masking.

An inaccessible required address has no promised error code or rollback behavior.

## Uncertainties

The following remain `documented_only`:

- The public calling convention and delivery of implicit inputs.
- Accuracy of the represented operation boundaries and final control transfer.
- The exact numerical count of opening repetitions.
- Byte order, alignment, address conversion, access permissions, and fault delivery.
- Signed arithmetic overflow and negative signed-right-shift behavior.
- Shift counts greater than 31.
- Storage validity, lifetimes, layouts, and ownership.
- Concurrency, device-memory semantics, and physical access ordering.
- The indirect target’s identity and complete behavior.
- Higher-level purpose and interpretation of constants.

The absence of a secondary-base-relative read is established, not an unresolved readability requirement. Aliasing may add requirements at a numerical address only through other actual accesses.

## Evidence

This contract is based solely on static examination of the supplied representation for F-9ce3d6c7c7fc. Its behavioral claims derive from represented access widths, offsets, ordering, captured values, arithmetic, and the final indirect invocation.

The secondary-base role was checked against its uses: it supplies destination addresses and a low-byte value, with no represented memory read through that base. Primary and tertiary bases do supply word-read addresses.

Embedded comments, the prior draft, and reviewer feedback are not independent proof of executable behavior. No implementation access, dependency contract, runtime observation, or memory trace was available. **No runtime tests have been performed.**

Acceptance examples are conditional deductions from static evidence, not reports of observed execution.
