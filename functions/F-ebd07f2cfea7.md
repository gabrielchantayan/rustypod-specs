<!-- Trusted provenance: F-ebd07f2cfea7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ebd07f2cfea7

## Purpose

F-ebd07f2cfea7 assigns an index to an optional record and updates a mutable processing context. Depending on record classification and spelling, it assigns a new sequential index, uses an explicitly converted number, or searches registered records for an index to copy.

A successful search comparison does **not** independently determine whether sequential assignment is skipped. After the search ends, including after a matching record’s index has been copied, the function rereads the registration count. It performs sequential assignment and possible registration when that count is less than or equal to the search’s stopping slot index. A comparison dependency that reduces the count can therefore cause a copied index to be overwritten.

After processing a nonnull record, the function checks the context’s index high-water mark against a configured limit unless an error/status field is nonzero. Exceeding the limit triggers diagnostic-related updates without rolling back assignment or registration.

The supplied diagnostic text supports an interpretation as SQL-variable numbering. Exact record types, accepted language syntax, and parser lifecycle remain inferred intent rather than established interface guarantees.

## Interface

### Arguments and result

The function accepts:

- **Context:** an address identifying mutable processing state.
- **Record:** a nullable address identifying a mutable record.

There is no returned value. Observable outputs consist of memory updates and dependency calls.

The supplied representation uses 32-bit integers and 32-bit address-bearing fields. The layouts below describe that representation, not a portable layout for arbitrary machines.

### Context layout

Offsets are byte offsets from the context address.

| Offset | Width | Established role |
|---|---:|---|
| `0x00` | 32 bits | Address of a shared configuration/state object |
| `0x04` | 32 bits | Status field conditionally set to `1` during final limit reporting |
| `0x08` | 32 bits | Diagnostic-related value passed to a dependency and subsequently replaced |
| `0x40` | 32 bits | Error/status field whose nonzero value suppresses the final limit check |
| `0x14C` | 32 bits | Signed index high-water mark |
| `0x150` | 32 bits | Signed registration count |
| `0x154` | 32 bits | Capacity-related value used in the growth predicate and update |
| `0x158` | 32 bits | Address of a table containing 32-bit record addresses |

### Shared-object layout

The shared-object address is read at entry and retained for later limit, growth, and append-control operations.

| Offset | Width | Established role |
|---|---:|---|
| `0x1E` | 8 bits | Zero permits appending a newly sequentially numbered named record |
| `0x74` | 32 bits | Signed maximum permitted index |

The meaning of the byte at `0x1E` is not established. In particular, interpreting it as an allocation-failure indicator requires information not supplied here.

### Record layout

| Offset | Width | Established role |
|---|---:|---|
| `0x14` | 32 bits | Address of spelling bytes |
| `0x18` | 32 bits | Unsigned classification |
| `0x24` | 32 bits | Assigned index |

Classification comparisons use the unsigned classification shifted right by one bit. The least significant bit does not distinguish classes for this function.

### Preconditions

The context must permit the entry read at `0x00`, even when the record argument is null.

For a nonnull record, memory reached by the selected route must support the applicable reads and writes. This includes the record’s classification and index fields; spelling pointers and bytes on spelling-based routes; visited table slots; fields of nonnull stored records; shared-object fields; and any appended table slot.

Dependencies must receive arguments valid under their own contracts, which are not supplied. The function performs no independent validation of pointer accessibility, table extent, spelling termination, or record lifetime.

## Behavior

### Entry and null record

The function reads and retains the shared-object address from the context before examining whether the record argument is null.

A null record causes an immediate return after that read. There is no classification read, index assignment, table search, or final limit check. An excessive existing high-water mark is therefore not reported by a null-record invocation.

### Designated classification

The function reads the incoming record’s unsigned classification. If its right-shifted value is `1`, it uses sequential assignment. Raw classifications `2` and `3` both select this route.

Sequential assignment reads the current high-water mark, writes its incremented value to the context, and then writes the same computed value to the incoming record’s index field.

