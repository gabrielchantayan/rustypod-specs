<!-- Trusted provenance: F-d124af1bcb5a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d124af1bcb5a

## Purpose

F-d124af1bcb5a constructs an index-like candidate for a table, resolves its columns and collation references, and conditionally generates compilation operations, inserts the candidate into a schema container, attaches it to a table chain, or abandons it.

Index creation is inferred intent, supported by diagnostic categories, formatting inputs, and record relationships. The established behavior is the supplied body’s accesses, branches, calls, and direct writes. Complete database semantics and dependency implementations are not established.

The function has no explicit return value. Outcomes are observable through diagnostics, dependency effects, record mutations, and chain changes.

## Interface

### Arguments

Pointers and ordinary integer arguments have 32-bit representations in the supplied body.

| Position | Role | Representation |
|---|---|---|
| 1 | Operation context | Pointer |
| 2 | Name or qualification input | Opaque 32-bit value |
| 3 | Additional qualification input | 32-bit value, conditionally dereferenced |
| 4 | Explicit target description; zero selects the current table | Pointer-like 32-bit value |
| 5 | Column list; zero requests synthesis | Pointer |
| 6 | Conflict-policy selector | 32-bit integer |
| 7 | First creation-text input | 32-bit value |
| 8 | Second creation-text input | 32-bit value |
| 9 | Synthesized-entry ordering value | One byte |
| 10 | Existing-index tolerance flag | 32-bit integer, tested against zero |

Argument 6 is stored in the candidate as one byte. Special chain placement instead compares its original full-width value with 5. Arguments 7 and 8 jointly control whether a creation-text formatting dependency is called; their complete interpretation is unresolved.

### Observed layouts

Offsets and strides are in bytes. Fields are 32-bit unless explicitly described as bytes.

| Record | Relevant fields |
|---|---|
| Context | Connection pointer: 0; stop/error word: 64; compilation counter: 72; current-table pointer: 384; initial-gate word: 408 |
| Connection | Database-array pointer: 8; flags: 12; allocation-failure byte: 30; default-collation pointer chain: 44; schema-loading value: 124; schema-loading mode byte: 128 |
| Database descriptor | Stride 24; database-name reference: 0; schema-related pointer: 20 |
| Table | Name: 0; column count: 4; column-array pointer: 8; index-chain head: 16; view discriminator: 24; prohibition byte: 52; second prohibition byte: 57; schema association: 76 |
| Table column | Stride 20; name: 0; collation reference: 12 |
| Column list | Count: 0; entry-array pointer: 12 |
| List entry | Stride 12; expression-related pointer: 0; column-name pointer: 4; ordering byte: 8 |

The candidate has a 48-byte header:

| Offset | Field |
|---|---|
| 0 | Name pointer |
| 4 | Column count |
| 8 | Column-number array pointer |
| 12 | Additional-array pointer |
| 16 | Table pointer |
| 20 | Schema-loading value |
| 24 | Conflict-policy byte |
| 25 | Generated-name classification byte |
| 32 | Next-index pointer |
| 36 | Schema-related pointer |
| 40 | Ordering-array pointer |
| 44 | Collation-reference array pointer |

### Preconditions and ownership

There is no comprehensive validation of pointers, counts, database indexes, or allocation arithmetic. Reached memory must be valid for its observed access; traversed chains must terminate. Safe inputs require representable size and address calculations.

Default synthesis accesses the last table column without a visible zero-column guard.

Supplied and synthesized lists both reach the final list-cleanup dependency. This establishes a call obligation, not a release or ownership-transfer contract. No unknown dependency is assumed to preserve shared state, retain memory, or release memory.

## Behavior

### Entry and target resolution

The connection pointer is read from the context and retained. Entry requires, in short-circuit order:

1. Context word 64 equals zero.
2. The retained connection’s allocation-failure byte equals zero.
3. The low byte of context word 408 equals zero.

Rejection proceeds to final cleanup.

For argument 4 equal to zero, the current-table pointer is read. Null exits. Otherwise, F-49f750c2c596 receives the connection and the table’s association at offset 76, producing a database index. The name selector remains zero.

For an explicit target:

- F-9c6e06f5ecb8 receives the context, arguments 2 and 3, and selector output storage. A negative database index exits.
- If schema-loading mode is zero, F-255803dd327e receives the context and target.
- The index becomes 1 when argument 3 is nonzero, its word at offset 4 is 0 or 1, the preceding dependency returned nonzero, and that returned record’s association matches database descriptor 1’s schema-related pointer.
- F-bc66bd0e1d7a receives temporary storage, context, database index, an index-category input, and selector. A nonzero result causes F-215d8d8ab216 to receive the temporary storage and target.
- F-3747d6ba198f receives the context, zero, and target fields at offsets 12 and 8. A null result exits.

