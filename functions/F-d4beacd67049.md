<!-- Trusted provenance: F-d4beacd67049; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d4beacd67049

## Purpose

F-d4beacd67049 conditionally updates a caller-supplied state record, invokes one of two external dependencies, and marks the record through a one-byte write.

For either recognized mode, the function constructs an ordered pair from:

- The third input, retained as an unsigned 32-bit value.
- The least significant eight bits of a stored 32-bit word, widened to an unsigned 32-bit value.

The mode determines the order of this pair. The function stores the pair and the second input into the first three words of the record before selecting and invoking a dependency.

An unrecognized mode causes an early return without record writes or dependency invocation.

The supplied evidence establishes these operations, but not their application-level meaning. Descriptions such as “coordinates,” “activation,” or “initialization” would be inferred intent, not established behavior.

## Interface

### Inputs and result

The function accepts three inputs:

| Input | Contract |
|---|---|
| State pointer | Points to caller-supplied storage containing the fields described below. The function reads and modifies this storage. |
| Second input | An unsigned 32-bit value stored unchanged in the record and, on one dependency path, passed unchanged as an argument. |
| Third input | An unsigned 32-bit value placed in one of the first two record words according to the mode. It is also supplied to the argument-bearing dependency in the corresponding position. |

The function has no returned value. Its observable results are record mutations, dependency invocation, and any effects of the invoked dependency.

No null-pointer, bounds, alignment, or access-permission checks are present.

### Record layout

Offsets are byte offsets from the state pointer. A word is 32 bits in the supplied representation.

| Offset | Width | Use |
|---|---:|---|
| 0 | 32 bits | First selected value; written on recognized modes. |
| 4 | 32 bits | Second selected value; written on recognized modes. |
| 8 | 32 bits | Second input; written on recognized modes. |
| 12 | 32 bits | Read to select the dependency, using only its least significant eight bits. |
| 12 | 8 bits | Written with numeric value 1 after the selected dependency returns normally. |
| 15 | 8 bits | Read as an additional argument on the argument-bearing dependency path. |
| 16 | 32 bits | Read on the argument-bearing dependency path; its least significant eight bits are converted to a signed character-sized value and then widened to an integer. |
| 28 | 32 bits | Read on recognized modes; its least significant eight bits provide the stored member of the selected pair. |
| 30 | 8 bits | Mode selector. |

The distinction between numeric low-byte extraction and byte-addressed access is important. Extracting the least significant eight bits of a word does not, without a byte-order assumption, establish that the byte at that word’s starting address is being selected.

The final write is explicitly a byte write at offset 12. It is not a replacement of the entire word at offset 12.

### Storage and ownership preconditions

The state pointer must support every access reached on the selected path:

- The mode access requires a readable byte at offset 30.
- A recognized mode additionally requires a readable 32-bit word at offset 28, writable words at offsets 0, 4, and 8, and a readable word at offset 12.
- The argument-bearing dependency path additionally requires a readable byte at offset 15 and a readable word at offset 16.
- After a normally returning dependency call, the byte at offset 12 must remain writable.

A suitably aligned, readable and writable record of at least 32 bytes is a straightforward sufficient storage arrangement before the call. It is not a guarantee that an unknown dependency will preserve that arrangement.

The function itself contains no allocation or release operation. Ownership transfer, if any, through external state or dependency behavior is unknown.

## Behavior

### 1. Mode selection

The function first examines the byte at offset 30.

- If the byte is zero, it selects mode 0.
- Otherwise, it checks the mode byte again and proceeds only if that second check equals 1.
- If the second check does not equal 1, it returns immediately.

Under an ordinary stable-memory assumption, the recognized modes are therefore exactly 0 and 1. Every other byte value causes an early return.

The two nonzero-path checks are separate accesses in the supplied representation. If the record can change between them, selection cannot be described solely by the mode value observed at entry. Such concurrent modification is not established as supported.

### 2. Selecting the first two stored values

For a recognized mode, the function reads the 32-bit word at offset 28 and retains only its least significant eight bits. The retained value is widened without sign extension, so its numeric range is 0 through 255.

