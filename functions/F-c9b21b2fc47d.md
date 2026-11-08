<!-- Trusted provenance: F-c9b21b2fc47d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c9b21b2fc47d

## Purpose

F-c9b21b2fc47d prepares a destination object using a primary record collection, optionally processes indices selected by a secondary collection, and compares two destination fields. If those fields differ, it calls F-4457a9a44a68 with a fixed static argument, then processes every index below a destination bound read after that call.

Construction is staged, not atomic. Each resource-request result is published into the destination before its associated status is checked. A failure can therefore leave a partially updated destination, with no explicit rollback or release.

**Inferred intent:** The secondary collection appears to select indices for processing, and the final pass appears intended to complete processing. F-4457a9a44a68 may serve a diagnostic purpose. Neither diagnostic output, complete coverage, nor equality of the destination fields after processing is established by this function.

## Interface

### Arguments

The function accepts five arguments:

1. **Destination address:** An address of writable storage containing 32-bit fields. Successful initialization accesses fields through byte offset 28. Attaching a non-null secondary descriptor additionally writes the field at byte offset 32.
2. **Primary descriptor address:** An address of readable storage with:
   - Byte offset 0: unsigned 32-bit primary record count.
   - Byte offset 8: 32-bit address of primary records.
3. **Optional secondary descriptor address:** Null disables secondary traversal. A non-null descriptor contains:
   - Byte offset 0: signed 32-bit secondary record count.
   - Byte offset 8: 32-bit address of secondary records.
4. **Initial status:** A 32-bit value used to seed a local status cell.
5. **Dependency context:** An opaque 32-bit value passed unchanged as the first argument of each call to F-6dd003795db7.

The observed address-bearing fields and address calculations belong to a 32-bit address model. A wider-pointer interface is not established.

Neither descriptor’s field at byte offset 4 is directly read by this function.

### Result

The return value is 64 bits:

- Bits 63–32 are zero.
- Bits 31–0 contain the final 32-bit contents of the local status cell.

Each resource-request continuation gate tests whether that cell is zero. The initial status does not prevent the first request: the dependency is called before the first status check and may replace the initial value.

The later index-processing and fixed-argument calls provide no return value used by this function. Their effects are not converted into a separate status.

### Destination fields

| Byte offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Receives the captured primary count; later participates in comparison and supplies the final-pass bound |
| 4 | 32 bits | Initialized to zero; later compared with offset 0 |
| 8 | 32 bits | Receives the second resource-request result; later supplies the destination-record address |
| 12 | 32 bits | Receives the first resource-request result |
| 16 | 32 bits | Receives the current field at offset 12 plus four times the captured primary count |
| 20 | 32 bits | Initialized to zero |
| 24 | 32 bits | Receives the third resource-request result |
| 28 | 32 bits | Initialized to zero |
| 32 | 32 bits | Receives a non-null secondary descriptor address |

These roles do not establish semantic field names, allocation ownership, or the internal layouts of the first and third returned regions.

## Behavior

### 1. Capture the primary count and request resources

The first primary-descriptor access reads its unsigned 32-bit count. Call the captured value **N**. Later changes to that descriptor field do not change the request quantities or the number of primary records copied.

The local status cell is seeded with the fourth argument. Up to three calls to F-6dd003795db7 follow:

| Request | Arguments, in order | Destination field receiving result |
|---|---|---|
| First | context, 4, 0, 2N, 0, address of status cell | Offset 12 |
| Second | context, 28, 0, N, 0, address of status cell | Offset 8 |
| Third | context, 16, 0, 2N + 1, 0, address of status cell | Offset 24 |

The expressions 2N and 2N + 1 use unsigned 32-bit arithmetic and wrap modulo \(2^{32}\). Overflow is not checked.

For each request, the order is:

1. Call the dependency.
2. Store its 32-bit result into the designated destination field.
3. Inspect the local status cell.
4. Continue only if status is zero.

A nonzero status after the first request prevents the second and third requests. A nonzero status after the second prevents the third. A nonzero status after the third prevents all metadata initialization, primary copying, secondary traversal, and final comparison.

These exits do not undo earlier stores or explicitly release earlier results. A result is published even when its associated status is nonzero.

The arguments resemble resource dimensions, but their meaning is not supplied. Zero status alone does not establish that a returned address is usable, sufficiently large, or initialized.

### 2. Initialize destination metadata

If all three status gates pass, the function performs these writes in order:

1. Store N at destination offset 0.
2. Read the current destination field at offset 12 and store that value plus 4N at offset 16.
3. Store zero at offset 4.
4. Store zero at offset 20.
5. Store zero at offset 28.

The multiplication by four and the addition for offset 16 use 32-bit arithmetic and can wrap. The value at offset 12 is reread after the resource calls; the calculation does not necessarily use the original first return value if an intervening dependency changed that field.