The database-array pointer is then read, and the selected descriptor address is computed using a 24-byte stride and cached. The context stop/error word is checked again.

Caching this address does not snapshot its contents or guarantee correspondence with a later database-array pointer.

### Restrictions, naming, and collisions

Restrictions are checked in this order:

1. Nonzero table byte 52: table-specific prohibition.
2. Nonzero table field 24: view-related prohibition.
3. Nonzero table byte 57: virtual-table-related prohibition.

Each uses F-2022b0f73f3b and then exits through final cleanup.

A zero selector requests a generated name. The existing index chain is traversed, followed by F-ea27adaa2931 and F-2cb24bca4183 using automatic-index naming inputs. The traversal establishes no usable sequence number in the supplied body. The suffix is unresolved. A null generated-name pointer directly sets the allocation-failure byte to 1 and exits.

For a nonzero selector:

1. F-e7c4e7dd076d supplies the name.
2. F-10ba02f1ed27 runs before the null-name check.
3. Its nonzero result, a null name, or a nonzero F-1f4e8061d0b5 result exits.
4. Outside schema loading, F-10ba02f1ed27 runs again. If permitted, F-4e77314e8c36 checks for a table-name collision; a hit produces a diagnostic.
5. F-cf5dd1ff88c3 checks for an existing index using the name and the database-name reference read through the cached descriptor.

An existing-index hit exits without the already-exists diagnostic when argument 10 is nonzero. Otherwise it reports that diagnostic. This tolerance bypasses none of the earlier checks.

### Authorization and synthesized list

The database-name reference is read through the cached descriptor and retained before authorization.

F-0055bebe0706 first receives operation 18. Only a zero result permits a second call, using operation 3 for database index 1 and operation 1 otherwise, together with the index name, table name, and retained database-name reference. A nonzero authorization result abandons further construction.

If argument 5 is zero, temporary metadata is prepared:

- One word receives the last table column’s name reference.
- F-1cb585febd84 is invoked without explicit arguments shown.
- A neighboring word retains its previous low bit and combines it with the helper result shifted left by one bit.

The neighboring word has no visible initialization. Neither its original low bit nor the helper’s input is established. The result cannot be specified as the last column name’s length.

F-a9dcd3e71f90 receives the context, two zero inputs, and the metadata address. Its return replaces the list pointer. Null abandons construction. Otherwise, argument 9 is written to the first returned entry’s ordering byte. A one-column default list is inferred intent, not a fully established dependency result.

### Allocation and candidate partition

For each list entry whose expression-related pointer is nonzero, F-1cb585febd84 measures nested text. Each result plus one contributes to expression-storage total \(E\). Another call measures the temporary index name, producing \(L\).

With allocation-time list count \(n\), F-f053f140662a receives the connection and size:

\[
53 + 13n + L + E.
\]

The partition established by pointer arithmetic is:

| Region | Size |
|---|---:|
| Header | 48 |
| Collation references | \(4n\) |
| Column numbers | \(4n\) |
| Additional array | \(4(n+1)\) |
| Ordering bytes | \(n\) |
| Name storage | \(L+1\) |
| Expression storage | \(E\) |

No overflow check is visible. Calculations use 32-bit representations; invalid counts or unrepresentable calculations do not have a safe allocation contract.

The subsequent success check examines the connection’s allocation-failure byte, not the returned pointer. Safe dereference therefore depends on the allocator’s unresolved pointer/failure-flag relationship.

After internal pointers are established, F-unmapped-01 receives the name-storage address, temporary name pointer, and \(L+1\). These resemble copying inputs, but copying, termination, ownership, and state preservation are not independently established.

After that call, direct writes record the table pointer, a freshly read list count, the low byte of argument 6, and classification byte 1 for selector zero or 0 otherwise.

The candidate schema pointer is obtained through a fresh connection database-array pointer. The ordering mask instead uses the earlier cached descriptor address: its schema-related pointer leads to byte 88. Values below 4 select mask zero; other values select all ones.

These accesses follow intervening dependencies and need not identify the same database-array allocation or schema object. Later count reads need not equal allocation-time \(n\).

### Column and collation resolution

List entries are processed in order. For each entry, its column-name input is compared against table columns from the beginning using F-e6db2171781a. The search continues while the current position is below the table’s then-read column count and comparisons return nonzero.

A zero comparison stops the search, but does **not** alone establish selection. After the loop, the table column count is read again. If that count is less than or equal to the current position, F-2022b0f73f3b reports a missing column and the candidate is abandoned. Thus, a comparison dependency that reduces the count to or below the current position can cause failure even after returning zero.

