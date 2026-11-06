<!-- Trusted provenance: F-f7bcc88a3d89; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f7bcc88a3d89

## Purpose

F-f7bcc88a3d89 conditionally replaces linked state associated with a destination object using information obtained from a source object.

For unequal object addresses, it calls a preparation dependency, obtains and publishes a replacement link, updates two further link fields, and copies a 32-bit source field into the destination. Equal object addresses bypass these operations.

The access pattern is consistent with assignment of a sentinel-based container. That interpretation is **inferred intent**, not an established type, ownership policy, or deep-copy guarantee.

The established behavior is an ordered sequence of calls, reads, and writes. It does not guarantee that each stored value survives until return: later dependencies or overlapping writes can change previously updated storage.

## Interface

### Inputs

The function receives four 32-bit arguments.

| Position | Established use |
|---|---|
| First | Destination object address; retained as the low 32 bits of the result. |
| Second | Source object address; compared with the destination and used for source reads on the unequal-address path. |
| Third | No direct use in the supplied body. |
| Fourth | Copied into mutable temporary storage passed by address to F-d0acc006efef. |

The first two arguments are interpreted as addresses for memory access. Their displayed integer representation does not establish portable signed-address arithmetic, permitted address ranges, or wraparound behavior.

The third argument has no established policy, allocator, status, or ownership role. The meaning of the fourth argument beyond its preparation-call use is unknown.

### Accessed storage

Every listed field read or write is 32 bits wide. Offsets are in bytes.

| Base | Offset | Established use |
|---|---:|---|
| Destination object | `0x10` | Repeatedly read to select a linked-state record. |
| Source object | `0x10` | Read after preparation to select a source record. |
| Destination object | `0x14` | Receives the trailing source-field copy. |
| Source object | `0x14` | Supplies the trailing source-field copy. |
| Source record | `0x04` | Supplies an argument to F-cbe466322151. |
| Destination record | `0x04` | Receives the replacement result; freshly read on the nonzero branch. |
| Destination record | `0x08` | Read before preparation; written on either replacement branch. |
| Destination record | `0x0C` | Written after the branch-specific offset-`0x08` update. |

“Object,” “record,” and “link” describe access roles, not established types.

The destination record address is not captured once for the entire operation. Repeated reads of destination offset `0x10` can select different records. Object storage and record storage are not established to be disjoint.

### Result

Normal return produces a 64-bit bit-pattern composition:

- Low 32 bits: the original destination argument.
- High 32 bits: the final contents of a mutable 32-bit return-state temporary.

This construction concatenates two 32-bit halves; it is not sign extension.

The return-state temporary initially contains the source argument. On the unequal-address path, the function replaces that value with a destination record address before passing the temporary’s address to F-d0acc006efef.

That dependency can modify the temporary. The function does not explicitly restore it afterward. Later effects through an escaped temporary address are also not excluded by the available evidence.

### Preconditions and error handling

There is no explicit validation, null check, bounds check, allocation-failure check, or error-result conversion.

On the unequal-address path, every address actually dereferenced must support the corresponding 32-bit access. Store locations must be writable. Dependencies may impose additional, unknown preconditions.

A zero replacement result selects a normal update branch. It is not established to indicate failure.

Equal object addresses bypass all explicit object-memory accesses, even when the shared value would not be a valid dereferenceable address.

## Behavior

### Equal-address path

The function initializes the return-state temporary from the source argument, then compares the source and destination addresses.

If they are equal:

- No dependency is called.
- No explicit object or record field is read or written.
- Both halves of the result contain the shared address value.

The unused third argument and the fourth argument do not affect this path’s established result.

### Preparation

For unequal addresses, the function first reads destination offset `0x10` and places that value into the return-state temporary.

It separately reads destination offset `0x10` again, then reads offset `0x08` through the record address obtained by that second read. The resulting 32-bit value is placed in another temporary.

The fourth input is copied into a third temporary.

These are separate destination-record selections. Without a stability assumption, the record address saved for the eventual result need not equal the record address used for the offset-`0x08` read.

F-d0acc006efef is then called with:

1. The address of the temporary containing the fourth input.
2. The destination object address.
3. The address of the temporary containing the earlier record-offset-`0x08` value.
4. The address of the return-state temporary.

The three temporary addresses identify mutable call-local storage. The dependency’s result, if any, is not used.

The function makes no later explicit use of the first or third addressed temporary. Changes to the fourth addressed temporary affect the high half of the eventual result.

No source-field, destination-field, global-state, or reachable-memory preservation guarantee is established for this dependency.

### Replacement acquisition

After preparation returns, the function obtains:

- The 32-bit link at offset `0x04` of the record selected by the current source offset-`0x10` value.
- The current destination offset-`0x10` value.

F-cbe466322151 receives, in order:

1. The destination object address.
2. The source link just obtained.
3. The destination record address just obtained.

The relative evaluation order of the independent source-link and destination-record argument reads is not established by the supplied representation. Both occur after preparation and before entry into F-cbe466322151.

