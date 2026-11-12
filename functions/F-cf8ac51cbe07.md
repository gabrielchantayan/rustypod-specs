<!-- Trusted provenance: F-cf8ac51cbe07; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cf8ac51cbe07

## Purpose

F-cf8ac51cbe07 attempts to associate a nonempty byte sequence with a one-based entry identifier in a mutable storage object. A retained mode flag determines whether it first attempts to reuse an existing entry and whether insertion increments a per-entry counter. Otherwise, it obtains a free entry, obtains a byte offset, conditionally grows storage, and publishes the entry.

An interning or deduplicating store is **inferred intent**. The established behavior is the sequence of calls, reads, writes, branches, and status handling described below; dependency implementations are unavailable.

A zero-length request returns success without lookup or insertion, but only after validation, output reset, and object-state checks.

## Interface

### Arguments and result

The four arguments have 32-bit representations:

| Argument | Established interpretation |
|---|---|
| Object address | Address of the mutable object described below. |
| Sequence value | Opaque value passed to comparison, transfer, specialized lookup, and notification dependencies. Interpretation as a source-buffer address is inferred. |
| Length | Unsigned 32-bit quantity. Some operations interpret its bit pattern as signed. |
| Identifier address | Address of a four-byte output slot. Its incoming contents are read for validation before the address is checked for null. |

The result is a signed 32-bit status:

- `0`: accepted empty request, successful lookup path, or completed insertion path.
- `-50`: either availability check returns zero, the state word is nonzero, or rejection flag `0x40` is set.
- Nonzero validation or checked growth status: returned unchanged on the paths specified below.

An identifier explicitly published by this function is one-based. The explicit reset value is zero. Success following specialized lookup does not independently guarantee a valid identifier: that dependency controls its output.

### Object and backing-storage layout

Every listed object field is four bytes wide.

| Object offset | Established use |
|---|---|
| `+0x04` | Flags: `0x01` enables reuse and insertion counter updates; `0x04` selects specialized lookup and conditional notification; `0x40` rejects the operation. |
| `+0x08` | Address of a cell containing the entry-table base address. |
| `+0x0c` | Address of a cell containing the counter-array base address. |
| `+0x10` | Address of a cell containing the byte-storage base address. |
| `+0x1c` | Entry count. Linear lookup compares it as signed. |
| `+0x20` | One-based free-list head; zero indicates no available entry. |
| `+0x24` | Available-byte quantity, compared as signed against the length interpreted as signed. |
| `+0x28` | Unsigned byte-storage growth quantum. |
| `+0x2c` | Entry-table growth quantum; tested as unsigned for doubling and initialization bounds. |
| `+0x30` | State word; must be zero to proceed. |

Each entry occupies eight bytes. Its first word is an occupied-entry byte offset or the free marker `0x80000000`. Its second word is an occupied-entry length or the next one-based free-list identifier.

Each counter occupies four bytes and corresponds to the entry at the same zero-based index.

### Preconditions and limits

The identifier slot must be readable when validation is reached. The later null check does not make a null identifier address safe.

All reached object, handle-cell, table, counter, and byte-storage accesses require valid memory. No visible bounds checks establish sufficient table or counter capacity, valid free-list links, accessible sequence bytes, safe overlap, or validity of the byte offset returned by a dependency.

Addresses, stored quantities, counters, and scaled requests are 32-bit. No explicit overflow checks protect count additions, counter increments, capacity arithmetic, identifier calculations, or growth-size calculations. Safe operation requires representable, coherent ranges; this contract does not supply missing overflow or invalid-pointer semantics.

## Behavior

### Initial validation and rejection

The function first calls F-0aff6a5050f5 without explicit arguments in the supplied representation. A zero result causes immediate return of `-50`, before any function-owned output reset.

Otherwise, it reads the incoming identifier and passes that value with the object to F-808c5d99c3bd. A nonzero result is returned immediately.

After successful validation:

