<!-- Trusted provenance: F-bd84bce4313a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bd84bce4313a

## Purpose

F-bd84bce4313a retrieves one of two 32-bit values associated with an indexed record and writes the selected result to caller-provided storage.

Selection 0 exposes the low 27 bits of its field. Selection 1 exposes all 32 bits of a different field. Every other selection returns an error bit pattern without accessing the output destination.

Before rejecting an unsupported selection, the function reads an external table-base datum and then reads a record address from the indexed table slot. An unsupported selection therefore does not bypass either of these reads.

This contract describes behavior established by the supplied static representation. The meanings of the records and fields, their allocation, and the surrounding application’s validity rules are not established.

## Interface

### Inputs

| Input role | Width and interpretation | Function |
|---|---|---|
| Record index | Signed 16-bit integer | Selects an address-bearing table slot at a stride of 396 bytes. |
| Selection | Unsigned 32-bit integer | Chooses field 0 or field 1. All other values are unsupported. |
| Output destination | Address of storage for one unsigned 32-bit value | Receives the result for an accepted selection. |

The record index ranges from −32,768 through 32,767. This representable range is not an established valid-record range. The function contains no explicit index bounds check and does not reject negative indices.

The selection is unsigned. For example, `0xFFFFFFFF` is an unsupported selection, not a negative input with separate behavior.

The supplied representation does not independently establish the complete calling convention or the width of the output pointer.

### Return value

The return value is 32 bits wide.

| Condition | Returned bit pattern |
|---|---|
| Selection 0 or 1, after the required output write completes normally | `0x00000000` |
| Any other selection, after the preliminary reads complete normally | `0xFFFFFF06` |

The error bit pattern represents −250 if interpreted as a signed 32-bit integer. Its symbolic name and application-level error category are unknown.

The selected field value is delivered through the output destination, not through the return value. No memory-access failure is explicitly converted into either return value.

### External state and relevant layout

The function obtains its table base by reading an external datum. That datum is storage containing the base value; it is not established as an immutable address constant.

Using the base value obtained by that read, the function locates a table slot with a stride of 396 bytes, or `0x18C`. At the beginning of the selected slot, it reads a 32-bit address-bearing value identifying a record.

The two fields are relative to that record address:

| Selection | Record-relative offset | Logical read width | Delivered value |
|---|---:|---:|---|
| 0 | 2,712 bytes, or `0xA98` | 32 bits | Field value masked with `0x07FFFFFF` |
| 1 | 2,840 bytes, or `0xB18` | 32 bits | Field value unchanged |

The table-slot stride does not establish the size of the addressed record. Neither complete structure layout is supplied.

The declaration of the external table-base datum is not supplied, so its exact storage width is not independently established. The 32-bit slot read and the two 32-bit field reads are explicit in the supplied representation.

### Preconditions for normal completion

Every path, including an unsupported selection, requires:

- Readable storage for the external table-base datum.
- A base value that, together with the signed index displacement, identifies a readable 32-bit table slot.
- Address arithmetic and accesses permitted by the execution environment.

An accepted selection additionally requires:

- A record address from the slot that permits the selected 32-bit field read.
- An output destination that permits one 32-bit write.
- Storage lifetimes covering the accesses that use them.

There are no explicit null, bounds, alignment, or accessibility checks.

An unsupported selection does not require a usable record address or output destination. This exemption applies only after the external table-base datum and indexed address slot have been read successfully.

## Behavior

### Preliminary address acquisition

The function first reads the external table-base datum. It uses the resulting base value and the signed record index to calculate the address of the selected slot. It then reads the 32-bit record address stored at that slot.

Both reads precede selection rejection. Describing the initial access merely as a table-slot read would omit an observable dependency: the table-base datum must itself be read.

The index displacement is the signed index multiplied by 396 bytes. Across the full signed 16-bit range, its mathematical bounds are:

- Minimum: −12,976,128 bytes.
- Maximum: 12,975,732 bytes.

The multiplication fits in a signed 32-bit integer. That fact does not establish safe addition to an arbitrary base, valid storage at the resulting address, or behavior at address-space boundaries.

