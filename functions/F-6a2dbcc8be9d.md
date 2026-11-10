<!-- Trusted provenance: F-6a2dbcc8be9d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6a2dbcc8be9d

## Purpose

F-6a2dbcc8be9d performs an ordered transfer of selected fields from a source object to a destination object. Most fields are delegated directly to F-12d234fad301. A final field, when selected, is transferred through a two-stage export followed by an import.

The low bit of the options argument controls selection:

- **Low bit clear:** process every listed field, including fields whose source value is zero.
- **Low bit set:** process only fields whose source value is nonzero.

An auxiliary source flag independently controls whether processing continues beyond the first group of fields.

This is a contract for the behavior visible in the supplied static representation. It does not establish what the transferred fields mean, whether a transfer copies or consumes resources, or whether delegated operations succeed when their return values are ignored.

## Interface

### Arguments

| Argument | Representation | Role |
|---|---|---|
| Source | Address of a record containing 32-bit words | Supplies a base address, an auxiliary-record address, and field values. |
| Destination | Address of a record containing 32-bit words | Supplies a base address and writable destinations for delegated operations. |
| Options | Unsigned 32-bit integer | Only bit 0 affects this function’s own control flow. |

The function returns an unsigned 32-bit value. Depending on the path, that value is a source flag byte, zero, a dependency’s status, or the explicit value `0xFFFFFF94`.

### Record layout used here

Offsets are byte offsets from the corresponding record address. A “word” below is four bytes.

| Record offset | Observed use |
|---|---|
| `0x00` | Base address used to form dependency context addresses. |
| `0x10` | Address of an auxiliary record. |
| `0x28` through `0x50` | Eleven primary field words, processed in the order specified below. |

The source auxiliary record is accessed as follows:

| Auxiliary offset | Observed use |
|---|---|
| `0x00` | One-byte flag; bit 0 enables auxiliary processing. |
| `0x04` | Field word. |
| `0x08` | Field word processed before the word at `0x04`. |
| `0x0C` | Field word. |
| `0x10` | Final field word, transferred through export/import. |

The destination auxiliary record provides corresponding field locations at offsets `0x04`, `0x08`, `0x0C`, and `0x10`. This function does not directly copy the auxiliary flag byte.

The field words are passed as 32-bit quantities. Their interpretation as handles, pointers, identifiers, or scalar values is not established.

### Preconditions

For normal completion, all memory actually accessed must be valid with the access widths described here. Destination field locations passed to dependencies must satisfy those dependencies’ requirements.

The initial base-address reads tolerate a null source or destination by substituting a zero base. That is **not** a general null-record contract:

- The source record is subsequently dereferenced on every normal path.
- The source auxiliary address is dereferenced after primary processing.
- Destination field addresses or destination auxiliary contents are used when selected operations require them.
- No validation or structured error is provided for invalid record addresses.

No non-aliasing requirement is established by the visible implementation. However, source/destination aliasing can make dependency mutations affect subsequent selections and values.

## Behavior

### 1. Capture the two base addresses

The function first obtains the source base word from source offset `0x00`, or uses zero if the source argument is null. It then obtains the destination base word from destination offset `0x00`, or uses zero if the destination argument is null.

These two base values remain the basis of all later context-address calculations. A dependency changing either record’s word at offset `0x00` does not cause the base to be refreshed.

Other record fields and auxiliary addresses are not generally snapshotted at entry.

### 2. Process the primary fields

For each selected row below, F-12d234fad301 receives:

1. Captured source base plus the context offset.
2. The source field’s current 32-bit value.
3. Captured destination base plus the same context offset.
4. The address of the corresponding destination field word.

Calls occur in this order:

| Order | Source and destination record offset | Context offset |
|---:|---:|---:|
| 1 | `0x28` | `0x0C0` |
| 2 | `0x2C` | `0x118` |
| 3 | `0x3C` | `0x118` |
| 4 | `0x30` | `0x170` |
| 5 | `0x34` | `0x1C8` |
| 6 | `0x38` | `0x220` |
| 7 | `0x40` | `0x278` |
| 8 | `0x44` | `0x2D0` |
| 9 | `0x48` | `0x328` |
| 10 | `0x4C` | `0x380` |
| 11 | `0x50` | `0x3D8` |

The third row deliberately precedes the row for record offset `0x30`. The second and third rows deliberately share context offset `0x118`.

With options bit 0 clear, every row is called without a preceding nonzero selection test. With bit 0 set, each row is called only if its source field is nonzero when tested.