1. The identifier slot is explicitly set to zero if its address is nonzero.
2. F-0aff6a5050f5 is called with the object.
3. A zero result causes `-50`.
4. The state word at `+0x30` is read. A nonzero value causes `-50`.
5. Only if the state word is zero are the flags read. Flag `0x40` causes `-50`.

The value of flag `0x01` from this flags read is retained for the rest of the invocation. Later dependencies may change the flags, but insertion counter handling and counter-array growth use this retained value rather than refreshing it.

If the length is zero, the function returns `0` at this point. There is no lookup, allocation-loop call, transfer, or explicit counter update. Dependencies already called may have performed their own mutations.

### Optional lookup

Lookup requires retained flag `0x01` to be set, the `+0x08` handle field to be nonzero, and then the `+0x10` handle field to be nonzero. These handle fields are checked in that order. Missing handles skip lookup; they do not establish safety for later allocation accesses.

F-efaaaab570a3 is called before lookup. The function then rereads flags and selects lookup using the current value of bit `0x04`.

#### Linear lookup

When bit `0x04` is clear, the function captures, in order:

1. The byte-storage base through `+0x10`.
2. The counter-array base through `+0x0c`.
3. The entry-table base through `+0x08`.

It visits entries in increasing zero-based index order. Each loop condition rereads `+0x1c` and compares the index against that field as signed. A nonpositive count admits no candidates at that check.

For each candidate:

- F-88337f8f5d95 receives the entry address.
- If its result is nonzero, the entry’s second word is compared with the requested length.
- If lengths match, the comparison dependency receives the sequence value, captured byte-storage base plus the entry’s first word, and the requested length.
- A nonzero comparison result is a match.

On the first match, the counter selected from the captured counter-array base is incremented before the one-based identifier is written. F-2a517f9b9ef3 is then called, and the function returns `0`.

The entry traversal pointer and captured bases are not refreshed after each eligibility or comparison call. Their continued validity across those calls is not established by the supplied dependency contracts.

#### Specialized lookup

When current bit `0x04` is set, F-849b785f50ce receives the object, sequence value, length, and identifier address.

Any nonzero result, including a negative result, is treated as handled: F-2a517f9b9ef3 is called, then this function returns `0`. The dependency’s status is not propagated.

A zero result falls through to allocation. Both lookup modes call F-2a517f9b9ef3 after an unsuccessful search as well. The pairing with F-efaaaab570a3 suggests resource acquisition and release, but synchronization semantics remain unknown.

### Selecting a free entry

Every allocation-loop iteration begins by calling F-30a0e9a392e7 with the object and zero. The free-list head is read afterward.

For a nonzero head, the function uses the current entry-table base and that one-based head to locate the eight-byte entry. It reads the entry’s second word and writes it to `+0x20`, popping the entry.

Only after this read and head update does it test whether the selected entry address is null. Consequently, that test does not protect the preceding access. If the selected address is null, execution falls through to entry-table growth.

For a nonnull selected address, the function derives its one-based identifier from the selected address and a reread of the entry-table base, using a signed difference shifted right by three, then adding one. It calls F-ec8cbc7f302e with the object. The returned 32-bit value is retained as a byte offset without a visible failure check.

### Conditional byte-storage growth

The available-byte field at `+0x24` is read after the offset-provider call. Growth occurs only if its signed interpretation is less than the signed interpretation of the requested length.

For growth:

- The byte-growth quantum is read from `+0x28`.
- The request amount is the unsigned maximum of that read value and the length.
- If the read quantum is below `0x20000`, twice that value is written to `+0x28` before the growth call.
- F-7eff13797b9b receives the address of `+0x10` and the selected request amount.

The request uses the quantum from before doubling. Doubling is threshold-controlled, not capped at the threshold.

After the call, whether it succeeded or failed, the selected entry address is recomputed from the current entry-table base and saved identifier.

On nonzero growth status, the function writes the free marker to that entry, writes the then-current free-list head into its second word, publishes the selected identifier as the new head, and returns the growth status. This does not undo the quantum update or arbitrary dependency mutations.

On zero growth status, the selected request amount is added to the then-current available-byte field.