Under ordinary address arithmetic without a boundary crossing, a negative index selects a slot before the obtained base. No error return is specifically assigned to negative indices.

The base is the value obtained from external storage for this invocation. The function does not establish that the datum is constant across invocations or protected against concurrent modification.

### Unsupported selection

Every unsigned selection greater than 1 is rejected after the preliminary reads.

If those reads complete normally, the function returns `0xFFFFFF06`. On this path:

- Neither record field is read.
- The record address obtained from the table is not subsequently dereferenced.
- The output destination is not read or written.
- No output initialization or fallback value is supplied.

If access to the table-base datum or indexed slot fails, a normal error return is not established. The function has no explicit recovery path that converts such a failure into selection rejection.

### Selection 0

For selection 0, the function reads the 32-bit field at record offset `0xA98`. It retains bits 0 through 26 and clears bits 27 through 31 by applying `0x07FFFFFF`.

The delivered value is therefore in the inclusive range zero through `0x07FFFFFF`.

The function writes that result to the output destination and then returns zero. The masking operation does not itself write back to the source field. A source-field change can nevertheless occur if the output destination aliases that field.

### Selection 1

For selection 1, the function reads the 32-bit field at record offset `0xB18`, writes all 32 bits unchanged to the output destination, and then returns zero.

No masking, scaling, signed-value conversion, or combination with the other field is established.

The supplied control flow contains a further selection check within the path already restricted to accepted nonzero selections. For the unsigned 32-bit input described here, that path can only have selection 1. It does not establish an additional reachable success case without an output write.

### Observable access order

For a normally completing accepted selection, the established logical order is:

1. Read the external table-base datum.
2. Read the 32-bit record address from the indexed slot calculated using that base value.
3. Read the selected 32-bit record field.
4. Write one 32-bit result to the output destination.
5. Return zero.

For an unsupported selection, the established order is:

1. Read the external table-base datum.
2. Read the 32-bit record address from the indexed slot.
3. Reject the selection and return `0xFFFFFF06`, without a field read or output access.

Only the selected field is read. The function does not read both fields and then choose between their values.

This ordering matters for aliasing. The required base, slot, and selected field reads precede the output write. If the destination overlaps any of those storage locations and the write is otherwise valid, the later write can modify that storage. Such a modification does not cause this invocation to repeat the earlier reads.

These are logical accesses established by the static representation. They do not specify physical bus transactions, atomicity, memory barriers, or guarantees under concurrent modification.

## Side effects

For an accepted selection that completes normally, the function performs one explicit logical 32-bit memory write: the result write to the output destination.

For an unsupported selection, it performs no explicit memory write. It still reads both the external table-base datum and the indexed address slot.

No allocation, release, ownership transfer, logging, registration, or external function invocation is shown. The function neither establishes ownership of the table and records nor takes ownership of the output storage.

There is no separately targeted write to the table-base datum, table slot, or record fields. However, any of these locations can be modified by the output write if the destination aliases it. Claims that those locations remain unchanged therefore require a non-overlap condition.

No synchronization is shown. In particular, the function does not establish a coherent snapshot across the base read, slot read, and field read. Device-memory effects, concurrent-access legality, and fault handling remain dependent on the surrounding environment.

## Dependencies

No called-function dependency is shown. No additional opaque function IDs are needed to describe the established behavior.

The function depends on external state and platform behavior:

| Dependency | Required role |
|---|---|
| External table-base datum | Supplies the base value through a read on every path. |
| Indexed table slot | Supplies a 32-bit record address through a read on every path. |
| Selected record field | Supplies the value for an accepted selection. |
| Caller-provided output storage | Receives one 32-bit result for an accepted selection. |
| Execution environment | Defines address interpretation, access validity, alignment requirements, and memory-failure behavior. |

Initialization, allocation extent, lifetime management, and synchronization of the external datum, table, and records are not shown. None may be attributed to this function.

