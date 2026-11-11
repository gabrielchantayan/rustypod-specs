<!-- Trusted provenance: F-d0af131d41ba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d0af131d41ba

## Purpose

F-d0af131d41ba processes a declaration against a context’s current table. It records that a declaration has been encountered, marks selected columns, and either stores a specialized single-column representation, delegates general processing, or reports a diagnostic.

The primary-key interpretation is supported by the diagnostics and the comparison with `INTEGER`. Downstream key and row-identity semantics are not established.

A supplied declaration list follows one of two caller-side disposition paths:

- After general processing is delegated, this function performs no local cleanup of the list.
- On other returning paths, this function executes the local cleanup sequence described below.

Neither path establishes what memory an opaque dependency actually releases.

## Interface

The function takes five arguments and returns without an established result value.

| Argument | Width | Observed role |
|---|---:|---|
| Context | 32-bit address | Supplies the current table and a processing-inhibition byte. |
| Declaration list | Nullable 32-bit address | Supplies named entries; subject to delegation or local cleanup. |
| Policy | 32 bits | Passed unchanged to general processing; truncated to eight bits on specialized success. |
| Automatic-increment request | 32-bit integer | Tested in full against zero; truncated to eight bits on specialized success. |
| Additional key option | 32-bit integer | Must be zero for specialization; otherwise passed unchanged to general processing when that path is selected. |

“Policy” and “additional key option” are descriptive labels, not established semantic types.

### Object layouts

Offsets are in bytes. Pointer fields and ordinary words are 32 bits.

| Object | Offset | Accessed meaning |
|---|---:|---|
| Context | `0x180` | Current table address |
| Context | `0x198` | One-byte inhibition field |
| Table | `0x00` | Diagnostic argument |
| Table | `0x04` | Signed column count |
| Table | `0x08` | Column-array address |
| Table | `0x0c` | Specialized column index |
| Table | `0x36` | One-byte declaration-seen marker |
| Table | `0x37` | One-byte stored policy |
| Table | `0x38` | One-byte stored automatic-increment value |
| Declaration list | `0x00` | Signed entry count |
| Declaration list | `0x0c` | Entry-array address |

Columns have a stride of `0x14` bytes:

- `0x00`: name-like comparison value.
- `0x08`: nullable type-name address.
- `0x11`: one-byte membership marker.

Declaration entries have a stride of `0x0c` bytes:

- `0x00`: first cleanup resource.
- `0x04`: name-like comparison value and second cleanup resource.
- `0x08`: word not otherwise interpreted here.

Local list cleanup additionally accesses allocation metadata preceding the list and allocator state described under Side effects.

### Preconditions

Normal operation requires valid memory for every reached access, including nested arrays, allocator metadata, globals, and dependency-selected counters. Counts, extents, and address arithmetic must be compatible with those accesses. There are no general bounds, overflow, or underflow checks.

Without an explicit list, normal operation requires at least one column: membership marking occurs before candidate validation.

Dependencies must return for subsequent operations to occur, except F-a817d8d58c10, whose observed path is nonreturning. No dependency is assumed to preserve objects, globals, or resources.

## Behavior

### Eligibility and duplicate declarations

The current table address is read once from the context and retained for table processing. Later changes to the context’s table field do not redirect this invocation’s table accesses.

If the table address is zero, processing skips directly to list cleanup; the inhibition byte is not read on that branch. Otherwise, a nonzero inhibition byte likewise bypasses table processing.

With processing enabled, a nonzero declaration-seen byte causes F-2022b0f73f3b to receive the context, a multiple-primary-key diagnostic, and the table’s first word. The function then enters local list cleanup without performing name searches or its own new membership writes.

Otherwise, the declaration-seen byte is set to one before column selection. Subsequent diagnostics or delegation do not roll it back.

### Explicit-list searches and reloads

For a non-null list, the entry index begins at zero and the candidate begins at negative one. Each outer-loop condition freshly reads the signed list count.

Each processed entry starts a column search at index zero. The caller-side access sequence is:

