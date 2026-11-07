<!-- Trusted provenance: F-7ce13fe0c26f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7ce13fe0c26f

## Purpose

F-7ce13fe0c26f applies one selected state operation to an object obtained through an indexed record table. Its inputs are a signed record index and a 32-bit operation mask. Its observable state effects are confined, in the supplied representation, to three 32-bit fields in the selected object.

The mask selects operations by precedence; it does not describe a collection of operations to apply independently. The highest-priority recognized bit determines the complete action. Several actions replace fields with constants, while four actions update one field using a bitwise operation.

The established behavior is state manipulation. Describing the fields as application features, permissions, status indicators, or enable flags would require evidence not supplied here.

## Interface

### Inputs

| Input | Represented type | Contract |
|---|---|---|
| Record index | Signed 16-bit integer | Selects a record at a signed displacement from a shared table base. |
| Operation mask | Unsigned 32-bit integer | Selects an action through recognized bits `0x001` through `0x200`, with the precedence specified below. |

The represented signed index domain is −32,768 through 32,767. Negative indices are not rejected. Whether any negative index identifies valid storage is an external precondition, not a guarantee of this function.

The operation mask has 32 bits. Bits outside `0x000003FF` do not influence action selection. They do not prevent recognized lower bits from taking effect.

### Result

There is no explicit return value. The function does not report success, invalid input, or an unsuccessful memory access through a status result.

### Storage layout

The supplied representation establishes the following addressing scheme:

- A shared location supplies a table base.
- Records are spaced `0x18C` bytes apart, equivalent to 396 bytes.
- The first four bytes at the selected record are read as an object address.
- Three four-byte fields are accessed relative to that object address.

For this contract, those fields have neutral names:

| Field | Byte offset from object base | Width |
|---|---:|---:|
| A | `0x1004` | 32 bits |
| B | `0x1008` | 32 bits |
| C | `0x100C` | 32 bits |

A is interpreted as an unsigned 32-bit value when read for bitwise updates. The shown accesses to B and C are constant writes only; their prior contents are not read.

These names do not imply any meaning beyond identifying storage locations.

### Preconditions

Normal completion requires the shared table-base access and selected record-address read to be valid. Any field read or write selected by the mask must also be valid for the target object.

There are no visible checks for:

- Index bounds.
- Null table or object addresses.
- Object extent.
- Access permissions.
- Alignment.
- Concurrent modification.

An object extent covering all three fields would extend through byte offset `0x100F`, inclusive. This is a sufficient storage extent for the shown field accesses, not a claim about the complete object size.

## Behavior

### Target resolution

Before inspecting the operation mask, the function resolves the target object:

1. Obtain the shared table base.
2. Select the record at the base plus the signed record index multiplied by 396 bytes.
3. Read the four-byte object address at the start of that record.

This resolution occurs even when the operation mask ultimately selects no field action. Consequently, a mask of zero does not eliminate the table and record reads.

The signed index is not converted to an unsigned 16-bit index in the supplied expression. The multiplication by 396 fits within a signed 32-bit integer across the entire signed 16-bit index domain. The subsequent address calculation can still identify invalid storage; no bounds or address-validity protection is visible.

### Action selection

The first matching row below determines the entire action. Lower-priority recognized bits are ignored once a row matches.

| Priority | Selected mask bit | Field action |
|---:|---:|---|
| 1 | `0x200` | Write A = 0, then B = 0, then C = 0. |
| 2 | `0x100` | Write A = 3, then B = 1, then C = 1. |
| 3 | `0x001` | Write B = 1. |
| 4 | `0x002` | Write B = 0. |
| 5 | `0x004` | Write C = 1. |
| 6 | `0x008` | Write C = 0. |
| 7 | `0x010` | Read A, then write A with its prior value bitwise-ORed with `0x00000001`. |
| 8 | `0x020` | Read A, then write A with its prior value bitwise-ANDed with `0x00000002`. |
| 9 | `0x040` | Read A, then write A with its prior value bitwise-ORed with `0x00000002`. |
| 10 | `0x080` | Read A, then write A with its prior value bitwise-ANDed with `0x00000001`. |
| None | No recognized bit set | Perform no object-field access after target resolution. |

All constants in field writes occupy the full 32-bit field. For example, writing B = 1 replaces all four bytes of B; it is not a byte write or an update of only its lowest bit.

### Replacement and preservation rules

The `0x200` and `0x100` actions replace all three fields. Neither action reads their previous values.

The `0x001` and `0x002` actions replace B only. A and C are not accessed through their field offsets on those paths.

The `0x004` and `0x008` actions replace C only. A and B are not accessed through their field offsets on those paths.

The remaining four actions read and replace A only:

- `0x010` sets bit 0 while preserving all other bits.
- `0x020` retains only bit 1. Every other bit is cleared.
- `0x040` sets bit 1 while preserving all other bits.
- `0x080` retains only bit 0. Every other bit is cleared.

In particular, the two AND actions are not general-purpose single-bit clearing operations. If A contains high-order bits, both discard those bits. The `0x020` action produces only 0 or 2; the `0x080` action produces only 0 or 1.

### Ordering and write occurrence

The three-write actions occur in A, B, C order. The supplied representation does not make these actions transactional. No rollback is shown if an access fails after an earlier write.

Each A read-modify-write action reads A before writing its replacement. There is no second visible read of A on that action’s path.

