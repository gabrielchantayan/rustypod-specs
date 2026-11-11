<!-- Trusted provenance: F-bdce4cbb1669; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bdce4cbb1669

## Purpose

F-bdce4cbb1669 conditionally prepares removal of a named index, then cleans up the supplied name-list object.

The established behavior consists of eligibility checks, lookup, diagnostic dispatch or removal-preparation calls, and cleanup on ordinary paths. Preparation includes a catalog-deletion request, a conditional statistics-deletion request, schema-related calls, and an operation carrying the index name.

Compilation of an index-removal statement is inferred intent. The supplied representation does not establish whether dependencies execute persistent changes, queue operations, or provide transaction guarantees.

## Interface

The function accepts three arguments and supplies no defined return value:

| Argument | Width and requirements | Established use |
|---|---|---|
| Context | 32-bit pointer to writable storage | Supplies the database pointer and eligibility field; receives a status-byte write on missing lookup |
| Name list | Nullable 32-bit pointer | Supplies lookup fields and cleanup entries; its containing allocation enters allocator cleanup |
| Missing-name option | 32-bit value | Zero enables the missing-index diagnostic; nonzero suppresses it; also forwarded to a later check |

The context must be valid even when preparation is skipped. A null name list is safe only on paths that do not reach its lookup-field reads.

### Context and schema layout

All offsets are bytes.

| Object | Offset | Access |
|---|---:|---|
| Context | 0 | 32-bit database pointer |
| Context | 0x12 | Byte written to 1 after missing lookup |
| Context | 0x40 | 32-bit preparation eligibility field |
| Database | 8 | 32-bit schema-slot-array pointer |
| Database | 0x1e | Byte preparation eligibility field |
| Schema slot | 0 | 32-bit schema-name value |

Schema slots have stride 0x18. No schema-index bounds check is visible.

### Name-list layout

The list contains a signed 16-bit count at offset 0 and entries beginning at offset 8 with stride 0x30. Lookup reads 32-bit values at list offsets 0x0c and 8, passing them in that argument order. Their interpretation as qualification and name components is inferred.

Each cleanup iteration reads the following 32-bit entry fields immediately for their respective calls:

| Entry offset | Dependency |
|---:|---|
| 0 | F-94cb53877c15, with an additional unresolved argument |
| 4 | F-94cb53877c15 |
| 8 | F-94cb53877c15 |
| 0x0c | F-da1a289a3277 |
| 0x10 | F-6c7ac9769d41 |
| 0x1c | F-841dc33ee157 |
| 0x20 | F-11bd3a21366b |

There are no local per-field null checks.

### Resolved-index layout

| Offset | Access |
|---:|---|
| 0 | 32-bit index-name value |
| 0x10 | 32-bit pointer whose first 32-bit field is later read |
| 0x14 | 32-bit argument to F-fa17eddb09c3 |
| 0x19 | Byte controlling rejection |
| 0x24 | 32-bit argument to schema-slot resolution |

Valid extents, alignment, pointer lifetimes, allocation metadata, and metadata-derived indices are preconditions. The function provides no general recovery for malformed objects.

## Behavior

### Eligibility and lookup

The function first reads the context’s database pointer and 32-bit eligibility field. The database pointer is retained for subsequent preparation calls; it is not reloaded from the context.

A nonzero context eligibility field skips preparation. Only when that field is zero is the database eligibility byte read. A nonzero byte also skips preparation.

If both fields are zero, F-10ba02f1ed27 receives the context. A nonzero result skips preparation. A zero result permits F-cf5dd1ff88c3 to receive the captured database pointer and the two name-list fields described above.

All ordinary skip paths continue to cleanup. Skipping preparation does not imply that cleanup is unnecessary.

### Missing or rejected index

If lookup returns zero:

1. When the option is zero, F-2022b0f73f3b receives the context and a missing-index diagnostic template. No separate substitution argument is visible.
2. After that call returns, or immediately when the option is nonzero, context byte 0x12 is written to 1.
3. Cleanup follows.

Suppressing the diagnostic does not suppress the status write.