Only after this fresh bounds check is the selected zero-based column number written.

For a null expression-related pointer, the collation reference is read from the selected column using the table’s column-array pointer read at that point. A zero reference is replaced through the connection’s default-collation pointer chain.

Otherwise, F-ea27adaa2931 formats nested text into trailing candidate storage. F-1cb585febd84 measures the formatted region; the trailing cursor advances by the result plus one. That region supplies the collation reference. Formatting bounds and termination depend on unresolved dependency contracts.

Schema-loading mode is then read again. When zero, F-039d16ce0621 receives the context, chosen reference, and 32-bit all-ones value. A zero result abandons the candidate.

After this optional dependency returns successfully, the collation reference is stored. Only then is the entry’s ordering byte read and ANDed with the previously sampled mask for storage. An earlier supplied or synthesized ordering value is not necessarily the value consumed: intervening dependencies may modify the entry.

F-0ce959bd1680 runs after the loop.

### Equivalent-index handling

This search occurs only if the table equals the context’s current-table pointer when the comparison is reached.

An existing index is considered equivalent when the reached count comparisons and per-position checks permit it:

- Column numbers match.
- Ordering bytes match.
- Collation pointers match, or F-e6db2171781a returns zero for the unequal-pointer case.

No explicit arguments are shown for that unequal-pointer invocation. Its inputs and comparison meaning remain **documented_only**; collation-name equality is not established. Counts and referenced records are not immutable snapshots across dependency calls.

For an equivalent index, the candidate is not attached. Policy handling initially distinguishes:

- Equal bytes: no direct policy update.
- Existing byte 99: replace it with the candidate byte.
- Candidate byte 99 and a different existing byte: leave the existing byte unchanged.
- Differing bytes both other than 99: issue a conflicting-policy diagnostic.

After that diagnostic returns, the existing byte is reread. If it is now 99, the candidate byte is read and assigned to it. Neither record’s preservation by the diagnostic dependency is established. The candidate then follows cleanup.

### Compilation, schema insertion, and attachment

When schema-loading mode is zero at this branch:

1. The context counter at offset 72 is directly incremented.
2. F-9b3ea8470521 obtains a compilation handle. Null abandons the candidate.
3. F-274b53956040 and F-2ea344827b50 run.
4. If arguments 7 and 8 are both nonzero, F-1983453691c7 supplies creation text; otherwise its value is zero.
5. A database-name reference is read through a fresh database-array pointer.
6. F-66c61ffd0a02 receives a catalog-insertion formatting request.
7. F-94cb53877c15 receives the temporary creation-text value.

The counter increment precedes F-9b3ea8470521. This function performs no direct rollback on the null-handle path. Persistence of the increment through that dependency or cleanup is unknown.

For an explicit target, F-2b0966dfc5bf and F-2e31112ab290 follow. F-1983453691c7 produces a name-selection value passed to F-0b0cd8d831b7, followed by F-211638bb75da. Complete formatting arguments and emitted operations are unresolved.

When schema-loading mode is nonzero instead, F-1cb585febd84 measures the candidate name. F-30162766ee7c receives candidate schema pointer plus 24, name pointer, measured length plus one, and candidate.

A nonzero low 32-bit return component directly sets the allocation-failure byte and abandons the candidate. Zero permits setting connection flag mask 0x10. For an explicit target, connection word 124 is then copied to candidate word 20.

Schema-loading mode is reread before attachment. Attachment occurs if it is now nonzero or argument 4 is zero. Otherwise the candidate is cleaned up.

Attachment normally prepends. Full-width argument 6 equal to 5 instead inserts before the first existing policy-5 entry when the head exists and is not policy 5; if none exists, it appends. An empty chain or policy-5 head uses prepending.

## Side effects

On every normally returning path, final calls occur in this order, without visible null guards:

1. F-e5380ae57ea1 receives the current supplied-or-synthesized list.
2. F-ed9979230848 receives argument 4.
3. F-94cb53877c15 receives the temporary name.

An abandoned nonnull candidate first reaches F-732402c6dc47. Table-chain attachment bypasses that call.

These are cleanup-shaped calls, not established release contracts. Successful schema-container insertion does not establish ownership transfer or make later candidate cleanup harmless.

Direct writes affect allocation-failure state, flag mask 0x10, the compilation counter, candidate fields and arrays, an existing policy byte, and chain links. No general direct rollback is visible. Dependencies may independently alter these states; final persistence is not guaranteed merely because a direct write occurred.

## Dependencies

Opaque IDs identify visible roles, not complete contracts.