This classification route does not inspect spelling or perform table lookup, growth, or registration. It proceeds to the final error/status gate.

### Explicit-number spelling

For other classifications, the function reads the record’s spelling address and first spelling byte. A first byte of `?` selects explicit-number processing.

F-b902b5c89b26 receives the address immediately following that byte. Its signed 32-bit result becomes the requested index. The function writes that result to the incoming record’s index field before reporting any range error.

The range decision rejects a result that is:

- Less than or equal to zero; or
- Positive and greater than the signed limit obtained through the retained shared-object address.

The limit read for this decision is conditional on a positive result. The supplied overflow-aware comparison expressions establish signed comparison behavior: the minimum signed 32-bit result, `-2147483648`, is rejected along with other nonpositive results.

For a rejected index, F-2022b0f73f3b receives the context and a range-diagnostic argument. The full diagnostic wording and formatting contract are unavailable.

If that dependency returns, processing continues. The function rereads the high-water mark and raises it to the converted result when the current high-water mark is smaller under signed comparison. The converted result remains the value used for that comparison; it is not obtained by rereading the record’s index after reporting.

There is no local rollback or unconditional early return following range reporting. Consequently, an excessive positive explicit index can raise the high-water mark even after being reported as invalid.

This route does not search or append to the registration table. It proceeds to the final gate.

Conversion syntax, complete-input consumption, malformed-input handling, and conversion overflow are delegated to F-b902b5c89b26.

### Named-record search

A spelling whose first byte is not `?` selects the registration-table search.

The search starts at slot index zero. Before each visit, it compares the signed slot index against the current signed registration count. The count is read from the context for the loop condition rather than fixed as an entry snapshot.

For each visited slot, the function obtains the current table address and reads the stored 32-bit record address at a four-byte slot offset. A zero address is skipped.

A nonnull stored record is eligible for spelling comparison only if its right-shifted classification equals the incoming classification captured before the search. For an eligible record, F-7f14f1ab6851 receives the stored record’s spelling address first and the incoming record’s spelling address second. These spelling addresses are read for the comparison call; the incoming spelling address is not established as a fixed search-wide snapshot.

A zero comparison result causes the function to read the selected stored record’s index and copy it to the incoming record’s index field. That read occurs after the comparison dependency returns. The stored record address used for the copy is the address selected before that call, not a fresh lookup of the table slot.

The search then stops without incrementing its slot index. A nonzero comparison result continues the search, advancing the slot index before the next loop condition.

The copied index receives no direct positivity or range validation. Copying it does not itself update the high-water mark.

### Independent post-search decision

After the search ends, the function rereads the current registration count and compares it with the slot index at which the search stopped.

**New sequential assignment occurs when the current signed registration count is less than or equal to that slot index.** Otherwise, the function skips sequential assignment and registration.

This decision is independent of whether a spelling comparison succeeded. There is no separate match flag controlling allocation.

Under stable state, a match at an in-range slot leaves the count greater than the stopping index, so the copied index is retained. A search that exhausts a stable nonnegative count reaches an equal stopping index and selects sequential assignment.

Dependency mutation changes those consequences. If a comparison at slot `2` returns zero but reduces the registration count to `2` or less, the function first copies the stored record’s index, then selects sequential assignment. The newly assigned index overwrites the copied index, and growth and registration remain possible.

Conversely, when the reread count is greater than the stopping index, no new assignment occurs through this branch. This is the controlling condition; “a match was found” is not a sufficient standalone contract.

### Sequential assignment and possible registration after search

When the post-search condition selects this branch, the function increments the current high-water mark, writes the incremented value to the context, and then writes it to the incoming record’s index field.

It next evaluates the signed growth predicate:

**capacity minus one is less than or equal to registration count.**

When true, it writes **twice the previous capacity plus ten** to the capacity field before calling F-814e6ca769cc. That dependency receives the shared-object address retained at entry and the current table address. Its returned 32-bit value replaces the context’s table address without a local success check.