The external table-base datum must not be treated as a fixed address constant or assumed to retain the same value between invocations.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless otherwise stated, all required reads, arithmetic, and writes are valid and complete normally.

| Conditions | Expected behavior |
|---|---|
| Selection 0; the selected field contains `0xFFFFFFFF` | Output becomes `0x07FFFFFF`; return is zero. |
| Selection 0; the selected field contains `0xF8000000` | Output becomes zero; return is zero. |
| Selection 0; the selected field contains `0x01234567` | Output becomes `0x01234567`; return is zero. |
| Selection 1; the selected field contains `0x89ABCDEF` | Output becomes `0x89ABCDEF`; return is zero. |
| Selection 2; output initially contains `0x13579BDF` | The base datum and indexed slot are read; output remains `0x13579BDF`; return is `0xFFFFFF06`. |
| Selection `0xFFFFFFFF` | The base datum and indexed slot are read; neither field nor output is accessed; return is `0xFFFFFF06`. |

Additional ordering and boundary cases:

- **Unsupported selection with an unreadable table-base datum:** A normal error return is not guaranteed. The function requires the datum read before it rejects the selection.
- **Unsupported selection with a readable base datum but an inaccessible indexed slot:** A normal error return is not guaranteed. The slot read also precedes rejection.
- **Unsupported selection with an unusable record address:** If both preliminary reads complete, the unusable address is not dereferenced, and the function returns `0xFFFFFF06`.
- **Unsupported selection with an unusable output destination:** If both preliminary reads complete, the destination is not accessed.
- **Negative index with accessible preceding storage:** For index −1, the slot is 396 bytes before the obtained base under ordinary address arithmetic. The index is not explicitly rejected.
- **Accepted selection with an unusable output destination:** No normal return or particular error code is established for a failing output write.
- **Output exactly aliases the selected field:** The field is read before being overwritten. Selection 0 replaces it with the masked result; selection 1 writes back the previously read value.
- **Output overlaps the table-base datum:** If the overlap permits the required 32-bit write, the output can modify that datum after its value has already been used. No second base read is established in the same invocation.
- **The table-base datum has different values on two invocations:** Each invocation obtains its base through its own datum read. With otherwise valid storage, the same index can consequently select different slots.

## Uncertainties

### Established static evidence

The supplied representation establishes the signed 16-bit index, unsigned 32-bit selection, external base-datum read, 396-byte slot stride, 32-bit slot read, selection rejection, two field offsets, mask, output write, and return bit patterns.

It also establishes the ordering of the required reads before the result write or unsupported-selection return.

### Inferred intent

The function appears intended to retrieve one of two per-record properties. The mask suggests that only 27 bits of the first property are exposed through this interface. It does not establish why the remaining bits are excluded or what they mean.

The unsupported-selection return appears to be an API error code. Its symbolic interpretation is not established.

### documented_only

The following remain unresolved:

- The valid application-level index range and table extent.
- Whether negative indices have a legitimate application use.
- The external table-base datum’s declared type, exact storage width, initialization, lifetime, and permitted changes.
- The full table and record layouts.
- The meanings of the two fields.
- The complete pointer model and public calling convention.
- Byte order and alignment requirements.
- Address arithmetic behavior at address-space boundaries.
- Memory-access failure handling.
- Atomicity and memory-ordering guarantees.
- Whether any storage is volatile, device-backed, or concurrently modified.
- Caller-enforced aliasing restrictions.
- Ownership and release responsibilities outside this function.
- The symbolic meaning of `0xFFFFFF06`.

These unknowns do not remove the established requirement to read the external base datum before the indexed slot, including for unsupported selections.

## Evidence

This contract is based solely on static inspection of the supplied function representation. It does not rely on the prior draft as independent evidence.

The representation supports the input widths, preliminary external-datum and slot reads, signed displacement calculation, unsigned selection handling, field offsets, masking rule, result write, and returned bit patterns described above.

No external datum declaration, caller, allocation routine, surrounding structure definition, or dependency implementation was supplied. No runtime tests have been performed. All acceptance examples are derived expectations rather than observed executions.