1. Each search condition freshly reads the table’s signed column count.
2. If the column index is below that count, comparison arguments are obtained through freshly read entry-array and column-array fields. The selected words are entry offset `0x04` and column offset `0x00`.
3. F-e6db2171781a is called. A zero low 32-bit result ends the search; a nonzero result advances the column index and repeats the condition.
4. After search termination, the entry index advances. The column count is freshly read again to decide whether the resulting column index is still below it.
5. If so, the column-array field is freshly read and byte one is written at that column’s offset `0x11`.

The argument-field reloads are established; a stronger relative evaluation order between the two comparison arguments is not established here.

Thus, without relevant dependency mutation, the first matching column is marked, and an unsuccessful search ends at the column count without marking. Those statements are conditional: comparison callbacks can alter counts, array fields, or contents before the post-search accesses.

After entry processing ends, the list count is read again. A value of two or greater discards the candidate, regardless of how many entries were actually processed or how many distinct columns matched.

If the final count is below two, the retained candidate is eligible for range checking. With a stable one-entry list, it is that entry’s resulting search index. With an initially zero or negative count and no intervening mutation, no entry is processed and the candidate remains negative one.

Existing membership marks are not cleared. Repeated names can mark the same column repeatedly.

### Selection without a list

With a null list, the candidate is the column count minus one. The function immediately writes byte one to that candidate’s membership field through the table’s column-array field.

With a valid positive count, this marks the last column. With count zero, the calculated write address is three bytes before the column-array base. No controlled rejection precedes that write.

### Specialized representation

A candidate’s type-name word is read only if its index is signed nonnegative and strictly below a freshly read signed column count. The column-array field is then read to locate the type-name word.

A null type-name word prevents specialization. Otherwise, F-e6db2171781a compares it with `INTEGER`. Equality is a zero low 32-bit result; the comparator’s encoding and equivalence rules are unknown.

Specialization requires equality and an additional key option of exactly zero. On success, the function writes, in order:

1. The 32-bit candidate index at table offset `0x0c`.
2. The policy’s low eight bits at `0x37`.
3. The automatic-increment request’s low eight bits at `0x38`.

The table pointer and candidate are retained across the type comparison. The function does not repeat candidate validation or type lookup after that dependency returns.

No general-processing call occurs. Any supplied list enters local cleanup.

The automatic-increment request need not be zero or one. For example, `0x100` qualifies on this path but stores byte zero.

### General processing or restriction diagnostic

If specialization fails, the full automatic-increment argument selects the outcome.

For zero, F-d124af1bcb5a receives ten arguments, in order:

| Position | Value |
|---:|---|
| 1 | Context |
| 2–4 | Zero |
| 5 | Declaration-list address, possibly null |
| 6 | Full policy |
| 7–8 | Zero |
| 9 | Full additional key option |
| 10 | Zero |

After this call returns, local cleanup is suppressed. This establishes caller-side handoff, not the dependency’s actual ownership or disposal behavior.

For nonzero, F-2022b0f73f3b receives the context and the diagnostic restricting automatic increment to the qualifying integer-primary-key case. Any supplied list then enters local cleanup.

Neither outcome reverses earlier declaration or membership writes.

## Side effects

### Incremental table changes

Direct table changes comprise the declaration-seen byte, membership bytes, specialized index, policy byte, and automatic-increment byte. Changes are incremental, not transactional. Dependencies may cause additional effects not specified here.

### Entry cleanup: retained cursor and fresh reads

For a non-null list requiring local cleanup, the entry-array field is read once before the loop to initialize a retained cleanup cursor.

Each iteration:

1. Freshly reads the list count for the signed loop condition.
2. Reads the first resource at the retained cursor and calls F-841dc33ee157 with it.
3. After that call returns, reads the second resource at cursor offset `0x04` and calls F-94cb53877c15 with it.
4. Advances the cursor by `0x0c` bytes and increments the loop index.

The cursor is not reloaded from the list between iterations. Consequently:

- Changing the list’s entry-array field does not redirect this traversal.
- Changing the current entry’s second resource during F-841dc33ee157 changes the subsequent cleanup argument.
- Changing the list count can shorten or extend traversal.
- Resource values are passed without explicit null checks.