Selected writes occur even if the destination already contains the requested value. A zero result from an AND operation does not suppress its write.

These ordering statements describe the supplied access sequence. They do not establish hardware atomicity, inter-thread synchronization, memory barriers, or device-memory behavior.

## Side effects

### Memory effects

The function reads shared addressing state and the selected record’s object-address entry on every path reaching mask selection.

Depending on the selected action, it then:

- Writes three object fields.
- Writes one object field.
- Reads and writes A.
- Performs no object-field access.

There is no explicit modification of the shared table base or record entry. However, the supplied material does not establish that the object storage is disjoint from those locations. If storage aliases, a field write may also alter data used for addressing by later invocations.

### Ownership and lifetime

No allocation, release, reference-count adjustment, ownership transfer, or lifetime extension is visible. The function operates on externally supplied storage reached through the shared table.

The caller or surrounding environment must provide an appropriate lifetime for the table, record, and accessed object fields. The function offers no visible protection against their concurrent replacement or release.

### Failure behavior

No explicit error branch, exception handler, diagnostic, or recovery action appears in the supplied representation.

Invalid memory accesses have no defined outcome in this contract beyond the absence of a visible handling path. A multi-field action must not be assumed to complete fully if a later access fails. Platform-specific fault delivery and language-level consequences are documented_only.

No I/O, callback, or external call is visible.

## Dependencies

No called-function dependency is present in the supplied body. There are therefore no additional opaque function IDs to enumerate.

The behavioral dependencies are data and execution-environment dependencies:

- A readable shared table-base location.
- A record layout with a 396-byte stride.
- A readable four-byte object-address entry at the selected record start.
- Readable or writable object fields as required by the selected action.
- An address representation compatible with the represented four-byte object-address load.

No callee behavior needs to be assumed. In particular, this contract does not rely on an unknown callee preserving globals or managing memory.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not results of executed tests. Unless stated otherwise, storage is valid, field locations are distinct from addressing storage, and no concurrent actor modifies them.

### Selection and precedence

| Mask | Initial A, B, C | Required resulting A, B, C | Required field-access behavior |
|---:|---|---|---|
| `0x200` | Any values | 0, 0, 0 | Write A, then B, then C; no prior-field reads. |
| `0x3FF` | Any values | 0, 0, 0 | `0x200` wins over every other recognized bit. |
| `0x100` | Any values | 3, 1, 1 | Write A, then B, then C. |
| `0x101` | Any values | 3, 1, 1 | `0x100` wins over `0x001`. |
| `0x003` | 9, 7, 5 | 9, 1, 5 | `0x001` wins; only B is written. |
| `0x006` | 9, 7, 5 | 9, 0, 5 | `0x002` wins over `0x004`. |
| `0x00C` | 9, 7, 5 | 9, 7, 1 | `0x004` wins over `0x008`. |
| `0x018` | 9, 7, 5 | 9, 7, 0 | `0x008` wins over `0x010`. |

### Full-width bitwise behavior

| Mask | Initial A | Required resulting A | Reason |
|---:|---:|---:|---|
| `0x010` | `0x80000002` | `0x80000003` | Set bit 0; preserve higher bits. |
| `0x020` | `0xFFFFFFFF` | `0x00000002` | Retain only bit 1. |
| `0x020` | `0x80000001` | `0x00000000` | Neither the high bit nor bit 0 survives. |
| `0x040` | `0x80000001` | `0x80000003` | Set bit 1; preserve other bits. |
| `0x080` | `0xFFFFFFFF` | `0x00000001` | Retain only bit 0. |
| `0x030` | `0x80000000` | `0x80000001` | `0x010` wins; the AND action is not applied. |

B and C receive no field accesses in these examples.

### No-action and indexing cases

- With mask `0x00000000`, the shared base and selected record entry are still read, but no A, B, or C field access follows.
- With mask `0x80000000`, behavior is the same no-field-action path.
- With mask `0x80000002`, B is written to zero; the high mask bit has no selection effect.
- With index 1, the object-address entry is read 396 bytes after the table base.
- With index −1, the object-address entry is read 396 bytes before the table base, conditional on that access being valid.
- With `0x001` and B already equal to 1, a 32-bit write of 1 still occurs.

## Uncertainties

### Established static evidence

The supplied representation establishes the input widths, signed index use, record stride, field offsets, action precedence, constants, bitwise operations, and explicit access order described above.

### Inferred intent

The paired writes to B and C are consistent with binary state controls. The operations on A are consistent with manipulating its lowest two bits. These interpretations are not required behavior: neither field semantics nor a restriction of A to values 0 through 3 is established.

### documented_only

The following remain unspecified:

- Table length and valid index range.
- Meaning and initialization of the shared addressing state.
- Application meaning of A, B, and C.
- Complete object layout and size.
- Byte order, alignment guarantees, and exact address-arithmetic rules.
- Whether the represented addresses are native pointers or another compatible address encoding.
- Aliasing constraints.
- Thread safety, atomicity, and synchronization.
- Fault behavior for invalid addresses or inaccessible fields.

## Evidence

This contract is based solely on the supplied static function representation for F-7ce13fe0c26f. It separates directly represented memory behavior from inferred field intent and unavailable environmental guarantees.

No implementation execution, runtime tests, fault experiments, concurrency checks, or independent binary verification have been performed. Acceptance examples express required outcomes under their stated preconditions; they are not observations of a running program.