For a selectively processed row, the selection read and the value supplied to the call are separate visible accesses. The source value is not established as an entry-time snapshot.

All return values from these calls are ignored. A returned failure status does not stop this function from considering later rows.

### 3. Inspect the auxiliary flag

After primary processing, the function reads the byte at offset `0x00` of the current source auxiliary record.

If bit 0 is clear, auxiliary processing is skipped. The function reads the flag byte again and returns that second byte value, zero-extended to 32 bits. Consequently, a stable flag byte of `0x02` produces return value `2`, not zero.

If bit 0 is set, processing continues with a call to F-c4407f9efc54, passing the destination record address.

This call occurs even if selective processing will find every auxiliary field zero. Its return value does not control any subsequent decision in this function. Its mutations, if any, can still affect later reads.

### 4. Process three auxiliary fields

The first three auxiliary fields are delegated to F-12d234fad301 in this order:

| Order | Auxiliary field offset | Context offset |
|---:|---:|---:|
| 1 | `0x08` | `0x7A0` |
| 2 | `0x04` | `0x7F8` |
| 3 | `0x0C` | `0x850` |

Each selected call receives:

1. Captured source base plus the context offset.
2. The current source auxiliary field word.
3. Captured destination base plus the context offset.
4. The address of that field within the current destination auxiliary record.

With options bit 0 clear, all three calls occur, including for zero-valued source fields.

With bit 0 set, each call is conditional on its source auxiliary field being nonzero. These tests take place after F-c4407f9efc54 and after any preceding field calls. The auxiliary address and field values must therefore not be treated as immutable entry snapshots.

Return values from these three calls are also ignored.

### 5. Decide whether to transfer the final field

The final field is at source auxiliary offset `0x10`.

- With options bit 0 set, a zero value causes an immediate return of zero.
- With options bit 0 clear, the final transfer is attempted even when this field is zero.

When proceeding, the function captures the current destination auxiliary address and the current source final-field word before calling the export dependency. Those captured values are reused throughout the final transfer.

Thus, a dependency subsequently replacing the destination record’s auxiliary address does not redirect the eventual import: its output location is based on the previously captured auxiliary address.

### 6. Query the export size

F-5ef10d079522 is called with:

1. Captured source base plus `0x8A8`.
2. Captured source final-field word.
3. A null output-buffer address.
4. An address for a 32-bit count.
5. Maximum count `0x7FFFFFFF`.

A nonzero returned status is returned immediately and unchanged as a 32-bit value. No buffer acquisition, second export, or import then occurs.

If the status is zero, the function relies on the dependency having supplied the count. It does not initialize that count before the query.

### 7. Select storage and export

The queried count is compared as an unsigned 32-bit quantity:

- Counts from `0` through `512` use a local 512-byte buffer.
- Counts of `513` or greater cause a call to F-f195b74c9758.

No explicit arguments to F-f195b74c9758 are visible in the supplied representation. In particular, it does not establish that the queried count is passed to this dependency.

If the chosen buffer address is null, the function returns `0xFFFFFF94`. This bit pattern corresponds to signed 32-bit `−108`, but the declared return representation is unsigned.

Otherwise, F-5ef10d079522 is called again with:

1. The same source context address.
2. The same captured final-field word.
3. The selected output buffer.
4. The same count address.
5. The count value present immediately before this call.

The second export may update the count through its address.

If this export returns nonzero, its status becomes the final status, subject to releasing nonlocal storage as described below. Import is skipped.

### 8. Import and release temporary storage

If the second export returns zero, F-7834b08494a4 is called with:

1. Captured destination base plus `0x8A8`.
2. The export buffer.
3. The count as it stands after the second export.
4. Captured destination auxiliary address plus `0x10`.

Its returned status becomes the final status.

After either second-export failure or import completion, the function calls F-1137792fba7d with the buffer address if that address differs from the local buffer address. The release dependency’s return value, if any, does not replace the selected status.

The local buffer is never passed to F-1137792fba7d.

## Side effects

The function directly reads records and temporary storage and supplies addresses through which dependencies may write destination state. It does not visibly assign destination field words itself.

Observable dependency order is:

1. Selected primary-field calls.
2. Auxiliary flag read.
3. F-c4407f9efc54, only when the flag permits auxiliary processing.
4. Selected auxiliary-field calls.
5. Optional size query.
6. Optional buffer acquisition.
7. Optional second export.
8. Optional import.
9. Conditional release.