After the loop, the entry-array field is freshly read from list offset `0x0c` and passed to F-94cb53877c15. This argument can differ from the array used by the retained cursor.

Zero or negative counts skip per-entry calls but not the array-release call or subsequent allocation handling.

### Allocation accounting: global reads and destinations

After the array-release call returns, the function captures the accounting-global pointer. It then reads the displacement word at list address minus four and subtracts it from the list address to derive an allocation-related base.

Next, the established access order is:

1. Read a 32-bit accounting quantity at derived base minus eight.
2. Read the accounting-global pointer again and read the low word at its offset `0x38`.
3. Read the global again to select the high-word source address at offset `0x3c`.
4. Read the global again to select the low-word destination at offset `0x38`, then store the low-word subtraction there.
5. Read the high word through the previously selected source address and store its adjusted value at offset `0x3c` through the initially captured pointer.

The low-word subtraction is modulo \(2^{32}\). The high-word adjustment subtracts the sum of:

- The arithmetic sign extension of the accounting quantity’s top bit, yielding zero or negative one.
- The unsigned borrow from the low-word subtraction.

The high-word result is also 32-bit arithmetic. For quantities with bit 31 set, this must not be simplified to ordinary unsigned 64-bit subtraction.

If the accounting-global pointer remains stable, these accesses address one two-word value. The distinct pointer reads and retained destination remain part of the contract; pointer stability is not guaranteed.

F-2fd8826557a4 is then called. Only afterward does the function dereference the separate allocator-context global to obtain the allocator context. It therefore does not retain a pre-call allocator-context value.

### Header validation and allocator counters

If the derived base equals numeric address `8`, header validation and the remaining allocator sequence are skipped. Entry cleanup, array release, accounting, F-2fd8826557a4, and allocator-context acquisition have already occurred.

Otherwise, the header address is derived base minus `0x10`. The header is read to check for zero. If nonzero, it is read again to check bit zero. Either zero or a set bit zero calls nonreturning F-a817d8d58c10.

After successful validation, F-c0fc6d033caa receives the retained allocator context. Following its return, accesses occur in this order:

| Order | Access or update |
|---:|---|
| 1 | Read the first unsigned 16-bit class index at base minus `0x0c`. |
| 2 | Read the 32-bit header and mask it with `0x3ffffffc` to obtain a size-like quantity. |
| 3 | Subtract that quantity from context offset `0x224 + 4 × first index`. |
| 4 | Subtract the same quantity from context offset `0x30c`. |
| 5 | Read the second unsigned 16-bit class index at base minus `0x0a`. |
| 6 | Subtract the retained quantity from context offset `0x0e0 + 4 × second index`. |
| 7 | Subtract the retained quantity from context offset `0x220`. |
| 8 | Call F-09dcb55e8e0a to obtain another index. |
| 9 | Decrement the 32-bit counter at context offset `0x310 + 4 × returned index`. |
| 10 | Decrement the 32-bit counter at context offset `0x390`. |
| 11 | Freshly read the header, mask with `0x3ffffffc`, and subtract that quantity from context offset `0x08`. |
| 12 | Call F-cc6c9b4a0f68 with the retained context and header address. |
| 13 | Call F-e586bff64fb7 with the retained context. |

The first four size-based subtractions share one retained quantity. The subtraction at offset `0x08` uses a fresh header read after F-09dcb55e8e0a.

The retained allocator context is not reacquired after these calls. Counter arithmetic is 32-bit, with no explicit underflow or index-range checks. Header validation is not repeated after F-c0fc6d033caa or F-09dcb55e8e0a.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-e6db2171781a | Name and type comparisons; low 32-bit zero selects equality. |
| F-2022b0f73f3b | Duplicate-declaration and automatic-increment restriction diagnostics. |
| F-d124af1bcb5a | General processing with the ten-argument arrangement above. |
| F-841dc33ee157 | First resource from each cleanup entry. |
| F-94cb53877c15 | Second entry resource, then freshly loaded entry-array address. |
| F-2fd8826557a4 | Called after accounting and before allocator-context acquisition. |
| F-a817d8d58c10 | Nonreturning invalid-header response. |
| F-c0fc6d033caa | Receives allocator context before class-index reads and counter updates. |
| F-09dcb55e8e0a | Supplies the subsequent counter-selection index. |
| F-cc6c9b4a0f68 | Receives allocator context and header address. |
| F-e586bff64fb7 | Final call with allocator context on ordinary release. |

