<!-- Trusted provenance: F-59dc8ee8c07f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-59dc8ee8c07f

## Purpose

F-59dc8ee8c07f constructs a grouped byte sequence from an ordered entry table and supplies the resulting buffer and a computed length to a downstream operation.

Adjacent entries with the same effective classification contribute payload to the same group. The sequence always starts with a classification-1 group, even when that group remains empty. Classification 5 stops construction without contributing a group or payload. Normal finalization appends the two bytes `80 03`.

**Established static behavior:** two source traversals, explicit byte-level headers and lengths, adjacency-based grouping, a classification-5 stop condition, and asymmetric error handling.

**Inferred intent:** grouped serialization or conversion of source records. Neither the external format nor the semantic meaning of classifications is established.

**Important limitation:** reaching the downstream operation does not establish that every byte covered by its length argument was initialized. Payload-transfer failures do not immediately stop construction, and the transfer dependency’s guarantees are unavailable.

## Interface

The function accepts six arguments and returns a 32-bit status.

| Argument | Observed representation | Role |
|---|---|---|
| Context reference | Pointer to a readable 32-bit word | Its first word is read and cached as a resource handle. The original pointer is later passed downstream. |
| Source handle | Opaque 32-bit value | Supplied to positioning, metadata, classification, and payload-transfer dependencies. |
| Entry-table address | 32-bit address representation | Base of a table containing four-byte entry values. |
| Entry count | Signed 32-bit integer | Controls both traversals. Nonpositive values cause both traversals to process no entries. |
| Mode | Signed 32-bit integer | Accepts only 0 and −1. Both produce downstream mode 0. |
| Additional argument | Opaque 32-bit value | Passed unchanged downstream. |

The context word is read **before mode validation**. An unsupported mode therefore does not protect an invalid context reference from access.

Each table entry is read as a four-byte value and passed to F-e40a3ceddab2. This function does not establish whether that value denotes an offset, address, identifier, or another source-specific quantity. Table values are not decoded using the output format’s explicit little-endian rules.

There is no direct buffer or length output parameter.

### Return contract

| Path | Returned value |
|---|---|
| Unsupported mode | 1 |
| First-pass positioning failure | The nonzero status returned by F-e40a3ceddab2 |
| First-pass metadata failure | The checked nonzero local status |
| Allocation reports failure | The local status after F-3ce007b692cc |
| Second-pass positioning failure | The local status **after** F-bf7242c98d06 executes |
| Second-pass metadata or classification failure | The checked nonzero local status |
| Downstream operation reached | The return value of F-08ec95defbd3 |

The second-pass positioning failure does **not** guarantee return of the original positioning error. The status storage’s address has already escaped to dependencies. F-bf7242c98d06 runs before that storage is read for return, and no dependency contract establishes preservation across that call.

A payload-transfer error is stored but does not immediately cause a return. Subsequent operations can overwrite it.

## Behavior

### Validation and cached state

The local status starts at 1. The first context word is then read and retained as the resource handle used for allocation and the explicit cleanup call.

Mode −1 is normalized to 0. Mode 0 remains unchanged. Other values return 1 without table traversal or dependency calls.

The cached resource handle is not reloaded from the context before allocation or cleanup. The original context pointer, rather than a pointer to the cached word, is passed downstream.

### Sizing traversal

For a positive count, entries are visited in increasing index order at four-byte spacing.

For each visited entry, F-e40a3ceddab2 receives the source handle and the entry value. A nonzero return ends the function immediately. If positioning succeeds, F-10d2ce969adf receives the source handle and the address of the local status. Its result is treated as a signed 32-bit metadata quantity. A nonzero status ends the function immediately.

Each successful entry contributes its metadata quantity plus 6 to the sizing total. No classification query or payload transfer occurs during this traversal.

Consequently, classification 5 cannot shorten sizing: even entries after a future construction stop are visited during a successful sizing traversal.

The allocation request is the sizing total plus 2. F-3ce007b692cc receives that request, the cached resource handle, and the status address.