There is no rollback. Earlier destination changes can remain when a later export or import fails. Failures reported by F-12d234fad301 do not affect this control flow.

The final imported resource’s ownership is not established. Passing temporary bytes to F-7834b08494a4 does not prove that it copies them, retains them, or assumes ownership. What is established is that this function proceeds to release nonlocal temporary storage after the import returns, including when the import reports failure.

Individual argument-expression evaluation order within a dependency call is not specified by the supplied representation. The ordered stages above should not be strengthened into an unsupported ordering among all reads used to form a single call’s arguments.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-12d234fad301 | Receives source context, source word, destination context, and destination-word address. Return ignored. | Transfer semantics, ownership, failure effects, global mutations. |
| F-c4407f9efc54 | Called once with the destination when auxiliary processing is enabled. Return does not control selection. | Whether it initializes, allocates, validates, or replaces auxiliary state. |
| F-5ef10d079522 | First queried with null buffer, then potentially called with storage and a count address. Zero permits continuation. | Encoding, buffer-safety guarantees, count guarantees, global effects. |
| F-f195b74c9758 | Supplies a buffer address on the large-count path; null triggers `0xFFFFFF94`. | Calling convention details, requested capacity, allocation mechanism. |
| F-7834b08494a4 | Receives exported bytes, post-export count, destination context, and final-field address. Status returned. | Data format, retention, ownership, partial-write behavior. |
| F-1137792fba7d | Called on the nonlocal buffer after the second export/import path. | Actual release semantics, allocator pairing, global effects. |

No dependency is assumed to preserve globals, preserve aliased records, or leave unrelated fields untouched.

## Acceptance examples

These examples are conditional on valid memory and dependencies returning normally with the stated behavior.

1. **All primary fields, auxiliary processing disabled.**  
   Options bit 0 is clear and the source flag remains `0x02`. All eleven primary calls occur in table order, even for zero-valued fields. F-c4407f9efc54 is not called. The return value is `2`.

2. **Selective primary processing.**  
   Options bit 0 is set, only primary offsets `0x2C` and `0x3C` are nonzero, and dependencies do not change later source fields. Exactly those two primary calls occur, in that order, both using context offset `0x118`.

3. **Enabled but empty selective auxiliary processing.**  
   The flag’s low bit is set, options bit 0 is set, and all four auxiliary field words remain zero. F-c4407f9efc54 is called. No auxiliary transfer or export call occurs, and the function returns zero.

4. **A 512-byte export.**  
   The query succeeds with count `512`. The second export receives local storage and maximum count `512`. If it succeeds and leaves count `500`, import receives count `500`. No buffer-acquisition or release dependency is called.

5. **A 513-byte export with failed acquisition.**  
   The query succeeds with count `513`, and F-f195b74c9758 returns null. The function returns `0xFFFFFF94`; second export and import do not occur.

6. **Second-export failure using nonlocal storage.**  
   Acquisition succeeds, but the second export returns nonzero status `S`. Import is skipped, F-1137792fba7d receives the buffer, and the function returns `S`.

7. **Import failure.**  
   Both export calls succeed and import returns nonzero status `T`. The function returns `T`, releasing nonlocal temporary storage first. Prior destination changes are not rolled back.

## Uncertainties

The following are **documented_only**:

- Semantic field names, resource types, and serialization format.
- Dependency effects on global state, source state, destination state, and ownership.
- The actual calling contract and capacity guarantee of F-f195b74c9758.
- Whether either export enforces its supplied maximum or always initializes the count on success.
- Whether import accepts zero-length data or retains the buffer.
- Defined behavior for invalid pointers, inadequate storage, or concurrent mutation.

The visible function does not validate that the second export’s resulting count fits the selected buffer. Safe operation therefore depends on the export dependency’s contract; buffer safety cannot be established independently here.

Addresses and field words are represented at 32-bit width. Context offsets are byte additions, not word indices. The function contains no explicit overflow checks. The supplied representation does not establish portable signed-overflow behavior or usable addresses after overflow, so valid, representable address arithmetic is a precondition rather than an assumed wraparound guarantee.

## Evidence

**Established static evidence:** the supplied representation supports the layouts, call order, selection rules, access widths, return paths, count threshold, and conditional release behavior documented above.

**Inferred intent:** “field transfer,” “export,” “import,” and “buffer acquisition/release” describe the apparent roles of the calls. They do not establish unshown dependency behavior.

**Verification limit:** no runtime tests have been performed. Dependency implementations and independent runtime observations were not supplied.