If lookup returns a nonzero result whose byte at offset 0x19 is nonzero, F-2022b0f73f3b receives the context, an opaque diagnostic datum, and zero. Cleanup follows without schema resolution. A protected or automatically managed index is an inferred interpretation of this rejection condition, not an established category.

### Schema resolution and continuation checks

For a result whose rejection byte is zero:

1. F-49f750c2c596 receives the captured database pointer and index field 0x24.
2. Its 32-bit result becomes the schema-slot index.
3. The database’s schema-array pointer and the selected slot’s first 32-bit field are read.
4. The index pointer at offset 0x10 is read and retained.
5. F-0055bebe0706 receives the context and selector 9.

A nonzero selector-9 result stops preparation. The already-read schema-name value remains distinct from later schema-name reads.

If selector 9 returns zero, F-0055bebe0706 receives:

- The context.
- Selector 10 for schema index zero, otherwise selector 12.
- The then-read index-name field.
- The first 32-bit field reached through the previously captured offset-0x10 pointer.
- The previously captured schema-name value.
- The original missing-name option.

Thus the pointed-to object’s first field is read after the selector-9 call, even though its pointer was obtained before that call. No preservation of that object by the dependency is assumed.

A nonzero second-check result stops preparation. Otherwise F-9b3ea8470521 receives the context. A zero result stops preparation; a nonzero result is retained as the operation-builder value.

### Removal preparation

With a nonzero operation-builder value, calls occur in this order:

1. **F-274b53956040:** context, value 1, schema index.
2. **F-66c61ffd0a02:** context, catalog-deletion template, freshly read schema-name value, catalog name, and current index-name value. Schema index 1 selects `sqlite_temp_master`; every other index selects `sqlite_master`.
3. **F-4e77314e8c36:** captured database pointer, `sqlite_stat1`, and another freshly read schema-name value.
4. **Conditional F-66c61ffd0a02:** context and statistics-deletion template, only if F-4e77314e8c36 returned nonzero.
5. **F-2e31112ab290:** context and schema index.
6. **F-fa17eddb09c3:** context, current index field 0x14, and schema index.
7. **F-0b0cd8d831b7:** operation-builder value, selector 0x3a, schema index, two zeros, current index-name value, and a final zero.

The statistics-lookup result is explicitly tested. Other call results in this sequence are not used to control continuation.

The catalog-deletion template targets the catalog’s `name` field. Exact formatting and execution semantics belong to the dependency. The statistics-deletion call has no additional explicit substitution arguments in the supplied representation; its complete predicate is unresolved.

There is no visible local rollback or compensating path.

### Entry cleanup and condition-time global reads

A null name list bypasses all list cleanup.

For a non-null list, traversal begins at offset 8. Each loop-condition evaluation first reads an **accounting-pointer global**, then compares a signed 32-bit iteration index with the freshly read signed 16-bit count.

This global-pointer read occurs:

- Before the first iteration.
- Before every subsequent iteration.
- On the terminating evaluation.
- On the first evaluation even when the count is zero or negative.

The pointer sampled by the terminating evaluation is retained for a later high-word accounting write. It is not necessarily the same pointer subsequently obtained from that global.

For each admitted iteration, the seven cleanup calls occur in the interface-table order. The entry address then advances by 0x30, and the iteration index increases by one.

The count is not captured once. Cleanup dependencies that change it can extend or shorten traversal. With an unchanged count, zero and negative counts admit no iterations.

The first F-94cb53877c15 call has an additional argument. On the first iteration, its represented value is:

- Initially the name-list pointer.
- The captured schema-name value after successful schema resolution.
- Zero after reaching final operation emission.

For later iterations, it is an unresolved value exposed after the preceding iteration’s final cleanup call. This argument is not established as an ordinary ownership or allocator parameter.

### Global allocation accounting

After entry cleanup, the function reads a 32-bit displacement at list offset −4 and subtracts it from the list address to obtain a derived allocation address. It then reads a 32-bit accounting quantity at derived-address offset −8.

The accounting-pointer observations have distinct purposes:

1. The pointer retained from the terminating loop condition supplies the **high-word write destination**, at offset 0x3c.
2. A fresh accounting-pointer-global load supplies the **low-word read address**, at offset 0x38.
3. Another fresh load supplies the **high-word source address**, at offset 0x3c.
4. Another fresh load supplies the **low-word write destination**, at offset 0x38.

These observations must not be treated as one stable counter pointer.

The low-word write subtracts the accounting quantity from the sampled low word, with 32-bit wrapping. The high-word source address has already been captured when this low-word write occurs; its value is then read for the high-word update.

Let \(L\) be the sampled unsigned low word, \(Q\) the accounting quantity, and \(H\) the value read through the captured high-word source address. The high-word destination receives, modulo \(2^{32}\),

\[
H-\bigl(s+b\bigr),
\]

where \(b=1\) when unsigned \(L<Q\), otherwise zero, and \(s=-1\) when bit 31 of \(Q\) is set, otherwise zero. This is not generally ordinary unsigned 64-bit subtraction.

After both writes, F-2fd8826557a4 is called. The allocator-state pointer is then read through a separate global pointer.

### Allocation validation and allocator updates

If the derived allocation address equals numeric address 8, allocator-header processing is skipped. The preceding accounting writes, F-2fd8826557a4 call, and allocator-state-pointer read have already occurred.

Otherwise, the allocation header is located at derived-address offset −16:

- A zero 32-bit header read invokes nonreturning F-a817d8d58c10.
- If that read is nonzero, a separate header read tests bit 0. A set bit invokes the same nonreturning dependency.

These observations do not establish a stable header.

When validation permits continuation:

1. F-c0fc6d033caa receives the captured allocator-state pointer.
2. After it returns, unsigned 16-bit metadata at derived offset −12 is read.
3. The header is reread and masked with 0x3ffffffc to obtain a size.
4. That size is subtracted from the selected 32-bit bucket at allocator-state offset \(0x224+4m\), where \(m\) is the first metadata value, then from the aggregate at 0x30c.
5. Only after those writes, unsigned 16-bit metadata at derived offset −10 is read.
6. The same captured size is subtracted from the selected bucket at \(0xe0+4n\), where \(n\) is the second metadata value, then from the aggregate at 0x220.
7. F-09dcb55e8e0a supplies another bucket index.
8. The selected 32-bit count at \(0x310+4k\) is decremented, then the aggregate count at 0x390 is decremented.
9. The header is read again, masked with 0x3ffffffc, and that newly observed size is subtracted from the byte counter at allocator-state offset 8.
10. F-cc6c9b4a0f68 receives the captured allocator-state pointer and header address.
11. F-e586bff64fb7 receives the captured allocator-state pointer.

The validation-time header is not the size source across F-c0fc6d033caa. The header observation after F-09dcb55e8e0a is separate again. Metadata-derived bucket indices are not visibly range-checked. Counter updates use 32-bit word arithmetic.

## Side effects

Directly established writes are:

- Context byte 0x12 on missing lookup.
- Low- and high-word accounting destinations selected by distinct global-pointer observations.
- Allocator size buckets, size aggregates, count buckets, aggregate count, and byte counter.

Dependencies may diagnose errors, mutate objects or globals, prepare operations, release memory, or terminate nonlocally. Their complete effects are unknown.

The list is treated as consumed on ordinary cleanup paths. Its nested fields and containing allocation are submitted to cleanup machinery; successful deallocation is not independently established without dependency contracts. The caller’s pointer and entry fields are not cleared.

No direct release of the resolved index is visible. Passing its name to operation emission does not establish copying, borrowing, or ownership transfer.

No locally established synchronization guarantees stable globals, headers, counts, or schema objects across observations. Dependency calls must not be assumed to preserve them.

## Dependencies