Next, the function reads the destination-record address from destination offset 8, then the source-record address from primary-descriptor offset 8. Both reads occur even when N is zero.

### 3. Transfer primary records

Primary records have a 12-byte stride. Destination records have a 28-byte stride.

For each of the N records, in increasing record order, the function performs these transfers:

| Source record offset | Destination record offset | Width |
|---|---|---:|
| 0 | 0 | 32 bits |
| 4 | 4 | 32 bits |
| 8 | 16 | 32 bits |

Each source word is read and written before the next listed transfer. Thus, the first destination write precedes the second source-word read, and the second destination write precedes the third source-word read.

After each record, traversal advances by 12 source bytes and 28 destination bytes. No record-body access occurs when N is zero.

Destination-record offsets 8, 12, 20, and 24 are not directly initialized by this function. Their contents depend on prior storage contents or dependency behavior.

The transfer is not established as overlap-safe. If a destination write overlaps a source location read later, that later read can observe the changed value. The captured source and destination traversal addresses are not reread from their descriptor fields between records.

### 4. Optionally attach and traverse the secondary collection

A null third argument skips secondary-descriptor reads and secondary traversal. This function does not clear destination offset 32 on that path.

For a non-null third argument, the order is:

1. Read the secondary-record address from descriptor offset 8.
2. Read the signed 32-bit record count from descriptor offset 0.
3. Store the secondary descriptor address at destination offset 32.
4. Traverse secondary records while the working record count is nonzero.

The descriptor is therefore attached even when its count is zero. Both descriptor fields are still read in that case.

Secondary records have a 16-byte stride. The fields directly used are:

| Record offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Unsigned number of bits to inspect |
| 8 | 32 bits | Address of packed bit bytes |

For each record, the bit-data address is read before the bit count. Both values are captured before scanning that record. Its other words are not directly read.

The bit index starts at zero for every secondary record. Bits are inspected in increasing index order, most significant bit first within each byte:

- Index 0 uses mask 0x80 in the first byte.
- Index 7 uses mask 0x01 in the first byte.
- Index 8 uses mask 0x80 in the second byte.

A byte is fetched only when its first relevant bit is reached. The fetched byte is retained while its bits are examined. A normally completed scan of B bits performs \(\lceil B/8\rceil\) byte reads, with the ceiling interpreted mathematically rather than as a potentially overflowing 32-bit expression.

For B = 0, the record’s bit-data address and bit count are read, but no byte is fetched through that address.

Each set bit causes F-621afaaa3bec to be called with:

1. The destination address.
2. The current unsigned 32-bit bit index.

Clear bits cause no call. The function does not check whether the index is less than N and does not deduplicate selections across records.

Dependency mutation matters at fetch boundaries. A call cannot change the remaining decisions derived from an already-cached byte, but it can change bytes not yet fetched. It can also change later secondary-record fields before those fields are read. The current record’s captured bit count and bit-data traversal address are not reread after each call.

After each secondary record, traversal advances by 16 bytes and the signed working record count is decremented. Zero suppresses traversal. Negative counts are not rejected and must not be interpreted as an empty collection. Exact behavior across signed decrement overflow is **documented_only**.

### 5. Compare destination fields and conditionally process a range

After secondary processing, or immediately after primary copying if no secondary descriptor was supplied, the function compares the current destination fields at offsets 4 and 0.

This comparison does not use the original N as a substitute for the current field at offset 0.

If the fields are equal, neither F-4457a9a44a68 nor the final processing pass is invoked.

If they differ:

1. Call F-4457a9a44a68 with one fixed static argument.
2. After that call returns, read destination offset 0 as an unsigned 32-bit bound, **M**.
3. Call F-621afaaa3bec with the destination address and each index from 0 through M − 1, in increasing order.

For M = 0, the fixed-argument call still occurs, but the final pass makes no index-processing calls.

The bound is captured after F-4457a9a44a68 returns. It can therefore reflect changes made by that dependency. Subsequent changes to destination offset 0 do not change the captured final-pass bound.

The final pass does not skip previously selected indices, stop when the compared fields become equal, or recheck equality after completion.

## Side effects

Direct writes comprise the destination fields and destination-record words described above. F-6dd003795db7 additionally receives access to a writable local status cell.

The function stores the secondary descriptor’s address, not a copy of the descriptor or its bit data. Any requirement to keep that storage alive after return is **documented_only**.

There is no explicit release of any returned resource, including results obtained before a later request fails. Ownership transfer, cleanup responsibility, and eventual release are **documented_only**.

The function performs no direct writes to source descriptors, source records, or bit bytes. Aliasing with destination storage, or dependency mutation, can nevertheless change data read later.