Names such as “cleanup” and “release” describe placement and apparent role, not proven dependency ownership, synchronization, or deallocation semantics.

## Acceptance examples

These are conditional static expectations, not executed tests. Except where mutation is specified, they assume valid memory, normal dependency returns, and no relevant mutation.

1. **Missing table or inhibited processing.** No main-path table writes or diagnostic occur. A supplied list still undergoes local cleanup. With a missing table, the inhibition field is not read.

2. **Duplicate declaration.** A nonzero declaration marker produces the duplicate diagnostic with the table’s first word. No name search occurs before cleanup.

3. **Implicit specialized column.** With three columns, no list, last-column type comparing equal to `INTEGER`, and additional option zero, column two is marked and index two is stored. Policy `0x12345678` stores `0x78`; automatic-increment request `0x100` stores zero.

4. **Explicit single-column outcomes.** A one-entry list matching column zero can specialize index zero. With no match and a stable column count, no membership write occurs; zero automatic increment delegates, while nonzero produces the restriction diagnostic.

5. **Multiple or empty entries.** Two entries matching the same column mark it twice but disable specialization. A zero-count list selects no candidate and delegates when automatic increment is zero.

6. **Disqualifying additional option.** Even an `INTEGER` candidate cannot specialize with a nonzero additional option. The full automatic-increment argument then selects delegation or diagnostic.

7. **Mutation during name comparison.** If a comparison returns equality but reduces the column count to the current index, the subsequent count reload prevents marking. If the index remains below the count but the column-array field changes, marking targets the newly selected array.

8. **Final list-count mutation.** If one entry is processed but the freshly read final count is two, specialization is disabled. Eligibility is not based solely on entries already traversed.

9. **Cleanup cursor mutation.** If F-841dc33ee157 changes the current entry’s second resource, the next cleanup call receives the changed value. If it changes the list’s array field, cursor traversal continues through the original array, but the final array-release call receives the newly loaded field.

10. **Allocator mutation.** A changed allocator-context global during F-2fd8826557a4 affects subsequent context selection. A header change during F-09dcb55e8e0a affects the subtraction at context offset `0x08`, not the four earlier size-based subtractions.

11. **Empty implicit table.** With zero columns and no list, marking addresses three bytes before the column-array base. This violates normal-operation preconditions; safe rejection is not established.

## Uncertainties

- **Inferred intent:** declaration processing concerns primary keys; specialization appears to encode an integer-primary-key column. Downstream semantics are unknown.
- **Documented_only:** policy and additional-option meanings are not established.
- **Documented_only:** comparison encoding, case sensitivity, null handling, and malformed-name behavior belong to F-e6db2171781a.
- **Documented_only:** general processing’s diagnostics, table changes, and eventual list disposal are unknown.
- **Documented_only:** dependencies’ ownership, null tolerance, reentrancy, synchronization, and global mutations are unknown. Caller-side reloads and retained values remain established despite those unknown effects.
- **Documented_only:** allocator metadata meanings, permitted class indices, and the returned counter index’s valid range are unknown.
- **Documented_only:** malformed pointers, inconsistent extents, arithmetic overflow, and invalid resources have no general recoverable-error contract. Only the explicit invalid-header nonreturning path is established.

## Evidence

This contract is based solely on the supplied static function representation: access widths and offsets, retained addresses, reload locations, branch conditions, dependency arguments, arithmetic, and observable ordering.

Behavioral labels are interpretations of those accesses, not independently verified type definitions. Dependency implementations were not supplied.

No runtime tests, execution traces, or independent implementation inspection have been performed.