No new-capacity argument is explicit in the supplied call. The function’s capacity update does not establish the dependency’s allocation size.

After the optional growth call, the function reads the byte at `0x1E` through the retained shared-object address. A nonzero byte suppresses the append, but does not undo numbering or capacity changes.

When the byte is zero, the function reads the current registration count, increments that count in the context, and then writes the incoming record address into the table slot indexed by the old count. The table address used for the slot write is read after the count update.

The append therefore uses post-dependency context state, not necessarily the count or table address present before growth. It stores the record address itself, without copying the record or spelling.

### Final error/status gate and limit reporting

All nonnull-record routes reach the final stage if their dependency calls return.

A nonzero context field at `0x40` causes return without the final limit comparison. It does not suppress assignment, search, growth, or registration that preceded this gate.

Otherwise, the function compares the current signed high-water mark with the signed limit reached through the shared-object address retained at entry. No report is made when the high-water mark is less than or equal to that limit.

When the high-water mark exceeds the limit, the visible ordering is:

- Increment the context field at `0x40`.
- Pass the current value at `0x08` to F-94cb53877c15.
- Call F-adee04f6d176 with a freshly read shared-object address from context, the diagnostic text `too many SQL variables`, and an additional stack-associated address.
- Store its returned 32-bit value at `0x08`.
- Read the status at `0x04` and write `1` only if it is zero.

The shared-object address supplied to F-adee04f6d176 can differ from the address retained at entry if a dependency changed the context.

Reporting does not locally roll back any assigned index, high-water mark, capacity, registration count, or table entry.

### Arithmetic boundaries

The representation uses 32-bit index increments, registration-count increments, capacity arithmetic, and table-address calculations. No explicit overflow rejection or saturation is present for those operations.

For nonoverflowing values, sequential assignment adds one, capacity growth doubles and adds ten, and table slots are four bytes apart.

Exact execution semantics of overflowing signed arithmetic and address formation are not independently established by the supplied representation. Overflow-boundary outcomes are `documented_only`, not promises of wrapping, rejection, saturation, or safe storage access.

## Side effects

Direct writes can affect:

- The incoming record’s index, potentially twice after a matching comparison.
- The context’s high-water mark, capacity, table address, and registration count.
- A registration-table slot.
- The context’s error/status, diagnostic-related, and status fields.

The function performs no explicit record copy, spelling copy, or direct memory release.

Registration retains the incoming record address for later dereference. This creates a lifetime requirement for subsequent lookups, but does not establish ownership transfer. Replacing table or diagnostic-related addresses does not prove that previous storage is released.

Dependency side effects are not constrained by this function. In particular, the count-based decision after comparison makes dependency mutation immediately relevant to observable behavior; it is not merely a general uncertainty.

No synchronization or transactional rollback is visible. Guarantees for concurrent mutation, reentrancy, and overlapping context, record, or table storage are not established.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-b902b5c89b26 | Receives the spelling address after `?`; returns the signed 32-bit requested index | Grammar, overflow, malformed input, state mutation |
| F-7f14f1ab6851 | Receives stored and incoming spelling addresses; zero triggers index copying and search termination | Comparison rules, encoding, termination requirements, memory and state effects |
| F-814e6ca769cc | Receives retained shared-object address and current table address; result replaces table address | Allocation size, content preservation, failure signaling, ownership |
| F-2022b0f73f3b | Receives context and range-diagnostic argument after the explicit index is stored | Full message, status effects, return behavior |
| F-94cb53877c15 | Receives the previous diagnostic-related value before replacement | Release behavior, other side effects |
| F-adee04f6d176 | Receives current shared-object address, limit diagnostic, and stack-associated address; result replaces diagnostic-related value | Third-argument meaning, allocation, ownership, failure behavior |

No dependency is assumed to preserve globals, context fields, records, table contents, or memory ownership.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless specified otherwise, assume valid nonoverlapping storage, no arithmetic overflow, returning dependencies, and preservation of relevant state across dependency calls.