No explicit null-address, capacity, bounds, overflow, alignment, or consistency validation is visible. Safe execution requires valid storage for every reached access and dependency results compatible with subsequent accesses. No recoverable error mechanism for malformed storage is established.

No dependency is assumed to preserve globals, destination fields, source data, unread bit bytes, or memory ownership. Only explicitly captured caller values remain independent of later rereads.

## Dependencies

### F-6dd003795db7

Receives the six arguments listed in the request table, including the local status-cell address, and returns a 32-bit value.

Established caller behavior is publication of the result followed by a zero/nonzero status gate. Resource semantics, capacity, initialization, status-setting guarantees, and ownership are **documented_only**.

### F-621afaaa3bec

Receives the destination address and an unsigned 32-bit index. Its return value is ignored.

It is called for selected secondary bits and, conditionally, for the final captured range. Its mutations, supported indices, duplicate-call behavior, and failure mechanism are **documented_only**.

### F-4457a9a44a68

Receives one fixed static argument when the destination comparison differs. Its return value is ignored.

The static argument’s contents and interpretation, any output, and all other effects are **documented_only**. Describing it as diagnostic is **inferred intent**, not an established output guarantee.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. They require valid storage and normally returning dependencies. Additional preservation conditions are stated where needed.

1. **Initial status does not block the first request.** With initial status 7, the first request still occurs. If it replaces status with zero, the second request is reached.

2. **First request fails.** If the first call leaves status 9, its result is stored at destination offset 12 before return. No later request or construction stage occurs. The return value has low 32 bits equal to 9 and high 32 bits zero.

3. **Second request fails.** If the first status gate passes and the second call leaves nonzero status, destination offsets 12 and 8 have received the respective results. The third request is absent. No explicit cleanup occurs.

4. **Two primary records.** For N = 2 and three passing status gates, the request quantities are 4, 2, and 5. Assuming the relevant source words retain their intended values until read, two records are transferred to destinations 28 bytes apart, updating only record offsets 0, 4, and 16.

5. **Empty primary collection.** For N = 0 and passing status gates, all three requests occur with quantities 0, 0, and 1. Both record-address fields are read, but no primary record body is accessed. With null secondary input and no intervening mutation of the initialized comparison fields, both fields are zero and the final pass is skipped.

6. **Bit order with mutable input.** For one secondary record with captured bit count 10, suppose the first byte is 0x81 when fetched and the second byte is 0x40 when later fetched. The selected-index calls are 0, 7, and 9, in that order. Changes to the first byte after its fetch do not alter its remaining decisions. Merely giving the second byte an initial value of 0x40 is insufficient: calls at indices 0 or 7 could change it before its fetch.

7. **No selected-index range check.** With N = 2, if a secondary scan encounters a set bit at index 5, F-621afaaa3bec is called with index 5. Any rejection or unsafe access is dependency behavior, not caller-side validation here.

8. **Duplicate selection.** If two secondary records each present a set bit at index 3 when fetched, index 3 is passed twice. A later final pass may pass it again.

9. **Changed final bound.** If the compared fields differ and F-4457a9a44a68 changes destination offset 0 to 3 before returning, the final pass uses indices 0, 1, and 2. Later changes to that field do not extend or shorten this pass.

10. **Null secondary input.** If destination offset 32 initially holds an old address, null secondary input causes no direct write to that field. Preservation against dependency mutation is not guaranteed.

11. **Wrapped request quantity.** For N = 0x80000000, the first quantity is zero and the third is one. This establishes request arguments only, not safe completion of the subsequent large transfer.

## Uncertainties

- **documented_only:** Resource-request semantics, actual capacities, initialization, ownership, and release obligations.
- **documented_only:** Destination-field meanings beyond their explicit uses.
- **documented_only:** Index-processing mutations, admissible indices, duplicate safety, and errors.
- **documented_only:** Fixed static argument contents and F-4457a9a44a68 output or purpose.
- **documented_only:** Exact execution for negative secondary counts, especially across signed decrement overflow.
- **documented_only:** Alignment requirements, externally imposed aliasing restrictions, and storage lifetimes after return.
- **Inferred intent:** Selection-driven processing followed by a completion-oriented pass.
- **Not established:** Atomic construction, overflow protection, dependency preservation of globals, diagnostic emission, or a postcondition that the destination fields agree.

## Evidence

The contract derives solely from static examination of the supplied function body and opaque dependency mapping.

Established evidence includes explicit field widths, accesses, arithmetic, request arguments, publication-before-status order, traversal order, bit-byte caching, conditional calls, and return-value construction. Dependency internals and broader object contracts were not supplied.

No runtime tests have been performed. Acceptance examples state conditional requirements; they do not report observed execution.