Construction depends on the reported status being zero. There is no independent null-pointer or capacity check on the returned buffer.

For a nonpositive count, the sizing total remains zero and the requested size is 2. The status still holds its initial value, 1, when allocation is called. For a positive count with a successful sizing traversal, the last checked status is zero. Whether the allocator interprets the incoming status value is unknown.

### Initial group and output layout

After allocation reports zero status, six bytes are written in ascending offset order:

| Buffer-relative offset | Value |
|---|---|
| 0 | `80` |
| 1 | `01` |
| 2–5 | `00 00 00 00` |

The active classification is 1. Its accumulated payload length is zero, its length field begins at offset 2, and the current output-end offset is 6.

Every ordinary group has this layout:

| Relative position | Meaning |
|---|---|
| 0 | Marker byte `80` |
| 1 | One-byte classification |
| 2–5 | Four-byte payload length, least-significant byte first |
| 6 onward | Group payload |

Lengths count payload bytes only. They exclude the six-byte group header.

The initial group is never removed because it is empty.

### Construction traversal

Construction revisits the table from the beginning. It rereads entry values and obtains metadata again; sizing results are not reused.

For every reached entry, the dependency order is:

- F-e40a3ceddab2 for positioning.
- F-10d2ce969adf for metadata.
- F-8f49ed59b67e for classification.

Status is checked after each operation. Positioning failure takes the cleanup path described below; metadata and classification failures return without that cleanup call.

The classification result is masked to its low 16 bits. The effective classification is bits 8–15 of that masked value, yielding a value from 0 through 255. Bits above bit 15 and bits 0–7 do not affect grouping or termination.

The construction metadata quantity minus 2 is the payload length. There is no check that this result is nonnegative.

#### Matching classification

When the effective classification matches the active classification, the payload length is added to the group’s accumulated length.

F-0c21b5cad025 then receives the source handle, the buffer address at the current output end, and the payload length. Its returned status is stored. The output-end offset advances by the requested payload length regardless of that status, and construction proceeds to the next entry.

No group header is inserted.

#### Classification change

When the effective classification differs, the active group’s accumulated length is first written to its four-byte length field, from least-significant byte to most-significant byte.

If the new classification is not 5, a new six-byte group header is written at the current output end. Its marker is `80`, its classification byte is the new value, and its length bytes are initially zero.

The new group’s accumulated length becomes the current entry’s payload length. F-0c21b5cad025 is called for the payload destination following the new header. Its return is stored, and the output-end offset advances by the payload length without an immediate error branch.

Grouping is strictly adjacent. A classification that reappears after a different classification creates a separate group.

### Classification 5

Classification 5 stops construction after the preceding group’s length has been written.

The initial classification is 1, and classification 5 is never installed as an active group. Thus every reached classification-5 entry takes this stop path.

For the stop entry:

- Positioning, metadata, and classification have already occurred.
- The metadata-minus-2 calculation occurs, but its result is not used for a payload transfer.
- No classification-5 header is written.
- No payload-transfer call occurs.
- Later entries are not visited during construction.
- Finalization still occurs.

A successful sizing traversal has already processed all entries, including entries after this stop.

### Finalization and downstream call

On count exhaustion or classification-5 termination, the function writes `80` and then `03` at the current output end and the following byte.

It then writes the active group’s accumulated length into that group’s four-byte length field, least-significant byte first. This final length update occurs **after** the terminator writes.

On classification-5 termination, the active length field is therefore written twice: once before the stop decision and once after the terminator.

F-08ec95defbd3 receives, in order:

1. The original context-reference pointer.
2. The allocated buffer pointer.
3. The current output-end offset plus 2.
4. Mode 0.
5. A fixed descriptor address.
6. The unchanged additional argument.

Its return becomes the function’s return.

The third argument defines a range by computed offsets. It does not prove that every byte in that range was initialized or successfully transferred.

### Error handling and cleanup