The ordered pair is:

| Selected mode | First value | Second value |
|---|---|---|
| 0 | Third input | Extracted unsigned eight-bit value |
| 1 | Extracted unsigned eight-bit value | Third input |

The third input is not narrowed. Values above 255, including values with the high bit set, retain their full unsigned 32-bit value.

Bits 8 through 31 of the word at offset 28 do not contribute to the selected value. The word overlaps the mode byte’s storage, but the relationship between those numeric bit positions and offset 30 depends on byte order.

### 3. Updating the leading record words

The function performs three full-word writes in this order:

1. First selected value to offset 0.
2. Second selected value to offset 4.
3. Second input to offset 8.

These writes occur before dependency selection and before either dependency is invoked.

There is no rollback mechanism in the supplied function. If a later access or dependency invocation does not complete normally, the leading record writes have already been reached.

### 4. Selecting and invoking a dependency

After the three writes, the function reads the word at offset 12 and tests whether its least significant eight bits are zero. The upper 24 bits do not affect this decision.

#### Zero selector

If the tested eight-bit value is zero, the function invokes **D-01**, an opaque external dependency, with five arguments:

| Position | Value |
|---|---|
| 1 | First selected value, unsigned 32-bit. |
| 2 | Second selected value, unsigned 32-bit. |
| 3 | Second input, unsigned 32-bit. |
| 4 | Raw eight-bit value read from offset 15. |
| 5 | Integer obtained from the least significant eight bits of the word at offset 16 through a character-sized signed interpretation. |

For the fifth argument, the supplied representation indicates signed widening: low-byte values 0 through 127 remain nonnegative, while values 128 through 255 represent −128 through −1. The original target’s plain-character signedness is not independently established; this qualification matters for the upper half of the byte range.

The supplied evidence does not establish the relative evaluation order of the offset-15 read and the offset-16 word read. Both supply arguments to D-01 and occur after the leading record writes.

#### Nonzero selector

If the tested eight-bit value is nonzero, the function invokes **D-02**, an opaque external dependency, with no explicit arguments.

On this path, the function does not perform the additional argument reads at offsets 15 and 16.

### 5. Final marker write

After the selected dependency returns normally, the function writes the one-byte value 1 at offset 12 and returns.

This write occurs on both dependency paths. It does not occur:

- On an unrecognized-mode early return.
- Before the dependency call.
- If the dependency does not return normally.

The function does not inspect a dependency result or condition the final write on reported success.

The final byte store can overwrite a dependency’s intervening modification to that same byte. Other dependency changes are not restored by this function.

## Side effects

### Direct record effects

For a recognized mode and normal completion, the function directly performs:

- Three 32-bit writes at offsets 0, 4, and 8.
- One dependency invocation.
- One eight-bit write of 1 at offset 12 after that invocation returns.

The final byte write directly leaves the other three bytes of the word at offset 12 untouched. That does not guarantee they retain their entry values: a dependency may have changed them.

For an unrecognized mode, the function performs no direct record writes and invokes neither dependency.

### External effects and lifetime

The dependencies’ effects are unknown. The function does not explicitly pass the state pointer to either dependency, but this is not evidence that they cannot access the record through globals, aliases, or other external state.

No preservation guarantee is established for:

- Global state.
- Record contents.
- Allocations or resource ownership.
- Reentrancy or callbacks.
- The lifetime of the supplied storage.

The caller’s valid-storage obligation extends through the final byte write. If a dependency invalidates that storage, the function has no visible protection against the subsequent access.

No locking, atomic synchronization, explicit error reporting, or recovery mechanism appears in the supplied body.

## Dependencies

The following are document-local opaque dependency identifiers. Their implementations and authoritative external identities were not supplied.

| Identifier | Established use | Unknowns |
|---|---|---|
| D-01 | Invoked when the selector’s low eight bits are zero; receives the five arguments described above. Any result is unused. | Behavior, ABI details beyond the visible argument expressions, external effects, resource handling, and failure behavior. |
| D-02 | Invoked when the selector’s low eight bits are nonzero; receives no explicit arguments. Any result is unused. | Behavior, implicit inputs, external effects, resource handling, and failure behavior. |