The dependency supplies a 32-bit replacement result.

### Publication and branch selection

After F-cbe466322151 returns, the function rereads destination offset `0x10` and stores the replacement result at offset `0x04` of the selected record.

This publication target need not be the record address passed to F-cbe466322151. A dependency-induced change to destination offset `0x10` can redirect publication.

The branch decision tests the replacement result itself. It does not reload the published field. Consequently, an aliased store or later change to the published field does not retroactively change branch selection.

### Zero replacement

When the replacement result is zero, the function:

- Stores a value read from destination offset `0x10` into offset `0x08` of a record selected through destination offset `0x10`.
- Reads destination offset `0x10` again.
- Uses that last address as both the value and the record base for the offset-`0x0C` store.

For the offset-`0x08` store, the relative evaluation order of the read supplying the stored value and the read selecting the destination record is not established.

If the destination record address remains stable throughout publication and these updates, then immediately after the offset-`0x0C` store:

- Offset `0x04` contains zero.
- Offset `0x08` contains the record’s own address.
- Offset `0x0C` contains the record’s own address.

These are branch-boundary values, not unconditional final-state guarantees. The trailing destination-field copy may overwrite them.

Neither F-571f9b0c71c1 nor F-93a046b34348 is called on this branch.

### Nonzero replacement

When the replacement result is nonzero, the following ordering is established:

1. F-571f9b0c71c1 is called without explicit arguments.
2. After it returns, destination offset `0x10` is read, and its 32-bit result is stored at the selected record’s offset `0x08`.
3. Destination offset `0x10` is read again, followed by offset `0x04` of the selected record.
4. That freshly loaded link is passed to F-93a046b34348.
5. After this dependency returns, destination offset `0x10` is read again.
6. Its 32-bit result is stored at offset `0x0C` of the record selected by this last read.

F-93a046b34348 need not receive the original replacement result. F-571f9b0c71c1 may change either the selected record address or the offset-`0x04` contents before the fresh argument read.

The two helper-result stores need not target the same record.

The offset-`0x08` store establishes its value at that point in the sequence, but not necessarily at return. F-93a046b34348 executes afterward and may overwrite offset `0x08`, even while preserving both the destination record address and its offset-`0x04` link.

### Trailing field copy and return

After either branch, the function reads source offset `0x14` and writes that 32-bit value to destination offset `0x14`.

The source read occurs after all dependency calls and branch-specific link stores. Earlier writes or dependency effects may therefore influence the value copied.

The destination store occurs after the record-offset-`0x0C` store. If its four-byte range overlaps an earlier record-field write, it can overwrite that earlier value.

The function then constructs the 64-bit result using the return-state temporary and the original destination argument.

## Side effects

### Direct writes

On a normally completing unequal-address path, the function directly performs:

- One replacement-result store at a selected record’s offset `0x04`.
- One branch-specific store at a selected record’s offset `0x08`.
- One branch-specific store at a selected record’s offset `0x0C`.
- One source-field copy into destination offset `0x14`.

These identify access locations, not necessarily four disjoint storage regions.

There is no direct store explicitly addressed as destination offset `0x10`. An indirectly addressed record-field store may nevertheless overlap and change that location. Dependencies may change it as well.

Distinct object addresses do not establish disjoint objects, disjoint records, or separation between object and record storage.

### Preservation limits

A stored value is established immediately after its store completes. Its persistence requires that subsequent operations do not overwrite its storage.

In particular:

- The nonzero branch’s first helper may affect the published offset-`0x04` link.
- The second helper may affect the earlier offset-`0x08` store.
- The trailing copy may overwrite any overlapping record field.

Preserving the destination record address and the published link is insufficient to establish preservation of offset `0x08`.

### Ownership and lifetime

No allocation or release operation is directly identifiable in this body. Dependency names and the linked-state pattern do not establish allocation, destruction, ownership transfer, sharing, or deep copying.

F-d0acc006efef receives addresses of local temporaries. Whether it retains those addresses is unknown. No lifetime extension beyond the activation is established, and no guarantee excludes later access through escaped addresses.

### Failure and interruption

The supplied body contains no visible transaction, rollback, synchronization, or recovery mechanism.

If a dependency does not return normally, or a required memory access cannot complete, subsequent operations are not established to occur. Earlier changes are not established to be undone.

## Dependencies

| Opaque ID | Explicit call-site interface | Established role |
|---|---|---|
| F-d0acc006efef | Temporary address, destination address, temporary address, return-state temporary address | Preparation before replacement; receives mutable local storage. |
| F-cbe466322151 | Destination address, source link, destination record address; 32-bit result | Supplies the published replacement and the branch decision. |
| F-571f9b0c71c1 | No explicit arguments; 32-bit result | Nonzero branch only; supplies the offset-`0x08` store value. |
| F-93a046b34348 | One freshly loaded destination link; 32-bit result | Nonzero branch only; supplies the offset-`0x0C` store value. |