1. **Null record.** A zero record argument causes the entry shared-object-address read followed by return. An excessive existing high-water mark does not trigger reporting.

2. **Designated class.** With raw classification `2` or `3`, high-water mark `4`, limit `10`, and zero error/status, the high-water mark and record index become `5`. No spelling lookup or registration occurs.

3. **Explicit index above the high-water mark.** If conversion returns `7`, the high-water mark is `3`, and the limit is `10`, the record receives `7` and the high-water mark becomes `7`.

4. **Explicit index below the high-water mark.** If conversion returns `2` and the high-water mark is `8`, the record receives `2`; the high-water mark remains `8`.

5. **Invalid explicit index.** A converted result of `0` or `-2147483648` is written before F-2022b0f73f3b is called. Subsequent state depends on that dependency’s effects and return behavior.

6. **Stable matching search.** With count `3`, a matching comparison at slot `1`, and no count mutation, the stored index is copied. The post-search comparison sees `3 > 1`, so no sequential assignment or append occurs.

7. **Match followed by overwrite.** With count initially `3`, high-water mark `4`, and a candidate at slot `1` holding index `9`, suppose comparison returns zero and changes only the count to `1`. The record first receives `9`. The post-search condition then sees `1 <= 1`, causing high-water mark and record index to become `5`. With capacity `10` and zero append-control byte, count becomes `2` and slot `1` receives the incoming record address.

8. **Count reduction that does not trigger assignment.** At slot `1`, suppose a successful comparison reduces count from `4` to `2`. The copied index remains because the post-search comparison sees `2 > 1`.

9. **Stable exhausted search.** With count `2`, no match, capacity `10`, high-water mark `4`, and zero append-control byte, the stopping slot index is `2`. Sequential assignment produces `5`; count becomes `3`; slot `2` receives the incoming record address.

10. **Growth boundary.** With capacity `3` and count `2` when growth is evaluated, capacity becomes `16` before F-814e6ca769cc is called. Appending remains conditional on the later byte read.

11. **Append suppression.** A nonzero append-control byte does not undo sequential numbering. The function performs no append count increment or slot write on that branch.

12. **Existing error.** A nonzero field at `0x40` suppresses only the final limit check, not preceding assignment or registration.

13. **Final limit report.** Starting with high-water mark `10`, limit `10`, and zero error/status, sequential numbering first produces `11`. Final reporting follows. If dependencies preserve the relevant fields, `0x40` becomes `1`, and a zero status at `0x04` becomes `1`.

## Uncertainties

The following remain `documented_only`:

- Exact integer-to-address execution semantics and signed-overflow outcomes.
- Conversion and spelling-comparison contracts.
- Full range-diagnostic wording.
- Meaning of the append-control byte.
- Table allocation size, content preservation, and failure behavior.
- Ownership and release of records, tables, spellings, and diagnostic-related storage.
- Meaning and lifetime of the stack-associated argument.
- Dependency behavior under nonlocal exits, reentrancy, or arbitrary memory mutation.
- Safety for negative counts, invalid capacities, malformed tables, null spellings, or overlapping storage.

These unknowns do not obscure the established post-search rule: the current signed count, compared with the stopping slot index, independently controls sequential assignment after both unsuccessful searches and successful index copies.

## Evidence

This contract derives solely from the supplied static representation and dependency-ID mapping: access widths and offsets, signed and unsigned comparisons, branch structure, dependency arguments, and visible ordering of reads and writes.

The separate post-search count comparison is directly present after search termination, including the termination path that copies a matching record’s index. Its consequences under count mutation are conditional deductions from that control flow, not assumptions about a dependency implementation.

No dependency implementations were supplied. No runtime tests, smoke runs, mutation experiments, allocation-failure checks, or overflow-boundary executions have been performed. The acceptance examples state conditional consequences of static evidence, not observed results.