| Role | Dependencies |
|---|---|
| Resolution | F-49f750c2c596, F-9c6e06f5ecb8, F-255803dd327e, F-3747d6ba198f |
| Qualification processing | F-bc66bd0e1d7a, F-215d8d8ab216 |
| Naming and collisions | F-e7c4e7dd076d, F-10ba02f1ed27, F-1f4e8061d0b5, F-4e77314e8c36, F-cf5dd1ff88c3 |
| Authorization and diagnostics | F-0055bebe0706, F-2022b0f73f3b |
| Text operations | F-1cb585febd84, F-ea27adaa2931, F-2cb24bca4183, F-1983453691c7 |
| Construction and resolution | F-a9dcd3e71f90, F-f053f140662a, F-e6db2171781a, F-039d16ce0621, F-0ce959bd1680 |
| Compilation | F-9b3ea8470521, F-274b53956040, F-2ea344827b50, F-66c61ffd0a02, F-2b0966dfc5bf, F-2e31112ab290, F-0b0cd8d831b7, F-211638bb75da |
| Schema insertion | F-30162766ee7c |
| Cleanup-shaped operations | F-732402c6dc47, F-e5380ae57ea1, F-ed9979230848, F-94cb53877c15 |
| Unmapped external operation | F-unmapped-01 |

F-unmapped-01 is a document-local opaque ID without a supplied mapping.

## Acceptance examples

These are conditional static expectations, not executed tests. Reached memory is valid, arithmetic is representable, and dependencies return normally unless otherwise specified.

1. **Entry rejection:** A nonzero context word 64 prevents target resolution. The three final cleanup calls still occur in order.

2. **Restriction precedence:** With table byte 52 and the view discriminator both nonzero, the table-specific diagnostic is reached, not the view diagnostic.

3. **Existing-name tolerance:** An existing-index hit with nonzero argument 10 omits the already-exists diagnostic. An earlier table-name collision is not bypassed.

4. **Unresolved synthesis:** A null list causes last-column metadata preparation. The preserved metadata bit and argumentless helper input remain unresolved; no fully determined synthesized entry follows from this body alone.

5. **Allocation:** For \(n=2\), \(L=7\), and \(E=0\), the requested size is 86 bytes: 48 header, 8 collation, 8 column-number, 12 additional-array, 2 ordering, and 8 name bytes.

6. **Ordering read timing:** A sampled schema byte of 3 produces mask zero, so any subsequently read ordering byte stores as zero. A sampled byte of 4 stores 1 if the entry’s ordering byte is 1 **when read after** any F-039d16ce0621 call. Merely supplying 1 earlier is insufficient without a preservation condition. A replacement database array does not redirect the earlier cached descriptor address used for mask selection.

7. **Fresh column-count rejection:** At position 2, F-e6db2171781a returns zero but changes the table count to 2. The subsequent count check reports a missing column; no column-number write for that entry is reached.

8. **Post-diagnostic policy change:** Initial policies 2 and 3 trigger the conflict diagnostic. If it leaves the existing policy at 99 and candidate policy at 3, the subsequent direct write stores 3. If the existing policy remains 2, that write is skipped.

9. **Width distinction:** Argument 6 equal to 261 stores byte 5 but does not select special placement. Full-width 5 inserts before the first policy-5 entry in a stable sequence 2, 3, 5.

10. **Failure after a direct write:** The counter increment occurs before F-9b3ea8470521. A null handle causes candidate cleanup without a direct counter rollback; the eventual counter value is unknown. A nonzero low schema-insertion result directly sets the allocation-failure byte before candidate cleanup, without guaranteeing that later dependencies preserve it.

## Uncertainties

The following remain **documented_only**:

- Generated suffixes, complete formatting inputs, and exact emitted text.
- Default-synthesis helper input and the metadata word’s prior low bit.
- Unequal-collation comparison arguments and semantics.
- F-unmapped-01 effects, including whether bytes are copied.
- Allocator initialization and pointer/failure-flag correspondence.
- Additional-array initialization and meaning.
- Dependency ownership, release behavior, shared-state mutations, and abnormal exits.
- Valid selector, policy, and count ranges beyond the observed tests.
- Unsafe-input overflow behavior and complete schema-insertion return conventions.
- Transaction guarantees and persistence of direct mutations through dependencies.

Cached addresses, retained values, and fresh reads are distinct. No dependency state-preservation guarantee is supplied.

## Evidence

Established static evidence consists of the supplied body’s access widths, offsets, arithmetic, call order, branch conditions, direct writes, and cleanup routing, interpreted using the supplied opaque dependency mapping.

Database-oriented purpose and dependency role descriptions are inferred intent where definitions are absent. Prior prose and reviewer feedback do not independently establish behavior; unresolved external semantics remain unresolved.

No runtime tests have been performed.