| Failure location | Observable action |
|---|---|
| First-pass positioning or metadata | Return before allocation |
| Allocation status | Return without constructing output |
| Second-pass positioning | Call F-bf7242c98d06 with cached resource handle and buffer; then return the current local status |
| Second-pass metadata | Return immediately; no explicit cleanup call |
| Second-pass classification | Return immediately; no explicit cleanup call |
| Payload transfer | Store return status, advance bookkeeping, and continue |
| Downstream operation | Return its result; no subsequent explicit cleanup call |

F-bf7242c98d06’s return value is not used to assign the local status. Nevertheless, preservation of that status is not established: its address was previously supplied to dependencies, which may retain access. The post-cleanup return must therefore remain specified as the then-current status.

There is no visible retry, rollback, short-transfer adjustment, or validation of payload contents.

## Side effects

### Reads and dependency-visible state

The function directly reads the first context word and the reached four-byte table entries. It does not directly assign to the context word or table.

That absence of direct writes is not a preservation guarantee. Dependencies may mutate shared state, and the original context pointer is passed downstream.

Source positioning and source reads performed inside dependencies are unknown. Metadata, table contents, or source contents may differ between traversals; no consistency check is present.

### Buffer writes and exposed range

The direct write order is:

1. Initial group header.
2. At classification changes, the previous group’s length and, unless stopping, a new group header.
3. Payload-transfer calls at their designated destinations.
4. Final two-byte terminator.
5. Final active-group length.
6. Downstream call.

Payload-transfer calls are not themselves proof of completed writes, even when their return status is zero, because their contract is unavailable.

The downstream length is the current output-end offset plus 2. The allocated request can exceed that range, but can also be insufficient for the attempted writes. Neither successful allocation status nor successful finalization establishes adequate capacity.

### Ownership

The buffer is obtained through F-3ce007b692cc and passed to F-08ec95defbd3.

F-bf7242c98d06 is explicitly called only after second-pass positioning failure. Its precise release semantics are unknown. No explicit local release is visible after successful finalization or after second-pass metadata or classification failure.

Whether these paths transfer ownership, retain storage, release storage indirectly, or leak is **documented_only**.

### Arithmetic and storage preconditions

Counts, metadata quantities, sizing totals, payload lengths, and output offsets use 32-bit arithmetic in the supplied representation. Classification extraction uses a masked 32-bit result. Encoded group lengths contain four bytes derived from the accumulated 32-bit value.

There are no explicit checks for:

- Table-address multiplication or addition overflow.
- Sizing-total or allocation-size overflow.
- Metadata below 2.
- Group-length or output-offset overflow.
- Buffer capacity at any write.
- A null buffer accompanied by zero status.

Portable behavior for overflowing signed arithmetic is not established.

Safe execution requires readable input storage, valid dependency arguments, representable arithmetic, and actual writable buffer capacity for every attempted write. These conditions are not enforced locally.

## Dependencies

| Opaque ID | Observed role | Unresolved contract |
|---|---|---|
| F-e40a3ceddab2 | Positions using source handle and entry value; returns checked status | Entry meaning and source-state effects |
| F-10d2ce969adf | Returns signed 32-bit metadata; receives status address | Units, valid range, stability, retained access |
| F-8f49ed59b67e | Returns classification bits; receives status address | Bit meanings and state changes |
| F-0c21b5cad025 | Receives source handle, destination, and payload length; returns status | Transfer completeness, initialization, short transfers, negative lengths |
| F-3ce007b692cc | Receives resource handle, requested size, and status address; returns buffer | Capacity, incoming-status interpretation, ownership, retained access |
| F-bf7242c98d06 | Receives resource handle and buffer on one failure path | Release behavior and preservation of escaped status storage |
| F-08ec95defbd3 | Receives context, buffer, computed length, normalized mode, descriptor, and additional argument | Consumption, retention, release, descriptor meaning, mutations |

No dependency is assumed to preserve globals or previously exposed storage, or to own or release memory, without an independent contract.

## Acceptance examples

Byte-layout examples are conditional on representable arithmetic, valid addresses, sufficient actual capacity, compatible results across traversals, and payload transfers writing the stated bytes.