| Opaque ID | Established boundary role |
|---|---|
| F-10ba02f1ed27 | Additional eligibility gate |
| F-cf5dd1ff88c3 | Index lookup |
| F-2022b0f73f3b | Diagnostic dispatch |
| F-49f750c2c596 | Schema-index production |
| F-0055bebe0706 | Selector-based continuation checks |
| F-9b3ea8470521 | Operation-builder acquisition |
| F-274b53956040 | Initial removal-preparation call |
| F-66c61ffd0a02 | Catalog/statistics deletion requests |
| F-4e77314e8c36 | Statistics lookup whose result controls a branch |
| F-2e31112ab290 | Schema-indexed preparation |
| F-fa17eddb09c3 | Preparation using index field 0x14 |
| F-0b0cd8d831b7 | Final operation emission |
| F-94cb53877c15 | First three entry cleanup calls |
| F-da1a289a3277 | Entry-field 0x0c cleanup |
| F-6c7ac9769d41 | Entry-field 0x10 cleanup |
| F-841dc33ee157 | Entry-field 0x1c cleanup |
| F-11bd3a21366b | Entry-field 0x20 cleanup |
| F-2fd8826557a4 | Call after global accounting |
| F-c0fc6d033caa | Allocator-state call before metadata and size reads |
| F-09dcb55e8e0a | Count-bucket-index production |
| F-cc6c9b4a0f68 | Allocator call receiving header address |
| F-e586bff64fb7 | Final allocator-state call |
| F-a817d8d58c10 | Nonreturning invalid-header path |

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Dependencies are assumed to return normally except where stated, and required storage remains valid.

1. **Preparation disabled:** A nonzero context eligibility field prevents the database eligibility-byte read and lookup. A non-null list still enters cleanup.

2. **Null list on a skipped path:** With preparation disabled and a null list, no cleanup-loop condition or allocation accounting occurs. This does not establish null safety on an eligible lookup path.

3. **Missing index:** A zero lookup result with option zero causes diagnostic dispatch before the status-byte write. With a nonzero option, dispatch is omitted but the write still occurs.

4. **Rejected result:** A nonzero index byte at 0x19 causes diagnostic dispatch and prevents schema resolution and operation emission.

5. **Denied checks:** A nonzero selector-9 result prevents the second check. A nonzero second-check result prevents operation-builder acquisition.

6. **Temporary schema:** Schema index 1 selects selector 12 and `sqlite_temp_master`; the final operation receives schema index 1.

7. **Statistics branch:** A zero F-4e77314e8c36 result omits statistics deletion but not the later preparation calls. A nonzero result includes that deletion request.

8. **Nonpositive count:** A non-null list with unchanged count zero or negative causes one accounting-pointer-global read during the first, terminating condition evaluation, no entry calls, and subsequent allocation accounting.

9. **Distinct accounting pointers:** If successive global observations differ, the high-word write still targets the terminating-condition sample. Low-word read, high-word source address, and low-word destination follow their own later observations.

10. **Changed header after allocator call:** If validation succeeds but F-c0fc6d033caa changes the header, the first two size-accounting groups use the post-call masked header, not the validation value. No second validity check is visible.

11. **Later header change:** If F-09dcb55e8e0a changes the header, the final byte-counter decrement uses the later masked value, while earlier size-bucket decrements retain their previously captured size.

12. **Invalid header:** A zero first header read, or a nonzero first read followed by an odd second read, reaches the nonreturning dependency only after global accounting and F-2fd8826557a4.

## Uncertainties

The following remain **documented_only**:

- Dependency contracts, including global preservation, ownership, null tolerance, locking, and nonlocal exits.
- Exact opaque diagnostic data and complete statistics-deletion predicate.
- Meaning and later-iteration value of the extra cleanup argument.
- Whether operation emission copies or retains the index name.
- Whether preparation calls execute persistent changes immediately.
- Meaning of the derived-address-8 special case.
- Valid schema and allocator-bucket index ranges.
- Environmental guarantees preventing inconsistent global-pointer or header observations.

Source expression structure does not establish a total evaluation order among all argument reads within a single call. The contract preserves explicit sequencing without inventing finer ordering.

## Evidence

This contract derives from static inspection of the supplied function representation and opaque dependency mapping. Prior prose and reviewer feedback were not treated as independent evidence.

Established evidence comprises access widths and offsets, branch conditions, visible arguments, explicit call sequencing, repeated pointer and header observations, direct writes, and arithmetic. Index-removal intent and broader dependency semantics remain interpretations or documented unknowns.

No dependency implementations or external contracts were supplied. No runtime tests have been performed.