Algorithms, hidden inputs, global effects, memory preservation, ownership effects, and exceptional behavior remain unspecified.

Absence of an explicit argument does not establish purity. A dependency is not assumed to preserve memory merely because that memory is absent from its explicit arguments.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unequal-address examples assume valid accessed storage, valid address calculations, and normal dependency returns.

### Equal addresses

For destination and source arguments both equal to `0x12345678`, the result is `0x1234567812345678`.

No dependency calls or explicit object-memory accesses occur.

### Zero replacement with separated trailing storage

Assume a stable destination record address, a zero replacement result, and no overlap between destination offset `0x14` and the three record fields.

The record contains zero at offset `0x04` and its own address at offsets `0x08` and `0x0C` after the branch updates. With no intervening external mutation, those values survive the trailing copy.

Neither nonzero-branch helper runs.

### Trailing copy overwrites the published zero

Assume:

- Destination address `0x1000`.
- Source address `0x2000`.
- Stable destination record address `0x1010`.
- A zero replacement result.
- No dependency changes relevant to the example.
- Source offset `0x14` contains `0xA5A5A5A5` when read.

Destination offset `0x14` and record offset `0x04` both identify address `0x1014`.

Publication writes zero there. The zero branch writes `0x1010` at addresses `0x1018` and `0x101C`. The trailing copy then replaces the zero at `0x1014` with `0xA5A5A5A5`.

The final offset-`0x04` value is therefore nonzero, but neither nonzero-branch helper runs: branch selection used the replacement result.

### Nonzero replacement with explicit preservation

Assume:

- A stable destination record address.
- A nonzero replacement result.
- Neither helper changes the published offset-`0x04` link.
- F-93a046b34348 preserves offset `0x08` after F-571f9b0c71c1’s result has been stored there.
- The trailing destination-field write does not overlap the three record fields.
- No external mutation affects those fields.

Then the final record contains:

| Offset | Final value |
|---|---|
| `0x04` | The replacement result. |
| `0x08` | F-571f9b0c71c1’s result. |
| `0x0C` | F-93a046b34348’s result. |

F-93a046b34348 receives the published replacement link.

The explicit offset-`0x08` preservation assumption is necessary; stability of the record address and published link alone does not imply it.

### Later helper overwrites offset `0x08`

Assume a stable record, a preserved published link, and no overlap with the trailing copy.

Suppose F-571f9b0c71c1 returns `0x11111111`, which the function stores at offset `0x08`. During its subsequent call, F-93a046b34348 writes `0x22222222` to that same field and returns `0x33333333`, without changing the record address or offset `0x04`.

With no further overwrite:

- Final offset `0x08` is `0x22222222`.
- Final offset `0x0C` is `0x33333333`.

This behavior is consistent with the established call sequence. No available dependency contract excludes it.

### Dependency-induced redirection

If F-cbe466322151 changes destination offset `0x10`, publication uses the post-call record address, not necessarily its third argument.

If F-571f9b0c71c1 changes the current offset-`0x04` link, F-93a046b34348 receives the freshly loaded value.

### Return-state mutation

If preparation changes the return-state temporary to `0x89ABCDEF`, the destination argument is `0x12345678`, and no later effect changes that temporary, the result is `0x89ABCDEF12345678`.

The high half is not required to equal either object address or any record address.

## Uncertainties

The following remain **documented_only**:

- Concrete container type and semantic field names.
- Meaning of object offset `0x14`.
- Meaning of the third input and the fourth input beyond their explicit treatment.
- Dependency algorithms, hidden inputs, preservation guarantees, ownership, allocation, release, and failure behavior.
- Whether the result represents one logical 64-bit value or an interface-specific pairing.
- Alignment requirements, permitted address ranges, and address-overflow behavior.
- Behavior under invalid storage or concurrent mutation.
- Relative evaluation order of independent reads within individual expressions.
- Whether caller contracts prohibit particular overlaps.
- Whether temporary addresses escape and what subsequent access that permits.

All directly accessed data and dependency results described here are 32 bits. The explicit return composition is 64 bits. Fixed byte offsets establish access locations, not a portable rule for overflowing address calculations. No element-count arithmetic or overflow-recovery mechanism is visible.

## Evidence

The supplied body establishes the equality guard, 32-bit accesses, repeated record-address reads, dependency argument positions, replacement-result branch, ordered link stores, trailing field copy, and two-half return composition.

The preservation limit at offset `0x08` follows from ordering: its store precedes F-93a046b34348, and no supplied contract restricts that dependency from modifying the field. A stable record address and an unchanged offset-`0x04` link do not restrict writes to offset `0x08`.

The overlap example follows from the accessed offsets: destination `0x1000` plus `0x14` and record `0x1010` plus `0x04` both identify `0x1014`. The trailing copy occurs after publication and branch updates.

The sentinel-container interpretation remains inferred intent. Dependency implementations, caller contracts, concrete type declarations, and runtime traces were not supplied.

No runtime tests have been performed.