If no byte growth occurs, the selected entry pointer is not recomputed after F-ec8cbc7f302e.

### Transfer and publication

F-70a021fc7c24 receives the sequence value, the current byte-storage base plus the retained offset, and the length. Its return value is ignored.

After that call, explicit writes occur in this order:

1. Subtract the length from the then-current available-byte field.
2. Write the length to the selected entry’s second word.
3. Write the retained offset to its first word.
4. Write the saved one-based identifier to the output slot.

The selected entry pointer is not refreshed after the transfer call.

If retained flag `0x01` was clear, the function returns `0`. Otherwise, it obtains the current counter-array base and increments the selected counter. It then rereads flags. If current bit `0x04` is set, it calls F-4cffcc8d6e9b with the object and sequence value, ignoring the result. It returns `0`.

The selected counter is not explicitly initialized before incrementing.

### Entry-table growth and consequential rereads

Entry-table growth is reached when no nonzero free head is available or the selected entry-address test falls through.

The function tests `+0x2c` as unsigned against `0x1000`. If below that threshold, it reads the field for doubling and writes the doubled value back. A zero quantum remains zero.

Growth requests do **not** share a quantum snapshot across dependency calls:

1. **Entry-table request:** F-7eff13797b9b receives the address of `+0x08` and eight times the current `+0x2c` value. A nonzero result is returned immediately.
2. **Counter request:** only if retained flag `0x01` is set, the function rereads `+0x2c` after successful entry-table growth. F-7eff13797b9b receives the address of `+0x0c` and four times this newly read value.
3. **Compensation:** if counter growth fails, the function rereads `+0x2c` again. It requests negative eight times this latest value for `+0x08`, ignores the compensation result, and returns the counter-growth failure.

Thus counter growth can use a different quantum from entry-table growth, and compensation can use a third quantum. Compensation is not guaranteed to reverse the earlier request.

After all required growth calls succeed, initialization proceeds from current fields rather than pre-growth snapshots:

- It obtains the then-current entry-table base and reads the then-current `+0x1c` count to locate the starting entry.
- Its unsigned iteration index begins at zero. Every loop condition rereads `+0x2c`.
- Each visited entry’s first word is written as `0x80000000`.
- Starting with the second visited entry, the preceding entry’s second word is written using a fresh read of `+0x1c`, plus the current iteration index, plus one.
- Traversal advances by eight bytes per iteration; the starting pointer is not recomputed from later count reads.

The final visited entry’s second word is not explicitly initialized.

After initialization, the function separately reads `+0x1c` and writes that value plus one to the free-list head. It then reads `+0x1c` and `+0x2c` for the count update, writing their sum to `+0x1c`. These are not uses of a count or quantum retained before the growth calls.

The allocation loop then repeats. Newly added counters receive no explicit initialization here.

## Side effects

Function-owned writes can affect the identifier slot, entry words, counters, free-list head, entry count, available-byte quantity, and both growth quanta. Dependencies may additionally change handles, backing storage, object fields, or other state.

Failure does not imply rollback:

- Initial availability or validation failure precedes the explicit output reset.
- Later rejection follows that reset.
- Growth-quantum changes survive checked growth failure unless some dependency changes them.
- Counter-growth compensation is unchecked and may use a different quantum.
- Byte-growth failure reinserts one entry into the then-current free list, not necessarily the original list state.
- Unsuccessful specialized lookup is not established to preserve output or object contents.

Successful publication also does not establish that the output remains unchanged afterward: counter handling and notification follow the output write on some paths, and dependency effects or aliasing may matter.

Managed, relocatable storage is **inferred intent** from handle indirection and storage-adjustment calls. Ownership, release obligations, relocation guarantees, and lifetime rules are **documented_only**. No explicit general release of the object or sequence appears.

## Dependencies