Neither dependency is established to preserve globals or memory ownership. Normal return is required to reach the final marker write, but normal return alone does not prove application-level success.

## Acceptance examples

These examples are conditional static expectations, not results of executed tests. Unless stated otherwise, assume valid storage, stable record contents during local reads, and a dependency that returns normally without modifying the record.

### Mode 0 preserves a full-width third input

Given:

- Mode byte 0.
- Third input `0xFEDCBA98`.
- Second input `0x12345678`.
- Least significant byte of the word at offset 28 equal to `0xAB`.

The leading word writes must be:

| Offset | Written value |
|---|---|
| 0 | `0xFEDCBA98` |
| 4 | `0x000000AB` |
| 8 | `0x12345678` |

The third input must not be truncated to `0x98`.

### Mode 1 reverses the pair

Given mode byte 1 and the same other values, the writes must instead be:

| Offset | Written value |
|---|---|
| 0 | `0x000000AB` |
| 4 | `0xFEDCBA98` |
| 8 | `0x12345678` |

Dependency selection still depends on the word at offset 12, not on the mode.

### An unrecognized stable mode does nothing further

Given a stable mode byte of 2 or 255, the function must return without:

- Writing the leading words.
- Reading the stored pair value at offset 28.
- Invoking either dependency.
- Writing the final marker.

This path still requires the mode-byte read itself to be valid.

### A zero low byte selects D-01 despite other set bits

Given a recognized mode and a selector word whose least significant eight bits are zero, D-01 is selected even if other bits in that word are nonzero.

D-01 receives the same selected pair and second input used for the leading writes. Its fourth argument comes from offset 15; its fifth comes from the low eight bits of the word at offset 16.

Under signed eight-bit character interpretation, fifth-argument boundary examples are:

| Extracted bits | Integer argument |
|---|---:|
| `0x7F` | 127 |
| `0x80` | −128 |
| `0xFF` | −1 |

### Any nonzero low byte selects D-02

Given a recognized mode and selector low byte `0x80`, D-02 is selected. The function neither validates the selector against 1 nor reads the additional argument fields for D-01.

After normal return, the byte at offset 12 is written with 1.

### Dependency effects remain visible except at the final byte

If a dependency changes the word at offset 0, the function does not restore the earlier written value.

If a dependency changes the byte at offset 12 to another value and returns with the storage still valid, the function overwrites that byte with 1.

### Repeated-call behavior depends on byte order and dependency effects

Under little-endian word layout, normal completion with no dependency modification leaves the selector word’s least significant byte equal to 1. A later recognized-mode call therefore selects D-02 unless something changes that selector first.

This conclusion is conditional. The unconditional evidence establishes a byte write at offset 12, not a target-independent assignment to the selector word’s least significant bits.

## Uncertainties

The following remain **documented_only**:

- **Target byte order.** Numeric low-byte extraction and byte-addressed fields are distinguishable without resolving their physical correspondence.
- **Original character signedness.** Signed fifth-argument behavior follows the supplied representation; independent ABI confirmation is absent.
- **Machine-level access trace.** The source-facing order above does not prove the number, width, or order of physical memory transactions after compilation.
- **Concurrent modification semantics.** No synchronization is shown; changing mode or record fields during execution is not established as a supported use.
- **Dependency behavior.** No guarantee exists concerning memory lifetime, global preservation, resource ownership, success, or failure.
- **Semantic field meanings.** The record’s domain and the purpose of the two dependency paths are unknown.
- **Invalid-access behavior.** There is no explicit handling for null, inaccessible, misaligned, or prematurely invalidated storage.

## Evidence

The supplied function body is the sole static evidence. It supports the mode branches, widths and offsets, unsigned extraction, pair ordering, three leading writes, dependency selection, argument construction, and final byte write after normal dependency return.

No dependency implementations, independent type declarations, ABI documentation, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples describe conditional consequences of the static evidence, not verified execution results.