### Adjacent classification-1 entries

Metadata quantities 5 and 4 produce payload lengths 3 and 2. For payloads `11 12 13` and `21 22`, the downstream length is 13 and the bytes are:

`80 01 05 00 00 00 11 12 13 21 22 80 03`

The group length is 5. No intervening header appears.

### First entry has classification 2

For one entry with metadata 4 and payload `AA BB`, the downstream length is 16 and the bytes are:

`80 01 00 00 00 00 80 02 02 00 00 00 AA BB 80 03`

The initial empty classification-1 group remains.

The sizing request is only 12 bytes: metadata 4, plus 6, plus 2. This example therefore requires actual storage of at least 16 bytes despite that request; a buffer of exactly the requested capacity is insufficient.

### Classification returns to an earlier value

For classifications 1, 2, and 1, each with a one-byte payload, construction produces three groups in that order. The first and third payloads are not combined.

### Classification 5

For classifications 1, 5, and 2, sizing visits all three entries. Construction queries the first two classifications, transfers only the first payload, and stops before visiting the third entry. The terminator follows the first group’s payload.

### Ignored classification bits

Results `000002AA` and `FFFF02BB` have the same effective classification, 2. Their differing low bytes and upper 16 bits do not create a group boundary.

### Unsupported mode

With a readable context and mode 2, the context word is read and the function returns 1. No dependency is called.

### Positioning failure after allocation

If second-pass positioning returns a nonzero status, F-bf7242c98d06 is called before return.

If that call preserves the local status, the original positioning error is returned. If it changes the escaped status storage, the changed value is returned. The supplied evidence does not choose between those outcomes.

### Payload-transfer failure

If a transfer returns nonzero, the output-end offset still advances by the requested payload length.

If construction reaches finalization and F-08ec95defbd3 returns zero, this function returns zero. The downstream range still includes the failed transfer’s destination span; its contents are not thereby established.

### Nonpositive count

For count zero or negative, allocation is requested for 2 bytes. If allocation reports success, the function attempts an initial six-byte header, a two-byte terminator, and the final header-length update, then passes downstream length 8.

This is not a safe empty-output guarantee. Actual writable capacity must support those accesses despite the two-byte request.

## Uncertainties

- **documented_only — Descriptor:** the fixed descriptor’s contents and meaning are unavailable.
- **documented_only — Metadata meaning:** subtracting 2 establishes arithmetic, not the semantic meaning of the excluded quantity.
- **documented_only — Cross-pass consistency:** dependencies provide no supplied stability guarantees.
- **documented_only — Transfer guarantees:** neither complete initialization nor successful filling of the downstream range is established.
- **documented_only — Cleanup preservation:** the status returned after F-bf7242c98d06 may differ from the positioning failure that triggered it.
- **documented_only — Ownership and capacity:** allocation, release, retention, and downstream ownership contracts are absent.
- **documented_only — Invalid lengths and overflow:** dependency behavior for negative lengths and execution behavior outside representable arithmetic are unresolved.
- **documented_only — External mutation:** preservation of shared state and previously exposed storage is not established.
- **Inferred intent:** classification 5 appears to be a source-side stop marker; its broader format meaning remains unknown.

## Evidence

This contract is based on the supplied static function body and opaque dependency mapping. No runtime tests have been performed.

Direct evidence establishes the context read before validation, accepted modes, two traversals, allocation expression, classification masking, grouping rules, explicit byte-write order, terminator, downstream argument order, status overwrites, and asymmetric cleanup.

The post-cleanup return reads the local status after F-bf7242c98d06 executes. Earlier calls expose that status storage’s address; preservation across cleanup is not established.

The downstream length is computed from the output-end offset plus 2. The absence of an immediate branch after payload-transfer failure prevents treating that length as evidence of a fully initialized byte range.

Dependency implementations, format definitions, allocator guarantees, and runtime outcomes were not supplied. Their behavior remains documented rather than assumed.