| Opaque ID | Established caller-side role |
|---|---|
| F-0aff6a5050f5 | Two availability checks; zero causes `-50`. |
| F-808c5d99c3bd | Validation using the object and incoming identifier; nonzero propagates. |
| F-efaaaab570a3 | Called before optional lookup. |
| F-2a517f9b9ef3 | Called on every normal exit from optional lookup. |
| F-88337f8f5d95 | Linear candidate eligibility; nonzero permits length comparison. |
| F-849b785f50ce | Specialized lookup; nonzero becomes overall success. |
| F-30a0e9a392e7 | Called with the object and zero before every free-list attempt. |
| F-ec8cbc7f302e | Returns the value used as the byte offset, unchecked. |
| F-7eff13797b9b | Storage adjustment using an object-field address and scaled amount. |
| F-70a021fc7c24 | Transfer-like operation; result ignored. |
| F-4cffcc8d6e9b | Conditional notification after insertion; result ignored. |

The supplied mapping contains no opaque ID for the comparison dependency. Its established convention here is nonzero for a match. Its identity and broader contract remain **documented_only**; no replacement ID is invented.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless specified otherwise, assume valid memory, representable arithmetic, no conflicting aliasing, and dependencies that preserve fields not explicitly varied.

1. **Initial rejection.** If the first availability result is zero, expect `-50`, no validation call, and no function-owned output reset.

2. **Validation failure.** If validation returns a particular nonzero status, expect that exact status. Its identifier argument must be the incoming slot contents, not the reset value.

3. **Empty request.** With successful validation and availability checks, zero state, and rejection flag clear, zero length returns `0` after the explicit reset. No lookup or allocation follows.

4. **Linear match.** If the third entry is the first eligible equal-length candidate whose comparison returns nonzero, expect counter index two to increment before identifier `3` is written, followed by F-2a517f9b9ef3 and status `0`.

5. **Specialized negative result.** A negative nonzero specialized-lookup result leads to F-2a517f9b9ef3 and overall status `0`. Valid output requires an additional dependency guarantee.

6. **Byte-growth threshold.** With old byte quantum `0x18000`, growth writes quantum `0x30000`. That request still uses the unsigned maximum of `0x18000` and the length.

7. **Byte-growth failure.** A checked nonzero result causes free-marker restoration, linkage to the current free head, publication of the selected identifier as head, and return of that result. Restoration of the old quantum is not required.

8. **Different growth-request quanta.** Suppose entry quantum `2` doubles to `4`, so the entry-table request is `32` bytes. If that successful call changes the quantum to `6`, the counter request is `24` bytes. If counter growth fails after changing the quantum to `7`, compensation requests `-56` bytes, and the counter-growth failure is returned regardless of compensation’s result.

9. **Post-growth initialization state.** Suppose successful required growth leaves count `9`, quantum `3`, and a particular current table base. With no subsequent mutations, initialization starts at index `9` of that base and marks indices `9`, `10`, and `11` free. Their first two links become identifiers `11` and `12`; the final second word is untouched. The published head is `10`, and the updated count is `12`, regardless of the count or quantum before growth.

## Uncertainties

The following remain **documented_only**:

- Dependency ownership, synchronization, relocation, preservation, and failure semantics beyond visible result handling.
- Exact ABI arguments of the first availability call.
- Comparison identity, transfer semantics, overlap support, and transfer failure reporting.
- Initialization of the final free-list link and newly added counters.
- Validity of specialized-lookup output and offset-provider results.
- Continued validity of retained pointers and captured bases across calls.
- Caller restrictions on aliasing among output, object, sequence, and backing storage.

High-bit lengths participate in signed capacity comparison and can skip growth despite being large unsigned quantities. Zero growth quantum does not establish progress: initialization can visit no entries yet still publish count plus one as head. Malformed lists, negative counts, arithmetic overflow, and invalid pointers have no established safe recovery behavior.

## Evidence

This contract uses only the supplied static representation and dependency-ID mapping. Established evidence includes access widths, offsets, branch conditions, call order, checked and ignored results, retained values, field rereads, and explicit write order.

Interning, reference counting, resource acquisition/release, and managed storage are inferred interpretations, not dependency guarantees.

No dependency implementations were available. No runtime tests, execution traces, or memory-layout experiments have been performed.
